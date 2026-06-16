---
name: ios-plan
description: "Create detailed, phased implementation plans for an iOS project. Use for feature planning, refactors, ticket resolution. Outputs a phased plan to the project's plans dir. Does NOT implement — hands off to ios-execute. Reads .claude/ios-profile.md. Subcommands: archive, red-team, validate."
argument-hint: "[task | TICKET-ID | archive | red-team | validate]"
model: best
effort: max
---

# Plan — iOS

Create detailed technical implementation plans through codebase analysis, ticket context,
and phased documentation.

**YAGNI + KISS + DRY.** Honest, brutal, concise.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility_ids`, `feature_flags`, `verify_command`,
`high_rigor_domains`, `plans_dir`, `source_roots`, `generated_paths`, `rules_file`,
ticket fields. If missing → run `ios-project-init`.

Plans touching `high_rigor_domains` must address security, rollback safety, feature-flag
rollout (if `feature_flags` != none), and analytics impact.

## When to Use
New feature; resolving a ticket; refactor with cascade risk; removing a feature flag;
backend schema change needing coordinated client work.

## When NOT to Use
| Case | Use instead |
|---|---|
| Single-file fix, < 20 LoC | Implement directly |
| Need trade-off analysis | `ios-brainstorm` |
| Just find files | `ios-scout` |

---

## Argument Parsing
- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch`.
- **Free-form** → planning subject.
- **Subcommands:** `archive`, `red-team`, `validate`.
- **Mode flags:** `--fast`, `--hard`, `--two`, `--no-tasks`.

If no args, ask via `AskUserQuestion` (default = create plan; or archive/red-team/validate).

## Pre-Creation Scan
Before a new plan, scan `{plans_dir}` for unfinished plans (`status != completed/cancelled`):
read frontmatter, compare scope (overlapping files, shared DI keys, same feature), set
`blockedBy`/`blocks` on BOTH plans when a dependency is found; `AskUserQuestion` if ambiguous.

## Workflow Modes
| Flag | Research | Red Team | Validate |
|---|---|---|---|
| default (auto-detect) | per complexity | per complexity | per complexity |
| `--fast` | skip | skip | skip |
| `--hard` | yes | yes | optional |
| `--two` | 2 researchers | after selection | after selection |

Auto-detect: trivial(1 file,<30 LoC)→`--fast`; single clear feature→default;
cross-module / `high_rigor_domains`→`--hard`; real uncertainty between 2 approaches→`--two`.

---

## Process Flow
```
1. Parse args
2. Pre-creation scan ({plans_dir})
3. Fetch ticket (ticket_fetch) if TICKET-ID
4. Scope challenge (skip if --fast)
5. Codebase analysis (ios-scout or Glob/Grep)
6. (optional) Research (ios-research) for --hard / --two
7. Solution design
8. Write plan.md + per-phase files under {plans_dir}
9. (optional) Red team → red-team.md
10. (optional) Validate interview → validation.md
11. Output execute handoff (do NOT auto-invoke)
```

## Phase 1 — Context
Load `rules_file` + relevant `docs_root` files + existing brainstorm report for the same
ticket. Fetch ticket via `ticket_fetch`. Scope challenge (skip if `--fast`/trivial): solving
the problem or the symptom? sub-tasks deferrable? one plan or 2-3 sequential?

## Phase 2 — Research (skip if `--fast`)
`--hard`/`--two` → `ios-research <topic>`. Default → lightweight `WebSearch` only if truly
needed. **Don't research patterns already in the codebase.**

## Phase 3 — Codebase analysis
If scope spans 3+ modules → `ios-scout`. Capture per file to modify:
`path:line` · existing pattern to follow (DRY) · test file + pattern · DI keys touched ·
navigation closures touched · network operations touched (never edit `generated_paths`) ·
localization keys + accessibility ids needed.

## Phase 4 — Solution design
Per approach: View / state ({state_type}) / navigation split · DI wiring ({di}) · network
operation + cache strategy ({networking}) · feature flag (`feature_flags`) + default-off
rollout · backwards-compat/migration · testing strategy (unit/snapshot/UI with accessibility
ids) · rollback plan. `--two` → comparison table.

## Phase 5 — Plan documentation
Dir: `{plans_dir}/{YYMMDD-HHMM}-{TICKET|slug}/` — `{YYMMDD-HHMM}` from
`bash -c 'date +%y%m%d-%H%M'`. Create if missing.

```
{plans_dir}/{date-ticket-slug}/
├── plan.md                 # master plan + frontmatter
├── phase-1-foundation.md   # only if 3+ phases
├── phase-2-data.md
├── phase-3-ui.md
└── phase-4-tests.md
```

`plan.md` frontmatter:
```yaml
---
title: <short title>
ticket: <id | n/a>
status: pending            # pending | in-progress | completed | cancelled
complexity: MEDIUM         # LOW | MEDIUM | HIGH — drives team dev-count (LOW=solo·MEDIUM=2·HIGH=3)
mode: auto                 # auto | fast | hard | two
blockedBy: []
blocks: []
created: <YYYY-MM-DD>
---
```

`plan.md` body:
```markdown
## Overview            <2-4 sentences>
## Acceptance Criteria
## Approach            <chosen + rationale; link brainstorm report if any>
## Phases              1. Foundation 2. Data 3. UI 4. Tests
## File Changes (Summary Table)  | File | Module | Type | Change | Owner |
       # Owner = dev-1/dev-2/dev-3 for team runs (one owner per file → clean merges); "-" for solo
## Feature Flag        Name / Default off / Rollout  (or "n/a" if feature_flags==none)
## Testing Strategy    Unit / Snapshot / UI (accessibility ids needed)
## Risks & Mitigations | Risk | Likelihood | Mitigation |
## Out of Scope
## Open Questions
```

Per-phase file:
```markdown
## Phase N: <name>
### Goal           <1-2 sentences>
### Steps
1. **<Step>** (file: `<path>:<line>`) — what to change · why · test to add/update
### Verification
- Run `{verify_command}`  (if unset → build-only; state it)
- Manual: <if applicable>
### Estimated Effort   S / M / L
```

## Phase 6 — Red Team (`--hard`/`--two` or subcommand)
`ios-plan red-team <plan-dir>` — spawn an adversarial reviewer:
> Review this plan as a hostile reviewer. Find missing edge cases, unstated assumptions,
> security gaps, state-transition holes, DI lifecycle issues, networking/cache pitfalls,
> rollback risks, accessibility gaps. Be brutal.

Save to `{plans_dir}/{plan-dir}/red-team.md`; update the plan before proceeding.

## Phase 7 — Validation interview (optional)
`ios-plan validate <plan-dir>` — `AskUserQuestion`: ACs mapped to phases? feature flag
confirmed with PM? backend schema timing confirmed? design signed off? localization strings
provided? accessibility ids reviewed? QA bandwidth? Save to `validation.md`.

## Phase 8 — Execute handoff (do NOT auto-invoke)
```
Plan ready: {plans_dir}/{plan-dir}/plan.md
- Implement now            → ios-execute {plans_dir}/{plan-dir}/plan.md
- Adversarial review first → ios-plan red-team <plan-dir>
- Validate with team       → ios-plan validate <plan-dir>
```
(When `ios-resolve` drives the flow, it runs this handoff for you.)

## Subcommands
- **archive** — move `status: completed|cancelled` plans → `{plans_dir}/_archive/{YYMM}/`, append journal.
- **red-team / validate** — standalone Phase 6 / 7.

## Greenfield note
First 1-2 features have no prior art — the plan **establishes** the pattern. Relax the
"match existing pattern" DRY checks; be extra explicit about the conventions being set
(they become what later plans DRY against).

## Constraints
- **DO NOT** implement — plans only. **DO NOT** auto-invoke `ios-execute`.
- **DO NOT** create plans outside `{plans_dir}`. **MUST** follow `rules_file`.
- **MUST** include `{verify_command}` in each phase's Verification (or note build-only).
- **Sacrifice grammar for concision.**
