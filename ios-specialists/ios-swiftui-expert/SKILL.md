---
name: ios-swiftui-expert
description: "Expert SwiftUI guidance — view composition, state (@State / @Observable / @Binding / @Environment), layout, performance, navigation, animation, accessibility, Liquid Glass. Consolidated from curated sources by ios-skill-consolidate. Loaded by ios-execute (apply while writing) and ios-code-review (critique) when work touches SwiftUI (View, @State, body: some View, etc.). Reads .claude/ios-profile.md for min_ios + ui_framework + architecture."
---

# SwiftUI Expert

> **Generated skill — original wording, consolidated by `ios-skill-consolidate`** from:
> twostraws/SwiftUI-Agent-Skill · AvdLee/SwiftUI-Agent-Skill · Dimillian/Skills (swiftui-*) ·
> arjitj2/swiftui-design-principles (all MIT). Last consolidated **2026-06-15**. Don't hand-edit —
> change `SOURCES.yaml` and re-consolidate.

How it's used: **ios-execute** loads this and *applies* it while writing SwiftUI; **ios-code-review**
uses it as the SwiftUI *critique* lens.

**Source of truth:** Apple's official SwiftUI documentation / WWDC / HIG is the **primary authority**
(see `SOURCES.yaml`) and wins on any conflict, gated to the profile's `min_ios`. The community sources
below fill in detail; genuinely contested calls are flagged, not silently picked.
**Architecture:** follow the project's `architecture` from `.claude/ios-profile.md`; where the project
is unopinionated, SwiftUI's native **Model–View** (state in the view tree, no reflexive view-models)
is the default. Don't impose MVVM where the project doesn't ask for it.

---

## 1. State & data flow
- **Baseline stack (iOS 17+):** an `@Observable` class for shared/reference state, owned via `@State`, passed down as a plain property, mutated through `@Bindable`, read app-wide via `@Environment(Type.self)`. Prefer this over `ObservableObject`/`@Published`/`@StateObject`/`@EnvironmentObject` (legacy fallback only). `@Observable` invalidates **per-property actually read**, not per-object.
- Mark `@Observable` model classes `@MainActor` (unless the module already uses MainActor default isolation).
- `@State`, `@StateObject`, `@FocusState` are **always `private`** — non-private leaks them into the synthesized init where callers' values are silently ignored.
- **Never** declare a passed-in value as `@State`/`@StateObject` — it captures only the first value and ignores parent updates (silent bug). Use `let` (read-only), `@Binding` (child mutates parent), `@Bindable` (injected `@Observable` needing bindings), or `var` + `.onChange` (reactive read).
- A view that *owns* an `@Observable` must hold it in `@State`, not `let` — otherwise it can be recreated on a parent redraw and lose state.
- Inside an `@Observable` class, annotate storage wrappers (`@AppStorage`, `@SceneStorage`, `@Query`) with `@ObservationIgnored` (they conflict with observation and otherwise won't compile); they still notify via their own mechanism.
- Pick the **narrowest** wrapper for the job; don't introduce a reference model when value state + `@Binding` suffices. Don't nest `@Observable` objects inside other `@Observable` objects.
- Don't keep frequently-changing values (geometry, timers) in `@Environment` — every change re-checks all readers.

## 2. View composition & reuse
- Extract sections into real `View` **structs**, not `@ViewBuilder` computed-props/funcs returning `some View` — a struct lets SwiftUI skip `body` when inputs are unchanged; a builder func re-runs on every parent change. (Strong consensus across all sources.)
- Use `@ViewBuilder` only for small static sections or genuine multi-type branching.
- Container views take `@ViewBuilder let content: Content` (diffable), **never** `let content: () -> Content` (closures can't diff → always re-render).
- Keep `body` pure and cheap: no filtering/sorting/mapping/formatter-creation inline — move it to init/model/helpers. Move button actions and business logic out of `body`/`.task`/`.onAppear` into methods (testable; `body` "reads like UI").
- One primary type per file; split files past ~300 lines; flag a `body` longer than ~one screen.
- Avoid `AnyView` — use `@ViewBuilder`/`Group`/generics. Avoid the popular `if`-based conditional-modifier `View` extension (returns different types → destroys identity).
- `Label` for icon+text; `.overlay`/`.background` to *decorate* a primary view, `ZStack` for *peer* composition. `.compositingGroup()` before `.clipShape()` on layered views to avoid corner fringing.
- Centralize fonts/colors/spacing/timings in a shared constants type; don't hard-code padding/spacing ad hoc.

## 3. Layout
- Never size from `UIScreen.main.bounds` — views must work in any context (sheet/popover/embedded). Prefer `containerRelativeFrame`, `.visualEffect`, the `Layout` protocol, or `onGeometryChange` (iOS 17+); `GeometryReader` only when layout truly depends on measured geometry (it's not deprecated, just last-resort).
- One full-width child → `.frame(maxWidth: .infinity, alignment:)`, not `HStack { view; Spacer() }`.
- Give frames flexibility; avoid fixed frames that break across device sizes / Dynamic Type. Fix layout properly instead of `minimumScaleFactor` hacks.
- `Form` (`.formStyle(.grouped)`, `Section`, `@FocusState`) for settings/entry; avoid heavy custom layouts inside `Form`/`List` rows. `ContentUnavailableView` for empty states (`.search` variant with `searchable`).
- **Design polish (arjitj2):** spacing on a grid (4/8/12/16/20/24/32/40/48); ≤5 distinct font sizes (hierarchy via *weight*, not size); one font `design:` everywhere; semantic system colors (`Color(.secondarySystemBackground)`, `.foregroundStyle(.secondary)`) so it adapts to light/dark + a11y; equal `lineWidth` for paired strokes; 10pt card corners; system `Divider()` over custom.

## 4. Performance
- **Toggle modifier *values* with a ternary**, not `if/else` that swaps views — branching creates `_ConditionalContent`, destroys identity, recreates platform views. Use `if` only for genuinely different views.
- SwiftUI does **not** diff before updating: in hot paths (scroll/gesture/`onPreferenceChange`) guard `if new != current` before assigning; store a threshold *flag*, never raw continuous scroll offset, in `@State`.
- Pass the **specific values** a view needs, not whole models — narrows the invalidation fan-out. Per-item `@Observable` (one per row) instead of one shared array means mutating a row doesn't re-run every row's `body`.
- Stable identity in `ForEach`/`List`: real domain IDs; **never `.indices`** for dynamic content (can crash on removal); `id: \.self` only for stable `Hashable` values; keep a constant view-count per element. Don't `.filter` inline in `ForEach` — pre-filter + cache.
- Lazy containers (`LazyVStack`/`LazyVGrid`) for large/unknown counts; plain stacks for small fixed content. `.task {}` (auto-cancels) over `onAppear` for async.
- Decode/downsample images **off the main thread** and store the result; never `UIImage(data:)` in `body` or full-size images in rows. Cache formatters in a `static let`.
- `.equatable()` / `Equatable` only when the body is expensive *and* inputs are small value-types (keep `==` in sync). Off-main closures (`Shape.path`, `visualEffect`, `Layout`) must be `Sendable` — capture values, don't touch `@MainActor` state.
- Debug with `Self._printChanges()` / `_logChanges()` (remove before shipping). Dense widget visuals → one `Canvas` pass, not hundreds of nested views (widgets have a ~30 MB limit).

## 5. Navigation & presentation
- `NavigationStack` / `NavigationSplitView` only — **never** `NavigationView` (deprecated). Value-based: `NavigationLink(value:)` + `.navigationDestination(for:)`; register each type's destination **exactly once** and **never mix** `navigationDestination` with `NavigationLink(destination:)` in one hierarchy.
- Per-tab `NavigationStack` with its own path + router; route enums `Hashable`, store lightweight data (never view instances). Centralize destination mapping in one modifier. Reset `path` on account/logout changes.
- **`.sheet(item:)` over `.sheet(isPresented:)`** for model-driven content (safe unwrap). Multiple sheets → one `Identifiable` enum + a single `.sheet(item:)` `switch`, not N booleans. Sheets **own their dismiss** via `@Environment(\.dismiss)` — don't prop-drill `onSave`/`onCancel`. Wrap a modal in `NavigationStack` if it must push.
- `confirmationDialog` (not `actionSheet`); single "OK" alert → omit the button. `Inspector` (iOS 17+) for trailing panels; `Tab` API (iOS 18+) not `tabItem`, bound to an enum.

## 6. Animation & transitions
- **Always** `.animation(_:value:)` — the no-value `.animation(_:)` is deprecated and animates everything. Place it *after* the properties it should affect. Implicit for value-tied changes; `withAnimation` for event-driven.
- Transitions animate insert/remove and need an animation context **outside** the `if` (parent `.animation(value:)` or `withAnimation`); `.asymmetric`, `.combined(with:)` to compose.
- Prefer GPU transforms (`scaleEffect`/`offset`/`rotationEffect`) over animating layout (`frame`/`padding`); scope to the smallest subview; never animate every scroll frame.
- Custom `Animatable` **must** implement `animatableData` or it jumps to the final value (iOS 26 `@Animatable` macro synthesizes it; `@AnimatableIgnored` to exclude). Sequence via `phaseAnimator`/`keyframeAnimator` or `withAnimation { } completion:` — not `DispatchQueue.asyncAfter`. `.spring`/`.bouncy` for interactive, `.easeInOut` for appearance, `.linear` only for progress.

## 7. Accessibility
- **`Button` over `.onTapGesture`** for anything tappable (free VoiceOver/focus/traits); reserve gestures for when you need location/count (then add `.accessibilityAddTraits(.isButton)`).
- Icon-only controls hurt VoiceOver — keep text (`Button("Label", systemImage:)`, `.labelStyle(.iconOnly)` to stay visually icon-only). Decorative images `Image(decorative:)` / `.accessibilityHidden`; informative images get `.accessibilityLabel`.
- Dynamic Type: built-in text styles; custom via `Font.custom(_:size:relativeTo:)`, non-text metrics via `@ScaledMetric` — avoid fixed sizes for primary content. Respect Reduce Motion (swap big motion for opacity) and Differentiate-Without-Color (add icons/patterns).
- Group with `accessibilityElement(children:)`; use the dedicated `.accessibilityLabel/Value/Hint/AddTraits` (not the deprecated generic `.accessibility(...)`). Add accessibility labels/ids once the UI is interactive (and expose ids from the profile's `accessibility_ids` for UI tests).

## 8. Concurrency in SwiftUI
- `@Observable`/UI-touching view-models are `@MainActor`; `.task {}` ties async work to view lifetime. Off-main SwiftUI closures must be `Sendable` (capture values). Prefer `async`/`await`/actors over GCD; treat `CancellationError` as normal (never surface it as a user error); never start network work directly from `body`. *(Deep concurrency belongs to a future `ios-concurrency-expert` — keep this SwiftUI-specific.)*

## 9. Modern API / always-never (currency)
- `foregroundStyle` not `foregroundColor`; `clipShape(.rect(cornerRadius:))` not `cornerRadius()`; `.topBarLeading`/`.topBarTrailing` not `.navigationBarLeading/Trailing`; 2-param (or 0-param) `onChange`, never the deprecated 1-param; `@Entry` macro for custom Environment/Focus keys; `Text(verbatim:)` only for non-localized literals; `Date.now`, `Task.sleep(for:)`, `count(where:)`, `.scrollIndicators(.hidden)`. `bold()` over `fontWeight(.bold)`. Hierarchical styles (secondary/tertiary) over manual opacity.

## 10. Testing & previews
- `#Preview` macro (not `PreviewProvider`), one per meaningful state (default/empty/error/loading). iOS 18+ `@Previewable @State` for inline interactive state. Previews **self-contained** — never live network/disk; expose `.sample`/`.preview` fixtures and inject mocks via protocols. Wrap previewed destinations in `NavigationStack`.
- Put core logic in models/services so it's unit-testable; UI tests only when a unit test can't reach it.

## 11. Liquid Glass (iOS 26+)
- **Adopt only when asked** — never proactively convert UI. Always `#available(iOS 26, *)`-gate with a materials fallback (`.background(.ultraThinMaterial, in:)`).
- `.glassEffect(.regular.tint(…).interactive(), in: .rect(cornerRadius:))` — default shape is `.capsule`; `.interactive()` only on truly interactive elements; **there is no `.prominent` glass variant** (common LLM hallucination — raise tint opacity instead). Apply *after* layout/appearance modifiers.
- Wrap grouped glass in `GlassEffectContainer` (glass can't sample other glass across containers); morph via `@Namespace` + `.glassEffectID`; `.buttonStyle(.glass)` / `.glassProminent` for buttons.

## Contested / judgment calls
- **View-models:** Dimillian + AvdLee favor SwiftUI-native MV (no reflexive VM); twostraws is neutral. → Defer to the project's `architecture`; default MV only when the project is unopinionated.
- **Image caching:** older sources say "roll your own AsyncImage cache" — see Currency watch (iOS 27 changes this).
- **arjitj2** is a *design-polish* lens (opinionated toward one aesthetic, e.g. `.monospaced`) — take its *rules* (consistency, fewer values, semantic colors), not its specific aesthetic, as universal.

## Currency
Apple's official docs are the **source of truth** (above). The community sources here reflect an
**iOS 17+ Observation** baseline. No version-specific "what's new" claims are included until verified
directly against Apple. Note: Apple ships its **own SwiftUI Specialist Skill with Xcode 27** (WWDC 2026
session 269), exportable via `xcrun agent skills export` — that's the ideal primary input; fold it in
once a machine on Xcode 27 can export it. (Specific iOS-27 API claims remain excluded until verified.)
