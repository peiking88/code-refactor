# Technical Debt Assessment Reference

Framework for inventorying, quantifying, and prioritizing project-wide technical debt. Loaded when the user asks to assess technical debt or create a remediation plan.

## Debt Inventory

Scan for debt across five categories:

| Category | What to Scan | Quantify |
|---|---|---|
| **Code** | Duplicated logic, complex methods (>50 lines), deep nesting (>3 levels), god classes (>500 lines / >20 methods) | Lines duplicated, cyclomatic complexity, hotspot count |
| **Architecture** | Missing/leaky abstractions, violated boundaries, circular dependencies, monolithic components | Component size, dependency violations, coupling metrics |
| **Exception** | Missing try/catch on I/O and network calls, swallowed exceptions, overly broad catch, wrong exception types, exception-driven control flow | Bare catch count, generic-exception count, unhandled-failure-point count |
| **Testing** | Untested paths, missing edge cases, no integration tests, brittle/flaky/slow tests | Coverage %, critical untested paths, test runtime |
| **Documentation** | Undocumented public APIs, missing architecture diagrams, no onboarding guides | Undocumented public API count |
| **Infrastructure** | Manual deployment steps, no rollback, missing monitoring, no performance baselines | Deployment time/failure rate |

## Impact Quantification

For each debt item, estimate real cost to justify remediation effort:

| Dimension | Template | Example |
|---|---|---|
| **Velocity** | `N hours/bug × M bugs/month = monthly cost` | Duplicate validation in 5 files: 2h/bug × 10 bugs = 20h/month |
| **Quality** | `bug rate × cost per bug (investigate + fix + test + deploy)` | Missing integration tests: 3 prod bugs/month × 9h each = 27h/month |
| **Risk** | Critical (security/data loss) / High (outages) / Medium (slow delivery) / Low (style) | Outdated auth library: Critical |

## Prioritized Remediation Roadmap

Rank by ROI (savings ÷ effort), not by severity alone:

| Tier | Timeframe | Criteria | Examples |
|---|---|---|---|
| **Quick Wins** | Week 1-2 | High value, low effort (< 16h), immediate ROI | Extract duplicate validation, add error monitoring, automate deployments |
| **Medium-Term** | Month 1-3 | Moderate effort (40-100h), positive ROI within 3 months | Split god class, upgrade framework, add test coverage to critical paths |
| **Long-Term** | Quarter 2-4 | Large effort (100h+), strategic value, positive ROI within 6 months | DDD adoption, comprehensive test suite, architecture modernization |

For each item, state: `Effort: Nh | Savings: Nh/month | ROI: positive after X months`

## Incremental Migration Strategy

When replacing legacy code, use the Strangler Fig pattern — never big-bang rewrites:

```
Phase 1: Facade over legacy (new clean interface, legacy underneath)
Phase 2: New implementation alongside legacy
Phase 3: Gradual migration with feature flags
Phase 4: Remove legacy path
```

Each phase is a separate deployable increment. If any phase fails, roll back to the previous phase.

## Prevention Strategy

**Automated Quality Gates** (pre-commit / CI):

| Gate | Threshold |
|---|---|
| Cyclomatic complexity | Max 10 per function |
| Duplication | Max 5% of codebase |
| Test coverage (new code) | Min 80% |
| Dependency audit | No high-severity vulnerabilities |
| Performance regression | Max 10% degradation |
| Bare / generic catch | Zero allowed — all catch blocks must specify a concrete exception type |
| Swallowed exceptions | Zero allowed — every catch block must log, rethrow, or recover |
| Unhandled I/O / network paths | Zero — every file, network, DB, or external API call must have error boundary |

**Debt Budget**: Allow max 2% monthly increase, require 5% quarterly reduction. Track with tooling (SonarQube, CodeCov, Dependabot).

## Success Metrics

| Period | Metrics |
|---|---|
| **Monthly** | Debt score trend, bug rate, deployment frequency, lead time, test coverage |
| **Quarterly** | Architecture health score, developer satisfaction, performance benchmarks, security audit results |

Target: debt score -5%/month, bug rate -20%, deployment frequency +50%.
