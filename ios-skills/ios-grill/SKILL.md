---
name: ios-grill
description: "Stress-test an iOS implementation plan by interrogating you one decision at a time — each question carries a recommended answer, and the codebase is checked before anything is asked. Surfaces and resolves hidden assumptions, open questions, and unstated decisions, then hardens plan.md + writes a decisions log. Auto-run by ios-resolve between plan and approval (skippable when the plan is already unambiguous). Does NOT implement. Reads .claude/ios-profile.md. Subcommand: standalone on a plan dir."
argument-hint: "[plan-dir | TICKET-ID]"
model: best
effort: high
---

# Grill — iOS plan stress-test

Walk an implementation plan one decision at a time and **interrogate the human** until every
hidden assumption is either answered by the code or decided on the record. Each question comes
with a **recommended answer**; the codebase is consulted **before** you ask. Output is a
**hardened plan** + a decisions log — not new code, not a new plan.

**YAGNI + KISS + DRY.** Resolve the few decisions that actually change the implementation; don't
manufacture questions to look thorough.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `architecture`, `state_type`, `di`, `navigation`, `networking`,
`localization`, `accessibility_ids`, `feature_flags`, `verify_command`, `high_rigor_domains`,
`plans_dir`, `rules_file`, ticket fields. If missing → run `ios-project-init`.

## What this is (and is NOT)

| Is | Is NOT |
|---|---|
| Interactive — **you** decide; the skill proposes a recommended answer per question | An autonomous reviewer that decides for you → that's `ios-plan red-team` |
| Excavates assumptions a plan left implicit, resolves them on the record | Trade-off ideation from scratch → `ios-brainstorm` |
| Edits an **existing** plan to remove ambiguity | Writing the plan → `ios-plan` · finding code → `ios-scout` |

Pairs with `ios-plan red-team`: red-team **finds** holes hostilely (no user); grill **closes** them
with you. Run red-team first on a hard plan, then grill to resolve what it surfaced.

## When to Use / NOT

| Use | Skip |
|---|---|
| Plan has Open Questions, likely-unmitigated Risks, or fuzzy ACs | Trivial single-file plan, ACs concrete, no open questions |
| `high_rigor_domains` touched (security/rollback/flag decisions) | Every decision is already answered by code / ticket / profile |
| State shape (success/error/empty/loading) or flag rollout undecided | You explicitly passed `--no-grill` / `--skip-approval` upstream |

**Auto-skip is a feature.** If, after Step 2, every open decision resolves to ≥ ~95% confidence
from the code/ticket/profile, do **not** ask anything — print one line ("Plan is unambiguous —
nothing to grill") and hand straight back. Never invent questions to justify the step.

---

## Argument parsing
- **plan-dir** — path to a plan directory under `{plans_dir}` (contains `plan.md`). Default: the
  plan `ios-resolve` just produced, else the most recent dir in `{plans_dir}`.
- **TICKET-ID** (matches `ticket_pattern`) → find the matching plan dir under `{plans_dir}`.
- If no plan exists → say so; point at `ios-plan`. Grill never plans from nothing.

---

## Flow
```
0. Load profile
1. LOAD        plan.md + per-phase files + scope.md + rules_file + ticket
2. BUILD       extract candidate decisions → answer from code FIRST → confidence-gate (auto-skip if all ≥95%)
3. ORDER       dependency order (blockers first) + impact (high-impact first)
4. GRILL       one question at a time, each with a recommended answer; update state after each
5. WRITE       fold decisions back into plan.md + per-phase files; write grill.md log
6. HANDOFF     back to the approval gate (or standalone menu)
```

### Step 1 — Load
Read the plan dir: `plan.md` (Overview, ACs, Approach, Phases, **Open Questions**,
**Risks & Mitigations**, **Out of Scope**, Feature Flag, Testing Strategy) + per-phase files.
Reuse upstream context (DRY): `scope.md` (the scout map) if present, `rules_file`, and the
fetched ticket. Don't re-scout — read what `ios-resolve`/`ios-plan` already gathered.

### Step 2 — Build the decision list (check the code FIRST)
Harvest candidate decisions from the plan, each a node in a small decision tree:
- **Open Questions** verbatim.
- **Risks** whose mitigation is "TBD"/likely/unowned.
- **Out of Scope** — confirm each boundary is intended, not an oversight.
- **State shape** (`state_type`): are success / error / **empty** / **loading** all specified?
- **Feature flag** (`feature_flags` != none): name, default-off, rollout/kill-switch decided?
- **`high_rigor_domains` touch** → security surface, rollback safety, analytics, migration.
- **Ambiguous ACs / phases** — any phase whose "done" you can't test objectively.
- **Cross-module ripple** — DI lifecycle, navigation ownership, cache invalidation left implicit.

> **For every candidate, try to answer it from code / `scope.md` / ticket / profile / `rules_file`
> before adding it to the ask-list.** Anything the repo already answers is dropped (cite where).
> This is the grill-me "delegate to the codebase first" rule — and the engine of the auto-skip.

Assign each surviving decision a **0–100% confidence** (your best read). All ≥ ~95% → auto-skip
(see above). Otherwise keep the < ~95% set — that's the grill list.

### Step 3 — Order
Sort the grill list by **dependency** (a decision that constrains others is asked first — e.g.
"is this behind a flag?" before "what's the flag's default rollout?") then by **impact**
(what most changes the implementation). Re-evaluate after each answer: a later question may be
fully resolved by an earlier one — drop it rather than ask.

### Step 4 — Grill loop (one at a time)
Use `AskUserQuestion`, **one focused decision per turn** (bundle only tightly-coupled
sub-parts). Every question MUST carry a **recommended answer as the first option, labeled
`(Recommended)`**, with 2–4 real alternatives. State the recommendation's *why* in one line.

```
Decision N/<total>: <the decision, concretely>
Code says: <what scope.md/the code already constrains, or "nothing — undecided">
Recommend: <your pick> — <1-line rationale>
[ options via AskUserQuestion: (Recommended) … · alt · alt ]
```

After each answer:
- **Listen for "should-want" signals** — buzzwords ("scalable", "robust") or "I should probably…"
  offered without a concrete need. Push back once: "what breaks if we *don't*?" Decide on the
  real need, not the best-practice reflex.
- Update working state; re-check the remaining list (Step 3) and drop now-answered decisions.

**Stop condition:** when you can confidently predict the user's answer to the next three
questions you'd ask, you're done — don't drain the list for completeness.

### Step 5 — Write back (only what changed)
Fold each decision into the plan **surgically** — touch only what the decisions changed:
- Resolve **Open Questions** (move to a decided line / delete) and sharpen affected **ACs**.
- Update **Risks**, **Out of Scope**, **Feature Flag**, **Testing Strategy**, and any phase
  whose steps/checkpoint the decision altered.
- Don't rewrite the plan or re-slice phases — that's `ios-plan`'s job. If grilling reveals the
  plan is structurally wrong, **stop** and recommend re-running `ios-plan` with the findings.

Write the decisions log: `{plan-dir}/grill.md`
```markdown
# Grill: <plan title>
**Plan**: {plan-dir}/plan.md   **Ticket**: <id|n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
## Decisions
| # | Decision | Recommended | Chosen | Plan change |
|---|---|---|---|---|
## Answered by the codebase (not asked)
- <decision> — <file:line / scope.md / ticket that settled it>
## Assumptions surfaced
- <implicit assumption the plan was making, now explicit>
## Still open (owner)
- <decision deferred to PM / BE / Design — who, and why it can wait>
```
(`<YYYY-MM-DD>` / any timestamp from `bash -c 'date +%F'`.)

### Step 6 — Handoff
- **Driven by `ios-resolve`:** return control; resolve runs its approval gate on the hardened plan.
- **Standalone:** print the menu (do NOT auto-invoke):
```
Plan hardened → {plan-dir}/plan.md   (decisions: {plan-dir}/grill.md)
- Implement now            → ios-execute {plan-dir}/plan.md
- Hostile review first     → ios-plan red-team {plan-dir}
- Re-plan (structural gaps) → ios-plan <ticket-or-topic>
```

---

## Anti-rationalisation
| Tempting | Reality |
|---|---|
| "The plan looks complete" | Complete-looking plans hide the costliest unstated assumptions. |
| "I'll just pick the obvious answer myself" | Recommend it — but it's the user's call to confirm; that's the point of grilling. |
| "Ask everything to be safe" | Asking what the code already answers burns trust. Check first, drop the answered. |
| "They said 'make it scalable'" | Buzzword, not a decision. Get the concrete need or cut it. |

## Sources
Methodology adapted (by hand, selectively) from:
- [mattpocock/skills — grill-me](https://github.com/mattpocock/skills/tree/733d312884b3878a9a9cff693c5886943753a741/skills/productivity/grill-me)
  — decision-tree walkthrough, one question at a time with a recommended answer, codebase-first delegation.
- [addyosmani/agent-skills — interview-me](https://github.com/addyosmani/agent-skills/tree/main/skills/interview-me)
  — confidence-number gating, hypothesis-per-question, "should-want" signal detection, predict-the-next-3 stop condition.

Tracked in `SOURCES.yaml` (`mode: audit-only`): `ios-skill-consolidate` flags staleness / surfaces
newer sources, but this skill is hand-curated — adaptation is never auto-merged.

## Constraints
- **DO NOT** implement, write a new plan, or scout from scratch. Edit an existing plan only.
- **MUST** check code / `scope.md` / ticket / profile **before** asking — drop anything already answered.
- **MUST** ask one decision at a time, each with a `(Recommended)` answer and a 1-line rationale.
- **MUST** auto-skip (one line, no questions) when every open decision is ≥ ~95% confident.
- **MUST** honor `rules_file`; resolve `high_rigor_domains` decisions explicitly (security/rollback/flag).
- **Sacrifice grammar for concision.**
