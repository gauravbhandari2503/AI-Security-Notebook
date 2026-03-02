# AI SDK Security Best Practices

This document outlines the security best practices for implementing the Vercel AI SDK in production applications. Following these guidelines will help protect your application from common security vulnerabilities and ensure secure handling of sensitive data.

## Table of Contents

1. [API Key Management](#api-key-management)
2. [Authentication & Authorization](#authentication--authorization)
3. [Rate Limiting](#rate-limiting)
4. [Input Validation & Sanitization](#input-validation--sanitization)
5. [Error Handling](#error-handling)
6. [Security Middleware & Guardrails](#security-middleware--guardrails)
7. [Data Privacy & Retention](#data-privacy--retention)
8. [Network Security](#network-security)
9. [Monitoring & Observability](#monitoring--observability)

---

## 1. API Key Management

### 1.1 Environment Variables

**Always store API keys in environment variables, never hardcode them in your source code.**

```bash
# .env or .env.local
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
AI_GATEWAY_API_KEY=xxxxxxxxx
```

```typescript
// Good ✅
const apiKey = process.env.OPENAI_API_KEY;

// Bad ❌
const apiKey = "sk-1234567890abcdef";
```

### 1.2 Provider Instance Configuration

Use environment variables when creating provider instances:

```typescript
import { createOpenAI } from "@ai-sdk/openai";

const openai = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

### 1.3 Bring Your Own Key (BYOK)

For multi-tenant applications, use request-scoped BYOK credentials with AI Gateway:

```typescript
import { generateText } from "ai";

const { text } = await generateText({
  model: "anthropic/claude-sonnet-4",
  prompt: "Your prompt here",
  providerOptions: {
    gateway: {
      byok: {
        anthropic: [
          {
            apiKey: userSpecificApiKey, // Request-scoped credential
          },
        ],
        bedrock: [
          {
            accessKeyId: "...",
            secretAccessKey: "...",
          },
        ],
      },
    },
  },
});
```

### 1.4 Key Rotation Best Practices

- Regularly rotate API keys (recommended: every 90 days)
- Maintain multiple API keys for zero-downtime rotation
- Revoke compromised keys immediately
- Use separate keys for development, staging, and production environments

---

## 2. Authentication & Authorization

### 2.1 Server-Side Validation

**Always validate authentication tokens on the server before processing requests:**

```typescript
'use server';

import { cookies } from 'next/headers';
import { createStreamableUI } from '@ai-sdk/rsc';
import { validateToken } from '../utils/auth';

export const getWeather = async () => {
  const token = cookies().get('token');

  if (!token || !validateToken(token)) {
    return {
      error: 'This action requires authentication',
    };
  }

  // Continue with protected logic
  const streamableDisplay = createStreamableUI(null);
  streamableDisplay.update(<Skeleton />);
  streamableDisplay.done(<Weather />);

  return {
    display: streamableDisplay.value,
  };
};
```

### 2.2 Custom Transport with Authorization Headers

Configure authentication headers for client-side requests:

```typescript
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: "/api/custom-chat",
    headers: {
      Authorization: `Bearer ${userToken}`,
      "X-API-Version": "2024-01",
    },
    credentials: "include",
  }),
});
```

### 2.3 User Tracking & Spend Attribution

Track API usage by user for cost control and security monitoring:

```typescript
const { text } = await generateText({
  model: "anthropic/claude-sonnet-4",
  prompt: "Explain quantum computing",
  providerOptions: {
    gateway: {
      user: "user-123", // Track usage by user
      tags: ["chat", "v2"], // Categorize requests
    },
  },
});
```

### 2.4 Tool Approval Workflow

Require user approval for sensitive tool calls:

```typescript
const result = await generateText({
  model: openai("gpt-4o"),
  tools: {
    deleteData: tool({
      description: "Delete user data",
      inputSchema: z.object({ userId: z.string() }),
      execute: async ({ userId }) => {
        // Sensitive operation
      },
      requireApproval: "always", // Always require approval
    }),
    safeTool: tool({
      // ...
      requireApproval: {
        never: {
          toolNames: ["safe_tool", "another_safe_tool"],
        },
      },
    }),
  },
  prompt: "Process user request",
});
```

---

## 3. Rate Limiting

### 3.1 Implement API Endpoint Rate Limiting

Use Upstash Ratelimit with Vercel KV to protect endpoints:

```typescript
import kv from "@vercel/kv";
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { Ratelimit } from "@upstash/ratelimit";
import { NextRequest } from "next/server";

export const maxDuration = 30;

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.fixedWindow(5, "30s"), // 5 requests per 30 seconds
});

export async function POST(req: NextRequest) {
  // Extract client identifier
  const ip = req.ip ?? "ip";
  const { success, remaining } = await ratelimit.limit(ip);

  if (!success) {
    return new Response("Rate limit exceeded", {
      status: 429,
      headers: {
        "X-RateLimit-Remaining": remaining.toString(),
      },
    });
  }

  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o"),
    messages,
  });

  return result.toUIMessageStreamResponse();
}
```

### 3.2 Rate Limit Strategies

Choose appropriate rate limiting strategies:

- **Fixed Window**: Simple, predictable (e.g., 100 requests per hour)
- **Sliding Window**: More accurate, prevents burst traffic
- **Token Bucket**: Allows bursts with long-term limits
- **User-Based**: Different limits for different user tiers

### 3.3 Handle Rate Limit Errors

Properly handle and communicate rate limit errors to clients:

```typescript
catch (error) {
  if (error instanceof Response && error.status === 429) {
    const retryAfter = error.headers.get('Retry-After');
    return {
      error: 'Rate limit exceeded',
      retryAfter: retryAfter ? parseInt(retryAfter) : 60,
    };
  }
}
```

---

## 4. Input Validation & Sanitization

### 4.1 Validate UI Messages

Always validate messages before processing:

```typescript
import { validateUIMessages, TypeValidationError } from "ai";

export async function POST(req: Request) {
  const { message, id } = await req.json();

  let validatedMessages;

  try {
    const previousMessages = await loadMessagesFromDB(id);
    validatedMessages = await validateUIMessages({
      messages: [...previousMessages, message],
      tools,
      metadataSchema,
    });
  } catch (error) {
    if (error instanceof TypeValidationError) {
      console.error("Message validation failed:", error);
      // Start with empty history or implement migration
      validatedMessages = [];
    } else {
      throw error;
    }
  }

  // Continue with validated messages...
}
```

### 4.2 Use Zod Schemas for Input Validation

Define strict schemas for all inputs:

```typescript
"use server";

import { z } from "zod";

const inputSchema = z.object({
  content: z.string().min(1).max(10000),
  userId: z.string().uuid(),
  type: z.enum(["text", "image", "file"]),
});

export const createResource = async (input: unknown) => {
  try {
    const validated = inputSchema.parse(input);

    // Process validated input
    const result = await processResource(validated);

    return { success: true, data: result };
  } catch (error) {
    if (error instanceof z.ZodError) {
      return {
        success: false,
        error: "Invalid input",
        details: error.errors,
      };
    }
    throw error;
  }
};
```

### 4.3 Safe Validation Pattern

Use safe validation to avoid throwing errors in critical paths:

```typescript
import { safeValidateUIMessages } from "ai";

const result = await safeValidateUIMessages({
  messages,
});

if (!result.success) {
  console.error(result.error.message);
  // Handle validation failure gracefully
  return { error: "Invalid message format" };
} else {
  const validatedMessages = result.data;
  // Continue processing
}
```

### 4.4 Tool Input Validation

Validate tool inputs to prevent injection attacks:

```typescript
import { tool } from "ai";
import { z } from "zod";

const safeSearchTool = tool({
  description: "Search for information",
  inputSchema: z.object({
    query: z
      .string()
      .min(1)
      .max(500)
      .regex(/^[a-zA-Z0-9\s\-_.]+$/, "Invalid characters in query"),
  }),
  execute: async ({ query }) => {
    // Safe to use validated query
    return await search(query);
  },
});
```

---

## 5. Error Handling

### 5.1 Graceful Error Handling for Non-Streaming APIs

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

try {
  const { text } = await generateText({
    model: openai("gpt-4o"),
    prompt: "Your prompt here",
  });
} catch (error) {
  // Log error for monitoring
  console.error("AI generation failed:", error);

  // Return user-friendly message
  return {
    error: "Unable to process your request. Please try again.",
  };
}
```

### 5.2 Error Handling for Streaming APIs

Use `onError` callback for streaming responses:

```typescript
import { streamText } from "ai";

const result = streamText({
  model: openai("gpt-4o"),
  prompt: "Write a story",
  onError({ error }) {
    // Log to error tracking service (e.g., Sentry)
    console.error("[AI SDK ERROR]:", error);

    // Optionally notify monitoring system
    notifyErrorTrackingService(error);
  },
});
```

### 5.3 Sanitize Error Messages for Clients

**Never expose internal error details to clients:**

```typescript
return result.toUIMessageStreamResponse({
  onError: (error) => {
    // Log full error internally
    logger.error("Stream error:", error);

    // Return sanitized error to client
    return {
      errorCode: "STREAM_ERROR",
      message: "An error occurred while processing your request",
      // ❌ DON'T: message: error.message (may leak sensitive info)
    };
  },
});
```

### 5.4 Handle Specific Error Types

```typescript
import { NoSuchToolError, InvalidToolInputError } from "ai";

try {
  const result = await generateText({
    // ...
  });
} catch (error) {
  if (NoSuchToolError.isInstance(error)) {
    console.error("Model attempted to call non-existent tool:", error.toolName);
    return { error: "Invalid operation requested" };
  } else if (InvalidToolInputError.isInstance(error)) {
    console.error("Invalid tool input:", error.toolInput);
    return { error: "Invalid request parameters" };
  } else {
    console.error("Unexpected error:", error);
    return { error: "An unexpected error occurred" };
  }
}
```

### 5.5 Client-Side Error Handling

```typescript
'use client';

import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, sendMessage, error, regenerate } = useChat();

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}: {m.parts.map(p => p.type === 'text' ? p.text : '').join('')}
        </div>
      ))}

      {error && (
        <div className="error-banner">
          <p>An error occurred while processing your request.</p>
          <button onClick={() => regenerate()}>
            Retry
          </button>
        </div>
      )}
    </div>
  );
}
```

---

## 6. Security Middleware & Guardrails

### 6.1 Superagent Security Tools

Integrate Superagent for comprehensive security checks:

```typescript
import { generateText, stepCountIs } from "ai";
import { guard, redact, verify } from "@superagent-ai/ai-sdk";
import { openai } from "@ai-sdk/openai";

const { text } = await generateText({
  model: openai("gpt-4o-mini"),
  prompt: userInput,
  tools: {
    guard: guard(), // Prompt injection detection
    redact: redact(), // PII redaction
    verify: verify(), // Claim verification
  },
  stopWhen: stepCountIs(3),
});
```

### 6.2 Custom Guardrails Middleware

Implement custom content filtering:

```typescript
import type { LanguageModelV3Middleware } from "@ai-sdk/provider";

export const contentFilterMiddleware: LanguageModelV3Middleware = {
  wrapGenerate: async ({ doGenerate }) => {
    const { text, ...rest } = await doGenerate();

    // Filter sensitive information
    const cleanedText = text
      ?.replace(/\b\d{3}-\d{2}-\d{4}\b/g, "<SSN-REDACTED>") // SSN
      ?.replace(/\b\d{16}\b/g, "<CARD-REDACTED>") // Credit card
      ?.replace(/badword/g, "<REDACTED>"); // Custom filters

    return { text: cleanedText, ...rest };
  },
};
```

### 6.3 Tool Input Examples for Better Security

Add examples to guide model behavior:

```typescript
import { wrapLanguageModel, addToolInputExamplesMiddleware } from "ai";

const secureModel = wrapLanguageModel({
  model: yourModel,
  middleware: addToolInputExamplesMiddleware({
    prefix: "Valid Input Examples:",
    format: (example, index) =>
      `${index + 1}. ${JSON.stringify(example.input)}`,
  }),
});
```

### 6.4 MCP Tool Approval

Control which tools require approval:

```typescript
const model = openai("gpt-4o");

const config = {
  requireApproval: {
    never: {
      toolNames: ["safe_tool", "read_only_tool"],
    },
  },
};

// Tools not in 'never' list will require approval
```

---

## 7. Data Privacy & Retention

### 7.1 Zero Data Retention

Restrict routing to providers with zero data retention policies:

```typescript
const { text } = await generateText({
  model: "anthropic/claude-sonnet-4",
  prompt: sensitivePrompt,
  providerOptions: {
    gateway: {
      zeroDataRetention: true, // Only use providers with zero retention
    },
  },
});
```

### 7.2 Disable Response Storage

For sensitive operations, disable response storage:

```typescript
import { openai } from "@ai-sdk/openai";
import { generateText } from "ai";

const result = await generateText({
  model: openai("gpt-5"),
  prompt: "Process sensitive data",
  providerOptions: {
    openai: {
      store: false, // Don't store in OpenAI's systems
      include: [], // Don't include extra metadata
    },
  },
});
```

### 7.3 Encrypted Content Handling

Request and handle encrypted content for reasoning chains:

```typescript
import {
  openai,
  type OpenaiResponsesReasoningProviderMetadata,
} from "@ai-sdk/openai";

const result = await generateText({
  model: openai("gpt-5"),
  prompt: "Sensitive analysis",
  providerOptions: {
    openai: {
      store: false,
      include: ["reasoning.encrypted_content"],
    },
  },
});

for (const part of result.content) {
  if (part.type === "reasoning") {
    const metadata = part.providerMetadata as
      | OpenaiResponsesReasoningProviderMetadata
      | undefined;

    const { itemId, reasoningEncryptedContent } = metadata?.openai ?? {};
    // Handle encrypted content securely
  }
}
```

### 7.4 PII Redaction

Automatically redact personally identifiable information:

```typescript
import { redact } from "@superagent-ai/ai-sdk";

const result = await generateText({
  model: openai("gpt-4o"),
  prompt: userMessage,
  tools: {
    redact: redact(), // Automatically detect and redact PII
  },
});
```

---

## 8. Network Security

### 8.1 Use HTTPS/TLS

Always use encrypted connections:

```typescript
const openai = createOpenAI({
  baseURL: "https://api.openai.com/v1", // ✅ HTTPS
  apiKey: process.env.OPENAI_API_KEY,
});

// ❌ Never use HTTP for production
// baseURL: 'http://api.example.com/v1'
```

### 8.2 Custom Headers for Security

Add security headers to requests:

```typescript
import { createOpenAI } from "@ai-sdk/openai";

const openai = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  headers: {
    "X-API-Version": "2024-01",
    "X-Request-ID": generateRequestId(),
    // Don't add sensitive data in headers
  },
});
```

### 8.3 Credential Management

Configure credentials appropriately:

```typescript
import { DefaultChatTransport } from "ai";

const transport = new DefaultChatTransport({
  api: "/api/chat",
  credentials: "include", // Include cookies for same-origin requests
  headers: {
    "Content-Type": "application/json",
  },
});
```

### 8.4 Service Account Authentication (Google Vertex)

Use service account credentials for server-side applications:

```typescript
import { createVertexAnthropic } from "@ai-sdk/google-vertex/anthropic";

const vertexAnthropic = createVertexAnthropic({
  googleAuthOptions: {
    credentials: {
      client_email: process.env.GOOGLE_CLIENT_EMAIL,
      private_key: process.env.GOOGLE_PRIVATE_KEY.replace(/\\n/g, "\n"),
    },
  },
});
```

---

## 9. Monitoring & Observability

### 9.1 Token Usage Tracking

Monitor token usage for cost control and abuse detection:

```typescript
import { streamObject } from "ai";
import { z } from "zod";

const result = streamObject({
  model: openai("gpt-4.1"),
  schema: z.object({
    recipe: z.object({
      name: z.string(),
      ingredients: z.array(z.string()),
    }),
  }),
  prompt: "Generate a recipe",
});

// Track usage
result.usage.then((usage) => {
  console.log("Prompt tokens:", usage.inputTokens);
  console.log("Completion tokens:", usage.outputTokens);
  console.log("Total tokens:", usage.totalTokens);

  // Send to monitoring system
  recordTokenUsage(usage);
});

// Consume the stream
for await (const partialObject of result.partialObjectStream) {
  // Process stream
}
```

### 9.2 Separate Usage Tracking

AI SDK 5.0+ provides separate usage and totalUsage:

```typescript
const result = await generateText({
  model: openai("gpt-4o"),
  prompt: "Multi-step task",
});

// Final step usage
console.log("Last step:", result.usage);

// Total across all steps
console.log("Total:", result.totalUsage);
```

### 9.3 Integrate Observability Tools

#### Helicone

```typescript
import { createOpenAI } from "@ai-sdk/openai";

const openai = createOpenAI({
  baseURL: "https://oai.helicone.ai/v1",
  headers: {
    "Helicone-Auth": `Bearer ${process.env.HELICONE_API_KEY}`,
  },
});
```

#### LangWatch

```typescript
import { LangWatchSpanKind, withLangWatchTrace } from "langwatch";

export async function POST(req: Request) {
  return withLangWatchTrace(
    async (trace) => {
      const result = await generateText({
        model: openai("gpt-4o"),
        experimental_telemetry: {
          isEnabled: true,
          recordInputs: true,
          recordOutputs: true,
        },
      });

      return result;
    },
    {
      spanKind: LangWatchSpanKind.LLM,
    },
  );
}
```

#### Laminar

```bash
# .env
LMNR_PROJECT_API_KEY=your-laminar-api-key
```

### 9.4 Custom Request Tagging

Tag requests for filtering and analysis:

```typescript
const { text } = await generateText({
  model: "anthropic/claude-sonnet-4",
  prompt: userInput,
  providerOptions: {
    gateway: {
      user: userId,
      tags: ["production", "chat-v2", "premium-user"],
    },
  },
});
```

---

## Security Checklist

Use this checklist to ensure your AI SDK implementation follows security best practices:

### API Keys & Credentials

- [ ] API keys stored in environment variables
- [ ] No API keys in source code or version control
- [ ] Separate keys for dev/staging/production
- [ ] Regular key rotation implemented
- [ ] BYOK configured for multi-tenant apps

### Authentication & Authorization

- [ ] Server-side authentication validation
- [ ] User authorization checks before sensitive operations
- [ ] Token-based authentication configured
- [ ] Tool approval workflow for sensitive operations
- [ ] User tracking for spend attribution

### Rate Limiting

- [ ] API endpoints protected with rate limiting
- [ ] Per-user rate limits implemented
- [ ] Rate limit errors handled gracefully
- [ ] Monitoring for rate limit violations

### Input Validation

- [ ] All inputs validated with Zod schemas
- [ ] UI messages validated before processing
- [ ] Tool inputs sanitized
- [ ] Safe validation patterns used

### Error Handling

- [ ] Try-catch blocks for non-streaming APIs
- [ ] onError callbacks for streaming APIs
- [ ] Error messages sanitized for clients
- [ ] Specific error types handled appropriately
- [ ] Errors logged to monitoring systems

### Security Middleware

- [ ] Guardrails implemented (prompt injection, PII)
- [ ] Content filtering middleware configured
- [ ] Tool input examples provided
- [ ] Security tools integrated (Superagent, etc.)

### Data Privacy

- [ ] Zero data retention enabled for sensitive operations
- [ ] Response storage disabled where appropriate
- [ ] PII redaction implemented
- [ ] Encrypted content handling configured

### Network Security

- [ ] HTTPS/TLS used for all connections
- [ ] Security headers configured
- [ ] Credentials management proper
- [ ] Service account authentication for server apps

### Monitoring

- [ ] Token usage tracking implemented
- [ ] Observability tools integrated
- [ ] Request tagging configured
- [ ] Usage alerts set up
- [ ] Error tracking integrated

---

## Additional Resources

- [AI SDK Documentation](https://ai-sdk.dev/)
- [AI SDK Security Guide](https://ai-sdk.dev/docs/advanced/security)
- [Rate Limiting Guide](https://ai-sdk.dev/docs/advanced/rate-limiting)
- [Error Handling Guide](https://ai-sdk.dev/docs/ai-sdk-core/error-handling)
- [Authentication Guide](https://ai-sdk.dev/docs/ai-sdk-rsc/authentication)
- [Middleware Guide](https://ai-sdk.dev/docs/ai-sdk-core/middleware)

---

## License

This document is provided as-is for educational purposes. Always refer to the official AI SDK documentation for the most up-to-date security recommendations.
