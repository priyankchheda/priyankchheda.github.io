+++
date = '2025-06-30'
title = 'Simple Factory Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a notification service. It needs to send messages through different channels -- email, SMS, push notifications, Slack. The channel is determined at runtime: a user's preference, a config value, a feature flag. The sending logic differs per channel, but the caller just wants to say "send this message through the right channel" without knowing how each channel works internally.

The naive approach scatters `if/else` chains across every call site. Each function that needs to send a notification decides which struct to instantiate, how to configure it, and which method to call. Add a new channel -- webhook, for instance -- and you're modifying five different files. Remove one and you're grepping the codebase for direct references.

The **Simple Factory** pattern centralizes that decision into a single function. You give it a channel name, it gives you back an object that satisfies a common interface. The caller doesn't know or care which concrete type it received. It just calls `Send()` and the right thing happens.

This isn't a formal Gang of Four pattern -- it's simpler than that. But it's one of the most common patterns you'll encounter in Go codebases. `sql.Open()`, `errors.New()`, `sha256.New()` -- they all follow this shape. In this article, we'll build a notification factory, evolve it from a basic switch to a registration-based system, and explore how the pattern shows up in Go's standard library.

## What Is the Simple Factory Pattern?

> Encapsulate object creation in a single function that returns an interface based on input, so clients don't need to know about concrete types.

The intent is straightforward: **move the "which type to create" decision into one place.** The factory function takes some input (a string, an enum, a config value), creates the appropriate concrete type, and returns it as an interface. The caller works with the interface. The concrete type stays hidden.

Without a factory, notification sending looks like this:

```go
func SendNotification(channel string, msg string) error {
    switch channel {
    case "email":
        n := &EmailNotifier{smtpHost: "smtp.example.com", port: 587}
        return n.Send(msg)
    case "sms":
        n := &SMSNotifier{apiKey: "secret", region: "us-east-1"}
        return n.Send(msg)
    case "slack":
        n := &SlackNotifier{webhookURL: "https://hooks.slack.com/..."}
        return n.Send(msg)
    }
    return fmt.Errorf("unknown channel: %s", channel)
}
```

Every function that sends notifications repeats this switch. The concrete types and their initialization details leak into application logic. Adding a new channel means finding and updating every switch statement. Testing means the test must know about `EmailNotifier`, `SMSNotifier`, and `SlackNotifier` directly.

With a factory:

```go
notifier, err := NewNotifier("email")
if err != nil {
    return err
}
return notifier.Send(msg)
```

One function creates the right type. The caller works with an interface. Adding a new channel means updating one function. Testing means swapping the factory to return a mock.

## Core Components

Three participants.

**Product interface** -- the common contract that all types returned by the factory must satisfy.

```go
type Notifier interface {
    Send(message string) error
}
```

**Concrete products** -- the actual implementations. Each handles a specific channel.

**Factory function** -- the single function that accepts input and returns the appropriate concrete type as the interface.

```go
func NewNotifier(channel string) (Notifier, error)
```

The flow:

```txt
Client --> Factory(input) --> Concrete Product (returned as interface) --> Client uses interface
```

The client never imports or references concrete types. It depends only on the interface and the factory function.

## Code Walkthrough: A Notification Factory

We're building a notification system where the channel is selected at runtime -- from user config, environment variables, or feature flags.

### The Interface

```go
type Notifier interface {
    Send(message string) error
}
```

One method. Any type that can send a message is a Notifier.

### Concrete Implementations

**Email:**

```go
type EmailNotifier struct {
    from string
}

func (n *EmailNotifier) Send(message string) error {
    fmt.Printf("[email] from=%s | %s\n", n.from, message)
    return nil
}
```

**SMS:**

```go
type SMSNotifier struct {
    region string
}

func (n *SMSNotifier) Send(message string) error {
    fmt.Printf("[sms] region=%s | %s\n", n.region, message)
    return nil
}
```

**Slack:**

```go
type SlackNotifier struct {
    channel string
}

func (n *SlackNotifier) Send(message string) error {
    fmt.Printf("[slack] #%s | %s\n", n.channel, message)
    return nil
}
```

Each notifier has its own configuration. The factory handles initialization; the caller never touches these details.

### The Factory Function

```go
func NewNotifier(channel string) (Notifier, error) {
    switch channel {
    case "email":
        return &EmailNotifier{from: "noreply@example.com"}, nil
    case "sms":
        return &SMSNotifier{region: "us-east-1"}, nil
    case "slack":
        return &SlackNotifier{channel: "alerts"}, nil
    default:
        return nil, fmt.Errorf("unsupported notification channel: %s", channel)
    }
}
```

The factory returns `(Notifier, error)`. Unknown channels produce an error rather than a nil interface or a panic. The concrete types and their configuration are completely hidden from the caller.

### Using It

```go
func main() {
    channels := []string{"email", "sms", "slack", "carrier-pigeon"}

    for _, ch := range channels {
        notifier, err := NewNotifier(ch)
        if err != nil {
            fmt.Printf("[error] %v\n", err)
            continue
        }
        notifier.Send("System deployment complete")
    }
}

// OUTPUT:
// [email] from=noreply@example.com | System deployment complete
// [sms] region=us-east-1 | System deployment complete
// [slack] #alerts | System deployment complete
// [error] unsupported notification channel: carrier-pigeon
```

The loop doesn't know about `EmailNotifier`, `SMSNotifier`, or `SlackNotifier`. It works entirely through the `Notifier` interface. Adding a new channel means adding a case to `NewNotifier` -- nothing else changes.

## Evolving the Factory: Registration-Based Approach

The switch-based factory works well for small sets of types. But it violates the Open/Closed Principle: adding a new channel means modifying the factory function. For larger systems -- especially plugin architectures or modular services -- a **registration-based factory** is cleaner.

```go
var registry = map[string]func() Notifier{}

func Register(name string, constructor func() Notifier) {
    registry[name] = constructor
}

func NewNotifier(channel string) (Notifier, error) {
    constructor, ok := registry[channel]
    if !ok {
        return nil, fmt.Errorf("unsupported notification channel: %s", channel)
    }
    return constructor(), nil
}
```

Now each channel registers itself:

```go
func init() {
    Register("email", func() Notifier {
        return &EmailNotifier{from: "noreply@example.com"}
    })
}

func init() {
    Register("sms", func() Notifier {
        return &SMSNotifier{region: "us-east-1"}
    })
}
```

Adding a new channel means writing a new file with an `init()` function. The factory function never changes. This is the same pattern `database/sql` uses -- drivers register themselves via `sql.Register()` in their `init()` functions, and `sql.Open()` looks them up by name.

The tradeoff: implicit registration via `init()` can make it harder to trace which types are available. The factory's behavior depends on which packages were imported. This is fine for well-documented systems but can confuse newcomers who don't know they need to `import _ "myapp/notifiers/slack"` to make Slack available.

## The Pattern in Go's Standard Library

The Simple Factory shows up throughout Go's standard library, just without the name:

**`sql.Open(driver, dsn)`** -- accepts a driver name, looks it up in an internal registry, returns a `*sql.DB`. You never construct a Postgres driver struct directly.

**`errors.New(message)`** -- returns an `error` interface. The concrete type (`*errorString`) is unexported and invisible to callers.

**`sha256.New()`** -- returns a `hash.Hash` interface. The internal `digest` struct is hidden. Different algorithms (`sha512.New()`, `md5.New()`) all return the same interface.

**`os.Open(path)`** -- returns an `*os.File` that wraps OS-specific file descriptors. The caller works through a uniform API regardless of the underlying operating system.

All follow the same shape: a function accepts input, creates a concrete type internally, and returns it as an interface or abstract type. The caller depends on the abstraction, not the implementation. That's the Simple Factory pattern.

## When to Use It

The Simple Factory fits when **the type of object to create is decided at runtime, and the caller shouldn't know about concrete types**:

- **Runtime type selection** based on user input, config files, or environment variables. "Which notification channel?" "Which storage backend?" "Which serialization format?"
- **Centralized initialization logic** where concrete types need specific configuration (API keys, connection strings, default values) that shouldn't leak into calling code.
- **Interface-driven architectures** where application code depends on abstractions and concrete types are plugged in at startup or per-request.
- **Testing with swappable implementations** where the factory can be overridden to return mocks or fakes without changing application code.

Skip it when:

- **There's only one implementation.** If you'll never have more than one concrete type, a plain constructor function (`NewEmailNotifier()`) is simpler and more direct.
- **The type is always known at compile time.** If there's no runtime decision, the factory adds indirection for zero benefit. Just construct the struct directly.
- **You need extensive extensibility with plugin isolation.** For truly modular plugin systems, the Factory Method pattern (with interface-based factories) gives better extensibility than a single function with a switch or map.

## Pros and Cons

**Pros:**

- **Centralized creation logic** -- object initialization lives in one place. Change how an email notifier is configured? One function, one edit.
- **Client decoupling** -- calling code depends on the interface, not concrete types. Swapping implementations (real notifier for a mock, email for Slack) doesn't touch application logic.
- **Consistent initialization** -- every notifier created through the factory gets the same default configuration. No risk of one call site forgetting to set the API key.
- **Natural Go idiom** -- a function that returns an interface is how Go developers already write constructors. The pattern doesn't fight the language.

**Cons:**

- **Switch statement grows with types** -- the basic form violates Open/Closed. Adding a new channel means modifying the factory. The registration variant fixes this but adds complexity.
- **Runtime errors for unknown types** -- if the input string doesn't match any case, the error surfaces at runtime, not compile time. Typos in channel names produce silent failures in production.
- **Hides available types** -- the factory's switch or map determines what's available, but there's no compile-time enumeration. Discoverability depends on documentation or code reading.

## Best Practices

- **Always return `(Interface, error)`, not just `Interface`.** Unknown inputs are inevitable. A nil return without an error is a bug waiting to happen. Surface the failure explicitly.
- **Use the registration pattern for systems with 5+ types.** Once the switch statement gets long, a map of constructor functions keeps the factory itself stable and lets new types register without modifying it.
- **Make the factory a variable for testability.** `var NewNotifier = newNotifier` lets test code swap the factory to return mocks without touching production code. Restore it in test cleanup.

```go
var NewNotifier = newNotifier

func newNotifier(channel string) (Notifier, error) {
    // production implementation
}

// In tests:
func TestSomething(t *testing.T) {
    original := NewNotifier
    NewNotifier = func(channel string) (Notifier, error) {
        return &MockNotifier{}, nil
    }
    defer func() { NewNotifier = original }()
    // test code here
}
```

- **Keep the factory free of business logic.** The factory creates and returns. It doesn't validate messages, check permissions, or make business decisions. Those belong in the caller or the concrete types.

## Common Mistakes

**Returning concrete types instead of interfaces.** If `NewNotifier` returns `*EmailNotifier` instead of `Notifier`, the caller is coupled to the concrete type and can access fields or methods that aren't part of the contract. The whole point of the factory is to hide the concrete type behind an interface.

**Not handling the unknown-type case.** A factory that panics on unknown input, or worse, returns a nil interface without an error, causes crashes at runtime that are hard to trace. Always return an explicit error for unsupported inputs.

**Putting configuration inside the factory function.** If the factory hardcodes `smtpHost: "smtp.prod.example.com"`, you can't use it in tests or development environments. Pass configuration in (via parameters, a config struct, or closures) so the factory is environment-agnostic.

**Using a factory when a direct constructor is sufficient.** If there's only one implementation and no runtime selection, `NewEmailNotifier(config)` is clearer than `NewNotifier("email")`. The factory adds value when there's a *decision* to make. Without a decision, it's just indirection.

## Final Thoughts

The Simple Factory is one of the most pragmatic patterns in Go. It takes a common need -- "create the right object based on runtime input" -- and gives it a clean, centralized implementation. The caller works with an interface. The factory handles the decision. Adding new types is contained to one place.

Go's ecosystem validates this pattern constantly. `sql.Open()`, `errors.New()`, `sha256.New()` -- they all follow the same shape. A function takes input, creates a concrete type, and returns it as an interface. It's not a formal GoF pattern, but it's everywhere in production Go code because it solves a real problem simply and idiomatically.

The main judgment call is when to stop using the basic switch and graduate to registration. For three channels, the switch is fine. For ten, the map is better. For a plugin system, registration via `init()` is the way. Match the complexity of the factory to the complexity of the system it serves.

> **Give clients the interface they need, hide the type they don't, and keep the decision in one place.**
