+++
date = '2025-06-18'
title = 'Builder Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about constructing an HTTP client. It needs a base URL, a timeout, retry configuration, authentication headers, a transport with TLS settings, maybe a rate limiter, maybe a custom logger. Some of these are required. Most are optional. Many interact -- retries without a timeout is dangerous, TLS settings without HTTPS in the base URL is a misconfiguration.

The straightforward approach is a constructor with twelve parameters. Nobody remembers parameter order. Half the call sites pass zero values for options they don't care about. Changing the constructor signature breaks every caller. Adding validation means the constructor becomes a 50-line function that does too many things.

The **Builder pattern** separates construction from representation. Instead of one giant constructor call, you build the object step by step -- each step named, each step optional, each step independently understandable. A final `Build()` call assembles and validates the result. The construction process is explicit, readable, and safely extensible.

Go lacks constructor overloading and default parameters, which makes this problem especially acute. But Go also offers a powerful alternative: the **functional options pattern**, which achieves Builder-like ergonomics through closures. In this article, we'll build an HTTP client configurator using both the fluent builder and functional options approaches, compare them, and show when each is the right tool.

## What Is the Builder Pattern?

> "Separate the construction of a complex object from its representation so that the same construction process can create different representations."
> -- Gang of Four

The intent: **build complex objects step by step, with named configuration methods, and produce a validated final product.** The builder accumulates state through a series of calls, and `Build()` assembles it into the finished object. The caller never sees a half-constructed state.

Without a builder, complex construction looks like this:

```go
client := NewHTTPClient(
    "https://api.example.com", 30 * time.Second, 3, 2 * time.Second,
    "Bearer abc123", true, nil, "info")
```

What's the third parameter? Is it retries or timeout? What does `true` mean? What's `nil` for? The code compiles fine but communicates nothing. Change the parameter order and you have a silent bug.

With a builder:

```go
client, err := NewHTTPClientBuilder().
    BaseURL("https://api.example.com").
    Timeout(30 * time.Second).
    Retries(3, 2*time.Second).
    BearerToken("abc123").
    EnableTLS().
    LogLevel("info").
    Build()
```

Every value is named. Optional steps are simply omitted. Validation runs inside `Build()`. The construction reads like documentation.

## Core Components

Four participants, though in Go the Director is usually omitted.

**Product** -- the complex object being constructed. In our case, an HTTP client configuration.

```go
type HTTPClient struct {
    baseURL      string
    timeout      time.Duration
    maxRetries   int
    retryBackoff time.Duration
    authToken    string
    tlsEnabled   bool
    logLevel     string
}
```

**Builder** -- accumulates configuration and produces the product. Holds mutable state during construction, then returns an immutable (or at least complete) product.

**Director** (optional) -- encapsulates predefined construction sequences. "Build a production client" or "build a development client" with preset configurations. In Go, this is often just a named constructor function rather than a separate type.

**Client** -- the code that uses the builder. It calls configuration methods and eventually `Build()`.

The flow:

```txt
Client --> Builder.BaseURL() --> Builder.Timeout() --> ... --> Builder.Build() --> Product
```

The client drives construction. The builder accumulates state. `Build()` validates and assembles.

## Code Walkthrough: An HTTP Client Builder

We're building a configurable HTTP client with validation, defaults, and proper error handling on `Build()`.

### The Product

```go
type HTTPClient struct {
    baseURL      string
    timeout      time.Duration
    maxRetries   int
    retryBackoff time.Duration
    authToken    string
    tlsEnabled   bool
    logLevel     string
}

func (c *HTTPClient) String() string {
    return fmt.Sprintf("HTTPClient{url=%s timeout=%s retries=%d tls=%v log=%s}",
        c.baseURL, c.timeout, c.maxRetries, c.tlsEnabled, c.logLevel)
}
```

Unexported fields. Once built, the client's configuration can't be modified externally.

### The Builder

Each setter returns `*Builder` for method chaining. Defaults are set in the constructor:

```go
type HTTPClientBuilder struct {
    baseURL      string
    timeout      time.Duration
    maxRetries   int
    retryBackoff time.Duration
    authToken    string
    tlsEnabled   bool
    logLevel     string
}

func NewHTTPClientBuilder() *HTTPClientBuilder {
    return &HTTPClientBuilder{
        timeout:      10 * time.Second,
        maxRetries:   0,
        retryBackoff: 1 * time.Second,
        logLevel:     "warn",
    }
}
```

Defaults live in one place. The caller overrides only what they care about.

The configuration methods:

```go
func (b *HTTPClientBuilder) BaseURL(url string) *HTTPClientBuilder {
    b.baseURL = url
    return b
}

func (b *HTTPClientBuilder) Timeout(d time.Duration) *HTTPClientBuilder {
    b.timeout = d
    return b
}

func (b *HTTPClientBuilder) Retries(max int, backoff time.Duration) *HTTPClientBuilder {
    b.maxRetries = max
    b.retryBackoff = backoff
    return b
}

func (b *HTTPClientBuilder) BearerToken(token string) *HTTPClientBuilder {
    b.authToken = token
    return b
}

func (b *HTTPClientBuilder) EnableTLS() *HTTPClientBuilder {
    b.tlsEnabled = true
    return b
}

func (b *HTTPClientBuilder) LogLevel(level string) *HTTPClientBuilder {
    b.logLevel = level
    return b
}
```

Each method does one thing: set a value and return the builder. No side effects, no validation here -- that's `Build()`'s job.

### Validation in Build()

```go
func (b *HTTPClientBuilder) Build() (*HTTPClient, error) {
    if b.baseURL == "" {
        return nil, fmt.Errorf("build: baseURL is required")
    }
    if b.tlsEnabled && !strings.HasPrefix(b.baseURL, "https://") {
        return nil, fmt.Errorf("build: TLS enabled but baseURL is not HTTPS")
    }
    if b.maxRetries > 0 && b.timeout == 0 {
        return nil, fmt.Errorf("build: retries configured without a timeout")
    }

    return &HTTPClient{
        baseURL:      b.baseURL,
        timeout:      b.timeout,
        maxRetries:   b.maxRetries,
        retryBackoff: b.retryBackoff,
        authToken:    b.authToken,
        tlsEnabled:   b.tlsEnabled,
        logLevel:     b.logLevel,
    }, nil
}
```

`Build()` is the gatekeeper. Cross-field validation happens here -- TLS requires HTTPS, retries require a timeout. The product is never returned in an invalid state. This is where the pattern earns its keep over struct literals: illegal configurations are caught at construction time, not at runtime when the first request fails.

### Using It

```go
func main() {
    client, err := NewHTTPClientBuilder().
        BaseURL("https://api.example.com").
        Timeout(30 * time.Second).
        Retries(3, 2*time.Second).
        BearerToken("secret-token").
        EnableTLS().
        LogLevel("debug").
        Build()

    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println(client)
}

// OUTPUT:
// HTTPClient{url=https://api.example.com timeout=30s retries=3 tls=true log=debug}
```

A misconfiguration:

```go
client, err := NewHTTPClientBuilder().
    BaseURL("http://api.example.com").
    EnableTLS().
    Build()

if err != nil {
    fmt.Println("Error:", err)
}

// OUTPUT:
// Error: build: TLS enabled but baseURL is not HTTPS
```

The invalid configuration is caught before the client is ever used.

### Directors for Preset Configurations

A Director encapsulates common build sequences. In Go, this is often just a function:

```go
func ProductionClient(token string) (*HTTPClient, error) {
    return NewHTTPClientBuilder().
        BaseURL("https://api.production.internal").
        Timeout(10 * time.Second).
        Retries(3, 500*time.Millisecond).
        BearerToken(token).
        EnableTLS().
        LogLevel("error").
        Build()
}

func DevelopmentClient() (*HTTPClient, error) {
    return NewHTTPClientBuilder().
        BaseURL("http://localhost:8080").
        Timeout(5 * time.Second).
        LogLevel("debug").
        Build()
}
```

No Director struct needed. The function *is* the director. It encapsulates a preset configuration that multiple callers can share.

## The Functional Options Alternative

Go has an idiomatic alternative to the Builder pattern that achieves similar goals with a different shape: **functional options**. Instead of a builder struct with setter methods, you pass configuration functions to the constructor.

```go
type HTTPClient struct {
    baseURL      string
    timeout      time.Duration
    maxRetries   int
    retryBackoff time.Duration
    authToken    string
    tlsEnabled   bool
    logLevel     string
}

type Option func(*HTTPClient)

func WithBaseURL(url string) Option {
    return func(c *HTTPClient) { c.baseURL = url }
}

func WithTimeout(d time.Duration) Option {
    return func(c *HTTPClient) { c.timeout = d }
}

func WithRetries(max int, backoff time.Duration) Option {
    return func(c *HTTPClient) {
        c.maxRetries = max
        c.retryBackoff = backoff
    }
}

func WithBearerToken(token string) Option {
    return func(c *HTTPClient) { c.authToken = token }
}

func WithTLS() Option {
    return func(c *HTTPClient) { c.tlsEnabled = true }
}

func WithLogLevel(level string) Option {
    return func(c *HTTPClient) { c.logLevel = level }
}
```

The constructor applies options over defaults:

```go
func NewHTTPClient(opts ...Option) (*HTTPClient, error) {
    c := &HTTPClient{
        timeout:  10 * time.Second,
        logLevel: "warn",
    }
    for _, opt := range opts {
        opt(c)
    }
    if c.baseURL == "" {
        return nil, fmt.Errorf("baseURL is required")
    }
    if c.tlsEnabled && !strings.HasPrefix(c.baseURL, "https://") {
        return nil, fmt.Errorf("TLS enabled but baseURL is not HTTPS")
    }
    return c, nil
}
```

Usage:

```go
client, err := NewHTTPClient(
    WithBaseURL("https://api.example.com"),
    WithTimeout(30 * time.Second),
    WithRetries(3, 2*time.Second),
    WithBearerToken("secret"),
    WithTLS(),
)
```

This is the pattern you'll find in `google.golang.org/grpc`, `go.uber.org/zap`, and many production Go libraries. It's less boilerplate than a full builder struct, naturally supports defaults, and keeps the option functions independently testable.

**When to use which:** Functional options are more idiomatic for Go libraries and APIs. The fluent builder is clearer when construction has many steps, when you need to clone and modify builders, or when a Director pattern is valuable for preset sequences. Both solve the same underlying problem -- neither is universally better.

## When to Use It

The Builder pattern (or functional options) fits when **object construction is complex enough that a constructor or struct literal becomes unclear or unsafe**:

- **Objects with many optional parameters** where different call sites configure different subsets. The builder makes each choice explicit and named.
- **Cross-field validation** where certain combinations are invalid. `Build()` is the natural place to catch misconfiguration before runtime.
- **Multiple preset configurations** (production, development, testing) that share a construction process but differ in values. Directors or named constructor functions encapsulate these.
- **Library APIs** where you control the constructor but want users to customize behavior without knowing the full list of options upfront.

Skip it when:

- **The struct has 2-3 fields.** A struct literal is simpler and more readable. `Point{X: 1, Y: 2}` doesn't need a builder.
- **There's no validation or construction logic.** If all fields are independent and any combination is valid, Go's named field initialization is sufficient.
- **The object is constructed once in one place.** If there's no reuse, no variants, and no complexity, the builder adds ceremony for zero benefit.

## Pros and Cons

**Pros:**

- **Self-documenting construction** -- each method name explains what it configures. No more guessing what the third parameter means.
- **Single validation point** -- `Build()` catches invalid combinations (TLS without HTTPS, retries without timeout) before the object is ever used. Struct literals can't enforce this.
- **Defaults and optionality** -- the constructor sets sensible defaults. Callers override only what they need. No zero-value confusion.
- **Immutable products** -- once `Build()` returns, the product's state is fixed. The builder holds mutable state during construction; the product doesn't.

**Cons:**

- **Boilerplate** -- every configurable field needs a setter method (or an option function). For a struct with 15 fields, that's 15 methods, each 3-4 lines. It adds up.
- **No compile-time enforcement of required fields** -- if the caller forgets `.BaseURL()`, the error shows up at runtime in `Build()`, not at compile time. Go's type system can't prevent this.
- **Two ways to do the same thing in Go** -- teams sometimes debate whether to use fluent builders or functional options. Having both in a codebase creates inconsistency.

## Best Practices

- **Return `(*Product, error)` from `Build()`, not just `*Product`.** If construction can fail (missing required fields, invalid combinations), the error return makes failure explicit. Panicking inside `Build()` is hostile to callers.
- **Set defaults in the constructor, not in `Build()`.** `NewHTTPClientBuilder()` should return a builder with sensible defaults already set. The caller overrides; they don't need to know what the defaults are.
- **Keep setter methods pure -- no side effects.** A setter should store a value and return the builder. Validation, logging, or external calls in setters make the construction sequence order-dependent and harder to test.
- **Consider functional options for public library APIs.** The Go community recognizes the `WithX(value) Option` pattern. For internal application code, a fluent builder might be clearer. Match the audience's expectations.
- **Make the product's fields unexported.** If the product has exported fields, callers can bypass the builder entirely and construct invalid instances with struct literals. Unexported fields force construction through the builder.

## Common Mistakes

**Validating in setters instead of `Build()`.** If `EnableTLS()` panics when the base URL isn't HTTPS, but the caller hasn't called `BaseURL()` yet, the order of method calls matters. Validation belongs in `Build()` where all fields are set and cross-field checks are meaningful.

**Forgetting to reset the builder after `Build()`.** If a builder is reused to construct multiple products without resetting, the second product inherits state from the first. Either document that builders are single-use, or have `Build()` clear the builder's state.

**Returning `*Builder` instead of `Builder` from clone methods.** If you implement `Clone()` by returning `&clone` from a shallow copy, and the builder holds slices or maps, both copies share the underlying data. Modifications to one corrupt the other. Deep-copy reference types in `Clone()`.

**Using a builder when a struct literal is simpler.** A `UserBuilder` for a struct with `Name` and `Email` fields is over-engineering. The litmus test: if your builder's `Build()` method does nothing but copy fields into a struct with no validation, you don't need a builder.

## Final Thoughts

The Builder pattern gives complex objects a construction process that's readable, validatable, and safely extensible. Each step is named. Optional configuration is simply omitted. Cross-field validation runs once at the end, not scattered across the codebase.

Go offers two natural expressions of this idea: the fluent builder (method chaining on a struct) and functional options (closures applied to a config). Both solve "constructor with too many parameters." The fluent builder is more visible and familiar from other languages. Functional options are lighter and more idiomatic for Go libraries. Pick the one that fits your audience and stick with it.

The test for whether you need a builder is straightforward: if your struct literal has more than five fields, some are optional, and some combinations are invalid, a builder will make that construction safer and clearer. If not, keep it simple.

> **Build complex objects step by step, validate them at the end, and never hand out a half-constructed product.**
