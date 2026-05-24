<!-- SCAFFOLD - Setup will fill this file -->

# Project Context

## What This Project Does

**Detected:** pnpm monorepo, with authentication (Better Auth), database (Prisma → postgresql, 63 models), and AI integration (Vercel AI). 1732 source files, 550 test files.
**Detected issues:** 1 warning — run `ana scan` for details

Inbox Zero is a 24/7 AI email assistant for people drowning in inbox volume. It handles the routine 80% of inbox work: categorizing mail, drafting replies in your tone, blocking cold emailers, bulk-unsubscribing, surfacing meeting briefs, and triaging from Slack or Telegram. Open source alternative to Fyxer, with a hosted version at getinboxzero.com and self-hosting via the CLI.

**Target users:** startup founders, VCs, and sales teams. Before Inbox Zero they were stuck manually triaging, or paying for speed without intelligence.

**Core product tensions:**

- **Provider flexibility is a constraint, not a convenience.** 11 AI providers wired through the Vercel AI SDK is deliberate. Claude for nuanced replies, GPT for fast categorization, Google for multilingual, etc. Never hardcode to one provider — swapping models per task must stay a config change.
- **Speed of iteration vs. trust.** Users expect weekly improvements, but this is people's actual email. A bug that deletes mail or leaks data is catastrophic. Test AI decision paths, not just the happy path.

## Architecture

**Detected:** pnpm · 13 packages (inbox-zero-ai, @inboxzero/image-proxy-worker, @inboxzero/image-proxy-aws, @inboxzero/worker, @inbox-zero/api)
**Detected surfaces:** web (apps/web, TypeScript, Next.js), api (packages/api, TypeScript), cli (packages/cli, TypeScript)
**Detected:** 7 directories mapped: .github/, .vscode/, apps/, docker/, docs/, packages/, scripts/
**Detected deployment:** Cloudflare Workers (image-proxy), GitHub Actions

Turborepo monorepo with `apps/web` (inbox-zero-ai) as the primary Next.js App Router application — it serves the UI, hosts API routes, runs server actions, and contains all the AI/email logic. Supporting deployables: `apps/worker` (background jobs), `apps/image-proxy`/`image-proxy-aws` (Cloudflare/AWS image proxy). Shared packages: `@inbox-zero/cli` (self-hosting wizard), `@inbox-zero/api` (public API wrapper), `@inboxzero/scheduling`, `@inboxzero/tinybird` + `tinybird-ai-analytics` (real-time analytics), `@inboxzero/resend` + `loops` (transactional + marketing email).

**Provider abstraction is load-bearing.** Gmail and Outlook are normalized behind a single `EmailProvider` interface (`apps/web/utils/email/types.ts`). Use `EmailProvider` everywhere — only fall back to `isGoogleProvider` / `isMicrosoftProvider` (in `utils/email/provider-types.ts`) at true integration boundaries.

**AI layer** is split: `apps/web/utils/llms/` owns provider selection, retry, fallback, usage tracking, and sensitive-content policy across 11 Vercel AI SDK providers; `apps/web/utils/ai/` owns the rule engine (`choose-rule/`), reply drafting (`reply/`), cold email detection, categorization, and other AI features. Always go through `utils/llms/index.ts` for model calls — it wraps PostHog tracing, retry, repair, and DLP redaction.

**Auth:** Better Auth (`apps/web/utils/auth.ts`) with Prisma adapter. Configured providers: Google, Microsoft, Apple, plus SSO/SCIM plugins for enterprise and an `expo` plugin for mobile. Login providers are gated by env config so disabled providers can't be invoked.

**Validation:** Zod everywhere. Server actions co-locate `.validation.ts` files; types are inferred via `z.infer<typeof schema>` rather than duplicated as interfaces.

**Data:** Prisma → PostgreSQL, 63 models in `apps/web/prisma/schema.prisma`. Tinybird handles analytics-scale data separately.

**Background work:** BullMQ + Upstash QStash + Vercel Queue. Worker app in `apps/worker`.

## Where to Make Changes

Tasks → location:

- **New API route:** `apps/web/app/api/<resource>/route.ts`. One resource per file. Wrap with `withError` (public), `withAuth` (user-level), or `withEmailAccount` (email-account-level). Mutations should be server actions, not POST routes — exception: mobile-native integrations that need a stable HTTP contract.
- **New mutation:** `apps/web/utils/actions/<feature>.ts` using `next-safe-action`. Co-locate `<feature>.validation.ts` (Zod) and `<feature>.test.ts`.
- **New AI feature / rule type:** `apps/web/utils/ai/<feature>/`. Model calls go through `apps/web/utils/llms/`.
- **Gmail/Outlook integration:** prefer `EmailProvider` from `apps/web/utils/email/`. Drop into `utils/gmail/` or `utils/microsoft/` only for provider-specific behavior.
- **Page / route:** `apps/web/app/(app)/<feature>/page.tsx`. Co-locate page components next to `page.tsx` — no nested `components/` subfolders inside route directories. Shared components go in `apps/web/components/`.
- **Schema change:** edit `apps/web/prisma/schema.prisma`, then `pnpm prisma migrate dev` from `apps/web/`.
- **New env var:** add to `.env.example`, `apps/web/env.ts`, and `turbo.json`. Prefix client-side with `NEXT_PUBLIC_`.
- **New workspace package:** also add the `package.json` COPY line to both `docker/Dockerfile.prod` and `docker/Dockerfile.local`.

**Active hotspots (last 2 weeks, by churn):** follow-up reminders (`utils/follow-up/`, `app/api/follow-up-reminders/`), rule notifications (`utils/messaging/rule-notifications.ts`), draft reply (`utils/ai/reply/draft-reply.ts`), Slack notifications (`__tests__/integration/slack-notifications.test.ts`). If your task touches these areas, expect concurrent change.

## Key Decisions

- **Database rules vs. prompt file as source of truth:** Users write a prompt file that gets converted to individual DB rules; the LLM only sees the DB rules, not the prompt file. Two-way sync between them is messy. Kept because: it lets us track per-rule usage, lets users define static actions precisely without LLM interference, and most rule decisions are condition-matching, not generation. Likely would be redesigned if built today (see `ARCHITECTURE.md` "AI Personal Assistant").
- **Reply tracking lives inside the AI assistant as a special rule type**, not as a separate feature. The cost: hard to push global prompt updates for reply tracking (unlike the cold email blocker, which has one global prompt). The benefit: reuses the existing assistant infrastructure.
- **Cold email blocker is intentionally NOT wired to the AI personal assistant.** It runs on its own pipeline, only for senders the user has never emailed before.
- **11 AI providers via Vercel AI SDK** (Anthropic, OpenAI, Google, Vertex, Bedrock, Azure, Groq, Perplexity, OpenRouter, Ollama, MCP). Different models for different tasks. Provider switching must stay a config change, never a rewrite.
- **Server actions for mutations, NOT POST API routes** (exception: mobile integrations needing stable HTTP).
- **Never use dynamic Prisma transactions** (`prisma.$transaction(async (tx) => ...)`).
- **Better Auth over NextAuth** for SSO/SCIM/expo plugin coverage.

## What Looks Wrong But Is Intentional

- **`apps/web/utils/llms/` and `apps/web/utils/ai/` are separate layers** — `llms/` is provider/transport infrastructure; `ai/` is feature logic. Don't merge them.
- **108 of 168 API routes have no top-of-file validation imports** (scan finding). Many use wrapper- or middleware-based validation that isn't detected by static import checks. Verify per route before assuming a route is unvalidated.
- **Catches that look empty** are usually intentional (`codePatterns.emptyCatches` shows 0 truly empty, 1 commented). When a swallow is needed, comment why.

## Key Files

- Database schema: `apps/web/prisma/schema.prisma` (63 models)
- Auth config: `apps/web/utils/auth.ts`
- AI provider abstraction: `apps/web/utils/llms/index.ts`, `apps/web/utils/llms/model.ts`, `apps/web/utils/llms/config.ts`
- AI rule engine: `apps/web/utils/ai/choose-rule/` (read `NOTES.md` in that directory)
- Reply drafting: `apps/web/utils/ai/reply/draft-reply.ts`
- Email provider abstraction: `apps/web/utils/email/types.ts`, `apps/web/utils/email/provider-types.ts`
- API middleware: `apps/web/utils/api-middleware.ts`, `apps/web/utils/api-auth.ts`
- Env config: `apps/web/env.ts` (client uses `NEXT_PUBLIC_` prefix)
- Architecture notes: `ARCHITECTURE.md` (root) — includes design rationale for the AI assistant
- Deployment config: `apps/image-proxy/wrangler.jsonc`
- CI pipeline: `.github/workflows/test.yml`, `ai-evals.yml`, `e2e-flows.yml`, `build_and_publish_docker.yml` (+ 8 more)

## Active Constraints

- Recent activity heavily focused on follow-up reminders, rule notifications, and draft reply grounding — expect concurrent edits in these areas.

*Add your current priorities, active migrations, or areas not to touch. Expand any time.*

## Domain Vocabulary

- **Email account** — a connected Gmail/Outlook account; not the user account itself. A user can have multiple email accounts. Most authorization checks scope to email-account level (`withEmailAccount` middleware).
- **Rule** — a user-defined automation triggered by incoming email. Stored as individual DB records, surfaced to users as a prompt file (with two-way sync).
- **Action** — what a rule does when matched: archive, label, reply, draft, etc. Actions are mostly static (no LLM at execution time) unless using templates.
- **Cold email** — incoming mail from a sender the user has never emailed. Runs on its own pipeline, separate from the AI assistant.
- **Reply Zero** — the user-facing name for reply tracking (emails awaiting responses, emails to reply to). Implemented as a special rule type inside the AI assistant.
- **Follow-up** — automated reminder when an outgoing email goes without a response. Distinct from Reply Zero.
- **Knowledge** — the user's accumulated context the AI uses to ground replies (writing style, prior decisions).
- **Digest** — a periodic summary email of low-priority mail batched together.
- **Group** — a saved set of senders/conditions reusable across rules.
- **Workspace package** — a pnpm workspace package. Not a Slack workspace.
- **Provider** — overloaded: an `EmailProvider` (Gmail/Outlook), an AI provider (Anthropic/OpenAI/...), a `MessagingProvider` (Slack/Telegram/Teams), or an auth provider (Google/Microsoft/Apple). Disambiguate from context.
- **Emulator** — local fake of Google or Microsoft for dev/E2E (`docker compose ... --profile google-emulator up -d`).
