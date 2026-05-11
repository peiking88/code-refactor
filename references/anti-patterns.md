# Anti-Patterns Reference

Comprehensive list of refactoring and code review anti-patterns. Each entry includes the pattern and why it's harmful.

## Refactoring Anti-Patterns

- **Refactoring test code alongside production code**: Tests are the specification. If a test breaks during refactoring, either you changed behavior (not refactoring) or the test was testing implementation details (fix the test in a separate, dedicated pass).
- **Premature extraction**: Extracting a 1-3 line block used 2-3 times into a named helper. The indirection costs more than the duplication.
- **Type-erasing helpers**: Any helper that accepts `unknown`/`Any`/`Object` to "reduce duplication" loses type safety.
- **Refactoring while fixing bugs**: Fix the bug minimally first, refactor in separate commit.
- **Batch-committing**: "Cleaned up the module" as one commit with 15 changes — impossible to review or revert.
- **Shotgun inlining**: Inlining everything with 1 caller regardless of context. Respect constructor families and complex logic.
- **Skipping the straggler sweep**: Code compiles but next person reads stale comments and wastes 30 minutes confused.
- **Identity functions**: `def f(x): return x` has callers but does nothing useful — it's dead code wearing a disguise.
- **Speculative v2 code**: Commented-out types or functions "deferred to v2". Git remembers — delete them.
- **Secondary key when primary exists**: Matching by name/slug/path when a stable id is available. Secondary keys can collide or change.
- **Simplifying code you don't understand**: Chesterton's Fence applies. Check git blame. Accumulated complexity often has no reason, but sometimes it does — verify before you simplify.

## Commit Discipline Anti-Patterns

- **"I'll clean it up later"**: Deferred cleanup rarely happens. The review is the quality gate — clean up before merge, not after.

## Code Review Anti-Patterns

- **Rubber-stamp review**: "LGTM" without evidence of review. Every change deserves actual scrutiny.
- **Reviewing only test results**: Tests passing is necessary but not sufficient — architecture, security, and readability need separate evaluation.

## Technical Debt Anti-Patterns

- **Big-bang rewrites**: Replacing a legacy system in one shot. Use Strangler Fig pattern instead — incremental migration with feature flags.
- **Debt without metrics**: "We have a lot of tech debt" without quantification. No ROI calculation means no prioritization — measure first.
- **100% remediation goal**: Eliminating all debt is neither possible nor desirable. Target the high-ROI items and maintain a debt budget.
