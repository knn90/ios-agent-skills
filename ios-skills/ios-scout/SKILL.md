---
name: ios-scout
description: "Fast, token-efficient codebase scouting for an iOS project. Spawns parallel Explore subagents scoped to feature/module folders. Use for file discovery, gathering task context, and locating where a feature lives before edits. Reads .claude/ios-profile.md for repo layout."
argument-hint: "[search-target] [quick|full]"
model: best
effort: xhigh
---

# Scout — iOS

Find things fast using parallel `Explore` subagents. Output is a **map**, not analysis.

## Step 0 — Load profile

Read `.claude/ios-profile.md` for `source_roots`, `test_roots`, `generated_paths`,
`ui_framework`, `navigation`, `networking`. If absent, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still absent → tell the
user to run `ios-project-init` first (greenfield repos have nothing to scout).

## When to Use

- A feature spans multiple modules / `source_roots`
- "Where is X?", "find Y", "locate the ViewModel/View/Reducer for Z"
- Before cross-module edits (shared state, DI registrations, network operations)
- DRY check — confirm a pattern exists before adding a new one

## When NOT to Use

| Case | Use instead |
|---|---|
| Single file, known path | `Read` |
| One symbol grep | `Grep` |
| Trade-off discussion | `ios-brainstorm` |
| Full implementation plan | `ios-plan` |
| Pattern outside the repo | `ios-research` |

---

## Argument Parsing

- **TARGET** (required) — type name, feature area, function, network operation, or NL description.
- **Depth** — `quick` (default, single agent) | `full` (2-3 parallel agents).

If TARGET missing, ask via `AskUserQuestion`.

---

## Workflow

### 1. Estimate scale (cheap probes)

Use profile paths:
```
Glob {source_roots}/**/<term>*.swift
Glob {test_roots}/**/<term>*.swift
Grep <term> across the narrowed dirs
```

Agent count:
- **1** — < 50 matched files, single module
- **2** — feature touches app target + a package module
- **3** — split: app target / package modules / tests

Don't spawn agents for trivial single-file lookups (overhead > benefit).

### 2. Spawn parallel `Explore` agents (single message → concurrent)

Each gets a tight scope. Prompt template:

```
Scope: {absolute path under a source_root}
Search target: {TARGET}

Find:
1. Primary implementation files (Views, ViewModels/Reducers, Coordinators/Routers, Services)
2. Tests covering them ({test_roots})
3. Direct usages: callers, navigation wiring, DI registrations
4. Related models / DTOs / network operations ({networking})
5. Accessibility identifier usages (if a UI surface and accessibility_ids != none)

Return brief markdown: `file_path:line — one-line description`.
Do NOT paste file contents. Cap at 25 most relevant matches.
```

**Never** scout into `generated_paths` from the profile.

### 3. Aggregate

Combine, dedupe, sort by relevance. Note any agent that timed out and continue.

---

## Report Format

```markdown
# Scout Report: <TARGET>

**Branch**: <current>   **Agents**: <N>

## Relevant Files
### App target
- `<path>:<line>` — <desc>
### Packages / modules
- `<path>:<line>` — <desc>
### Tests
- `<path>:<line>` — <desc>

## Patterns Observed
- <state pattern, DI mechanism, navigation wiring actually seen>

## Unresolved Questions
- <gap or ambiguity>
```

If invoked inline (no report needed), output to the user directly.

## Constraints

- **DO NOT** read full files into main context — that's the subagent's job.
- **DO NOT** scout into `generated_paths`.
- **DO NOT** spawn agents for trivial single-file lookups.
- **Verify, don't guess** — every returned path must exist. Cap at top 25.
