---
name: api-patterns
description: "Invoke when implementing API routes, request handling, middleware, or error responses. Contains validation, error format, route architecture, and authorization patterns."
---

# API Patterns

## Detected
- Framework: Next.js
- Validation: zod (95%)
- Auth: Better Auth

### Library Rules
- Verify Stripe webhook signatures using `stripe.webhooks.constructEvent()` with the raw body and signing secret. Never trust webhook payloads without signature verification — they can be forged.
- Use `.safeParse()` instead of `.parse()` for user input validation. `.parse()` throws on invalid input — `.safeParse()` returns a result object with typed errors, enabling structured error responses.
- Server Components fetch data directly from service functions or the database. Route Handlers are for EXTERNAL clients (webhooks, mobile apps, third-party integrations). Never call your own Route Handlers from Server Components — that adds an unnecessary network hop.

## Rules

### Route vs server action

- **Mutations are server actions, NOT POST API routes.** Use `next-safe-action` in `apps/web/utils/actions/<feature>.ts`. Co-locate `<feature>.validation.ts` (Zod) and `<feature>.test.ts`. On the client, use `useAction` and `getActionErrorMessage(error.error)` for error handling.
- **Exception:** mobile-native integrations may use POST API routes when they require a stable HTTP contract.
- **API routes are for external clients** (webhooks, third-party integrations, cron, mobile, public API) and GET data fetching. Server Components fetch directly via service functions — never call your own route handlers from Server Components.

### Route structure

- **One resource per route file** at `apps/web/app/api/<resource>/route.ts`.
- **Wrap every handler in middleware:**
  - `withError("route-name", handler)` — public routes, no auth (still gives you `request.logger` and Sentry-wrapped errors).
  - `withAuth(handler)` — requires authenticated user.
  - `withEmailAccount(handler)` — requires authenticated user AND an active email account; most authorization in this product scopes to email account, not user.
- **Export the response type** from the handler file: `export type ResponseType = Awaited<ReturnType<typeof getData>>`. Clients import this type for SWR/`useAction` results.
- **Cron-style routes** authenticate via `hasPostCronSecret(request)`. Internal jobs use `INTERNAL_API_KEY`.
- **Set `export const maxDuration = N`** at the top of long-running route files (default Vercel limits are too short for AI calls).
- **Use `request.logger`** rather than constructing a logger inside the handler — the middleware provides a scoped logger with request context.

### Validation

- **Zod via `.safeParse()` in route handlers**, not `.parse()`. Return 400 with structured validation errors on failure.
- **Infer types via `z.infer<typeof schema>`** — never duplicate as a separate interface.
- **108 of 168 routes lack top-of-file validation imports** per scan — some use middleware/wrapper-based validation. Before adding validation to a route, check whether the wrapper already handles it.

### Errors & response shape

- **Consistent error shape from every endpoint.** Never leak stack traces, database errors, or internal paths in production responses. The `withError` wrapper handles Sentry capture.
- **Verify webhook signatures** (Stripe, Lemon Squeezy) using the raw body before processing.

### Authorization

- **API-layer auth** lives here (middleware wrappers). Data-layer scoping (the `where: { emailAccountId }` clause) lives in `data-access`. Both layers must hold — never rely on only one.
- **Verify the requesting user owns the resource.** An authenticated user must not access another user's data by changing an ID in the URL.

### Provider boundaries

- **Use the `EmailProvider` abstraction** (`apps/web/utils/email/types.ts`) for Gmail/Outlook operations. `isGoogleProvider` / `isMicrosoftProvider` checks belong only at true integration boundary code (`utils/gmail/`, `utils/microsoft/`).

## Gotchas
- Don't call Route Handlers from Server Components. Call the data function directly — the Route Handler is for external clients, not internal server-side calls.
- Always verify webhook signatures before processing Stripe events. Use `stripe.webhooks.constructEvent()` with the raw body — never trust the payload without verification.
- Use `.safeParse()` in route handlers, not `.parse()`. `.parse()` throws on invalid input — use `.safeParse()` and return a 400 with validation error details.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
