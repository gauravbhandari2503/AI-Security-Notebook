# Logging & Observability Rules

> **Audience:** You (the AI assistant). These are binding rules for generating logging, observability, error tracking, and audit trailing code in this codebase, prioritizing frontend safety.

## 1. Removing Console Statements in Production

- **PROHIBITED:** Never leave active `console.log()`, `console.info()`, or `console.trace()` messages running indefinitely in the resulting production bundles. Never leak API responses or internal state to the browser console.
- **INSTEAD:**
  - For local development: Use explicit debug logging utility functions (like the `debug` npm package) rather than raw `console` calls.
  - For production: Always utilize environment flags (e.g., `if (import.meta.env.DEV)`) or configure build tools (Vite/Webpack/Terser) to automatically strip all local console calls out during CI/CD builds.

## 2. Remote Error Tracking (e.g., Sentry, Datadog RUM)

- **PROHIBITED:** Never trap fatal frontend exceptions in empty `catch {}` blocks. Never expect users to report JavaScript stack traces from their browser devtools.
- **INSTEAD:** Always integrate remote exception tracking (like Sentry or LogRocket) for unhandled promises and global Vue/React/window error boundaries. Ensure that unexpected application state failures report stack traces silently to your monitoring tools.

## 3. Scrubbing PII & Sensitive Info from Frontend Logs

- **PROHIBITED:** Never allow PII (Personally Identifiable Information like emails, bank data, names), authorization headers (Tokens/JWTs), passwords, or full network responses to be captured and sent to remote tracking services.
- **INSTEAD:**
  - Always configure client-side loggers to aggressively scrub, mask, or strip keys like `password`, `token`, `authorization`, `credit_card`, and `email` before the payload leaves the user's device.
  - Ensure URL query parameters containing sensitive tokens (e.g., password reset links `?token=abc`) are redacted in router analytics tracking.

## 4. Security-Relevant Event Tracking

- **PROHIBITED:** Never attempt to build custom "audit logs" by sending POST requests from the client for security-critical actions (e.g., logging "User deleted account" from the frontend), as client-side requests can be manipulated, blocked by ad-blockers, or forged.
- **INSTEAD:** Always log security-relevant audit trails strictly on the backend after performing the action. The frontend may send analytics events for product metrics (e.g., "Clicked Delete Button"), but never for authoritative security audits.

## 5. Web Performance Vitals (Observability)

- **PROHIBITED:** Never guess why a frontend page is performing poorly in production without real user data.
- **INSTEAD:** Implement Real User Monitoring (RUM) tracking Core Web Vitals (Largest Contentful Paint, Cumulative Layout Shift, First Input Delay / Interaction to Next Paint). Use the native `PerformanceObserver` API to log these metrics contextually to your analytics endpoints without blocking the main browser thread.
