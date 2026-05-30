+++
date = '2025-11-06'
title = 'Bridge Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about a notification system. You have different notification *types* -- alerts, reminders, marketing messages -- each with different formatting and urgency rules. You also have different delivery *channels* -- email, SMS, push notifications, Slack. These are two independent dimensions that vary separately.

Without the Bridge pattern, you end up with a combinatorial explosion: `AlertEmail`, `AlertSMS`, `AlertSlack`, `ReminderEmail`, `ReminderSMS`, `ReminderSlack`, `MarketingEmail`... Each combination is its own struct. Add a new channel (webhook) and you write three new structs. Add a new message type (digest) and you write four more. The codebase grows multiplicatively.

The **Bridge pattern** separates these two dimensions. The notification type (the *abstraction*) holds a reference to the delivery channel (the *implementation*). They connect through an interface, not inheritance. Mix and match at runtime: any notification type can use any channel. Add a channel without touching notification types. Add a notification type without touching channels. Each dimension grows independently.

Go's interfaces and composition make this especially natural. The abstraction struct holds an interface field for the implementation. No class hierarchies, no inheritance chains, no framework. In this article, we'll build a notification system with multiple message types and delivery channels, show how the Bridge prevents combinatorial explosion, and explore how it differs from patterns that look similar (Adapter, Strategy).

## What Is the Bridge Pattern?

> "Decouple an abstraction from its implementation so that the two can vary independently."
> -- Gang of Four

The intent: **separate two dimensions of variation into independent hierarchies connected by composition.** The abstraction defines high-level behavior. The implementation defines low-level mechanics. They interact through an interface, and either side can be extended without affecting the other.

The key distinction from other structural patterns: **Bridge is a design decision made upfront, not a fix applied after the fact.** Adapter retrofits compatibility between existing interfaces. Bridge *designs* for independent variation from the start.

Without Bridge, a notification system with rigid coupling looks like this:

```go
type EmailAlert struct{}
func (e *EmailAlert) Send(msg string) { /* format as alert, send via email */ }

type SMSAlert struct{}
func (s *SMSAlert) Send(msg string) { /* format as alert, send via SMS */ }

type EmailReminder struct{}
func (e *EmailReminder) Send(msg string) { /* format as reminder, send via email */ }

type SMSReminder struct{}
func (s *SMSReminder) Send(msg string) { /* format as reminder, send via SMS */ }
```

Four structs for 2x2 combinations. Each struct mixes formatting logic with delivery logic. Adding Slack means three new structs. Adding a "digest" type means four new structs. Neither dimension can evolve independently.

With Bridge, you get two small hierarchies and composition:

```go
notification := &Alert{channel: &EmailChannel{}}
notification.Notify("Server is down")

notification.channel = &SlackChannel{}
notification.Notify("Server recovered")
```

The alert knows how to format. The channel knows how to deliver. They connect at runtime.

## Core Components

Four participants.

**Implementor** -- the interface defining the low-level operations that vary independently from the abstraction.

```go
type Channel interface {
    Deliver(recipient string, subject string, body string) error
}
```

**ConcreteImplementor** -- actual implementations of the low-level interface. One per delivery mechanism.

**Abstraction** -- the high-level struct that holds a reference to the Implementor and defines the high-level behavior.

```go
type Notification struct {
    channel Channel
}
```

**RefinedAbstraction** -- extends the abstraction with specialized behavior (different notification types) without changing the implementor.

The flow:

```txt
Client --> RefinedAbstraction (Alert/Reminder) --> Implementor interface (Channel) --> ConcreteImplementor (Email/SMS/Slack)
```

Two hierarchies, one bridge. The abstraction hierarchy grows vertically (new notification types). The implementor hierarchy grows vertically (new channels). They connect horizontally through the interface.

## Code Walkthrough: A Notification System

We're building a notification system with two dimensions: notification types (alert, reminder, digest) and delivery channels (email, SMS, Slack). The Bridge keeps them independent.

### The Implementor Interface (Channel)

```go
type Channel interface {
    Deliver(recipient string, subject string, body string) error
}
```

One method. Any delivery mechanism that can take a recipient, subject, and body satisfies it.

### Concrete Implementors

**Email:**

```go
type EmailChannel struct {
    smtpHost string
}

func (c *EmailChannel) Deliver(recipient string, subject string, body string) error {
    fmt.Printf("  [email → %s] Subject: %s\n  Body: %s\n", recipient, subject, body)
    return nil
}
```

**SMS:**

```go
type SMSChannel struct {
    apiKey string
}

func (c *SMSChannel) Deliver(recipient string, subject string, body string) error {
    fmt.Printf("  [sms → %s] %s: %s\n", recipient, subject, body)
    return nil
}
```

**Slack:**

```go
type SlackChannel struct {
    webhookURL string
}

func (c *SlackChannel) Deliver(recipient string, subject string, body string) error {
    fmt.Printf("  [slack → #%s] *%s*\n  %s\n", recipient, subject, body)
    return nil
}
```

Each channel has its own configuration (SMTP host, API key, webhook URL) and its own formatting. They share nothing beyond the `Channel` interface.

### The Abstraction (Notification)

The base abstraction holds the channel and provides a method that refined abstractions can call:

```go
type Notification struct {
    channel   Channel
    recipient string
}

func NewNotification(channel Channel, recipient string) Notification {
    return Notification{channel: channel, recipient: recipient}
}

func (n *Notification) send(subject string, body string) error {
    return n.channel.Deliver(n.recipient, subject, body)
}
```

### Refined Abstractions (Notification Types)

Each notification type knows how to format its content, then delegates delivery to the channel.

**Alert** -- urgent, prefixed with severity:

```go
type Alert struct {
    Notification
    severity string
}

func NewAlert(channel Channel, recipient string, severity string) *Alert {
    return &Alert{
        Notification: NewNotification(channel, recipient),
        severity:     severity,
    }
}

func (a *Alert) Notify(message string) error {
    subject := fmt.Sprintf("[%s ALERT]", a.severity)
    body := fmt.Sprintf("URGENT: %s\nAction required immediately.", message)
    return a.send(subject, body)
}
```

**Reminder** -- friendly, includes a deadline:

```go
type Reminder struct {
    Notification
    deadline string
}

func NewReminder(channel Channel, recipient string, deadline string) *Reminder {
    return &Reminder{
        Notification: NewNotification(channel, recipient),
        deadline:     deadline,
    }
}

func (r *Reminder) Notify(message string) error {
    subject := "Reminder"
    body := fmt.Sprintf("%s\nDeadline: %s", message, r.deadline)
    return r.send(subject, body)
}
```

**Digest** -- batches multiple items into one message:

```go
type Digest struct {
    Notification
    items []string
}

func NewDigest(channel Channel, recipient string) *Digest {
    return &Digest{
        Notification: NewNotification(channel, recipient),
    }
}

func (d *Digest) Add(item string) {
    d.items = append(d.items, item)
}

func (d *Digest) Notify(title string) error {
    subject := fmt.Sprintf("Daily Digest: %s", title)
    body := ""
    for i, item := range d.items {
        body += fmt.Sprintf("  %d. %s\n", i+1, item)
    }
    return d.send(subject, body)
}
```

Three notification types, each with different formatting logic. None of them know or care whether they're delivered via email, SMS, or Slack. That's the channel's job.

### Putting It Together

```go
func main() {
    email := &EmailChannel{smtpHost: "smtp.example.com"}
    sms := &SMSChannel{apiKey: "key123"}
    slack := &SlackChannel{webhookURL: "https://hooks.slack.com/..."}

    alert := NewAlert(email, "ops@example.com", "CRITICAL")
    alert.Notify("Database connection pool exhausted")

    fmt.Println()

    reminder := NewReminder(sms, "+1-555-0100", "2024-03-15")
    reminder.Notify("Submit quarterly report")

    fmt.Println()

    digest := NewDigest(slack, "engineering")
    digest.Add("PR #421 merged")
    digest.Add("Deploy v2.3.1 completed")
    digest.Add("3 new issues opened")
    digest.Notify("Engineering Updates")
}
```

```txt
  [email → ops@example.com] Subject: [CRITICAL ALERT]
  Body: URGENT: Database connection pool exhausted
Action required immediately.

  [sms → +1-555-0100] Reminder: Submit quarterly report
Deadline: 2024-03-15

  [slack → #engineering] *Daily Digest: Engineering Updates*
    1. PR #421 merged
    2. Deploy v2.3.1 completed
    3. 3 new issues opened
```

Three notification types, three channels, nine possible combinations -- but only six structs total (three channels + three notification types), not nine. And adding a fourth channel (webhook) requires writing one struct, not three. Adding a fourth notification type (weekly summary) requires one struct, not four.

### Runtime Swapping

The bridge connects at runtime. The same alert can be routed to different channels based on context:

```go
alert := NewAlert(email, "ops@example.com", "HIGH")
alert.Notify("Disk usage at 90%")

escalated := NewAlert(slack, "oncall", "CRITICAL")
escalated.Notify("Disk usage at 95% - escalating")
```

Same notification type, different channel. No new structs. No conditionals.

## Bridge vs Adapter vs Strategy

These three patterns all involve connecting things through interfaces, but with different intents.

**Bridge** separates two *hierarchies* that both need to grow independently. It's a design-time decision about structure. The abstraction and implementation are both *yours* -- you're choosing to keep them decoupled from the start.

**Adapter** makes *existing incompatible things* work together after the fact. You didn't design them to fit; the adapter forces compatibility. It's a retrofit, not an architectural choice.

**Strategy** swaps *algorithms* behind a single interface. There's no abstraction hierarchy -- just one context that delegates to interchangeable strategies. It looks structurally similar to Bridge but has only one dimension of variation (the strategy), not two.

The test: if you have two independent dimensions that both grow over time, it's Bridge. If you're fixing an interface mismatch, it's Adapter. If you're swapping behavior behind a stable context, it's Strategy.

## When to Use It

The Bridge pattern fits when **you have two or more independent dimensions of variation that would cause combinatorial explosion if combined through inheritance**:

- **Multi-channel notification/messaging systems** where message types and delivery channels vary independently. New channels shouldn't require new message types and vice versa.
- **Cross-platform abstractions** where high-level operations (file system, rendering, networking) must work across multiple backends (Linux/Windows/Mac, OpenGL/Vulkan, TCP/UDP).
- **Payment processing** where payment methods (credit card, bank transfer, cryptocurrency) and payment gateways (Stripe, PayPal, Adyen) are two separate dimensions.
- **Rendering or formatting** where content types (report, invoice, receipt) and output formats (PDF, HTML, plaintext) are orthogonal concerns.

Skip it when:

- **There's only one dimension of variation.** If you have multiple channels but only one notification type, a simple interface with multiple implementations (Strategy pattern) is sufficient. Bridge adds unnecessary structure.
- **The dimensions won't grow.** If you'll only ever have two message types and two channels, four structs is fine. Bridge pays off at scale; for small, stable systems it's over-engineering.
- **The two dimensions are tightly coupled.** If an alert *must* use email and only email, there's no independence to preserve. Bridge is for orthogonal concerns.

## Pros and Cons

**Pros:**

- **Linear growth instead of multiplicative** -- M abstractions + N implementations = M + N structs, not M * N. This is the primary structural benefit.
- **Independent extensibility** -- add a new channel without touching any notification type. Add a new notification type without touching any channel. Each side evolves on its own schedule.
- **Runtime composition** -- the abstraction can switch implementations at runtime. Route alerts to email during business hours and to SMS after hours, without changing the alert's code.
- **Clean separation of concerns** -- formatting logic stays in notification types, delivery logic stays in channels. Neither bleeds into the other.

**Cons:**

- **Upfront design cost** -- you must identify the two dimensions of variation before building. If you guess wrong about what varies independently, you've added structure for no benefit.
- **More types to navigate** -- the codebase has two interface hierarchies plus the connecting structs. For developers unfamiliar with the pattern, understanding the flow requires tracing through the bridge.
- **Indirection in debugging** -- when a notification fails, you trace through the abstraction, then across the bridge, then into the implementor. More layers between the call site and the actual behavior.

## Best Practices

- **Define the implementor interface from the abstraction's perspective.** The `Channel` interface should expose what the notification *needs* (`Deliver`), not what each channel *can do*. Keep it minimal -- one or two methods that represent the essential operation.
- **Inject the implementor through the constructor.** `NewAlert(channel, recipient, severity)` makes the dependency explicit. Avoid setters that allow the implementor to be nil between construction and first use.
- **Keep refined abstractions independent of each other.** `Alert` shouldn't know about `Reminder`. They share the base `Notification` struct for the bridge connection, but otherwise have no relationship. If they start depending on each other, the abstraction hierarchy isn't properly designed.
- **Don't put formatting logic in the channel.** The channel delivers. The notification type formats. If `EmailChannel.Deliver` starts wrapping content in HTML templates, it's doing the notification type's job. Keep the boundary clean.

## Common Mistakes

**Identifying the wrong dimensions.** If "notification type" and "channel" aren't actually orthogonal in your system -- if alerts always go to Slack and reminders always go to email -- the Bridge adds structure without providing flexibility. Before applying the pattern, verify that both dimensions genuinely need to vary independently.

**Making the implementor interface too rich.** A `Channel` interface with `Deliver`, `FormatHeader`, `FormatFooter`, `ValidateRecipient`, and `GetDeliveryStatus` forces every channel implementation to handle concerns that might not apply. SMS doesn't have headers or footers. Keep the interface minimal and push non-universal behavior into the specific implementations or into the abstraction layer.

**Using Bridge when Strategy would suffice.** If there's no abstraction hierarchy -- just one `NotificationService` that delegates delivery to interchangeable channels -- that's Strategy, not Bridge. The distinction matters because Bridge's value comes from having *two* growing hierarchies. If you only have one, the pattern adds unnecessary complexity.

**Confusing Bridge with Adapter.** Adapter is applied retroactively to make incompatible things work together. Bridge is designed proactively to prevent tight coupling between two dimensions. If you're building a new system and choosing how to structure it, that's Bridge territory. If you're integrating an existing library you can't change, that's Adapter territory.

## Final Thoughts

The Bridge pattern separates two dimensions of variation into independent hierarchies connected by an interface. Each side grows on its own. Combinations happen at runtime through composition, not at compile time through inheritance. The result is linear growth instead of combinatorial explosion.

Go makes this particularly clean. The implementor is an interface. The abstraction is a struct with an interface field. Refined abstractions embed the base struct. No abstract classes, no inheritance trees, no framework -- just interfaces, structs, and composition. It's the Bridge pattern expressed in Go's native idioms.

The honest test: if you can draw your system as a matrix (notification types on one axis, delivery channels on the other), and both axes will grow independently, Bridge is the right structure. If one axis is fixed or the axes aren't truly independent, a simpler pattern will serve you better.

> **When two things vary independently, connect them through an interface and let each grow on its own.**
