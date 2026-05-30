+++
date = '2026-01-10'
title = 'Decorator Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about an HTTP handler. It processes a request and returns a response. Simple. Then you need logging -- what came in, what went out, how long it took. Then authentication -- reject unauthenticated requests before they reach the handler. Then rate limiting. Then request tracing. Then response compression.

Each feature is a cross-cutting concern that wraps the core logic. You don't want to modify the handler for each one -- that violates Open/Closed and turns a 10-line handler into a 200-line monster. You don't want separate handler types for every combination (`AuthenticatedLoggingCompressedHandler`) -- that's combinatorial explosion.

The **Decorator pattern** solves this by wrapping. Each decorator implements the same interface as the thing it wraps, adds one behavior (logging, auth, rate limiting), and delegates to the wrapped handler. Stack them: `Logging(Auth(RateLimit(handler)))`. Each layer is independent, composable, and removable. The handler at the center has no idea it's wrapped.

Go's `io.Reader`, `io.Writer`, and `http.Handler` all use this pattern extensively. `gzip.NewWriter(w)` wraps a writer with compression. `bufio.NewReader(r)` wraps a reader with buffering. HTTP middleware stacks are decorators by another name. In this article, we'll build a data processing pipeline with layered decorators, then connect it to the middleware pattern that every Go web developer already uses.

## What Is the Decorator Pattern?

> "Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality."
> -- Gang of Four

The intent: **add behavior to an object without modifying it, by wrapping it in another object that implements the same interface.** The wrapper intercepts calls, does something extra (before, after, or around), and delegates to the wrapped object. The client sees the same interface regardless of how many decorators are stacked.

The key constraint: **decorators don't change the interface.** They add behavior, not methods. The wrapped object and every decorator around it all satisfy the same contract. This is what makes them stackable and transparent -- the client can't tell (and doesn't care) whether it's talking to the core object or ten layers of decoration.

Without the pattern, adding features means modifying the original:

```go
func (h *Handler) Process(data []byte) ([]byte, error) {
    start := time.Now()
    log.Printf("processing %d bytes", len(data))

    if !h.isAuthenticated() {
        return nil, ErrUnauthorized
    }

    result, err := h.coreLogic(data)

    log.Printf("completed in %v", time.Since(start))
    return result, err
}
```

Logging, auth, and timing are mixed with business logic. Testing the core logic requires dealing with auth. Removing logging for production means editing the function. Each concern is welded to the handler.

With decorators, each concern is a separate wrapper:

```go
handler := NewTimingDecorator(
    NewLoggingDecorator(
        NewAuthDecorator(
            &CoreHandler{},
        ),
    ),
)
```

Each layer does one thing. Remove timing by removing one line. Add compression by wrapping one more layer. The core handler is untouched.

## Core Components

Four participants (though in Go the third is often omitted).

**Component** -- the interface that both the core object and all decorators implement.

```go
type DataProcessor interface {
    Process(data []byte) ([]byte, error)
}
```

**ConcreteComponent** -- the core object that does the actual work. No awareness of decoration.

**Decorator** -- a wrapper that implements the same interface and holds a reference to another Component. Each decorator adds one specific behavior.

**Client** -- works with the Component interface. Doesn't know how many decorators are stacked.

The flow (from outside in):

```
Client --> Decorator A --> Decorator B --> Decorator C --> ConcreteComponent
              |                |               |                |
          adds timing    adds logging    adds validation    does real work
```

Each decorator calls `wrapped.Process(data)` internally, adding its behavior before, after, or around the delegation.

## Code Walkthrough: A Data Processing Pipeline

We're building a data processing system where the core processor transforms data, and decorators add logging, timing, validation, and compression around it -- each independently, each stackable.

### The Interface

```go
type DataProcessor interface {
    Process(data []byte) ([]byte, error)
}
```

One method. Every processor -- core or decorator -- satisfies this.

### The Core Processor

```go
type JSONTransformer struct{}

func (t *JSONTransformer) Process(data []byte) ([]byte, error) {
    var raw map[string]interface{}
    if err := json.Unmarshal(data, &raw); err != nil {
        return nil, fmt.Errorf("transform: invalid JSON: %w", err)
    }
    raw["processed"] = true
    raw["timestamp"] = time.Now().Unix()
    return json.Marshal(raw)
}
```

A simple transformer that parses JSON, adds fields, and re-encodes. This is the core logic that decorators will wrap.

### Decorator 1: Logging

Logs the input size and output size:

```go
type LoggingDecorator struct {
    wrapped DataProcessor
    label   string
}

func NewLoggingDecorator(wrapped DataProcessor, label string) *LoggingDecorator {
    return &LoggingDecorator{wrapped: wrapped, label: label}
}

func (d *LoggingDecorator) Process(data []byte) ([]byte, error) {
    fmt.Printf("[%s] input: %d bytes\n", d.label, len(data))
    result, err := d.wrapped.Process(data)
    if err != nil {
        fmt.Printf("[%s] error: %v\n", d.label, err)
        return nil, err
    }
    fmt.Printf("[%s] output: %d bytes\n", d.label, len(result))
    return result, nil
}
```

### Decorator 2: Timing

Measures and reports execution duration:

```go
type TimingDecorator struct {
    wrapped DataProcessor
}

func NewTimingDecorator(wrapped DataProcessor) *TimingDecorator {
    return &TimingDecorator{wrapped: wrapped}
}

func (d *TimingDecorator) Process(data []byte) ([]byte, error) {
    start := time.Now()
    result, err := d.wrapped.Process(data)
    fmt.Printf("[timing] %v\n", time.Since(start))
    return result, err
}
```

### Decorator 3: Validation

Rejects empty or oversized input before it reaches the core:

```go
type ValidationDecorator struct {
    wrapped  DataProcessor
    maxBytes int
}

func NewValidationDecorator(wrapped DataProcessor, maxBytes int) *ValidationDecorator {
    return &ValidationDecorator{wrapped: wrapped, maxBytes: maxBytes}
}

func (d *ValidationDecorator) Process(data []byte) ([]byte, error) {
    if len(data) == 0 {
        return nil, fmt.Errorf("validation: empty input")
    }
    if len(data) > d.maxBytes {
        return nil, fmt.Errorf("validation: input exceeds %d bytes", d.maxBytes)
    }
    return d.wrapped.Process(data)
}
```

### Stacking Decorators

```go
func main() {
    core := &JSONTransformer{}

    pipeline := NewTimingDecorator(
        NewLoggingDecorator(
            NewValidationDecorator(core, 1024),
            "json-processor",
        ),
    )

    input := []byte(`{"name":"alice","role":"engineer"}`)

    result, err := pipeline.Process(input)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", string(result))
}

// OUTPUT:
// [json-processor] input: 34 bytes
// [json-processor] output: 79 bytes
// [timing] 42.125µs
// Result: {"name":"alice","processed":true,"role":"engineer","timestamp":1710000000}
```

The execution flows inward: timing starts the clock, logging records input size, validation checks bounds, the core transformer does its work. Results flow outward: validation passes through, logging records output size, timing stops the clock.

The visualization:

```
TimingDecorator
  └── LoggingDecorator
        └── ValidationDecorator
              └── JSONTransformer (core)
```

Remove timing by not wrapping it. Add compression by wrapping another layer. Reorder by changing the nesting. The core transformer never changes.

### Error Propagation

When validation rejects input:

```go
result, err := pipeline.Process([]byte{})
```

```txt
[json-processor] input: 0 bytes
[json-processor] error: validation: empty input
[timing] 208ns
Error: validation: empty input
```

The validation decorator returns an error without calling the core processor. Logging still prints (it wraps validation and sees both the input and the error). Timing still reports (it wraps everything). The core transformer is never reached. Each layer handles errors according to its position in the stack.

## The Pattern in Go's Standard Library

Go's `io` package is built on decorators. Every wrapper satisfies the same interface it wraps:

**`bufio.NewReader(r io.Reader)`** -- wraps a Reader with buffering. Returns an `io.Reader`.

**`gzip.NewReader(r io.Reader)`** -- wraps a Reader with decompression. Returns an `io.Reader`.

**`io.LimitReader(r io.Reader, n int64)`** -- wraps a Reader with a byte limit. Returns an `io.Reader`.

Stack them:

```go
gr, _ := gzip.NewReader(bufio.NewReader(file))
r := io.LimitReader(gr, 1024*1024)
```

Buffered reading → gzip decompression → limit to 1MB. Each layer is a decorator wrapping the previous one. The consumer calls `r.Read(buf)` and doesn't know how many layers exist.

HTTP middleware follows the same pattern with `http.Handler`:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
```

`loggingMiddleware` takes a handler, wraps it, returns a handler. Same interface in, same interface out. That's the Decorator pattern.

## Functional Decorators: The Go-Idiomatic Variation

For interfaces with a single method, Go offers a lighter alternative: function types with decorator functions.

```go
type ProcessFunc func(data []byte) ([]byte, error)

func WithLogging(fn ProcessFunc, label string) ProcessFunc {
    return func(data []byte) ([]byte, error) {
        fmt.Printf("[%s] processing %d bytes\n", label, len(data))
        result, err := fn(data)
        if err == nil {
            fmt.Printf("[%s] produced %d bytes\n", label, len(result))
        }
        return result, err
    }
}

func WithTiming(fn ProcessFunc) ProcessFunc {
    return func(data []byte) ([]byte, error) {
        start := time.Now()
        result, err := fn(data)
        fmt.Printf("[timing] %v\n", time.Since(start))
        return result, err
    }
}
```

Compose them:

```go
process := WithTiming(WithLogging(coreProcess, "transform"))
result, err := process(input)
```

No structs, no constructors -- just functions wrapping functions. This is exactly how `http.HandlerFunc` middleware works. The struct-based approach is better when decorators need configuration (like `maxBytes` for validation) or when the interface has multiple methods.

## When to Use It

The Decorator pattern fits when **you need to add behavior to objects dynamically, without modifying them, and different combinations of behaviors are needed**:

- **Cross-cutting concerns** like logging, timing, tracing, and metrics that should wrap business logic without contaminating it.
- **Configurable feature layers** that are enabled or disabled per environment (compression in production, verbose logging in development).
- **I/O wrapping** where buffering, compression, encryption, or rate-limiting layers compose around a base reader/writer.
- **HTTP middleware** where authentication, CORS, rate limiting, and request logging stack around handlers.

Skip it when:

- **The behavior is always present.** If every processor always needs validation, put it in the core type. A decorator that's never removed is just indirection.
- **The interface has many methods.** A decorator for a 10-method interface must implement all 10 methods, with 9 of them just delegating. This becomes boilerplate-heavy. Consider a different pattern or interface segregation.
- **Order never varies and the stack is fixed.** If the decorators are always the same set in the same order, a single composite type might be clearer than a chain of wrappers.

## Pros and Cons

**Pros:**

- **Open/Closed compliance** -- add new behaviors (new decorators) without modifying existing components. The core processor and existing decorators stay untouched.
- **Single Responsibility per decorator** -- each decorator does exactly one thing. Logging logs. Timing times. Validation validates. Easy to test, easy to understand, easy to remove.
- **Runtime composition** -- decorators are stacked at runtime. Enable compression for large payloads, skip it for small ones. The same core logic serves both paths.
- **Same interface throughout** -- clients don't know they're talking to a decorated object. The entire decorator chain is substitutable for the core component.

**Cons:**

- **Deep call stacks** -- ten decorators means ten layers of function calls to trace through when debugging. Stack traces get long and repetitive.
- **Order matters silently** -- logging outside timing records total time. Logging inside timing records only inner processing time. The ordering contract isn't enforced by the type system; it's up to the developer.
- **Identity confusion** -- `decorated == core` is false even though they implement the same interface. If code uses pointer identity for equality checks or caching, decoration breaks it.

## Best Practices

- **One behavior per decorator.** A `LoggingAndTimingDecorator` should be two decorators composed, not one struct doing both. This keeps each testable and removable independently.
- **Propagate errors faithfully.** If the wrapped processor returns an error, the decorator should return it (possibly wrapped with context) rather than swallowing or replacing it. The walkthrough's `LoggingDecorator` demonstrates this: it logs the error and returns it.
- **Document the intended stacking order.** If auth must happen before rate limiting, which must happen before logging, write that down. A comment at the composition site or a constructor function that enforces the order prevents misconfiguration.
- **Use function decorators for single-method interfaces.** If `DataProcessor` had only `Process`, the functional approach (closures wrapping closures) is lighter than structs. Switch to structs when the interface grows or when decorators need configuration state.

## Common Mistakes

**Decorating when modification is simpler.** If you own the core type and the behavior is always needed, adding it directly is clearer than wrapping. A decorator that's never removed is indirection without benefit. Decorators shine for *optional* or *combinatorial* behaviors, not for mandatory ones.

**Breaking the interface contract in a decorator.** A validation decorator that returns a different error type than the core processor, or a compression decorator that changes the semantics of the output (compressed bytes where the caller expects raw bytes), violates Liskov Substitution. Decorators must preserve the contract -- they add behavior *around* the call, not change what the call means.

**Forgetting that decorators compose, not configure.** If you need to pass configuration through the decorator chain (e.g., "should I compress this particular request?"), decorators are awkward -- each layer would need to inspect context or carry config. Use a different pattern (Strategy, middleware with context) for per-request configuration. Decorators are best for static, always-on layers.

**Not testing the decorator chain as a unit.** Individual decorators are easy to test in isolation. But the composed chain -- where order matters, error propagation crosses layers, and behaviors interact -- also needs integration tests. A timing decorator that logs *before* the error is returned from validation produces confusing output. Only integration tests catch ordering bugs.

## Final Thoughts

The Decorator pattern adds behavior to objects at runtime by wrapping them in layers, each implementing the same interface. The core object stays clean. Each layer adds one concern. The client sees one interface regardless of how many layers exist.

Go makes this pattern idiomatic at two levels: structs wrapping interfaces for multi-method contracts, and functions wrapping functions for single-method contracts. The standard library validates this daily -- `io.Reader` chains, `http.Handler` middleware, `gzip.Writer` wrapping `bufio.Writer` wrapping `os.File`. If you've written an HTTP middleware in Go, you've already used the Decorator pattern.

The discipline is keeping each decorator small, single-purpose, and faithful to the interface contract. Stack them intentionally, document the ordering, and test the composition. The pattern's power is in its composability -- and composability only works when each piece is clean.

> **Wrap to extend, delegate to preserve. Each layer adds one thing and trusts the rest of the chain.**
