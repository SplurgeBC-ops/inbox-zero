# Scope: Type the failure of extractEmailOrThrow with SafeError

**Created by:** Ana
**Date:** 2026-05-24

## Intent

`extractEmailOrThrow` in `apps/web/utils/senders/record.ts:11-15` currently throws a generic `Error("Invalid newsletter email address")`. A caller wrapping it in a try/catch has no safe way to distinguish "the input email was invalid" from any other failure (DB error, network error, etc.) — they'd have to string-match on the message, which is brittle.

Switch the throw to `SafeError` with `statusCode: 400`. This is a foundation change: behavior of current callers is unchanged, but the typed signal is now available for any future caller that needs to branch on this specific failure (e.g., return a 400 to the client instead of a generic 500, surface a user-facing message, or skip the row in a batch instead of aborting).

This work was narrowed from an initial framing of "add input validation to the unsubscribe flow." On inspection, validation already exists at every public entry point — `unsubscribeSenderAction` is gated by `z.string().email()` in `unsubscriber.validation.ts:11`, and the notification-unsubscribe route at `app/api/unsubscribe/route.ts` doesn't take an email at all (it takes a token). The one genuine gap is the untyped error from `extractEmailOrThrow`, which this scope addresses.

## Complexity Assessment

- **Kind:** chore
- **Size:** small
- **Surface:** web
- **Files affected:**
  - `apps/web/utils/senders/record.ts` — change one throw (line 13)
  - `apps/web/utils/senders/record.test.ts` — strengthen one existing assertion (lines 39-43)
- **Blast radius:** Two production call sites: `upsertSenderRecord` in `record.ts:26`, and `unsubscribeSenderAndMark` in `apps/web/utils/senders/unsubscribe.ts:66`. Neither is modified; both continue to propagate. The error message stays identical, so any logging / Sentry breadcrumbs / `error.message` consumers continue to work.
- **Estimated effort:** ~15 minutes
- **Multi-phase:** no

## Approach

Replace `new Error("Invalid newsletter email address")` with `new SafeError("Invalid newsletter email address", 400)`. `SafeError` is the project's established pattern for typed, client-safe error signaling — used 30+ times across the codebase, with branching done via `instanceof SafeError && error.statusCode === N` (canonical example: `app/book/[slug]/opengraph-image.tsx:37`).

Keep the message string identical. The change is purely structural — adding a type/statusCode discriminator without altering content or behavior. No new abstractions, no new files, no new patterns.

## Acceptance Criteria

- AC1: `extractEmailOrThrow` in `apps/web/utils/senders/record.ts` throws `SafeError` (not generic `Error`) when `extractEmailAddress` returns an empty string.
- AC2: The thrown `SafeError` has `statusCode === 400` and `message === "Invalid newsletter email address"` (unchanged from current).
- AC3: The existing test in `apps/web/utils/senders/record.test.ts:39-43` is strengthened to assert: the thrown error is an `instanceof SafeError`, `statusCode === 400`, and `message === "Invalid newsletter email address"`.
- AC4: Both existing callers (`upsertSenderRecord` at `record.ts:26`, `unsubscribeSenderAndMark` at `unsubscribe.ts:66`) compile without modification and propagate the thrown error unchanged. No caller behavior changes in this scope.
- AC5: `apps/web/utils/senders/record.test.ts` and `apps/web/utils/senders/unsubscribe.test.ts` continue to pass.

## Edge Cases & Risks

- **`error.name` changes from `"Error"` to `"SafeError"`.** Any consumer that string-matches `error.name === "Error"` would behave differently. `SafeError` is already thrown ~30+ times in the codebase, so Sentry/logger/middleware tolerate it — but worth scanning for `name === "Error"` checks specifically in the propagation path (`upsertSenderRecord` → server action handler → `safe-action.ts`, and `unsubscribeSenderAndMark` → server action handler).
- **Server-action handler behavior.** `actionClient` in `apps/web/utils/actions/safe-action.ts:59` already special-cases `SafeError` by returning `error.message` to the client. This means: today, a malformed email reaching `extractEmailOrThrow` would surface as a generic server error. After this change, it would surface as `"Invalid newsletter email address"` to the client. This is a user-visible improvement, not a regression — and the message is already safe (no PII, no internals). Flag explicitly so AnaPlan and AnaVerify confirm it's intentional.
- **No new branching introduced.** The current callers still just propagate. The capability to branch is added, but exercising it is out of scope.

## Rejected Approaches

- **Introduce a new subclass `InvalidNewsletterEmailError extends SafeError`.** Would give more precise discrimination than `statusCode === 400`. Rejected because the project has zero `SafeError` subclasses today — introducing the first is a new pattern that compounds (next person extends it too, then we have a hierarchy). The two current call sites have a narrow enough error surface that `statusCode` is sufficient. Revisit only if a future caller needs to distinguish multiple 400s from the same try/catch.
- **Add a new `code` field to `SafeError` itself.** Would touch the shared `SafeError` class for a single caller's benefit. Out of scope and unjustified by current need.
- **Build a separate email validation utility with custom error types.** Initial framing. Rejected because the project's pattern is Zod schemas co-located with server actions (`*.validation.ts`), and `z.string().email()` already covers format and empty-string rejection at every public entry point. A parallel utility would be scaffolding, not foundation.
- **Leave the generic `Error` throw alone.** Status quo. Rejected because it forces any future caller that wants to branch on this failure to string-match on the message — brittle, and inconsistent with the rest of the codebase.

## Open Questions

None. Scope is fully resolved.

## Exploration Findings

### Patterns Discovered

- `apps/web/utils/error.ts:149-159` — `SafeError` class definition (`safeMessage`, `statusCode`, extends `Error`)
- `apps/web/app/book/[slug]/opengraph-image.tsx:37, 147` — canonical caller-side branching pattern: `error instanceof SafeError && error.statusCode === N`
- `apps/web/utils/actions/safe-action.ts:59` — `actionClient` server-error handler that special-cases `SafeError` and forwards its message to the client
- `apps/web/utils/middleware.ts:138, 206, 511` and `apps/web/utils/api-middleware.ts:179` — route-middleware handling of `SafeError` (uses `statusCode`)

### Constraints Discovered

- [TYPE-VERIFIED] `SafeError` constructor signature `(safeMessage?: string, statusCode?: number)` — `apps/web/utils/error.ts:153`
- [OBSERVED] Zero subclasses of `SafeError` exist in the codebase as of this scope (`grep -rn "extends SafeError"` returned no results). The project's idiom is `instanceof SafeError` + `statusCode`.
- [OBSERVED] `extractEmailOrThrow` has exactly 2 production call sites (`record.ts:26`, `unsubscribe.ts:66`) and 1 test reference (`record.test.ts:40`).

### Test Infrastructure

- `apps/web/utils/senders/record.test.ts` — Vitest, mocks `@/utils/prisma`. Existing test for `extractEmailOrThrow` uses `expect(() => ...).toThrow(string)`. Will be strengthened to a structured assertion (`expect.toThrow` with an object matcher, or a `try/catch` + instance/property checks).
- `apps/web/utils/senders/unsubscribe.test.ts` — exists; should be run as part of this change to confirm no regressions.

## For AnaPlan

### Structural Analog

`apps/web/app/api/user/draft-cleanup-settings/route.ts:20` — `if (!emailAccount) throw new SafeError("Email account not found");` — same shape as what we want: small validate-then-throw-SafeError. The version with statusCode is e.g. `apps/web/app/api/user/booking-links/route.ts:76` — `throw new SafeError("User not found", 404);`. This is exactly the call shape to mirror.

### Functional Analog (same domain, different shape)

`apps/web/utils/email.ts:37` — `extractEmailAddress`. Same domain (parsing an email string), inverse shape: returns `""` on failure instead of throwing. `extractEmailOrThrow` is the throwing wrapper around it. Reading this clarifies why `extractEmailOrThrow` exists at all.

### Relevant Code Paths

- `apps/web/utils/senders/record.ts:11-15` — the function being changed
- `apps/web/utils/senders/record.ts:26` — call site #1 (inside `upsertSenderRecord`)
- `apps/web/utils/senders/unsubscribe.ts:66` — call site #2 (inside `unsubscribeSenderAndMark`)
- `apps/web/utils/senders/record.test.ts:39-43` — existing test to strengthen
- `apps/web/utils/error.ts:149-159` — `SafeError` definition (import source)
- `apps/web/utils/actions/safe-action.ts:59` — confirms that `SafeError.message` surfaces to the client; this scope inherits that behavior

### Patterns to Follow

- Import `SafeError` from `@/utils/error` (per `apps/web/app/api/user/booking-links/route.ts:76` and many others)
- statusCode `400` is appropriate for client-input failures (see e.g. `apps/web/app/api/user/drive/preview/attachments/route.ts:69-75` for `SafeError` with status-bearing intent)
- Test pattern: project uses Vitest's `expect().toThrow()` widely; for richer assertions, the common shape is a `try { ... fail() } catch (e) { expect(e).toBeInstanceOf(SafeError); expect((e as SafeError).statusCode).toBe(400); }` or `expect.toThrow(expect.objectContaining({ ... }))` — AnaPlan should pick whichever already appears in this file or its neighbors for consistency

### Known Gotchas

- **`safeMessage` vs `message`.** `SafeError`'s constructor sets BOTH `message` (via `super(safeMessage)`) and a separate `safeMessage` property. Tests should assert on `message` (Vitest's `.toThrow(string)` matches `message`). Don't accidentally assert only on `safeMessage` and miss the `message` regression.
- **Don't change the message string.** Keeping `"Invalid newsletter email address"` identical preserves Sentry grouping, log breadcrumbs, and any caller that reads `error.message`.
- **`actionClient` will now leak the message to the client.** This was buried before (generic 500). It's intentional and an improvement, but call it out in the build report so AnaVerify can confirm.

### Things to Investigate

- Quick sanity check: search for any `error.name === "Error"` or `Object.getPrototypeOf(error) === Error.prototype` checks in the propagation path. None found in my scoping pass, but worth a final grep before changing.
- Confirm whether the test in `record.test.ts` should use `toThrow(expect.objectContaining(...))` or `try/catch + expect.toBeInstanceOf` — pick whichever pattern already exists in the surrounding test files for consistency.
