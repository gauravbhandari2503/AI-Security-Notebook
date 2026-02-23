# Authentication & Authorization Rules

> **Audience:** You (the AI assistant). These are binding rules for generating authentication and authorization logic in this codebase.

## 1. Zero Trust and Security by Obscurity

- **PROHIBITED:** Never implement "security by obscurity." Never hide UI elements or endpoints as the sole method of authorization.
- **INSTEAD:** Always enforce access control explicitly on the server-side for every protected route, regardless of whether the client interface hides the entry point.

## 2. Server-Side Role Enforcement

- **PROHIBITED:** Never trust role claims, permissions, or user properties sent directly from the client (e.g., trust an `isAdmin: true` payload in a request body).
- **INSTEAD:** Always derive the user's role and identity from a secure, server-verified session (HttpOnly cookie) or a cryptographically verified JWT.

## 3. Insecure Direct Object References (IDOR)

- **PROHIBITED:** Never return, modify, or delete a database record solely based on an ID provided in the request URL or body without checking ownership.
- **INSTEAD:** Always scope database queries to the authenticated user.
  - _Example:_ Instead of `SELECT * FROM invoices WHERE id = ?`, use `SELECT * FROM invoices WHERE id = ? AND user_id = ?`.

## 4. Session & Token Storage

- **PROHIBITED:** Never store sensitive authentication tokens (like JWTs or raw session IDs) in `localStorage` or `sessionStorage`.
- **INSTEAD:** Always store auth tokens in `HttpOnly`, `Secure`, `SameSite=Strict` (or `Lax`) cookies. Use `localStorage` only for non-sensitive public state (e.g., theme preference).
