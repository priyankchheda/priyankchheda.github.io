+++
date = '2025-10-02'
title = 'Object Pool Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a high-throughput API server that handles 10,000 requests per second. Each request needs a database connection to execute a query. Opening a connection involves DNS resolution, a TCP handshake, TLS negotiation, and authentication -- easily 50-100ms of work. At 10,000 requests per second, creating and destroying a connection per request means the server spends more time on connection setup than on actual query execution. And the database, hit with 10,000 concurrent connection attempts, collapses under the load.

The **Object Pool** pattern solves this by creating a fixed set of connections upfront, handing them out to requests on demand, and recycling them when requests finish. No connection is ever created or destroyed during normal operation. The pool caps concurrency (preventing database overload), eliminates setup latency (connections are already open), and reduces garbage collection pressure (no constant allocation and deallocation).

Go makes this pattern especially natural. Buffered channels are thread-safe queues out of the box -- a buffered channel of connections *is* a pool. `Acquire` reads from the channel (blocking if empty), `Release` writes back to it. No mutexes, no lock contention, no external libraries. In this article, we'll build a connection pool from scratch using channels, add context-based timeouts, explore `sync.Pool` for a different use case, and cover the mistakes that turn a pool into a resource leak.

## What Is the Object Pool Pattern?

> "Manage a set of reusable objects that clients can borrow and return, avoiding the cost of repeated creation and destruction."

The intent: **pre-create expensive objects, lend them out on demand, and recycle them when returned.** The pool controls how many objects exist, ensures they're reused rather than recreated, and provides a bounded concurrency mechanism -- when the pool is empty, callers wait.

The lifecycle is simple: **create once, acquire, use, release, reuse.**

Without a pool, every request pays the full initialization cost:

```go
func handleRequest(query string) error {
    conn, err := openConnection("postgres://localhost:5432/app")
    if err != nil {
        return err
    }
    defer conn.Close()
    return conn.Execute(query)
}
```

At high concurrency, this creates thousands of connections, overwhelms the database, and spends most CPU time on setup/teardown rather than work.

With a pool:

```go
func handleRequest(pool *ConnPool, query string) error {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return err
    }
    defer pool.Release(conn)
    return conn.Execute(query)
}
```

The connection already exists. `Acquire` is near-instant. The database sees a bounded number of connections. The garbage collector handles fewer allocations.

## Core Components

Three participants.

**Pool** -- manages the collection of reusable objects. Handles creation, tracking, and enforcement of size limits.

**Reusable Object** -- the expensive resource being pooled. Must be safe to reuse after a reset. Database connections, network sockets, worker goroutines, memory buffers.

**Client** -- borrows objects from the pool and returns them when done. Uses `defer` to guarantee release even on panics or early returns.

The lifecycle:

```txt
Pool creates N objects at startup
  --> Client calls Acquire() --> Pool hands out an object (blocks if empty)
    --> Client uses the object
      --> Client calls Release() --> Object returns to pool for next client
```

In Go, a buffered channel models this perfectly:

```go
type Pool struct {
    resources chan *Resource
}
```

The channel capacity is the pool size. Sending to the channel returns an object. Receiving from the channel acquires one. Blocking semantics are built in.

## Code Walkthrough: A Database Connection Pool

We're building a production-style connection pool with context-based timeouts, proper error handling, and graceful shutdown.

### The Resource

```go
type Connection struct {
    id     int
    dsn    string
    active bool
}

func newConnection(id int, dsn string) *Connection {
    fmt.Printf("  [pool] creating connection %d\n", id)
    return &Connection{id: id, dsn: dsn, active: true}
}

func (c *Connection) Execute(query string) error {
    fmt.Printf("  [conn %d] executing: %s\n", c.id, query)
    return nil
}

func (c *Connection) Reset() {
    c.active = true
}

func (c *Connection) Close() {
    fmt.Printf("  [conn %d] closed\n", c.id)
    c.active = false
}
```

Each connection has an ID for tracing, a `Reset()` to prepare it for reuse, and a `Close()` for shutdown.

### The Pool

```go
type ConnPool struct {
    resources chan *Connection
    dsn       string
}

func NewConnPool(size int, dsn string) *ConnPool {
    pool := &ConnPool{
        resources: make(chan *Connection, size),
        dsn:       dsn,
    }
    for i := 1; i <= size; i++ {
        pool.resources <- newConnection(i, dsn)
    }
    return pool
}

func (p *ConnPool) Acquire(ctx context.Context) (*Connection, error) {
    select {
    case conn := <-p.resources:
        return conn, nil
    case <-ctx.Done():
        return nil, fmt.Errorf("pool: acquire timed out: %w", ctx.Err())
    }
}

func (p *ConnPool) Release(conn *Connection) {
    conn.Reset()
    p.resources <- conn
}

func (p *ConnPool) Close() {
    close(p.resources)
    for conn := range p.resources {
        conn.Close()
    }
}

func (p *ConnPool) Available() int {
    return len(p.resources)
}
```

`Acquire` uses a `select` with context -- if no connection is available before the context deadline, the caller gets an error instead of blocking forever. `Release` resets the connection before returning it. `Close` drains and closes all connections for graceful shutdown.

### Using It

```go
func main() {
    pool := NewConnPool(3, "postgres://localhost:5432/app")
    fmt.Printf("Pool created: %d connections available\n\n", pool.Available())

    var wg sync.WaitGroup
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
            defer cancel()

            conn, err := pool.Acquire(ctx)
            if err != nil {
                fmt.Printf("[client %d] %v\n", id, err)
                return
            }
            defer pool.Release(conn)

            conn.Execute(fmt.Sprintf("SELECT * FROM orders WHERE client=%d", id))
            time.Sleep(500 * time.Millisecond)
        }(i)
    }

    wg.Wait()
    fmt.Printf("\nAll done. %d connections available\n", pool.Available())
    pool.Close()
}
```

```txt
  [pool] creating connection 1
  [pool] creating connection 2
  [pool] creating connection 3
Pool created: 3 connections available

  [conn 1] executing: SELECT * FROM orders WHERE client=1
  [conn 2] executing: SELECT * FROM orders WHERE client=2
  [conn 3] executing: SELECT * FROM orders WHERE client=3
  [conn 1] executing: SELECT * FROM orders WHERE client=4
  [conn 2] executing: SELECT * FROM orders WHERE client=5

All done. 3 connections available
  [conn 1] closed
  [conn 2] closed
  [conn 3] closed
```

Three connections serve five clients. Clients 4 and 5 wait until connections 1 and 2 are released. The timeout prevents indefinite blocking -- if the pool stayed exhausted for more than 2 seconds, those clients would get an error instead of hanging.

## `sync.Pool`: The Standard Library's Lightweight Pool

Go's `sync.Pool` is often confused with the Object Pool pattern, but they solve different problems.

`sync.Pool` is for **temporary, stateless objects** where reducing allocation pressure matters but resource lifecycle doesn't. Byte buffers, JSON encoders, regex matchers -- things that are cheap to recreate but expensive to allocate at high frequency.

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func processRequest(data []byte) {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)

    copy(buf, data)
    // process...
}
```

The critical difference: **the garbage collector can clear `sync.Pool` at any time.** Objects disappear between GC cycles. There's no size limit, no blocking, no lifecycle guarantees. This makes it unsuitable for database connections or any resource that must persist and be explicitly managed.

Use `sync.Pool` for allocation optimization. Use a channel-based pool for resource management. They're different tools for different problems.

## When to Use It

The Object Pool pattern fits when **object creation is expensive, the object count must be bounded, and objects are safely reusable**:

- **Database connections** where setup involves network handshakes and authentication. Go's `database/sql` uses a pool internally for exactly this reason.
- **Network sockets** in high-throughput services where TLS handshakes are expensive and connection reuse (keep-alive) is the standard optimization.
- **Worker goroutines** in task processing systems where unbounded goroutine creation would exhaust memory or overwhelm schedulers.
- **Memory buffers** in proxies, streaming servers, or serialization-heavy services where large allocations per request create GC pressure.

Skip it when:

- **Object creation is cheap.** Small structs, value types, or objects with no external resource dependencies are faster to allocate fresh than to manage through a pool.
- **Objects can't be safely reused.** If resetting state between uses is complex, error-prone, or security-sensitive (data leaking between tenants), pooling introduces more risk than it saves.
- **Go's GC is already handling it well.** Modern Go has a highly optimized garbage collector. Profile before assuming you need a pool -- you might not.

## Pros and Cons

**Pros:**

- **Eliminates repeated initialization cost** -- connections, sockets, and buffers are created once and reused thousands of times. For resources with 50-100ms setup time, this is transformative.
- **Bounds concurrency naturally** -- a pool of 20 connections means at most 20 concurrent database operations. The pool *is* the rate limiter, preventing the downstream system from being overwhelmed.
- **Reduces GC pressure** -- fewer allocations means fewer objects for the garbage collector to track and sweep. In high-throughput systems, this translates to more predictable latency.

**Cons:**

- **Lifecycle complexity** -- the pool must handle creation, health checking, reset, timeout, and shutdown. That's more code to write, test, and maintain than simple direct allocation.
- **Resource leak risk** -- if a client acquires a connection and doesn't release it (missing `defer`, panic without recovery, goroutine leak), the pool slowly drains until it deadlocks.
- **Stale resource danger** -- a connection that's been idle for hours might be closed by the server. The pool needs health checks or max-lifetime policies to avoid handing out dead connections.

## Best Practices

- **Always use `context.Context` for acquisition.** A pool that blocks forever when empty will deadlock your service under load. Context-based timeouts let callers fail fast and return meaningful errors.
- **Reset objects before returning them to the pool.** A connection with uncommitted transaction state, a buffer with leftover data, or a worker with stale context must be cleaned before reuse. `Reset()` is part of the release contract.
- **Implement `Close()` for graceful shutdown.** When the service stops, drain the pool and close all resources. Without this, connections leak, file descriptors accumulate, and the OS eventually runs out.
- **Monitor pool utilization.** Track available vs in-use counts, acquisition wait time, and timeout frequency. A pool that's always exhausted is too small. A pool that's always full is too large (or unnecessary).

## Common Mistakes

**Forgetting `defer pool.Release(conn)` after acquire.** The number one cause of pool starvation. A function with multiple return paths where one of them doesn't release the connection slowly drains the pool. Always `defer` immediately after a successful acquire -- before any business logic that might return early or panic.

**Using `sync.Pool` for database connections.** `sync.Pool` provides no size limit, no blocking, and no persistence guarantee. The GC can clear it at any time. Database connections that vanish mid-request cause query failures. Use a channel-based pool with explicit lifecycle management for anything that must stay alive.

**Not implementing health checks on idle resources.** A connection that's been in the pool for 30 minutes might have been closed by the server due to an idle timeout. Handing it out causes the first query to fail. Either ping connections before returning them from `Acquire`, or implement max-idle-time eviction (like `database/sql`'s `SetConnMaxIdleTime`).

**Making the pool too large.** A pool of 500 connections to a database that supports 100 concurrent connections doesn't help -- it just wastes memory and confuses monitoring. Size the pool to match the downstream system's capacity, not your application's peak concurrency.

## Final Thoughts

The Object Pool pattern trades simplicity for performance: instead of creating and destroying resources per use, you maintain a fixed set and recycle them. For expensive resources with bounded capacity -- database connections, network sockets, heavy buffers -- this trade is almost always worth it.

Go makes the implementation elegant. A buffered channel is a thread-safe, blocking, fixed-size pool with zero additional infrastructure. Add context for timeouts, `defer` for guaranteed release, and a `Close()` for shutdown, and you have a production-ready pool in under 50 lines.

The discipline is lifecycle management. Every acquire needs a release. Every pool needs a shutdown path. Every idle resource needs a health check. Get those right, and the pool will serve you for millions of requests without a single resource leak.

> **Create expensive things once, lend them out on demand, and always get them back.**
