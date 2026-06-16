---
name: ios-concurrency-expert
description: "Expert Swift Concurrency guidance — async/await, actors & isolation, Sendable & data-race safety, structured concurrency (Task/TaskGroup/async let), @MainActor & global actors, Swift 6 language mode, region-based isolation, @concurrent, cancellation, continuations, AsyncSequence/AsyncStream, task-local values, testing. Consolidated by ios-skill-consolidate. Loaded by ios-execute (apply while writing) and ios-code-review (critique) when work touches concurrency (async/await, actor, Sendable, @MainActor, Task, isolation). Reads .claude/ios-profile.md."
---

# Swift Concurrency Expert

> **Generated skill — original wording, consolidated by `ios-skill-consolidate`** from:
> AvdLee/Swift-Concurrency-Agent-Skill · twostraws/Swift-Concurrency-Agent-Skill ·
> Dimillian/Skills (swift-concurrency-expert) — all MIT. **Authoritative semantics** are from
> Swift.org (the Swift 6 migration guide + SE proposals). Last consolidated **2026-06-15**.
> Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

How it's used: **ios-execute** loads this and *applies* it while writing concurrency code;
**ios-code-review** uses it as the concurrency *critique* lens.

**Source of truth:** Swift Concurrency is a *language* feature, so **Swift.org — the Concurrency
book chapter, the Swift 6 Concurrency Migration Guide, and the SE proposals — is the primary
authority** and wins on any conflict. The community sources supply practical LLM-mistake patterns
and review heuristics, not semantics. **Baseline: Swift 6.2 / Xcode 26 "Approachable Concurrency"**
(default `@MainActor` isolation, `nonisolated async` on the caller's actor, region-based isolation).
Annotate isolation explicitly when code must compile across toolchains.

---

## 1. Data-race safety model (the foundation)
- Every declaration sits in exactly one **isolation domain**: **nonisolated**, **actor-isolated**, or **global-actor-isolated** (e.g. `@MainActor`). [SE-0306/0316]
- The one enforced invariant: **non-`Sendable` mutable state must not be reachable from more than one isolation domain at a time.** Anything crossing a boundary must be `Sendable` (or provably safe to transfer under region-based isolation). [SE-0302/0414]
- **Isolation ≠ atomicity.** It serializes access; it does **not** make a multi-step sequence atomic across an `await`. [SE-0306]
- Swift 6 mode = **compile-time** data-race safety (warnings→errors). Adoptable **per-module**, incrementally. Swift prevents **data races**, not logic **race conditions** — you still sequence correctly. Catch residual races with Thread Sanitizer.

## 2. async/await semantics & pitfalls
- `await` marks a **potential suspension point**; after resuming, work may continue on a **different thread** (never assume thread identity). A synchronous function **cannot** call async; long synchronous work inside an async function **still blocks** its thread. [SE-0296]
- Prefer `async`/`await` over completion-handler APIs when both exist (the compiler enforces results; no forgotten callbacks). Order is `try await`.
- `async let` for a **fixed, compile-time-known** set of parallel work (starts immediately, awaited at point of use, auto-cancels on scope exit). Use **typed throws** for precise error contracts where it helps.
- Don't reason about threads (`Thread.current` is unavailable in async contexts under Swift 6) — reason about **isolation domains**.

## 3. Actors & isolation — reentrancy is the #1 bug
- An actor serializes access to its mutable state; external access is `await` + values crossing must be `Sendable`. Actors are implicitly `Sendable`. [SE-0306]
- **Reentrancy (the headline mistake):** at **every `await`** inside an actor, other queued work can run — so **actor state can change across a suspension.** Never assume invariants hold after `await`. Complete mutations in synchronous steps; **re-read/re-check state after each `await`.** The classic bug: check-then-act (`if cache[k] == nil { cache[k] = await load() }`) → duplicate work + a force-unwrap crash. Fix: capture into a local, re-check, then assign. Dedup in-flight work by storing the `Task` handle.
- `nonisolated` only for genuinely immutable / thread-safe members. **Avoid `MainActor.assumeIsolated`** (crashes if wrong) — prefer explicit `@MainActor` / `await MainActor.run`. `assertIsolated()` is fine (debug-only).
- For **synchronous** fine-grained locking, use `Mutex` (`import Synchronization`, iOS 18+) — also a way to give a non-Sendable type Sendable access; choose an `actor` when you're in an async context or want logical isolation.

## 4. @MainActor & global actors
- `@MainActor` **only** for truly UI-bound code — **never blanket-apply to silence warnings.** Replace `DispatchQueue.main.async` with a `@MainActor` function (best) or `await MainActor.run { }`.
- Isolation **inference propagates**: subclass of a `@MainActor` class, conformance to a `@MainActor` protocol (SwiftUI `View` is one → the whole type is `@MainActor`), actor-isolated property-wrapper storage. Don't redundantly re-annotate. It does **not** propagate to closures passed to non-isolated functions. [SE-0316]
- **Default `@MainActor` isolation** (SE-0466, Swift 6.2): a *per-module* setting that infers `@MainActor` on unannotated decls — recommended for **app** targets (not frameworks/libraries) to kill false-positive races in mostly-main code. **Key misconception:** enabling it does **not** move `URLSession`/networking onto the main actor (that code lives in other modules); suspending I/O doesn't block main.

## 5. Sendable & data-race safety
- Cross-boundary values must be `Sendable`. Structs/enums conform automatically when members are Sendable (public non-frozen needs explicit conformance). A **class** needs `final` + only immutable `Sendable` `let`s; actor-isolated classes are implicitly Sendable. `@Sendable` closures capture **by value** (use capture lists to snapshot). [SE-0302]
- **`@unchecked Sendable` = last resort** — it hides the race, doesn't fix it. Only legitimate for types with real internal synchronization (lock/`Mutex`/atomics); document the invariant + leave a migration note. Before reaching for it, check whether **region-based isolation** already makes the code compile.
- **Region-based isolation (SE-0414):** the compiler allows transferring a non-`Sendable` value across a boundary when the sender provably doesn't use it (or anything reaching it) afterward. `sending` parameters enforce ownership transfer (value unusable after). Globals must be safe: `@MainActor`, immutable `static let`, or `nonisolated(unsafe)` (last resort, pair with `private(set)`).

## 6. nonisolated / @concurrent — offloading work (Swift 6.2, SE-0461)
- A `nonisolated async` function now **runs on the caller's actor by default** (was: always the global executor). So calling such methods on actor state no longer triggers Sendable errors and **doesn't leave the actor**. `nonisolated(nonsending)` is the explicit form.
- **`@concurrent`** opts back into running on the **global concurrent pool** — use it for genuinely CPU-heavy work (parsing, image processing, large transforms), **not** ordinary async I/O (which already suspends). To background a function: make the type `nonisolated`, add `@concurrent`, `async`, and `await` at call sites.
- Diagnostic "Sending 'x' risks data races" → fix order: (1) let region-based isolation handle it (stop using the value after sending), (2) mark the param `sending`, (3) make the type `Sendable`, (4) `nonisolated(nonsending)`, (5) last resort `@unchecked Sendable`.

## 7. Structured concurrency
- **Prefer structured over unstructured.** `async let` = fixed, heterogeneous parallel work; `withTaskGroup`/`withThrowingTaskGroup` = dynamic, same-type work. **`Task {}` in a loop is a smell** (loses cancellation propagation, can't await all, leaks on failure) → use a group. [SE-0304]
- **Throwing-group error propagation is NOT automatic** — you must iterate (`for try await` / `next()`); the first thrown error implicitly cancels the remaining children. For partial results, `catch` *inside* each child and return a `Result`. `withDiscardingTaskGroup` for fire-and-forget.
- `Task {}` inherits priority, task-locals, and **actor/execution context**; **`Task.detached` inherits none — last resort** (people usually want `@concurrent`). Timeout = race `operation()` vs `Task.sleep` in a throwing group, then **`group.cancelAll()`** once the winner returns.

## 8. Cancellation (cooperative)
- `cancel()` only **sets a flag**; it never interrupts execution. Long work must check **`Task.isCancelled`** (Bool) or **`try Task.checkCancellation()`** (throws) at safe breakpoints. CPU loops with no `await` **never** observe cancellation unless you check. [SE-0304]
- Parent cancellation propagates to **structured** children, but children must still check; **unstructured `Task`/`Task.detached` must be cancelled via a stored handle** (and in `deinit`). `withTaskCancellationHandler` bridges legacy cancel APIs (its `onCancel` can run on any thread).
- **Don't swallow `CancellationError`** (`try?` + `Task.sleep` in a loop hides it → loop runs forever). Filter it explicitly; cancellation is a normal lifecycle event, not a user-facing error.

## 9. Continuations
- **Resume a continuation EXACTLY once on every path.** Zero → the awaiting task hangs forever; twice → the *checked* variant traps (`unsafe` = UB). [SE-0300]
- **Default to `withCheckedContinuation` / `…ThrowingContinuation` everywhere, including production** — the runtime checks surface misuse (`SWIFT TASK CONTINUATION MISUSE` log / trap). Switch to `withUnsafe…` only after profiling proves a hot path. Audit early returns, thrown errors, dealloc, and "callback may never fire" timeouts. Don't wrap an API that already has an async overload.

## 10. AsyncSequence / AsyncStream
- Iterate with `for await` (`for try await` if throwing); terminal state is sticky (once `nil`/throws, always `nil`), single-pass. [SE-0298]
- Prefer **`AsyncStream.makeStream(of:)`** (returns `(stream, continuation)`) to bridge delegates/callbacks/manual events. **Always `continuation.finish()` exactly once** or consumers hang; set **`continuation.onTermination`** for cleanup. **Default buffer is `.unbounded`** → set `.bufferingNewest(n)`/`.bufferingOldest(n)` for high-throughput producers. **Single-consumer only** — for multi-consumer use `AsyncChannel` (swift-async-algorithms) or broadcast via an `@Observable`.
- For time/combining operators use **swift-async-algorithms**: `debounce`, `throttle`, `merge`, `combineLatest`, `zip`, `chain`. Don't hand-roll debounce with `Task.sleep` (spawns a task per event → out-of-order). Don't wrap a single-value API in a stream — use a plain async function.

## 11. Task-local values
- `@TaskLocal static var x` — set only via **scoped binding** `$x.withValue(v) { … }` (immutable for the task; nested bindings shadow). **Inherited** by `async let`/group children and `Task { }`, but **NOT by `Task.detached`** (re-bind there). [SE-0311]
- Reads are synchronous but relatively costly (tree traversal) — **hoist out of loops.** Great for injecting test dependencies / scoped config instead of shared mutable globals.

## 12. Swift 6 migration (incremental, per-module)
- Swift 6 doesn't change how concurrency works — it **enforces** the rules strictly. Adopt gradually: strict-concurrency **Minimal → Targeted → Complete**, then flip the module to Swift 6 mode; a module can adopt independently of its dependencies. [Swift.org migration guide]
- **Validation loop:** build → fix **one** category → rebuild → test → next. Small, reviewable PRs; never batch unrelated fixes with a migration. `@preconcurrency import` to silence Sendable warnings from modules you don't control (document why; revisit). Use Xcode's "Migrate" mode / `swift package migrate`.
- **"Approachable Concurrency" caution:** don't flip the whole bundle on an existing project — migrate feature-by-feature first. The bundle enables default `@MainActor` (apps), `nonisolated async` on the caller's actor, region-based isolation inference, etc.

## 13. Testing async code
- **Swift Testing first** (`@Test`, `#expect`, `confirmation`). Make tests `async`; **don't** wrap in `Task {}`, semaphores, or `XCTestExpectation`. **Never `sleep` to "wait"** (flaky) — await the real work, or bridge a callback with `withCheckedContinuation`/`confirmation()`.
- Mark a test/suite `@MainActor` when the code under test requires it. To test internal `Task`s deterministically, make the API `async`/return the handle, or use `withMainSerialExecutor` + `await Task.yield()` (needs `.serialized`). **`.serialized` only affects *parameterized* tests** (a common LLM misconception). Test cancellation via the production check (`#expect(throws: CancellationError.self)`). Enable Thread Sanitizer in a CI job for runtime races.

## 14. Common mistakes / anti-patterns (consolidated)
- **Actor reentrancy / force-unwrap of actor state after `await`** — the top latent crashes (§3).
- **Blanket `@MainActor`** to silence warnings; **`Task.detached` "to go background"** (use `@concurrent`, §6); **`@unchecked Sendable`** hiding a real race on a class with unsynchronized `var`s.
- **`Task {}` in a loop** (use a group); **swallowed errors** in `Task { try await … }` (handle inside); **unbounded `AsyncStream` buffer**; **ignoring `CancellationError`**.
- **Blocking the main actor with synchronous CPU work** — more likely under 6.2 since `nonisolated async` stays on the caller; offload with `@concurrent`.
- **Manual `Task.sleep` debounce**; reasoning by thread instead of isolation; treating `await` as blocking (it suspends, freeing the thread; tasks aren't pinned to threads).

## Contested / judgment calls
- **UI ↔ async boundary:** keep async work in models/services; let the UI react to state synchronously (SwiftUI action callbacks are synchronous by design). Defer to the project's `architecture`.
- **Core Data:** never pass `NSManagedObject` across actors — pass the `Sendable` `NSManagedObjectID` (or a snapshot); always use `perform { }`; `@unchecked Sendable` on a managed object does **not** make it safe.
- **Currency:** content is Swift 6.2 / Xcode 26 era. Some APIs (`@concurrent`, `Task.immediate`, `isolated deinit`, default `@MainActor`) require 6.2+ — gate by the profile's toolchain and annotate isolation explicitly for cross-version code.
