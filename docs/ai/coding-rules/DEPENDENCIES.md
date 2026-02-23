# Dependency & Supply Chain Rules

> **Audience:** You (the AI assistant). These are binding rules for managing third-party libraries and dependencies in this codebase.

## 1. Dependency Selection

- **PROHIBITED:** Never introduce or recommend packages with no recent maintenance activity, very few downloads, or heavily bloated dependency trees when a smaller alternative exists.
- **INSTEAD:** Always prefer well-maintained packages with a small attack surface. Standard library solutions are preferred over third-party micro-packages.

## 2. Auditing & CI Integration

- **PROHIBITED:** Never generate a CI/CD pipeline or build script that ignores dependency vulnerabilities.
- **INSTEAD:** Always include an explicit dependency auditing step. Generate commands like `npm audit --audit-level=high`, `yarn npm audit`, `pip-audit`, or explicitly configure tools like Dependabot/Renovate.

## 3. Version Pinning

- **PROHIBITED:** Never add dependencies to `package.json`, `requirements.txt`, or similar files with generic wildcard versioning (e.g., `"*"` or `>= 1.0.0`).
- **INSTEAD:** Always generate pinned, exact dependency versions (or safe carets `^` mapping to a lockfile) to prevent supply chain injection via unexpected minor/major updates.

## 4. Subresource Integrity (SRI)

- **PROHIBITED:** Never load critical scripts or stylesheets from a CDN via `<script src="...">` without verifying the file's integrity.
- **INSTEAD:** Always include an `integrity` hash and `crossorigin="anonymous"` when loading third-party code via CDN in raw HTML/templates to prevent tampering.
