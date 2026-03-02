# KEYS_AND_SECRETS — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow whenever you generate, review, or suggest code that involves API keys, tokens, passwords, certificates, or any form of secret.

---

## Rule 1 · Never Hardcode Secrets

This is an absolute, zero-exception rule.

1. **NEVER** produce code that contains a literal secret value — no API keys, passwords, tokens, connection strings, or certificates inline in source code.
2. **NEVER** include placeholder strings that look like real keys (e.g., `sk-abc123...`). Use clearly fake placeholders: `YOUR_API_KEY_HERE`, `<REPLACE_WITH_SECRET>`, or environment variable references.
3. **ALWAYS** load secrets from environment variables or a secret manager.

```python
# ✅ CORRECT — from environment
import os
api_key = os.environ["OPENAI_API_KEY"]

# ✅ CORRECT — from secret manager
from aws_secretsmanager import get_secret
api_key = get_secret("openai-api-key")

# ❌ VIOLATION — hardcoded secret
api_key = "sk-proj-abc123def456ghi789"
```

```typescript
// ✅ CORRECT
const apiKey = process.env.OPENAI_API_KEY;

// ❌ VIOLATION
const apiKey = "sk-proj-abc123def456ghi789";
```

---

## Rule 2 · Environment Variable & Secret Manager Patterns

**WHEN** you generate code that requires secrets:

1. **ALWAYS** use `process.env.SECRET_NAME` (Node.js), `os.environ["SECRET_NAME"]` (Python), or the language-equivalent environment variable accessor.
2. **ALWAYS** add a validation check that the secret exists at startup — fail fast with a clear error rather than proceeding with `undefined`.
3. **ALWAYS** recommend `.env` files for local development and note that `.env` MUST be in `.gitignore`.
4. **ALWAYS** recommend a dedicated secret manager for production (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, GCP Secret Manager).

```typescript
// ✅ CORRECT — fail-fast validation
const apiKey = process.env.OPENAI_API_KEY;
if (!apiKey) {
  throw new Error(
    "OPENAI_API_KEY environment variable is required but not set.",
  );
}
```

---

## Rule 3 · Git Safety

**WHEN** you generate or modify project configuration:

1. **ALWAYS** ensure `.env`, `.env.local`, `.env.*.local`, `*.pem`, `*.key`, and `credentials.json` are listed in `.gitignore`.
2. **ALWAYS** recommend pre-commit secret scanning tools: `trufflehog`, `git-secrets`, or `detect-secrets`.
3. **WHEN** you detect that a `.gitignore` is missing these entries, you MUST flag it and generate the fix.
4. **NEVER** generate CI/CD config that prints, echoes, or logs secret values — even for debugging.

```gitignore
# ✅ REQUIRED entries in .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
credentials.json
service-account.json
```

---

## Rule 4 · Least Privilege & Key Rotation

**WHEN** you recommend or configure API keys:

1. **ALWAYS** recommend scoping keys to the minimum required permissions.
2. **ALWAYS** recommend separate keys per environment (development, staging, production).
3. **ALWAYS** recommend a key rotation schedule and include a comment:
   ```
   // KEY ROTATION: This key should be rotated every 90 days or immediately upon suspected compromise.
   ```
4. **NEVER** generate code that shares a single key across multiple environments or services.

---

## Rule 5 · Client-Side Safety

1. **NEVER** generate frontend/client-side code (browser, mobile, SPA) that directly references private API keys.
2. **ALWAYS** route AI API calls through a backend proxy that holds the secret and authenticates the client.
3. **ONLY** use public-prefixed environment variables in frontend code (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `NUXT_PUBLIC_`) and verify they contain **non-secret** values only (e.g., public URLs, feature flags).

```typescript
// ✅ CORRECT — frontend calls backend proxy
const response = await fetch("/api/ai/complete", {
  method: "POST",
  body: JSON.stringify({ prompt: userInput }),
});

// ❌ VIOLATION — private key in frontend code
const openai = new OpenAI({ apiKey: "sk-..." });
```

### 5A · SDK Key Scoping (Google Maps, Firebase, Stripe, etc.)

4. **WHEN** frontend code requires a public API key (e.g., Google Maps, Firebase, Stripe publishable key):
   - **ALWAYS** restrict the key to specific domains/referrers in the provider's console.
   - **ALWAYS** restrict the key to only the APIs it needs (e.g., Maps JavaScript API only, not Geocoding + Places + Directions).
   - **ALWAYS** add a code comment noting the restriction:
     ```typescript
     // KEY SCOPE: Google Maps key restricted to *.example.com, Maps JS API only
     ```
   - **NEVER** use an unrestricted public API key in production.

```typescript
// ✅ CORRECT — scoped Firebase config (public keys restricted at provider level)
const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY, // Restricted to example.com in Firebase console
  authDomain: "myapp.firebaseapp.com",
  projectId: "myapp",
  // KEY SCOPE: Firebase API key restricted to example.com domain, Auth + Firestore only
};
```

### 5B · SSR / SSG Server-Only Secrets

5. **WHEN** generating code for SSR/SSG frameworks (Next.js, Nuxt, SvelteKit, Remix):
   - **ALWAYS** place secrets in server-only contexts:
     - **Next.js:** `getServerSideProps`, `getStaticProps`, API routes, Route Handlers, Server Components, Server Actions. Access via `process.env.SECRET_NAME` (no `NEXT_PUBLIC_` prefix).
     - **Nuxt:** `server/` directory only. Access via `useRuntimeConfig().secretKey` (not `public`).
     - **SvelteKit:** `+page.server.ts`, `+server.ts`, `hooks.server.ts` only. Access via `$env/static/private`.
   - **NEVER** import or reference server-only secrets in client-side components or pages.

```typescript
// ✅ CORRECT — Next.js server-only secret access
// app/api/ai/route.ts (Server Route Handler)
export async function POST(req: Request) {
  const apiKey = process.env.OPENAI_API_KEY; // Server-only, NOT prefixed with NEXT_PUBLIC_
  // ... use apiKey to call LLM
}

// ❌ VIOLATION — secret in client component
("use client");
const apiKey = process.env.OPENAI_API_KEY; // This will be undefined (or worse, bundled if misconfigured)
```

```typescript
// ✅ CORRECT — Nuxt server-only
// server/api/ai.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig();
  const apiKey = config.openaiApiKey; // Server-only runtime config
  // ...
});
```

### 5C · Source Maps & Build Artifacts

6. **NEVER** enable source maps in production builds — they expose original source code and may reveal secrets, internal logic, or API patterns.
7. **ALWAYS** disable source maps in production configuration:

```javascript
// ✅ CORRECT — Next.js: next.config.js
module.exports = {
  productionBrowserSourceMaps: false, // Default, but be explicit
};

// ✅ CORRECT — Vite: vite.config.ts
export default defineConfig({
  build: {
    sourcemap: false, // Disable in production
  },
});
```

### 5D · Pre-Signed URLs for File Uploads

8. **WHEN** generating code for file uploads:
   - **ALWAYS** use pre-signed URLs (S3, GCS, Azure Blob) generated by the backend.
   - **NEVER** embed cloud storage credentials or access keys in frontend upload code.

```typescript
// ✅ CORRECT — frontend requests pre-signed URL from backend, then uploads directly
const { uploadUrl } = await fetch("/api/upload/presign", {
  method: "POST",
}).then((r) => r.json());
await fetch(uploadUrl, { method: "PUT", body: file });

// ❌ VIOLATION — AWS credentials in frontend
import AWS from "aws-sdk";
const s3 = new AWS.S3({ accessKeyId: "AKIA...", secretAccessKey: "..." });
```

---

## Rule 6 · Secret Detection in Code Review

**WHEN** you review code:

1. **ALWAYS** scan for patterns that resemble secrets: `sk-`, `Bearer `, `-----BEGIN`, `AKIA`, `ghp_`, `glpat-`, `xox[bpas]-`.
2. **IF** you detect any, you MUST flag it immediately as a **critical security violation** and recommend remediation.
3. **ALWAYS** recommend adding CI-level secret scanning (e.g., GitHub Secret Scanning, TruffleHog GitHub Action).

---

---

## Rule 7 · Prompt Secret Scrubbing

**WHEN** processing user input or developer prompts before sending them to an LLM:

1. **ALWAYS** intercept and scrub potential secrets or API keys from the prompt text.
2. **IF** a user or developer mistakenly pastes a secret key into the AI system prompt, the system **MUST NOT** forward that secret to the external LLM provider for security purposes.
3. **ALWAYS** replace detected secrets with a safe placeholder (e.g., `[REDACTED_SECRET]`) before making the LLM API call.
4. **ALWAYS** log the interception event (without logging the actual secret) for security auditing.

---

## Enforcement

You MUST apply these rules to every response — for both backend and frontend code. If a user asks you to hardcode a key, embed a secret in a commit, expose a private key in frontend code, ship source maps to production, or use unrestricted public API keys — you MUST refuse, explain the security risk, and produce the compliant alternative.
