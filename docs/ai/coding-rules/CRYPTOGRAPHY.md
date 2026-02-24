# Cryptography Rules

> **Audience:** You (the AI assistant). These are binding rules for generating cryptographic operations (hashing, encryption, signing) in this codebase, including frontend and backend environments.

## 1. Custom Cryptography

- **PROHIBITED:** Never implement custom encryption algorithms, custom hashing logic, or "roll your own crypto."
- **INSTEAD:** Always use established, audited standard libraries (e.g., Node's `crypto` module, Python's `hashlib` or `cryptography` libraries, or libsodium). For frontend operations, utilize the native `window.crypto` (Web Crypto API).

## 2. Cryptography in the Browser

- **PROHIBITED:** Never encrypt sensitive data (like passwords, PII, or financial records) purely on the frontend as a replacement for transport security. Never store static, hardcoded cryptographic keys, secret sauces, or signing secrets in the frontend bundle (e.g., in `.env` files exposed to React/Vue/Vite) where anyone can read them.
- **INSTEAD:** Rely on HTTPS/TLS for secure data transmission. If client-side encryption is strictly required (e.g., End-to-End Encryption or encrypting local PWA storage), use the Web Crypto API framework natively (`crypto.subtle`), deriving keys securely (e.g., using PBKDF2) derived from user passwords, never from bundled static secrets.

## 3. Password Hashing (Backend primarily, but relevant to Auth flows)

- **PROHIBITED:** Never use MD5, SHA1, SHA256, or plain text for storing user passwords, nor attempt to hash them on the frontend and send the hash to the server instead of the raw password.
- **INSTEAD:** Always use purpose-built, memory-hard algorithms specifically designed for password hashing on the server. Prefer **Argon2**, **Bcrypt**, or **Scrypt** with appropriately high work factors/salt rounds. The frontend should send the password securely over HTTPS.

## 4. Token & Signature Verification

- **PROHIBITED:** Never manually parse and "verify" JSON Web Tokens (JWTs) or other signed payloads by splitting strings. Never accept `alg: "none"` in a JWT header.
- **INSTEAD:** Always use a reputable library (e.g., `jsonwebtoken`, `PyJWT`) and explicitly enforce the expected algorithm (e.g., `algorithms: ["RS256"]`). Ensure the signature is verified on the backend before reading any claims. The frontend can decode a JWT payload for UI purposes (like displaying a username) but must never trust those claims for authorization logic.

## 5. Randomness & Secrets Generation

- **PROHIBITED:** Never use `Math.random()` or time-based seeds for generating passwords, state tokens (e.g., OAuth state or CSRF tokens), API keys, session identifiers, or initialization vectors (IVs).
- **INSTEAD:** Always use a Cryptographically Secure Pseudorandom Number Generator (CSPRNG).
  - On the backend: `crypto.randomBytes()`, or Python's `secrets` module.
  - On the frontend: `window.crypto.getRandomValues()`.

## 6. Secure Contexts & TLS/HTTPS

- **PROHIBITED:** Never perform cryptographic operations, access PII, or transmit sensitive credentials over insecure HTTP connections.
- **INSTEAD:** Ensure the entire application strictly enforces HTTPS (HSTS). The Web Crypto API (`window.crypto.subtle`) is only available in Secure Contexts (HTTPS or localhost).
