---
name: ios-sequential-thinking
description: "Step-by-step analysis with revision/branching for complex iOS problems — multi-layer data-flow bugs, cache/normalisation issues, navigation routing tangles, state-transition holes, Swift Concurrency races, money/PII correctness audits. Internal reasoning aid, not a planner. Reads .claude/ios-profile.md for context."
argument-hint: "[problem to analyse step-by-step]"
---

# Sequential Thinking — iOS

Structured problem-solving via reflective thought sequences with dynamic adjustment.

## Step 0 — Load profile (light)
Skim `.claude/ios-profile.md` for `architecture`, `state_type`, `navigation`,
`networking`, `high_rigor_domains` so the patterns below map onto this app's vocabulary.
If missing, skim the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it).

## When to Apply
At least one of:
- **Multi-step data flow** — Service → … → View; trace where state diverges.
- **Cache / normalisation** (if `networking` uses a cache) — query returns right data, UI shows stale.
- **Navigation routing** — push/pop/sheet ordering, retained closures, race after `await`.
- **State transitions** (`state_type`) — loading→loaded→error edges; empty vs error vs initial; pagination handoff.
- **Swift Concurrency** — `Task.cancel()` propagation, `@MainActor` violations, `Sendable`, actor reentrancy.
- **Hypothesis-driven debugging** — symptom is N layers from cause.
- **`high_rigor_domains` correctness audits** — wrong logic ships real-money / PII bugs.

## When NOT to Use
| Case | Use instead |
|---|---|
| Simple one-step answer | Just answer |
| Brutal trade-off comparison | `ios-brainstorm` |
| Plan + phases | `ios-plan` |
| Bug needs log/CI investigation | Direct `Bash`/`Read` |

---

## Core Process

1. **Loose estimate** — `Thought 1/N: [framing]`. Adjust N dynamically.
2. **Structure each thought** — build on prior context, one aspect each, state assumptions/
   uncertainties, signal what the next thought addresses.
3. **Dynamic adjustment** — Expand (more complexity), Contract (simpler), Revise, Branch.
4. **Revision**
   ```
   Thought 5/8 [REVISION of Thought 2]: <corrected understanding>
   - Original: … - Why revised: … - Impact: …
   ```
5. **Branching**
   ```
   Thought 4/7 [BRANCH A from 2]: <approach A>
   Thought 4/7 [BRANCH B from 2]: <approach B>
   ```
   Compare, converge with rationale.
6. **Hypothesis & verification**
   ```
   Thought 6/9 [HYPOTHESIS]: <cause/solution>
   Thought 7/9 [VERIFICATION]: <checked file:line — found …>
   ```
   Verification means reading actual code (path:line), not abstract reasoning.
7. **Complete only when ready** — `Thought N/N [FINAL]`.

## Modes
- **Explicit** (visible chain) — user asks for the trace; complexity warrants interception;
  `high_rigor_domains` correctness reasoning that needs an audit trail.
- **Implicit** (internal) — routine multi-step reasoning where visibility just adds noise.

---

## Reusable Patterns (adapt names to `state_type`/`navigation`/`networking`)

### State-transition audit
```
1: Define expected sequence (initial → loading → loaded/empty/error)
2: Locate the state holder (path:line)
3: Trace each emit point — does each branch exit cleanly?
4: Pagination — cursor handoff between pages
5 [HYPOTHESIS]: skipped state / lost cursor / double-emit
6 [VERIFICATION]: read tests — is this edge covered?
N [FINAL]: root cause + fix + test gap
```

### Cache / normalisation debug  (only if networking caches)
```
1: Identify query/fragment + cache key policy
2: What writes that key? (mutation, manual store update)
3: Observers — lifecycle attached to which owner?
4 [HYPOTHESIS]: stale entity after partial update
5 [VERIFICATION]: read generated query + key fn (never edit generated_paths)
N [FINAL]: cache fix + invalidation strategy
```

### Swift Concurrency race / cancellation
```
1: Identify Task boundaries (created / awaited / cancelled)
2: @MainActor vs nonisolated — any UI write off main?
3: Cancellation propagation — does inner await honour Task.isCancelled?
4: Reentrancy — re-callable while a previous await is suspended?
5 [HYPOTHESIS]: e.g. cursor advanced while previous fetch in-flight
6 [VERIFICATION]: read the type + service; confirm guard
N [FINAL]: race surface + guard/cancellation fix
```

### Navigation routing knot
```
1: Diagram screens + desired transitions
2: Locate the router/coordinator — what closures pass to children?
3: When captured? [weak self] / retain cycle?
4: Presentation order — sheet over cover? push during dismiss animation?
5 [HYPOTHESIS]: missed dismiss / animation timing / state mismatch
6 [VERIFICATION]: trace one path end-to-end
N [FINAL]: routing fix + invariant a test must protect
```

### Money / PII correctness audit  (high_rigor_domains)
```
1: What value flows here? (Decimal? Double? string from backend?)
2: Precision boundaries — where formatting/rounding happens
3: Sign / direction — refund vs charge, credit vs debit
4: Edge cases — zero, negative, very large, multi-currency
5: PII — what's logged? crosses analytics / crash reporting?
6 [HYPOTHESIS]: precision loss / leaked PII / wrong sign
7 [VERIFICATION]: run the math on paper + grep log emissions on the value
N [FINAL]: pass/fail per case + remediation
```

## Constraints
- **DO NOT** implement during thinking — just reason.
- **DO NOT** treat this as plan generation — use `ios-plan`.
- **DO NOT** dump the full chain when only a result was asked — collapse to conclusion + key reasoning.
- **Verify, don't speculate** — a claim about code is verified by the next thought (path:line).
- **Stop when done** — don't pad to hit a count.
