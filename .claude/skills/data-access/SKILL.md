---
name: data-access
description: "Invoke when working with database queries, schema changes, migrations, or data models. Contains project-specific ORM conventions and data access patterns."
---

# Data Access

## Detected
- Database: Prisma
- Schema: prisma → postgresql, 63 models, apps/web/prisma/schema.prisma

### Library Rules
- Run `prisma generate` after any `schema.prisma` change. The Prisma client is generated code — schema changes require regeneration before the new types are available.
- Scope every user-specific query to the authenticated user. Include `userId` (or equivalent ownership field) in every WHERE clause for user-specific data. A query without ownership scoping is an IDOR vulnerability — any authenticated user can access any other user's data by changing an ID in the URL.
- Never interpolate user input into `$queryRaw` or `$executeRaw`. Use parameterized queries: `prisma.$queryRaw\`SELECT * FROM users WHERE id = ${userId}\`` (tagged template — safe) not `prisma.$queryRawUnsafe("SELECT * FROM users WHERE id = " + userId)` (string concat — SQL injection).
- Paginate all list queries. Never return unbounded results from `findMany()`. Use `take` + `skip` or cursor-based pagination. An unbounded query on a table with 100K rows returns all 100K rows into memory.

## Rules

### Client & imports

- **Import the Prisma client from `@/utils/prisma`** (default export). It's a `globalThis`-cached singleton wrapping `PrismaClient` with the `PrismaPg` adapter and two extensions: `encryptedTokens` (transparent token encryption) and `auditPrismaQueries` (audit logging). Never instantiate `PrismaClient` directly — you lose both extensions and create a parallel connection pool.
- **Import generated types from `@/generated/prisma/client`**, NOT `@prisma/client`. The schema specifies a custom client output path (`../generated/prisma`) so the standard `@prisma/client` import will be empty or stale.
- **Import enums from `@/generated/prisma/enums`** (e.g., `MessagingProvider`). The CI step `check-enums` validates these import paths.
- **Prisma 7 is installed** — the Rust binary is gone, so no serverless binary warnings or copy steps. Don't carry forward Prisma 5/6 advice.

### Transactions

- **Never use dynamic Prisma transactions** — `prisma.$transaction(async (tx) => {...})` is forbidden. Use the array form `prisma.$transaction([op1, op2])` or restructure the mutation. Dynamic transactions hold connections open across awaits and exhaust the pool.
- **For transient connection errors**, wrap the operation in `withPrismaRetry(...)` (`@/utils/prisma-retry`). Used heavily in messaging, follow-up, and notification flows.

### Authorization scoping

- **Scope queries to email account, not user.** Most data in this product belongs to an `EmailAccount` (a connected Gmail/Outlook account), not directly to a `User`. A user can have multiple email accounts. Filter by `emailAccountId` in `where` clauses for email/rule/category/cold-email/follow-up data — not `userId`.
- **A missing `where` clause is an IDOR.** API-layer auth (`withEmailAccount`) is necessary but not sufficient — data queries must also scope. Both layers must hold.

### Raw queries

- **Never interpolate user input into `$queryRaw` or `$executeRaw`.** Use tagged templates (parameterized): `` prisma.$queryRaw`SELECT ... WHERE id = ${userId}` ``. Never `$queryRawUnsafe` with string concatenation.

### Schema changes & migrations

- **Schema file:** `apps/web/prisma/schema.prisma` (63 models). Run `pnpm prisma generate` after any schema edit — the client is generated code.
- **Migrations:** `pnpm prisma migrate dev` from `apps/web/`. E2E uses `prisma:migrate:e2e`; local uses `prisma:migrate:local`.

### Query performance

- **Paginate all list queries.** Never return unbounded `findMany()`. Use `take` + cursor or `take` + `skip`.
- **Select only the fields you need.** `select` clauses over fetching whole records.
- **No queries inside loops.** Use `findMany({ where: { id: { in: ids } } })` or relation includes — each loop iteration is a separate round trip.

### Analytics

- **Email-volume / analytics data goes to Tinybird**, not Postgres. Use `packages/tinybird` for time-series and aggregate queries. Postgres holds transactional state.

## Gotchas
- Always run `npx prisma generate` after schema changes. The Prisma client is generated code — schema changes are not reflected until regenerated.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
