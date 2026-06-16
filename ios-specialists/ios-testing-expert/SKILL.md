---
name: ios-testing-expert
description: "Expert iOS testing guidance for BOTH Swift Testing (@Test, #expect/#require, traits, parameterized, suites, confirmation, withKnownIssue) AND XCTest (XCTestCase, XCTAssert, expectations, XCUITest, performance) — when to use each, coexistence, XCTest→Swift Testing migration, test doubles/DI, async testing, parallelization & flakiness. Consolidated by ios-skill-consolidate. Loaded by ios-execute (apply while writing tests) and ios-code-review (critique) when work touches tests (import Testing, @Test, #expect, XCTestCase, XCTAssert). Reads .claude/ios-profile.md."
---

# iOS Testing Expert (Swift Testing + XCTest)

> **Generated skill — original wording, consolidated by `ios-skill-consolidate`** from:
> twostraws/Swift-Testing-Agent-Skill · bocato/swift-testing-agent-skill ·
> AvdLee/Swift-Testing-Agent-Skill — all MIT. **Authoritative semantics** from Apple's official
> docs (Swift Testing, XCTest, the migration guide) + WWDC sessions. Last consolidated **2026-06-16**.
> Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

How it's used: **ios-execute** *applies* this while writing tests; **ios-code-review** uses it as the
testing *critique* lens.

**Source of truth:** **Apple's official documentation** — the [Swift Testing](https://developer.apple.com/documentation/testing),
[XCTest](https://developer.apple.com/documentation/xctest), and [migration](https://developer.apple.com/documentation/testing/migratingfromxctest)
docs + WWDC sessions — is the **primary authority** and wins on any conflict. Community sources supply
practical idioms and mistake patterns, not semantics.

**Default stance:** new unit / logic / integration tests → **Swift Testing**. Keep **XCTest** for what
Swift Testing doesn't cover: **UI automation** (`XCUIApplication`) and **performance** (`measure`/`XCTMetric`),
and Objective-C test code. The two **coexist in one target** — migrate incrementally, never big-bang.

---

## 1. When to use which (+ coexistence)
- **Swift Testing** (macros + Swift concurrency) for unit/functional/logic tests. **XCTest** owns **UI tests** and **performance tests** — those have no Swift Testing equivalent and **stay**. [Apple: testing / xctest / migration]
- Both frameworks run in the **same target**; a file may `import XCTest` *and* `import Testing` during migration. Xcode 27 adds **test-framework interoperability** (call across the two) with modes — `limited` (cross-framework issues = warnings), `complete` (errors, default for new projects), `strict` (fatal), `none` (opt out) — set in Test Plan → Test Execution. [WWDC26 s267]
- Don't rewrite existing XCTest unless asked; write *new* tests in Swift Testing.

## 2. Swift Testing — basics
- `import Testing` **in test targets only**. Declare with `@Test func userCanLogOut() { … }` — no `test` prefix, no `XCTestCase` subclass; add a display name for triage: `@Test("Adding an item increases the cart count")`.
- Suites are plain types — prefer **`struct`** (value semantics avoid accidental shared state); `@Suite` is **optional** (any type with `@Test` methods is a suite) — add it only to name the suite or attach traits. Nest suites to mirror the feature structure.
- A suite's instance test methods each run on a **fresh instance**, so stored properties + `init`/`deinit` give per-test setup/teardown. Instance-method suites need a zero-arg `init` (may be `async`/`throws`).
- Put `@available(iOS 26, *)` on **individual tests, never the suite**. A test with no `#expect`/`#require` is assumed to pass.

## 3. Expectations — `#expect` vs `#require`
- **`#expect(…)`** records a failure and **continues** (see all failures in one run); **`try #require(…)`** throws and **stops** the test — use it for preconditions a later line depends on (replaces XCTest's `continueAfterFailure = false`). `#expect` captures sub-expression values for rich diagnostics. [Apple: expectations]
- `try #require(optional)` unwraps (the `try XCTUnwrap` replacement). **Never `!`-negate inside `#expect`** (`#expect(isLoggedIn == false)`, not `#expect(!isLoggedIn)`) — negation defeats the macro's diagnostics.
- Errors: name the case — `#expect(throws: GameError.notInstalled) { … }`; assert no throw with `#expect(throws: Never.self)`. The throwing form **returns the caught error** (Swift 6.1+) for further assertions. Avoid the broad `Error.self`. Manual failure: `Issue.record("…")` (the `XCTFail` replacement).

## 4. Parameterized tests
- `@Test(arguments:)` makes **each argument its own independently-runnable case** (failure names the exact input) — prefer it over an in-test `for` loop (worse diagnostics) and over copy-pasted tests.
- **Two argument collections = Cartesian product** (multiplies fast — `5×100 = 500`). For paired inputs use `zip(a, b)` — but watch its traps: **silent truncation** (stops at the shorter) and **order fragility** with `zip(allCases, allCases)`. AvdLee's tip: prefer an **array of tuples** (can't misalign) or a dict over `zip`.
- Use concrete literal expectations, not values derived from the same logic under test (that masks bugs). Keep argument arrays inline.

## 5. Traits & tags
- Tags: `extension Tag { @Tag static var networking: Self }` then `@Test(.tags(.networking))` — categorize/filter across suites. `.bug("url"/id:)` links a tracker. Suite-level traits **inherit** to contained tests.
- Conditional: `.disabled("reason")` / `.disabled(if:)` / `.enabled(if:)` (replaces `XCTSkip*`) — prefer over commenting out (reason shows in reports).
- `.timeLimit(.minutes(1))` — **only `.minutes`, not `.seconds`** (common mistake). `.serialized` opts out of parallelism (see §9).

## 6. Async & confirmation
- Tests are **async-first** — just add `async`/`throws` and `await` naturally; no `XCTestExpectation` plumbing.
- For callbacks/events: `await confirmation(expectedCount: 2) { confirm in … }` — the work **must finish before the closure returns** (a `Task`-spawning method the test can't await will fail; make it `async` or return the `Task` and `await .value`). `expectedCount` accepts a `Range` (`1...` = "at least once"; `0` = "must never happen"). Bridge completion handlers with `withCheckedContinuation`.
- Pin actor with `@Test @MainActor` (or `isolation:` on `confirmation`/`withKnownIssue`) only when the code needs it — it reduces parallelism. For multi-fire callbacks under strict concurrency, use actor-isolated counters, not shared `var`s.

## 7. Known issues
- `withKnownIssue("reason") { … }` wraps a known-failing path so it still compiles/runs and keeps signal; it **fails if no issue occurs** (tells you the bug may be fixed). Scope it **narrowly** (assertions outside still run). `isIntermittent: true` for flaky failures; `when:` / `matching:` to qualify. Replaces `XCTExpectFailure`.

## 8. Test doubles, DI & fixtures (from bocato)
- Double taxonomy (Fowler): Dummy · Fake (working shortcut, e.g. in-memory) · Stub (canned returns) · Spy (records calls) · **SpyingStub** (the common "Mock") · Mock (self-verifying expectations). **Prefer state verification over behavior verification** — less brittle; reserve behavior/Mock for delegate/interaction contracts.
- Place **test doubles and fixtures next to the protocol/model under `#if DEBUG`** (not buried in a test target) — shared across targets, zero release cost. Fixtures = `static func fixture(...)` with a default for every property (tests set only what matters). **Deterministic data only** — no `Date()`/random; inject a fixed clock/date (e.g. `swift-dependencies`).
- Inject hidden deps (`URLSession.shared`, `UserDefaults.standard`) — default-param or protocol + mock; never live-network in unit tests. For `UserDefaults`, use a per-test UUID suite name and clean up.

## 9. Parallelization & isolation
- Swift Testing runs tests **in parallel, in randomized order, by default** — across suites *and* across a parameterized test's cases. So: **no order dependence, no shared mutable global state.** This surfaces hidden coupling and speeds CI. [Apple: parallelization]
- Fix isolation **before** reaching for `.serialized` — it's a transitional tool for genuinely must-serialize tests (shared DB, migrated serial XCTest); add a TODO to remove it. Note: **`.serialized` only affects parameterized cases / a suite's tests**, and XCTest runs sync tests on the main actor while Swift Testing runs on arbitrary tasks (add `@MainActor` if needed).
- Flakiness checklist: no order reliance · no unreset globals/singletons · no `sleep`-based waiting · no hidden external deps · deterministic fixtures + stable clock/RNG · `withKnownIssue` for temporary failures.

## 10. XCTest — essentials (still needed)
- `final class FooTests: XCTestCase` with `test`-prefixed methods; `setUp()/tearDown()` (sync or `async throws`). Assertions: `XCTAssertEqual/True/False/Nil/Identical/GreaterThan…`, `XCTAssertThrowsError`, `XCTAssertNoThrow`, `XCTFail`; `try XCTUnwrap(opt)`; `continueAfterFailure` to halt on first failure. [Apple: xctest]
- Async: `XCTestExpectation` via `expectation(description:)` + `.fulfill()` + `await fulfillment(of:)` / `wait(for:timeout:)`; `expectedFulfillmentCount`, `assertForOverFulfill`. Test methods may be `async throws`. Skips: `throw XCTSkip(…)`, `XCTSkipIf/Unless`.
- **Keep in XCTest:** UI automation (`XCUIApplication`/`XCUIElement` queries) and performance (`measure(metrics:)`/`XCTMetric`) — no Swift Testing equivalent.

## 11. Migration XCTest → Swift Testing (official mapping)
| XCTest | Swift Testing |
|---|---|
| `class X: XCTestCase` / `func testFoo()` | `struct X` (or `@Suite`) / `@Test func foo()` |
| `setUp()` / `tearDown()` | `init()` / `deinit` (per-test instance) |
| `XCTAssertEqual(a,b)` / `XCTAssertTrue(x)` | `#expect(a == b)` / `#expect(x)` |
| `XCTAssertNil(x)` / `try XCTUnwrap(x)` | `#expect(x == nil)` / `try #require(x)` |
| `XCTAssertThrowsError` / `XCTAssertNoThrow` | `#expect(throws:)` / `#expect(throws: Never.self)` |
| `XCTFail("m")` / `continueAfterFailure=false` | `Issue.record("m")` / `try #require(…)` |
| `XCTestExpectation` + `fulfillment(of:)` | `confirmation(…)` (`expectedFulfillmentCount=10` → `expectedCount: 10...`) |
| `XCTSkipIf/Unless` | `.disabled(if:)` / `.enabled(if:)` traits |
| `XCTExpectFailure` (`.nonStrict()`) | `withKnownIssue` (`isIntermittent: true`) |

Order: convert assertions → drop `test` prefix / add `@Test` → classes to suites → collapse repeats into parameterized → add traits/tags. Migrate leaf tests first, one reviewable file at a time; run both in CI during transition. **No float tolerance built in** — use Swift Numerics `isApproximatelyEqual` (ask before adding the dependency).

## 12. Common mistakes / anti-patterns
- `@Suite` on every type (unnecessary); reskinning `XCTAssert*` instead of idiomatic `#expect`; `!`-negation in expectations; force-unwrap instead of `try #require`.
- Believing `.serialized` serializes a plain non-parameterized test (it doesn't); `.timeLimit(.seconds(…))` (only `.minutes`); `#expect(throws: Error.self)` instead of the specific case; `@available` on a suite.
- Testing SwiftUI views directly — test the view-model instead. Hidden deps / live networking in unit tests. Helpers that wrap `#expect` without threading `sourceLocation: SourceLocation = #_sourceLocation` (failures point at the helper, not the test). Non-deterministic fixtures (`Date()`, random). Over-stubbing; defaulting to behavior-verifying Mocks. Letting `.disabled` tests rot instead of `withKnownIssue`.

## Currency
Apple docs are the source of truth (above). Several Swift Testing features are version-gated: error-returning `#expect(throws:)` and `confirmation` ranges (Swift 6.1+), exit tests / attachments / custom scoping traits / raw identifiers (Swift 6.2+) — gate by the project's toolchain. **Apple ships a built-in "testing" specialist + an XCTest→Swift Testing *migration* skill with Xcode 27** (WWDC26 s102 State of the Union; s267), exportable via `xcrun agent skills export` — fold those in as primary once on Xcode 27 (the consolidating Mac is on 26.4). WWDC refs: [Meet Swift Testing (s10179)](https://developer.apple.com/videos/play/wwdc2024/10179/) · [Go further (s10195)](https://developer.apple.com/videos/play/wwdc2024/10195/) · [Migrate to Swift Testing (s267)](https://developer.apple.com/videos/play/wwdc2026/267/).
