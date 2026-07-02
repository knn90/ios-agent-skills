# Team Execution — parallel dev agents in worktrees

Loaded by `ios-execute` for `--team N` runs. You are the **Team Lead**: spawn a dev team in
isolated git worktrees, give them context, peer-review, merge, and run the verification gate.

> **Team Lead does NOT write implementation code.** Direct, review, converge.

Generalized — **every project fact comes from `.claude/ios-profile.md`** (`source_roots`,
`test_roots`, `verify_command`, `rules_file`, `architecture`, `di`, `navigation`,
`networking`, `feature_flags`, `generated_paths`, `high_rigor_domains`). No hardcoded app
names, schemes, or simulators.

## Inputs (set by the caller before loading this file)

| Var | From | Notes |
|---|---|---|
| `N` | `--team N` / `--devs N`, else auto from the plan's `complexity` (LOW=1·MEDIUM=2·HIGH=3) | `N == 1` → use the solo path in `ios-execute` Phase 3, not this engine |
| `SLUG` | task / plan dir name | |
| `TASK_DIR` | `{plans_dir}/{SLUG}/` — holds `scope.md`, `plan.md`, `_status.md` | |
| `BASE` | **the branch the team builds on.** From `ios-resolve` = the task branch `{type}/{SLUG}`. Standalone = the current branch (else `default_base_branch`). | every dev + integrate branch is cut from this — so the merge stays within the task's lineage |
| `MODE` | `feature` (default) · `deprecation` · `flag-removal` | |

Each dev owns a **disjoint** set of files — read the **Owner** column of `plan.md`'s File
Changes table. If the plan has no Owner column, partition the files yourself (one owner each)
and record the assignment in `_status.md` before spawning.

---

## Dev agent system prompt (native isolation; fill `{…}` from profile)

Spawn each dev with `isolation: "worktree"`. Its **first action** is to create its branch.

```
You are {AGENT_NAME}, a senior iOS engineer working in this repo ({source_roots}).
Authoritative conventions: read .claude/ios-profile.md and {rules_file} first, and follow
them exactly (if ios-profile.md is absent in your worktree — it is usually gitignored — read
the main checkout's copy: $(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md) — architecture {architecture}, state {state_type}, DI {di},
navigation {navigation}, networking {networking}, localization {localization},
accessibility ids {accessibility_ids}, feature flags {feature_flags}.
Never edit anything under {generated_paths}. Money = Decimal, never Double. No PII in logs.

First action in your worktree:  git checkout -b {SLUG}/{AGENT_NAME} {BASE}
Work ONLY on the files assigned to you (your rows in the plan's File Changes table).
TDD always: write a failing test BEFORE the code for every unit of behavior you own
(RED → GREEN → REFACTOR). No implementation without a test that fails without it.
Commit with /commit (one concern per commit). Do NOT push.
Before reporting done: make your worktree COMPILE (build it) so the integrate gate isn't
tripped by typos. (The full {verify_command} test gate runs once at integrate, not per dev.)
When done, write {TASK_DIR}/{AGENT_NAME}.md and reply to Team Lead.
Await your task assignment via SendMessage.
```

If the project ships focused convention skills named in `plan.md`, tell the agent to apply
them too — but never assume any exist. The profile + `rules_file` are the floor.

## Role assignments (by MODE)

| MODE | dev-1 | dev-2 | dev-3 (only if N≥3) |
|---|---|---|---|
| **feature** | a vertical slice (e.g. view + view-model) **+ its tests** | another slice (e.g. service + networking) **+ its tests** | integration / DI wiring + edge cases **+ its tests** |
| **deprecation** | remove tests + mocks for target | remove implementation + DI registrations | remove references (imports, XIB/Storyboard, `@objc`/dynamic dispatch) |
| **flag-removal** | simplify the kept code path | remove the flag + dead code | test cleanup (drop flag-toggling cases) |

For `N == 2`, fold dev-3's column into dev-2.

> **Vertical slices + local TDD:** each dev owns a self-contained slice **and its test files**
> (per the plan's Owner column), so test and code live in the *same* worktree — real
> RED→GREEN→REFACTOR with local feedback, and the dev runs their slice's tests before reporting
> done. No dedicated test author; independent edge-case coverage comes from the dedicated
> reviewer in Phase D below.

---

## Phase A — SPAWN
Spawn `dev-1 … dev-N` via Agent `isolation:"worktree"`, **in parallel, in a single message**,
each with the system prompt above. Wait for confirmation.

The harness creates and **auto-cleans** each dev worktree. What we merge later is the **branch**
`{SLUG}/dev-N` (created by the agent's first action) — branch refs are shared across all
worktrees in the repo and persist after the worktree is gone.

## Phase B — CONTEXT
`SendMessage` each agent their slice only (don't make them re-explore):
```
## Task: {title}   Mode: {MODE}
### Your files (you own these — don't touch another agent's)
{this dev's rows from plan.md File Changes table}
### Scope (relevant slice)   {parts of scope.md touching your files}
### Plan (your steps)        {your phase steps from plan.md}
### Rules
- Follow .claude/ios-profile.md + {rules_file}. Never touch {generated_paths}.
- Commit with /commit. Build before reporting done. Do NOT push.
- Write {TASK_DIR}/{your-name}.md and reply when done.
```

## Phase C — BUILD
Monitor via `SendMessage`; resolve blockers; broker shared interface contracts between slices.
Enforce file ownership. Do **not** merge yet. When every agent reports done (worktree
compiles) and has written its `dev-N.md` → peer review.

## Phase D — PEER REVIEW (branch-based — no paths needed)

| N | Pattern |
|---|---|
| 1 | Team Lead reviews dev-1 |
| 2 | dev-1 ↔ dev-2 |
| ≥3 | round-robin: 1→2, 2→3, 3→1 |

Each reviewer diffs the author's **branch** (refs are shared, so this works from any worktree):
```bash
git diff {BASE}...{SLUG}/dev-N
```

Checklist (same lens as `ios-code-review` Stage 2):
- [ ] Matches `plan.md`; nothing outside assigned scope
- [ ] Each acceptance criterion / removal target addressed
- [ ] Tests cover new behaviour (transition holes: loading→loaded/empty/error)
- [ ] `@MainActor` on UI-touching view-models; `[weak self]` in escaping closures (no races/leaks)
- [ ] State via `{state_type}`; DI via `{di}`; navigation per `{navigation}` (no push/pop from a VM)
- [ ] No hardcoded user-facing strings (if `localization` != none); nothing under `{generated_paths}`
- [ ] Money = `Decimal`; no PII in logs (esp. `high_rigor_domains`)
- [ ] No commented-out/dead code or debug prints

### Edge-case & logic-gap review (dedicated agent — runs alongside the dev-to-dev reviews)

Because each dev writes their own tests, the independent "what did we miss?" pass lives here.
In parallel with the peer reviews, spawn **one read-only `general-purpose` Agent** as an
adversarial reviewer over the **whole feature** (all slices, not just one). Prompt:

> You are an adversarial edge-case reviewer for an iOS feature. Read `{TASK_DIR}/plan.md` +
> `scope.md`, then every dev branch's diff: `git diff {BASE}...{SLUG}/dev-N` for N = 1..{N}.
> Hunt for what the implementers (who wrote their own tests) likely MISSED — *across* slices,
> not just within one:
> - unhandled inputs: nil / empty / zero / negative / max / very-large / malformed
> - error & failure paths, cancellation, timeouts, retries, partial success
> - state-transition holes (loading → loaded / empty / error), actor reentrancy after `await`
> - boundary / off-by-one; ordering & race conditions where slices interact
> - money: rounding, sign, precision (Decimal) — if in `high_rigor_domains`
> - logic branches with NO test covering them
> Output per finding: `{ severity, file:line or area, the gap, a concrete failing scenario,
> the missing test to add, the fix }`. Be brutal; assume the implementers were optimistic.
> Do NOT repeat the style/convention findings the peer reviewers already cover.

Append findings to `{TASK_DIR}/peer-review.md` under `## Edge cases & logic gaps`. Each
**blocking** finding routes to the owning dev as a NEEDS CHANGES item — and per TDD, the fix is
**a failing test first**, then the code. This is the early, pre-merge catch; the post-merge
`ios-code-review` adversarial pass (in `ios-execute` Phase 5) stays as the final gate.

Verdicts → `{TASK_DIR}/peer-review.md`:

| Verdict | Action |
|---|---|
| **APPROVED** | → Phase E |
| **NEEDS CHANGES** | `SendMessage` the author a specific fix list (file:line + cited rule); re-review. 2 failed rounds → **BLOCKED** |
| **BLOCKED** | set `_status.md` Phase=BLOCKED, report to user, STOP |

Then **Team Lead sign-off**: diff every dev branch against `plan.md` + conventions; route final fixes.

## Phase E — MERGE (skill-managed integrate worktree)

```bash
git worktree add .worktrees/{SLUG}/integrate -b {SLUG}/integrate {BASE}
cd .worktrees/{SLUG}/integrate
git merge {SLUG}/dev-1 --no-ff -m "merge: dev-1"
git merge {SLUG}/dev-2 --no-ff -m "merge: dev-2"
# … dev-3 if N≥3
```
Conflict → set `_status.md` Phase=BLOCKED, report the conflicting files + owners, **STOP**.
(Clean file ownership from the plan's Owner column should make conflicts rare.)

## Phase F — VALIDATE (verification gate — once, post-merge)

Serialized here (never per-dev) to avoid simulator/test contention:
```
cd .worktrees/{SLUG}/integrate
{verify_command}
```
- **`verify_command` unset/empty** → build-only; state it explicitly. Never claim tests passed.
- **deprecation / flag-removal** → reference check (target must be gone):
  ```bash
  rg "{TARGET}" {source_roots}
  rg "{TARGET}" -g '*.xib' -g '*.storyboard'
  ```
  Non-empty → route to the reference-cleanup owner, re-merge, re-run.

On failure: identify the responsible branch (compiler/test path or `git blame`) → `SendMessage`
a fix request → the dev fixes on `{SLUG}/dev-N` → Team Lead re-merges that branch → re-run the
failed gate. **2 failed attempts → BLOCKED.** Read the actual output; "should pass" → run it.

## Done — hand back, do NOT review or push here

Update `{TASK_DIR}/_status.md`: `Phase: DONE`, `Status: READY`. Report to the caller:
```
✓ Team execute complete: {SLUG}
- Devs: {N}   Mode: {MODE}   Integrate branch: {SLUG}/integrate (built on {BASE})
- Peer review: APPROVED   Team Lead sign-off: APPROVED
- Edge-case review: {k} gaps found, {k} resolved (failing test added first, then fix)
- Verify: ✅ {verify_command} PASSED  (or "build-only — verify_command unset")
- Reference check: CLEAN   (deprecation/flag-removal only)
- Files: N   Tests: <n> added/updated   Reports: {TASK_DIR}/dev-1.md … dev-N.md
```
The **review gate runs in `ios-execute`**, not here — it reviews the merged work with
`ios-code-review` on `git diff {BASE}...{SLUG}/integrate` (NOT `--pending`, which can't see
committed merges), then finalises. **Never auto-merge to the real base or auto-push.**

## Artifacts

`{plans_dir}/{SLUG}/`: `_status.md`, `dev-1.md … dev-N.md`, `peer-review.md`
Branches: `{SLUG}/dev-1 … dev-N`, `{SLUG}/integrate` (these persist; dev worktrees do not).

## Cleanup (after the PR is opened, or on abort)
```bash
git worktree remove .worktrees/{SLUG}/integrate   # skill-managed — remove it
git branch -D {SLUG}/dev-1 {SLUG}/dev-2 …         # only after a successful integrate merge
```
Keep `{SLUG}/integrate` until the PR merges. **Dev worktrees: nothing to remove** — the harness
auto-cleans them (their branches already merged into integrate).
