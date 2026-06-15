---
name: ios-execute
description: "ALWAYS activate before implementing ANY feature, plan, or fix in an iOS project. The only implementer. Enforces plan-first, test-first (TDD always), verify-before-claim (runs the project's verify_command), and a mandatory code-review gate. Solo by default; --team N spawns parallel dev agents in isolated worktrees with peer review + merge. Heightened rigor for high_rigor_domains (checkout/auth/payment/PII). Reads .claude/ios-profile.md."
argument-hint: "[task | plan-path | TICKET-ID] [--fast | --auto | --no-test | --team N | --solo]"
---

# Execute — iOS implementation

End-to-end implementation skill (**the only implementer** in the suite). Enforces plan-first
execution, follows the project's architecture, runs `verify_command` before claiming done.
Solo by default; scales to a parallel dev team in isolated worktrees with `--team N`.

**YAGNI + KISS + DRY.** Token efficiency. Concise reports.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility_ids`, `feature_flags`, `crash_reporting`,
`verify_command`, `high_rigor_domains`, `generated_paths`, `plans_dir`, `rules_file`,
ticket fields. If missing → run `ios-project-init`.

## When to Use
Before writing **any** implementation code (feature, fix, refactor); translating an
approved plan into code; resolving a ticket end-to-end.

## When NOT to Use
| Case | Use instead |
|---|---|
| Need trade-off discussion | `ios-brainstorm` |
| Need a written plan first | `ios-plan` |
| Just find files | `ios-scout` |
| Library docs | `WebFetch` against the official URL |
| Trivial fix (< 20 lines, single file) | Direct edit |

---

## HARD-GATE: Plan-First
```
Do NOT write implementation code until a plan exists and has been reviewed.
Exceptions:
- --fast — skips research but still does scout → micro-plan → code
- User explicitly says "just code it" / "skip planning" — respect the override
"Simple" tasks are where unexamined assumptions waste the most time.
```

| Tempting | Reality |
|---|---|
| "Too simple to plan" | 30-sec plan prevents 30-min rework |
| "I know how to do this" | Knowing ≠ planning. Write the file paths down. |
| "I'll plan as I go" | That's hoping, not planning |

## Argument Parsing
- **Plan path** (`{plans_dir}/.../plan.md`/`phase-*.md`) → execute plan mode.
- **TICKET-ID** → `ticket_fetch` → matching plan, else `ios-plan TICKET-ID`.
- **Free-form** → interactive.
- **Flags:** `--fast` (scout→micro-plan→code), `--auto` (auto-approve gates, sparingly),
  `--no-test` (skip the final verify *run* only — you'll run it yourself; TDD still drives the code; record in report),
  `--team N` (parallel dev team in worktrees — see Phase 3), `--solo` (force single-agent),
  `--devs N` (alias for `--team N`).

If no args, ask via `AskUserQuestion` (what to implement + mode).

---

## Process Flow
```
1. Parse args
2. Resolve plan (load OR ios-plan creates OR --fast micro-plan)
3. Plan Review Gate — user approval (skip only with --auto)
4. Domain rigor flag — touches high_rigor_domains? → HIGH-RIGOR
5. Implementation — test-first (TDD); solo (direct edits / one subagent) OR --team N (worktree dev team)
6. Verification — {verify_command} (MANDATORY)
7. Code Review — ios-code-review --pending (MANDATORY)
8. Finalise — update plan status, ask before commit
```

## Phase 1 — Load context
Resolve the plan (plan path → Read; ticket → fetch + find/create plan; free-form → trivial
inline plan or `ios-plan`). Load `rules_file` + relevant `docs_root`.

**Domain rigor flag:** if the task touches any `high_rigor_domains` → mark **HIGH-RIGOR**:
- adversarial review **mandatory** at the review gate
- `Decimal` for monetary values, never `Double`
- feature flag default-off + gradual rollout (if `feature_flags` != none)
- no PII in logs / analytics / `crash_reporting` breadcrumbs

## Phase 2 — Plan Review Gate
User approval required (skipped only with `--auto`). Present: files to change (path:line),
phases, test strategy (UI accessibility ids?), feature flag, HIGH-RIGOR flag.
`AskUserQuestion`: Approved → proceed · Revise → regenerate · Abort → stop.
**Do NOT start implementing without approval.**

## Phase 3 — Implementation

**Pick the execution path by plan size** (its complexity / phase count):

| Plan size | Path | How |
|---|---|---|
| trivial / small | **direct** | Edit/Write yourself |
| 1 feature, 5+ files | **delegated** | one `general-purpose` Agent with the plan + `rules_file` |
| large / parallelizable, or `--team N` | **team** | N dev agents in isolated worktrees + peer review + merge → **`references/team-execution.md`** |

**Auto dev count** (when none of `--team N` / `--devs N` / `--solo` is given): read the plan's
`complexity` field (set by ios-plan) → **LOW = solo · MEDIUM = 2 · HIGH = 3**. `--solo` forces 1; an explicit
`--team N` / `--devs N` always wins. For team runs, **load and follow
`references/team-execution.md`** (spawn → context → build → peer review → merge → validate);
it reuses the same profile + discipline below. The discipline applies to **every** path —
your own edits *and* the dev agents' work.

**Test-First (TDD) — always.** Drive every unit of behavior with a test:
**RED** (write a failing test) → **GREEN** (minimum code to make it pass) → **REFACTOR** (tidy,
tests stay green). Never write implementation before a test that fails without it. This holds in
**solo and team** mode, **greenfield and existing** code (greenfield: the first test also
establishes the test pattern). The only exemptions are pure non-logic changes — copy, assets,
config, formatting — and you must say so in the report. If a change is hard to test, that's a
design signal: fix the seam (inject the dependency, split the type), don't skip the test.

**Discipline (enforce `rules_file` + profile conventions while editing):**
- **State** — use `{state_type}`. No ad-hoc one-off enums when a shared type exists.
- **DI** — inject via `{di}`. Don't reach for global singletons from a view-model when DI exists.
- **Strings** — if `localization` != none, every user-facing string via the localization
  mechanism. Never hardcode. Never edit generated localization files.
- **Navigation** — follow `{navigation}` (e.g. closures from coordinator → view-model;
  view-models don't push/pop directly).
- **Protocols** — view-models/services have a protocol for mockability (if that's the convention).
- **Accessibility** — every UI surface used in tests gets an id from `accessibility_ids`.
- **Generated code** — never edit anything under `generated_paths`.
- **Comments** — WHY-only. No history, no "added for ticket X", no paraphrasing the code.

**Commit cadence — one concern per commit, via `/commit`** (which extracts the ticket id
from the branch). Commit at each milestone (phase done, self-contained refactor, new
types/DI wiring, new localization keys, new network operation + codegen, tests, snapshot
updates). Run relevant tests before each commit — never commit a broken build mid-feature.
Always use `/commit`, not raw `git commit`.

**Tracking:** multi-phase work → `TaskCreate`/`TaskUpdate`; mark `in_progress` before
starting, `completed` when done, don't batch. Update plan checkboxes inline with commit shas.

## Phase 4 — Verification Gate (MANDATORY)
**Iron law: no completion claim without fresh verification.**

Run the profile's command:
```
{verify_command}
```
Read the actual output. Claim done only on success.

- **`verify_command` unset/empty** → run a **build-only** check (e.g. `xcodebuild build`)
  and state explicitly in the report: "verify_command not configured — build-only, tests
  not run." Don't pretend tests passed.
- **UI work** → also exercise the change in the simulator (`/run` or `/verify` skill) —
  type-checking verifies code, not feature correctness.
- User said "I'll test myself" → respect, but **say so explicitly** and skip running it.

**Red flags — stop if you think** "should pass" / "probably fine" / "seems to work" → run
it, read the output, *then* claim.

## Phase 5 — Code Review Gate (MANDATORY)
Run `ios-code-review --pending`. **Team mode:** the work is committed on `{SLUG}/integrate`, so
review the merged diff instead — `ios-code-review` on `git diff {BASE}...{SLUG}/integrate`
(`--pending` can't see committed merges). For HIGH-RIGOR tasks, adversarial review is mandatory
(the review skill runs it automatically). Resolve all **Critical** before proceeding;
**Important** → fix or document deferral with a ticket.

## Phase 6 — Finalise (only after 4 + 5 pass)
- Update `plan.md` frontmatter `status: completed`; tick remaining checkboxes.
- Update docs if implementation changed architecture/conventions (light edits OK; rewrites
  need approval).
- **Plan lifecycle:** the plan stays on the branch during dev. If your PR flow removes it
  before opening the PR, that's the PR step's job — not here. Mention `ios-plan archive`
  if worth keeping as a decision record.
- **Final commit:** most work is already committed via granular `/commit`. If the tree is
  dirty (checkbox/status/doc tweaks), `/commit` once more. **Do NOT push** without explicit
  instruction.

Final report (solo; team runs add peer-review/merge lines per `references/team-execution.md`):
```
✓ Execute complete: <task slug>
- Plan: {plans_dir}/{slug}/plan.md → status: completed
- Files changed: N (paths…)
- Tests: <n> added, all passing   (or "build-only — verify_command unset")
- Verify: ✅ PASSED / build-only / user-tested
- Review: passed (X critical resolved, Y important deferred)
- Commit: <sha or pending user action>
- Open follow-ups: <list or none>
```

## Greenfield note
First feature establishes the pattern — there's no prior art to match. Be deliberate about
the conventions you set (they're what every later run inherits). Once a reference feature
and a real `verify_command` exist, full rigor resumes.

## Constraints
- **DO NOT** start coding without an approved plan. **DO NOT** auto-commit/push without approval.
- **DO NOT** write implementation before a failing test (TDD) — except pure non-logic changes (state which).
- **DO NOT** skip the verify gate (Phase 4) or the review gate (Phase 5).
- **DO NOT** edit `generated_paths`. **MUST** follow `rules_file`.
- **Correctness for money** — `Decimal` only in `high_rigor_domains`. **No PII in logs.**
- **Sacrifice grammar for concision** in reports.
