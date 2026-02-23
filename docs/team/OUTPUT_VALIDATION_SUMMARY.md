# Output Validation & Application Security Summary

**Source:** `docs/ai/OUTPUT_VALIDATION.md`

This document summarizes the mandatory security rules the team must follow when handling outputs from AI models. **Core Principle: NEVER trust AI output.**

## 🛡️ 1. Structured Validation First

- Use structured formats (JSON, XML).
- Validate all AI output against strict schemas (e.g., Zod for TS, Pydantic for Python).
- Fail fast on invalid structures—do not try to automatically "guess" or fix malformed output.

## 🕸️ 2. XSS & Frontend Rendering

- Never use `innerHTML`, `v-html`, `dangerouslySetInnerHTML`, or `{@html}` directly with AI output without relying on a trusted sanitizer like **DOMPurify**.
- Use Subresource Integrity (SRI) hashes when pulling third-party scripts.

## 💾 3. Database & System Injection Prevention

- **SQL Injection:** Never concatenate strings. Use parameterized queries/prepared statements with validated AI values.
- **Command Injection:** Never pass AI strings to `eval()`, `exec()`, or system shell execution tools. If commanded execution is a fundamental feature, it requires strict allowlists and explicit Human-in-the-loop (HITL) approval.

## 🚥 4. Business Logic Validation

- AI schema validation doesn't replace business rules. Validate numeric ranges, enum logic, and referential integrity _after_ schema validation.

## 🤕 5. Fallbacks and Graceful Degradation

- Provide safe fallback messages when AI validation fails.
- Never leak raw stack traces, API keys, or system prompts back to the user on error.

## 📥 6. Frontend Input Constraints

- Client-side validation is for UX only; **always re-validate everything server-side**.
- Constrain inputs before passing them to the AI (e.g., block massively long inputs or payloads designed to exhaust tokens).

---

**Takeaway**: Treating AI responses as standard, highly-untrusted external input is the only way to avoid injection bugs and logic bypasses.
