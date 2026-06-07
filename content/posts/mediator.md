+++
date = '2026-03-18'
title = 'Mediator Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Consider an order processing system. When an order comes in, inventory checks stock, payment charges the card, shipping schedules delivery, notification emails the customer. In the first version, each service calls the next one directly. Inventory calls Payment. Payment calls Shipping. Shipping calls Notification. Four services, a clean chain.

Then requirements shift. If payment fails, inventory needs to release reserved stock. If shipping isn't available in the region, the customer gets a different notification. If the order exceeds a certain amount, fraud checks happen between inventory and payment. Suddenly every service holds references to two or three others, changes in one ripple across the system, and nobody can test the payment service without spinning up inventory, shipping, and notification alongside it.

The **Mediator pattern** exists for exactly this kind of mess. Instead of services calling each other, they all report to a single coordinator. The coordinator knows the interaction rules -- who reacts when, in what order, under what conditions. The services themselves become simpler: they do their work and emit events. They don't decide what happens next.

In this article, we'll build an order processing coordinator in Go that handles the happy path, payment failures, and shipping failures with proper rollback. Then we'll explore how Go's channels make the pattern feel native. Interfaces for decoupling, goroutines for concurrency, channels for communication -- the pieces fit together well.

## What Is the Mediator Pattern?

> "Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently."<br>-- Gang of Four

The core idea: **objects stop talking to each other and start talking through a middleman.** Each object (called a **colleague**) knows only the mediator interface. When something happens, the colleague notifies the mediator. The mediator decides who else needs to know and in what order.

Without this, a system with N interacting objects can develop up to N*(N-1) direct connections -- a web where every change sends ripples everywhere. With a mediator, each object has exactly one dependency: the mediator interface. The web collapses into a star.

The mediator doesn't do the work -- it coordinates the work. Inventory still checks stock. Payment still charges cards. But the *rules about what happens after* each step live in one place, not scattered across every service.

Here's what order processing looks like without the pattern:

```go
type PaymentService struct {
    inventory    *InventoryService
    shipping     *ShippingService
    notification *NotificationService
}

func (p *PaymentService) Process(orderID string) error {
    if err := p.charge(orderID); err != nil {
        p.inventory.Release(orderID)
        p.notification.Send(orderID, "payment_failed")
        return err
    }
    p.shipping.Schedule(orderID)
    p.notification.Send(orderID, "payment_success")
    return nil
}
```

`PaymentService` knows about three other services. It contains coordination logic that isn't about payment. Testing it means mocking inventory, shipping, and notification. Adding a fraud check means modifying `PaymentService` even though fraud has nothing to do with charging a card.

The Mediator pattern moves that coordination into a dedicated object. Payment charges the card and tells the mediator "done" or "failed." What happens next isn't its problem.

## Core Components

Four participants.

**Mediator** -- the interface declaring how colleagues communicate with the coordinator.

```go
type Mediator interface {
    Handle(event OrderEvent) error
}
```

**ConcreteMediator** -- implements the coordination logic. Knows all colleagues and the rules for how they interact.

**Colleague** -- any object participating in the mediated interaction. It holds a reference to the mediator interface and reports events. It doesn't know about other colleagues.

**ConcreteColleague** -- the actual services. Each does its own work, notifies the mediator on completion, and never touches another service directly.

The dependency structure is a star:

```
InventoryService ---> Mediator <--- PaymentService
                         ^
                         |
                   ShippingService
                         ^
                         |
                  NotificationService
```

Colleagues point inward to the mediator. The mediator points outward to all colleagues. Colleagues never point at each other.

## Code Walkthrough: An Order Processing Coordinator

We're building the order system from the intro. Four services coordinate through a mediator. The coordinator handles the happy path, payment failures, and shipping failures -- each with appropriate rollback.

### The Event Type

Instead of stringly-typed sender/event pairs, we'll use a typed event:

```go
type OrderEvent struct {
    OrderID string
    Kind    string
}
```

`Kind` identifies what happened: `"reserved"`, `"charged"`, `"charge_failed"`, `"shipped"`, `"shipping_failed"`. The coordinator branches on this value to decide what happens next.

### The Services (Colleagues)

Each service takes a mediator in its constructor and reports events after doing its work.

**Inventory:**

```go
type InventoryService struct {
    mediator Mediator
}

func NewInventoryService(m Mediator) *InventoryService {
    return &InventoryService{mediator: m}
}

func (s *InventoryService) Reserve(orderID string) error {
    fmt.Printf("  [inventory] reserving stock for %s\n", orderID)
    return s.mediator.Handle(OrderEvent{OrderID: orderID, Kind: "reserved"})
}

func (s *InventoryService) Release(orderID string) {
    fmt.Printf("  [inventory] releasing stock for %s\n", orderID)
}
```

**Payment:**

```go
type PaymentService struct {
    mediator   Mediator
    shouldFail bool
}

func NewPaymentService(m Mediator) *PaymentService {
    return &PaymentService{mediator: m}
}

func (s *PaymentService) Charge(orderID string) error {
    if s.shouldFail {
        fmt.Printf("  [payment] charge FAILED for %s\n", orderID)
        return s.mediator.Handle(OrderEvent{OrderID: orderID, Kind: "charge_failed"})
    }
    fmt.Printf("  [payment] charged for %s\n", orderID)
    return s.mediator.Handle(OrderEvent{OrderID: orderID, Kind: "charged"})
}

func (s *PaymentService) Refund(orderID string) {
    fmt.Printf("  [payment] refunded for %s\n", orderID)
}
```

**Shipping:**

```go
type ShippingService struct {
    mediator   Mediator
    shouldFail bool
}

func NewShippingService(m Mediator) *ShippingService {
    return &ShippingService{mediator: m}
}

func (s *ShippingService) Schedule(orderID string) error {
    if s.shouldFail {
        fmt.Printf("  [shipping] scheduling FAILED for %s\n", orderID)
        return s.mediator.Handle(OrderEvent{OrderID: orderID, Kind: "shipping_failed"})
    }
    fmt.Printf("  [shipping] scheduled for %s\n", orderID)
    return s.mediator.Handle(OrderEvent{OrderID: orderID, Kind: "shipped"})
}
```

**Notification:**

```go
type NotificationService struct{}

func (s *NotificationService) Send(orderID string, msg string) {
    fmt.Printf("  [notification] %s: %s\n", orderID, msg)
}
```

Key observation: no service holds a reference to any other service. Payment doesn't know Inventory exists. Shipping doesn't know about Notification. Services that participate in the workflow chain depend on exactly one thing -- the `Mediator` interface -- injected through their constructor. `NotificationService` is the exception: it's a terminal service that never reports back, so it has no mediator dependency at all.

### The Coordinator (Concrete Mediator)

The coordinator uses a handler map for dispatching. Each event kind maps to a function that orchestrates the response:

```go
type OrderCoordinator struct {
    inventory    *InventoryService
    payment      *PaymentService
    shipping     *ShippingService
    notification *NotificationService
    handlers     map[string]func(OrderEvent) error
}

func NewOrderCoordinator() *OrderCoordinator {
    c := &OrderCoordinator{}
    c.handlers = map[string]func(OrderEvent) error{
        "reserved":        c.onReserved,
        "charged":         c.onCharged,
        "charge_failed":   c.onChargeFailed,
        "shipped":         c.onShipped,
        "shipping_failed": c.onShippingFailed,
    }
    return c
}

func (c *OrderCoordinator) Handle(event OrderEvent) error {
    h, ok := c.handlers[event.Kind]
    if !ok {
        return fmt.Errorf("unhandled event: %s", event.Kind)
    }
    return h(event)
}
```

Each handler is a small, focused method:

```go
func (c *OrderCoordinator) onReserved(e OrderEvent) error {
    return c.payment.Charge(e.OrderID)
}

func (c *OrderCoordinator) onCharged(e OrderEvent) error {
    return c.shipping.Schedule(e.OrderID)
}

func (c *OrderCoordinator) onChargeFailed(e OrderEvent) error {
    c.inventory.Release(e.OrderID)
    c.notification.Send(e.OrderID, "Payment failed. Order cancelled.")
    return nil
}

func (c *OrderCoordinator) onShipped(e OrderEvent) error {
    c.notification.Send(e.OrderID, "Order confirmed and shipping scheduled.")
    return nil
}

func (c *OrderCoordinator) onShippingFailed(e OrderEvent) error {
    c.payment.Refund(e.OrderID)
    c.inventory.Release(e.OrderID)
    c.notification.Send(e.OrderID, "Shipping unavailable. Order refunded.")
    return nil
}
```

No giant switch statement. Adding a new event means writing a handler method and adding one line to the map. Each handler reads like a plain description of what should happen: "when shipping fails, refund the payment, release inventory, and notify the customer."

### Wiring It Together

```go
func main() {
    coord := NewOrderCoordinator()

    coord.inventory = NewInventoryService(coord)
    coord.payment = NewPaymentService(coord)
    coord.shipping = NewShippingService(coord)
    coord.notification = &NotificationService{}

    fmt.Println("--- Successful order ---")
    coord.inventory.Reserve("ORD-001")

    fmt.Println("\n--- Payment failure ---")
    coord.payment.shouldFail = true
    coord.inventory.Reserve("ORD-002")

    fmt.Println("\n--- Shipping failure ---")
    coord.payment.shouldFail = false
    coord.shipping.shouldFail = true
    coord.inventory.Reserve("ORD-003")
}
```

```
--- Successful order ---
  [inventory] reserving stock for ORD-001
  [payment] charged for ORD-001
  [shipping] scheduled for ORD-001
  [notification] ORD-001: Order confirmed and shipping scheduled.

--- Payment failure ---
  [inventory] reserving stock for ORD-002
  [payment] charge FAILED for ORD-002
  [inventory] releasing stock for ORD-002
  [notification] ORD-002: Payment failed. Order cancelled.

--- Shipping failure ---
  [inventory] reserving stock for ORD-003
  [payment] charged for ORD-003
  [shipping] scheduling FAILED for ORD-003
  [payment] refunded for ORD-003
  [inventory] releasing stock for ORD-003
  [notification] ORD-003: Shipping unavailable. Order refunded.
```

Three paths through the same workflow. The services didn't change between any of them -- only the conditions. Adding a fraud check between inventory and payment means writing an `onReserved` handler that calls `fraud.Check()` first, with its own success and failure events. The services remain untouched.

### Testing

Testing a service in isolation requires only a mock mediator:

```go
type MockMediator struct {
    events []OrderEvent
}

func (m *MockMediator) Handle(event OrderEvent) error {
    m.events = append(m.events, event)
    return nil
}

func TestInventoryReserve(t *testing.T) {
    mock := &MockMediator{}
    svc := NewInventoryService(mock)

    svc.Reserve("ORD-TEST")

    if len(mock.events) != 1 || mock.events[0].Kind != "reserved" {
        t.Error("expected 'reserved' event")
    }
}
```

Testing the coordinator means mocking the services and verifying the right methods are called in the right order for each event path. Each layer tests independently.

## Channels as Mediators

Go's channels map naturally to mediated communication. Instead of a struct-based coordinator, a goroutine consumes events from a channel and dispatches responses. In this model, both the colleagues *and* the coordinator communicate through the channel -- colleagues send events in, and the coordinator sends responses back or triggers side effects.

Here's a self-contained sketch of the coordinator side:

```go
func RunCoordinator(events <-chan OrderEvent) {
    for e := range events {
        switch e.Kind {
        case "reserved":
            fmt.Printf("  [coordinator] triggering payment for %s\n", e.OrderID)
        case "charged":
            fmt.Printf("  [coordinator] triggering shipping for %s\n", e.OrderID)
        case "charge_failed":
            fmt.Printf("  [coordinator] releasing inventory, notifying customer for %s\n", e.OrderID)
        case "shipped":
            fmt.Printf("  [coordinator] notifying customer for %s\n", e.OrderID)
        case "shipping_failed":
            fmt.Printf("  [coordinator] refunding payment, releasing inventory for %s\n", e.OrderID)
        }
    }
}
```

Colleagues send events into a shared channel instead of calling a method on a mediator interface:

```go
type ChannelInventory struct {
    events chan<- OrderEvent
}

func (s *ChannelInventory) Reserve(orderID string) {
    fmt.Printf("  [inventory] reserving %s\n", orderID)
    s.events <- OrderEvent{OrderID: orderID, Kind: "reserved"}
}
```

In a full implementation, the coordinator goroutine would hold references to channel-based colleagues and send events to trigger their work, creating a fully async event loop. The key architectural difference from the struct-based approach: everything communicates through channels, so there's no `Mediator` interface at all -- the channel *is* the mediator.

Because the coordinator runs in a single goroutine, all coordination logic is serialized -- no mutex needed. This follows Go's concurrency proverb: "share memory by communicating."

The channel approach fits naturally when events are asynchronous -- background jobs, webhook handlers, message queues. For synchronous request-response workflows where the caller needs the result before proceeding (like our order processing walkthrough), the struct-based approach is more straightforward. Pick based on whether your system is request-driven or event-driven.

## Mediator vs Observer vs Facade

These three patterns all deal with communication between objects, but they solve different problems.

**Mediator** has a central coordinator that knows all participants and actively decides who reacts to what. The logic is explicit: "when inventory reserves stock, tell payment to charge." The coordinator contains the workflow rules.

**Observer** is publish-subscribe. The subject broadcasts events. Observers subscribe independently. The subject doesn't know or care who's listening. There's no coordination logic -- just notification. "Something happened; react if you want."

**Facade** simplifies access to a subsystem behind a clean interface. It doesn't mediate between peers -- it wraps complexity. `orderFacade.Process()` might call inventory, payment, and shipping internally, but the facade controls the flow directly rather than coordinating between independent objects.

The decision heuristic: if objects need sequenced coordination with branching logic, use Mediator. If objects just need to be notified that something happened, use Observer. If you're wrapping a complex subsystem behind a simpler API, use Facade.

## When to Use It

The Mediator pattern earns its keep when **coordination complexity outpaces object complexity**:

- **Workflow orchestration** with branching and rollback -- order processing, onboarding flows, approval chains where the "what happens next" depends on what just happened.
- **Systems where interaction rules change frequently.** The services are stable; the coordination is volatile. Isolating the volatile part means the stable services don't change.
- **Components that need to be reusable across contexts.** A payment service that only knows a mediator interface can appear in an order workflow, a subscription workflow, and a refund workflow -- each with a different coordinator.
- **Concurrent systems in Go** where a goroutine consuming from a channel naturally serves as a thread-safe coordinator.

Skip it when:

- **Two objects have a simple, stable relationship.** A direct call is clearer than routing through a mediator for a single interaction.
- **The coordination is trivial.** If "A calls B, done" covers it, a mediator adds indirection for nothing.
- **You're tempted to put business logic in the mediator.** The coordinator should call `payment.Charge()`, not contain charging logic. If domain logic is creeping into the mediator, the services are too thin.

## Pros and Cons

**Pros:**

- **Decoupling** -- colleagues depend only on the mediator interface. Add, remove, or replace a service without touching the others.
- **Centralized coordination** -- all interaction rules live in one place. When someone asks "what happens after payment fails?", there's exactly one file to read.
- **Testability** -- mock the mediator to test a service in isolation. Mock the services to test the coordinator's routing logic. Each layer verifies independently.
- **Natural fit for Go's concurrency model** -- a single goroutine consuming from a channel gives you serialized coordination without explicit locking.

**Cons:**

- **God mediator risk** -- the mediator can grow into the most complex object in the system. Every new interaction rule goes there. Without discipline (splitting by workflow, using handler maps), it becomes a monolith.
- **Indirection cost** -- following the flow of execution means tracing through the coordinator. A direct call chain is easier to read when the coordination is simple.
- **Wiring overhead** -- the mediator needs references to all colleagues and all colleagues need a reference to the mediator. The setup code can feel ceremonial for smaller systems.

## Best Practices

- **Split coordinators by workflow.** An `OrderCoordinator` and a `RefundCoordinator` are better than a `MegaCoordinator` that handles everything. Each stays small, testable, and understandable.
- **Guard against circular notifications.** If service A notifies the coordinator, which calls service B, which notifies the coordinator, which calls A again -- you have infinite recursion. Be explicit about which events are "initial" (from external triggers) and which are "reactive" (from the coordinator's orchestration). Reactive calls typically shouldn't re-notify the coordinator.
- **Return errors from reactive calls.** When the coordinator calls `shipping.Schedule()` and it fails, the coordinator needs to handle that failure (refund payment, release inventory). If those reactive calls can't surface errors, the coordinator can't orchestrate rollback.
- **Test the coordinator's routing, not just the services.** Teams often test services in isolation (good) but skip verifying that the coordinator calls the right services in the right order for each event path. The coordinator contains the workflow -- it deserves its own test suite, including failure paths.

## Common Mistakes

**Stuffing domain logic into the coordinator.** The mediator should orchestrate, not compute. If you find it validating payment amounts, calculating shipping costs, or transforming data, that logic belongs in the services. The coordinator calls methods on services; it doesn't replace them.

**Using Mediator when Observer would suffice.** If there's no conditional routing -- just "notify everyone that something happened" -- Observer is simpler and more appropriate. Mediator's value is in the branching *decisions* ("if payment fails, do X; if it succeeds, do Y"), not the broadcasting.

**Letting the coordinator reach into colleague internals.** `coordinator.payment.balance = 0` is a code smell. The coordinator interacts through the colleague's public API. If the coordinator is manipulating fields directly, the colleague's API is incomplete.

**Ignoring the coordinator's own lifecycle.** In long-running systems, the coordinator may need graceful shutdown (closing channels, draining in-flight events), health checks, or metrics. Treating it as throwaway glue code leads to operational blind spots.

## Final Thoughts

The Mediator pattern takes a web of many-to-many dependencies and replaces it with a star: every service talks to one coordinator, and the coordinator talks to everyone. Services get simpler. Coordination gets centralized. Changes to the workflow happen in one place instead of rippling across every service.

Go makes this pattern particularly natural. Interfaces keep the mediator swappable. Constructors enforce that every service has a mediator from the start. Channels serve as built-in mediators for concurrent systems, and a single goroutine gives you serialized coordination without mutexes.

The discipline is keeping the coordinator thin. It orchestrates. It doesn't compute. The moment the coordinator contains business logic -- validation, pricing, data transformation -- it's a god object wearing a coordinator's name. Keep the services smart, the coordinator simple, and the pattern will serve you well.

> **When objects spend more time coordinating than working, give them a coordinator and let them focus.**
