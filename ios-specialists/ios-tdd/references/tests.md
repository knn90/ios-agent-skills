# Good vs bad tests (Swift)

> Reference for [`ios-tdd`](../SKILL.md). Swift-native rewrite of mattpocock/skills `tdd/tests.md` (MIT).
> The *one rule*: test observable behavior through the **public interface**, never internal structure.

## Good — integration-style, behavior through the public API
Exercises real code paths the way a caller would; survives internal refactors; the name says **what**, not how.

```swift
@Test("user can checkout with a valid cart")
func checkoutSucceeds() async throws {
    var cart = Cart()
    cart.add(product)
    let result = try await checkout(cart, using: paymentMethod)
    #expect(result.status == .confirmed)        // outcome the caller cares about
}
```
Characteristics: behavior callers care about · public API only · one logical assertion · survives refactors · describes WHAT.

## Bad — coupled to implementation
Breaks when you refactor even though behavior is unchanged — the signal it was testing *how*, not *what*.

```swift
// BAD: asserts an internal interaction, not an outcome
@Test("checkout calls paymentService.process")
func checkoutCallsProcess() async throws {
    let payment = PaymentServiceSpy()
    _ = try await checkout(cart, using: payment)
    #expect(payment.processCallCount == 1)      // breaks on any refactor; proves nothing about the result
}
```
Red flags: mocking internal collaborators · testing private methods · asserting call counts/order · name describes HOW · verifying through a back channel instead of the interface.

## Verify through the interface, not a back channel
```swift
// BAD: reaches past the interface into the store
@Test func createUserWritesRow() async throws {
    _ = try await createUser(name: "Alice")
    let rows = try await db.query("SELECT * FROM users WHERE name = ?", ["Alice"])
    #expect(!rows.isEmpty)                       // couples the test to the schema
}

// GOOD: round-trips through the public API
@Test func createdUserIsRetrievable() async throws {
    let created = try await createUser(name: "Alice")
    let fetched = try await user(for: created.id)
    #expect(fetched.name == "Alice")
}
```

For SwiftUI, the "interface" of a screen is its **view-model's observable state** — test that, not the `View`
(see `ios-testing-expert §12`). For `#expect`/`#require`, parameterized cases, and traits, see `ios-testing-expert §3–5`.
