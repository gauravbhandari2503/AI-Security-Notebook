# Dependency & Supply Chain Rules

> **Audience:** You (the AI assistant). These are binding rules for managing third-party libraries and dependencies in this codebase, with a strong emphasis on frontend performance and security.

## 1. Minimal Client-Side Dependencies (Native First)

- **PROHIBITED:** Never introduce heavy, outdated legacy libraries when native Browser APIs are well-supported (e.g., recommending `moment.js` instead of native `Intl` or `date-fns`; recommending `jQuery` instead of native DOM querying; recommending `axios` if native `fetch` suffices).
- **INSTEAD:** Always prioritize native Web APIs or modern, module-based micro-libraries to reduce the impact of JavaScript on the main thread and overall bundle size.

## 2. Bundle Size & Tree-Shaking Support

- **PROHIBITED:** Never confidently add a new `npm` package without considering its impact on the frontend bundle size or its module format. Never recommend packages that do not support tree-shaking (CommonJS heavy packages) if an ES Module (ESM) alternative exists.
- **INSTEAD:** Always favor ESM-compatible packages so build tools (Vite, Webpack, Rollup) can tree-shake unused code. When proposing a new dependency, implicitly consider its "Bundlephobia" cost.

## 3. Typo-Squatting & Dependency Selection

- **PROHIBITED:** Never introduce or recommend packages with no recent maintenance activity, very few downloads, or heavily bloated dependency trees when a smaller alternative exists.
- **INSTEAD:** Always verify the exact, correct spelling of the package name to prevent typo-squatting supply chain attacks. Prefer well-maintained, battle-tested utilities from reputable publishers.

## 4. Auditing & CI Integration

- **PROHIBITED:** Never generate a CI/CD pipeline or build script that ignores frontend dependency vulnerabilities.
- **INSTEAD:** Always include an explicit dependency auditing step. Generate commands like `npm audit --audit-level=high` or configure automated tools like Dependabot/Renovate to keep core frontend frameworks and libraries secure and up to date.

## 5. Version Pinning & Lockfiles

- **PROHIBITED:** Never add dependencies to `package.json` with generic wildcard versioning (e.g., `*` or `>= 1.0.0`). Never instruct developers to blindly delete `package-lock.json` or `yarn.lock` to "fix" an install error.
- **INSTEAD:** Always generate pinned, exact dependency versions (e.g., `1.2.3`) or safe carets (`^1.2.3`) mapping strictly to a committed lockfile. This ensures reproducible frontend builds and prevents supply chain injection via unexpected updates.

## 6. Subresource Integrity (SRI) for Third-Party CDNs

- **PROHIBITED:** Never load critical layout scripts, analytics, or stylesheets from an external CDN via `<script src="...">` or `<link href="...">` in an `index.html` file without verifying the file's integrity.
- **INSTEAD:** Always include an `integrity` hash (e.g., `sha384-...`) and `crossorigin="anonymous"` when loading third-party code via CDN in raw HTML/templates. This prevents malicious modification if the external CDN is compromised.
