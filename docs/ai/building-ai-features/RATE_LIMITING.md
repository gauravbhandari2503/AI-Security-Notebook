# RATE_LIMITING — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate or review code that calls LLM APIs or any external AI service. Rate limiting and token control are critical for cost safety, abuse prevention, and system reliability.

---

## Rule 1 · Per-User / Per-Session Rate Limits

**WHEN** you generate code that exposes AI functionality to users:

1. **ALWAYS** implement rate limiting on AI endpoints — no AI API route should be unlimited.
2. **ALWAYS** enforce limits at **multiple layers**:
   - **Per-user:** Max N requests per minute/hour (e.g., 20 requests/min per authenticated user).
   - **Per-IP:** Fallback limit for unauthenticated or anonymous access (e.g., 5 requests/min per IP).
   - **Global:** Hard ceiling across all users to protect against coordinated abuse (e.g., 1000 requests/min total).
3. **ALWAYS** return HTTP `429 Too Many Requests` with a `Retry-After` header when limits are exceeded.
4. **NEVER** generate AI endpoints without rate limiting — even for internal/admin APIs.

```python
# ✅ CORRECT — rate-limited AI endpoint (Python / FastAPI example)
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/ai/complete")
@limiter.limit("20/minute")
async def ai_complete(request: Request, body: PromptRequest):
    # ... AI call logic
    pass
```

```typescript
// ✅ CORRECT — rate-limited AI endpoint (Express example)
import rateLimit from "express-rate-limit";

const aiLimiter = rateLimit({
  windowMs: 60 * 1000,   // 1 minute
  max: 20,                // 20 requests per window per user
  standardHeaders: true,  // Return rate limit info in headers
  legacyHeaders: false,
  message: { error: "Rate limit exceeded. Please retry after the indicated time." },
});

app.use("/api/ai/", aiLimiter);
```

---

## Rule 2 · Token Budgets & Max Tokens

**WHEN** you generate code that calls an LLM:

1. **ALWAYS** set `max_tokens` (or equivalent) explicitly on every LLM API call. NEVER leave it undefined or set it to the model's maximum.
2. **ALWAYS** calculate and enforce a **per-request token budget**:
   - `total_budget = prompt_tokens + max_completion_tokens`
   - Ensure `total_budget ≤ model_context_window`
3. **ALWAYS** enforce a **per-user daily/hourly token budget** to prevent cost explosion from a single user.
4. **ALWAYS** track cumulative token usage and reject requests that would exceed the budget.

```python
# ✅ CORRECT — explicit token limits with budget tracking
MAX_TOKENS_PER_REQUEST = 1000
MAX_TOKENS_PER_USER_PER_HOUR = 50_000

# Check user budget before calling LLM
user_usage = token_tracker.get_usage(user_id, window="1h")
if user_usage + MAX_TOKENS_PER_REQUEST > MAX_TOKENS_PER_USER_PER_HOUR:
    raise TokenBudgetExceeded(f"Hourly token limit reached: {user_usage}/{MAX_TOKENS_PER_USER_PER_HOUR}")

response = llm.complete(
    prompt=safe_prompt,
    max_tokens=MAX_TOKENS_PER_REQUEST,  # ✅ Always explicit
    temperature=0.2,
)

# Track actual usage
token_tracker.record(user_id, response.usage.total_tokens)
```

```typescript
// ✅ CORRECT — explicit max_tokens
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: prompt }],
  max_tokens: 1000,  // ✅ ALWAYS set explicitly
  temperature: 0.2,
});

// ❌ VIOLATION — no max_tokens set (model will use maximum, causing unpredictable cost)
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: prompt }],
});
```

---

## Rule 3 · Input Length Validation

**WHEN** you generate code that accepts user input for AI processing:

1. **ALWAYS** validate and enforce a maximum input length **before** sending to the LLM.
2. **ALWAYS** count tokens (not characters) using a tokenizer — characters undercount the actual cost.
3. **ALWAYS** reject inputs that exceed the limit with a clear error message.
4. **NEVER** silently truncate user input — inform the user that their input is too long.

```python
# ✅ CORRECT — input length validation
import tiktoken

MAX_INPUT_TOKENS = 4000
encoder = tiktoken.encoding_for_model("gpt-4")

input_tokens = len(encoder.encode(user_input))
if input_tokens > MAX_INPUT_TOKENS:
    raise InputTooLong(
        f"Input is {input_tokens} tokens, maximum is {MAX_INPUT_TOKENS}. "
        "Please shorten your input."
    )
```

---

## Rule 4 · Cost Controls & Billing Alerts

**WHEN** you generate or architect AI-powered services:

1. **ALWAYS** recommend setting billing alerts / hard spending limits in the AI provider's dashboard.
2. **ALWAYS** implement application-level cost tracking:
   - Calculate estimated cost per request: `(prompt_tokens × input_price + completion_tokens × output_price)`
   - Track cumulative daily/monthly spend.
   - Auto-disable the AI feature if spend exceeds the budget threshold.
3. **ALWAYS** add a cost comment on LLM calls:
   ```python
   # COST: gpt-4 @ ~$0.03/1K input + $0.06/1K output — budget: $500/month
   ```
4. **NEVER** deploy AI features to production without a cost circuit breaker.

```python
# ✅ CORRECT — cost circuit breaker
MONTHLY_BUDGET_USD = 500.0

current_spend = cost_tracker.get_monthly_spend()
estimated_cost = estimate_call_cost(prompt_tokens=len(prompt), model="gpt-4")

if current_spend + estimated_cost > MONTHLY_BUDGET_USD:
    logger.critical("AI budget exceeded", extra={"spend": current_spend, "budget": MONTHLY_BUDGET_USD})
    raise BudgetExceeded("Monthly AI budget has been reached. Service temporarily disabled.")
```

---

## Rule 5 · Abuse Prevention Patterns

**WHEN** you generate AI endpoints:

1. **ALWAYS** implement these abuse prevention layers:
   - **Authentication required:** No anonymous access to AI features.
   - **Request deduplication:** Reject identical prompts from the same user within a short window (e.g., 5 seconds).
   - **Prompt length limits:** Reject excessively long inputs (Rule 3).
   - **Response caching:** Cache identical prompts to avoid redundant LLM calls.
2. **ALWAYS** log rate limit violations for security monitoring (following `LOGGING_POLICY.md`).
3. **ALWAYS** implement progressive penalties for repeat offenders (e.g., increasing backoff periods).

```python
# ✅ CORRECT — deduplication + caching
import hashlib

prompt_hash = hashlib.sha256(safe_prompt.encode()).hexdigest()

# Check cache first
cached = cache.get(f"ai:response:{prompt_hash}")
if cached:
    return cached  # Serve from cache — no LLM call, no cost

# Check deduplication
if dedup_store.exists(f"ai:dedup:{user_id}:{prompt_hash}"):
    raise DuplicateRequest("Duplicate request detected. Please wait before retrying.")

dedup_store.set(f"ai:dedup:{user_id}:{prompt_hash}", ttl=5)  # 5-second dedup window
```

---

## Rule 6 · Queueing & Concurrency Control

**WHEN** you generate code for high-traffic AI services:

1. **ALWAYS** use a request queue (Redis, SQS, Bull) to manage AI API call concurrency.
2. **ALWAYS** limit concurrent LLM API calls to prevent provider-side rate limiting (e.g., max 10 concurrent calls).
3. **ALWAYS** provide feedback to queued users (estimated wait time, queue position).
4. **NEVER** fire-and-forget LLM calls without tracking their lifecycle.

```python
# ✅ CORRECT — semaphore-based concurrency control
import asyncio

MAX_CONCURRENT_LLM_CALLS = 10
llm_semaphore = asyncio.Semaphore(MAX_CONCURRENT_LLM_CALLS)

async def rate_controlled_llm_call(prompt: str) -> str:
    async with llm_semaphore:
        return await llm.acomplete(prompt=prompt, max_tokens=1000)
```

---

## Rule 7 · Client-Side Rate Limiting & UX

**WHEN** you generate frontend code that calls AI-powered endpoints:

### 7A · Throttling & Debouncing

1. **ALWAYS** debounce user-triggered AI calls (e.g., search-as-you-type, auto-complete) — minimum 300ms debounce.
2. **ALWAYS** throttle repeated submissions — disable the submit button after click and re-enable only after the response completes or a timeout.
3. **NEVER** allow rapid-fire API calls from the frontend (e.g., on every keystroke without debounce).

```typescript
// ✅ CORRECT — debounced AI search input (React)
import { useDebouncedCallback } from "use-debounce";

const handleSearch = useDebouncedCallback(async (query: string) => {
  if (query.length < 3) return; // Minimum input threshold
  const result = await fetch(`/api/ai/search?q=${encodeURIComponent(query)}`);
  setResults(await result.json());
}, 300); // 300ms debounce

// ✅ CORRECT — disable button during submission
const [isSubmitting, setIsSubmitting] = useState(false);

const handleSubmit = async () => {
  setIsSubmitting(true);
  try {
    await fetch("/api/ai/complete", { method: "POST", body: JSON.stringify({ prompt }) });
  } finally {
    setIsSubmitting(false);
  }
};

<button disabled={isSubmitting} onClick={handleSubmit}>
  {isSubmitting ? "Processing..." : "Submit"}
</button>

// ❌ VIOLATION — unthrottled AI call on every keystroke
<input onChange={(e) => fetch(`/api/ai/search?q=${e.target.value}`)} />
```

### 7B · Rate Limit Feedback to Users

4. **ALWAYS** handle HTTP `429` responses gracefully in the frontend:
   - Parse the `Retry-After` header.
   - Display a user-friendly message with the wait time.
   - Automatically retry after the wait period (with a maximum retry count).
5. **NEVER** show raw error messages like "429 Too Many Requests" to users.
6. **ALWAYS** show loading/progress indicators for AI operations to prevent impatient re-submissions.

```typescript
// ✅ CORRECT — graceful 429 handling in frontend
const callAI = async (prompt: string, retries = 0): Promise<string> => {
  const res = await fetch("/api/ai/complete", {
    method: "POST",
    body: JSON.stringify({ prompt }),
  });

  if (res.status === 429) {
    if (retries >= 3) {
      throw new Error("Service is busy. Please try again later.");
    }
    const retryAfter = parseInt(res.headers.get("Retry-After") || "5", 10);
    setStatus(`Busy — retrying in ${retryAfter}s...`);
    await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
    return callAI(prompt, retries + 1);
  }

  if (!res.ok) throw new Error("Request failed. Please try again.");
  return res.json();
};
```

### 7C · Abort / Cancel Support

7. **ALWAYS** implement request cancellation (using `AbortController`) for AI calls that are superseded by new user input.
8. **ALWAYS** cancel in-flight requests when the user navigates away from the page/component.

```typescript
// ✅ CORRECT — AbortController for cancellable AI requests
const abortControllerRef = useRef<AbortController | null>(null);

const fetchAI = async (prompt: string) => {
  // Cancel previous in-flight request
  abortControllerRef.current?.abort();
  abortControllerRef.current = new AbortController();

  const res = await fetch("/api/ai/complete", {
    method: "POST",
    body: JSON.stringify({ prompt }),
    signal: abortControllerRef.current.signal,
  });
  return res.json();
};

// Cleanup on unmount
useEffect(() => () => abortControllerRef.current?.abort(), []);
```

---

## Enforcement

You MUST apply these rules to every response involving AI API calls — for both backend and frontend code. If a user asks you to create an AI endpoint without rate limiting, skip token budgets, omit cost controls, or build a frontend without debouncing/throttling — you MUST refuse, explain the cost/abuse risk, and produce the rate-limited, budget-controlled alternative.
