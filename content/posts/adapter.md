+++
date = '2025-10-13'
title = 'Adapter Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about integrating a third-party payment gateway into your system. Your application has a clean `PaymentProcessor` interface -- `Charge(amount Money) error`. The new gateway's SDK has a completely different API: `ExecuteTransaction(cents int64, currency string, metadata map[string]string) (*Result, error)`. Different method names, different parameter types, different return values. You can't change the SDK -- it's a third-party package. You don't want to change your interface -- twenty services already depend on it.

The **Adapter pattern** bridges this gap. You write a thin struct that implements your `PaymentProcessor` interface and internally translates calls to the gateway's SDK. Your application code sees the interface it expects. The SDK does the work it was built to do. Neither side changes. The adapter is the translator in the middle.

This is one of the most practically useful structural patterns, and Go's implicit interfaces make it especially clean. The adapter struct implements the target interface without declaring anything -- it just has the right methods. No inheritance, no abstract classes, no `implements` keyword. In this article, we'll build a payment gateway adapter, show how Go's composition model makes adapters lightweight, and explore the line between adapters and other structural patterns that look similar.

## What Is the Adapter Pattern?

> "Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces."
> -- Gang of Four

The intent: **make an existing type usable through a different interface without modifying either side.** The adapter wraps the incompatible type (the **adaptee**) and exposes the interface the client expects (the **target**). Calls flow through the adapter, which translates method names, parameter types, and return values as needed.

The key constraint: the adapter doesn't add behavior. It translates. If you're adding caching, logging, or error handling, that's a Decorator. If you're simplifying a complex subsystem, that's a Facade. The Adapter's sole job is making one interface look like another.

Without an adapter, integrating the payment gateway directly looks like this:

```go
func chargeCustomer(gateway *StripeSDK, amount Money) error {
    cents := int64(amount.Dollars * 100)
    result, err := gateway.ExecuteTransaction(cents, amount.Currency, nil)
    if err != nil {
        return err
    }
    if result.Status != "success" {
        return fmt.Errorf("charge failed: %s", result.Message)
    }
    return nil
}
```

Every function that charges a customer must understand Stripe's API: cents instead of dollars, string status codes, metadata maps. Change gateways (Stripe to Square, Square to Adyen) and every call site changes. The adapter centralizes this translation.

## Core Components

Four participants.

**Target** -- the interface your application expects. Client code depends on this.

```go
type PaymentProcessor interface {
    Charge(amount Money) error
    Refund(transactionID string, amount Money) error
}
```

**Adaptee** -- the existing type with an incompatible interface. You can't (or don't want to) change it.

```go
type StripeSDK struct { /* third-party */ }

func (s *StripeSDK) ExecuteTransaction(cents int64, currency string, meta map[string]string) (*StripeResult, error)
func (s *StripeSDK) ReverseTransaction(txID string, cents int64) (*StripeResult, error)
```

**Adapter** -- implements the Target interface, holds a reference to the Adaptee, and translates between them.

**Client** -- works with the Target interface. Doesn't know the adapter or adaptee exist.

The flow:

```
Client --> PaymentProcessor.Charge() --> StripeAdapter.Charge() --> StripeSDK.ExecuteTransaction()
```

The client sees `PaymentProcessor`. The SDK sees its own types. The adapter translates in between.

## Code Walkthrough: A Payment Gateway Adapter

We're building an adapter that wraps a third-party payment SDK behind our application's `PaymentProcessor` interface.

### The Target Interface

```go
type Money struct {
    Dollars  float64
    Currency string
}

type PaymentProcessor interface {
    Charge(amount Money) error
    Refund(transactionID string, amount Money) error
}
```

This is what our application code works with. Clean, domain-specific, stable.

### The Adaptee (Third-Party SDK)

```go
type StripeResult struct {
    Status  string
    TxID    string
    Message string
}

type StripeSDK struct {
    apiKey string
}

func NewStripeSDK(apiKey string) *StripeSDK {
    return &StripeSDK{apiKey: apiKey}
}

func (s *StripeSDK) ExecuteTransaction(cents int64, currency string, meta map[string]string) (*StripeResult, error) {
    fmt.Printf("  [stripe] charging %d %s (key=%s)\n", cents, currency, s.apiKey[:8]+"...")
    return &StripeResult{Status: "success", TxID: "tx_abc123"}, nil
}

func (s *StripeSDK) ReverseTransaction(txID string, cents int64) (*StripeResult, error) {
    fmt.Printf("  [stripe] refunding %d cents for %s\n", cents, txID)
    return &StripeResult{Status: "success", TxID: txID}, nil
}
```

Different method names (`ExecuteTransaction` vs `Charge`), different parameter types (cents vs dollars, separate currency string vs `Money` struct), different return types (`*StripeResult` vs `error`). Incompatible with our interface.

### The Adapter

```go
type StripeAdapter struct {
    sdk *StripeSDK
}

func NewStripeAdapter(apiKey string) *StripeAdapter {
    return &StripeAdapter{sdk: NewStripeSDK(apiKey)}
}

func (a *StripeAdapter) Charge(amount Money) error {
    cents := int64(amount.Dollars * 100)
    result, err := a.sdk.ExecuteTransaction(cents, amount.Currency, nil)
    if err != nil {
        return fmt.Errorf("stripe charge failed: %w", err)
    }
    if result.Status != "success" {
        return fmt.Errorf("stripe charge declined: %s", result.Message)
    }
    return nil
}

func (a *StripeAdapter) Refund(transactionID string, amount Money) error {
    cents := int64(amount.Dollars * 100)
    result, err := a.sdk.ReverseTransaction(transactionID, cents)
    if err != nil {
        return fmt.Errorf("stripe refund failed: %w", err)
    }
    if result.Status != "success" {
        return fmt.Errorf("stripe refund declined: %s", result.Message)
    }
    return nil
}
```

The adapter does three things: converts dollars to cents, maps method names, and translates the SDK's result struct into a Go error. That's it. No business logic, no caching, no retry logic -- pure translation.

### Using It

```go
func processOrder(processor PaymentProcessor, orderTotal Money) error {
    fmt.Printf("Charging $%.2f %s...\n", orderTotal.Dollars, orderTotal.Currency)
    if err := processor.Charge(orderTotal); err != nil {
        return fmt.Errorf("order failed: %w", err)
    }
    fmt.Println("Order processed successfully")
    return nil
}

func main() {
    adapter := NewStripeAdapter("sk_live_abc123xyz789")

    order := Money{Dollars: 49.99, Currency: "USD"}
    if err := processOrder(adapter, order); err != nil {
        fmt.Println("Error:", err)
    }
}
```

```
Charging $49.99 USD...
  [stripe] charging 4999 USD (key=sk_live_...)
Order processed successfully
```

`processOrder` doesn't know about Stripe. It works with `PaymentProcessor`. Tomorrow you could swap in a `SquareAdapter` or a `MockProcessor` for tests -- the function doesn't change.

### Adding a Second Adapter

When you switch providers, you write a new adapter:

```go
type SquareSDK struct{}

func (s *SquareSDK) CreatePayment(amountCents int, currencyCode string) error {
    fmt.Printf("  [square] payment of %d %s\n", amountCents, currencyCode)
    return nil
}

func (s *SquareSDK) CreateRefund(paymentID string, amountCents int) error {
    fmt.Printf("  [square] refund %d cents for %s\n", amountCents, paymentID)
    return nil
}

type SquareAdapter struct {
    sdk *SquareSDK
}

func (a *SquareAdapter) Charge(amount Money) error {
    cents := int(amount.Dollars * 100)
    return a.sdk.CreatePayment(cents, amount.Currency)
}

func (a *SquareAdapter) Refund(transactionID string, amount Money) error {
    cents := int(amount.Dollars * 100)
    return a.sdk.CreateRefund(transactionID, cents)
}
```

Same `PaymentProcessor` interface, different SDK underneath. The client code (`processOrder`) works identically with either adapter.

## Adapter vs Decorator vs Facade

These three patterns all involve wrapping, but they solve different problems.

**Adapter** changes the *interface*. The adaptee has one shape; the client expects another. The adapter translates. No new behavior is added.

**Decorator** changes the *behavior*. The wrapped object and the wrapper share the same interface. The decorator adds functionality (logging, caching, metrics) without changing the contract.

**Facade** simplifies *access*. It wraps a complex subsystem (multiple objects, multiple methods) behind a simpler API. It's not about interface translation -- it's about reducing complexity.

The test: if you're converting method signatures, it's an Adapter. If you're adding a feature to an existing implementation, it's a Decorator. If you're hiding complexity behind a simpler entry point, it's a Facade.

## When to Use It

The Adapter pattern fits when **you need to use an existing type through an interface it doesn't satisfy, and you can't or shouldn't modify either side**:

- **Third-party SDK integration** where the SDK's API doesn't match your application's interfaces. Payment gateways, cloud SDKs, analytics services.
- **Legacy system migration** where old components need to work with new interfaces during a gradual refactor. The adapter keeps both sides functional while you transition.
- **Testing with external dependencies** where you want your application to depend on an interface, and the real implementation (wrapped in an adapter) is swapped for a mock in tests.
- **Multi-provider support** where multiple vendors (Stripe/Square, AWS/GCP, Datadog/NewRelic) must satisfy a common interface. Each gets its own adapter.

Skip it when:

- **You control both sides.** If you can modify the type to directly implement the interface, do that. An adapter between two things you own is usually a sign that one of them should be redesigned.
- **The mismatch is trivial.** If the only difference is a method name and you call it once, a direct call with inline translation is simpler than a full adapter struct.
- **You're adding behavior, not translating.** If the "adapter" also adds caching, retries, or logging, it's a Decorator (or multiple concerns mixed together). Keep the adapter thin and compose other patterns around it.

## Pros and Cons

**Pros:**

- **Reuses existing code without modification** -- the adaptee doesn't change. Critical when you can't modify third-party code or when changing legacy code is risky.
- **Insulates application code from external changes** -- when the SDK updates its API, only the adapter changes. All consumers of `PaymentProcessor` remain untouched.
- **Enables clean testing** -- your application depends on the target interface, which can be mocked. The adapter itself is a thin layer that barely needs testing beyond verifying the translation.
- **Supports multi-vendor architectures** -- one interface, multiple adapters, runtime selection based on config. Swap providers without touching business logic.

**Cons:**

- **Extra indirection** -- every call passes through the adapter before reaching the real implementation. Usually negligible, but visible in profiling for very hot paths.
- **Can mask design debt** -- adapting around a bad interface instead of fixing it delays proper refactoring. Adapters should be bridges, not permanent fixtures.
- **Adapter proliferation** -- ten vendors with three methods each means ten adapter structs. The code is simple but voluminous.

## Best Practices

- **Keep adapters thin -- translate, don't transform.** The adapter converts method names and parameter types. If it's doing data validation, error categorization, or retry logic, those concerns should be separate (middleware, decorators, or service layers).
- **Return wrapped errors with context.** `fmt.Errorf("stripe charge failed: %w", err)` gives callers the ability to unwrap to the original error while adding adapter-level context. Don't discard the underlying error.
- **One adapter per adaptee.** Don't build a single "super adapter" that wraps Stripe, Square, and Adyen in one struct with conditionals. Each vendor gets its own adapter implementing the shared interface.
- **Put the adapter in its own package in larger codebases.** `internal/payment/stripe/adapter.go` keeps the third-party dependency contained. Application code imports only the `payment` package and its interface.

## Common Mistakes

**Mixing translation with business logic.** An adapter that validates payment amounts, applies discounts, or enforces daily limits has taken on too many responsibilities. The adapter should translate the call faithfully. Business rules belong in the service layer above it.

**Exposing the adaptee's types through the adapter.** If `Charge()` returns `*StripeResult` instead of `error`, the client is now coupled to Stripe's types through the adapter. The whole point is hiding the adaptee -- return only types defined by the target interface.

**Building an adapter when a function would suffice.** If the "adaptation" is a single method with a one-line body, a free function or closure might be clearer than a struct:

```go
func adaptLegacyLogger(legacy *OldLogger) Logger {
    return LoggerFunc(func(msg string) { legacy.WriteLog(msg) })
}
```

This is lighter than a full adapter struct when the translation is trivial and the adaptee has one method.

**Not testing the translation edge cases.** The adapter converts dollars to cents (`int64(amount.Dollars * 100)`). What happens with $0.01? With $999,999.99? With negative amounts? Floating-point precision means `49.99 * 100` might not be exactly `4999`. The adapter is thin, but its translation logic still needs tests.

## Final Thoughts

The Adapter pattern takes two incompatible interfaces and connects them through a thin translation layer. Neither side changes. The adapter converts method names, parameter types, and return values so the client sees what it expects and the adaptee receives what it understands.

Go makes this particularly elegant. Implicit interface satisfaction means the adapter struct doesn't need to declare that it implements the target -- it just needs the right methods. Composition over inheritance means the adapter holds a reference to the adaptee rather than inheriting from it. The result is lightweight, explicit, and easy to replace.

The discipline is keeping it thin. The moment an adapter starts doing more than translating -- validating, caching, retrying, logging -- it's become something else. Keep the adapter focused on its one job: making the incompatible compatible.

> **Translate the interface, don't transform the behavior. An adapter is a bridge, not a building.**
