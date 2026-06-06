+++
date = '2026-03-21'
title = 'Flyweight Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about a log aggregation system processing millions of log entries per minute. Each entry has a severity level (INFO, WARN, ERROR, DEBUG), a source service name, a timestamp, and a message. Naively, every log entry stores its own copy of the severity string and source service name. With 10 million entries from 50 services across 4 severity levels, that's 10 million string allocations for data that has at most 200 unique combinations (50 services x 4 levels). The rest is redundant copies consuming gigabytes of memory.

The **Flyweight pattern** solves this by separating shared state from per-instance state. The severity level and service configuration (formatting rules, output destinations) are shared -- one object per unique combination, reused across millions of entries. The timestamp and message are unique per entry and stored separately. Instead of 10 million full objects, you have 200 shared flyweights and 10 million lightweight entries that reference them.

Go's pointer semantics and map-based caching make this pattern straightforward to implement. In this article, we'll build a log processing system that uses flyweights to handle millions of entries with bounded memory, explore the intrinsic/extrinsic state distinction, and show how the factory ensures sharing.

## What Is the Flyweight Pattern?

> "Use sharing to support large numbers of fine-grained objects efficiently."
> -- Gang of Four

The intent: **reduce memory consumption by sharing common state across many objects instead of duplicating it in each one.** The key insight is splitting object state into two categories: **intrinsic state** (shared, immutable, stored in the flyweight) and **extrinsic state** (unique per instance, passed in at runtime, stored externally).

Without this separation, a system with millions of similar objects stores redundant data millions of times:

```go
type LogEntry struct {
    Severity    string
    Service     string
    Format      string
    Destination string
    Timestamp   time.Time
    Message     string
}
```

If 5 million entries are INFO-level from the "auth-service," all 5 million store identical copies of `"INFO"`, `"auth-service"`, the format string, and the destination. That's repeated memory allocation for data that never changes between instances.

With Flyweight, the shared data lives in one object:

```go
type LogConfig struct {
    Severity    string
    Service     string
    Format      string
    Destination string
}

type LogEntry struct {
    Config    *LogConfig
    Timestamp time.Time
    Message   string
}
```

Five million entries share one `*LogConfig` pointer. The memory savings are dramatic.

## Core Components

Four participants.

**Flyweight** -- the shared object containing intrinsic state. Must be immutable after creation.

```go
type LogConfig struct {
    Severity    string
    Service     string
    Format      string
    Destination string
}
```

**FlyweightFactory** -- creates and caches flyweights. Ensures only one instance exists per unique intrinsic state combination.

```go
type ConfigFactory struct {
    configs map[string]*LogConfig
}
```

**Client** -- holds extrinsic state and references a flyweight for the shared parts.

**Context** -- the extrinsic state that varies per instance (timestamp, message, request ID).

The flow:

```txt
Client requests flyweight --> Factory checks cache
  --> exists? return it
  --> doesn't exist? create, cache, return
Client stores flyweight reference + extrinsic state
```

Millions of clients share a handful of flyweights. The factory is the gatekeeper that prevents duplicate creation.

## Code Walkthrough: A Log Processing System

We're building a log processor that handles millions of entries. Each entry references a shared configuration (severity + service + formatting), and carries its own unique timestamp and message.

### The Flyweight: LogConfig

The shared, immutable state:

```go
type LogConfig struct {
    Severity    string
    Service     string
    Format      string
    Destination string
}

func (c *LogConfig) FormatEntry(ts time.Time, msg string) string {
    return fmt.Sprintf(c.Format, c.Severity, c.Service, ts.Format("15:04:05"), msg)
}
```

Each `LogConfig` knows how to format an entry using its intrinsic state plus extrinsic data (timestamp and message) passed in. The flyweight *uses* extrinsic state but doesn't *store* it.

### The Factory: ConfigFactory

Creates and caches configurations:

```go
type ConfigFactory struct {
    mu      sync.RWMutex
    configs map[string]*LogConfig
}

func NewConfigFactory() *ConfigFactory {
    return &ConfigFactory{
        configs: make(map[string]*LogConfig),
    }
}

func (f *ConfigFactory) GetConfig(severity, service string) *LogConfig {
    key := severity + ":" + service

    f.mu.RLock()
    if cfg, ok := f.configs[key]; ok {
        f.mu.RUnlock()
        return cfg
    }
    f.mu.RUnlock()

    f.mu.Lock()
    defer f.mu.Unlock()

    if cfg, ok := f.configs[key]; ok {
        return cfg
    }

    cfg := &LogConfig{
        Severity:    severity,
        Service:     service,
        Format:      "[%s] %s %s: %s",
        Destination: destinationFor(severity),
    }
    f.configs[key] = cfg
    return cfg
}

func destinationFor(severity string) string {
    if severity == "ERROR" || severity == "FATAL" {
        return "stderr"
    }
    return "stdout"
}

func (f *ConfigFactory) Stats() int {
    f.mu.RLock()
    defer f.mu.RUnlock()
    return len(f.configs)
}
```

The factory uses double-checked locking with `sync.RWMutex` -- read lock for the common case (config already exists), write lock only for creation. The key is a combination of severity and service, ensuring one flyweight per unique pair.

### The Context: LogEntry

The per-instance data that references the shared flyweight:

```go
type LogEntry struct {
    Config    *LogConfig
    Timestamp time.Time
    Message   string
}

func (e *LogEntry) String() string {
    return e.Config.FormatEntry(e.Timestamp, e.Message)
}
```

Each entry is lightweight: one pointer (8 bytes), one `time.Time` (24 bytes), and one string (16 bytes for the header). Compare that to storing the full config inline -- the savings multiply across millions of entries.

### The Log Processor

Ties everything together:

```go
type LogProcessor struct {
    factory *ConfigFactory
    entries []LogEntry
}

func NewLogProcessor() *LogProcessor {
    return &LogProcessor{
        factory: NewConfigFactory(),
    }
}

func (p *LogProcessor) Log(severity, service, message string) {
    cfg := p.factory.GetConfig(severity, service)
    p.entries = append(p.entries, LogEntry{
        Config:    cfg,
        Timestamp: time.Now(),
        Message:   message,
    })
}

func (p *LogProcessor) Flush() {
    for _, entry := range p.entries {
        fmt.Println(entry.String())
    }
    p.entries = p.entries[:0]
}

func (p *LogProcessor) Stats() {
    unique := p.factory.Stats()
    fmt.Printf("Entries: %d, Shared configs: %d\n", len(p.entries), unique)
}
```

### Using It

```go
func main() {
    processor := NewLogProcessor()

    processor.Log("INFO", "auth-service", "user login successful")
    processor.Log("INFO", "auth-service", "token refreshed")
    processor.Log("ERROR", "auth-service", "invalid credentials")
    processor.Log("INFO", "payment-service", "charge completed")
    processor.Log("WARN", "payment-service", "retry attempt 2")
    processor.Log("ERROR", "payment-service", "gateway timeout")
    processor.Log("INFO", "auth-service", "user logout")

    processor.Stats()
    fmt.Println()
    processor.Flush()
}
```

```txt
Entries: 7, Shared configs: 5

[INFO] auth-service 14:23:01: user login successful
[INFO] auth-service 14:23:01: token refreshed
[ERROR] auth-service 14:23:01: invalid credentials
[INFO] payment-service 14:23:01: charge completed
[WARN] payment-service 14:23:01: retry attempt 2
[ERROR] payment-service 14:23:01: gateway timeout
[INFO] auth-service 14:23:01: user logout
```

Seven entries, but only five unique configs. The three `INFO/auth-service` entries share one `*LogConfig`. At scale -- millions of entries from dozens of services -- the ratio becomes dramatic: millions of entries sharing hundreds of configs.

### Memory Impact at Scale

To illustrate the savings concretely:

Without flyweight: 1,000,000 entries x ~200 bytes each (full config inline) = ~200MB.

With flyweight: 1,000,000 entries x ~48 bytes (pointer + timestamp + message header) + 100 unique configs x ~200 bytes = ~48MB + negligible shared state.

That's a 4x reduction from sharing alone. In practice, with long format strings and destination metadata, the savings can be 10x or more.

## Intrinsic vs Extrinsic: The Key Distinction

The entire pattern hinges on correctly identifying which state is **intrinsic** (shared) and which is **extrinsic** (per-instance).

**Intrinsic state** is:
- The same across many objects
- Immutable after creation
- Context-independent (doesn't change based on who's using it)

In our example: severity level, service name, format string, destination. These are determined by the *type* of log, not by the specific entry.

**Extrinsic state** is:
- Unique per instance
- Changes with each use
- Context-dependent

In our example: timestamp, message content. These are different for every single log entry.

The rule: **if you can move it out of the object and pass it in at call time, it's extrinsic.** If it must stay inside for the object to function as a shared resource, it's intrinsic.

Getting this split wrong undermines the pattern entirely. If you include a timestamp in the flyweight, it can't be shared (every entry has a different time). If you exclude the severity from the flyweight and pass it every time, you lose the sharing benefit.

## When to Use It

The Flyweight pattern fits when **you have a large number of objects that share significant amounts of common state**:

- **Text rendering** where millions of characters share glyph data (font metrics, vector paths) and only position/color vary per character.
- **Game entities** where thousands of enemies, particles, or projectiles share type data (sprites, stats, behavior) and only position/health vary per instance.
- **Log/event processing** where millions of entries share configuration per source/severity but carry unique timestamps and payloads.
- **Caching and interning** where string or object deduplication eliminates redundant copies of identical data (Go's `string` interning is a form of this).

Skip it when:

- **You don't have many objects.** If there are hundreds, not millions, the memory savings don't justify the added complexity.
- **Most state is extrinsic.** If every object is almost entirely unique with very little shared data, the flyweight is mostly empty and the pattern adds overhead without meaningful savings.
- **Objects need mutable shared state.** Flyweights must be immutable. If the "shared" state changes, it can't be shared safely without complex synchronization.

## Pros and Cons

**Pros:**

- **Dramatic memory reduction** -- when thousands or millions of objects share the same intrinsic data, storing it once instead of per-instance saves orders of magnitude of memory.
- **Improved cache locality** -- fewer unique objects means the CPU cache holds more useful data. Hot flyweights stay in cache because they're accessed frequently.
- **Reduced GC pressure** -- fewer allocations means less work for Go's garbage collector. Shared objects have long lifetimes and don't contribute to GC churn.

**Cons:**

- **Complexity of state separation** -- identifying what's intrinsic vs extrinsic requires careful analysis. A wrong split means either broken sharing (mutable intrinsic state) or lost savings (too much extrinsic).
- **Trading memory for CPU** -- looking up flyweights in a map, computing keys, and passing extrinsic state at every call adds CPU overhead. The pattern only wins when memory is the bottleneck.
- **Immutability requirement** -- flyweights must be immutable to be safely shared. This constraints the design and can make certain updates awkward (you need a new flyweight instead of modifying the existing one).

## Best Practices

- **Make flyweights immutable.** Once created and cached, a flyweight must never change. If it did, all objects sharing it would be affected simultaneously. Use unexported fields and no setter methods.
- **Use a factory with sync for concurrent access.** Multiple goroutines requesting flyweights need thread-safe access to the cache. `sync.RWMutex` with double-checked locking (as shown in the walkthrough) avoids unnecessary write locks.
- **Choose the cache key carefully.** The key must uniquely identify the intrinsic state combination. If two different configurations hash to the same key, they'll incorrectly share a flyweight. Composite keys (`severity + ":" + service`) work but must cover all intrinsic fields.
- **Profile before applying.** The pattern adds complexity (factory, key computation, indirection). Verify that memory is actually the bottleneck. Go's `runtime.MemStats` or `pprof` heap profiles will tell you whether object duplication is the problem.

## Common Mistakes

**Including extrinsic state in the flyweight.** If the flyweight contains a timestamp, request ID, or any per-instance data, it can't be shared. Two entries with different timestamps would need different flyweights, defeating the purpose entirely. Audit every field: "Is this the same for all users of this flyweight?" If no, it's extrinsic and must be passed in.

**Mutating flyweights after creation.** If any code modifies a flyweight's field after it's cached, every object sharing that flyweight sees the change. This creates action-at-a-distance bugs that are extremely hard to diagnose. The fix: make flyweight fields unexported and provide no mutators. If the configuration needs to change, create a new flyweight with the new state -- don't modify the existing one.

**Not measuring the actual savings.** A flyweight pattern applied to a system with 100 objects that each save 50 bytes of duplication saves 5KB. The factory, the map, and the extra pointer per object might cost more than that in overhead. Always benchmark. The pattern pays off at scale (thousands to millions of shared objects), not at small counts.

**Using string concatenation for complex keys in hot paths.** `key := severity + ":" + service` allocates a new string on every `GetConfig` call. For a log system processing millions of entries per second, this adds GC pressure. Consider pre-computing keys, using a struct key with a map, or using `strings.Builder` for complex keys.

## Final Thoughts

The Flyweight pattern reduces memory consumption by sharing immutable state across many objects. Split state into intrinsic (shared, stored once) and extrinsic (unique, stored per instance), cache the shared parts in a factory, and let millions of lightweight objects reference the shared data through pointers.

Go makes this clean: pointers are cheap, maps provide natural caches, and `sync.RWMutex` handles concurrent access. The pattern doesn't require frameworks or reflection -- just a struct, a map, and discipline about immutability.

The honest test: if you have thousands of similar objects and profiling shows memory is the bottleneck, Flyweight can reduce consumption by an order of magnitude. If you have hundreds of objects or memory isn't constrained, the added complexity doesn't pay off. Profile first, optimize second.

> **Share what's common, store what's unique, and never mutate what's shared.**
