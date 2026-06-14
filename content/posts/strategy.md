+++
date = '2026-06-14'
title = 'Strategy Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a file compression tool. It needs to support multiple algorithms -- gzip for general use, zstd for speed-sensitive workloads, brotli for web assets. The algorithm is chosen at runtime based on user input, file type, or configuration. The compression pipeline (read file, compress, write output, report stats) is always the same. Only the compression step varies.

The naive approach: a switch statement inside the compression function. `case "gzip"`, `case "zstd"`, `case "brotli"`. Works fine with three algorithms. Then someone adds lz4. Then snappy. Then a custom algorithm for a specific client. The switch grows, the function does too many things, and testing any single algorithm means importing all of them.

The **Strategy pattern** extracts each algorithm into its own object behind a shared interface. The compression pipeline holds a strategy and calls it. Swap the strategy, swap the algorithm. The pipeline doesn't change. Adding a new algorithm means writing a new struct -- no switch to modify, no existing code to touch.

This is arguably the most natural pattern in Go. An interface with one method, multiple implementations, and a field or parameter that accepts it -- that's Strategy. Go developers use it constantly without naming it. In this article, we'll build a compression service with interchangeable algorithms, show the functional alternative for single-method strategies, and draw the line between Strategy and State.

## What Is the Strategy Pattern?

> "Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it."
> -- Gang of Four

The intent: **make a behavior pluggable by defining it behind an interface and letting the caller choose which implementation to use.** The context (the thing doing the work) delegates one specific responsibility to a strategy object. Different strategy objects produce different behavior. The context doesn't know or care which one it has.

Without the pattern, algorithm selection lives inside the function:

```go
func compress(data []byte, algorithm string) ([]byte, error) {
    switch algorithm {
    case "gzip":
        return gzipCompress(data)
    case "zstd":
        return zstdCompress(data)
    case "brotli":
        return brotliCompress(data)
    default:
        return nil, fmt.Errorf("unsupported: %s", algorithm)
    }
}
```

Every new algorithm means a new case. The function imports every compression library. Testing one algorithm means the test binary includes all of them. The function violates Open/Closed and Single Responsibility simultaneously.

With Strategy:

```go
func compress(data []byte, strategy Compressor) ([]byte, error) {
    return strategy.Compress(data)
}
```

One line. The caller decides which strategy. The function just uses it. Adding lz4 means writing a struct that implements `Compressor` -- nothing else changes.

## Core Components

Three participants.

**Strategy** -- the interface defining the interchangeable behavior.

```go
type Compressor interface {
    Compress(data []byte) ([]byte, error)
    Name() string
}
```

**ConcreteStrategy** -- each implementation of the algorithm.

**Context** -- the object that uses a strategy. It holds a reference to the strategy interface and delegates to it.

```go
type CompressionService struct {
    strategy Compressor
}
```

The flow:

```
Client selects strategy --> Context.Process(data) --> Strategy.Compress(data) --> result
```

The client picks the algorithm. The context uses it. The strategy does the work. Each can change independently.

## Code Walkthrough: A Compression Service

We're building a compression service that processes files using an interchangeable compression algorithm. The service handles the pipeline (reading, compressing, reporting); the strategy handles the algorithm.

### The Strategy Interface

```go
type Compressor interface {
    Compress(data []byte) ([]byte, error)
    Name() string
}
```

Two methods: compress the data and identify yourself. Every algorithm satisfies this contract.

### Concrete Strategies

**Gzip** -- general-purpose, widely supported:

```go
type GzipCompressor struct {
    level int
}

func NewGzipCompressor(level int) *GzipCompressor {
    return &GzipCompressor{level: level}
}

func (c *GzipCompressor) Compress(data []byte) ([]byte, error) {
    var buf bytes.Buffer
    w, err := gzip.NewWriterLevel(&buf, c.level)
    if err != nil {
        return nil, fmt.Errorf("gzip: %w", err)
    }
    if _, err := w.Write(data); err != nil {
        return nil, fmt.Errorf("gzip write: %w", err)
    }
    if err := w.Close(); err != nil {
        return nil, fmt.Errorf("gzip close: %w", err)
    }
    return buf.Bytes(), nil
}

func (c *GzipCompressor) Name() string {
    return fmt.Sprintf("gzip (level %d)", c.level)
}
```

**NoOp** -- passthrough for testing or when compression is disabled:

```go
type NoOpCompressor struct{}

func (c *NoOpCompressor) Compress(data []byte) ([]byte, error) {
    return data, nil
}

func (c *NoOpCompressor) Name() string {
    return "none"
}
```

**RunLength** -- a simple algorithm for demonstration:

```go
type RunLengthCompressor struct{}

func (c *RunLengthCompressor) Compress(data []byte) ([]byte, error) {
    if len(data) == 0 {
        return data, nil
    }
    var buf bytes.Buffer
    count := 1
    for i := 1; i < len(data); i++ {
        if data[i] == data[i-1] && count < 255 {
            count++
        } else {
            buf.WriteByte(byte(count))
            buf.WriteByte(data[i-1])
            count = 1
        }
    }
    buf.WriteByte(byte(count))
    buf.WriteByte(data[len(data)-1])
    return buf.Bytes(), nil
}

func (c *RunLengthCompressor) Name() string {
    return "run-length"
}
```

Three algorithms, each focused on one thing. They don't know about the service, each other, or the pipeline.

### The Context

The compression service runs the pipeline and delegates the algorithm:

```go
type CompressionService struct {
    strategy Compressor
}

func NewCompressionService(strategy Compressor) *CompressionService {
    return &CompressionService{strategy: strategy}
}

func (s *CompressionService) SetStrategy(strategy Compressor) {
    s.strategy = strategy
}

func (s *CompressionService) Process(data []byte) ([]byte, error) {
    fmt.Printf("[compress] using %s on %d bytes\n", s.strategy.Name(), len(data))

    result, err := s.strategy.Compress(data)
    if err != nil {
        return nil, fmt.Errorf("compression failed (%s): %w", s.strategy.Name(), err)
    }

    ratio := float64(len(result)) / float64(len(data)) * 100
    fmt.Printf("[compress] %d bytes --> %d bytes (%.1f%%)\n", len(data), len(result), ratio)
    return result, nil
}
```

The service logs, delegates, and reports. It doesn't know which algorithm it's using -- just that it satisfies `Compressor`. Swap the strategy, swap the behavior.

### Using It

```go
func main() {
    data := bytes.Repeat([]byte("hello world "), 1000)

    svc := NewCompressionService(NewGzipCompressor(gzip.BestSpeed))
    svc.Process(data)

    fmt.Println()

    svc.SetStrategy(&RunLengthCompressor{})
    svc.Process(data)

    fmt.Println()

    svc.SetStrategy(&NoOpCompressor{})
    svc.Process(data)
}
```

```
[compress] using gzip (level 1) on 12000 bytes
[compress] 12000 bytes --> 62 bytes (0.5%)

[compress] using run-length on 12000 bytes
[compress] 12000 bytes --> 22000 bytes (183.3%)

[compress] using none on 12000 bytes
[compress] 12000 bytes --> 12000 bytes (100.0%)
```

Same service, same pipeline, three different results. The strategy is swapped at runtime with `SetStrategy`. Adding lz4 means writing one struct -- the service and existing strategies don't change.

## Functional Strategies: The Go-Idiomatic Variation

For single-method interfaces, Go offers a lighter alternative: function types.

```go
type CompressFunc func(data []byte) ([]byte, error)

func WithGzip(level int) CompressFunc {
    return func(data []byte) ([]byte, error) {
        var buf bytes.Buffer
        w, _ := gzip.NewWriterLevel(&buf, level)
        w.Write(data)
        w.Close()
        return buf.Bytes(), nil
    }
}

func WithNoOp() CompressFunc {
    return func(data []byte) ([]byte, error) {
        return data, nil
    }
}
```

Usage:

```go
func process(data []byte, compress CompressFunc) ([]byte, error) {
    return compress(data)
}

result, _ := process(data, WithGzip(gzip.BestSpeed))
```

No interface, no struct, no method -- just a function value. This is idiomatic for strategies that are stateless or carry minimal configuration. The struct-based approach is better when strategies need multiple methods (like `Name()` in our walkthrough) or carry mutable state.

Go's standard library uses this pattern extensively. `sort.Slice` accepts a comparison function. `http.HandleFunc` accepts a handler function. `strings.Map` accepts a character mapping function. All are functional strategies.

## Strategy vs State

These patterns are structurally identical -- a context delegates to an interface -- but differ in a fundamental way.

**Strategy** is chosen *externally* and stays constant during use. The client picks the algorithm (gzip, zstd, brotli) and the service uses it for the entire operation. Strategies don't transition to other strategies.

**State** transitions *internally* as the object moves through its lifecycle. An order moves from pending to paid to shipped. States know about and trigger transitions to other states. The object's behavior evolves over time.

The test: does the pluggable object *change itself* during the context's lifecycle? If it transitions to another implementation after certain actions, it's State. If it stays put until the client explicitly replaces it, it's Strategy.

## When to Use It

The Strategy pattern fits when **you have multiple algorithms for the same task and the choice should be configurable, testable, and extensible**:

- **Algorithm selection at runtime** -- compression format, pricing model, sorting order, retry policy. The user or config determines which algorithm runs.
- **Eliminating conditional logic** -- when a switch on type/mode/flag keeps growing, each case is a strategy waiting to be extracted.
- **Testing with simulated behavior** -- a `NoOpCompressor` or `MockPaymentGateway` strategy lets you test the pipeline without real compression or real payments.
- **Plugin-style extensibility** -- third parties can write new strategies without modifying your codebase. They implement the interface; your system uses it.

Skip it when:

- **There's only one algorithm and it won't change.** If you'll only ever gzip, just call gzip. The pattern's value is in *interchangeability* -- without multiple options, it's unnecessary indirection.
- **The "strategies" share significant logic.** If 80% of the code is the same across all variants with minor tweaks, a Template Method or simple parameterization is cleaner than duplicating the shared code across strategy structs.
- **The decision is trivial and local.** A one-line `if` that picks between two formulas doesn't need a strategy interface. Extract to a pattern when the conditional appears in multiple places or the algorithm set grows.

## Pros and Cons

**Pros:**

- **Open/Closed compliance** -- new algorithms are new structs implementing the interface. Existing code stays untouched.
- **Isolated testing** -- each strategy is testable independently of the context. The context is testable with a mock strategy. No integration test needed to verify a single algorithm.
- **Runtime swappability** -- change behavior without restarting, recompiling, or restructuring. Feature flags, A/B tests, and user preferences map directly to strategy selection.
- **Clean separation** -- the context handles orchestration (logging, error handling, reporting). The strategy handles computation. Neither bleeds into the other.

**Cons:**

- **Client must know the strategies.** Someone has to decide which strategy to use. If the decision logic is complex, you've moved the switch from the context to the client (or to a factory, which adds another layer).
- **Interface overhead for trivial cases.** A strategy interface with one method for a problem with two known algorithms adds a type, a constructor, and indirection where a simple function parameter would suffice.
- **Strategies can't easily share state with the context.** If the algorithm needs deep access to the context's internals, the interface becomes either too wide (many parameters) or the strategy becomes tightly coupled to the context's structure.

## Best Practices

- **Use function types for single-method strategies.** If the interface has one method and strategies are stateless, `type CompressFunc func([]byte) ([]byte, error)` is lighter and more idiomatic than a struct with one method. Graduate to structs when you need state or multiple methods.
- **Inject strategies through constructors, not setters.** `NewCompressionService(strategy)` makes the dependency explicit and prevents a nil strategy between construction and first use. Add `SetStrategy` only if runtime swapping is a genuine requirement.
- **Name strategies descriptively.** A `Name() string` or `String()` method on the strategy helps with logging and debugging. When something goes wrong, "gzip (level 1) failed" is more useful than "compression failed."
- **Consider a strategy registry for user-facing selection.** If users pick strategies by name (from config, CLI flags, or API parameters), a `map[string]Compressor` registry converts names to implementations cleanly.

## Common Mistakes

**Creating a strategy interface too early.** If you have one algorithm today and "might add more later," the interface adds ceremony for speculation. Start with a concrete implementation. When the second algorithm actually arrives, *then* extract the interface. It's a 5-minute refactor in Go (rename the type, extract the methods into an interface). Premature abstraction costs more than reactive refactoring.

**Strategies that depend on context internals.** A `PricingStrategy` that needs the customer's purchase history, loyalty tier, seasonal discounts, and regional tax rates to compute a price is too coupled to the context. Either pass a focused data struct to the strategy, or reconsider whether this is truly "interchangeable algorithms" versus just "complex business logic that belongs in the context."

**Using Strategy when a simple parameter would suffice.** If the "strategies" are `multiply by 0.8`, `multiply by 0.9`, and `multiply by 1.0`, you don't need three structs -- you need a discount percentage parameter. Strategy is for *different logic*, not *different values applied to the same logic*.

**Not providing a sensible default.** If the context requires a strategy and most callers want the same one, force them all to choose anyway creates friction. Provide a default (`NewCompressionService()` uses gzip by default) and let callers override when they need something specific.

## Final Thoughts

The Strategy pattern extracts interchangeable algorithms into separate objects behind a shared interface. The context delegates without knowing which algorithm it's using. New algorithms slot in without modifying existing code. The switch statement disappears, replaced by a field that holds the right implementation.

Go makes this pattern feel invisible because it *is* the language's default approach to polymorphism. An interface, multiple implementations, and a field or parameter that accepts it -- that's how Go developers write code every day. Naming it "Strategy" just recognizes the structural intent: this field exists to make behavior pluggable.

The discipline is knowing when the pattern earns its keep (multiple algorithms, runtime selection, independent testing) and when it's over-engineering (one algorithm, trivial logic, values not logic). When in doubt, start concrete and extract the interface when the second implementation arrives.

> **Define the what, let the caller choose the how. One interface, many algorithms, zero conditionals.**
