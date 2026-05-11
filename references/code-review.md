# Code Review Reference

Detailed code review process, anti-patterns, and best practices. Loaded when the task involves reviewing code before merge.

## Review Steps

1. **Understand context** — What does this change accomplish? What spec/task does it implement?
2. **Review tests first** — Tests reveal intent. Do they test behavior (not implementation)? Are edge cases covered? Would they catch a regression?
3. **Review implementation** — Walk through each file with the five axes (correctness, readability, architecture, security, performance)
4. **Categorize findings** — Label every comment with severity (Critical / no prefix / Nit / Optional / FYI)
5. **Verify the verification** — What tests ran? Build pass? Manual testing done? UI screenshots?

## Multi-Model Review

Different models have different blind spots. Use cross-review:

```
Model A writes code → Model B reviews → Model A addresses feedback → Human decides
```

This catches issues that a single model might miss.

## Change Descriptions

Every commit needs a description that stands alone in version control history:

- **First line**: Short, imperative, standalone. "Delete the FizzBuzz RPC" not "Deleting..."
- **Body**: What changed, why, decisions made, links to context
- **Anti-patterns**: "Fix bug", "Fix build", "Phase 1", "Moving code from A to B"

## Review Speed

- Respond within one business day (maximum, not target)
- Fast individual feedback beats slow final approval
- Large changes → ask the author to split rather than reviewing one massive diff

## Handling Disagreements

Apply this hierarchy when resolving disputes:

1. **Technical facts and data** override opinions
2. **Style guides** are absolute authority on style
3. **Engineering principles** override personal preference
4. **Codebase consistency** is acceptable if it doesn't degrade health

## Dependency Discipline

Before adding any dependency:

1. Does the existing stack solve this? (Often it does.)
2. How large is it? (Check bundle impact.)
3. Is it actively maintained? (Last commit, open issues.)
4. Known vulnerabilities? (`npm audit`, `pip audit`)
5. Compatible license?

Prefer standard library and existing utilities. Every dependency is a liability.

## Honesty in Review

- **Don't rubber-stamp.** "LGTM" without evidence of review helps no one.
- **Don't soften real issues.** "This might be minor" when it's a production bug is dishonest.
- **Quantify problems.** "This N+1 adds ~50ms per item" beats "this could be slow."
- **Push back on clear problems.** Sycophancy is a failure mode.
- **Accept override gracefully.** If the author has full context, defer to their judgment.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It works, that's good enough" | Unreadable/insecure code creates compounding debt |
| "I wrote it, so I know it's correct" | Authors are blind to their own assumptions |
| "We'll clean it up later" | Later never comes. The review is the quality gate |
| "AI-generated code is probably fine" | AI code needs more scrutiny — it's confident even when wrong |
| "The tests pass, so it's good" | Tests don't catch architecture, security, or readability issues |

## Red Flags

- PRs merged without review
- "LGTM" without evidence of actual review
- Review that only checks if tests pass (ignoring other axes)
- Security changes without security-focused review
- Large PRs that are "too big to review" (split them)
- No regression tests with bug fix PRs
- Accepting "I'll fix it later" — it never happens
