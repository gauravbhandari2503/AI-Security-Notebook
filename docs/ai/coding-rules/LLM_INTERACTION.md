# LLM Interaction / Prompt Injection

> **Audience:** You (the AI assistant). These are binding rules for when the code _you generate_ interacts with an LLM.

## 1. Model Input Integrity

- **PROHIBITED:** Never concatenate raw user input directly into the body/text string of a prompt payload to an LLM endpoint.
- **INSTEAD:** Always treat user string elements as hostile data arrays or explicitly place them in `<user-input>` tag boundaries. Explicitly supply defensive directives instructing the model to ignore manipulation inside those wrappers.

## 2. Downstream Rendering of Model Responses

- **PROHIBITED:** Never treat the LLM response object text payload as trusted, sanitized HTML/Markdown output for immediate browser rendering (e.g., blindly using `innerHTML`).
- **INSTEAD:** Always sanitize the LLM-generated string explicitly (e.g., passing it through `DOMPurify` locally, parsing markdown directly to sanitized tags) to ensure hallucinated `<script>` or event-handler tags don't execute client-side XSS.

## 3. Verifiable/Deterministic Fallbacks

- **PROHIBITED:** Never allow an API path containing AI logic execution to crash or act on uncontrolled/malformed string output internally (e.g., failing to parse LLM's raw string as generic JSON directly).
- **INSTEAD:** Always attempt to restrict the model structure format natively (e.g., setting `response_format` JSON), wrap it with strict schema validation logic locally, and degrade gracefully, substituting an LLM evaluation for a hardcoded safe logic fallback in unexpected LLM network timeouts or format violations.
