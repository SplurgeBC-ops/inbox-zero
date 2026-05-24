---
name: deployment
description: "Invoke when working on deployment configuration, CI/CD pipelines, environment variables, or release processes. Contains project-specific deploy platform conventions."
---

# Deployment

## Detected
- Platform: Cloudflare Workers (image proxy only); hosted web app deploy not in repo
- Config: `apps/image-proxy/wrangler.jsonc`
- CI: GitHub Actions, 12 workflows

## Rules

### Pipelines

- **`test.yml`** runs on every push to `main` and every PR: `pnpm check` (Biome/ultracite lint) → enum import check → server action export check → client redirect check → unit tests (`pnpm -F inbox-zero-ai test`) → package tests → integration tests. Docs-only changes (`docs/**`, `**/*.md`, `**/*.mdx`) are skipped. Pre-commit hook only runs lint — CI catches typecheck and tests.
- **`ai-evals.yml`** runs only when `apps/web/utils/ai/**`, `apps/web/utils/llms/**`, or AI test files change, gated by repository variable `AI_EVALS_ENABLED=true`. Uses OpenRouter with `anthropic/claude-sonnet-4.5` as default. Eval reports uploaded as workflow artifact.
- **`e2e-flows.yml`** and **`smoke.yml`** cover end-to-end and smoke checks.
- **`build_and_publish_docker.yml`** runs on push to `main` and `v*` tags. Builds via Depot, pushes to `ghcr.io/elie222/inbox-zero` and `elie222/inbox-zero` on Docker Hub. Tagged with short SHA; `latest` tracks main; formal releases use version tags (e.g., `v2.26.0`).
- **`deploy-image-proxy.yml`** auto-deploys the Cloudflare Worker on push to `main` when `apps/image-proxy/**`, `packages/image-proxy/**`, or workflow/lock files change. Requires `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` secrets and `CLOUDFLARE_IMAGE_PROXY_DOMAIN` variable.

### Release process

- **`@inbox-zero/cli`** publishes on `v*` tag (`cli-release.yml`): cross-compiles binaries for darwin-arm64, darwin-x64, linux-x64 via `bun build --compile`, publishes to npm with provenance, uploads tarballs to GitHub Release, opens a PR to update the Homebrew formula (`Formula/inbox-zero.rb`). `workflow_dispatch` available for manual runs.
- **`@inbox-zero/api`** publishes on `api-v*` tag (`api-release.yml`) — note the different tag prefix from CLI. Same npm provenance pattern.
- Both publish jobs guard against republishing the same version: if the version already exists on npm, the job exits cleanly.

### Hosted web app

- ⚠ **ASSUMPTION (not verified in repo):** the hosted getinboxzero.com auto-deploys from `main` (likely via Vercel — README has Vercel OSS badge, no explicit hosted deploy workflow exists). `pre-prod` is treated as the artifact/staging gate. Any spec that depends on the exact production-deploy mechanism must confirm with Ahmed before shipping.

### Adding env vars

- New env var must be added to all three of: `.env.example`, `apps/web/env.ts`, and `turbo.json`. Prefix client-side vars with `NEXT_PUBLIC_`.
- Local self-host path: `npx @inbox-zero/cli setup` (wizard) → `npx @inbox-zero/cli start` (containers).
- Local dev path: `docker compose -f docker-compose.dev.yml up -d` (Postgres + Redis) → `pnpm install` → `npm run setup` → `pnpm prisma migrate dev` (from `apps/web/`) → `pnpm dev`.

### Workspace packages and Docker

- When adding a new workspace package, add its `package.json` COPY line to **both** `docker/Dockerfile.prod` and `docker/Dockerfile.local`. Skipping this breaks the Docker build silently for self-hosters.

## Gotchas
*Not yet captured. Add as you discover them during development.*

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
