---
name: ios-code-review
description: "Adversarial, multi-lens code review for an iOS project (Swift / SwiftUI / UIKit). Resolves a precise scope, checks spec compliance, runs a general + architecture + simplification pass, routes change-typed slices to installed specialist skills (ios-swiftui-expert / ios-concurrency-expert / ios-testing-expert / …), then an adversarial red-team, and synthesizes high-confidence findings. Reads .claude/ios-profile.md. Use for PR / commit / pending-diff review."
argument-hint: "[#PR | COMMIT | --pending | codebase]"
model: best
effort: xhigh
---

# Code Review — iOS

Adversarial, profile-driven review for native iOS Swift. **Precision over recall** — a focused list
of real, high-confidence findings beats a wall of nits.

**Principles:** YAGNI + KISS + DRY. Technical correctness over social comfort. **Honest, brutal, concise.**

> **Generated/maintained by `ios-skill-consolidate`** — distilled from the repo's prior reviewer +
> Dimillian/Skills (review-swarm, review-and-simplify-changes), Viniciuscarvalho/swift-code-reviewer-skill,
> efremidze/swift-architecture-skill (all MIT), with **Apple's Swift API Design Guidelines + the
> project's `rules_file` as the source of truth**. Last consolidated **2026-06-16**. See `SOURCES.yaml`.

## Step 0 — Load profile
Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`, `networking`,
`localization`, `accessibility_ids`, `crash_reporting`, `verify_command`, `high_rigor_domains`,
`generated_paths`, **`specialists`**, `rules_file`, `test_roots`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing → run `ios-project-init`.

---

## Phase 0 — Resolve scope (do this FIRST; everything keys off it)

Build ONE canonical `scope` and **print a banner before any finding** — if the scope is wrong, every
finding is suspect.

| Input | Mode | Diff |
|---|---|---|
| `#123` / PR URL | PR | `gh pr diff 123` (base via `gh pr view --json baseRefName`) |
| `abc1234` (7+ hex) | Commit | `git show <sha>` |
| `--pending` | Pending | `git diff` + `git diff --cached` |
| *(no args, recent edits)* | Default | files edited this session |
| `codebase` | Codebase | full scan |

- **Base-branch fallback:** `gh pr view --json baseRefName` → `git rev-parse origin/HEAD` → `origin/main`|`origin/master`.
- **Buckets:** `modified` (review in full, all severities) · `tests-for-modified` (coverage → main report) · `related/context` (read for context only) · `deleted` (spec reasoning only).
- **Exclude always:** `generated_paths`, Pods/Carthage/.build/DerivedData/.swiftpm, `*.generated.swift`/`*.pb.swift`.
- **Adjacent quarantine:** issues in context-only files go in an "Adjacent observations" section and **do NOT count** toward the verdict.
- **Banner:** `Scope: PR #123 · base: main · modified: 7 · tests: 2 · HIGH-RIGOR: yes`.
- **Adaptive depth:** tiny diff (≤2 files, ≤30 lines, not HIGH-RIGOR) → review locally, skip the parallel fan-out.

**HIGH-RIGOR:** diff touches any domain listed in the profile's `high_rigor_domains` → adversarial pass + correctness audit are **mandatory**; log `[HIGH-RIGOR]`. (The project defines its own domains via the profile; the skill assumes none.)

---

## Stage 1 — Spec compliance (skip if no plan/spec)
Read the plan (`{plans_dir}/{slug}/plan.md`) and/or the PR/issue (`gh pr view --json title,body,closingIssuesReferences`). Emit a **requirement-coverage table**:

| Requirement / AC | Status | Where |
|---|---|---|
| <criterion> | ✅ met / ⚠️ partial / ❌ missing | `file:line` |

Flag **scope creep** (unrelated refactors bundled in) and **missing work** (`TODO`, `fatalError("unimplemented")`, empty bodies, stubbed mocks). No spec available → mark "not assessed" (don't invent one). **Stage 1 FAIL → stop, report, ask.**

---

## Stage 2 — Multi-lens review (run lenses in parallel for non-trivial diffs)

### 2.0 — Specialist routing (FIRST — this is the point)
For each domain the diff touches, route it to the matching specialist below **if that specialist skill
is installed** in this project — **on by default, no profile entry needed**. Spawn a **read-only
`general-purpose` Agent** that loads the specialist's `SKILL.md` and reviews **only its slice** of the
diff; the general lens (2.1) then skips that domain. The profile's `specialists:` is an **optional
override**: if set, restrict routing to that list; `specialists: none` turns routing off.

| Diff signal | Route to (if installed) | Reviews |
|---|---|---|
| `View`, `@State`/`@Observable`/`@Binding`, `body: some View`, `.task{}`, nav/layout modifiers | **ios-swiftui-expert** | view composition, state ownership, perf, navigation, a11y, Liquid Glass |
| `async`/`await`, `actor`, `Sendable`, `@MainActor`, `Task`, `nonisolated`, `AsyncSequence` | **ios-concurrency-expert** | data races, actor reentrancy, Sendable, structured concurrency, Swift 6 isolation |
| `import Testing`, `@Test`, `#expect`/`#require`, `XCTestCase`, `XCTAssert`, files under `test_roots` | **ios-testing-expert** | Swift Testing / XCTest idioms, coverage, flakiness, parallelization |
| new/changed types, protocols, services, view-models; DI / composition wiring; large classes; cross-layer changes | **ios-solid-expert** | SOLID (SRP/OCP/LSP/ISP/DIP), decoupling (Decorator/Composite/Adapter), composition-root DI, framework isolation |
| *(future specialists, e.g. networking/persistence)* | **ios-`<domain>`-expert** | its domain |

Specialist agent prompt:
```
Load .claude/skills/<specialist>/SKILL.md and apply it. Review ONLY the <domain> aspects of this diff:
<the relevant hunks>. Skip what a general reviewer covers (naming, layering, localization).
Output each finding as { severity, file:line, problem, fix, confidence: high|med|low }. Be specific; cite the rule.
```
No specialists installed / none match → skip 2.0 (the general lens still covers the floor).

### 2.1 — General iOS lens (always; the floor when a specialist isn't installed)
Spawn a `general-purpose` Agent (fill `{…}` from profile). **Authoritative: `rules_file` + Apple's Swift API Design Guidelines** — read them.

```
Review this iOS Swift diff with senior rigor. Flag { severity, file:line, problem, fix, confidence }.

THREADING & CONCURRENCY (highest-value bugs — unless ios-concurrency-expert handled it):
- @MainActor on UI-touching view-models; published writes land on main; flag Task.detached / nonisolated(unsafe) writes to UI state.
- actor reentrancy: re-check state after every await. [weak self] in escaping closures. No blocking the cooperative pool (semaphore/group .wait).
- off-main completion callbacks (URLSession, notifications, Timer) mutating UI without a main hop.
MEMORY & LIFETIME: retain cycles, unbounded caches, leaked observers/timers, AnyCancellable dropped.
SECURITY & PRIVACY: token leakage; Keychain (not UserDefaults) for credentials; missing ATS; PII in logs/analytics/{crash_reporting}/print.
MONEY (if a money/finance domain is configured in high_rigor_domains): Decimal not Double; rounding; sign errors; currency parsing.
ACCESSIBILITY: VoiceOver labels/order, Dynamic Type, contrast (unless ios-swiftui-expert handled it).
HYGIENE: comments WHY-only; no dead/commented code; no back-compat shims for code this diff removed.

PROFILE-GATED (run only those that apply):
- state_type != none → uses {state_type}; transition holes (loading→loaded/empty/error) covered.
- di != manual-init → injected via {di}; no singleton reach-in from view-models.
- navigation coordinator → nav closures from coordinator→VM; VMs don't push/pop.
- networking != none → DTO→domain mapping at the boundary; UI doesn't import generated types.
- localization != none → no hardcoded user-facing strings; new keys added; generated catalogs untouched.
- accessibility_ids != none → tested UI exposes an id from {accessibility_ids}.
- Always: never edit generated_paths.
```

### 2.2 — Architecture lens (profile-gated)
Check against the project's `architecture`: **dependency direction** (View → VM → UseCase/Repo; Model independent), **single source of truth** (no `@State` mirroring VM state), **God view-model / massive reducer** (extract UseCases), **presentation isolation** (no UIKit import in a VM; navigation as value types), **stale-async overwrite / missing cancellation**, business logic living in Views. Align to the *existing* pattern — don't propose an architecture switch for a small change. (When `ios-solid-expert` is installed, 2.0 already covered the SOLID/decoupling depth; this lens then focuses on pattern-fit + the project's `architecture`, not re-deriving the principles.)

### 2.3 — Reuse & simplification lens
Flag both directions: **over-engineering** (duplicated logic that an existing helper covers; redundant/derived state stored; parameter sprawl; needless abstraction/indirection; stringly-typed where an enum exists) **and over-simplification** (distinct concerns collapsed into one unclear unit). Only findings that materially improve maintainability/correctness/cost — never churn for style. (If the repo has a `simplify` skill, this lens can defer the *applying* to it; here it only flags.)

---

## Stage 3 — Adversarial red-team (always-on except trivial; HIGH-RIGOR never skips)
Try to **break** the change across four lenses (parallel agents for large diffs). Define regression relative to the **stated intent** (what should change vs what must stay unchanged):

1. **Intent & regression** — behavior outside stated scope; broken edge/fallback paths; contract drift between callers & callees; adjacent flows that should have changed together but didn't.
2. **Security & privacy** — authn/authz gaps, unsafe input, injection, secret/token exposure, risky defaults, trust of unverified data, PII leakage.
3. **Performance & reliability** — duplicate work / redundant I/O; new work on hot paths (startup/render/request); leaks, retry storms, subscription drift; ordering/race/cancellation brittleness; image decode on main.
4. **Contracts & coverage** — API/schema/type/flag mismatches; migration & back-compat fallout; missing/weak tests for changed behavior; **detectability** (would a future regression even be observable — logs/metrics/assertions?).

Each finding: `{ severity, file:line, problem, exploit/scenario, fix, confidence }`.

---

## Synthesis (precision over recall)
Treat all lens output as raw input, then:
- **Dedup** across lenses; **drop** weak/speculative/style-only items and anything that conflicts with the stated intent.
- "May be wrong but intent unclear" → a **question**, not a finding.
- **Normalize:** `{ file:line, category (spec|concurrency|swiftui|testing|architecture|security|perf|contracts|simplification|hygiene), severity, why it matters, fix, confidence }`.
- **Order:** high-severity high-confidence first. If nothing material, **say so** — don't manufacture feedback.

## Agent-loop feedback (self-improving rules)
Group findings by rule. A rule firing **≥2×** across the diff = a recurring pattern → propose a one-line **directive** for the project's `rules_file` (e.g. "Always inject `URLSession` via init, never `.shared` in view-models"). Especially useful for AI-generated diffs — it turns repeated misses into standing rules. (Propose only; don't edit `rules_file` without approval.)

## Verification gate
**Iron law: no "review passed" without fresh verification.** Tests/build → run `{verify_command}` (unset → build-only, say so). Bug-fix → the original reproducer no longer reproduces. Stop if you catch yourself thinking "should pass" — run it.

## Severity model
- **Critical** — crash · data race · state corruption · `Decimal` precision loss in money · credential leak · PII in logs → **must fix before merge**.
- **Important** — state-transition hole · layer/dependency violation · memory leak · missing test for changed behavior · missing a11y id on tested UI · missing localization key → **fix before merge or file a ticket**.
- **Nit** — style, naming, API cleanliness → author's call.

## Report format
```markdown
Scope: <banner>
# Code Review — <mode>   HIGH-RIGOR: yes/no   Verdict: APPROVED / CHANGES REQUESTED / BLOCKED

## Summary   Critical: N · Important: N · Nit: N
## Stage 1 — Spec       <coverage table> · scope-creep: <…>
## Findings (by severity)   [Sev] category — file:line — problem → fix (confidence)
   - Specialist findings grouped under their sub-heading (ios-swiftui-expert / …)
## Agent-loop feedback   <recurring pattern → proposed rules_file directive>  (or none)
## Adjacent observations (out of scope, not counted)
## Critical open items · Recommended follow-ups
```

## Constraints
- **Read-only review** — find and recommend; do NOT apply fixes (that's `ios-execute` / a `simplify` skill).
- **DO NOT** approve while a Critical remains, or skip Stage 3 (unless trivial & non-HIGH-RIGOR).
- **DO NOT** rely on memory — read the actual diff + `rules_file`. Specialists win on their domain.
- **MUST** print the scope banner, record adjudication reasoning, and carry confidence end-to-end.
