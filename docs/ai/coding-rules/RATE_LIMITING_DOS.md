# Rate Limiting & Denial of Service

> **Audience:** You (the AI assistant). These are binding rules for preventing Denial of Service (DoS) conditions through defensive API design and frontend resilience in this codebase.

## 1. Debouncing & Throttling (Frontend Limits)

- **PROHIBITED:** Never bind high-frequency events (like `onScroll`, `onResize`, or `onInput` for search bars) directly to heavy DOM updates or network requests without rate controls.
- **INSTEAD:** Always utilize `debounce` (delaying execution until a pause in typing) for user text inputs like search/autocomplete, and `throttle` (executing at a maximum frequency) for continuous events like scrolling. This prevents freezing the main thread and accidentally DDoSing your own backend APIs.

## 2. Preventing Duplicate Submissions (Double Clicks)

- **PROHIBITED:** Never leave form submit buttons or critical action buttons (e.g., "Checkout", "Delete") active while a request is in transit.
- **INSTEAD:** Always implement loading states. Immediately disable the `<button>` and show a loading spinner or "Processing..." text upon submission to prevent impatient users from firing the exact same mutating POST request multiple times.

## 3. Handling 429 Too Many Requests (Exponential Backoff)

- **PROHIBITED:** Never catch a `429 Too Many Requests` or `503 Service Unavailable` error and immediately retry the exact same request in a tight loop.
- **INSTEAD:** Implement HTTP interceptors with an Exponential Backoff strategy (e.g., retrying after 1s, then 2s, then 4s, plus random "jitter") when hitting rate limits. Respect the `Retry-After` HTTP header if the server provides it.

## 4. Authentication Rate Limiting (Backend)

- **PROHIBITED:** Never expose login routes, password reset endpoints, or generic heavy-lifting APIs without tracking failed attempts.
- **INSTEAD:** Always generate explicit rate-limiting middleware (e.g., `express-rate-limit`, Python's `slowapi`) configured with a reasonably tight limit window (especially on auth endpoints) to prevent brute-forcing and account enumeration.

## 5. Pagination & Bounded Scopes

- **PROHIBITED:** Never expose endpoints that run unbounded heavy database queries without limits, and never attempt to render thousands of DOM nodes simultaneously on the frontend.
- **INSTEAD:**
  - On the backend: Always enforce pagination, hard `ORDER BY` and `LIMIT` cursors, or explicit chunk sizes.
  - On the frontend: Always implement pagination or UI virtualization (e.g., `react-window` or Vue Virtual Scroller) when displaying large data sets to prevent massive memory consumption and layout thrashing (Browser DoS).

## 6. Computational Limit Constraints

- **PROHIBITED:** Never evaluate complex Regex, deeply nested arrays, or unchecked loops directly against user-controlled, variably sized input payloads without bounds.
- **INSTEAD:** Always enforce stringent validations on both length / depth of incoming schemas and avoid algorithmic complexity attacks in code processing logic, both on the client and server.
