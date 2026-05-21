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
- Test behavior, not implementation. Assert on what the code returns or produces — not which internal functions it calls. Tests should survive refactoring when behavior is unchanged.
- Prefer real implementations over mocks. Mock only what you can't control: network calls, time, randomness. Every mock is a lie about how the system actually behaves.
- Cover the error path, not just the happy path. For each feature test, write at least one test for invalid input, missing data, or service failure.
- Assert on specific expected values from real inputs. `expect(status).toBe(200)` not `expect(status).toBeDefined()`. A test that passes regardless of whether the feature works catches nothing. Never write tautological tests — `expect(true).toBe(true)` proves nothing. If you can't determine the specific expected value, read the contract's `matcher`/`value` fields before falling back to a weak assertion.
- Never weaken a test to make it pass. If a test fails, fix the code or fix the expectation — never broaden assertions or catch exceptions to force green.

## Gotchas
- Vitest defaults to watch mode. Always pass `--run` in CI and non-interactive environments (e.g., `pnpm run test -- --run`).
- Playwright's auto-waiting means you rarely need waitFor* patterns — `locator.click()` and `expect(locator).toBeVisible()` auto-retry until the element is ready. `page.waitForSelector()` is legacy. Prefer `page.getByRole()`, `page.getByText()`, and `page.getByLabel()` over CSS selectors — they survive DOM refactors and match accessibility intent. Tests run isolated and parallel by default; use `test.describe.serial` only when sequential execution is required.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
