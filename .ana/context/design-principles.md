# Design Principles

<!-- Starting principles for AI-augmented development.
     Edit to match your team's philosophy, or replace entirely.
     Ana reads this to understand HOW your team thinks. -->

## Name the disease, not the symptom

Before fixing something, state the root cause in one sentence. A fix that addresses the cause is one fix forever. A fix that addresses the symptom is the first of many.

## Surface tradeoffs before committing

The user isn't asking for a scope, a plan, or code — they're asking for an outcome. Every approach has costs; if the obvious path undermines that outcome, say so before building. Show them the paths, not just the fastest one.

## Every change should be foundation, not scaffolding

Foundation is code you build on top of. Scaffolding is code you tear down later. The test: would a senior engineer approve this — not just for correctness, but for craft? If the answer is "this works, but it's not how we'd do it if we had time" — you don't have time NOT to do it right.

## Never lock to a single AI provider

11 providers wired through the Vercel AI SDK isn't bloat — it's insurance. Claude for nuanced replies, GPT for fast categorization, Google for multilingual. When introducing a new AI feature, the model choice goes through the `utils/llms/` abstraction. If a change makes provider switching require a code rewrite (not a config change), it's a regression.

## Test AI decision paths, not just the happy path

This is people's real email — a wrong decision deletes mail, leaks data, or mis-replies to a customer. Pre-commit hooks run lint only; trust comes from evals (`ai-evals.yml`) and unit tests on the rule-matching, draft-reply, and classification logic. New AI features ship with at least one eval that exercises a failure mode (wrong-rule match, hallucinated content, sensitive data leak), not just success cases. No new prompt or model change merges without an eval that would have caught the regression.

## No keyword or regex band-aids for model behavior

When a model produces bad wording, don't patch it with a keyword blacklist in the prompt, eval, or test. Assert the semantic failure mode with a judge/eval criterion instead. The product runs across languages, so English-specific text checks are especially brittle. The same logic applies at runtime: never gate context injection or tool behavior on ad hoc user-text matching — use structured state, metadata, or explicit events instead.

<!-- Add your team's principles below. What tradeoffs do you consistently make?
     What quality bar do you hold? What does "good" mean here?

     A principle changes decisions. "Write clean code" is a platitude.
     "We prefer Result<T,E> over thrown errors" is a principle. -->
