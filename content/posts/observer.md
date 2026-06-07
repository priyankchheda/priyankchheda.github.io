+++
date = '2026-05-06'
title = 'Observer Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about an order service. When an order is placed, several things need to happen: an email confirmation goes out, the analytics system records the event, the inventory count decreases, and the audit log gets a new entry. The naive approach calls all of these directly from the `CreateOrder` function. Works fine with two side effects. Becomes a problem at five. Becomes unmaintainable at ten.

Every new reaction means modifying the order service. The order service now knows about email, analytics, inventory, auditing -- concerns that have nothing to do with creating an order. Testing it means mocking half the system. Adding a new reaction (Slack notification, webhook, cache warmup) means opening a file that shouldn't need to change.

The **Observer pattern** flips this around. The order service announces "an order was created" and doesn't care who's listening. Interested parties subscribe. They react independently. The order service stays focused on orders. New reactions are added without touching existing code.

In this article, we'll build an event notification system for a backend service, cover both synchronous and channel-based approaches, handle concurrency safely, and explore the trade-offs between push and pull notification models. Go's interfaces make the contract clean; its channels make the async version feel native.

## What Is the Observer Pattern?

> "Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically."
> -- Gang of Four

The core idea: **one object publishes state changes, many objects react, and neither side knows the details of the other.** The publisher (called the **Subject**) maintains a list of subscribers (called **Observers**). When something changes, the subject iterates through its observers and notifies each one. Observers decide independently how to react.

Without this separation, event handling looks like this:

```go
func (s *OrderService) CreateOrder(order Order) error {
    if err := s.repo.Save(order); err != nil {
        return err
    }
    s.emailService.SendConfirmation(order)
    s.analytics.Track("order_created", order.ID)
    s.audit.Log("order_created", order)
    s.inventory.Decrement(order.Items)
    return nil
}
```

`OrderService` knows about four other services. Adding a Slack notification means modifying this function. Removing the analytics call means modifying this function. Every reaction is hardcoded, and the function grows with every new requirement.

With the Observer pattern, this becomes:

```go
func (s *OrderService) CreateOrder(order Order) error {
    if err := s.repo.Save(order); err != nil {
        return err
    }
    s.Notify(OrderCreatedEvent{Order: order})
    return nil
}
```

One line for notification. The order service doesn't know who's listening or what they do. Observers are added and removed externally, at runtime, without touching the order service.

## Core Components

Two interfaces and their implementations.

**Observer** -- any object that wants to react to state changes. Defines a single method that gets called on notification.

```go
type Observer interface {
    OnEvent(event Event)
}
```

**Subject** -- the object that owns state and manages observers. Provides attach, detach, and notify operations.

```go
type Subject interface {
    Subscribe(o Observer)
    Unsubscribe(o Observer)
    Notify(event Event)
}
```

The flow:

```
State changes --> Subject.Notify() --> Observer.OnEvent() (for each)
```

The subject doesn't call specific methods on specific services. It calls one interface method on every registered observer. This is the decoupling.

## Code Walkthrough: An Event Notification System

We're building the event system for a backend service. When domain events occur (order created, payment received, user registered), observers react independently -- sending emails, recording metrics, writing audit logs.

### The Event Type

```go
type Event struct {
    Name    string
    Payload map[string]string
}
```

A lightweight event with a name and arbitrary payload. In production you might use typed events, but a generic event keeps the walkthrough focused on the pattern mechanics.

### The Observer Interface

```go
type Observer interface {
    OnEvent(event Event)
}
```

One method. Any struct that implements `OnEvent` is an observer. Go's implicit interface satisfaction means observers don't need to declare they implement anything -- they just do.

### The Subject (Event Emitter)

The subject manages a list of observers and notifies them when events occur:

```go
type EventEmitter struct {
    mu        sync.RWMutex
    observers []Observer
}

func NewEventEmitter() *EventEmitter {
    return &EventEmitter{}
}

func (e *EventEmitter) Subscribe(o Observer) {
    e.mu.Lock()
    defer e.mu.Unlock()
    e.observers = append(e.observers, o)
}

func (e *EventEmitter) Unsubscribe(o Observer) {
    e.mu.Lock()
    defer e.mu.Unlock()
    for i, obs := range e.observers {
        if obs == o {
            e.observers = append(e.observers[:i], e.observers[i+1:]...)
            return
        }
    }
}

func (e *EventEmitter) Notify(event Event) {
    e.mu.RLock()
    observers := make([]Observer, len(e.observers))
    copy(observers, e.observers)
    e.mu.RUnlock()

    for _, o := range observers {
        o.OnEvent(event)
    }
}
```

A few things worth noting. The mutex protects the observer slice from concurrent modification -- subscribe and unsubscribe might happen from different goroutines than notification. `Notify` copies the slice before iterating, so an observer that unsubscribes during notification doesn't corrupt the loop. The lock is released *before* calling observers, preventing deadlocks if an observer calls back into the subject.

### Concrete Observers

An email notifier:

```go
type EmailNotifier struct {
    recipient string
}

func (n *EmailNotifier) OnEvent(event Event) {
    if event.Name == "order_created" {
        fmt.Printf("[email] Sending confirmation to %s for order %s\n",
            n.recipient, event.Payload["orderID"])
    }
}
```

An audit logger:

```go
type AuditLogger struct{}

func (a *AuditLogger) OnEvent(event Event) {
    fmt.Printf("[audit] %s: %v\n", event.Name, event.Payload)
}
```

A metrics recorder:

```go
type MetricsRecorder struct{}

func (m *MetricsRecorder) OnEvent(event Event) {
    fmt.Printf("[metrics] recorded event: %s\n", event.Name)
}
```

Each observer handles only what it cares about. The `EmailNotifier` only reacts to `order_created`. The `AuditLogger` logs everything. The `MetricsRecorder` records everything. None of them know about the order service or each other.

### Putting It Together

```go
func main() {
    emitter := NewEventEmitter()

    email := &EmailNotifier{recipient: "customer@example.com"}
    audit := &AuditLogger{}
    metrics := &MetricsRecorder{}

    emitter.Subscribe(email)
    emitter.Subscribe(audit)
    emitter.Subscribe(metrics)

    emitter.Notify(Event{
        Name:    "order_created",
        Payload: map[string]string{"orderID": "ORD-42"},
    })

    fmt.Println()

    emitter.Unsubscribe(metrics)
    emitter.Notify(Event{
        Name:    "payment_received",
        Payload: map[string]string{"orderID": "ORD-42", "amount": "99.99"},
    })
}
```

```
[email] Sending confirmation to customer@example.com for order ORD-42
[audit] order_created: map[orderID:ORD-42]
[metrics] recorded event: order_created

[audit] payment_received: map[amount:99.99 orderID:ORD-42]
```

The first event triggers all three observers. Then metrics is unsubscribed. The second event reaches only email (which ignores non-order events) and audit. The emitter didn't change. The order service didn't change. Only the subscription list changed.

## Channel-Based Observer: The Go-Native Approach

The interface-based approach above is synchronous -- each observer runs in the caller's goroutine. For systems where observers might be slow (HTTP calls, database writes, external APIs), a channel-based approach decouples notification from processing.

```go
type ChannelEmitter struct {
    mu          sync.RWMutex
    subscribers []chan Event
}

func NewChannelEmitter() *ChannelEmitter {
    return &ChannelEmitter{}
}

func (e *ChannelEmitter) Subscribe() <-chan Event {
    e.mu.Lock()
    defer e.mu.Unlock()
    ch := make(chan Event, 16)
    e.subscribers = append(e.subscribers, ch)
    return ch
}

func (e *ChannelEmitter) Notify(event Event) {
    e.mu.RLock()
    defer e.mu.RUnlock()
    for _, ch := range e.subscribers {
        select {
        case ch <- event:
        default:
            // subscriber is full, skip to avoid blocking
        }
    }
}

func (e *ChannelEmitter) Close() {
    e.mu.Lock()
    defer e.mu.Unlock()
    for _, ch := range e.subscribers {
        close(ch)
    }
    e.subscribers = nil
}
```

Each subscriber gets a buffered channel. `Notify` uses a non-blocking send -- if a subscriber's channel is full (it's too slow), the event is dropped rather than blocking the publisher. This is a deliberate trade-off: the publisher never slows down, but slow observers can miss events.

Observers run as goroutines consuming from their channels:

```go
func main() {
    emitter := NewChannelEmitter()

    auditCh := emitter.Subscribe()
    metricsCh := emitter.Subscribe()

    go func() {
        for event := range auditCh {
            fmt.Printf("[audit] %s: %v\n", event.Name, event.Payload)
        }
    }()

    go func() {
        for event := range metricsCh {
            fmt.Printf("[metrics] %s\n", event.Name)
        }
    }()

    emitter.Notify(Event{Name: "order_created", Payload: map[string]string{"orderID": "ORD-1"}})
    emitter.Notify(Event{Name: "payment_received", Payload: map[string]string{"orderID": "ORD-1"}})

    time.Sleep(10 * time.Millisecond)
    emitter.Close()
}
```

The channel approach is more idiomatic for concurrent Go systems. Each observer processes events at its own pace. There's no shared callback execution -- each goroutine owns its channel and its processing. The `Close()` method signals all subscribers to stop, preventing goroutine leaks.

The choice between interface-based and channel-based comes down to synchronous vs asynchronous needs. If observers must complete before the publisher continues (transactional consistency), use interfaces. If observers can process independently and lag is acceptable, use channels.

## Push vs Pull Models

The walkthrough above uses the **push model** -- the subject sends the full event to observers. There's an alternative.

In the **pull model**, the subject just says "something changed." Observers query back for the data they need:

```go
type Observer interface {
    OnChange()
}

type OrderService struct {
    observers []Observer
    lastOrder Order
}

func (s *OrderService) LastOrder() Order {
    return s.lastOrder
}
```

An observer pulls what it needs:

```go
func (a *AuditLogger) OnChange() {
    order := a.service.LastOrder()
    fmt.Printf("[audit] order: %s\n", order.ID)
}
```

Push is simpler and more common. Pull gives observers more flexibility -- they only fetch what they need, and adding new observer types doesn't require changing the event payload. The trade-off is that pull creates a dependency from observer back to subject, while push keeps observers fully independent.

For most Go systems, push with a generic `Event` struct is the pragmatic choice. Pull makes sense when the subject's state is large and different observers need different slices of it.

## When to Use It

The Observer pattern fits when **multiple objects need to react to state changes in another object, and the set of reactors changes over time**:

- **Domain event systems** where order creation, user registration, or payment completion triggers multiple independent side effects that grow over time.
- **Plugin architectures** where third-party code registers handlers for events without the core system knowing about them.
- **Configuration reloading** where multiple services need to update when a config value changes, and the set of services varies by deployment.
- **Monitoring and observability** where metrics, logging, and alerting attach to business events without polluting business logic.

Skip it when:

- **There's only one reactor.** A direct function call is simpler and more traceable than an observer subscription with a single subscriber.
- **Execution order matters.** Observers run independently -- if A must happen before B, Observer is the wrong tool. Use a pipeline or middleware chain instead.
- **You need request/response semantics.** Observer is fire-and-forget. If the publisher needs a result from the reaction, it's not observation -- it's a function call.

## Pros and Cons

**Pros:**

- **Open/Closed compliance** -- new reactions are added by subscribing new observers, not by modifying the subject. The order service never changes when you add Slack notifications.
- **Runtime flexibility** -- observers subscribe and unsubscribe dynamically. Temporary debugging observers, feature-flagged integrations, and test hooks all become trivial.
- **Testability** -- test the subject by subscribing a mock observer that records events. Test an observer by feeding it synthetic events. Each side tests independently.
- **Natural concurrency model in Go** -- channels turn the pattern into async fan-out with built-in backpressure, goroutine-per-observer isolation, and graceful shutdown via `close()`.

**Cons:**

- **Hidden control flow** -- when debugging "what happens when an order is created?", you can't just read the `CreateOrder` function. You have to find all subscribers. This makes the system harder to reason about for newcomers.
- **Memory and goroutine leaks** -- forgotten unsubscribes leave observers in memory. With channels, unclosed channels leak goroutines. Both require disciplined cleanup.
- **Notification storms** -- a subject that changes frequently with many observers generates O(changes * observers) callbacks. In hot paths, this matters.

## Best Practices

- **Copy the observer slice before notifying.** Iterating over the live slice while observers might unsubscribe during callbacks leads to skipped observers or panics. A cheap `copy()` before the loop prevents this.
- **Release the lock before calling observers.** Holding a mutex while calling `OnEvent()` invites deadlocks if an observer calls back into the subject (subscribe, unsubscribe, or trigger another notification). Lock, copy, unlock, then iterate.
- **Use buffered channels with non-blocking sends for async observers.** This prevents a slow subscriber from blocking the publisher. Accept that slow observers may miss events -- design for it with retry or persistence if needed.
- **Provide a `Close()` or shutdown mechanism.** For channel-based observers, close the channels to signal goroutines to exit. For interface-based observers, clear the subscriber list. Without explicit cleanup, long-running systems accumulate leaked resources.
- **Notify only on meaningful changes.** `if s.state != newState` before notifying prevents redundant callbacks that waste observer processing time and make debugging noisy.

## Common Mistakes

**Modifying the observer list during notification.** An observer's `OnEvent` calls `Unsubscribe(self)`, which mutates the slice while the subject is iterating over it. Result: skipped observers or index-out-of-range panics. The fix is copying the slice before iteration (as shown in the walkthrough).

**Making observers too coupled to event internals.** An `OnEvent(order *Order, user *User, config *Config)` signature couples every observer to three domain types. If the signature changes, every observer breaks. Keep event payloads minimal and generic -- a name plus a map, or a typed event struct that the observer can inspect or ignore.

**Forgetting that synchronous observers block the publisher.** If `OnEvent` makes an HTTP call that takes 2 seconds, and you have 5 observers, `Notify()` takes 10 seconds. The publisher is blocked the entire time. Either move to async notification (channels/goroutines) or enforce that observers are fast and non-blocking.

**Using Observer when Mediator is the right fit.** If the "observers" need to coordinate with each other -- A must happen before B, and if B fails then A should roll back -- that's orchestration, not observation. Observer is for independent reactions. Coordinated reactions need a Mediator or a pipeline.

## Observer vs Pub-Sub vs Mediator

These three are often confused because they all deal with "something happened, now react."

**Observer** is direct: the subject knows it has observers (via a slice) and calls them. It's in-process, synchronous by default, and the subject manages the subscription lifecycle.

**Pub-Sub** adds a broker between publisher and subscriber. The publisher doesn't know subscribers exist. The broker handles routing, filtering, and delivery. It's the right choice for distributed systems (Kafka, RabbitMQ, Redis Pub/Sub) and for in-process systems where complete decoupling is worth the extra layer.

**Mediator** coordinates interactions between peers. It doesn't just notify -- it decides who reacts and in what order, with branching logic. "When payment succeeds, ship the order; when payment fails, release inventory" is Mediator territory, not Observer.

The heuristic: if reactions are independent and fire-and-forget, use Observer. If reactions are distributed across services, use Pub-Sub. If reactions are coordinated with sequencing and branching, use Mediator.

## Final Thoughts

The Observer pattern takes a growing list of hardcoded reactions and turns it into an open subscription system. The publisher announces. Subscribers react. Neither knows about the other's internals. New reactions are added without modifying existing code, and removed without leaving holes.

Go makes both flavors clean. Interface-based observers give you synchronous, testable, contract-driven notification. Channel-based observers give you async, concurrent, naturally-backpressured event streams. Pick based on whether your observers need to complete before the publisher moves on, or whether they can process independently.

The discipline is cleanup. Observers that subscribe must eventually unsubscribe. Channels that open must eventually close. Goroutines that start must eventually stop. The pattern provides the mechanism for decoupling; the engineer provides the discipline for lifecycle management.

> **Announce changes, don't orchestrate reactions. Let observers decide what matters to them.**
