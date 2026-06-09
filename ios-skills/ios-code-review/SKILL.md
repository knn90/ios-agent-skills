---
name: ios-code-review
description: "Adversarial code review for an iOS project (Swift / SwiftUI / UIKit). 3-stage protocol: spec compliance → code quality (universal iOS checklist + profile-gated architecture checks) → adversarial red-team. Always-on adversarial + correctness audit for high_rigor_domains. Delegates to specialist skills if installed. Reads .claude/ios-profile.md."
argument-hint: "[#PR | COMMIT | --pending | codebase]"
---

# Code Review — iOS

Adversarial review with technical rigor for native iOS Swift code. Every review covers
spec compliance, code quality, and an adversarial red-team pass.

**Principles:** YAGNI + KISS + DRY. Technical correctness over social comfort. **Honest, brutal, concise.**

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility_ids`, `crash_reporting`, `verify_command`,
`high_rigor_domains`, `generated_paths`, `specialists`, `rules_file`. If missing → run
`ios-project-init`.

---

## Input Modes (auto-detect; ask via `AskUserQuestion` if ambiguous)

| Input | Mode | Resolved diff |
|---|---|---|
| `#123` / PR URL | PR | `gh pr diff 123` |
| `abc1234` (7+ hex) | Commit | `git show <sha>` |
| `--pending` | Pending | `git diff` (staged + unstaged) |
| *(no args, recent edits)* | Default | recent edits this session |
| `codebase` | Codebase | full scan |

## HIGH-RIGOR detection
Diff touches any `high_rigor_domains` (default keywords below) → **adversarial review
mandatory + correctness audit**. Log `[HIGH-RIGOR]` in the report header.

| Domain | Typical files / patterns |
|---|---|
| Checkout / Cart | `Checkout`, `Cart`, `LineItem`, `Total`, `Fee`, `Tax`, `Promo`, `Discount` |
| Payment / Wallet | `Payment`, `Card`, `Wallet`, `Refund`, `Charge`, `Currency` |
| Auth | `Login`, `Logout`, `Session`, `Token`, `Biometric`, `Keychain`, `OAuth`, `2FA` |
| PII / Account | `Profile`, `Address`, `OrderHistory`, `Email`, `Phone`, `KYC` |
| Money math | `Decimal`, currency formatting/conversion |

(Use the project's actual `high_rigor_domains` list; the above are sensible defaults.)

---

## Three-Stage Protocol

### Stage 1 — Spec Compliance (skip if no plan/spec exists)
- Matches `{plans_dir}/{slug}/plan.md` + phase files?
- Missing acceptance criteria? Unjustified extras (scope creep)?
- Feature flag wired (if `feature_flags`)? Localization keys exist (if `localization`)?
- Accessibility id exposed for new UI used in tests (if `accessibility_ids`)?

**MUST pass before Stage 2.** Fail → output failures, stop, ask user.

### Stage 2 — Code Quality

#### 2.0 — Specialist routing (first)
For each installed specialist in `specialists`, if the diff matches its signal, delegate
that slice of the review to it (it knows nuances the general lens misses). Spawn one
`general-purpose` Agent per match (parallel), telling it to load that skill's SKILL.md and
review only its speciality, output `{ severity, file:line, problem, fix }`, and skip
findings the general reviewer covers (architecture, localization, accessibility, DI).

| Signal in diff | Delegate to (if in `specialists`) | Covers |
|---|---|---|
| `async`/`await`, `Task`, `actor`, `@MainActor`, `Sendable`, `nonisolated`, `AsyncSequence` | `swift-concurrency` | data races, actor isolation, Sendable, Swift 6 |
| `View`, `@State`, `@StateObject`, `@Observable`, `body: some View`, `.task{}`, nav modifiers | `swiftui-expert` | view composition, state lifecycle, perf |
| `import Testing`, `@Test`, `#expect`, `@Suite`, or changes under `test_roots` | `swift-testing-expert` | macros, traits, parameterized tests |
| `*.graphql`, network codegen, client/query/mutation (only if `networking: Apollo-GraphQL`) | `apollo-ios` | codegen, cache keys, watchers, interceptors |

If none match or none installed → skip 2.0.

#### 2.1 — General iOS lens
Spawn a `general-purpose` Agent with this prompt (fill placeholders from profile):

```
Review this iOS Swift diff with Copilot-level rigor.
Authoritative rules: {rules_file} + {docs_root}/* (read them).
Flag every Critical / Important / Nit as { severity, file:line, problem, fix }.

── UNIVERSAL iOS CHECKS (always on, every project) ──

Threading & Concurrency Safety (the highest-value class of bugs):
1. Every ObservableObject/@Observable view-model that touches UI is @MainActor; every
   published write lands on main. Flag nonisolated(unsafe)/Task.detached writes to UI state.
2. Type-level isolation, not scattered per-property @MainActor (per-property splits break Sendable).
3. MainActor.assumeIsolated only when an SDK contract guarantees main-thread delivery — it
   crashes in release if wrong. Prefer await MainActor.run when the path is reachable.
4. nonisolated on delegate callbacks that arrive off-main (CLLocationManagerDelegate,
   URLSessionDelegate, etc.), with an explicit hop to main inside.
5. Task.detached only for genuinely independent work. Flag any Task.detached touching UI /
   published state / cancellation-sensitive state — use plain Task {} to inherit context.
6. Tasks in .task{} auto-cancel on disappear (prefer it). Tasks in .onAppear must be stored
   and cancelled in .onDisappear.
7. Cancellation cooperation in async loops — at least one try Task.checkCancellation() or
   await Task.yield() per iteration (esp. paginated fetches).
8. No blocking primitives bridging async→sync (DispatchSemaphore.wait, DispatchGroup.wait,
   RunLoop.run(until:)) — they deadlock the cooperative pool.
9. Actor reentrancy — every await invalidates prior reads; re-check state after await if a
   later decision depends on it (the "check balance → await → debit" bug).
10. New cross-boundary types are Sendable. @unchecked Sendable ONLY with a comment stating
    the manual invariant.
11. [weak self] in every escaping closure capturing self unless lifetime is provably tied.
12. Capture stable references BEFORE await if the property may be mutated during suspension.
13. Combine AnyCancellable stored in a Set — flag any .sink whose return is dropped.
14. Network completion callbacks may fire off-main — flag UI/state mutation inside them
    without a @MainActor hop. (URLSession, WKWebView nav delegate, NSNotification, Timer.)
15. Heavy work (image/JSON decode, regex compile, Decimal formatting in tight loops) on the
    main actor — flag if not offloaded.

Memory & lifetime: retain cycles, unbounded caches/lists, leaked observers/timers.

Security & privacy: token leakage; Keychain misuse; missing ATS; PII in UserDefaults;
missing biometric protection; PII written to logs/analytics/{crash_reporting}/print.

Money correctness (if high_rigor_domains include money): Double anywhere money flows (must
be Decimal); rounding direction; sign errors (refund vs charge); currency parsing from backend.

Accessibility: VoiceOver order, Dynamic Type clipping, contrast, meaningful labels.

General hygiene: comments WHY-only (no history/ticket refs/paraphrase); no dead/commented-out
code; no backwards-compat shims for code this diff itself removed.

── PROFILE-GATED CHECKS (run only the ones that apply) ──

If state_type != none:        state uses {state_type}; no ad-hoc enums; transition holes
                              (loading→loaded/empty/error) covered.
If di != manual-init:         dependencies injected via {di}; flag global-singleton access
                              from a view-model where DI exists.
If navigation is coordinator: navigation closures passed from coordinator → view-model;
                              flag view-models that push/pop directly.
If protocol-backed VMs are    every view-model/service has a protocol for mockability.
the convention:
If networking != none:        operations in the right module; DTO→domain mapping at the
                              boundary; UI doesn't import generated network types directly;
                              (cache/watcher lifecycle if the client caches).
If localization != none:      no hardcoded user-facing strings; new keys added to the catalog;
                              generated localization files never edited.
If accessibility_ids != none: UI referenced by UI tests exposes an id from {accessibility_ids};
                              list items use stable ids (not array index).
Always:                       never edit anything under generated_paths.

Severity guide:
- Critical — data race / crash / state corruption / Decimal precision loss in money path /
  credential leak / PII in logs
- Important — state-transition hole, layer violation, memory leak, missing accessibility id
  on a tested element, missing localization key
- Nit — style, API cleanliness, naming
```

Aggregate specialist findings under a per-speciality sub-section.

### Stage 3 — Adversarial Review (always-on, except trivial)
Skip only if ≤ 2 files AND ≤ 30 lines AND **not** HIGH-RIGOR. HIGH-RIGOR → never skip.

Spawn a `general-purpose` Agent as hostile reviewer:
```
You are a hostile reviewer. Try to break this iOS Swift code.
Diff: <diff>   Context: HIGH-RIGOR <yes/no>, domain <…>

Find:
1. Security holes — token leakage, Keychain misuse, missing ATS, PII in UserDefaults, missing biometric.
2. False assumptions — "never nil/empty/zero/negative", "won't race".
3. Resource exhaustion — unbounded retries/leaks, image decode on main, unbounded list growth.
4. Races & threading — concurrent mutation, async ordering, re-entry on tap, suspended-task
   handoff, actor reentrancy, Task.detached touching UI, assumeIsolated without guarantee,
   off-main callback writes to UI, blocking the cooperative pool.
5. Supply chain — new deps, transitive risk, unpinned versions.
6. Observability gaps — silent catch, no crash breadcrumb on critical paths, swallowed errors.
7. Networking correctness (if networking != none) — nullable-vs-required mismatch, wrong
   cache key, watcher leaks, stale data after partial update.
8. Money correctness — Double in money flow, rounding, sign, currency parsing.
9. PII — anything to logs/analytics/{crash_reporting}/print; PII in snapshot-test screenshots.
10. Rollback safety — feature flag default-off? kill switch? backend change additive?
11. iOS-specific — deep links, Universal Links, scene lifecycle, background URL session.
12. Accessibility — VoiceOver order, Dynamic Type clipping, contrast.

Output per finding: { severity, file:line, problem, exploit/scenario, fix }. Be brutal.
```

Adjudicate: Critical → must fix before merge · Important → fix or document deferral with a
ticket · Nit → user choice.

---

## Verification Gate
**Iron law:** no "review passed" claim without fresh verification.
- Tests/build → run `{verify_command}` (unset → build-only; say so).
- Bug fixed → original reproducer no longer reproduces.
- Spec met → each AC mapped to changed code.

Stop if you think "should pass" / "probably fine" → run it, read output, then claim.

## Report Format
```markdown
# Code Review — <input mode>
**Diff scope**: <files>   **HIGH-RIGOR**: yes/no   **Stages**: spec / quality / adversarial

## Stage 1 — Spec Compliance     Status: PASS / FAIL
## Stage 2 — Code Quality
### 2.0 Specialist findings  (only sub-sections that fired)
### 2.1 General iOS lens      Critical / Important / Nit
## Stage 3 — Adversarial      | # | Severity | File:line | Issue | Fix |
Adjudication: Accept N · Defer N (tickets) · Reject N (reasoning)

## Verdict   APPROVED / CHANGES REQUESTED / BLOCKED
## Critical Open Items
## Recommended Follow-ups
```

## Constraints
- **DO NOT** approve without Stage 3 (unless trivial non-HIGH-RIGOR).
- **DO NOT** mark APPROVED while Critical findings remain.
- **DO NOT** rely on memory — read the actual diff + `rules_file`.
- **MUST** flag every hardcoded user-facing string (if `localization` != none) and every
  edit to `generated_paths`.
- **MUST** record adjudication reasoning per finding. Be brutal, specific, useful.
