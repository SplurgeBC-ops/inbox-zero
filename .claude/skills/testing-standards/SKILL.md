---
name: testing-standards
description: "Invoke when writing tests, reviewing test quality, or setting up test infrastructure. Contains project-specific testing framework conventions, fixture patterns, and coverage expectations."
---

# Testing Standards

## Detected
- Framework: Vitest, Playwright, Testing Library (550 test files)
- Test command: pnpm run test -- --run
- Testing patterns: vitest
- Test location: co-located with source

### Library Rules
- Always pass `--run` flag when invoking Vitest in CI or non-interactive contexts. Vitest defaults to watch mode, which hangs pipelines waiting for input.

## Rules

### Universal

- **Test behavior, not implementation.** Assert on what the code returns or produces — not which internal functions it calls. Tests should survive refactoring when behavior is unchanged.
- **Prefer real implementations over mocks.** Mock only what you can't control: network calls, time, randomness. Every mock is a lie about how the system actually behaves.
- **Tests use the real logger.** Do NOT mock `@/utils/logger`.
- **Cover the error path, not just the happy path.** For each feature test, write at least one test for invalid input, missing data, or service failure.
- **Assert on specific values.** `expect(status).toBe(200)`, not `expect(status).toBeDefined()`. Never tautological.
- **Never weaken a test to make it pass.** Fix the code or fix the expectation — don't broaden assertions or catch exceptions to force green.
- **Avoid low-value tests** that mostly restate implementation details. Prefer tests that catch a real behavioral regression.

### Project conventions

- **Unit tests co-locate with source** (`utils/example.test.ts`). Integration, E2E, AI, and eval tests live in `apps/web/__tests__/{integration,e2e,playwright,eval}/`.
- **TDD for bug fixes by default** — red/green/refactor when practical. AI prompt improvements should also be backed by evals; TDD often applies there too.
- **Mock Prisma via `@/utils/__mocks__/prisma`** with `vi.mock("@/utils/prisma")` at the top of the test file. The mocked client is fully typed.
- **Use shared helpers and mock factories** rather than re-rolling them per test: `createTestLogger`, `getMockMessage` from `@/__tests__/helpers`, `createMockEmailProvider` from `@/__tests__/mocks/email-provider.mock`, and the rest of `__tests__/mocks/`.
- **Use `vi.hoisted()`** when a `vi.mock()` factory needs to share state with the test body (so the state is initialized before mock hoisting).
- **Reach for the `EmailProvider` abstraction in tests too.** `createMockEmailProvider` covers both Gmail and Outlook paths — don't write provider-specific mocks unless you're at a true integration boundary.

### Running tests

- **`pnpm test` runs unit tests only.** Integration, AI, and E2E suites are env-gated and won't run unless their flag is set.
- **Integration tests:** `pnpm -F inbox-zero-ai test-integration` (sets `RUN_INTEGRATION_TESTS=true`).
- **AI tests / evals:** `pnpm --filter inbox-zero-ai test-ai` from repo root (sets `RUN_AI_TESTS=true`, default model is OpenRouter `anthropic/claude-sonnet-4.5`). Eval files in `__tests__/eval/` only run via this command — `pnpm test` skips them silently.
- **E2E flows:** `pnpm -F inbox-zero-ai test-e2e:flows` (sets `RUN_E2E_FLOW_TESTS=true`, loads `.env.e2e`).
- **Always pass `--run`** when invoking Vitest in CI or scripts. Default is watch mode.

### LLM tests specifically

- **Assert semantic failure modes, not text patterns.** For LLM behavior, use a judge/eval criterion or a structured-contract assertion. Do not add English-keyword or phrase blacklists to evals — the product runs across languages and the model's bad-wording du jour will change. Example: assert "does not invent a payment status or ask unnecessary clarification," not "does not contain the phrase 'could you clarify'."
- **Every new AI feature ships with an eval that exercises a real failure mode** (wrong-rule match, hallucinated content, sensitive data leak), not just success. Prompt and model changes need an eval that would have caught the regression.

## Gotchas
- Vitest defaults to watch mode. Always pass `--run` in CI and non-interactive environments (e.g., `pnpm run test -- --run`).
- Playwright's auto-waiting means you rarely need waitFor* patterns — `locator.click()` and `expect(locator).toBeVisible()` auto-retry until the element is ready. `page.waitForSelector()` is legacy. Prefer `page.getByRole()`, `page.getByText()`, and `page.getByLabel()` over CSS selectors — they survive DOM refactors and match accessibility intent. Tests run isolated and parallel by default; use `test.describe.serial` only when sequential execution is required.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
