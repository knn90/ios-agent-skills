---
name: ios-brainstorm
description: "Brainstorm solutions for an iOS app. Use for ideation, architecture decisions, feasibility checks, feature exploration. Brutally honest about trade-offs and over-engineering. Outputs a decision report. Does NOT implement — hands off to ios-plan. Reads .claude/ios-profile.md."
argument-hint: "[topic | TICKET-ID | description]"
model: best
effort: xhigh
---

# Brainstorm — iOS

Collaborate to find the best solution while being **brutally honest** about feasibility,
trade-offs, and over-engineering.

**YAGNI + KISS + DRY.** Prefer the boring, proven approach already used in the codebase.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility_ids`, `feature_flags`, `high_rigor_domains`,
`reports_dir`, `rules_file`, ticket fields. If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing →
run `ios-project-init`.

## Critical Constraints

- You **DO NOT** implement. Brainstorm and advise only.
- You **DO NOT** write plans — hand off to `ios-plan` once the user approves a direction.
- Validate feasibility **inside the actual codebase** before endorsing an approach.
- Honor the rules in `rules_file`.

<HARD-GATE>
Do NOT invoke any implementation skill, write code, or scaffold anything until you have
presented a design AND the user has approved it — regardless of perceived simplicity.
</HARD-GATE>

---

## Argument Parsing
- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch`.
- **Free-form topic** → use as-is.

## Process Flow
```
1. Parse args + load context (profile, rules_file, git log -5)
2. Scout relevant code (ios-scout or direct Glob/Grep) — verify claims
3. Clarifying questions (AskUserQuestion)
4. Assess scope — decompose if too large
5. Propose 2-3 approaches with trade-offs
6. Iterate → user approves one
7. Write report to {reports_dir}
8. Suggest next skill (do NOT auto-invoke)
```

## Phase 1 — Context
- Load `rules_file`, relevant `docs_root` files, branch name + `git log -5 --oneline`.
- If `TICKET-ID`: fetch via `ticket_fetch`; extract title, description, ACs, links.
- Scout for non-trivial topics (`ios-scout` if scope spans 3+ modules). **Never recommend
  something whose existence you haven't verified.**

## Phase 2 — Clarifying questions
`AskUserQuestion`, bundle 2-4. Typical: feature flag needed (`feature_flags`)? deadline?
`min_ios` floor? backend/schema change? design sign-off? localization keys? success/error/
empty/loading states (`state_type` shape)? accessibility ids needed for UI tests?
Skip anything already answered by the ticket or scout.

## Phase 3 — Scope assessment
| Signal | Action |
|---|---|
| Spans 3+ independent subsystems | **Decompose** — split into sub-topics |
| 1-line copy/colour change | **Skip brainstorm** — point at implementation |
| Touches `high_rigor_domains` | **HIGH-RIGOR** — extra scrutiny, security Qs, rollback discussion |
| Greenfield, no prior art | First feature **establishes** the pattern — relax DRY, focus on getting the foundation right |

## Phase 4 — Propose approaches (2-3)

```markdown
### Approach N: <name>
**Summary**: <1-2 sentences>
**How it works here**:
- View / UI ({ui_framework}):
- State ({state_type}):
- Navigation ({navigation}):
- Dependencies / DI ({di}):
- Networking ({networking}, if any):
- Localization + accessibility ids touched:
- Tests (unit / snapshot / UI):
**Pros / Cons** · **Effort** S/M/L · **Risk** LOW/MED/HIGH · **Maintainability**
**Recommended? Yes/No — because <reason>**
```

### Brutal-honesty checklist
- [ ] Matches existing patterns, or introduces a new abstraction?
- [ ] Hidden costs (build time, app size, runtime, CI)?
- [ ] Migration burden on unrelated screens?
- [ ] Is the user solving the right problem?
- [ ] Violates any `rules_file` rule?

If the idea is wrong / over-engineered / regression-prone — **say so directly.**

## Phase 5 — Decision & report
Path: `{reports_dir}/brainstorm-{YYMMDD-HHMM}-{TICKET|slug}.md`
(`{YYMMDD-HHMM}` from `bash -c 'date +%y%m%d-%H%M'`).

```markdown
# Brainstorm: <topic>
**Ticket**: <id|n/a>  **Date**: <YYYY-MM-DD>  **Branch**: <current>
## Problem Statement
## Constraints & Requirements
## Approaches Evaluated  (1 / 2 / 3 — summary, pros, cons, effort/risk)
## Recommendation  (chosen + rationale)
## Implementation Considerations
- state shape / DI wiring / navigation / networking / localization + a11y
- feature flag (if feature_flags): <yes/no, name, default-off?>
- migration / rollback / testing strategy
## Risks & Mitigations  | Risk | Likelihood | Mitigation |
## Success Metrics
## Out of Scope
## Open Questions  (owner: PM / BE / Design)
```

## Phase 6 — Hand-off (menu, do NOT auto-invoke)
```
Brainstorm complete → {reports_dir}/<file>.md
- Create plan      → ios-plan <ticket-or-topic>
- More research    → ios-research <sub-topic>
- Deep reasoning   → ios-sequential-thinking <problem>
- Stop here        → keep report as decision record
```

## Anti-Rationalisation
| Tempting | Reality |
|---|---|
| "Too simple to design" | Simple tasks hide the most unexamined assumptions. |
| "I already know the answer" | Then writing it down takes 30 seconds. |
| "Let me just prototype" | Prototypes ship. Design first. |

## Constraints
- **DO NOT** implement, plan, or scaffold. **DO NOT** auto-invoke any skill.
- **MUST** verify code claims before endorsing. **List open questions** at the end.
- **Sacrifice grammar for concision.**
