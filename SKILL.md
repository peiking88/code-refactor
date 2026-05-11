---
name: code-refactor
description: "Systematic, data-driven code refactoring and technical debt remediation methodology with caller-count auditing, type-safety boundary analysis, debt inventory, ROI-based prioritization, and tradeoff evaluation. Use when the user wants to systematically refactor a module, inline single-caller helpers, collapse duplicate branches, bundle parameter objects, eliminate code smells with evidence, assess refactoring safety, inventory technical debt, or plan a prioritized remediation roadmap — especially in Python, TypeScript, or Java codebases. Trigger on requests like 'refactor this module', 'audit this file for refactoring', 'how should I clean this up systematically', 'is this too complex to maintain', 'what code smells are in this file', 'assess our technical debt', 'create a debt remediation plan', 'prioritize tech debt cleanup'."
compatibility: "security-and-hardening (for security axis of pre-refactor scan), performance-optimization (for bottleneck profiling)"
---

# Code Refactor

Evidence-based code review and refactoring methodology. This file covers language-agnostic principles, workflow, and decision frameworks. **For language-specific code examples, read the corresponding reference file:**

| Language | Reference |
|---|---|
| Python | `references/python.md` |
| TypeScript | `references/typescript.md` |
| Java | `references/java.md` |

Read the reference file for the language you're working in when you need concrete syntax for a pattern. The reference files cover: type safety boundaries, all 10 code smells with Before/After examples, parameter objects, collapsing branches, and design patterns (Strategy, Chain of Responsibility).

---

## Core Principles

1. **Behavior is preserved** — Refactoring changes structure, not external behavior
2. **Small steps** — One logical change per commit, test between each
3. **Evidence first** — Count callers, show diffs, state tradeoffs before implementing
4. **Simple over clever** — Inline trivials, extract only when the abstraction earns its cost
5. **Never refactor tests** — Tests define expected behavior. Refactoring them alongside production code destroys your safety net. If a test needs to change, you're not refactoring — you're changing behavior. Stop and reassess.

## When NOT to Refactor

- **Code that won't change again**: If it works and has zero planned changes, refactoring adds risk without value
- **No test coverage**: Without tests, you're not refactoring — you're editing blind. Write tests first
- **Under tight deadline**: Refactoring requires safety net time. Defer and schedule it
- **Mixed with bug fixes**: Fix the bug minimally first in its own commit, then refactor separately
- **"Just because"**: Need a clear purpose — cleaner code is a means, not an end in itself
- **Test files**: Never refactor test code as part of a production-code refactoring pass. Tests are the specification of behavior — if they're messy, refactor them in a separate, dedicated pass with extra scrutiny
- **Code you don't understand**: Chesterton's Fence — if you see a fence across a road and don't know why it's there, don't tear it down. Understand the reason first, then decide if the reason still applies. Check git blame for original context before removing anything that looks odd

---

## Pre-Refactoring Assessment (Five-Axis Quick Scan)

Before touching any code, run a quick five-axis scan to confirm refactoring is the right intervention and to identify what needs attention:

| Axis | Question | Red Flag |
|---|---|---|
| **Correctness** | Does it do what it claims to do? | Edge cases silently ignored, error paths missing |
| **Readability** | Can a stranger understand this in one pass? | Generic names (`data`, `temp`, `result`), nested ternaries, dead comments |
| **Architecture** | Does it fit the system's design? | Circular dependencies, wrong abstraction level, scattered responsibility |
| **Security** | Any vulnerability at system boundaries? | Raw input unsanitized, secrets in code, injection risk |
| **Performance** | Any obvious bottleneck? | N+1 queries, unbounded loops, large allocations in hot paths |

If the scan reveals correctness or security issues, those are bugs — fix them first in a separate commit, then refactor. If it reveals only readability and architecture issues, proceed with refactoring.

For detailed security review, consult `security-and-hardening`. For detailed performance profiling, consult `performance-optimization`.

---

## Output Format

When presenting a refactoring result, always include:

```
1. WHAT changed (one line)
2. BEFORE/AFTER diff
3. TRADEOFF: what got better AND what got worse
```

No refactoring is purely "better" — each has a cost. State it explicitly.

Example:
```
Proposed: inline replaceCurrentSheet into write()

 write(text: string) {
     ydoc.transact(() => {
-        if (type === 'sheet') replaceCurrentSheet(text);
+        if (type === 'sheet') {
+            entry.columns.forEach((_, key) => entry.columns.delete(key));
+            entry.rows.forEach((_, key) => entry.rows.delete(key));
+            parseSheetFromCsv(text, entry);
+        }

Tradeoff: write() grows from 6 to 14 lines, but is now self-contained
and the single-caller helper replaceCurrentSheet is eliminated.
```

### Severity Labeling

Prefix every finding with its severity so the reader knows what's required vs optional:

| Prefix | Meaning |
|---|---|
| *(no prefix)* | Required — must address |
| **Critical:** | Blocks merge — security, data loss, broken behavior |
| **Nit:** | Minor, optional — formatting, style preferences |
| **Optional:** / **Consider:** | Worth considering but not required |

---

## Audit: Count Callers First

Before any change, map every internal function/class with its exact call count:

| Callers | Action |
|---|---|
| 0 | Dead code. Delete. |
| 1 | Inline candidate. Keep only if: complex logic worth naming, part of a constructor family, or caller is already long. |
| 2-3 | Evaluate. If all callers in same method, might still inline. |
| 4+ | Keep. |

### When to Keep a Single-Caller Function

- **Part of a family**: `pushText`, `pushSheet`, `pushRichtext` share structure — inlining one breaks symmetry
- **Complex logic worth naming**: Deep-clone, recursive tree walk, multi-step parsing
- **Caller already long**: Inlining 15 lines into a 50-line method hurts readability

---

## Type Safety Boundaries

Route all raw/untyped access through a single parsing boundary. Everything downstream uses typed results. Core idea: one function/module owns the conversion from raw to typed, and all other code uses the typed result.

```
BAD: Multiple raw access points scattered across methods
     getter → raw.get('type')    as Type
     write  → raw.get('columns') as Map
     update → raw.get('content') as Text

GOOD: Single boundary, everything else uses typed discriminated union
     parseEntry()        → THE boundary (raw → typed)
     typed getter        → parseEntry(last)
     write/update/read   → typed getter (discriminated)
```

See the language-specific reference for concrete syntax.

---

## Code Smells Reference

The 10 code smells below have language-specific Before/After examples in the reference files. This section describes the recognition pattern and fix strategy.

| # | Smell | Recognition | Fix |
|---|---|---|---|
| 1 | Long Method | >50 lines, multiple responsibility paragraphs | Extract focused sub-methods |
| 2 | Duplicated Code | Function B = Function A + one step | Compose A into B |
| 3 | Long Parameter List | 2+ params always travel together | Bundle into parameter object |
| 4 | Nested Conditionals | Arrow code, >2 indent levels | Guard clauses / early return |
| 5 | Feature Envy | Method uses another object's fields more than its own | Move method to the other object |
| 6 | Primitive Obsession | String/Int used for domain concepts (email, phone, status) | Wrap in domain type with validation |
| 7 | Magic Numbers | Unexplained literal values | Named constant or enum |
| 8 | Dead Code | Unused functions, imports, commented-out blocks | Delete (git remembers) |
| 9 | Large Class | >10 public methods spanning multiple responsibilities | Split by responsibility axis |
| 10 | Inappropriate Intimacy | `a.b.c.d` chained access through nested objects | Single public method on owning object |

---

## Simplification Patterns

Beyond the 10 code smells, these concrete patterns signal simplification opportunities. Each is a specific, recognizable signal — not a vague smell.

**Structural complexity:**

| Pattern | Signal | Simplification |
|---|---|---|
| Deep nesting (3+ levels) | Hard to follow control flow | Guard clauses or helper functions |
| Long functions (50+ lines) | Multiple responsibilities | Split into focused functions |
| Nested ternaries | Requires mental stack to parse | if/else chain, switch, or lookup object |
| Boolean parameter flags | `doThing(true, false, true)` | Options object or separate functions |
| Repeated conditionals | Same `if` check in multiple places | Extract to predicate function |

**Naming and readability:**

| Pattern | Signal | Simplification |
|---|---|---|
| Generic names | `data`, `result`, `temp`, `val`, `item` | Rename to describe content: `userProfile`, `validationErrors` |
| Abbreviated names | `usr`, `cfg`, `btn` | Use full words unless universal (`id`, `url`, `api`) |
| Misleading names | Function named `get` that also mutates | Rename to reflect actual behavior |
| "What" comments | `// increment counter` above `count++` | Delete — code is clear enough |
| "Why" comments | `// Retry because API is flaky under load` | **Keep** — carries intent code can't express |

**Redundancy:**

| Pattern | Signal | Simplification |
|---|---|---|
| Duplicated logic | Same 5+ lines in multiple places | Extract to shared function |
| Dead code | Unreachable branches, unused imports, commented-out blocks | Remove after confirming |
| Unnecessary wrappers | Wrapper that adds no value | Inline, call underlying function directly |
| Over-engineered patterns | Factory-for-a-factory, strategy-with-one-strategy | Replace with direct approach |
| Redundant type assertions | Casting to a type already inferred | Remove the assertion |

---

## Collapsing Duplicate Branches

When a switch/if has 2+ branches doing the same thing with different inputs, collapse via a shared method. **The tell**: you can describe both branches with the same sentence.

```
BEFORE: "flatten to string and push" appears in both branches
  case 'richtext' → pushText(richtextToPlaintext(entry.content) + text)
  case 'sheet'    → pushText(sheetToCsv(entry) + text)

AFTER: read() already does "flatten to string"
  pushText(this.read() + text)
```

This also applies to `as*()` conversion methods. If every non-matching type branch does "read as string → push as target type", collapse to: early-return on same-type, then single conversion path for all others.

---

## Inline Heuristics

### Prefer Inline for Trivial Duplications

When a duplicated block is 1-3 lines and appears 2-3 times in the same file, keeping it inline is usually more readable than extracting a helper. A helper adds a name to learn, a definition to jump to, and an abstraction boundary to reason about — all cost more than the duplication saves.

**The test**: does a reader need to leave the call site to understand what's happening? If inline code is self-explanatory, extraction hurts more than it helps.

### When Extraction IS Worth It

- Block is 5+ lines (abstraction pays for itself)
- Appears 4+ times (pattern, not coincidence)
- Callers in different files (no local context to rely on)
- Logic is non-obvious and function name documents intent

### Inline Known-Behavior Calls

When a "smart" function branches internally but every caller already knows which branch it takes, inline the known branch at each call site. Keep the branching function only for callers that genuinely don't know the type.

### Over-Simplification Traps

Simplification has a failure mode. Watch for these:

- **Inlining too aggressively** — removing a helper that gave a concept a name makes the call site harder to read
- **Combining unrelated logic** — two simple functions merged into one complex function is not simpler
- **Removing necessary abstraction** — some abstractions exist for extensibility or testability, not complexity
- **Optimizing for line count** — fewer lines is not the goal; easier comprehension is

---

## Parameter Objects & Type Derivation

Two complementary patterns:

**Parameter objects**: When 2+ params always travel together, accept the bundling type instead of destructuring its fields.

**Type derivation**: Use intersection/composition instead of duplicating shared fields across types. The language reference files show the syntax for your language (TypeScript intersection types, Java record composition, Python TypedDict inheritance).

---

## Design Patterns

Six patterns that commonly emerge during refactoring. Each language reference file (`references/python.md`, `references/typescript.md`, `references/java.md`) contains concrete implementations.

| Pattern | Replaces | When to Apply |
|---|---|---|
| **Strategy** | Long if-else/switch chains | Algorithm selection varies at runtime |
| **Chain of Responsibility** | Nested validation conditionals | Multiple independent checks that can short-circuit |
| **Command** | Monolithic action handlers | Actions need undo, queue, or history support |
| **State** | Complex status-flag conditionals | Object behavior changes dramatically based on internal state |
| **Template Method** | Duplicated algorithm skeleton | Same workflow structure with varying step implementations |
| **Decorator** | Subclass explosion for feature combinations | Need to add optional behaviors without combinatorial inheritance |

See the Design Patterns section in each language reference file for Before/After code examples.

---

## Change Sizing & Commit Discipline

### Change Sizing

Small, focused changes are easier to review, faster to merge, and safer to deploy:

```
~100 lines changed   → Ideal. Reviewable in one sitting.
~300 lines changed   → Acceptable if a single logical change.
~1000 lines changed  → Too large. Split it.
```

**The Rule of 500**: If a refactoring would touch more than 500 lines, invest in automation (codemods, sed scripts, AST transforms) rather than manual edits. Manual changes at that scale are error-prone and exhausting to review.

**Splitting strategies:**

| Strategy | How | When |
|---|---|---|
| **Stack** | Submit a small change, start the next based on it | Sequential dependencies |
| **By file group** | Separate changes for groups needing different reviewers | Cross-cutting concerns |
| **Horizontal** | Create shared code/stubs first, then consumers | Layered architecture |
| **Vertical** | Break into smaller full-stack slices of the feature | Feature work |

**Separate refactoring from feature work.** A change that refactors existing code AND adds new behavior is two changes — submit them separately.

### Surgical Commits

One logical change per commit. Never mix two unrelated refactors:

```
edit → test → commit → next edit
```

Example of clean commit history:
```
98fcabe  refactor: inline ValidatedEntry type and single-use write helpers
af643fd  refactor: replace validated()/currentType() closures with this.currentEntry
19d108a  refactor: remove pushSheetFromCsv, compose pushSheet + parseSheetFromCsv
```

---

## Post-Refactor Straggler Sweep

After a refactor lands, hunt for dead references. Run these searches immediately after the refactor commit:

1. **Dead exports**: Grep for the removed symbol. If nothing imports it, un-export or delete.
2. **Stale imports**: Check each importing file for partially unused imports.
3. **Orphaned comments**: Comments referencing removed endpoints, deleted types, or old patterns — these mislead worse than no comment.
4. **Single-file directories**: If a directory now holds just an `__init__.py`/`index.ts` re-exporting one thing, flatten it.
5. **Unnecessary indirection**: A helper that wrapped a one-liner. A re-export chain where the middle barrel is a single passthrough.
6. **Indirect re-exports outside barrels**: `export { Foo } from './bar'` in non-barrel files. Implementation files export directly at the declaration.

The sweep is a separate commit labeled `refactor(scope): remove dead X`.

### Dead Code Hygiene

After identifying dead code, list it explicitly and **ask before deleting**:

```
DEAD CODE IDENTIFIED:
- formatLegacyDate() in src/utils/date.ts — replaced by formatDate()
- OldTaskCard component in src/components/ — replaced by TaskCard
- LEGACY_API_URL constant in src/config.ts — no remaining references
→ Safe to remove these?
```

Don't leave dead code lying around — it confuses future readers. But don't silently delete things you're unsure about. When in doubt, ask.

---

## Technical Debt Assessment

When the scope extends beyond a single module to project-wide debt, use this framework to inventory, quantify, and prioritize remediation.

### Debt Inventory

Scan for debt across five categories. Code debt overlaps with the Code Smells section above; the other four are unique to project-level assessment.

| Category | What to Scan | Quantify |
|---|---|---|
| **Code** | Duplicated logic, complex methods (>50 lines), deep nesting (>3 levels), god classes (>500 lines / >20 methods) | Lines duplicated, cyclomatic complexity, hotspot count |
| **Architecture** | Missing/leaky abstractions, violated boundaries, circular dependencies, monolithic components | Component size, dependency violations, coupling metrics |
| **Testing** | Untested paths, missing edge cases, no integration tests, brittle/flaky/slow tests | Coverage %, critical untested paths, test runtime |
| **Documentation** | Undocumented public APIs, missing architecture diagrams, no onboarding guides | Undocumented public API count |
| **Infrastructure** | Manual deployment steps, no rollback, missing monitoring, no performance baselines | Deployment time/failure rate |

### Impact Quantification

For each debt item, estimate real cost to justify remediation effort:

| Dimension | Template | Example |
|---|---|---|
| **Velocity** | `N hours/bug × M bugs/month = monthly cost` | Duplicate validation in 5 files: 2h/bug × 10 bugs = 20h/month |
| **Quality** | `bug rate × cost per bug (investigate + fix + test + deploy)` | Missing integration tests: 3 prod bugs/month × 9h each = 27h/month |
| **Risk** | Critical (security/data loss) / High (outages) / Medium (slow delivery) / Low (style) | Outdated auth library: Critical |

### Prioritized Remediation Roadmap

Rank by ROI (savings ÷ effort), not by severity alone:

| Tier | Timeframe | Criteria | Examples |
|---|---|---|---|
| **Quick Wins** | Week 1-2 | High value, low effort (< 16h), immediate ROI | Extract duplicate validation, add error monitoring, automate deployments |
| **Medium-Term** | Month 1-3 | Moderate effort (40-100h), positive ROI within 3 months | Split god class, upgrade framework, add test coverage to critical paths |
| **Long-Term** | Quarter 2-4 | Large effort (100h+), strategic value, positive ROI within 6 months | DDD adoption, comprehensive test suite, architecture modernization |

For each item, state: `Effort: Nh | Savings: Nh/month | ROI: positive after X months`

### Incremental Migration Strategy

When replacing legacy code, use the Strangler Fig pattern — never big-bang rewrites:

```
Phase 1: Facade over legacy (new clean interface, legacy underneath)
Phase 2: New implementation alongside legacy
Phase 3: Gradual migration with feature flags
Phase 4: Remove legacy path
```

Each phase is a separate deployable increment. If any phase fails, roll back to the previous phase.

### Prevention Strategy

**Automated Quality Gates** (pre-commit / CI):

| Gate | Threshold |
|---|---|
| Cyclomatic complexity | Max 10 per function |
| Duplication | Max 5% of codebase |
| Test coverage (new code) | Min 80% |
| Dependency audit | No high-severity vulnerabilities |
| Performance regression | Max 10% degradation |

**Debt Budget**: Allow max 2% monthly increase, require 5% quarterly reduction. Track with tooling (SonarQube, CodeCov, Dependabot).

### Success Metrics

| Period | Metrics |
|---|---|
| **Monthly** | Debt score trend, bug rate, deployment frequency, lead time, test coverage |
| **Quarterly** | Architecture health score, developer satisfaction, performance benchmarks, security audit results |

Target: debt score -5%/month, bug rate -20%, deployment frequency +50%.

---

## Anti-Patterns

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
- **"I'll clean it up later"**: Deferred cleanup rarely happens. The review is the quality gate — clean up before merge, not after.
- **Big-bang rewrites**: Replacing a legacy system in one shot. Use Strangler Fig pattern instead — incremental migration with feature flags.
- **Debt without metrics**: "We have a lot of tech debt" without quantification. No ROI calculation means no prioritization — measure first.
- **100% remediation goal**: Eliminating all debt is neither possible nor desirable. Target the high-ROI items and maintain a debt budget.

---

## Quick Checklist

### Before
- [ ] Tests exist and pass (write them if missing)
- [ ] Understand what the code does — run the five-axis scan
- [ ] Confirm this is NOT a test file (if it is, stop — test refactoring is a separate task)
- [ ] Commit current state
- [ ] Verify no bug fixes are mixed in (if they are, extract them first)

### During
- [ ] One logical change at a time
- [ ] Tests pass after each step
- [ ] Present diffs with tradeoffs before large changes
- [ ] Scope stays on the target code — no drive-by refactors of unrelated files
- [ ] Under 500 lines touched? If over, switch to automation

### After
- [ ] All tests pass without modification (if a test needed changes, you altered behavior — reassess)
- [ ] Straggler sweep completed (dead exports, stale imports, orphaned comments)
- [ ] No dead code left behind
- [ ] Diff is clean and reviewable — no unrelated changes mixed in
- [ ] Each simplification would pass the "new teammate" test: would a new team member understand this faster than the original?

### Debt Assessment
- [ ] Inventory across all 5 categories (Code, Architecture, Testing, Documentation, Infrastructure)
- [ ] Each item quantified with effort estimate and monthly cost/savings
- [ ] Roadmap prioritized by ROI, not just severity
- [ ] Prevention gates defined for future debt control
