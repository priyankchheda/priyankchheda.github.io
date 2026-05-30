+++
date = '2025-09-22'
title = 'Singleton Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a database connection pool in a web service. Opening a pool is expensive -- it involves DNS resolution, TCP handshakes, authentication, and connection negotiation. You want exactly one pool shared across all request handlers. If each handler creates its own pool, you exhaust database connections within seconds. If two goroutines race to create the "shared" pool simultaneously, you end up with two pools and the same resource problem.

The **Singleton pattern** ensures exactly one instance exists and provides a global access point to it. In Go, this is almost always implemented with `sync.Once` -- a stdlib primitive that guarantees a function runs exactly once, even when called from multiple goroutines simultaneously. It's the idiomatic answer to "initialize this expensive thing once and share it everywhere."

This is also one of the most controversial patterns in the GoF catalog. Used well, it manages shared resources cleanly. Used poorly, it becomes a global variable with extra steps -- hiding dependencies, breaking test isolation, and accumulating responsibilities until it's a god object. The pattern itself isn't the problem; the discipline around it is.

In this article, we'll build a connection pool manager as a singleton, explore the concurrency guarantees `sync.Once` provides, cover the testability problem and its solutions, and draw the line between legitimate use and the anti-patterns that give singletons a bad name.

## What Is the Singleton Pattern?

> "Ensure a class has only one instance and provide a global point of access to it."
> -- Gang of Four

The intent: **restrict instantiation to a single object and make it accessible from anywhere in the program.** The single instance lives for the program's lifetime. Every caller that asks for it gets the same object.

Two guarantees:
1. Only one instance is ever created, regardless of how many goroutines request it simultaneously.
2. There's a well-known way to access it (typically a `GetInstance()` function).

Without the pattern, sharing a connection pool looks like this:

```go
var pool *ConnectionPool

func init() {
    pool = NewConnectionPool("postgres://localhost:5432/app", 10)
}
```

This works but has problems. `init()` runs at import time -- before `main()`, before you can pass configuration, before you know if the pool is even needed. If initialization fails, you can't handle the error gracefully (panic or nothing). And the variable is exported implicitly through the package -- anyone can reassign it.

With `sync.Once`:

```go
var (
    pool *ConnectionPool
    once sync.Once
)

func GetPool() *ConnectionPool {
    once.Do(func() {
        pool = NewConnectionPool("postgres://localhost:5432/app", 10)
    })
    return pool
}
```

Lazy initialization (created on first use, not at import), thread-safe (guaranteed by `sync.Once`), and controlled access (through a function, not a raw variable).

## Core Components

The Singleton pattern is structurally the simplest creational pattern -- just three elements.

**Singleton type** -- the struct holding the shared state or resource.

```go
type ConnectionPool struct {
    dsn     string
    maxConn int
    conns   []*Connection
    mu      sync.Mutex
}
```

**Package-level instance and Once** -- the mechanism ensuring single creation.

```go
var (
    instance *ConnectionPool
    once     sync.Once
)
```

**Public accessor** -- the global access point.

```go
func GetPool() *ConnectionPool {
    once.Do(func() { ... })
    return instance
}
```

The flow:

```txt
Goroutine A calls GetPool() --> sync.Once runs initializer --> instance created
Goroutine B calls GetPool() --> sync.Once skips (already done) --> same instance returned
Goroutine C calls GetPool() --> sync.Once skips --> same instance returned
```

After the first call, `GetPool()` is effectively a pointer dereference -- `sync.Once` uses atomic operations internally and has near-zero overhead after initialization.

## Code Walkthrough: A Connection Pool Manager

We're building a database connection pool that must be initialized once and shared across all handlers in a web service.

### The Singleton Type

```go
type ConnectionPool struct {
    dsn      string
    maxConns int
    active   int
    mu       sync.Mutex
}

func newConnectionPool(dsn string, maxConns int) *ConnectionPool {
    fmt.Printf("[pool] initializing: dsn=%s max=%d\n", dsn, maxConns)
    return &ConnectionPool{
        dsn:      dsn,
        maxConns: maxConns,
    }
}

func (p *ConnectionPool) Acquire() error {
    p.mu.Lock()
    defer p.mu.Unlock()
    if p.active >= p.maxConns {
        return fmt.Errorf("pool exhausted: %d/%d connections in use", p.active, p.maxConns)
    }
    p.active++
    fmt.Printf("[pool] acquired connection (%d/%d)\n", p.active, p.maxConns)
    return nil
}

func (p *ConnectionPool) Release() {
    p.mu.Lock()
    defer p.mu.Unlock()
    if p.active > 0 {
        p.active--
    }
    fmt.Printf("[pool] released connection (%d/%d)\n", p.active, p.maxConns)
}

func (p *ConnectionPool) Stats() string {
    p.mu.Lock()
    defer p.mu.Unlock()
    return fmt.Sprintf("Pool{dsn=%s active=%d/%d}", p.dsn, p.active, p.maxConns)
}
```

The constructor `newConnectionPool` is unexported -- callers can't create their own pools. The mutex protects the `active` counter from concurrent access.

### The Singleton Accessor

```go
var (
    poolInstance *ConnectionPool
    poolOnce     sync.Once
)

func GetPool() *ConnectionPool {
    poolOnce.Do(func() {
        poolInstance = newConnectionPool("postgres://localhost:5432/app", 10)
    })
    return poolInstance
}
```

`sync.Once.Do` guarantees that even if 100 goroutines call `GetPool()` simultaneously at startup, the initializer runs exactly once. The other 99 goroutines block until initialization completes, then all receive the same instance.

### Using It

```go
func handleRequest(id int) {
    pool := GetPool()
    if err := pool.Acquire(); err != nil {
        fmt.Printf("[handler %d] %v\n", id, err)
        return
    }
    defer pool.Release()
    fmt.Printf("[handler %d] processing with %s\n", id, pool.Stats())
}

func main() {
    var wg sync.WaitGroup
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            handleRequest(id)
        }(i)
    }
    wg.Wait()
    fmt.Println("Final:", GetPool().Stats())
}
```

```txt
[pool] initializing: dsn=postgres://localhost:5432/app max=10
[pool] acquired connection (1/10)
[handler 1] processing with Pool{dsn=postgres://localhost:5432/app active=1/10}
[pool] acquired connection (2/10)
[handler 2] processing with Pool{dsn=postgres://localhost:5432/app active=2/10}
[pool] acquired connection (3/10)
[handler 3] processing with Pool{dsn=postgres://localhost:5432/app active=3/10}
[pool] released connection (2/10)
[pool] released connection (1/10)
[pool] released connection (0/10)
Final: Pool{dsn=postgres://localhost:5432/app active=0/10}
```

The initialization message appears once. Three goroutines share the same pool. The active count rises and falls as connections are acquired and released. One pool, many users.

## Making It Testable

The biggest legitimate criticism of singletons is testability. A test that calls `GetPool()` gets the production pool -- which might try to connect to a real database, can't be reset between tests, and leaks state across test cases.

The fix: **wrap the singleton behind an interface and inject it.**

```go
type Pool interface {
    Acquire() error
    Release()
    Stats() string
}
```

Production code receives the singleton through this interface:

```go
func handleRequest(pool Pool, id int) {
    if err := pool.Acquire(); err != nil {
        fmt.Printf("[handler %d] %v\n", id, err)
        return
    }
    defer pool.Release()
    fmt.Printf("[handler %d] processing\n", id)
}
```

In production:

```go
handleRequest(GetPool(), requestID)
```

In tests:

```go
type MockPool struct {
    acquireCalls int
}

func (m *MockPool) Acquire() error { m.acquireCalls++; return nil }
func (m *MockPool) Release()       {}
func (m *MockPool) Stats() string  { return "mock" }

func TestHandleRequest(t *testing.T) {
    mock := &MockPool{}
    handleRequest(mock, 1)
    if mock.acquireCalls != 1 {
        t.Errorf("expected 1 acquire call, got %d", mock.acquireCalls)
    }
}
```

The singleton still exists -- it's the production implementation of `Pool`. But client code depends on the interface, not the singleton directly. This gives you the resource-sharing benefit of Singleton without the testing pain.

## When to Use It

The Singleton pattern fits when **exactly one instance must exist for correctness, and that instance is shared across the program**:

- **Connection pools** where multiple pools would exhaust database connections. The pool manages a finite resource that must be centrally controlled.
- **Configuration loaded from external sources** (files, environment, remote config services) where loading is expensive and the result should be shared immutably.
- **Coordinated shared state** like a rate limiter or circuit breaker that must make decisions based on aggregate behavior across all goroutines, not per-goroutine state.
- **Caches** where duplication would waste memory and reduce hit rates. One shared cache is more effective than many isolated ones.

Skip it when:

- **"There should be one" is a preference, not a requirement.** If two instances would be inconvenient but not incorrect, dependency injection is simpler and more testable.
- **The object has no shared state.** A stateless utility doesn't need to be a singleton -- just call the function. Singleton adds ceremony for nothing.
- **You're using it as a dependency injection substitute.** If the only reason for the singleton is avoiding passing a parameter through three functions, pass the parameter. Explicit dependencies are clearer.

## Pros and Cons

**Pros:**

- **Guaranteed single instance** -- `sync.Once` eliminates the race condition between "check if exists" and "create if not." The guarantee is structural, not based on developer discipline.
- **Lazy initialization** -- expensive setup happens on first use, not at import time. If the singleton is never needed (certain code paths, certain configurations), the cost is never paid.
- **Centralized resource control** -- for finite resources (connections, file handles, rate limits), a single manager prevents over-allocation that multiple independent managers would cause.

**Cons:**

- **Hidden dependencies** -- when code calls `GetPool()` internally, it's not visible from the function signature. The function looks pure but depends on global state. Debugging "why did this test fail?" often leads to a singleton someone else modified.
- **Test isolation difficulty** -- the singleton persists across tests. One test's state leaks into the next. Parallel tests race on shared singleton state. The interface-wrapping approach (described above) mitigates this but adds indirection.
- **Attracts unrelated responsibilities** -- singletons tend to accumulate features. A config singleton starts holding database settings, then API keys, then feature flags, then caching rules. It becomes a god object because adding to it is easy and creating a new singleton feels heavy.

## Best Practices

- **Always use `sync.Once`, never check-then-create.** `if instance == nil { instance = ... }` is a data race in Go. It will pass `-race` detection and produce duplicate instances under load. `sync.Once` is the only correct lazy approach.
- **Keep the singleton unexported; export only the accessor.** `var instance` should be package-private. `GetPool()` is the only way in. This prevents external code from nil-ing the instance or replacing it (except through controlled test hooks).
- **Wrap behind an interface for consumer code.** Functions that *use* the singleton should accept an interface parameter, not call `GetPool()` directly. The singleton is the production implementation, but the consumer doesn't know that.
- **Limit to one responsibility.** A connection pool singleton manages connections. It doesn't also manage configuration, caching, and logging. If you need those, make them separate singletons (or better, separate injected dependencies).

## Common Mistakes

**Panicking inside `sync.Once` on initialization failure.** If `newConnectionPool` fails (bad DSN, unreachable host), panicking inside `once.Do` crashes the program on the first request. But you can't return an error from `GetPool()` after `once.Do` -- it only runs once. The solution is either: initialize eagerly during startup (where you can fail fast with a clear error), or store the error alongside the instance and check it on every access:

```go
var (
    poolInstance *ConnectionPool
    poolErr      error
    poolOnce     sync.Once
)

func GetPool() (*ConnectionPool, error) {
    poolOnce.Do(func() {
        poolInstance, poolErr = newConnectionPool(dsn, maxConns)
    })
    return poolInstance, poolErr
}
```

**Using Singleton when a scoped lifetime is more appropriate.** A per-request database transaction doesn't belong in a singleton -- it has request scope, not application scope. Singletons are for program-lifetime objects. Confusing scope leads to shared mutable state bugs where one request's transaction affects another.

**Resetting `sync.Once` in production code.** Some developers reset `once = sync.Once{}` to "reinitialize" the singleton (e.g., on config reload). This is unsafe -- other goroutines may be mid-access. For reloadable configuration, use `atomic.Value` instead of singleton-with-reset:

```go
var config atomic.Value

func LoadConfig(cfg Config) { config.Store(cfg) }
func GetConfig() Config     { return config.Load().(Config) }
```

**Making the singleton mutable without synchronization.** A config singleton where any goroutine can call `config.Set("key", "value")` without a mutex is a data race. Either make the singleton immutable after initialization (the safer choice), or protect all mutations with `sync.Mutex` or `sync.RWMutex`.

## Final Thoughts

The Singleton pattern guarantees one instance and provides global access to it. In Go, `sync.Once` makes the implementation trivial and thread-safe. The real challenge isn't implementing it -- it's using it with discipline.

The legitimate use cases are narrow: program-lifetime resources that must be shared and would break if duplicated. Connection pools, caches, rate limiters. Beyond that, dependency injection is almost always better -- it's more explicit, more testable, and doesn't hide relationships.

The best singletons are the ones you barely notice. They initialize once, serve everyone, and never accumulate responsibilities beyond their original purpose. The moment a singleton starts feeling like a convenience rather than a necessity, it's time to inject a dependency instead.

> **If you must have exactly one, make it a singleton. If you just want one, pass it as a parameter.**
