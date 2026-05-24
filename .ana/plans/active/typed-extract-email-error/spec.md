# Spec: Type the failure of extractEmailOrThrow with SafeError

**Created by:** AnaPlan
**Date:** 2026-05-24
**Scope:** .ana/plans/active/typed-extract-email-error/scope.md

## Approach

Replace the generic `throw new Error("Invalid newsletter email address")` in `extractEmailOrThrow` (`apps/web/utils/senders/record.ts:13`) with `throw new SafeError("Invalid newsletter email address", 400)`. The message string is preserved exactly. The change is purely structural — it adds a typed discriminator (`instanceof SafeError`, `statusCode === 400`) without altering content or runtime behavior at the call sites.

`SafeError` is the established project pattern for typed, client-safe error signaling — used 30+ times across the codebase. The structural analog is `apps/web/app/api/user/booking-links/route.ts:76` (`throw new SafeError("User not found", 404);`). Mirror that call shape exactly.

Strengthen the existing test in `record.test.ts:39-43` to assert all three properties of the thrown error: `instanceof SafeError`, `statusCode === 400`, and `message === "Invalid newsletter email address"`. Use a `try/catch + expect.fail()` block so all three assertions run against a single caught instance. Vitest's `.toThrow(SafeError)` only checks the class, and `.toThrow(string)` only checks the message — neither covers `statusCode`. The codebase has no precedent for asserting `statusCode` on a sync throw, so this spec establishes one.

No new abstractions, no new files. Two existing call sites (`upsertSenderRecord`, `unsubscribeSenderAndMark`) continue to propagate unchanged.

## Output Mockups

Not applicable — this change has no user-facing output. The error message string (`"Invalid newsletter email address"`) is preserved exactly, so anything downstream that reads `error.message` (Sentry, logger, `actionClient`'s server-error handler) continues to surface the same text.

One behavioral side effect, already flagged in scope: `actionClient` (`apps/web/utils/actions/safe-action.ts:59`) special-cases `SafeError` by forwarding `error.message` to the client. Before this change, a malformed email reaching `extractEmailOrThrow` would surface as a generic server error. After, it would surface as `"Invalid newsletter email address"` to the client. This is intentional and consistent with how every other `SafeError` in the codebase behaves.

## File Changes

### apps/web/utils/senders/record.ts (modify)
**What changes:** Import `SafeError` from `@/utils/error`. Replace `throw new Error("Invalid newsletter email address")` with `throw new SafeError("Invalid newsletter email address", 400)` inside `extractEmailOrThrow`. No other changes.
**Pattern to follow:** `apps/web/app/api/user/booking-links/route.ts:76` — `throw new SafeError("User not found", 404);` — same call shape with statusCode.
**Why:** Untyped `Error` forces any future caller that wants to branch on this failure to string-match on `error.message`, which is brittle. `SafeError` with `statusCode` is the project's idiomatic discriminator (zero subclasses of `SafeError` exist; `instanceof SafeError && statusCode === N` is the pattern).

### apps/web/utils/senders/record.test.ts (modify)
**What changes:** Strengthen the existing `"throws for invalid newsletter emails"` test (lines 39-43). Replace the single `expect(() => ...).toThrow(string)` assertion with a `try/catch + expect.fail()` block that asserts: the caught error is `instanceof SafeError`, has `statusCode === 400`, and has `message === "Invalid newsletter email address"`. Import `SafeError` from `@/utils/error`. Rename the test to `"throws SafeError with 400 for invalid newsletter emails"` so the test description reflects the new contract.
**Pattern to follow:** The `try/catch + expect.fail()` shape is the only Vitest idiom that asserts all three properties on a single caught instance. The codebase uses `.rejects.toThrow(SafeError)` for async throws (`apps/web/utils/api-auth.test.ts:26`); this is the sync equivalent that also asserts `statusCode`.
**Why:** The whole point of the source change is to make the error's *shape* (class + statusCode) part of the contract. A test that only asserts `message` would not catch a regression that removes the SafeError typing.

## Acceptance Criteria

- [ ] AC1: `extractEmailOrThrow` in `apps/web/utils/senders/record.ts` throws `SafeError` (not generic `Error`) when `extractEmailAddress` returns an empty string.
- [ ] AC2: The thrown `SafeError` has `statusCode === 400` and `message === "Invalid newsletter email address"` (unchanged from current).
- [ ] AC3: The existing test in `apps/web/utils/senders/record.test.ts:39-43` is strengthened to assert: the thrown error is an `instanceof SafeError`, `statusCode === 400`, and `message === "Invalid newsletter email address"`.
- [ ] AC4: Both existing callers (`upsertSenderRecord` at `record.ts:26`, `unsubscribeSenderAndMark` at `unsubscribe.ts:66`) compile without modification and propagate the thrown error unchanged. No caller behavior changes in this scope.
- [ ] AC5: `apps/web/utils/senders/record.test.ts` and `apps/web/utils/senders/unsubscribe.test.ts` continue to pass.
- [ ] AC6 (added): Lint passes (`pnpm lint`) — no new lint violations introduced by the change.
- [ ] AC7 (added): No `error.message === "Invalid newsletter email address"` string-match consumers exist that would silently be replaced by typed branching (verified by grep before build; documented in build report if any are found).

## Testing Strategy

- **Unit tests:** Strengthen the existing test in `record.test.ts` (do not add a new test — the existing one covers the same failure case and just needs a stronger assertion). Use `try/catch + expect.fail()` to assert all three properties on a single caught instance.
- **Integration tests:** None required. The two call sites (`upsertSenderRecord`, `unsubscribeSenderAndMark`) don't change behavior — they continue to propagate the thrown error. Existing tests for `unsubscribe.ts` (`apps/web/utils/senders/unsubscribe.test.ts`) must continue to pass; treat this as a regression-only check.
- **Edge cases:** Only the one already covered — `extractEmailAddress` returns empty string. No new edge cases introduced.

## Dependencies

None. `SafeError` already exists at `apps/web/utils/error.ts:149-159`. The import alias `@/utils/error` is the canonical import path (used by 30+ files).

## Constraints

- **Message string must remain exactly `"Invalid newsletter email address"`.** Sentry groups errors by message text; changing the string would split historical breadcrumbs into two groups. Logger and any caller reading `error.message` would also see the change.
- **No new abstractions.** Do not introduce a `SafeError` subclass or add a `code` field to `SafeError`. Scope's Rejected Approaches section explicitly rules these out.
- **Backward-compatible at call sites.** `upsertSenderRecord` and `unsubscribeSenderAndMark` must not be modified. They continue to propagate the thrown error as before.
- **No new keyword/regex band-aids.** This is a structural change, not a model-output check. Aligns with the project's "No keyword or regex band-aids" design principle.

## Gotchas

- **`SafeError`'s constructor sets BOTH `message` and `safeMessage`.** The line `super(safeMessage)` propagates the argument to `Error`'s constructor, which assigns it to `message`. Don't assert only on `safeMessage` and miss the `message` regression. The contract asserts `message` (the canonical field).
- **`error.name` changes from `"Error"` to `"SafeError"`.** Confirmed via grep that no `error.name === "Error"` or `Object.getPrototypeOf(error) === Error.prototype` checks exist in `apps/web`. Safe to proceed. If the build phase discovers a hit before opening the PR, flag it in the build report.
- **`actionClient` will now forward the message to the client.** `apps/web/utils/actions/safe-action.ts:59` special-cases `SafeError` and returns `error.message` to the caller. Before this change, a malformed newsletter email surfaced as a generic 500. After, it surfaces as `"Invalid newsletter email address"` (a 400 from the server's perspective, surfaced as the SafeError contract dictates). This is an intentional UX improvement — the message contains no PII or internals.
- **Do not add `expect.fail()`'s message argument as a string literal that would itself be brittle.** Use the canonical Vitest form: `expect.fail("Expected extractEmailOrThrow to throw");` — short, declarative, names the failure mode.

## Build Brief

### Rules That Apply

- **Imports at the top of files; no mid-file dynamic imports.** Add `import { SafeError } from "@/utils/error";` to the top of `record.ts` and `record.test.ts`.
- **Path alias `@/` for cross-module imports.** Do not use relative `../../../utils/error` — match the codebase convention (30+ files use `@/utils/error`).
- **Co-locate unit tests next to source files.** The test stays at `record.test.ts` next to `record.ts` — no move required.
- **Use Vitest's `expect.fail()` (not `throw new Error(...)`) to signal "expected throw did not happen".** Vitest's built-in is clearer in test output.
- **Only add comments for "why", not "what".** The change is self-explanatory; do not add comments to the modified line.
- **Do not change the error message string.** Preserves Sentry grouping and existing `error.message` consumers.
- **Tests use the real logger.** Not relevant here (no logger involvement), but consistent with project convention — don't mock `@/utils/logger`.

### Pattern Extracts

**Current state of `record.ts:11-15`:**
```ts
export function extractEmailOrThrow(newsletterEmail: string) {
  const email = extractEmailAddress(newsletterEmail);
  if (!email) throw new Error("Invalid newsletter email address");
  return email;
}
```

**Structural analog — `apps/web/app/api/user/booking-links/route.ts:76`:**
```ts
if (!emailAccount) throw new SafeError("User not found", 404);
```

**`SafeError` definition — `apps/web/utils/error.ts:149-159`:**
```ts
export class SafeError extends Error {
  safeMessage?: string;
  statusCode?: number;

  constructor(safeMessage?: string, statusCode?: number) {
    super(safeMessage);
    this.name = "SafeError";
    this.safeMessage = safeMessage;
    this.statusCode = statusCode;
  }
}
```

**Current test — `apps/web/utils/senders/record.test.ts:39-43`:**
```ts
it("throws for invalid newsletter emails", () => {
  expect(() => extractEmailOrThrow("invalid-email")).toThrow(
    "Invalid newsletter email address",
  );
});
```

**Async SafeError test idiom in this codebase — `apps/web/utils/api-auth.test.ts:26`:**
```ts
await expect(validateApiKey(getRequest(null))).rejects.toThrow(SafeError);
```
(This idiom asserts class only — for sync throws asserting *all three* properties, use try/catch as described in Testing Strategy.)

### Proof Context

No active proof findings for affected files. (This is the first pipeline cycle in the project; no proof chain has been built yet.)

### Checkpoint Commands

Surface: `web` (from scope).

- After modifying `record.ts`: `(cd 'apps/web' && pnpm run test -- --run utils/senders/record.test.ts)` — Expected: 2 tests pass (the existing upsert test plus the strengthened throw test). The throw test will fail until the test file is also updated.
- After modifying `record.test.ts`: `(cd 'apps/web' && pnpm run test -- --run utils/senders/record.test.ts utils/senders/unsubscribe.test.ts)` — Expected: both files green; no regressions in `unsubscribe.test.ts`.
- After all changes: `pnpm run test -- --run` — Expected: 4412 total tests, 3753 passing, 659 skipped (unchanged from baseline; no new tests added, the existing test is strengthened but still counts as 1).
- Lint: `(cd 'apps/web' && pnpm run lint)` — Expected: pass.

### Build Baseline

Ran `(cd 'apps/web' && pnpm run test -- --run apps/web/utils/senders/record.test.ts apps/web/utils/senders/unsubscribe.test.ts)` from repo root (note: vitest in this repo ignores the file filter when run via `pnpm test -- --run` and runs the full suite; the counts below are the full suite, which is fine as a regression baseline).

- Current tests: **3753 passing, 659 skipped (4412 total)** across **409 passing, 95 skipped (504 total) test files**
- Current test files: **504**
- Command used: `(cd 'apps/web' && pnpm run test -- --run apps/web/utils/senders/record.test.ts apps/web/utils/senders/unsubscribe.test.ts)` — surfaced full-suite counts
- After build: **expected 3753 passing, 659 skipped (4412 total) across 504 test files** — no new tests added; the strengthened test still counts as 1
- Regression focus: `apps/web/utils/senders/record.test.ts` (modified), `apps/web/utils/senders/unsubscribe.test.ts` (caller, unchanged but exercises the same code path)
