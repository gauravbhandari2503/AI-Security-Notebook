# Authentication & Authorization Rules

> **Audience:** You (the AI assistant). These are binding rules for generating authentication and authorization logic in this codebase, with a strong focus on secure frontend architecture.

## 1. Zero Trust and Security by Obscurity

- **PROHIBITED:** Never implement "security by obscurity." Never hide UI elements or endpoints as the sole method of authorization.
- **INSTEAD:** Always enforce access control explicitly on the server-side for every protected route, regardless of whether the client interface hides the entry point. Conditionally rendering UI components (like an "Admin Panel" button) is for UX, not security.

## 2. Server-Side Role Enforcement

- **PROHIBITED:** Never trust role claims, permissions, or user properties sent directly from the client (e.g., trust an `isAdmin: true` payload in a request body). Never assume a frontend JWT decode gives absolute truth without backend validation.
- **INSTEAD:** Always derive the user's role and identity from a secure, server-verified session (HttpOnly cookie) or a cryptographically verified JWT on the backend.

## 3. Insecure Direct Object References (IDOR)

- **PROHIBITED:** Never return, modify, or delete a database record solely based on an ID provided in the request URL or body without checking ownership.
- **INSTEAD:** Always scope database queries to the authenticated user.
  - _Example:_ Instead of `SELECT * FROM invoices WHERE id = ?`, use `SELECT * FROM invoices WHERE id = ? AND user_id = ?`.

## 4. Session & Token Storage

- **PROHIBITED:** Never store sensitive authentication tokens (like access JWTs or raw session IDs) in `localStorage` or `sessionStorage`. They are vulnerable to synchronous XSS attacks.
- **INSTEAD:** Always store auth tokens in `HttpOnly`, `Secure`, `SameSite=Strict` (or `Lax`) cookies. Use `localStorage` only for non-sensitive public state (e.g., theme preference, preferred language). If short-lived access tokens must be stored in memory, keep them in private variables or a state management store (like Vuex, Pinia, or Redux) that resets on hard refresh.

## 5. Frontend Route Guards & Protected Views

- **PROHIBITED:** Never allow unauthenticated users to load protected frontend routes and encounter unexpected errors (flashing content before crashing).
- **INSTEAD:** Implement frontend route guards (e.g., `beforeEach` in Vue Router, or layout wrappers in React Component/Next.js). If the user isn't authenticated, immediately redirect them to the `/login` route with a `?redirect=` parameter to preserve their intended destination. For role-based routes, redirect unauthorized users to a dedicated `403 Forbidden` page.

## 6. Centralized HTTP Interceptors (Error Handling)

- **PROHIBITED:** Never manually handle `401 Unauthorized` or `403 Forbidden` responses individually across dozens of different API calls.
- **INSTEAD:** Use a centralized HTTP client interceptor (e.g., Axios interceptors or specialized fetch wrappers).
  - On **401 Unauthorized**, seamlessly trigger a token refresh flow. If the refresh fails, automatically log the user out and redirect to `/login`.
  - On **403 Forbidden**, redirect the user to a permission denied view and display a user-friendly error toast.

## 7. Cross-Site Scripting (XSS) in Authentication Flows

- **PROHIBITED:** Never render unescaped user-provided data directly into the DOM (e.g., `v-html`, `dangerouslySetInnerHTML`), particularly data associated with user profiles (names, bios). Never hydrate initial global state into the window object (e.g., `window.__INITIAL_STATE__`) without serializing and escaping it.
- **INSTEAD:** Rely on your frontend framework's default text interpolation bindings, which automatically encode and sanitize output. When handling URLs (such as `redirect` parameters after login), validate that they are relative paths to prevent Open Redirect vulnerabilities.

## 8. Cross-Site Request Forgery (CSRF)

- **PROHIBITED:** Never assume that using cookies is automatically secure against cross-site requests.
- **INSTEAD:** Include an anti-CSRF token mechanism when relying exclusively on cookie-based authentication, or configure the authentication cookie with `SameSite=Strict` (or `Lax`) to prevent browsers from sending the auth cookie during cross-origin POST/PUT/DELETE requests. Modern frontend applications must include the CSRF token in the HTTP request headers (e.g., `X-CSRF-Token`).

## 9. Silent Authentication & Refresh Tokens

- **PROHIBITED:** Never force the user to re-enter their credentials frequently due to short-lived access tokens expiring. Never send long-lived refresh tokens in the response body.
- **INSTEAD:** Issue short-lived access tokens and long-lived refresh tokens. Store the refresh token in an `HttpOnly` cookie with a path restricted to the refresh endpoint (e.g., `/api/auth/refresh`). The frontend should silently call this endpoint (using an interceptor, as noted above) to exchange the old session for a newly minted short-lived access token.
