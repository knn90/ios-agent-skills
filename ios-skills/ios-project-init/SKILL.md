---
name: ios-project-init
description: "One-time bootstrap for the ios-* skill suite. Creates .claude/ios-profile.md — the single source of project-specific facts every other ios-* skill reads. Greenfield repos → prescriptive (decide conventions up front). Existing apps → descriptive (detect + record). Use before first running ios-scout/plan/execute/code-review/resolve on a project, or when the profile is missing/stale."
argument-hint: "[--greenfield | --detect]"
---

# Project Init — ios-* suite bootstrap

Produce `.claude/ios-profile.md`: the one file that makes the generic `ios-*` skills
concrete for THIS project. Run once per project (re-run to refresh a stale profile).

**Principles:** YAGNI + KISS + DRY. The profile holds facts, not opinions — keep it short.

---

## Two modes

| Repo state | Mode | Profile is… |
|---|---|---|
| Empty / no source files yet | **greenfield** (prescriptive) | a record of decisions you make up front |
| Has `*.xcodeproj`/`*.xcworkspace`/`Package.swift` + source | **detect** (descriptive) | a record of reality read from the code |

Auto-detect the mode:
```
Glob **/*.xcodeproj, **/*.xcworkspace, **/Package.swift
Glob **/Sources/**/*.swift, **/*.swift   (any real source?)
```
- Source present → `--detect`. Empty/scaffold only → `--greenfield`.
- `$ARGUMENTS` may force `--greenfield` / `--detect`.

If a profile already exists, show it and ask (via `AskUserQuestion`): keep / refresh / overwrite.

---

## Mode A — Detect (existing app)

Read the codebase, fill the profile from evidence. **Verify every value — never guess.**

| Profile field | How to detect |
|---|---|
| `project_file` | the `.xcworkspace` else `.xcodeproj` else `Package.swift` |
| `source_roots` / `test_roots` | top-level dirs containing `*.swift` / `*Tests` |
| `architecture` | grep: `import ComposableArchitecture`→TCA · `Coordinator`/`FlowViewModel`→MVVM-C · `@Observable`/`ObservableObject` only→plain SwiftUI · `Presenter`+`Interactor`→VIPER |
| `state_type` | grep for a shared state enum/struct (e.g. `ViewState<`), TCA `Reducer`, else none |
| `di` | grep `Factory`, `swift-dependencies`, a `DependencyContainer`, else manual |
| `navigation` | `NavigationStack`/`Router` vs `Coordinator`/`Flow` vs UIKit push |
| `networking` | `Apollo`/`*.graphql`→Apollo · `URLSession`/`Moya`→REST · none |
| `localization` | `*.xcstrings`→StringCatalog · `R.swift` · `NSLocalizedString` · none |
| `accessibility_ids` | grep for an AccessibilityIdentifiers module/enum |
| `feature_flags` / `crash_reporting` | grep `RemoteConfig`/`LaunchDarkly`, `Crashlytics`/`Sentry` |
| `generated_paths` | files with "generated" headers, GraphQL `API/`+`Mocks/`, `*.pbxproj` |
| `verify_command` | existing `scripts/*verify*`, `Makefile` test target, `fastlane` lane, else propose `xcodebuild test -scheme <S> -destination 'platform=iOS Simulator,name=iPhone 16'` |
| `rules_file` | `CLAUDE.md` else `Docs/Architecture.md` else none |
| `ticket_system`/`ticket_pattern` | branch names + `git log` (e.g. `ABC-123`), or ask |
| `high_rigor_domains` | grep for `Checkout`/`Payment`/`Auth`/`Profile`; default `[auth, PII]` if none |
| `default_base_branch` | `git symbolic-ref refs/remotes/origin/HEAD` (repo default), else `main` |
| `pr_tool` | `gh` if the GitHub CLI is installed + authed, else `none` |

Use `Explore`/`Grep`/`Glob` (or invoke `ios-scout` if the repo is large). Confirm the
detected `architecture`, `verify_command`, and `high_rigor_domains` with the user before
writing — these three drive the most downstream behaviour.

---

## Mode B — Greenfield (new app)

Nothing to detect. **Decide** the conventions via `AskUserQuestion`, then record them.
This is the highest-leverage moment in the project's life — treat the architecture choice
with real rigor (it's a multi-year commitment), not a quick menu pick.

Ask (bundle into 2-4 `AskUserQuestion` groups):

1. **Architecture** — TCA · MVVM-C · plain SwiftUI `@Observable` · other.
   For each, state the trade-off honestly (testability, boilerplate, team familiarity,
   iOS-version fit). If the user is unsure, recommend the boring proven default for their
   stated team size and push back on novelty. *(This is `ios-brainstorm` applied to
   foundations — borrow its brutal-honesty stance.)*
2. **UI framework + min iOS** — SwiftUI / UIKit / mixed; deployment target.
3. **Networking** — REST(URLSession) · GraphQL(Apollo) · none yet.
4. **DI · navigation · localization** — pick or defer (`none` is valid early).
5. **Verify command** — default suggestion `xcodebuild build` (no tests yet) or
   `xcodebuild test …` once a test target exists. May be left **empty** → build-only.
6. **Ticket system** + pattern, or `none`.
7. **HIGH-RIGOR domains** — which of checkout/payment/auth/PII this app will have.

Do **NOT** generate a `verify.sh` or any wrapper script. The verify mechanism is just the
`verify_command` string — keep it as configuration, not a committed artifact.

Optionally (ask first) create the docs scaffold the other skills expect:
`{docs_root}/Plans/` and `{docs_root}/Reports/`, and a starter `rules_file` (CLAUDE.md)
capturing the always/never rules implied by the chosen architecture.

---

## Output

Write `.claude/ios-profile.md` from `ios-profile.template.md`, filled in. Then:

```
✓ Profile written: .claude/ios-profile.md
- Mode: detect | greenfield
- Architecture: <…>   Networking: <…>   Verify: <command or "build-only (unset)">
- HIGH-RIGOR domains: <…>
- Specialists available: <list or none>

Next: the ios-* skills are now live for this project.
- Find code            → ios-scout
- Plan a feature       → ios-plan
- Implement            → ios-execute
- Resolve end-to-end   → ios-resolve <ticket | "description">  (scout→plan→execute→review→PR)
```

## Greenfield follow-through

For the first 1-2 features there's no prior art to DRY against — that's expected. The
other skills relax their "match existing patterns" checks until ≥1 reference feature
exists, then resume normal rigor. Once `verify_command` points at a real test target,
the verification gate goes from build-only to full.

## Constraints

- **DO NOT** create a verify script — verify is a profile string only.
- **DO NOT** guess detected values — confirm architecture/verify/rigor with the user.
- **DO NOT** invoke implementation skills — this only writes the profile (+ optional docs scaffold).
- **MUST** keep the profile minimal — facts that vary between apps, nothing else.
