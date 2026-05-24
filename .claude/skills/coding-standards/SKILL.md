---
name: coding-standards
description: "Invoke when implementing features, writing code, or reviewing code quality. Contains project-specific naming conventions, error handling patterns, import style, and deviations from standard practices."
---

# Coding Standards

## Detected
- Language: TypeScript with Next.js (1732 source files)
- Functions: PascalCase (53%, 1529 sampled)
- Classes: PascalCase (29%)
- Files: PascalCase (80%, 750 sampled)
- Imports: path aliases (@/) (97%)
- Indentation: spaces, 2 wide
- Error handling: exceptions (nextjs)
- Data fetching: swr
- State management: jotai
- Form handling: react-hook-form
- UI: shadcn/ui (Tailwind)

### Library Rules
- Use `import type` for type-only imports, separate from value imports. Prevents runtime imports of pure types.
- Default to Server Components. Only add `"use client"` when the component needs browser APIs, event handlers, or useState/useEffect. Data fetching belongs in Server Components — no useEffect waterfalls.

## Rules

### Imports

- **Use `@/` path aliases** (96.5% of imports are absolute via `@/`). Relative imports only for siblings or one level up. Never `../../../`.
- **Imports go at the top.** No mid-file dynamic imports.
- **No barrel files. No re-export patterns.** Import from the original source. Importing through a barrel hides dependencies and breaks tree-shaking.
- **Lodash:** import specific functions — `import groupBy from "lodash/groupBy"`, not `import { groupBy } from "lodash"`.

### Types

- **Infer types from Zod schemas** via `z.infer<typeof schema>` rather than duplicating as separate interfaces.
- **Don't export types/interfaces only used within the same file.** Keep file-local types unexported.
- **Avoid `any`** — use `unknown` and narrow with type guards. `any` is acceptable only for untyped third-party boundaries.

### Files & exports

- **Prefer named exports.** Default exports are rare (5 of 30 sampled files); reserve them for Next.js pages, layouts, and route handlers that require them.
- **No `null` for optional values** — use `?:` / `undefined`. Scan shows 46 optional vs 3 `| null` (strong preference). Only use `null` when the schema actually distinguishes "missing" from "explicitly null."
- **File naming is mixed by purpose, not arbitrary.** React components are `PascalCase.tsx`. Utility files, hooks, and route handlers are `kebab-case.ts`. Match the existing convention in the directory you're editing.
- **Helper functions go at the bottom of files**, not the top. Read top-down: public surface first, internals after.
- **Co-locate unit tests** next to source (`utils/example.test.ts`). Integration, E2E, and AI tests go in `__tests__/`.

### Logic & style

- **Inline and co-locate logic at the call site by default.** Don't extract helpers that just rename and forward parameters — that's a layer without meaning.
- **Avoid premature abstraction.** Small duplicated expressions are fine. Extract only when the helper names a meaningful domain concept, makes surrounding code clearer, or keeps correctness-sensitive rules in sync.
- **Avoid `useEffect` for mirroring fetched props/data into local state.** Prefer derived values or explicit edit state.
- **Avoid large/nested ternaries.** Prefer straightforward control flow, a small helper, or a lookup table.
- **Comments explain WHY, not WHAT.** Prefer self-documenting code.

### Errors

- **Every catch must do something deliberate** — re-throw, return a typed error, or log with context. Graceful degradation is fine when the degradation is logged and observable.
- **Use `SafeError`** for errors that should surface to users, and `captureException` for Sentry reporting. Don't leak raw error messages from third-party APIs.

### Logging

- **Tests use the real logger** — do NOT mock `@/utils/logger`.
- **Don't duplicate logger context fields** from higher in the call chain (the scoped logger already carries them).
- **Use `logger.trace()` for PII** (from, to, subject, message bodies). The authenticated user's own email may be logged at any level.

### Secrets & config

- **Never hardcode API keys, secrets, database URLs, or credentials.** Use env vars.
- **New env vars:** add to `.env.example`, `apps/web/env.ts`, and `turbo.json`. Prefix client-side with `NEXT_PUBLIC_`.

### Provider abstractions

- **Prefer `EmailProvider`** (`apps/web/utils/email/types.ts`). Only fall back to `isGoogleProvider` / `isMicrosoftProvider` (`utils/email/provider-types.ts`) at true integration boundaries.

### Lint

- **Don't disable lint rules inline.** When unavoidable, add a comment explaining why.

## Gotchas
- Next.js App Router components are Server Components by default. Add `'use client'` only when the component needs browser APIs, event handlers, or React hooks like useState/useEffect.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
