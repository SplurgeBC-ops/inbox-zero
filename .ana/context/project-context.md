<!-- SCAFFOLD - Setup will fill this file -->

# Project Context

## What This Project Does

**Detected:** pnpm monorepo, with authentication (Better Auth), database (Prisma → postgresql, 63 models), and AI integration (Vercel AI). 1732 source files, 550 test files.
**Detected issues:** 1 warning — run `ana scan` for details

Inbox Zero - your 24/7 AI email assistant Organizes your inbox, pre-drafts replies, manages your calendar, and organizes attachments. Chat with it from Slack or Telegram to manage your inbox on the go. Open source alternative to Fyxer, but more customizable and secure.

*What does this product do? Who uses it? What problem does it solve?*

## Architecture

**Detected:** pnpm · 13 packages (inbox-zero-ai, @inboxzero/image-proxy-worker, @inboxzero/image-proxy-aws, @inboxzero/worker, @inbox-zero/api)
**Detected surfaces:** web (apps/web, TypeScript, Next.js), api (packages/api, TypeScript), cli (packages/cli, TypeScript)
**Detected:** 7 directories mapped: .github/, .vscode/, apps/, docker/, docs/, packages/, scripts/
**Detected deployment:** Cloudflare Workers, GitHub Actions

*How is the codebase organized and why? What are the layer boundaries?*

## Where to Make Changes

*Common tasks and where to find the relevant code. What files are entry points for what kind of work?*

## Key Decisions

*Technology choices and patterns that look wrong but are intentional. What was tried and rejected?*

## Key Files

- Database schema: `apps/web/prisma/schema.prisma`
- Deployment config: `apps/image-proxy/wrangler.jsonc`
- CI pipeline: `.github/workflows/ai-evals.yml`, `.github/workflows/api-release.yml`, `.github/workflows/build-changelog.yml` + 9 more

*Add: database client location, auth config, AI wrapper, shared types, test helpers.*

## What Looks Wrong But Is Intentional

*Patterns that seem wrong for this stack but are deliberate. Anti-intuitive decisions with rationale.*

## Active Constraints

*Current priorities. Areas under active refactoring. Features not to touch right now.*

## Domain Vocabulary

*Terms with project-specific meaning. E.g., "workspace" = pnpm workspace package, not Slack workspace.*
