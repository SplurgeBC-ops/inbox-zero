---
name: ai-patterns
description: "Invoke when building features that call LLM APIs, handling AI responses, managing prompts, or integrating AI SDKs. Contains error handling, security, prompt management, and observability patterns."
---

# AI Patterns

## Detected
- AI SDK: Vercel AI
- Also detected: OpenAI, Vercel AI (Anthropic), Vercel AI (OpenAI), Vercel AI (Google), Vercel AI (Google Vertex), Vercel AI (Bedrock), Vercel AI (Azure), Vercel AI (Groq), Vercel AI (Perplexity), Vercel AI (Gateway), Vercel AI (MCP), Vercel AI (OpenAI Compatible), Vercel AI (OpenRouter)

## Rules

### Universal

- **All LLM calls route through `apps/web/utils/llms/index.ts`.** Never import directly from `ai` / `@ai-sdk/*` in feature code. The wrapper handles PostHog tracing, retry, `jsonrepair` for malformed JSON, error classification (insufficient credits, content filter, quota, deactivated key), DLP sensitive-content redaction, prompt hardening, and trial usage guarding. Bypassing it loses all of these in one step.
- **Never lock to a single provider.** Use the `getModel()` / `SelectModel` abstraction from `utils/llms/model.ts` so the choice stays a config change. If a change makes provider switching require a code rewrite, it's a regression. (See design principle: "Never lock to a single AI provider.")
- **Never interpolate raw user input into system prompts.** User content goes in user messages with clear role boundaries. System instructions stay immutable. The wrapper applies `applyPromptHardeningToMessages` / `applyPromptHardeningToSystem` — don't rebuild what it already does.
- **Treat all LLM output as untrusted.** Validate with Zod via `generateObject` before using in DB queries, HTML rendering, or business logic. Never regex-parse free-text LLM responses for application data.
- **No keyword/regex band-aids for model behavior.** When a model produces bad wording, assert the semantic failure mode with a judge/eval criterion — not an English-specific text check. Same rule at runtime: never gate context injection or tool behavior on ad hoc user-text matching; use structured state, metadata, or explicit events. (See design principle: "No keyword or regex band-aids for model behavior.")

### Project-specific

- **Prompts live inline in feature files** under `apps/web/utils/ai/<feature>/*.ts`, co-located with the logic they drive — there is no central `prompts/` directory. User-defined rules are the DB exception (they're user data, not engineering-managed prompts). If a centralized prompt file ever appears, that's the new convention; treat it as authoritative.
- **AI feature code lives in `apps/web/utils/ai/<feature>/`.** Transport/provider concerns live in `apps/web/utils/llms/`. Don't merge them — `ai/` is product logic, `llms/` is infrastructure.
- **Every new AI feature ships with at least one eval** in `apps/web/__tests__/eval/` that exercises a real failure mode (wrong-rule match, hallucinated content, sensitive data leak), not just the success path. Run via `pnpm --filter inbox-zero-ai test-ai` from repo root. Prompt and model changes need an eval that would have caught the regression. (See design principle: "Test AI decision paths, not just the happy path.")
- **PostHog usage tracking happens via `saveAiUsage` inside the wrapper** — don't re-emit usage events from feature code.

## Gotchas
- Use `generateObject()` for structured output and `streamText()` for streaming responses. Don't use `generateText()` with manual JSON parsing.
- OpenAI SDK supports `maxRetries` in the client constructor. Use `response_format: { type: 'json_object' }` for structured output instead of parsing free text.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
