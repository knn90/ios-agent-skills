# When (and where) to mock — Swift

> Reference for [`ios-tdd`](../SKILL.md). Swift-native rewrite of mattpocock/skills `tdd/mocking.md` (MIT).
> This is the *decision* (when/where) + design-for-substitutability. The **double taxonomy** (dummy/fake/
> stub/spy/mock), fixtures, and `#if DEBUG` placement live in `ios-testing-expert §8` — don't duplicate them here.

## Mock only at system boundaries
Substitute the things you don't control or can't make deterministic:
- **Network** — `URLSession` / your API client (never hit the live network in unit tests)
- **Persistence** — Core Data / SwiftData / a remote DB (prefer an **in-memory store** over a mock when feasible)
- **Time & randomness** — `Date()`, `Clock`, `UUID()`, RNG → inject a fixed clock/value (deterministic tests only)
- **File system** — sometimes; prefer a temp directory

**Don't mock what you own** — your own types, internal collaborators, value logic. Mocking internals produces
tests coupled to structure that pass while production breaks (see [tests.md](tests.md)). Prefer the real thing:
real > fake (in-memory) > stub > mock.

## Design for substitutability

**1. Inject dependencies — don't construct them inside.**
```swift
// Easy to substitute: the dependency comes in
func processPayment(_ order: Order, using client: PaymentClient) async throws -> Receipt {
    try await client.charge(order.total)
}

// Hard: builds its own concrete dependency — nothing to swap in a test
func processPayment(_ order: Order) async throws -> Receipt {
    let client = StripeClient(apiKey: Secrets.stripeKey)   // unmockable
    return try await client.charge(order.total)
}
```
Inject via the project's `di` (initializer/default-param injection, or a protocol + test double). A change
that's *hard to test* is a design signal — fix the seam, don't skip the test.

**2. Prefer SDK-style protocols over one generic `perform()`.**
One method per operation → each is independently stubbable with a single fixed return; no conditional logic
inside the double, and a test's dependencies are visible from the protocol it stubs.
```swift
// GOOD: one method per operation
protocol OrdersAPI {
    func user(id: User.ID) async throws -> User
    func orders(for userID: User.ID) async throws -> [Order]
    func createOrder(_ draft: OrderDraft) async throws -> Order
}

// BAD: a single generic entry point forces if/else branching inside every stub
protocol GenericAPI {
    func request(_ endpoint: Endpoint, _ options: RequestOptions) async throws -> Data
}
```
Benefits: each stub returns one concrete shape · no branching in test setup · the protocol documents which
operations a unit actually uses · type-safe per operation.
