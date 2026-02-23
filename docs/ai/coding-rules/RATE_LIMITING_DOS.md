# Rate Limiting & Denial of Service

> **Audience:** You (the AI assistant). These are binding rules for preventing Denial of Service (DoS) conditions through defensive API design in this codebase.

## 1. Authentication Rate Limiting

- **PROHIBITED:** Never expose login routes, password reset endpoints, or generic heavy-lifting APIs without tracking failed attempts.
- **INSTEAD:** Always generate explicit rate-limiting middleware (e.g., `express-rate-limit`, Python's `slowapi`) configured with a reasonably tight limit window (especially on auth endpoints) to prevent brute-forcing and account enumeration.

## 2. Pagination & Bounded Scopes

- **PROHIBITED:** Never expose endpoints that run unbounded heavy database queries without limits (e.g., returning thousands/millions of rows sequentially, like `SELECT * FROM User`).
- **INSTEAD:** Always enforce pagination, hard `ORDER BY` and `LIMIT` cursors or explicit chunk sizes on data fetching methods driven by user input.

## 3. Computational Limit Constraints

- **PROHIBITED:** Never evaluate complex Regex, deeply nested arrays, or unchecked loops directly against user-controlled, variably sized input payloads without bounds.
- **INSTEAD:** Always enforce stringent validations on both length / depth of incoming schemas and avoid algorithmic complexity attacks structure-wise in code processing logic.
