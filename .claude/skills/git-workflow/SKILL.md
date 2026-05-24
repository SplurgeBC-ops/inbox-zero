---
name: git-workflow
description: "Invoke before any git operations — branching, committing, merging, or creating pull requests. Contains project-specific branch naming, commit format, and merge strategy."
---

# Git Workflow

## Detected
- Default branch: main
- Contributors: 66
- Ana CLI: pipeline artifacts committed via `ana artifact save` with [slug] prefix. Build agent creates `{branchPrefix}{slug}` branches (read `branchPrefix` from `.ana/ana.json`, default `feature/`). Co-author from ana.json.

## Rules

### Branches & merges

- **Default branch is `main`.** Pipeline artifacts (scope, spec, contract) commit to `pre-prod` per `artifactBranch` in `.ana/ana.json`.
- **Merge strategy is "merge"** (no squash, no rebase). Preserve commit history on integration.
- **No consistent feature-branch prefix detected** across 66 contributors. The Ana CLI creates branches as `feature/{slug}` by default (configurable in `.ana/ana.json`).

### Commits

- **Commit format is NOT conventional commits.** The recent style is a sentence in the imperative + PR number, e.g. `Improve booking link metadata (#2725)`, `Fix follow-up notification completion (#2721)`. Match that style for manual commits.
- **Commit each logical change separately.** Don't batch unrelated changes.
- **Stage specific files** for each commit. Avoid `git add .` or `git add -A`.
- **Co-author trailer** is detected in history and managed automatically by the Ana CLI on artifact commits.

### Pre-commit and CI

- **Pre-commit hook runs lint only** (ultracite / Biome via `pnpm check`). It does NOT run typecheck or tests — CI does. Don't treat a green pre-commit as a green PR.
- **CI on PRs:** `pnpm check` → enum / server-action / client-redirect static checks → unit tests → package tests → integration tests. AI evals only fire when `apps/web/utils/ai/**`, `utils/llms/**`, or eval files change.
- **Docs-only PRs skip CI** — `**/*.md`, `**/*.mdx`, `docs/**` are excluded from the test workflow's paths filter. Split mixed PRs to avoid relying on the skip.
- **Never use `--no-verify`** to bypass hooks unless explicitly asked; fix the lint failure instead.

## Gotchas
*Not yet captured. Add as you discover them during development.*

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
