# ios-* skill suite

A **project-agnostic iOS engineering workflow** for Claude Code. Point it at any iOS app, fill in
**one** file (`.claude/ios-profile.md`), and you get a full **ticket → PR** pipeline plus
specialist reviewers (used automatically once installed). The skill bodies hardcode **no** app names, paths, architectures, or commands —
every project-specific fact lives in the profile and is read at runtime.

```
ios-project-init ──→ .claude/ios-profile.md            # run ONCE per project

ios-resolve  ── one command: ticket/context → scout → plan → execute → review → PR
      │
      ├─ ios-scout ────────┐
      ├─ ios-research ──────┤
      ├─ ios-brainstorm ────┼─→ ios-plan ─→ ios-execute ─→ ios-code-review
      └─ ios-sequential-thinking  (reasoning aid; plugs in anywhere)
                                          │ apply           │ route
                                          └─── ios-specialists ───┘
                                  (SwiftUI · Concurrency · Testing — Apple-primary)
```

## Setup (once per project)

1. **Install the skills** — copy or symlink the skill folders into your app (or globally):
   ```bash
   cp -R ios-skills/*       <project>/.claude/skills/    # the core workflow
   cp -R ios-specialists/*  <project>/.claude/skills/    # optional specialists you want
   ```
   (Or into `~/.claude/skills/`, or package as a plugin.) **Don't** ship `ios-skill-consolidate/` — it's repo-maintenance.
2. **Create the profile** — run **`ios-project-init`**. It *detects* an existing app's architecture/paths, or sets conventions for a greenfield app, and writes `.claude/ios-profile.md`. (Or copy `ios-profile.template.md` and fill it in by hand.)
3. **Specialists work automatically** once installed (step 1) — `ios-code-review` routes change-typed
   slices to them and `ios-execute` applies them while writing. No extra opt-in. The profile's
   `specialists:` field is an **optional override**:
   ```yaml
   # specialists: [ios-swiftui-expert, ios-concurrency-expert]   # restrict to just these
   # specialists: none                                            # turn specialist routing off
   ```
   Leave it unset to auto-use every `ios-*-expert` you installed.

## How to use it

### The whole loop, one command

```
ios-resolve ABC-123
ios-resolve "add pull-to-refresh to the orders list"
ios-resolve ABC-123 --team 2          # parallel dev team
ios-resolve ABC-123 --solo --no-pr    # single agent, stop before the PR
```

`ios-resolve` is the front door. It runs:

```
fetch ticket/context → create branch → ios-scout → ios-plan ──(you approve)──►
    ios-execute (TDD; solo or --team N) → ios-code-review → /commit → open PR
```

You approve the plan **before** any code is written, and nothing is pushed without you.

### Or à la carte

Every skill also stands alone:

```
ios-scout OrderListViewModel            # where does this live?
ios-research "offline sync options"     # sourced technical research
ios-brainstorm "offline support"        # brutal trade-off analysis → decision report
ios-plan ABC-123                        # phased plan (you approve)
ios-execute <plan-path> --team 2        # implement (TDD; parallel worktree team)
ios-code-review #42                     # adversarial, multi-lens review of a PR
```

## The skills

**Core workflow — `ios-skills/`**

| Skill | Role |
|---|---|
| `ios-project-init` | One-time: writes `.claude/ios-profile.md` (detect existing app / set greenfield conventions). |
| `ios-resolve` | **The front door.** ticket/context → scout → plan → execute → review → PR. Orchestrates; never implements directly. |
| `ios-scout` | Fast, token-efficient parallel code discovery — returns a *map*, not analysis. |
| `ios-research` | Sourced technical research grounded in the codebase. |
| `ios-brainstorm` | Brutally honest trade-off analysis → decision report. |
| `ios-sequential-thinking` | Step-by-step reasoning aid; plugs in anywhere. |
| `ios-plan` | Phased implementation plan, with an approval gate. |
| `ios-execute` | **The only implementer.** plan → code → verify → review. **TDD always**; solo, or `--team N` (parallel worktree devs + peer review + a dedicated edge-case reviewer + merge). |
| `ios-code-review` | Multi-lens adversarial review; **routes change-typed slices to the specialists**; precision-over-recall findings. |

**Specialists — `ios-specialists/` (optional add-ons; auto-used once installed)**

Deep domain reviewers, consolidated from curated community sources with **Apple / official docs as the source of truth**. `ios-code-review` triggers them per change type; `ios-execute` applies them while writing.

| Specialist | Domain |
|---|---|
| `ios-swiftui-expert` | SwiftUI — composition, state, performance, navigation, accessibility, Liquid Glass |
| `ios-concurrency-expert` | Swift Concurrency — async/await, actors, Sendable, Swift 6 isolation |
| `ios-testing-expert` | Testing — Swift Testing **and** XCTest, migration, flakiness |

**Maintenance — `ios-skill-consolidate/` (repo-only; not shipped)**

Rebuilds any specialist (or a core skill like `ios-code-review`) from its `SOURCES.yaml`: a staleness audit + **web discovery** for newer/better sources + fan-out extraction + synthesis. Run it with no argument to pick a skill from a menu (or `all`). Keeps skills from rotting; Apple/official sources always win on conflict.

## The profile — the one file that matters

`.claude/ios-profile.md` is the single source of every project-specific fact: `source_roots`,
`architecture`, `state_type`, `di`, `navigation`, `networking`, `verify_command`,
`high_rigor_domains`, `specialists`, `ticket_fetch`, `default_base_branch`, `pr_tool`, … Every skill
reads it first, so the skill bodies stay generic. Start from `ios-profile.template.md`.

## Layout

```
ios-agent-skills/
├── README.md
├── ios-profile.template.md      # copy to <project>/.claude/ios-profile.md and fill in
├── ios-skills/                  # core workflow (9 skills)        — ship these
├── ios-specialists/             # optional specialist reviewers   — ship the ones you want
├── ios-skill-consolidate/       # repo-maintenance (rebuilds skills from sources) — do NOT ship
└── docs/                        # design notes
```

## Core principles (every skill)

- **Profile-driven** — read `.claude/ios-profile.md` first; never hardcode project facts.
- **Plan-first** — no implementation before an approved plan (hard gate in `ios-execute`).
- **TDD always** — `ios-execute` writes a failing test before the code.
- **Verify before claiming** — run `verify_command`, read the output, *then* claim done (empty → build-only, stated).
- **Apple / official = source of truth** — specialists and review defer to Apple docs over community sources.
- **HIGH-RIGOR escalation** — diffs touching the profile's `high_rigor_domains` get a mandatory adversarial pass + correctness audit.
- **Precision over recall** — reviews surface a focused list of real findings, not a wall of nits.
