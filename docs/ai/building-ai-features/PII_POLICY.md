# PII_POLICY — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate, review, or suggest code that handles Personally Identifiable Information (PII) in any AI workflow.

---

## Rule 1 · PII Definition (Know What to Protect)

You MUST recognize the following as PII and apply all protection rules in this document whenever any of these appear in code, prompts, logs, tests, or data flows:

| Category | Examples |
|---|---|
| **Identity** | Full names, usernames, display names |
| **Contact** | Email addresses, phone numbers, physical/mailing addresses |
| **Network** | IP addresses, MAC addresses, device IDs |
| **Government** | SSN, passport numbers, driver's license numbers, tax IDs |
| **Financial** | Credit/debit card numbers, bank account numbers |
| **Biometric** | Fingerprints, facial recognition data, voice prints |
| **Health** | Medical records, health insurance IDs |
| **Auth** | Passwords, security questions/answers, session tokens |

---

## Rule 2 · Pre-Processing — Redact Before Sending to LLMs

**WHEN** you generate or review code that sends data to an external LLM provider:

1. **ALWAYS** ensure a PII scanning and redaction step runs **before** the data reaches the LLM API call.
2. **ALWAYS** use regex-based or NER-based detection to scan for PII patterns.
3. **ALWAYS** replace detected PII with typed placeholders: `[REDACTED_EMAIL]`, `[REDACTED_PHONE]`, `[REDACTED_SSN]`, etc.
4. **NEVER** pass raw user records, profiles, or form submissions to an LLM without redaction.

```python
# ✅ CORRECT — redact before sending
from pii_redactor import PIIRedactor

user_message = "My email is john@example.com and SSN is 123-45-6789"
safe_message = PIIRedactor.redact(user_message)
# Result: "My email is [REDACTED_EMAIL] and SSN is [REDACTED_SSN]"

response = llm.complete(prompt=safe_message)

# ❌ VIOLATION — raw PII sent to external LLM
response = llm.complete(prompt=user_message)
```

---

## Rule 3 · Anonymization Over Deletion

**WHEN** the AI workflow requires distinguishing between entities (e.g., "User A said X, User B said Y"):

1. **ALWAYS** replace real identifiers with stable anonymous tokens: `User-001`, `User-002`, `Entity-A`, etc.
2. **NEVER** use real names, emails, or IDs even if the user provides them in the prompt.
3. **ALWAYS** maintain a mapping only in secure server-side memory — never in logs, prompts, or client-side storage.

---

## Rule 4 · Local Processing Preference

**WHEN** you recommend or architect AI features that process PII-heavy data:

1. **ALWAYS** prefer local/on-premise models (e.g., Ollama, vLLM, self-hosted) that do not transmit data externally.
2. **ONLY** recommend external LLM providers if:
   - PII has been fully redacted **before** the API call, OR
   - The provider has a contractual Zero Data Retention (ZDR) policy (e.g., Azure OpenAI, OpenAI Enterprise with ZDR).
3. **ALWAYS** note the data residency implications in code comments when external providers are used.

---

## Rule 5 · Compliance Enforcement

**WHEN** you generate or review AI features:

1. **ALWAYS** verify the code respects user consent boundaries — never process data beyond what the user explicitly consented to.
2. **ALWAYS** include a mechanism for data subject rights (access, deletion, rectification) if PII is stored.
3. **ALWAYS** add a code comment referencing the applicable regulation when handling PII:
   ```
   // GDPR Art. 17 — Right to erasure: User can request deletion of this data
   // CCPA §1798.105 — Right to delete
   ```
4. **NEVER** generate code that silently collects, stores, or transmits PII without explicit user awareness.

---

## Rule 6 · PII in Test Data and Code Samples

1. **NEVER** use real PII in test fixtures, seed data, example code, or documentation.
2. **ALWAYS** use obviously fake data: `jane.doe@example.com`, `555-0100`, `123 Test Street`.
3. **WHEN** generating test data, use libraries like `Faker` (Python) or `@faker-js/faker` (JS) and note this in the code.

---

## Rule 7 · Client-Side PII Protection (Frontend)

**WHEN** you generate frontend code that handles user data:

### 7A · Browser Storage

1. **NEVER** store PII (names, emails, tokens, SSNs, etc.) in `localStorage` or `sessionStorage` — these are accessible to any JavaScript on the page, including XSS payloads.
2. **ALWAYS** store authentication tokens (JWTs, session IDs) in `HttpOnly`, `Secure`, `SameSite=Strict` cookies — never in `localStorage`.
3. **IF** client-side caching of non-sensitive user preferences is needed, use `sessionStorage` with non-PII identifiers only.

```typescript
// ✅ CORRECT — auth token in HttpOnly cookie (set by server)
// Server sets the cookie:
res.cookie("session_token", token, {
  httpOnly: true,   // Not accessible to JavaScript
  secure: true,     // HTTPS only
  sameSite: "strict",
  maxAge: 3600000,  // 1 hour
});

// Frontend reads auth state from API, NOT from cookie directly
const { user } = await fetch("/api/auth/me").then(r => r.json());

// ❌ VIOLATION — JWT stored in localStorage
localStorage.setItem("auth_token", jwt);
const token = localStorage.getItem("auth_token");

// ❌ VIOLATION — PII in localStorage
localStorage.setItem("user_email", "john@example.com");
```

### 7B · URL Parameters & Browser History

4. **NEVER** include PII in URL query parameters or path segments — URLs are logged by browsers, proxies, CDNs, and analytics.
5. **ALWAYS** use POST request bodies (over HTTPS) for transmitting PII.
6. **NEVER** include PII in `window.location`, `history.pushState`, or client-side routing parameters.

```typescript
// ❌ VIOLATION — PII in URL
window.location.href = "/profile?email=john@example.com&ssn=123-45-6789";

// ✅ CORRECT — PII in POST body
await fetch("/api/profile", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email, ssn }),
});
```

### 7C · Client-Side Analytics & Error Tracking

7. **NEVER** send PII to frontend analytics tools (Google Analytics, Mixpanel, Amplitude) or error tracking services (Sentry, LogRocket, Datadog RUM).
8. **ALWAYS** scrub PII from error payloads before they are sent to external services.
9. **ALWAYS** configure `beforeSend` hooks in error tracking SDKs to strip PII.

```typescript
// ✅ CORRECT — Sentry with PII scrubbing
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  beforeSend(event) {
    // Strip PII from error events
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    return event;
  },
  // NEVER send PII in breadcrumbs
  beforeBreadcrumb(breadcrumb) {
    if (breadcrumb.category === "xhr" || breadcrumb.category === "fetch") {
      delete breadcrumb.data?.url; // May contain PII in query params
    }
    return breadcrumb;
  },
});
```

### 7D · Form Data Handling

10. **ALWAYS** use `autocomplete="off"` or specific autocomplete values on sensitive form fields to prevent browser autofill caching of PII.
11. **NEVER** pre-populate forms with PII from insecure sources (URL params, localStorage).
12. **ALWAYS** clear sensitive form fields from component state after submission.

```tsx
// ✅ CORRECT — secure form handling (React)
<input type="text" name="ssn" autoComplete="off" />
<input type="password" name="password" autoComplete="new-password" />

// ✅ CORRECT — clear sensitive state after submit
const handleSubmit = async (data: FormData) => {
  await submitToServer(data);
  setFormState({ ssn: "", password: "" }); // Clear PII from memory
};
```

---

## Enforcement

You MUST apply these rules to every response — for both backend and frontend code. If a user asks you to send raw PII to an LLM, store tokens in localStorage, include PII in URLs, write tests with real user data, or skip redaction — you MUST refuse, explain the PII risk, and produce the compliant alternative instead.
