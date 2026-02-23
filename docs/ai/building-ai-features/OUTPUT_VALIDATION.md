# OUTPUT_VALIDATION — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate or review code that consumes, displays, stores, or acts upon output from an LLM or any AI model.

---

## Core Principle

**NEVER trust AI output.** Treat every response from an LLM — including your own — as untrusted input that MUST be validated, sanitized, and constrained before use.

---

## Rule 1 · Structured Output & Schema Validation

**WHEN** you generate code that processes LLM responses:

1. **ALWAYS** request structured output (JSON, XML) from the LLM instead of free-form text wherever possible.
2. **ALWAYS** validate the response against a strict schema before use:
   - **Python:** Use `Pydantic` with `BaseModel` or `TypeAdapter`.
   - **TypeScript:** Use `Zod` with `.parse()` (not `.safeParse()` unless you handle the error explicitly).
3. **ALWAYS** fail fast if validation fails — do NOT attempt to guess, patch, or auto-correct malformed output.
4. **ALWAYS** implement retry logic (with a retry limit) when validation fails, rather than silently accepting bad data.

```python
# ✅ CORRECT — strict schema validation
from pydantic import BaseModel, Field

class AIResponse(BaseModel):
    summary: str = Field(..., min_length=1, max_length=5000)
    confidence: float = Field(..., ge=0.0, le=1.0)

try:
    validated = AIResponse.model_validate_json(llm_output)
except ValidationError:
    # Retry or return safe fallback — NEVER use unvalidated output
    raise
```

```typescript
// ✅ CORRECT — Zod schema validation
import { z } from "zod";

const AIResponseSchema = z.object({
  summary: z.string().min(1).max(5000),
  confidence: z.number().min(0).max(1),
});

const validated = AIResponseSchema.parse(JSON.parse(llmOutput));
// Throws ZodError if invalid — this is the correct behavior
```

---

## Rule 2 · XSS Prevention (Web Output)

**WHEN** you generate code that renders LLM output — or any user/external data — in a web interface:

1. **NEVER** insert untrusted output into the DOM using `innerHTML`, `v-html`, or `dangerouslySetInnerHTML` without sanitization.
2. **ALWAYS** sanitize HTML output using a trusted library: `DOMPurify` (browser), `sanitize-html` (Node.js), `bleach` (Python).
3. **ALWAYS** prefer rendering as plain text (`textContent`, `innerText`, or framework auto-escaping) unless HTML rendering is explicitly required.
4. **ALWAYS** scan for dangerous patterns before rendering: `<script`, `javascript:`, `onerror=`, `onclick=`, `onload=`, `<iframe`.

### Framework-Specific Rules

5. **React:** **NEVER** use `dangerouslySetInnerHTML` with unsanitized data. If required, wrap with `DOMPurify.sanitize()` first.
6. **Vue:** **NEVER** use `v-html` with unsanitized data. Always pass through `DOMPurify.sanitize()` first.
7. **Angular:** Rely on Angular's built-in sanitization. **NEVER** bypass it with `bypassSecurityTrustHtml()` unless the content is from a fully trusted, server-controlled source — and add a code comment justifying the bypass.
8. **Svelte:** **NEVER** use `{@html ...}` with unsanitized data.

```typescript
// ✅ CORRECT — sanitized before rendering
import DOMPurify from "dompurify";
element.innerHTML = DOMPurify.sanitize(llmOutput);

// ✅ BETTER — plain text when HTML is not needed
element.textContent = llmOutput;

// ✅ CORRECT — React with DOMPurify
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(aiResponse) }} />

// ✅ CORRECT — Vue with DOMPurify
<template>
  <div v-html="sanitized"></div>
</template>
<script setup>
import DOMPurify from "dompurify";
const sanitized = computed(() => DOMPurify.sanitize(aiResponse.value));
</script>

// ❌ VIOLATION — unsanitized LLM output in DOM
element.innerHTML = llmOutput;
// ❌ VIOLATION — raw v-html
<div v-html="userInput" />
// ❌ VIOLATION — raw dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{ __html: llmOutput }} />
```

### Subresource Integrity (SRI)

9. **ALWAYS** add `integrity` and `crossorigin` attributes when loading scripts or stylesheets from external CDNs.
10. **NEVER** load third-party scripts from CDNs without SRI hashes.

```html
<!-- ✅ CORRECT — SRI hash on CDN resource -->
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxEnd=="
        crossorigin="anonymous"></script>

<!-- ❌ VIOLATION — CDN script without SRI -->
<script src="https://cdn.example.com/lib.js"></script>
```

---

## Rule 3 · SQL Injection Prevention

**WHEN** you generate code that uses LLM output in database operations:

1. **NEVER** interpolate or concatenate LLM output into SQL strings.
2. **ALWAYS** use parameterized queries / prepared statements.
3. **ALWAYS** validate that LLM-generated values conform to expected types and ranges before using them as query parameters.

```python
# ✅ CORRECT — parameterized query
cursor.execute("SELECT * FROM products WHERE category = %s", [validated_output.category])

# ❌ VIOLATION — string interpolation with LLM output
cursor.execute(f"SELECT * FROM products WHERE category = '{llm_output}'")
```

---

## Rule 4 · Command Injection Prevention

1. **NEVER** generate code that executes LLM output as a shell command, system call, or eval expression.
2. **IF** a use case absolutely requires executing AI-suggested commands:
   - **ALWAYS** validate against a strict allowlist of permitted commands.
   - **ALWAYS** require explicit human confirmation before execution.
   - **ALWAYS** add a comment: `// HITL: AI-generated command — requires human approval`
3. **NEVER** use `eval()`, `exec()`, `os.system()`, `child_process.exec()`, or similar with LLM output.

---

## Rule 5 · Business Logic Validation

**WHEN** LLM output drives business decisions:

1. **ALWAYS** apply domain-specific constraints after schema validation:
   - Numeric ranges (e.g., discount ≤ 50%, quantity > 0)
   - Enum membership (e.g., status must be one of `["pending", "approved", "rejected"]`)
   - Referential integrity (e.g., referenced IDs must exist)
2. **ALWAYS** cross-check AI output against deterministic sources of truth where possible.
3. **NEVER** let LLM output override security-critical business rules (authorization, pricing, access levels).

```python
# ✅ CORRECT — business rule applied after schema validation
validated = AIResponse.model_validate_json(llm_output)
if validated.discount > 50:
    raise ValueError("AI suggested discount exceeds maximum allowed (50%)")
```

---

## Rule 6 · Fallback & Graceful Degradation

1. **ALWAYS** implement a safe fallback for when LLM output fails validation:
   - Return a default safe value, show a generic message, or retry with a simpler prompt.
2. **NEVER** expose raw validation errors, stack traces, or model internals to end users.
3. **ALWAYS** log validation failures for monitoring (following `LOGGING_POLICY.md` redaction rules).

---

## Rule 7 · Frontend Input Validation

**WHEN** you generate frontend code that accepts user input (forms, search bars, file uploads, URL parameters):

1. **ALWAYS** validate input on BOTH client-side AND server-side. Client-side validation is for UX; server-side validation is the security boundary.
2. **ALWAYS** constrain inputs: max length, allowed characters, type checks, enum membership.
3. **NEVER** trust client-side validation alone — all data MUST be re-validated on the server.
4. **ALWAYS** sanitize file upload filenames and validate MIME types server-side.
5. **ALWAYS** use URL parameter validation — never parse query strings directly into DOM or database queries.

```typescript
// ✅ CORRECT — client-side + server-side validation (React + Zod)
import { z } from "zod";

// Shared schema — used on both frontend and backend
const ContactFormSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email().max(255),
  message: z.string().min(1).max(5000),
});

// Frontend: validate before submit
const result = ContactFormSchema.safeParse(formData);
if (!result.success) {
  setErrors(result.error.flatten());
  return;
}

// Backend: re-validate (NEVER trust frontend)
app.post("/api/contact", (req, res) => {
  const validated = ContactFormSchema.parse(req.body); // Throws on invalid
  // ... process validated data
});

// ❌ VIOLATION — trusting frontend input without server-side validation
app.post("/api/contact", (req, res) => {
  db.insert(req.body); // Dangerous — no validation
});
```

### Scoped Styles

6. **ALWAYS** use scoped CSS (Vue `<style scoped>`, CSS Modules, Styled Components, Tailwind) to prevent style injection and CSS exfiltration attacks.
7. **NEVER** dynamically generate CSS from user input or LLM output without sanitization.

---

## Enforcement

You MUST apply these rules to every response — for both backend and frontend code. If a user asks you to use raw LLM output in SQL, DOM, or shell execution without validation, or render unsanitized HTML in any frontend framework — you MUST refuse, explain the injection/XSS risk, and produce the safe alternative.
