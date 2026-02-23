# AI Security Guide Summary

**Source:** `docs/ai/SECURITY_GUIDE.md`

This document summarizes the core security rules that our AI assistants and applications must follow. It serves as a quick reference for the team to ensure we build secure, privacy-respecting, and resilient AI integrations.

---

## 🛡️ Core Security Principles

### 1. Defend Against Prompt Injection

Never trust raw user input. AI models cannot reliably distinguish between application logic and user data if they are mixed together.

- **Always isolate input:** Use explicit delimiters (like `<user_input>`) when inserting user text into prompts.
- **Instruct the AI defensively:** explicitly tell the model to ignore any instructions hidden within user input.

### 2. Enforce Human-in-the-Loop (HITL)

AI should never autonomously execute high-stakes actions.

- **What is high-stakes?** Modifying/deleting data, sending emails, deploying code, financial transactions, or changing permissions.
- **Require confirmation:** Always implement explicit user approval (like a confirmation dialog) before executing these tasks.

### 3. Protect Privacy (Data Minimization)

Never expose more data than absolutely necessary.

- **No PII:** Never include Personally Identifiable Information (PII) in prompts, logs, or test data.
- **Minimize contexts:** Strip out everything except the specific fields the AI needs to complete its task. Don't send full database records or session objects to the LLM.

### 4. Isolate Models by Trust Level

Use different models for different contexts to limit the potential blast radius of a compromised model.

- Keep user-facing chat models separate from models that analyze internal or sensitive data.
- Use built-in model safety filters whenever available.

### 5. Start with Secure Defaults

Always assume a hostile environment and build in safety nets.

- **Deny by default:** Allow access only to explicitly whitelisted resources.
- **Fail safely:** Ensure errors return safe fallbacks rather than leaking raw system details to the user.
- Never disable security features (like SSL or input validation) even in test environments.

---

## 🌐 Frontend Security Fundamentals

When building web interfaces (React, Vue, Next.js, etc.), apply standard web security:

- **CSP (Content Security Policy):** Use strict headers. Disallow `unsafe-inline` and `unsafe-eval`. Avoid wildcard (`*`) sources.
- **CSRF Protection:** Require CSRF tokens on all state-changing endpoints (POST/PUT/DELETE) alongside `SameSite=Strict` or `Lax` cookies.
- **Strict CORS:** Whitelist specific origins. Never use `Access-Control-Allow-Origin: *` for authenticated requests.
- **Security Headers:** Enforce `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and restrict `Permissions-Policy`.
- **Always HTTPS:** Never mix HTTP and HTTPS content. Downgrade attacks must be prevented using HSTS.

---

**Summary Takeaway:** Treat the AI like an untrusted participant on the internet. If you wouldn't let a random user run a specific function, you shouldn't let the AI do it autonomously either!
