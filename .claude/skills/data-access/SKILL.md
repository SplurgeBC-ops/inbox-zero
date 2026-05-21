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
- Import the database client from a single shared module. Avoid instantiating new clients in route handlers or service functions — each instance opens its own connection pool.
- Wrap multi-step mutations in a transaction. If any step can fail, partial writes corrupt data — all steps succeed or all roll back.
- Avoid querying the database inside loops — use eager loading or joins for related data. Each loop iteration is a separate round trip.
- Select only the fields you need. Avoid fetching entire records when the consumer needs a few columns.
- Always scope data queries to the authorized context. Filter by the authenticated user, organization, or tenant — don't rely solely on API-layer checks to prevent unauthorized access. A missing `where` clause is an IDOR vulnerability.

## Gotchas
- Always run `npx prisma generate` after schema changes. The Prisma client is generated code — schema changes are not reflected until regenerated.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
