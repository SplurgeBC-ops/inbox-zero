---
name: troubleshooting
description: "Invoke when debugging failures, diagnosing unexpected behavior, or investigating test failures. Contains project-specific failure modes, diagnostic workflows, and known issues."
---

# Troubleshooting

## Detected

### Common Issues
- **Hydration error: "Text content does not match server-rendered HTML" or "Hydration failed because the initial UI does not match"** — Find the component that renders differently on server vs client. Common causes: `Date.now()`, `Math.random()`, browser-only APIs (`window`, `localStorage`), or conditional rendering based on client state. Use `useEffect` for client-only values or `suppressHydrationWarning` for intentional mismatches. Prevention: Wrap browser-only code in `useEffect` or guard with `typeof window !== "undefined"` checks.
- **Middleware crashes with "Dynamic Code Evaluation not allowed" or "Module not found" errors for Node.js APIs (fs, crypto, etc.)** — Next.js middleware runs on Edge Runtime, which has no access to Node.js APIs. Use Web API equivalents (`crypto.subtle` instead of `node:crypto`). For heavy computation, move logic to an API route. Prevention: Keep middleware thin — auth checks, redirects, header rewrites only.
- **"useState is not a function" or "useEffect is not a function" or "Event handlers cannot be passed to Client Component props"** — Components using hooks (`useState`, `useEffect`) or event handlers (`onClick`, `onChange`) must have `"use client"` at the top of the file. Server Components cannot use React hooks or DOM event handlers.
- **Server Action throws "Functions cannot be passed directly to Client Components" or form submission silently fails** — Server Actions must be `async` functions marked with `"use server"` at the function or file level. If passing to a Client Component, the action must be in a separate file with `"use server"` at the top.
- **`params.id` is `undefined` or `params` is a Promise object instead of a plain object in Next.js 15+** — In Next.js 15+, `params` and `searchParams` are async. Use `const { id } = await params` instead of destructuring synchronously. This applies to page, layout, route handler, and generateMetadata functions.
- **Type errors after schema changes: "Property does not exist on type" or "PrismaClient is not generated" or types don't reflect new fields/models** — Run `npx prisma generate` after any `schema.prisma` change. The Prisma client is generated code — schema changes require regeneration before new types are available. Prevention: Add `"postinstall": "prisma generate"` to package.json scripts.
- **"The database schema is not in sync with the migration history" or `prisma migrate dev` wants to reset the database** — Run `npx prisma migrate diff` to see what drifted. If the database was modified directly (not through migrations), create a baseline migration with `npx prisma migrate diff --from-schema-datamodel --to-schema-datasource --script > fix.sql`. Apply and re-run `prisma migrate dev`. Prevention: Never modify the database directly in environments that use migrations. Use `prisma migrate dev` for all schema changes.
- **State value is stale inside `useEffect`, `setTimeout`, or event handlers — shows old value instead of current** — Closures capture state at render time. Use the functional updater form: `setState(prev => prev + 1)` instead of `setState(count + 1)`. For effects, add the dependency to the deps array. For refs that need current value, use `useRef`.
- **List items re-render incorrectly, input fields lose focus when typing, or "Each child in a list should have a unique key prop" warning** — Use a stable, unique identifier as the `key` prop — database IDs, not array indices. Index keys cause bugs when the list is reordered, filtered, or items are inserted. If no stable ID exists, generate one when the data is created.
- **TypeScript reports "possibly null" or "possibly undefined" after a type guard, but only in async functions — the guard works in sync code** — Type narrowing does not persist across `await` boundaries because the variable could be reassigned between suspension points. Re-narrow after each `await`: `if (!x) throw` before the `await`, then `if (!x) throw` again after.
- **Object literal type is widened to `string` instead of the literal value, or array type is `string[]` instead of a tuple** — Add `as const` to the declaration: `const x = { type: "success" } as const` gives `{ readonly type: "success" }` instead of `{ type: string }`. For arrays: `const arr = ["a", "b"] as const` gives `readonly ["a", "b"]` instead of `string[]`.
- **"ERR_REQUIRE_ESM" when importing an ESM-only package, or "SyntaxError: Cannot use import statement in a module"** — ESM-only packages (like `chalk` v5+, `execa` v6+, `node-fetch` v3+) cannot be `require()`'d. Either: (1) switch your project to ESM (`"type": "module"` in package.json), (2) use dynamic `import()` for the ESM package, or (3) pin an older CJS-compatible version.
- **Process crashes with "UnhandledPromiseRejection" — an async function throws but nothing catches it** — Ensure every Promise chain has a `.catch()` or is inside a `try/catch` in an `async` function. For fire-and-forget promises, add `.catch(err => logger.error(err))`. Check for missing `await` on async calls — without it, the rejection is unhandled. Prevention: Add `process.on("unhandledRejection", handler)` as a safety net, but fix the root cause — unhandled rejections indicate a code bug.
- **Tests hang indefinitely in CI or when run from scripts — process never exits** — Vitest defaults to watch mode in interactive terminals. Pass `--run` flag to run tests once and exit: `vitest run` instead of `vitest`. In CI, Vitest auto-detects non-interactive environments, but scripts piping output may still trigger watch mode. Prevention: Always use `vitest run` in CI scripts and non-interactive contexts.
- **Tests pass individually but fail when run together — mock from one test bleeds into another** — Call `vi.restoreAllMocks()` in `afterEach` or set `mockReset: true` / `restoreAllMocks: true` in `vitest.config.ts`. Module mocks (`vi.mock()`) persist across tests in the same file — use `vi.unmock()` or restructure to avoid shared module-level mocks.
- **Webhook signature verification fails with "No signatures found matching the expected signature" even though the signing secret is correct** — Stripe signature verification requires the RAW request body, not parsed JSON. In Next.js App Router: use `request.text()`. In Express: use `express.raw({ type: "application/json" })` on the webhook route. Body parsers (like `express.json()`) modify the body and break verification.

## Rules
*Not yet captured. This section grows from real debugging sessions.*

## Gotchas
*Not yet captured. Add as you discover them during development.*

## Examples
*Not yet captured. Add diagnostic workflows showing how to investigate common failures.*
