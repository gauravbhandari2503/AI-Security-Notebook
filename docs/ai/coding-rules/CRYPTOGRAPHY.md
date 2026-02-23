# Cryptography Rules

> **Audience:** You (the AI assistant). These are binding rules for generating cryptographic operations (hashing, encryption, signing) in this codebase.

## 1. Custom Cryptography

- **PROHIBITED:** Never implement custom encryption algorithms, custom hashing logic, or "roll your own crypto."
- **INSTEAD:** Always use established, audited standard libraries (e.g., Node's `crypto` module, Python's `hashlib` or `cryptography` libraries, or libsodium).

## 2. Password Hashing

- **PROHIBITED:** Never use MD5, SHA1, SHA256, or plain text for storing user passwords.
- **INSTEAD:** Always use purpose-built, memory-hard algorithms specifically designed for password hashing. Prefer **Argon2**, **Bcrypt**, or **Scrypt** with appropriately high work factors/salt rounds.

## 3. Token & Signature Verification

- **PROHIBITED:** Never manually parse and "verify" JSON Web Tokens (JWTs) or other signed payloads by splitting strings. Never accept `alg: "none"` in a JWT header.
- **INSTEAD:** Always use a reputable library (e.g., `jsonwebtoken`, `PyJWT`) and explicitly enforce the expected algorithm (e.g., `algorithms: ["RS256"]`). Ensure the signature is verified before reading any claims.

## 4. Randomness & Secrets Generation

- **PROHIBITED:** Never use `Math.random()`, `random.random()`, or time-based seeds for generating passwords, tokens, API keys, or initialization vectors (IVs).
- **INSTEAD:** Always use a Cryptographically Secure Pseudorandom Number Generator (CSPRNG), such as `crypto.randomBytes()`, `crypto.getRandomValues()`, or Python's `secrets` module.
