# Frontend Infrastructure & Deployment Rules

> **Audience:** You (the AI assistant). These are binding rules for generating frontend infrastructure configurations, CI/CD pipelines, build processes, and hosting setups.

## 1. Environment Variables in the Frontend

- **PROHIBITED:** Never confidently bundle secrets (e.g., database passwords, master API keys, private AWS credentials) into the frontend build. Never commit `.env.production` or `.env.local` files containing real production URLs or secrets to version control.
- **INSTEAD:** Only prefix environment variables with the framework's public identifier (e.g., `VITE_`, `REACT_APP_`, `NEXT_PUBLIC_`) if they are strictly meant to be exposed to the client (like a public API URL or a Google Analytics ID). Treat all other backend API secrets as strictly server-side.

## 2. Content Security Policy (CSP) & Headers

- **PROHIBITED:** Never deploy a frontend application without configuring basic security headers. Never use `unsafe-inline` or `unsafe-eval` in a CSP policy unless absolutely unavoidable (and explicitly documented).
- **INSTEAD:** Always configure the hosting provider (e.g., Vercel, Netlify, NGINX, AWS CloudFront) to inject a strict Content Security Policy (CSP), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and `Strict-Transport-Security` (HSTS) headers.

## 3. Bundle Optimization & Caching

- **PROHIBITED:** Never configure the web server or CDN to cache the main `index.html` indefinitely. Never serve unminified development bundles in production.
- **INSTEAD:**
  - Configure `index.html` with headers like `Cache-Control: no-cache, no-store, must-revalidate`.
  - Configure strictly hashed static assets (e.g., `app-a8b2c.js`) with aggressive long-term caching (`Cache-Control: public, max-age=31536000, immutable`).
  - Always ensure the CI/CD pipeline runs the framework's production build command (e.g., `npm run build`), which automatically minifies, treeshakes, and chunks the code.

## 4. CI/CD Pipeline Checks

- **PROHIBITED:** Never allow a pull request to merge or a production deployment to succeed if fundamental checks fail.
- **INSTEAD:** Always ensure the frontend CI/CD pipeline (e.g., GitHub Actions, GitLab CI) includes mandatory steps for:
  - `npm ci` (clean install)
  - `npm run lint` (static analysis)
  - `npm test` (unit/integration tests)
  - `npm run build` (verifying the build succeeds)
  - Failing the pipeline if any of these steps error out.

## 5. Hosting & CDN Distribution

- **PROHIBITED:** Never serve a global production frontend application directly from a single VM or container without a CDN in front of it.
- **INSTEAD:** Always deploy frontend assets globally using a Content Delivery Network (CDN) like Cloudflare, AWS CloudFront, Vercel Edge Network, or Netlify to ensure low latency and high availability for global users.
