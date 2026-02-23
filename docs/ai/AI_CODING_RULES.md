# AI Coding Rules (Production Guidelines)

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever generating, reviewing, or refactoring code in this repository.
>
> _Constraint Format: Every rule explicitly defines what is **PROHIBITED** and provides an exact **INSTEAD** pattern to use._

This document serves as the master index. For detailed examples and context on each category, refer to the specific files in the `docs/ai/coding-rules/` directory.

---

## 1. Secrets & Credentials ([Read More](./coding-rules/INFRASTRUCTURE.md))

- **PROHIBITED:** Never hardcode secrets (API keys, passwords, connection strings) directly into source code, Dockerfiles, or CI/CD pipelines.
- **INSTEAD:** Always retrieve secrets from environment variables or a dedicated secrets manager at runtime. Fail fast during application startup if required keys are missing.

## 2. Authentication & Authorization ([Read More](./coding-rules/AUTH_AND_AUTHZ.md))

- **PROHIBITED:** Never trust role claims sent from the client. Never fetch sensitive records inherently by an ID parameter without verifying user ownership (IDOR).
- **INSTEAD:** Always enforce authorization checks server-side against an authenticated session token (e.g., an HttpOnly cookie or validated JWT) and scope database queries to the authenticated user ID.

## 3. Web Injection & XSS ([Read More](./coding-rules/LLM_INTERACTION.md))

- **PROHIBITED:** Never insert untrusted string input into the DOM (e.g., `innerHTML`, `dangerouslySetInnerHTML`) or directly concatenate input into SQL query strings / LLM prompt bodies.
- **INSTEAD:** Always use parameterized queries for databases. Clean HTML input using sanitizers (like `DOMPurify`) or enforce plain text output (`textContent`). Wrap dynamic inputs for LLMs inside explicit tag boundaries (e.g., `<user-input>`).

## 4. Cryptography ([Read More](./coding-rules/CRYPTOGRAPHY.md))

- **PROHIBITED:** Never implement custom encryption protocols. Never use MD5 or SHA1 for passwords. Never manually splice JSON Web Tokens for verification.
- **INSTEAD:** Always use standard memory-hard algorithms (`bcrypt`, `argon2`) for hashing passwords and verifiable secure libraries (`jsonwebtoken`) for token validation logic.

## 5. Privacy & PII Handling ([Read More](./PII_POLICY.md))

- **PROHIBITED:** Never use genuine user PII in local database seeds, test fixtures, URL paths/queries, or `localStorage`/`sessionStorage` caches in the browser.
- **INSTEAD:** Always use libraries (e.g., Faker) to generate obviously fake data (`jane.doe@example.com`). Store tokens uniquely in `HttpOnly/Secure` cookies payloads natively transmitted via POST requests.

## 6. Dependency & Supply Chain ([Read More](./coding-rules/DEPENDENCIES.md))

- **PROHIBITED:** Never merge un-pinned package dependencies (e.g., using `*` versioning). Never merge packages abandoned by maintainers.
- **INSTEAD:** Always assert exact version pins or restrict to patch updates via `package-lock.json`. Explicitly generate steps invoking `npm audit` or equivalent security checkers dynamically in CI/CD pipeline code.

## 7. Rate Limiting & DoS ([Read More](./coding-rules/RATE_LIMITING_DOS.md))

- **PROHIBITED:** Never deploy authentication or heavy database query routes without bounds or tracking mechanisms.
- **INSTEAD:** Always implement Rate-Limiting middleware specifically on authentication flows. Constrain unbounded loops, apply cursor/offset pagination to DB read scopes, and aggressively cap maximum payload sizes.

## 8. Logging & Observability ([Read More](./coding-rules/LOGGING.md))

- **PROHIBITED:** Never include potentially raw PII payload objects or standard `Authorization` headers mapping in standard debug log tracers traversing the system.
- **INSTEAD:** Always log actionable security events (e.g., permission denials, login failures) utilizing explicit structured JSON log formats, stripping arbitrary request bodies implicitly before stdout parsing.

## 9. Accessibility & DOM Structure ([Read More](./coding-rules/ACCESSIBILITY.md))

- **PROHIBITED:** Never abstract naturally interactive components (like links or buttons) into generic `<div>` wrappers missing semantic outlines or overriding focus rings globally.
- **INSTEAD:** Always generate semantic HTML (`<nav>`, `<button>`, `<main>`). Affix accurate `aria-label` alternatives to interactive vector graphics and strictly preserve global keyboard navigation events.
