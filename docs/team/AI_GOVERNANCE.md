# AI Governance Policy (Team Summary)

**Sources:**

- [What is AI Governance? | IBM](https://www.ibm.com/think/topics/ai-governance)
- Internal `docs/ai/AI_GOVERNANCE.md` (AI Instruction Set)

## What is AI Governance?

AI governance refers to the processes, standards, and guardrails that ensure AI systems are safe, ethical, and respect human rights.

**Core Principles:**

1. **Empathy:** Anticipating the societal impact of AI.
2. **Bias Control:** Rigorously examining data to prevent embedded real-world biases.
3. **Transparency:** Maintaining openness in how algorithms operate.
4. **Accountability:** Maintaining responsibility for AI's impacts.

---

## 🏢 Company AI Policies & Rules

To uphold these principles, the following rules apply globally to all AI features developed at this company.

### 1. Approved Providers & Minimization

- **Only use approved providers** with Enterprise SLAs, DPAs, and zero-retention policies.
- Recommended (in order of preference for sensitive workloads): **Azure OpenAI**, **AWS Bedrock**, Self-hosted (Ollama/vLLM), OpenAI Enterprise (ZDR), Anthropic (Claude).
- **Never** use providers that train on customer data without an opt-out.
- Always default to the **smallest, cheapest model** that meets accuracy needs.

### 2. Department-Level Usage

Different departments have distinct boundaries for AI processing:

- **Engineering:** May use AI for code/docs/tests. _Prohibited:_ Autonomous production deployments.
- **Product/Design:** May use AI for copy/UX. _Prohibited:_ Publishing user-facing content without human review.
- **Support:** May use AI for drafts. _Prohibited:_ Autonomous responses without agent review.
- **HR/Legal/Finance:** _Prohibited:_ ANY automated decisions on personnel, legal, or finance. Raw records must NEVER go to external LLMs.

### 3. AI Risk Classification & Required Controls

Every AI feature must be classified by risk:

- 🔴 **HIGH RISK** (Financial, account suspension, health data):
  - **Requires:** Mandatory Human-in-the-loop (HITL), full audit trail, double verification, explicit user consent.
- 🟡 **MEDIUM RISK** (Customer-facing content, AI-generated reports):
  - **Requires:** Human review before publish, output validation, clear UI disclosure.
- 🟢 **LOW RISK** (Internal tools, autocomplete):
  - **Requires:** Automated log monitoring, output validation.

### 4. Transparency & UX Disclosures

- Any AI-generated UI component or output must visibly disclose the use of AI (e.g., `🤖 AI Generated`).
- All code must include defensive comments indicating model choice and policy adherence.

### 5. Frontend & Supply Chain Security

Governance extends the full stack:

- **Audits:** CI runs MUST audit dependencies (`npm audit` / `yarn audit`).
- **Dependencies:** All versions must be pinned.
- **Bundles:** Ensure server-only secrets aren't bundled into the frontend. Use Bundle Analyzer regularly.
- **OWASP:** Frontend code must adhere strictly to modern web security practices (CSP, CORS, CSRF, Secure Cookies).

### 6. Incident Response & Monitoring

We monitor metrics for accuracy degradation, cost spikes, and latency regressions.

- **PII Leak:** Revoke, assess GDPR notification need, fix redaction pipeline.
- **Cost Spike:** Auto-throttle, investigate source, reset budgets.
- **Prompt Injection:** Capture malicious prompt, strengthen validators.

---

**Summary Takeaway:** The principles of empathy, transparency, and accountability are enforced through strict data residency, explicit Human-in-the-Loop gates for high-stakes actions, and layered output validation across our entire tech stack.
