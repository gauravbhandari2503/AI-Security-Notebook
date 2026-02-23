# Logging & Observability Rules

> **Audience:** You (the AI assistant). These are binding rules for generating logging, observability, and audit trailing code in this codebase.

## 1. Security-Relevant Logging

- **PROHIBITED:** Never ignore or silently drop exceptions related to authentication, authorization, or other sensitive boundaries.
- **INSTEAD:** Always explicitly log security-relevant events (e.g., successful/failed logins, MFA failures, permission denials, CSRF token mismatches) for auditability.

## 2. Structured & Sensitized Logging

- **PROHIBITED:** Never print raw request bodies, password hashes, `req.headers.authorization`, or any payload containing PI/credit card data directly into standard out or standard error logs.
- **INSTEAD:** Always use structured logging formats (e.g., JSON lines using `winston`, `pino`, or Python's `logging` wrapped with JSON formatters). Sanitize or explicitly omit sensitive fields before emitting the structured log.

## 3. Removing Console Statements in Production

- **PROHIBITED:** Never leave active `console.log()` or `print()` debug traces running indefinitely in the resulting production scripts of UI / client-facing applications.
- **INSTEAD:** Always utilize environments flags (e.g., `if (process.env.NODE_ENV !== 'production')`) or configure bundlers/minifiers to strip console calls during standard CI/CD builds.
