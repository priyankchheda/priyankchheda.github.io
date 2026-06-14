+++
date = '2026-06-14'
title = 'State Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about an order in an e-commerce system. An order starts as "pending," then moves to "paid," then "shipped," then "delivered." The same action -- calling `Cancel()` -- does something completely different depending on the current state. A pending order can be cancelled freely. A paid order needs a refund. A shipped order might be too late to cancel. A delivered order definitely can't be cancelled.

The naive approach: a `switch` on the state string inside every method. `Pay()` has a switch. `Cancel()` has a switch. `Ship()` has a switch. `Refund()` has a switch. Add a new state ("on hold," "returned") and you're modifying every method. Add a new action and you're adding another case to every state. The object becomes a sprawling decision matrix where behavior, transitions, and validation are all tangled together.

The **State pattern** eliminates this by giving each state its own object. The state object knows what actions are valid, how to execute them, and what state comes next. The order delegates to its current state. When the state changes, behavior changes -- automatically, without conditionals.

In this article, we'll build an order lifecycle with the State pattern, show how it eliminates switch statements, and explore the design decisions (where transitions live, how to handle invalid actions) that make or break the implementation in Go.

## What Is the State Pattern?

> "Allow an object to alter its behavior when its internal state changes. The object will appear to change its class."
> -- Gang of Four

The intent: **move state-dependent behavior out of the main object and into dedicated state structs.** Each state implements the same interface. The context (the order) holds a reference to the current state and delegates all behavior to it. When the state transitions, the context swaps its state reference to a different struct. Different struct, different behavior -- no conditionals needed.

Without the pattern, every method is a decision tree:

```go
func (o *Order) Cancel() {
    switch o.status {
    case "pending":
        o.status = "cancelled"
        fmt.Println("Order cancelled")
    case "paid":
        o.status = "cancelled"
        o.refund()
        fmt.Println("Order cancelled, refund issued")
    case "shipped":
        fmt.Println("Cannot cancel: already shipped")
    case "delivered":
        fmt.Println("Cannot cancel: already delivered")
    }
}
```

Every new state adds a case to every method. Every new method adds a switch on every state. The object grows multiplicatively.

With the State pattern, `Cancel()` becomes one line:

```go
func (o *Order) Cancel() {
    o.state.Cancel(o)
}
```

The current state object handles it. The order doesn't know or care which state it's in.

## Core Components

Three participants.

**State** -- the interface defining all actions the context supports. Every concrete state implements this.

```go
type OrderState interface {
    Pay(o *Order) error
    Ship(o *Order) error
    Deliver(o *Order) error
    Cancel(o *Order) error
    String() string
}
```

**ConcreteState** -- a struct for each state, implementing the interface with behavior specific to that state. Invalid actions return errors; valid actions perform work and trigger transitions.

**Context** -- the main object (the order). Holds the current state, delegates calls, and provides a `setState()` method for transitions.

```go
type Order struct {
    id    string
    state OrderState
}
```

The flow:

```
Client calls order.Pay()
  --> Order delegates to state.Pay(order)
    --> ConcreteState executes behavior
    --> ConcreteState transitions: order.setState(&PaidState{})
```

The context never checks which state it's in. The state objects handle everything.

## Code Walkthrough: An Order Lifecycle

We're building an order system with four states: Pending, Paid, Shipped, Delivered. Each state knows which actions are valid, how to execute them, and which state comes next.

### The State Interface

```go
type OrderState interface {
    Pay(o *Order) error
    Ship(o *Order) error
    Deliver(o *Order) error
    Cancel(o *Order) error
    String() string
}
```

Five methods. Every state must handle every action -- even if "handling" means returning an error for invalid transitions.

### The Context

```go
type Order struct {
    ID    string
    state OrderState
}

func NewOrder(id string) *Order {
    return &Order{
        ID:    id,
        state: &PendingState{},
    }
}

func (o *Order) SetState(s OrderState) {
    fmt.Printf("  [%s] %s --> %s\n", o.ID, o.state, s)
    o.state = s
}

func (o *Order) State() string {
    return o.state.String()
}

func (o *Order) Pay() error   { return o.state.Pay(o) }
func (o *Order) Ship() error  { return o.state.Ship(o) }
func (o *Order) Deliver() error { return o.state.Deliver(o) }
func (o *Order) Cancel() error { return o.state.Cancel(o) }
```

The context is thin: it holds state, logs transitions, and delegates. No business logic, no switches. `SetState` prints the transition for visibility.

### Concrete States

**PendingState** -- the order has been created but not paid:

```go
type PendingState struct{}

func (s *PendingState) Pay(o *Order) error {
    fmt.Printf("  [%s] processing payment\n", o.ID)
    o.SetState(&PaidState{})
    return nil
}

func (s *PendingState) Ship(o *Order) error {
    return fmt.Errorf("cannot ship: order %s is not paid", o.ID)
}

func (s *PendingState) Deliver(o *Order) error {
    return fmt.Errorf("cannot deliver: order %s is not shipped", o.ID)
}

func (s *PendingState) Cancel(o *Order) error {
    fmt.Printf("  [%s] cancelled (no payment to refund)\n", o.ID)
    o.SetState(&CancelledState{})
    return nil
}

func (s *PendingState) String() string { return "pending" }
```

**PaidState** -- payment received, ready to ship:

```go
type PaidState struct{}

func (s *PaidState) Pay(o *Order) error {
    return fmt.Errorf("order %s is already paid", o.ID)
}

func (s *PaidState) Ship(o *Order) error {
    fmt.Printf("  [%s] shipping order\n", o.ID)
    o.SetState(&ShippedState{})
    return nil
}

func (s *PaidState) Deliver(o *Order) error {
    return fmt.Errorf("cannot deliver: order %s is not shipped yet", o.ID)
}

func (s *PaidState) Cancel(o *Order) error {
    fmt.Printf("  [%s] cancelling and issuing refund\n", o.ID)
    o.SetState(&CancelledState{})
    return nil
}

func (s *PaidState) String() string { return "paid" }
```

**ShippedState** -- in transit:

```go
type ShippedState struct{}

func (s *ShippedState) Pay(o *Order) error {
    return fmt.Errorf("order %s is already paid", o.ID)
}

func (s *ShippedState) Ship(o *Order) error {
    return fmt.Errorf("order %s is already shipped", o.ID)
}

func (s *ShippedState) Deliver(o *Order) error {
    fmt.Printf("  [%s] delivered to customer\n", o.ID)
    o.SetState(&DeliveredState{})
    return nil
}

func (s *ShippedState) Cancel(o *Order) error {
    return fmt.Errorf("cannot cancel: order %s is already in transit", o.ID)
}

func (s *ShippedState) String() string { return "shipped" }
```

**DeliveredState** -- terminal state:

```go
type DeliveredState struct{}

func (s *DeliveredState) Pay(o *Order) error {
    return fmt.Errorf("order %s is already completed", o.ID)
}

func (s *DeliveredState) Ship(o *Order) error {
    return fmt.Errorf("order %s is already completed", o.ID)
}

func (s *DeliveredState) Deliver(o *Order) error {
    return fmt.Errorf("order %s is already delivered", o.ID)
}

func (s *DeliveredState) Cancel(o *Order) error {
    return fmt.Errorf("cannot cancel: order %s is already delivered", o.ID)
}

func (s *DeliveredState) String() string { return "delivered" }
```

**CancelledState** -- terminal state:

```go
type CancelledState struct{}

func (s *CancelledState) Pay(o *Order) error {
    return fmt.Errorf("order %s is cancelled", o.ID)
}

func (s *CancelledState) Ship(o *Order) error {
    return fmt.Errorf("order %s is cancelled", o.ID)
}

func (s *CancelledState) Deliver(o *Order) error {
    return fmt.Errorf("order %s is cancelled", o.ID)
}

func (s *CancelledState) Cancel(o *Order) error {
    return fmt.Errorf("order %s is already cancelled", o.ID)
}

func (s *CancelledState) String() string { return "cancelled" }
```

Each state is small and focused. It handles its own valid transitions and rejects invalid ones with clear error messages. No state knows about the internals of other states.

### Using It

```go
func main() {
    order := NewOrder("ORD-100")

    fmt.Println("=== Happy path ===")
    must(order.Pay())
    must(order.Ship())
    must(order.Deliver())

    fmt.Println("\n=== Invalid actions ===")
    fmt.Println(order.Cancel())

    fmt.Println("\n=== Cancel after payment ===")
    order2 := NewOrder("ORD-200")
    must(order2.Pay())
    must(order2.Cancel())
    fmt.Println("Final state:", order2.State())
}

func must(err error) {
    if err != nil {
        fmt.Println("ERROR:", err)
    }
}
```

```
=== Happy path ===
  [ORD-100] processing payment
  [ORD-100] pending --> paid
  [ORD-100] shipping order
  [ORD-100] paid --> shipped
  [ORD-100] delivered to customer
  [ORD-100] shipped --> delivered

=== Invalid actions ===
cannot cancel: order ORD-100 is already delivered

=== Cancel after payment ===
  [ORD-200] processing payment
  [ORD-200] pending --> paid
  [ORD-200] cancelling and issuing refund
  [ORD-200] paid --> cancelled
Final state: cancelled
```

The happy path flows through four states. Invalid actions produce clear errors without panics or undefined behavior. Cancellation after payment triggers a refund. Each transition is logged. The order object never contains a single conditional.

### State Transition Diagram

```
Pending --Pay()--> Paid --Ship()--> Shipped --Deliver()--> Delivered
   |                |
   |--Cancel()-->   |--Cancel()-->  Cancelled
```

## State vs Strategy

These patterns are structurally identical (context delegates to an interchangeable object behind an interface) but differ in intent and transition behavior.

**State** implies *transitions*. The current state changes as the object moves through its lifecycle. The object's behavior evolves over time. States know about and trigger transitions to other states.

**Strategy** implies *selection*. You pick an algorithm at construction time (or based on input) and it stays. There's no lifecycle, no transitions, no "after using strategy A, switch to strategy B."

The test: do the implementations *transition between each other* during the object's lifetime? If yes, it's State. If the implementation is chosen once and stays, it's Strategy.

## When to Use It

The State pattern fits when **an object's behavior changes based on its internal state, and the number of states and transitions is significant enough that conditionals become unwieldy**:

- **Lifecycle-driven objects** (orders, tickets, subscriptions) that move through well-defined stages with different behavior at each stage.
- **Protocol implementations** (TCP connections, authentication handshakes) where the same method (`receive`, `send`) does different things depending on the connection state.
- **Workflow engines** where each step has different valid actions, and the flow between steps follows defined rules.
- **UI state management** where components behave differently in loading, error, empty, and populated states.

Skip it when:

- **You have 2-3 states with simple logic.** A switch statement with three cases is clearer than five files with interfaces and structs. The pattern adds value when the switch would be replicated across many methods.
- **States don't have meaningfully different behavior.** If every state does the same thing with minor variations (different log messages), the pattern is over-engineering. A field with a simple check suffices.
- **Transitions are trivial.** If the lifecycle is linear (`A -> B -> C`) with no branching and no invalid actions, a sequential pipeline or simple status field is simpler.

## Pros and Cons

**Pros:**

- **Open/Closed for new states** -- add a new state (e.g., "on hold") by creating one struct. No existing states change. No method-level switches grow.
- **Localized behavior** -- all logic for a given state lives in one struct. Understanding "what happens when an order is shipped?" means reading one file, not grep-ing through every method.
- **Explicit transitions** -- each state declares which transitions are valid. Invalid transitions return errors rather than silently succeeding or falling through a default case.
- **Testable in isolation** -- test `PaidState.Cancel()` by passing it a mock order and verifying the transition. No need to set up the full lifecycle just to test one state's behavior.

**Cons:**

- **More types** -- five states means five structs, each implementing the full interface. For interfaces with many methods, this is verbose even when most implementations are one-line error returns.
- **Scattered transition logic** -- "what are all valid paths from Paid?" requires reading `PaidState`'s methods. The full state machine isn't visible in one place (unlike a transition table or diagram).
- **Context-state coupling** -- states receive `*Order` to trigger transitions (`o.SetState(...)`). This creates a tight coupling between states and the context type. Testing states requires a real or mocked context.

## Best Practices

- **Return errors for invalid transitions, don't panic or silently no-op.** The caller needs to know their action was rejected. A silent no-op masks bugs; a panic crashes production. Return `fmt.Errorf("cannot ship: order is not paid")` -- clear, actionable, safe.
- **Log transitions in `SetState()`.** A central transition point that logs `oldState --> newState` gives you audit trail for free. When debugging "how did this order end up cancelled?", the log tells the story.
- **Keep states stateless when possible.** If a state struct has no fields (`type PaidState struct{}`), it's reusable and trivially testable. The moment states carry data, you're managing per-instance state objects rather than shared behavior objects.
- **Consider a transition table for documentation.** Even though the code distributes transitions across states, maintain a comment or doc that shows the full state machine at a glance. Engineers reading the code for the first time need the big picture before diving into individual states.

## Common Mistakes

**Making the context a god object.** If `Order` has 20 fields that every state reads and modifies, the states are effectively just methods on the order with extra indirection. Keep the context lean -- state objects should need minimal context to do their job. If a state needs the order's payment amount to issue a refund, pass it explicitly or expose a method, don't give states full access to every field.

**Putting transition decisions in the context.** If `Order.Pay()` contains `if o.state == "pending" { o.SetState(&PaidState{}) }`, you've re-introduced the conditional logic the pattern was supposed to eliminate. Transitions belong *inside* the state objects. The context just delegates.

**Creating circular dependencies between state and context packages.** In larger codebases, states and context might live in separate packages. If states import the context package (to call `o.SetState`) and the context imports the state package (to reference `&PaidState{}`), you have a cycle. Fix: define the `State` interface in the context package, and have states implement it. Or use a shared types package for the interface.

**Forgetting terminal states.** `DeliveredState` and `CancelledState` reject all actions. Without them, there's no clear "this order is done" -- callers might keep calling methods on completed orders expecting something to happen. Terminal states make the lifecycle explicit and bounded.

## Final Thoughts

The State pattern replaces sprawling switch statements with focused state objects, each handling its own behavior and transitions. The context delegates without asking "what state am I in?" New states are added without modifying existing ones. Invalid actions are rejected locally with clear errors.

Go's interfaces make the pattern clean: define the state contract, implement it per state, and let the context hold an interface field. No abstract classes, no inheritance chains -- just an interface, a handful of structs, and delegation. The resulting code reads like a state machine description because it *is* one, structured as code rather than encoded in conditionals.

The honest test: if your object has more than three states and more than three state-dependent methods, the switch statements are already getting painful. That's when the State pattern earns its keep.

> **Let each state own its behavior. The object just delegates and never asks "what am I?"**
