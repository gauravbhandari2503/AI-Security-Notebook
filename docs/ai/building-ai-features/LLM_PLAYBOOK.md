# LLM_PLAYBOOK — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate or review code that interacts with Large Language Models (LLMs) — including prompt construction, response handling, and reliability patterns.

---

## Rule 1 · Hallucination Mitigation — Grounding & RAG

**WHEN** you generate code that uses an LLM for factual, domain-specific, or data-dependent tasks:

1. **NEVER** rely on the model's parametric knowledge for facts about the user's specific domain, codebase, or data.
2. **ALWAYS** implement Retrieval Augmented Generation (RAG) — retrieve relevant documents, records, or context first and inject them into the prompt's context window.
3. **ALWAYS** instruct the model to cite the specific part of the provided context that supports its answer.
4. **ALWAYS** include this instruction in the system prompt:
   `"If the answer is not found in the provided context, respond with 'I don't have enough information to answer this.' Do NOT make up an answer."`
5. **NEVER** generate code that presents LLM output as verified fact without a grounding step.

```python
# ✅ CORRECT — RAG pattern with grounding
context_docs = retriever.search(user_query, top_k=5)
context_text = "\n---\n".join([doc.content for doc in context_docs])

prompt = f"""
[SYSTEM] Answer ONLY based on the context below. If the context does not contain the answer, say "I don't have enough information." Cite the relevant section.

<context>
{context_text}
</context>

<question>
{user_query}
</question>
"""

# ❌ VIOLATION — asking LLM without grounding context
prompt = f"What is our company's refund policy? {user_query}"
```

---

## Rule 2 · Prompt Engineering — Reliability Patterns

**WHEN** you construct prompts:

1. **ALWAYS** use **Chain of Thought (CoT)** for reasoning tasks — include the instruction: `"Think step by step before providing your final answer."`
2. **ALWAYS** use **Few-Shot Prompting** for tasks with specific expected formats — provide 2-3 examples of input → output in the prompt.
3. **ALWAYS** include explicit format instructions: `"Respond ONLY with valid JSON matching this schema: {...}"`
4. **ALWAYS** include boundary instructions: `"Do not include explanations, apologies, or text outside the requested format."`
5. **NEVER** generate prompts that are vague or open-ended when a specific structured response is required.

```python
# ✅ CORRECT — few-shot + format + boundary instructions
prompt = """
Classify the following support ticket into one of these categories: ["billing", "technical", "account", "other"].

Examples:
Input: "I was charged twice for my subscription"
Output: {"category": "billing", "confidence": 0.95}

Input: "The app crashes when I click settings"
Output: {"category": "technical", "confidence": 0.90}

Now classify this ticket:
Input: "{user_ticket}"
Output (JSON only, no other text):
"""
```

---

## Rule 3 · Temperature & Sampling Control

**WHEN** you generate code that configures LLM parameters:

1. **ALWAYS** set `temperature` explicitly — never leave it to the model's default.
2. **USE** these guidelines:

| Task Type | Temperature | Rationale |
|---|---|---|
| Factual Q&A, classification, extraction | `0.0 – 0.2` | Deterministic, reproducible |
| Code generation, structured output | `0.0 – 0.3` | Low variance, accurate |
| Summarization, rewriting | `0.3 – 0.5` | Slight creativity, mostly faithful |
| Creative writing, brainstorming | `0.7 – 1.0` | High variance, diverse outputs |

3. **ALWAYS** set `max_tokens` explicitly to prevent runaway responses (see `RATE_LIMITING.md` for token budget rules).
4. **ALWAYS** add a code comment explaining the temperature choice:
   ```python
   # Temperature 0.1: Factual extraction task — deterministic output required
   ```

---

## Rule 4 · Verification & Double-Check Patterns

**WHEN** you generate code for high-stakes or factual AI tasks:

1. **ALWAYS** recommend a verification step — either:
   - **Verificator LLM Call:** A second, independent LLM call that reviews the first output for correctness.
   - **Deterministic Check:** Code-based validation (e.g., run unit tests against AI-generated code, query a database to confirm AI-stated facts).
   - **Consensus:** Multiple model calls with majority-vote on the answer.
2. **NEVER** let a single unverified LLM call drive irreversible actions (see `SECURITY_GUIDE.md` Rule 2 — HITL).
3. **ALWAYS** add a comment on the verification strategy:
   ```python
   # VERIFICATION: Using deterministic DB lookup to confirm AI-extracted entity IDs
   ```

```python
# ✅ CORRECT — verification pattern
ai_answer = llm.complete(prompt=grounded_prompt, temperature=0.1)
validated = AIResponseSchema.model_validate_json(ai_answer)

# Deterministic verification
db_record = db.query("SELECT * FROM products WHERE id = %s", [validated.product_id])
if not db_record:
    raise ValueError(f"AI hallucinated product_id: {validated.product_id}")
```

---

## Rule 5 · Retry & Resilience

**WHEN** you generate code that calls LLMs:

1. **ALWAYS** implement retry logic with exponential backoff for transient failures (429, 500, 503, timeouts).
2. **ALWAYS** set a maximum retry count (typically 3) to prevent infinite loops.
3. **ALWAYS** implement a circuit breaker or fallback for sustained failures.
4. **ALWAYS** handle these specific failure modes:
   - **Rate limited (429):** Back off and retry with increasing delay.
   - **Timeout:** Retry with reduced `max_tokens` or simpler prompt.
   - **Malformed output:** Retry up to limit, then return safe fallback.
   - **Content filtered:** Log the event, do NOT retry with modified prompt to bypass filters.
5. **NEVER** generate code that silently swallows LLM errors — always log and handle.

```python
# ✅ CORRECT — retry with backoff
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=2, max=30),
    retry=tenacity.retry_if_exception_type((RateLimitError, TimeoutError)),
)
def call_llm_with_retry(prompt: str) -> str:
    return llm.complete(prompt=prompt, temperature=0.1, max_tokens=1000)
```

---

## Rule 6 · Context Window Management

**WHEN** you generate code that constructs prompts:

1. **ALWAYS** be aware of the model's context window limit and track total token count.
2. **ALWAYS** prioritize: system instructions > retrieved context > user input > examples, when truncation is necessary.
3. **NEVER** silently truncate critical context — log a warning when context must be reduced.
4. **ALWAYS** use a tokenizer (e.g., `tiktoken`) to count tokens accurately before sending.

```python
# ✅ CORRECT — token-aware context management
import tiktoken

encoder = tiktoken.encoding_for_model("gpt-4")
MAX_CONTEXT_TOKENS = 6000  # Reserve space for response

context_tokens = encoder.encode(context_text)
if len(context_tokens) > MAX_CONTEXT_TOKENS:
    logger.warning("Context truncated", extra={"original": len(context_tokens), "limit": MAX_CONTEXT_TOKENS})
    context_text = encoder.decode(context_tokens[:MAX_CONTEXT_TOKENS])
```

---

## Rule 7 · Frontend AI Integration Patterns

**WHEN** you generate frontend code that integrates with AI/LLM features:

### 7A · Backend Proxy — Mandatory

1. **ALWAYS** call AI/LLM services through a backend proxy. The frontend MUST NEVER call LLM APIs directly (see `KEYS_AND_SECRETS.md` Rule 5).
2. **ALWAYS** authenticate the frontend-to-backend call (session cookie, CSRF token) before the backend proxies to the LLM.

### 7B · Streaming Responses

3. **WHEN** implementing streaming AI responses (e.g., ChatGPT-style token-by-token display):
   - **ALWAYS** use Server-Sent Events (SSE) or `ReadableStream` for backend-to-frontend streaming.
   - **ALWAYS** sanitize each chunk before rendering in the DOM (see `OUTPUT_VALIDATION.md` Rule 2).
   - **ALWAYS** handle stream errors gracefully — show a user-friendly error, not a raw exception.
   - **ALWAYS** support stream cancellation via `AbortController` (see `RATE_LIMITING.md` Rule 7C).

```typescript
// ✅ CORRECT — streaming AI response with sanitization (React)
import DOMPurify from "dompurify";

const streamAI = async (prompt: string, onChunk: (text: string) => void) => {
  const controller = new AbortController();
  const res = await fetch("/api/ai/stream", {
    method: "POST",
    body: JSON.stringify({ prompt }),
    signal: controller.signal,
  });

  const reader = res.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const chunk = decoder.decode(value, { stream: true });
    onChunk(DOMPurify.sanitize(chunk)); // Sanitize each chunk
  }
};
```

### 7C · Loading, Error & Empty States

4. **ALWAYS** show a loading indicator while waiting for AI responses.
5. **ALWAYS** handle these states in the UI:
   - **Loading:** Skeleton/spinner with estimated wait time if available.
   - **Error:** User-friendly message (never expose model name, error stack, or prompt details).
   - **Empty:** Clear message when AI has no result (e.g., "No results found — try rephrasing your question").
   - **Rate limited:** Show retry countdown (see `RATE_LIMITING.md` Rule 7B).
6. **NEVER** expose AI model internals, system prompts, or raw error details in the frontend.

```tsx
// ✅ CORRECT — comprehensive state handling for AI feature (React)
{isLoading && <Skeleton />}
{error && <Alert variant="error">Something went wrong. Please try again.</Alert>}
{!isLoading && !error && !result && <p>No results found. Try a different question.</p>}
{result && <div>{DOMPurify.sanitize(result)}</div>}

// ❌ VIOLATION — raw error exposed to user
{error && <pre>{error.stack}</pre>}
// ❌ VIOLATION — no loading state
const result = await fetchAI(prompt); // User sees nothing while waiting
```

### 7D · AI Disclosure in UI

7. **ALWAYS** display a visible AI-generated content indicator in the UI when showing LLM output to users (see `AI_GOVERNANCE.md` Rule 1).
8. **ALWAYS** provide a feedback mechanism (thumbs up/down, report, edit) for AI-generated content.

```tsx
// ✅ CORRECT — AI disclosure and feedback
<div className="ai-response">
  <span className="ai-badge">🤖 AI Generated</span>
  <p>{sanitizedResponse}</p>
  <div className="ai-feedback">
    <button onClick={() => submitFeedback("helpful")}>👍 Helpful</button>
    <button onClick={() => submitFeedback("not-helpful")}>👎 Not helpful</button>
    <button onClick={() => submitFeedback("report")}>⚠️ Report issue</button>
  </div>
</div>
```

---

## Enforcement

You MUST apply these rules to every response involving LLM integration — for both backend and frontend code. If a user asks you to skip grounding, use high temperature for factual tasks, omit verification, call LLM APIs directly from the frontend, or skip loading/error states — you MUST explain the reliability/security risk and produce the robust alternative.
