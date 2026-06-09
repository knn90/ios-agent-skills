---
# ios-profile.md — single source of project-specific facts for the ios-* skill suite.
# Every ios-* skill reads this FIRST. Copy this file to `.claude/ios-profile.md`
# at the project root and fill it in (or let `ios-project-init` generate it).
#
# Rule of thumb: if a value differs between two iOS apps, it belongs HERE, not in a skill body.

app_name:                 # e.g. Acme
state: existing           # existing | greenfield

# ── Repo layout ────────────────────────────────────────────────
project_file:             # App.xcodeproj | App.xcworkspace | Package.swift
source_roots: []          # e.g. [App/, Packages/*/Sources]
test_roots: []            # e.g. [AppTests/, AppUITests/]
generated_paths: []       # NEVER edit — e.g. [*+Generated.swift, GraphQL/API/, *.pbxproj]
docs_root:                # Docs/ | none
plans_dir:                # Docs/Plans | .claude/plans
reports_dir:              # Docs/Reports | .claude/reports
rules_file:               # CLAUDE.md | Docs/Architecture.md | none  (always/never rules)

# ── Architecture ───────────────────────────────────────────────
architecture:             # MVVM-C | TCA | VIPER | plain-SwiftUI-Observable | other
state_type:               # e.g. ViewState<Value,Error> | TCA Reducer | @Observable | none
di:                       # DependencyContainer | Factory | swift-dependencies | manual-init
navigation:               # Coordinator/Flow | NavigationStack/Router | UIKit-push
ui_framework:             # SwiftUI | UIKit | mixed
min_ios:                  # e.g. 16

# ── Integrations (set to `none` if unused — skills skip the related checks) ──
networking:               # Apollo-GraphQL | URLSession-REST | Moya | none
localization:             # StringCatalog | R.swift | NSLocalizedString | none
accessibility_ids:        # module/convention name | none
feature_flags:            # Firebase-RemoteConfig | LaunchDarkly | none
crash_reporting:          # Crashlytics | Sentry | none
ticket_system:            # Jira | GitHub | Linear | none
ticket_pattern:           # regex, e.g. ALFMOB-\d+  (used to detect a ticket id in args/branch)
ticket_fetch:             # MCP tool or CLI to fetch a ticket | none

# ── Verification ───────────────────────────────────────────────
# Whatever this project actually runs to prove a change is correct.
# Free-form: xcodebuild test, fastlane test, swift test, make test, ./scripts/foo.sh …
# Empty/unset → skills fall back to BUILD-ONLY and say so explicitly in the report.
verify_command: |

# ── Rigor ──────────────────────────────────────────────────────
# Domains that force adversarial review + correctness audit every time.
high_rigor_domains: [checkout, payment, auth, PII, money]

# ── Specialist review skills (optional) ────────────────────────
# code-review delegates to these IF installed. Omit any not present.
specialists: []           # e.g. [swiftui-expert, swift-concurrency, swift-testing-expert, apollo-ios]
---

# Project Notes (free text)

Anything a skill should know that doesn't fit a field above — naming conventions,
forbidden patterns, "always do X / never do Y" rules specific to this app.
