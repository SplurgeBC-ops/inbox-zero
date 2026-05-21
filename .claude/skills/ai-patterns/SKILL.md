---
name: ai-patterns
description: "Invoke when building features that call LLM APIs, handling AI responses, managing prompts, or integrating AI SDKs. Contains error handling, security, prompt management, and observability patterns."
---

# AI Patterns

## Detected
- AI SDK: Vercel AI
- Also detected: OpenAI, Vercel AI (Anthropic), Vercel AI (OpenAI), Vercel AI (Google), Vercel AI (Google Vertex), Vercel AI (Bedrock), Vercel AI (Azure), Vercel AI (Groq), Vercel AI (Perplexity), Vercel AI (Gateway), Vercel AI (MCP), Vercel AI (OpenAI Compatible), Vercel AI (OpenRouter)

## Rules
- All LLM calls through a centralized client wrapper. Configure retry, timeout, and error handling once — not per-call.
- Never interpolate raw user input into system prompts. User content goes in user messages with clear role boundaries. System instructions stay immutable.
- Treat all LLM output as untrusted. Validate and sanitize before using in database queries, HTML rendering, or business logic.
- Handle LLM errors by type: retry rate limits with backoff, truncate input for context overflow, log content filter triggers, fail gracefully for API outages.
- Use structured output (JSON mode, tool_use) for data extraction. Never regex-parse free-text LLM responses for application data.
- Centralize prompt templates — don't scatter prompt strings across business logic. Prompts should be versionable, testable, and reviewable independently.
- Log model, token count, and latency per LLM call. You can't optimize cost or debug quality without knowing what each request consumed.

## Gotchas
- Use `generateObject()` for structured output and `streamText()` for streaming responses. Don't use `generateText()` with manual JSON parsing.
- OpenAI SDK supports `maxRetries` in the client constructor. Use `response_format: { type: 'json_object' }` for structured output instead of parsing free text.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
