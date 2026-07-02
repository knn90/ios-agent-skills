---
name: ios-research
description: "Strategic technical research for an iOS project (Swift / SwiftUI / UIKit / networking / concurrency / security / accessibility). Use for library evaluation, framework pattern questions, migration questions, CVE checks. Grounded first in what already works in the codebase. Outputs a sourced report. Reads .claude/ios-profile.md."
argument-hint: "[topic | TICKET-ID]"
---

# Research — iOS

Strategic research producing an actionable, sourced report grounded in current Swift/iOS
best practices **and** what already works in this codebase.

**Principles:** YAGNI + KISS + DRY. Honest, brutal, concise.
**Bias toward what already works in the repo.** Don't recommend a library the project
doesn't use unless you can show the existing pattern is genuinely insufficient.

## Step 0 — Load profile

Read `.claude/ios-profile.md`: `min_ios`, `architecture`, `networking`, `ui_framework`,
`reports_dir`, `ticket_system`/`ticket_pattern`/`ticket_fetch`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/ios-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing → run
`ios-project-init`.

## When to Use

- Evaluating a library/SDK for adoption
- Framework patterns (state, navigation, performance) for an unfamiliar problem
- Migration questions (networking layer upgrade, concurrency adoption, UIKit→SwiftUI)
- Security best practices (Keychain, biometrics, ATS, certificate pinning)
- Accessibility patterns beyond what the repo already documents
- Recent CVEs / breaking changes for an existing dependency

## When NOT to Use

| Case | Use instead |
|---|---|
| Pattern already exists in repo | `ios-scout` |
| API doc for a known package | `WebFetch` against the official URL |
| Trade-off discussion of approaches | `ios-brainstorm` |
| Step-by-step debug reasoning | `ios-sequential-thinking` |

---

## Argument Parsing

- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch` for context.
- **Free-form topic** → research as-is. If missing, ask via `AskUserQuestion`.

---

## Process

### Phase 1 — Scope
- Key terms; recency window (< 12 months unless historical); in/out of scope.
- Source ranking: Apple docs > WWDC > official framework/community docs > major eng blogs > recent SO.

### Phase 2 — Codebase reality check (FIRST)
Before any web search:
- Is there already a working pattern? → `ios-scout` / `Grep`.
- What's pinned? → `Package.resolved`, `Package.swift`, `Podfile.lock`.
- What's documented? → `rules_file` + `docs_root`.
- **If the codebase already solves it → say so and stop.** That's a successful outcome.

### Phase 3 — External research (≤ 5 search calls)
Preferred order: Apple official docs → WWDC (current + last 2y) → framework official docs
(e.g. networking lib for `networking`) → Swift Forums → reputable iOS eng blogs
(Point-Free, objc.io, Swift by Sundell, Use Your Loaf) → recent GitHub Discussions/SO.

Query patterns: `<topic> swiftui ios <min_ios>`, `<topic> swift concurrency sendable`,
`<topic> <networking-lib> <version>`, `<topic> CVE <year>`.

### Phase 4 — Cross-reference & validate
- 2+ independent sources per claim; flag conflicts.
- Reject APIs above `min_ios` unless the user agrees to bump the deployment target.
- Discard anything older than the framework major version in use.

### Phase 5 — Report
Save to `{reports_dir}/research-{YYMMDD-HHMM}-{TICKET|slug}.md`.
`{YYMMDD-HHMM}` MUST come from `bash -c 'date +%y%m%d-%H%M'`, not model memory.
Create `{reports_dir}` if absent.

---

## Report Template

```markdown
# Research: <topic>

**Ticket**: <id | n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
**Versions**: iOS target <min_ios>, Swift <Y>, <networking-lib <Z> if any>  ← from repo

## Executive Summary
<3-5 bullets — findings + recommendation>

## Codebase Context
- Existing pattern: <what's there> (path:line)
- Why current approach falls short: <reason | "n/a — greenfield">

## Findings
### Best Practices (current consensus)
### Security / Privacy  (Keychain, ATS, biometrics, ATT — flag affected pinned versions)
### Performance Insights  (numbers > vibes)
### Comparative Analysis (if multiple options)
| Option | Effort | Risk | Maintenance | Fit |

## Recommendation
**Chosen**: …   **Rationale**: …

## Implementation Sketch
- Affected modules / state shape / DI / networking / localization impact

## Sources
1. [Title](url) — checked YYYY-MM-DD

## Open Questions
- <unresolved — owner: PM / BE / Design>
```

## Constraints

- **DO NOT** implement — research only. **DO NOT** exceed 5 web searches.
- **DO NOT** recommend a library the project doesn't need (YAGNI).
- **MUST** cite sources with check dates; save under `{reports_dir}`.
- **Skip if already known** — codebase + `rules_file` answer it → say so and stop.
- **Sacrifice grammar for concision** — bullet > paragraph.
