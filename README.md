# ios-* skill suite

A project-agnostic iOS engineering workflow, generalized from the Alfie-iOS skills.
All project-specific facts live in **one** file — `.claude/ios-profile.md` — and every
skill reads it at runtime. The skill bodies themselves contain no hardcoded app names,
paths, architectures, or commands.

```
ios-project-init ──→ ios-profile.md (+ Docs/ tree)        # run ONCE per project
        │
        ├─ ios-scout ────────┐
        ├─ ios-research ──────┤
        ├─ ios-brainstorm ────┼─→ ios-plan ─→ ios-cook ─→ ios-code-review
        └─ ios-sequential-thinking  (reasoning aid; plugs in anywhere)
```

## Skills

| Skill | Implements code? | Purpose |
|---|---|---|
| `ios-project-init` | sets up | One-time bootstrap. Greenfield → prescriptive profile; existing app → detect & record. |
| `ios-scout` | no | Fast parallel code discovery. |
| `ios-research` | no | Sourced technical research grounded in the codebase. |
| `ios-brainstorm` | no | Brutally honest trade-off analysis. |
| `ios-sequential-thinking` | no | Step-by-step reasoning with revision/branching. |
| `ios-plan` | no | Phased implementation plan. |
| `ios-cook` | **yes** | The only implementer. plan → code → verify → review gates. |
| `ios-code-review` | no | 3-stage adversarial review; delegates to specialist skills if present. |

## Layout

```
ios-agent-skills/
├── README.md
├── ios-profile.template.md      # copy into each consuming project's .claude/ as ios-profile.md
└── ios-skills/                  # the skills — every child folder is one skill
    ├── ios-project-init/SKILL.md
    └── ios-{scout,research,brainstorm,sequential-thinking,plan,cook,code-review}/SKILL.md
```

## Setup

1. Copy `ios-profile.template.md` → `<project>/.claude/ios-profile.md` and fill it in,
   **or** run `ios-project-init` to generate it interactively.
2. Make the skills discoverable to Claude Code — copy or symlink the skill folders from
   `ios-skills/` into a project's `.claude/skills/`, into `~/.claude/skills/`, or package
   them as a plugin.
3. (Optional) Install specialist plugins for deeper review and list them under
   `specialists:` in the profile:
   - `swiftui-expert` — SwiftUI view/state/perf review
   - `swift-concurrency` — actor isolation, Sendable, Swift 6 readiness
   - `swift-testing-expert` — swift-testing macros/traits
   - `apollo-ios` — only if `networking: Apollo-GraphQL`
   ```
   /plugin marketplace add <owner/repo>
   /plugin install <skill>@<marketplace>
   ```
   code-review degrades gracefully if a specialist is absent — it just skips that
   sub-section.

## Core principles (shared by every skill)

- **YAGNI + KISS + DRY.** Minimum code that solves the problem.
- **Plan-first.** No implementation code before an approved plan (hard gate in `ios-cook`).
- **Verify before claiming.** Run `verify_command`; read the output; *then* claim done.
  Empty `verify_command` → build-only, stated explicitly.
- **HIGH-RIGOR escalation.** Diffs touching `high_rigor_domains` get mandatory
  adversarial review + correctness audit.
- **Profile-driven.** Read `.claude/ios-profile.md` first; never hardcode project facts.
