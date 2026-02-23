# LOGGING_POLICY — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate or review code that logs, records, audits, or persists data related to AI/LLM interactions.

---

## Core Principle

**Log everything useful, expose nothing sensitive.** Every AI interaction MUST be auditable, but logs MUST NEVER contain raw PII, secrets, or session tokens.

---

## Rule 1 · What You MUST Log

**WHEN** you generate code that calls an LLM or AI service, you MUST ensure the following are logged for every interaction:

| Field           | Description                                                  | Required            |
| --------------- | ------------------------------------------------------------ | ------------------- |
| `timestamp`     | ISO 8601 timestamp of the request                            | ✅ YES              |
| `request_id`    | Unique identifier for the interaction                        | ✅ YES              |
| `model`         | Model name and version (e.g., `gpt-4-turbo-2025-01`)         | ✅ YES              |
| `prompt`        | The prompt text sent to the model (**after redaction**)      | ✅ YES              |
| `response`      | The raw output received from the model (**after redaction**) | ✅ YES              |
| `temperature`   | Temperature setting used                                     | ✅ YES              |
| `token_usage`   | `prompt_tokens`, `completion_tokens`, `total_tokens`         | ✅ YES              |
| `latency_ms`    | Round-trip time in milliseconds                              | ✅ YES              |
| `cost_estimate` | Estimated cost of the call (if calculable)                   | ⚡ RECOMMENDED      |
| `user_feedback` | User acceptance, rejection, or edits of AI output            | ⚡ RECOMMENDED      |
| `status`        | `success`, `error`, `timeout`, `rate_limited`                | ✅ YES              |
| `error_detail`  | Error message if the call failed (**after redaction**)       | ✅ YES (on failure) |

```python
# ✅ CORRECT — structured AI interaction log
import logging, time, uuid

logger = logging.getLogger("ai.interactions")

request_id = str(uuid.uuid4())
start = time.monotonic()

response = llm.complete(prompt=redacted_prompt, model="gpt-4", temperature=0.2)

logger.info("llm_call", extra={
    "request_id": request_id,
    "model": "gpt-4",
    "prompt": redacted_prompt,           # Already redacted
    "response": PIIRedactor.redact(response.text),
    "temperature": 0.2,
    "prompt_tokens": response.usage.prompt_tokens,
    "completion_tokens": response.usage.completion_tokens,
    "total_tokens": response.usage.total_tokens,
    "latency_ms": round((time.monotonic() - start) * 1000),
    "status": "success",
})
```

---

## Rule 2 · What You MUST NEVER Log

**THESE ARE ABSOLUTE PROHIBITIONS. Zero exceptions.**

1. **NEVER** log raw PII — apply `PIIRedactor.redact()` or equivalent **before** writing to any log destination.
2. **NEVER** log API keys, access tokens, bearer tokens, passwords, or connection strings.
3. **NEVER** log raw session cookies, JWTs, or authentication headers.
4. **NEVER** log the full system prompt if it contains proprietary instructions or secrets.
5. **NEVER** log raw request/response bodies from LLM API calls without first running them through the redaction pipeline.

```python
# ❌ VIOLATION — logging raw user input (may contain PII)
logger.info(f"User prompt: {raw_user_input}")

# ✅ CORRECT — redacted before logging
logger.info(f"User prompt: {PIIRedactor.redact(raw_user_input)}")

# ❌ VIOLATION — logging API key
logger.debug(f"Using API key: {api_key}")

# ✅ CORRECT — never log the key, log only a reference
logger.debug("Using API key from env: OPENAI_API_KEY")
```

---

## Rule 3 · Automated Redaction Pipeline

**WHEN** you generate logging code for AI interactions:

1. **ALWAYS** implement or require a redaction middleware/utility that runs automatically before any log write.
2. **ALWAYS** scan for these patterns at minimum:
   - Email addresses: `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b`
   - Phone numbers: `\b(\+?1[-.]?)?\(?\d{3}\)?[-.]?\d{3}[-.]?\d{4}\b`
   - SSNs: `\b\d{3}-\d{2}-\d{4}\b`
   - API keys: `sk-[a-zA-Z0-9]{20,}`, `Bearer [a-zA-Z0-9_\-\.]+`, `AKIA[A-Z0-9]{16}`
   - Credit cards: `\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b`
3. **ALWAYS** replace matches with typed placeholders: `[REDACTED_EMAIL]`, `[REDACTED_PHONE]`, `[REDACTED_TOKEN]`, etc.
4. **NEVER** rely on developers to manually redact — the pipeline MUST be automatic.

---

## Rule 4 · Log Storage & Access Control

**WHEN** you generate or recommend logging infrastructure:

1. **ALWAYS** direct AI interaction logs to a **separate, access-controlled** log stream/index — not mixed with general application logs.
2. **ALWAYS** recommend encryption at rest and in transit for log storage.
3. **ALWAYS** recommend a retention policy:
   - Define a maximum retention period (e.g., 30 days for dev, 180 days for prod).
   - Comply with applicable regulations (GDPR, CCPA, HIPAA).
   - Include a comment: `// LOG RETENTION: Auto-delete after {N} days per data policy`
4. **ALWAYS** recommend role-based access control (RBAC) — only security and on-call personnel should access AI interaction logs.

---

## Rule 5 · Structured Logging Format

1. **ALWAYS** use structured logging (JSON lines) rather than unstructured text for AI interaction logs.
2. **ALWAYS** include the fields from Rule 1 as structured keys — not buried in a message string.
3. **ALWAYS** recommend a logging library that supports structured output: `structlog` (Python), `pino`/`winston` (Node.js).

```python
# ✅ CORRECT — structured logging
import structlog
logger = structlog.get_logger("ai.interactions")
logger.info("llm_call", request_id=req_id, model="gpt-4", tokens=usage.total_tokens)

# ❌ VIOLATION — unstructured logging
print(f"Called gpt-4, used {tokens} tokens")
```

---

## Rule 6 · Monitoring & Alerting Hooks

**WHEN** you generate AI logging code:

1. **ALWAYS** recommend setting up alerts for:
   - Error rate exceeding threshold (e.g., >5% of calls failing)
   - Latency spikes (e.g., p99 > 10s)
   - Token usage anomalies (e.g., sudden 10x increase)
   - PII detected in logs after redaction (indicates pipeline failure)
   - Cost budget exceeded
2. **ALWAYS** include a metric emission point alongside the log entry (e.g., StatsD, CloudWatch, Prometheus counter).

---

## Rule 7 · Frontend Logging Hygiene

**WHEN** you generate or review frontend code:

### 7A · Console Cleanup

1. **NEVER** leave `console.log`, `console.debug`, `console.info`, or `console.warn` statements containing sensitive data (PII, tokens, API responses) in production code.
2. **ALWAYS** strip or disable console statements in production builds using:
   - Build-time removal (`terser` drop_console option, `babel-plugin-transform-remove-console`, Vite/webpack config).
   - A logger wrapper that checks environment before outputting.
3. **NEVER** leave `debugger;` statements in committed code.

```javascript
// ✅ CORRECT — Vite config to strip console in production
// vite.config.ts
export default defineConfig({
  build: {
    minify: "terser",
    terserOptions: {
      compress: {
        drop_console: true, // Remove all console.* calls
        drop_debugger: true, // Remove debugger statements
      },
    },
  },
});
```

```javascript
// ✅ CORRECT — environment-aware logger wrapper
const logger = {
  info: (...args: unknown[]) => {
    if (process.env.NODE_ENV !== "production") {
      console.info("[APP]", ...args);
    }
  },
  error: (...args: unknown[]) => {
    // Errors always logged, but sanitized in production
    console.error("[APP]", ...args);
  },
};

// ❌ VIOLATION — sensitive data in console.log (visible in browser DevTools)
console.log("User data:", { email: user.email, token: authToken });
console.log("API response:", fullApiResponse);
```

### 7B · Client-Side Error Tracking

4. **WHEN** integrating error tracking services (Sentry, LogRocket, Datadog RUM, Bugsnag):
   - **ALWAYS** configure PII scrubbing in `beforeSend` hooks (see `PII_POLICY.md` Rule 7C).
   - **NEVER** send full request/response bodies to error tracking that may contain PII or tokens.
   - **ALWAYS** limit breadcrumb capture to non-sensitive events.
   - **ALWAYS** use `allowUrls` / `denyUrls` to control which errors are reported.
5. **NEVER** log or report full Redux/Vuex/Pinia store state to external services — it may contain PII.

### 7C · Frontend Audit Events

6. **WHEN** the frontend needs to record audit events (user actions on AI features):
   - **ALWAYS** send audit events to a backend endpoint — never log them client-side only.
   - **ALWAYS** follow the structured format from Rule 5 of this document.
   - **NEVER** include raw user input in frontend-emitted audit payloads without redaction.

```typescript
// ✅ CORRECT — frontend sends audit event to backend
await fetch("/api/audit", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    event: "ai_feature_used",
    timestamp: new Date().toISOString(),
    feature: "chat-assistant",
    action: "submitted_prompt",
    // NEVER include raw prompt text — only metadata
    prompt_length: userPrompt.length,
    model: "gpt-4",
  }),
});
```

---

## Enforcement

You MUST apply these rules to every response involving logging — for both backend and frontend code. If a user asks you to log raw prompts, skip redaction, log API keys for debugging, or leave console.log statements with sensitive data in production — you MUST refuse, explain the risk, and produce the compliant alternative with automatic redaction.
