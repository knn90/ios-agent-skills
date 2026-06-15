# Full-Flow iOS Workflow — Design & Implementation Plan

**Status:** v1 BUILT (2026-06-15) — decisions locked (see §6)
**Goal:** One entry point that takes a *ticket or free-form context* and drives it end-to-end:
**scout → plan → implement (parallel team) → test → review → PR**, project-agnostic (profile-driven).
**Decision taken:** generalized suite (`ios-skills/`) + port a proven team-execution engine from a prior project-specific suite.

---

## 1. What already exists (reuse — do NOT rebuild)

The generalized `ios-skills/` suite already covers most of the flow. Every skill reads `.claude/ios-profile.md`.

| Stage | Existing skill | Produces |
|---|---|---|
| Scout | `ios-scout` | a *map* (paths:line) — parallel Explore agents |
| Plan | `ios-plan` | `plan.md` + per-phase files under `plans_dir`, approval-aware |
| Implement | `ios-execute` | code; plan-first → verify → review gates; **solo only today** |
| Review | `ios-code-review` | 3-stage adversarial review + verdict |
| Research / ideate / reason | `ios-research`, `ios-brainstorm`, `ios-sequential-thinking` | reports / reasoning |
| Bootstrap | `ios-project-init` | the `ios-profile.md` itself |

**Source reference** (a prior project-specific suite, NOT reused directly): it had a proven end-to-end ticket→PR chain and a parallel `init → scope → plan → execute` team engine. We port the *logic*, generalized.

---

## 2. The gap (what we build)

1. **No orchestrator** chaining ticket → scout → plan → execute → review → PR.
2. **`ios-execute` implements solo** — no parallel team engine (worktree dev agents + peer review + merge + validation gate).

So we build **two things**: an orchestrator, and a team-execution mode for the implementer.

---

## 3. Design

### 3.1 New skill — `ios-resolve` (orchestrator)

> Name: **`ios-resolve`** (chosen — works for "resolve ticket" *and* free-form).

```
ios-resolve <TICKET-ID | "free-form description">
            [--devs N] [--base BRANCH] [--solo] [--skip-approval] [--no-pr]
```

Pipeline (each step delegates to an existing skill; orchestrator only sequences + gates):

```
0. Load .claude/ios-profile.md  (missing → ios-project-init)
1. Resolve input:
     TICKET-ID (matches ticket_pattern) → fetch via ticket_fetch
     free-form                           → use as task description
2. INIT      → create branch {type}/{slug} off {base}; create plans_dir/{slug}/ + _status.md
3. SCOUT     → ios-scout  → writes scope.md
4. PLAN      → ios-plan   → writes plan.md   ── APPROVAL GATE (unless --skip-approval)
5. IMPLEMENT → ios-execute (plan-path) [--team N | --solo]  → code + verify + review
6. COMMIT    → /commit (granular; ticket id from branch)
7. PR        → gh pr create --base {base}   (skip if --no-pr or no gh)
8. TICKET    → best-effort transition (e.g. → CODE REVIEW) if ticket_system supports it
9. REPORT    → summary (branch, PR url, files, tests, review verdict)
```

Branch type from ticket issue-type (Bug→`fix`, Story→`feat`, Task→`chore`) or inferred from description; default `feat`.

### 3.2 Extend the implementer — `ios-execute --team N` (renamed from `ios-execute`)

The implementer is **renamed `ios-execute` → `ios-execute`** and gains a team mode, honoring the suite principle *"one implementer."* Today Phase 3 is: direct edits, or delegate one subagent for big features. We add a third path:

| Plan size | Path |
|---|---|
| trivial / small | direct Edit/Write (today) |
| medium (1 feature, 5+ files) | one delegated subagent (today) |
| **large / parallelizable** | **`--team N`: N dev agents in isolated worktrees + peer review + merge + validation** (new) |

The team protocol is large, so it lives in **`ios-execute/references/team-execution.md`** (loaded only for `--team` runs), keeping the main SKILL.md readable. Protocol:

```
Phase A SPAWN   — N Agent(isolation:"worktree"), dev-1..N (parallel, single message)
Phase B CONTEXT — SendMessage each: their slice of scope.md + plan.md + role + rules_file
Phase C BUILD   — devs implement on their branch; dev-1 = tests-first (TDD); file ownership enforced
Phase D REVIEW  — peer round-robin (1←2, 2←3, 3←1); fix-loop; lead sign-off → peer-review.md
Phase E MERGE   — .worktrees/{slug}/integrate; git merge --no-ff each dev branch
Phase F VALIDATE— run verify_command in integrate worktree; route failures back; 2-strike → BLOCKED
```

Role table (generalized; modes = feature / deprecation / flag-removal):

| Mode | dev-1 | dev-2 | dev-3 (if N≥3) |
|---|---|---|---|
| feature | tests first (TDD) | core implementation | integration / DI / edge cases |
| deprecation | remove tests + mocks | remove implementation + DI | remove references (incl. XIB/Storyboard/@objc) |
| flag-removal | simplify kept path | remove flag + dead code | test cleanup |

### 3.3 Profile changes — **minimal** (push back on over-spec)

A sweep of the source engine found ~20 hardcoded project-specific facts. **Almost all are already covered** in the generalized model — adding fields for them would violate the suite's "facts that vary, nothing else" rule.

| Source-project hardcode (example) | Generalized source — **no new field** |
|---|---|
| `xcodebuild ... -scheme <App> ... -destination '<simulator>'` (build + test gates) | **`verify_command`** (already the single source of truth) |
| a design-system module, logging wrapper, DI container, view-model base protocol, crash reporter, async framework | **`rules_file`** + existing `architecture`/`di`/`networking`/`crash_reporting` + ios-execute & ios-code-review profile-gated checks |
| hardcoded source + test directories | **`source_roots`** / **`test_roots`** |
| a feature-flag enum + defaults type | **`feature_flags`** |
| `.worktrees/`, `.claude/tasks/`, `_status.md`, `dev-N.md`, `dev-N` naming | **engine internals** — hardcode sane defaults, not project facts |
| reference check via `rg` + `--type xib/storyboard`; `@objc`/dynamic-dispatch blockers | **universal iOS** — bake into engine, not profile |
| project-specific commit scopes | delegate to `/commit` + note in `rules_file` |

**Genuinely new — both OPTIONAL, with safe defaults:**

```yaml
default_base_branch: main      # branch/PR target; overridable by --base. (default: main)
pr_tool: gh                    # gh | none. none → stop after push, print manual PR steps.
```

Task workspace reuses **`plans_dir/{slug}/`** (where `plan.md` already lands) for `scope.md`, `_status.md`, `dev-N.md`, `peer-review.md`. No new "tasks dir" concept.

### 3.4 Where testing lives (the user's explicit ask)

Testing is woven through, not bolted on:

1. **Plan** — `ios-plan` already emits a *Testing Strategy* (unit / snapshot / UI + accessibility ids).
2. **Implement** — team mode dev-1 writes tests **first** (TDD); solo mode writes tests alongside.
3. **Verify gate** — `verify_command` (build + test) must pass; output is read, not assumed. Unset → build-only, stated explicitly.
4. **Review gate** — `ios-code-review` checks test coverage of changed logic + transition holes.

*Optional future ports (not v1):* `ios-add-test` (runs tests 5× to catch flakiness) and `ios-automation-test-identifier` (QA accessibility ids) — flag as add-ons if you want deeper test rigor.

---

## 4. End-to-end data flow

```
TICKET-ID / context
      │
      ▼
ios-resolve ──INIT──► branch + plans_dir/{slug}/_status.md
      │
      ├─► ios-scout ───────────► scope.md
      │
      ├─► ios-plan ────────────► plan.md ──[APPROVAL GATE]
      │
      ├─► ios-execute --team N ► .worktrees/{slug}/dev-1..N ──peer review──► integrate
      │        │                                                              │
      │        └─ VERIFY: verify_command ◄───────────────────────────────────┘
      │
      ├─► ios-code-review ─────► verdict (Critical must clear)
      │
      ├─► /commit ─────────────► granular commits
      │
      └─► gh pr create ────────► PR url ──► (best-effort ticket transition)
```

Artifacts under `plans_dir/{slug}/`: `_status.md`, `scope.md`, `plan.md`, `dev-1..N.md`, `peer-review.md`.

---

## 5. Implementation plan (phases)

| Phase | Deliverable | Verify |
|---|---|---|
| **1. Profile + scaffold** | Add `default_base_branch`, `pr_tool` to `ios-profile.template.md`; create `ios-resolve/` folder; add `ios-execute/references/` | template parses; folders exist |
| **2. Team engine** | `ios-execute/references/team-execution.md` (spawn→context→build→review→merge→validate, generalized prompts/roles/gates); wire `--team N` + `--solo` flags into `ios-execute` Phase 3 | dry-read: every source-project hardcode replaced by a profile ref or documented default |
| **3. Orchestrator** | `ios-resolve/SKILL.md` (full pipeline §3.1, gates, ticket fetch, branch logic, PR, report) | walk-through on a sample ticket id + a free-form string (no code run) |
| **4. Fallbacks** | Handle: no ticket system, `ticket_fetch: none`, `pr_tool: none`, `verify_command` unset, `gh` not authed, merge conflict → BLOCKED | each path prints actionable guidance |
| **5. Docs** | Update root `README.md` pipeline diagram to include `ios-resolve`; link this design | README shows new entry point |
| **6. Dry-run** | Exercise the chain end-to-end on one real iOS app with a filled profile (small ticket) | branch + scope.md + plan.md + PR produced; gates pass |

Phases 2 and 3 are the bulk. 1 is quick. 4–6 are hardening.

---

## 6. Decisions (locked 2026-06-15)

1. **Orchestrator name** — **`ios-resolve`**.
2. **Engine placement** — **folded into the implementer**, which is **renamed `ios-execute` → `ios-execute`**; the `--team N` protocol lives in `ios-execute/references/team-execution.md`.
3. **Dedicated test skills** — **deferred**. v1 testing = TDD in team mode + `verify_command` gate + review coverage check. (`ios-add-test` 5× flake runs, `ios-automation-test-identifier` QA ids are later add-ons.)
4. **Default dev count** — **auto from plan complexity**: LOW→solo, MEDIUM→2, HIGH→3; `--devs N` / `--solo` always override.

---

## 7. Risks

- **Worktree overhead** on consuming repos (disk + setup); mitigate by defaulting to solo for small plans, team only when warranted.
- **Simulator/test contention** when several dev agents run `verify_command` concurrently — keep verification at the *integrate* step (post-merge), not per-dev, to serialize it.
- **Profile drift** — engine must fail loud if `verify_command` is empty (build-only) rather than silently "passing".
- **Over-generalization** — resist re-adding the 20 fields; `verify_command` + `rules_file` are the pressure-relief valves.

---

## 8. Hardening decisions (review pass, 2026-06-15)

After a self-review of `ios-resolve` + `team-execution.md`:

- **Dev worktree model** — native harness `isolation:"worktree"`; each dev commits to branch `{slug}/dev-N`; merge + peer-review happen by **branch name** (`git diff {base}...{slug}/dev-N`), so paths don't matter; the harness auto-cleans dev worktrees; only the integrate worktree is skill-managed.
- **Plan inputs** — `ios-plan` now emits a `complexity` frontmatter field (drives the auto dev-count) and an **Owner** column on the File Changes table (one owner per file → clean parallel merges).
- **Test split** — *revised to **vertical slices***: each dev owns code **and** tests for their slice (real local TDD, clean per-file ownership), no dedicated test dev. Independent edge-case coverage moved into peer review as a dedicated adversarial **edge-case / logic-gap reviewer agent** (Phase D) that drives missing tests *pre-merge*; the post-merge `ios-code-review` adversarial pass stays the final gate. (Supersedes the earlier dedicated-test-dev choice and removes the cross-worktree TDD-compile friction.)
- **Bug fixes** — (1) the team engine builds on the **task branch** (`BASE`), ff-merged back into it; (2) team-mode review uses `git diff {base}...{slug}/integrate`, not `--pending`; (3) devs must compile their worktree before reporting done (full test gate stays serialized at integrate); (4) TDD-compile-post-merge documented.

**Known soft spot (for the dry-run):** the whole team engine is unexercised, and `ios-plan` reliably populating `complexity` / `Owner` is LLM-dependent. The dry-run (§5 phase 6) is where these get validated.
