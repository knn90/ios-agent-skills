---
name: ios-resolve
description: "Resolve a ticket or a free-form request end-to-end for an iOS project: branch → scout → plan → grill (harden) → approval → implement (solo or parallel team) → verify → review → commit → PR. The single front door that chains ios-scout, ios-plan, ios-grill, ios-execute, and ios-code-review. Reads .claude/ios-profile.md. Use when given a ticket id or told to 'resolve/ship/work on' something end-to-end."
argument-hint: "[TICKET-ID | \"description\"] [--devs N | --solo] [--base BRANCH] [--skip-approval] [--no-grill] [--no-pr]"
---

# Resolve — iOS end-to-end workflow

One command takes a **ticket or free-form context** all the way to an open PR. This skill only
**sequences and gates** — the real work is done by the other `ios-*` skills. Project-agnostic:
every fact comes from `.claude/ios-profile.md`.

**YAGNI + KISS + DRY.** Don't re-do what a sub-skill already does; don't skip a gate.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `source_roots`, `test_roots`, `plans_dir`, `rules_file`,
`verify_command`, `high_rigor_domains`, `architecture`, `feature_flags`,
`ticket_system`, `ticket_pattern`, `ticket_fetch`, `default_base_branch`, `pr_tool`.
If missing → run `ios-project-init` first. If `source_roots` is empty (greenfield, nothing to
scout/implement against) → tell the user to bootstrap the app first.

## When to Use / NOT

| Use | Don't — use instead |
|---|---|
| A ticket id to take to PR | Just find files → `ios-scout` |
| "Resolve / ship / work on X" end-to-end | Just a plan, no code → `ios-plan` |
| A described change spanning scope→plan→code→PR | Implement an existing plan → `ios-execute <plan-path>` |
| | Trade-off discussion → `ios-brainstorm` · review only → `ios-code-review` |

For a trivial one-file fix, skip this — edit directly (or `ios-execute --fast`).

---

## Syntax
```
ios-resolve ABC-123
ios-resolve "add pull-to-refresh on the orders list"
ios-resolve ABC-123 --devs 3 --base release/1.2
ios-resolve ABC-123 --solo --skip-approval
ios-resolve "fix crash on empty cart" --no-pr
```

## Argument parsing
- **TICKET-ID** — matches `ticket_pattern` → fetch via `ticket_fetch` (Step 1).
- **"free-form"** — used as the task description (no ticket fetch).
- `--devs N` / `--solo` — passed to `ios-execute` (else auto: LOW=solo·MEDIUM=2·HIGH=3).
- `--base BRANCH` — PR target + branch-from point (default `default_base_branch`, else `main`).
- `--skip-approval` — skip the plan approval gate (trivial/well-understood only). Also skips grill.
- `--no-grill` — skip the plan stress-test (Step 4.5) but keep the approval gate.
- `--no-pr` — stop after commit; print push/PR steps for the user.

---

## Pipeline

```
0. Load profile
1. RESOLVE INPUT   ticket → fetch context · free-form → use as description
2. INIT            branch {type}/{slug} off {PR_BASE}; create {plans_dir}/{slug}/_status.md
3. SCOUT           ios-scout → scope.md
4. PLAN            ios-plan  → plan.md
4.5 GRILL          ios-grill <plan-dir> → harden plan + grill.md   (skip if --no-grill/--skip-approval)
   PLAN GATE       ──► APPROVAL GATE on the hardened plan (unless --skip-approval)
5. IMPLEMENT       ios-execute <plan> [--team N|--solo]  (does verify + review gates itself)
6. COMMIT          ensure work is committed via /commit (granular)
7. PR              push + open PR → {PR_BASE}     (skip if --no-pr or pr_tool: none)
8. TICKET          best-effort transition (e.g. → In Review) if the system supports it
9. REPORT          branch · PR url · files · tests · review verdict
```

### Step 1 — Resolve input
- **TICKET-ID:** fetch via `ticket_fetch` (an MCP tool or CLI named in the profile — discover
  with `ToolSearch` if it's an MCP server). Extract: summary, description, acceptance criteria,
  issue type, labels. If `ticket_fetch: none` → ask the user to paste the ticket text.
- **Free-form:** the argument *is* the task; no fetch.
- Pick **branch type** from issue type (`Bug`→`fix` · `Story`/feature→`feat` · `Task`/chore→`chore`),
  else infer from the description; default `feat`.

### Step 2 — Init
- `PR_BASE` = `--base` / `default_base_branch` / `main`.
- `slug` = ticket id (if any) + short kebab summary (≤ 50 chars).
- `git fetch && git checkout -b {type}/{slug} origin/{PR_BASE}` (resume if it already exists).
- Create `{plans_dir}/{slug}/_status.md` (task title, ticket, base, branch, phase checklist).

### Step 3 — Scout
Invoke `ios-scout {target}` (target = main component/feature from the ticket, or the description).
Write its map to `{plans_dir}/{slug}/scope.md`. Skip only for a truly trivial single-file change.

### Step 4 — Plan
Invoke `ios-plan` for this task (point it at the Step 3 `scope.md` so it reuses that map instead
of re-scouting) → writes `{plans_dir}/{slug}/plan.md` (phased, with a Testing
Strategy, a `complexity` field, and an Owner column on the File Changes table for team runs).
**Read the plan's `complexity`** — it sets the auto dev count for Step 5.

### Step 4.5 — Grill (harden the plan)
Unless `--no-grill` or `--skip-approval`, invoke `ios-grill {plans_dir}/{slug}` to interrogate
the open decisions in `plan.md` one at a time (each with a recommended answer, codebase checked
first) → writes the resolved decisions back into `plan.md` + a `grill.md` log. `ios-grill`
auto-skips (no questions) when the plan is already unambiguous. Do **not** duplicate its
questioning here. The approval gate below then runs on the **hardened** plan.

### Approval gate
Unless `--skip-approval`, present a summary of the hardened plan and **wait**:
```
Plan ready: {plans_dir}/{slug}/plan.md
- Complexity: {…}   Suggested devs: {auto N}   Files: {n}   Feature flag: {name|n/a}   HIGH-RIGOR: {yes/no}
- Key changes: {3-5 bullets}
Reply: approved · revise: {feedback} · abort
```
`revise:` → re-run `ios-plan` with feedback. `abort` → set `_status.md` CANCELLED, stop.

### Step 5 — Implement
Hand the approved plan to the implementer **from the task branch** (so the team engine's
`BASE` = `{type}/{slug}` and the merge stays in-lineage):
```
git checkout {type}/{slug}
ios-execute {plans_dir}/{slug}/plan.md [--team N | --solo]
```
- Dev count: `--devs`/`--solo` if given, else the plan's `complexity` (LOW=solo·MEDIUM=2·HIGH=3).
- **Team mode:** the engine (`team-execution.md`) cuts the dev + integrate branches off
  `{type}/{slug}` and returns `{slug}/integrate` (already verified + reviewed inside `ios-execute`).
  Fast-forward it onto the task branch: `git checkout {type}/{slug} && git merge --ff-only {slug}/integrate`.
  Solo mode commits onto `{type}/{slug}` directly — nothing to merge.
- `ios-execute` owns the **verify** gate (`verify_command`) and the **review** gate
  (`ios-code-review`; team mode reviews `git diff {type}/{slug}...{slug}/integrate`, not `--pending`).
  Do **not** duplicate them here. If it returns BLOCKED → surface and stop.

### Step 6 — Commit
`ios-execute` commits granularly via `/commit`. If anything is still uncommitted (status/doc
tweaks, the integrate merge), run `/commit` once more. Never `git commit --no-verify`.

### Step 7 — PR
If `--no-pr` or `pr_tool: none` → push nothing; print the manual `git push` + PR command and stop at Step 9.
Otherwise (`pr_tool: gh`):
```bash
git push -u origin {type}/{slug}
gh pr create --base {PR_BASE} --fill   # or build a body from plan.md + ticket link
```
If a `create-pr-description` / PR-template skill is available, use it for the body. If `gh` is
absent/unauthed → print the manual steps (don't fail the whole run).

### Step 8 — Ticket transition (best-effort)
If a ticket was fetched and `ticket_system` supports transitions, move it to the review state
(e.g. "In Review" / "Code Review"). On failure → log a warning, continue. Skip for free-form runs.

### Step 9 — Report
```
## Resolved: {ticket id | description}
Branch: {type}/{slug} → {PR_BASE}     PR: {url | "not created (--no-pr)"}
Mode: {solo | team N}                 Ticket: {new state | n/a}
Files: {n}   Tests: {n} ({passing | build-only})   Review: {APPROVED | issues}
Artifacts: {plans_dir}/{slug}/ (scope.md, plan.md, _status.md[, dev-*.md, peer-review.md])
Next: review the PR · add reviewers · merge when green
```

---

## Gates (never skip)
1. **Approval** after planning (unless `--skip-approval`).
2. **Verify** — `verify_command` must pass (or build-only, stated). Owned by `ios-execute`.
3. **Review** — `ios-code-review`; Critical findings must clear. Owned by `ios-execute`.

## Error handling
| Situation | Action |
|---|---|
| Profile missing | run `ios-project-init`, then retry |
| `ticket_fetch: none` / fetch fails | ask the user to paste the ticket text; continue |
| `verify_command` unset | `ios-execute` does build-only and says so — do not claim tests passed |
| Team merge conflict | `ios-execute` reports BLOCKED with files/owners — surface, stop |
| `gh` absent/unauthed or `pr_tool: none` | finish through commit; print manual push + PR steps |
| Greenfield (no `source_roots`) | stop; tell the user to scaffold the app first |

## Related skills
`ios-scout` (Step 3) · `ios-plan` (Step 4) · `ios-grill` (Step 4.5, plan stress-test) ·
`ios-execute` (Step 5; `--team` → its
`references/team-execution.md`) · `ios-code-review` (review gate, inside `ios-execute`) ·
`/commit` (Step 6) · `ios-project-init` (profile bootstrap).

## Constraints
- **DO NOT** implement code yourself — delegate to `ios-execute`.
- **DO NOT** skip the approval/verify/review gates. **DO NOT** auto-merge to `{PR_BASE}`.
- **DO NOT** push without reaching Step 7 (and not at all under `--no-pr`).
- **MUST** read every project fact from `.claude/ios-profile.md`; never hardcode app specifics.
- **Sacrifice grammar for concision** in reports.
