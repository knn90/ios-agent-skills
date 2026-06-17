---
name: ios-solid-expert
description: "Expert SOLID + clean-decoupling review for iOS/Swift — the 5 principles by name (SRP, OCP, LSP, ISP, DIP), dependency inversion + composition-root DI, decoupling patterns (Decorator / Composite / Adapter / Facade), and framework isolation (keep UIKit/SwiftUI/CoreData out of the domain). Consolidated by ios-skill-consolidate. Loaded by ios-code-review (critique) and ios-execute (apply) when a change adds/alters types, protocols, services, view-models, or DI/composition wiring. Delegates *which architecture pattern* to the architecture lens. Reads .claude/ios-profile.md."
---

# SOLID / Decoupling Expert (iOS / Swift)

> **Generated skill — original wording, consolidated by `ios-skill-consolidate`.** Authority: **Robert C.
> Martin's SOLID + Clean Architecture** (canonical, general SW engineering) translated to **Apple's
> protocol-oriented design**. Worked example: `essentialdevelopercom/essential-feed-case-study` (MIT).
> Also references `ramziddin/solid-skills` (no license — learn-from/cite only, not vendored). Last
> consolidated **2026-06-16**. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

How it's used: **ios-code-review** routes structural/architectural slices here; **ios-execute** applies
this while writing. **Scope = principles, not pattern choice** — *which* architecture (MVVM / Clean /
VIPER / TCA) is decided by the architecture lens; this skill judges the SOLID/decoupling *beneath* any pattern.

**Source of truth:** SOLID is **general** software engineering (Robert C. Martin) — Apple takes no
official app-architecture stance. The Swift-idiomatic translation is **protocol-oriented design**
(Apple: "Design protocol interfaces in Swift", the Swift book's Protocols chapter). Gate to the
project's `architecture` + conventions in `rules_file`.

**North star (YAGNI):** SOLID serves *change*, not ceremony. An abstraction earns its keep only with
≥2 implementations **or** a real test seam. Don't add a protocol with one conformer and no test — that's speculative generality.

---

## The five principles (rule · Swift idiom · smell → fix)

**1. SRP — Single Responsibility.** One reason to change / one actor a type answers to. *Smell:* a
view-model that fetches **and** parses **and** caches **and** formats; a `Manager`/`Helper` doing
everything. *Fix:* one use-case/service per responsibility; the view-model only orchestrates.

**2. OCP — Open/Closed.** Add behavior by adding a type, not by editing existing code. *Swift:* a
protocol + a new conformer / injected strategy. *Smell:* a `switch kind { … }` you must edit for every
new case. *Fix:* polymorphism / strategy / a registry.

**3. LSP — Liskov Substitution.** A conformer must be usable everywhere the abstraction is, with no
surprises. *Swift:* prefer protocol composition over class inheritance. *Smell:* a conformer that
`fatalError()`s or no-ops part of the protocol ("unsupported"). *Fix:* split the protocol (→ ISP).

**4. ISP — Interface Segregation.** Many small, client-focused protocols over one fat one. *Swift:*
`FeedLoader`, `ImageCache` — compose with `&`. *Smell:* a god protocol with 15 methods; conformers
stubbing half of them. *Fix:* split by what each client actually uses.

**5. DIP — Dependency Inversion.** Policy depends on **abstractions it owns**, not on concrete
details. *Swift:* the **domain declares** `protocol FeedLoader`; the networking/persistence module
*implements* it; the composition root wires them. *Smell:* a view-model that `import`s `URLSession`/
CoreData or `init`s a concrete client. *Fix:* invert — depend on a domain protocol, inject the impl.

## Dependency injection & the composition root
- **Constructor injection.** The **composition root** (App/Scene entry, or a `Factory`/`*Composer`) is
  the *only* place that knows concrete types and assembles the graph. Everything else takes abstractions.
- *Smells:* service locators / singletons reached from policy code; **constructor over-injection**
  (≥4–5 deps ⇒ the type does too much → split it, or hide a subsystem behind a Facade).
- Default-param or protocol injection gives testability without ceremony (no live `URLSession.shared`,
  `UserDefaults.standard`, `Date()` reached from inside).

## Decoupling patterns (the Essential-Developer toolkit)
- **Decorator** — add a cross-cutting concern (logging, analytics, main-thread dispatch, caching) with
  the *same* protocol in and out, without touching the decorated type.
- **Composite** — combine implementations behind one protocol (e.g. remote-**with-fallback-to**-cache `FeedLoader`).
- **Adapter** — bridge a concrete/SDK type to the domain's protocol, keeping the SDK at the edge.
- **Facade** — hide a multi-step subsystem behind a simple interface for the composition root.

## Framework isolation
- The **domain is pure Swift** — no `import UIKit`/`SwiftUI`/`CoreData`/`Foundation`-networking in
  domain types. Frameworks are *replaceable details* behind protocols the domain owns; map DTO →
  domain at the boundary. *Smell:* `import UIKit` in a model; an `NSManagedObject` or a `Codable` DTO
  leaking into the domain/UI. (Pairs with the testing skill's mockability + concurrency's framework rules.)

## Protocol-oriented design (the Swift translation)
- Small protocols, value types, protocol composition, protocol extensions for shared default behavior —
  this is how SOLID lands idiomatically in Swift (Apple POP). Use a generic/`some`/`any` boundary
  deliberately. But heed the north star: **don't over-protocol.**

## Review checklist (what this skill flags)
Per changed/added type: single responsibility? depends on abstractions or concretes? a framework
leaking into the domain? protocol the right size (ISP) / honestly substitutable (LSP)? extended vs
modified (OCP)? wired at the composition root, not reached as a singleton? Output each finding as
`{ severity, file:line, principle, problem, fix }`. Severity: **Critical** (framework leak into domain,
concrete dependency that blocks testing a high-value path) · **Important** (God type, fat protocol,
over-injection, OCP switch-edit) · **Nit** (naming, minor seam). Be specific; don't invent abstraction
needs (flag *speculative* abstractions too).

## Boundaries (avoid duplication)
- **Pattern selection** (MVVM vs Clean vs VIPER vs TCA) → the architecture lens, not here.
- **Concurrency isolation / Sendable** → `ios-concurrency-expert`. **SwiftUI state ownership** →
  `ios-swiftui-expert`. **Test seams / doubles** → `ios-testing-expert`. This skill is the
  principle-level "is it decoupled and substitutable?" lens that sits beneath all of them.
