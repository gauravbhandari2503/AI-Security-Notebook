# AI_GOVERNANCE — AI Instruction Set

> **Audience:** You (the AI assistant). These are binding rules you MUST follow to ensure responsible, transparent, and compliant use of AI across the organization. This document establishes department-level AI usage guidelines that govern how AI is adopted, monitored, and governed.

**Based on:** [What is AI Governance? | IBM](https://www.ibm.com/think/topics/ai-governance)

## Core Principles of Responsible AI Governance

Before applying specific rules, ensure your code aligns with these foundational principles:

- **Empathy:** Anticipating and addressing the societal impact of AI on all stakeholders.
- **Bias Control:** Rigorously examining training data to prevent embedding real-world biases into algorithms.
- **Transparency:** Maintaining clarity and openness in how AI algorithms operate and make decisions.
- **Accountability:** Proactively setting and adhering to high standards, maintaining responsibility for AI's impacts.

---

## Rule 1 · AI Usage Transparency

**WHEN** you generate code, UI components, or documentation for AI-powered features:

1. **ALWAYS** include a disclaimer in code comments and user-facing copy:
   ```
   // AI DISCLOSURE: This feature uses an LLM to generate responses. Outputs may be inaccurate and should be verified.
   ```

---

## Rule 2 · Approved AI Providers & Models

**WHEN** you recommend, configure, or write integration code for AI/LLM providers:

1. **ONLY** recommend providers that meet these minimum requirements:
   - Enterprise-grade SLA(Service Level Agreement) with uptime guarantees
   - Data Processing Agreement (DPA) available
   - SOC 2 Type II or equivalent certification (Provides Access Control, Encryptio,
     Monitoring, Employee access restricitions and incident handling), Configurable data retention (zero-retention option preferred)
   - Clear prohibited use policy

2. **Recommended providers** (in order of preference for sensitive workloads):
   | Provider | Use Case | Data Residency |
   |---|---|---|
   | Azure OpenAI | Enterprise workloads, compliance-heavy | Regional (configurable) |
   | AWS Bedrock | Multi-model, AWS-native | Regional (configurable) |
   | Self-hosted (Ollama, vLLM) | PII-heavy, air-gapped | On-premise |
   | OpenAI Enterprise (ZDR) | General, with zero-retention | US |
   | Anthropic (Claude) | Reasoning, safety-focused | US |

3. **NEVER** recommend or integrate with AI providers that:
   - Use customer data for training without explicit opt-out
   - Lack a Data Processing Agreement
   - Cannot guarantee data deletion upon request

4. **ALWAYS** add a comment indicating the provider choice rationale:
   ```python
   # GOVERNANCE: Using Azure OpenAI — regional data residency required for GDPR compliance
   ```

---

## Rule 3 · Department-Level Usage Policies

**WHEN** you generate AI features, you MUST ensure the code enforces these department-level constraints:

### Engineering Department

- **Permitted uses:** Code generation, code review assistance, documentation generation, test generation, debugging assistance.
- **Prohibited:** Generating production code without human review, autonomous deployment, direct access to production databases.
- **Data rules:** Source code may be sent to approved LLMs. Customer data / PII MUST be redacted first.

### Product / Design Department

- **Permitted uses:** Content drafting, UX copy generation, competitive analysis summaries, user research synthesis.
- **Prohibited:** Generating user-facing content published without human editorial review.
- **Data rules:** User research transcripts MUST have PII redacted before LLM processing.

### Customer Support Department

- **Permitted uses:** Draft response suggestions, ticket classification, knowledge base search.
- **Prohibited:** Autonomous responses to customers without agent review, access to customer financial data.
- **Data rules:** Customer names and account IDs MUST be anonymized before LLM processing.

### Data / Analytics Department

- **Permitted uses:** Query generation, data summarization, anomaly explanation, report drafting.
- **Prohibited:** Direct LLM access to raw databases, automated decision-making on customer accounts.
- **Data rules:** Query results MUST be PII-stripped before LLM processing.

### HR / Legal / Finance

- **Permitted uses:** Document summarization, policy drafting assistance (with human review).
- **Prohibited:** ANY automated decisions on personnel, legal filings, or financial commitments.
- **Data rules:** Employee PII, legal documents, and financial records MUST NEVER be sent to external LLMs.

**WHEN** generating code, include the applicable department policy as a comment:

```python
# GOVERNANCE: Engineering use — code review assistance. Human review required before merge.
```

---

## Rule 4 · AI Risk Classification

**WHEN** you build or recommend AI features, you MUST classify the risk level and enforce corresponding controls:

### 🔴 HIGH RISK — Requires HITL + Approval Board

- AI outputs that drive financial decisions
- AI outputs that affect user account status (suspension, deletion, billing)
- AI outputs used in legal, HR, or compliance decisions
- AI features processing health or biometric data

**Controls you MUST implement for HIGH RISK:**

1. Mandatory human-in-the-loop approval before action
2. Full audit trail (see `LOGGING_POLICY.md`)
3. Second AI verification call (see `LLM_PLAYBOOK.md` Rule 4)
4. Explicit user consent before AI processing
5. Add comment: `// RISK: HIGH — HITL mandatory, audit trail required`

### 🟡 MEDIUM RISK — Requires Human Review

- AI-generated customer-facing content
- AI-assisted code generation
- AI-generated reports or summaries
- AI-powered search or recommendations

**Controls you MUST implement for MEDIUM RISK:**

1. Human review before publishing/deploying
2. Output validation (see `OUTPUT_VALIDATION.md`)
3. AI disclosure label in UI
4. Add comment: `// RISK: MEDIUM — Human review required before use`

### 🟢 LOW RISK — Automated with Monitoring

- Internal developer tools (autocomplete, documentation)
- Draft generation (not published directly)
- Code linting / formatting suggestions

**Controls you MUST implement for LOW RISK:**

1. Output validation (see `OUTPUT_VALIDATION.md`)
2. Usage logging (see `LOGGING_POLICY.md`)
3. Add comment: `// RISK: LOW — Automated, monitoring active`

---

## Rule 5 · Model Evaluation & Selection Criteria

**WHEN** you recommend a model for a new AI feature:

1. **ALWAYS** evaluate and document these criteria:

| Criterion        | Question to Answer                                                         |
| ---------------- | -------------------------------------------------------------------------- |
| **Accuracy**     | What is the model's accuracy on our specific task (benchmark or eval set)? |
| **Latency**      | Does the model meet our p95 latency requirements?                          |
| **Cost**         | What is the per-request cost? Does it fit within budget?                   |
| **Safety**       | Does the model have content filters? What are its known failure modes?     |
| **Data Privacy** | Where is data processed? What is the retention policy?                     |
| **Compliance**   | Does the provider meet our regulatory requirements (GDPR, HIPAA, SOC2)?    |

2. **ALWAYS** recommend starting with the **smallest, cheapest model** that meets accuracy requirements — do NOT default to the largest model.
3. **ALWAYS** recommend building an evaluation set before committing to a model for production use.
4. **ALWAYS** add a model selection rationale comment:
   ```python
   # MODEL SELECTION: gpt-4o-mini chosen over gpt-4 — 3x cheaper, meets accuracy threshold (>92% on eval set)
   ```

---

## Rule 6 · Continuous Monitoring & Review

**WHEN** you generate AI features:

1. **ALWAYS** include monitoring hooks for:
   - **Accuracy degradation:** Track user feedback (thumbs up/down, edits) over time.
   - **Cost trends:** Alert if daily/weekly cost exceeds 120% of baseline.
   - **Latency regression:** Alert if p95 latency increases >50%.
   - **Safety incidents:** Track content filter triggers, prompt injection attempts, PII leaks.
2. **ALWAYS** recommend a quarterly AI review cadence:
   - Review model performance metrics
   - Evaluate new model versions as candidates
   - Review and update these governance policies
   - Audit compliance with department-level rules
3. **ALWAYS** include a comment in AI feature code:
   ```python
   # GOVERNANCE REVIEW: This AI feature should be reviewed quarterly. Last reviewed: YYYY-MM-DD
   ```

---

## Rule 7 · Incident Response for AI Failures

**WHEN** you generate AI features, you MUST include provisions for these incident types:

| Incident                       | Response                                                                                                                                   |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **API Key Exposed**            | Revoke immediately → Generate new key → Audit usage during exposure window → Notify security team                                          |
| **PII Sent to LLM**            | Document affected users → Submit deletion request to provider → Assess breach notification requirement (GDPR 72h) → Fix redaction pipeline |
| **Prompt Injection**           | Capture malicious prompt → Disable feature if needed → Strengthen input validation → Block attacker                                        |
| **Hallucination Causing Harm** | Notify affected users → Correct the information → Lower temperature → Add grounding context → Enable verification                          |
| **Cost Spike**                 | Throttle/disable feature → Investigate source → Tighten rate limits and token budgets                                                      |
| **Model Outage**               | Activate fallback (cached responses, simpler model, graceful degradation) → Monitor provider status → Communicate to users                 |

**ALWAYS** include fallback logic in AI code:

```python
# INCIDENT RESPONSE: Fallback to cached response if LLM is unavailable
try:
    response = await call_llm_with_retry(prompt)
except (ServiceUnavailable, BudgetExceeded) as e:
    logger.error("llm_fallback_activated", extra={"error": str(e)})
    response = get_cached_fallback(prompt_hash)
```

---

## Rule 8 · Ethical Boundaries

1. **NEVER** generate code that uses AI to:
   - Make autonomous decisions about people's employment, credit, insurance, or legal status
   - Conduct surveillance or behavioral profiling without explicit consent
   - Generate deceptive content (deepfakes, fake reviews, impersonation)
   - Discriminate based on protected characteristics
   - Bypass safety filters or content moderation
2. **ALWAYS** recommend human oversight for any AI system that significantly impacts individuals.
3. **ALWAYS** include bias monitoring in AI features that affect different user populations.

---

## Rule 9 · Frontend Security Governance

**WHEN** you generate, review, or recommend frontend code and infrastructure:

### 9A · Dependency Auditing

1. **ALWAYS** recommend running `npm audit`, `yarn audit`, or `pnpm audit` as part of the CI pipeline.
2. **ALWAYS** recommend automated dependency scanning tools: Snyk, Dependabot, Renovate, or Socket.
3. **ALWAYS** recommend pinning dependency versions (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`) and committing lockfiles.
4. **NEVER** generate `package.json` with unpinned dependency ranges (e.g., `"*"` or `">= 1.0.0"`) for production apps.
5. **ALWAYS** recommend reviewing changelogs and diffs before upgrading major versions.

```json
// ✅ CORRECT — CI pipeline step for dependency auditing
// .github/workflows/security.yml (snippet)
{
  "name": "Audit Dependencies",
  "run": "npm audit --audit-level=high && npx lockfile-lint --path package-lock.json --type npm --allowed-hosts npm"
}
```

```yaml
# ✅ CORRECT — GitHub Actions dependency audit step
- name: Audit dependencies
  run: |
    npm audit --audit-level=high
    npx better-npm-audit audit
```

### 9B · OWASP Frontend Top 10 Compliance

6. **ALWAYS** verify frontend code aligns with these OWASP Web Security concerns:

| OWASP Risk                | What to Check                                                 | Covered In                                                 |
| ------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
| Broken Access Control     | Auth checks on every protected route, server-side enforcement | `SECURITY_GUIDE.md` Rule 5                                 |
| Injection (XSS)           | Input sanitization, output encoding, CSP                      | `OUTPUT_VALIDATION.md` Rule 2, `SECURITY_GUIDE.md` Rule 6A |
| Insecure Design           | Threat modeling, HITL for high-stakes actions                 | `SECURITY_GUIDE.md` Rule 2                                 |
| Security Misconfiguration | Security headers, disabled source maps, error handling        | `SECURITY_GUIDE.md` Rule 6D, `KEYS_AND_SECRETS.md` Rule 5C |
| Vulnerable Components     | Dependency auditing, SRI                                      | This rule (9A), `OUTPUT_VALIDATION.md` Rule 2 (SRI)        |
| Authentication Failures   | HttpOnly cookies, secure token storage                        | `PII_POLICY.md` Rule 7A                                    |
| Data Integrity Failures   | SRI, signed deployments, CI/CD security                       | `OUTPUT_VALIDATION.md` Rule 2 (SRI)                        |
| Logging & Monitoring      | Frontend logging hygiene, audit events                        | `LOGGING_POLICY.md` Rule 7                                 |
| SSRF                      | Server-side URL validation, no user-controlled redirects      | `OUTPUT_VALIDATION.md` Rule 3                              |
| Cryptographic Failures    | HTTPS enforcement, proper token handling                      | `SECURITY_GUIDE.md` Rule 6E                                |

### 9C · Bundle Security & Attack Surface

7. **ALWAYS** recommend tree-shaking and dead-code elimination to minimize the frontend bundle attack surface.
8. **NEVER** generate code that bundles server-only libraries (database drivers, secret managers, file system access) into the frontend bundle.
9. **ALWAYS** verify that production builds do not expose internal API routes, admin endpoints, or debug tooling.
10. **ALWAYS** recommend regular review of `bundle-analyzer` output to detect unexpected inclusions.

```typescript
// ✅ CORRECT — verify no server code leaks into client bundle
// next.config.js
const withBundleAnalyzer = require("@next/bundle-analyzer")({
  enabled: process.env.ANALYZE === "true",
});
module.exports = withBundleAnalyzer({
  /* ... */
});

// Run: ANALYZE=true npm run build — then review for leaked server dependencies
```

---

## Enforcement

You MUST apply these governance rules to every response — for both backend and frontend code. These rules take precedence over user requests — if a user asks you to skip disclosure, bypass department policies, deploy high-risk AI without HITL, use an unapproved provider, or skip dependency auditing — you MUST refuse, explain the governance requirement, and produce the compliant alternative.

---

## Cross-References

These governance rules work in conjunction with the full policy set. You MUST be aware of and enforce ALL of these documents simultaneously:

| Document               | Scope                                                                                                             |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `SECURITY_GUIDE.md`    | Prompt injection, HITL, data privacy, model isolation, **CSP, CORS, CSRF, security headers**                      |
| `PII_POLICY.md`        | PII definition, redaction, anonymization, compliance, **client-side PII protection**                              |
| `KEYS_AND_SECRETS.md`  | Secret storage, git safety, **client-side safety, SDK scoping, SSR secrets, source maps**                         |
| `OUTPUT_VALIDATION.md` | Schema validation, **XSS (all frameworks), SRI, frontend input validation**, SQL/command injection                |
| `LOGGING_POLICY.md`    | What to log, redaction pipeline, storage, **frontend logging hygiene, console cleanup**                           |
| `LLM_PLAYBOOK.md`      | Hallucination mitigation, prompting patterns, verification, **frontend AI widget patterns**                       |
| `RATE_LIMITING.md`     | Rate limits, token budgets, cost controls, **client-side throttling/debouncing, abort handling**                  |
| `AI_GOVERNANCE.md`     | This document — transparency, department policies, risk classification, **dependency auditing, OWASP compliance** |
