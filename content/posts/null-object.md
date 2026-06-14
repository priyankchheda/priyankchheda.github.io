+++
date = '2026-06-14'
title = 'Null Object Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a service that optionally logs, optionally collects metrics, and optionally sends notifications. In production, all three are wired up. In development, maybe just logging. In tests, none of them. The code that uses these dependencies shouldn't care which configuration it's running in. It just calls `logger.Log()`, `metrics.Record()`, `notifier.Send()` -- and the right thing happens.

Without the Null Object pattern, "the right thing" when a dependency is absent means checking for `nil` before every single call. `if logger != nil { logger.Log(...) }`. Repeated before every log statement, every metric recording, every notification. The business logic drowns in guard clauses. Forget one check and you get a nil pointer panic in production. And the cognitive load of "is this thing optional?" leaks into every function that touches it.

The **Null Object** pattern eliminates this by providing a real implementation that does nothing. Instead of a nil logger, you have a `NullLogger` that implements the `Logger` interface with empty methods. The caller doesn't check -- it just calls. The null object absorbs the call silently. No panics, no guards, no conditionals.

Go's interface system makes this pattern trivially natural. In this article, we'll build a service with optional logging, metrics, and notifications -- each backed by either a real implementation or a null object, chosen at configuration time.

## What Is the Null Object Pattern?

> "Provide an object as a surrogate for the lack of an object of a given type. The Null Object provides intelligent do-nothing behavior, hiding the details from its collaborators."
> -- Bobby Woolf, Pattern Languages of Program Design 3

The intent: **replace nil with an object that implements the expected interface but performs no operation.** Callers treat it exactly like a real object. They never check for absence. The null object absorbs calls and returns safe defaults -- zero values, empty strings, nil errors.

The key insight: **absence should behave, not be checked.** Instead of the caller deciding "is this present?" at every call site, the decision is made once at construction time. If the dependency is absent, wire in a null object. If it's present, wire in the real one. Either way, the interface is satisfied and calls are safe.

Without the pattern, optional dependencies pollute every use site:

```go
func (s *Service) Process(order Order) error {
    if s.logger != nil {
        s.logger.Log("processing order " + order.ID)
    }
    s.doWork(order)
    if s.metrics != nil {
        s.metrics.Record("orders_processed", 1)
    }
    if s.notifier != nil {
        s.notifier.Send(order.CustomerEmail, "Order received")
    }
    return nil
}
```

Three `nil` checks for three optional dependencies. Every method that uses these dependencies repeats the pattern. The business logic ("process order, do work, record, notify") is obscured by guard code.

With Null Objects:

```go
func (s *Service) Process(order Order) error {
    s.logger.Log("processing order " + order.ID)
    result := s.doWork(order)
    s.metrics.Record("orders_processed", 1)
    s.notifier.Send(order.CustomerEmail, "Order received")
    return result
}
```

Clean. No conditionals. If any dependency is absent, its null implementation absorbs the call. The decision about what's present was made at construction time, not at every call site.

## Core Components

Three participants -- the simplest structure imaginable.

**Interface** -- the contract that both real and null implementations satisfy.

```go
type Logger interface {
    Log(msg string)
}
```

**Real Object** -- the actual implementation that does meaningful work.

```go
type FileLogger struct { /* ... */ }
func (l *FileLogger) Log(msg string) { /* writes to file */ }
```

**Null Object** -- implements the same interface with do-nothing behavior.

```go
type NullLogger struct{}
func (l *NullLogger) Log(msg string) {}
```

The flow:

```
Service.Process()
  --> logger.Log()  // might be FileLogger (writes) or NullLogger (no-op)
  --> metrics.Record()  // might be Prometheus (records) or NullMetrics (no-op)
  --> notifier.Send()  // might be EmailNotifier (sends) or NullNotifier (no-op)
```

The service doesn't know which implementation it has. It just calls the interface. The decision was made at construction time.

## Code Walkthrough: A Configurable Service

We're building an order processing service where logging, metrics, and notifications are all optional -- present in production, absent in tests, partially configured in development.

### The Interfaces

```go
type Logger interface {
    Log(msg string)
}

type Metrics interface {
    Record(name string, value float64)
}

type Notifier interface {
    Send(to string, message string) error
}
```

Three small interfaces. Each has one responsibility.

### The Real Implementations

```go
type ConsoleLogger struct{}

func (l *ConsoleLogger) Log(msg string) {
    fmt.Printf("[LOG] %s\n", msg)
}

type PrometheusMetrics struct{}

func (m *PrometheusMetrics) Record(name string, value float64) {
    fmt.Printf("[METRIC] %s = %.1f\n", name, value)
}

type EmailNotifier struct {
    smtpHost string
}

func (n *EmailNotifier) Send(to string, message string) error {
    fmt.Printf("[EMAIL] to=%s: %s\n", to, message)
    return nil
}
```

Each does real work: logs to console, records metrics, sends emails.

### The Null Objects

```go
type NullLogger struct{}

func (l *NullLogger) Log(msg string) {}

type NullMetrics struct{}

func (m *NullMetrics) Record(name string, value float64) {}

type NullNotifier struct{}

func (n *NullNotifier) Send(to string, message string) error {
    return nil
}
```

Each implements the interface and does absolutely nothing. `NullNotifier.Send` returns `nil` (no error) -- it succeeds by doing nothing.

### The Service

```go
type OrderService struct {
    logger   Logger
    metrics  Metrics
    notifier Notifier
}

func NewOrderService(logger Logger, metrics Metrics, notifier Notifier) *OrderService {
    return &OrderService{
        logger:   logger,
        metrics:  metrics,
        notifier: notifier,
    }
}

func (s *OrderService) ProcessOrder(orderID string, email string) error {
    s.logger.Log("processing order " + orderID)

    s.logger.Log("validating payment")
    // ... payment logic ...

    s.logger.Log("reserving inventory")
    // ... inventory logic ...

    s.metrics.Record("orders_processed", 1)

    if err := s.notifier.Send(email, "Your order "+orderID+" is confirmed"); err != nil {
        s.logger.Log("notification failed: " + err.Error())
        return err
    }

    s.logger.Log("order " + orderID + " complete")
    return nil
}
```

No nil checks anywhere. The service calls every dependency confidently. If a dependency is a null object, the call is a no-op. If it's real, the call does real work. The logic reads cleanly either way.

### Configuration: Wiring It Up

**Production:**

```go
func main() {
    fmt.Println("=== Production ===")
    svc := NewOrderService(
        &ConsoleLogger{},
        &PrometheusMetrics{},
        &EmailNotifier{smtpHost: "smtp.prod.internal"},
    )
    svc.ProcessOrder("ORD-001", "alice@example.com")
}
```

```
=== Production ===
[LOG] processing order ORD-001
[LOG] validating payment
[LOG] reserving inventory
[METRIC] orders_processed = 1.0
[EMAIL] to=alice@example.com: Your order ORD-001 is confirmed
[LOG] order ORD-001 complete
```

**Testing (silent):**

```go
func main() {
    fmt.Println("=== Test ===")
    svc := NewOrderService(
        &NullLogger{},
        &NullMetrics{},
        &NullNotifier{},
    )
    svc.ProcessOrder("ORD-002", "test@example.com")
    fmt.Println("(no output -- all dependencies are null objects)")
}
```

```
=== Test ===
(no output -- all dependencies are null objects)
```

**Development (logging only):**

```go
func main() {
    fmt.Println("=== Development ===")
    svc := NewOrderService(
        &ConsoleLogger{},
        &NullMetrics{},
        &NullNotifier{},
    )
    svc.ProcessOrder("ORD-003", "dev@example.com")
}
```

```
=== Development ===
[LOG] processing order ORD-003
[LOG] validating payment
[LOG] reserving inventory
[LOG] order ORD-003 complete
```

Same service code. Three different configurations. The null objects absorb what's not needed. The service never knows the difference.

## Go's `io.Discard`: A Null Object in the Standard Library

Go's own standard library uses this pattern. `io.Discard` is a `Writer` that discards all data:

```go
var Discard io.Writer = devNull(0)

type devNull int

func (devNull) Write(p []byte) (int, error) {
    return len(p), nil
}
```

It satisfies `io.Writer`, accepts all writes, and throws everything away. When you pass `io.Discard` to a logger or encoder, it runs normally -- just without producing output. That's the Null Object pattern, baked into the standard library.

Similarly, `log.New(io.Discard, "", 0)` creates a logger that does nothing. No nil checks needed -- the logger works, it just discards everything.

## When to Use It

The Null Object pattern fits when **a dependency is optional and callers shouldn't need to know whether it's present**:

- **Optional logging** where some environments need logs and others don't, but the business logic shouldn't contain `if logger != nil` throughout.
- **Optional metrics/telemetry** that's wired up in production but absent in tests and local development.
- **Optional notifications** (email, Slack, webhooks) that are active in production but silent during testing.
- **Default behavior for missing configuration** where a feature flag is off, the corresponding service is a null object that gracefully does nothing.
- **Test isolation** where real dependencies (databases, APIs, queues) are replaced with null objects to test business logic without side effects.

Skip it when:

- **Absence needs to be visible.** If a missing logger should cause an error at startup (misconfiguration), using a null object hides the problem. Fail fast instead.
- **The caller genuinely needs to know.** If different logic runs based on whether a dependency is present (not just "call or skip"), a nil check or optional type is more honest than a silent null object.
- **The "do nothing" behavior is dangerous.** If a `NullPaymentProcessor` silently "succeeds" at charging cards, orders go through without payment. Some operations must either work or fail loudly.

## Pros and Cons

**Pros:**

- **Eliminates nil checks** -- no more `if x != nil` scattered throughout business logic. One decision at construction time replaces N decisions at call time.
- **Prevents nil pointer panics** -- the null object is a real object. Calling methods on it is always safe, unlike calling methods on nil interfaces.
- **Simplifies testing** -- pass null objects for dependencies you don't care about in a given test. No mocking framework needed for "I just don't want this to blow up."
- **Clean business logic** -- the service code reads as a straightforward sequence of actions without conditional clutter.

**Cons:**

- **Silent failures** -- if a real implementation was expected but a null object was wired in by mistake, nothing fails. The system silently does nothing where it should be doing work. Bugs become invisible.
- **Debugging obscurity** -- when investigating "why didn't the notification send?", the answer might be "because someone configured a NullNotifier" -- which isn't obvious from the service code.
- **Proliferation of no-op types** -- every interface needs a corresponding null implementation. Ten interfaces means ten null structs, each with empty methods. It's minor boilerplate but it adds up.

## Best Practices

- **Provide null object constructors alongside real ones.** If your package exports `NewConsoleLogger()`, also export `NewNullLogger()` or `NopLogger()`. Make the null object a first-class citizen, not something callers have to build themselves.
- **Document that a dependency accepts null objects.** If `NewOrderService` accepts any `Logger` including a null one, document it. Otherwise developers might assume nil is safe (it's not) or that a real logger is required (it's not with the null object).
- **Return meaningful zero values from null object methods.** If the interface method returns a value (`Get(key string) (string, error)`), the null object should return sensible defaults: empty string and nil error, not panic or garbage values.
- **Name null objects clearly.** `NullLogger`, `NopMetrics`, `DiscardNotifier` -- the name should make intent obvious. Avoid generic names like `DefaultLogger` which might suggest it actually logs somewhere.

## Common Mistakes

**Using a null object when failure should be loud.** A `NullPaymentGateway` that returns `nil` error from `Charge()` means orders process without payment. Some dependencies are *required* for correctness, not optional for convenience. If the system can't function without it, don't offer a null version -- require the real one at construction and fail if it's missing.

**Confusing null object with mock object.** A null object does nothing and asserts nothing. A mock records calls and lets tests verify behavior. They look similar but serve different purposes. Use `NullLogger` when you don't care about logging in a test. Use a mock logger when you want to assert that specific messages were logged.

**Creating null objects with side effects.** A "null" notifier that logs "notification suppressed" to stdout isn't truly null -- it's a different real implementation (a logging notifier). Null means *nothing happens*. No output, no side effects, no state changes. If you want to log suppressions, that's a separate "suppressed" implementation, not a null one.

**Not considering the nil interface trap in Go.** A common bug:

```go
var logger *ConsoleLogger = nil
var svc = NewOrderService(logger, ...) // logger is NOT nil interface!
```

A nil pointer stored in an interface variable is not a nil interface. The service will try to call methods on a nil pointer and panic. The fix: always construct an actual null object (`&NullLogger{}`), never pass a typed nil.

## Final Thoughts

The Null Object pattern replaces nil with a real object that implements the expected interface but performs no operation. The caller never checks for absence -- it just calls the interface, and the null object absorbs calls that have nowhere else to go. The decision about presence or absence is made once at construction time, not repeated at every call site.

Go's interfaces make this pattern trivially simple to implement. A struct with empty methods that satisfies an interface -- that's the entire implementation. No base classes, no special syntax. And the standard library validates the approach: `io.Discard` is the canonical Go null object.

The discipline is knowing when "do nothing" is safe and when it's dangerous. Optional logging, metrics, and notifications are natural fits. Payment processing, data persistence, and authentication are not. The pattern eliminates nil checks for things that are genuinely optional -- not for things that must work.

> **Make absence behave. Don't make callers handle it.**
