+++
date = '2026-04-02'
title = 'Iterator Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a service that fetches audit logs from a third-party API. The API returns results page by page -- 100 records per request, with a cursor token for the next page. The consumer of this data just wants to process logs one at a time. It doesn't care about pages, cursor tokens, or HTTP requests. It wants a loop: give me the next log entry, and tell me when there are no more.

Without an abstraction layer, every consumer of this API ends up reimplementing pagination logic. Manage the cursor. Buffer the current page. Fetch the next page when the buffer runs out. Handle the final page. That logic gets copied into every call site, and when the API changes its pagination scheme, every copy breaks.

The **Iterator pattern** solves this by separating *traversal* from *storage*. The collection (or API, or database, or file) knows how its data is organized. The iterator knows how to walk through it one element at a time. The client only asks two questions: "Is there more?" and "Give me the next one."

Go developers already use this pattern daily -- `sql.Rows`, `bufio.Scanner`, and `json.Decoder` all follow it. In this article, we'll build a paginated API iterator from scratch, layer on filtering and limiting, and connect it back to the patterns already baked into Go's standard library.

## What Is the Iterator Pattern?

> "Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation."
> -- Gang of Four

The core idea: **the collection stores data, the iterator walks through it, and the client never sees the internals.**

This matters when the "collection" isn't a simple slice. A paginated API, a database cursor, a tree, a stream of events from a message queue -- none of these map cleanly to `for range`. The traversal has state: a cursor position, a page token, a stack for depth-first search, a buffer of prefetched records. The Iterator pattern gives that state a home -- an object whose only job is moving through data one element at a time.

Without it, client code tends to look like this:

```go
func processLogs(client *AuditClient) error {
    cursor := ""
    for {
        page, nextCursor, err := client.FetchPage(cursor, 100)
        if err != nil {
            return err
        }
        for _, entry := range page {
            handle(entry)
        }
        if nextCursor == "" {
            break
        }
        cursor = nextCursor
    }
    return nil
}
```

Pagination logic, buffering, and cursor management are tangled with the actual processing. Every function that needs audit logs reimplements this. Change the page size, switch to offset-based pagination, add rate limiting -- every consumer changes.

With an iterator, the client becomes:

```go
it := client.LogIterator()
for it.Next() {
    handle(it.Value())
}
if err := it.Err(); err != nil {
    return err
}
```

The pagination complexity is hidden. The client asks for the next element, and the iterator handles everything else.

## Core Components

Four participants, though in Go they're often less formal than the textbook version.

**Iterator** -- the interface. Defines how clients traverse elements.

```go
type Iterator[T any] interface {
    Next() bool
    Value() T
    Err() error
}
```

This follows the same shape as `sql.Rows` and `bufio.Scanner`: `Next()` advances and returns whether there's more, `Value()` (or `Text()`, `Scan()`) returns the current element, and `Err()` surfaces any error that stopped iteration.

**ConcreteIterator** -- the actual implementation. Holds traversal state: the cursor, the buffer, the current index. Knows how to fetch the next batch of data.

**Aggregate** -- the collection or data source. Creates iterators.

**ConcreteAggregate** -- the actual collection. In our case, the API client that knows how to fetch pages.

The flow:

```
Client --> Iterator.Next() / Iterator.Value()
              |
              v
        ConcreteIterator (manages cursor, buffer, fetching)
              |
              v
        Data source (API, database, file, etc.)
```

The client never touches the data source directly. The iterator is the boundary.

## Code Walkthrough: A Paginated API Iterator

We're building an iterator over a paginated audit log API. The API returns pages of log entries with a cursor for the next page. The iterator hides all of that and presents a simple one-at-a-time interface.

### The Domain Type

```go
type AuditEntry struct {
    ID        string
    Action    string
    Timestamp time.Time
}
```

### The API Client (Data Source)

A simplified API client that returns pages of entries with a string cursor token -- the same shape as most real paginated APIs:

```go
type AuditAPI struct {
    entries  []AuditEntry
    pageSize int
    failAt   string
}

func (a *AuditAPI) FetchPage(cursor string) ([]AuditEntry, string, error) {
    if a.failAt != "" && cursor == a.failAt {
        return nil, "", fmt.Errorf("API error: rate limited")
    }

    start := 0
    if cursor != "" {
        for i, e := range a.entries {
            if e.ID == cursor {
                start = i
                break
            }
        }
    }

    end := start + a.pageSize
    if end > len(a.entries) {
        end = len(a.entries)
    }

    nextCursor := ""
    if end < len(a.entries) {
        nextCursor = a.entries[end].ID
    }

    return a.entries[start:end], nextCursor, nil
}
```

The cursor is an entry ID, not an integer index. The `failAt` field lets us simulate API failures mid-traversal. In production this would be an HTTP call with a token parameter; the in-memory version follows the same contract.

### The Iterator Interface

```go
type Iterator[T any] interface {
    Next() bool
    Value() T
    Err() error
}
```

Three methods: advance, access, and error check. This is the contract clients depend on.

### The Concrete Iterator

Here's where the pagination logic lives:

```go
type auditIterator struct {
    api        *AuditAPI
    buffer     []AuditEntry
    bufIndex   int
    nextCursor string
    hasMore    bool
    current    AuditEntry
    err        error
}

func NewAuditIterator(api *AuditAPI) Iterator[AuditEntry] {
    return &auditIterator{
        api:     api,
        hasMore: true,
    }
}

func (it *auditIterator) Next() bool {
    if it.err != nil {
        return false
    }

    if it.bufIndex < len(it.buffer) {
        it.current = it.buffer[it.bufIndex]
        it.bufIndex++
        return true
    }

    if !it.hasMore {
        return false
    }

    page, next, err := it.api.FetchPage(it.nextCursor)
    if err != nil {
        it.err = err
        return false
    }

    it.buffer = page
    it.bufIndex = 0
    it.nextCursor = next
    it.hasMore = next != ""

    if len(it.buffer) == 0 {
        return false
    }

    it.current = it.buffer[it.bufIndex]
    it.bufIndex++
    return true
}

func (it *auditIterator) Value() AuditEntry {
    return it.current
}

func (it *auditIterator) Err() error {
    return it.err
}
```

The complexity is real but contained. The buffer holds the current page. When it's exhausted, `Next()` fetches the next page using the cursor token. When the API returns an empty next cursor, iteration ends. If the fetch fails, `Next()` stores the error and returns `false`.

This is the `sql.Rows` pattern: iterate until `Next()` returns false, then check `Err()` to distinguish "finished successfully" from "stopped because something broke." The error doesn't surface inside the loop -- it surfaces *after*, giving the client a clean separation between processing and error handling.

### Using It

```go
func main() {
    api := &AuditAPI{
        entries: []AuditEntry{
            {ID: "1", Action: "login", Timestamp: time.Now()},
            {ID: "2", Action: "update", Timestamp: time.Now()},
            {ID: "3", Action: "delete", Timestamp: time.Now()},
            {ID: "4", Action: "login", Timestamp: time.Now()},
            {ID: "5", Action: "logout", Timestamp: time.Now()},
        },
        pageSize: 2,
    }

    it := NewAuditIterator(api)
    for it.Next() {
        entry := it.Value()
        fmt.Printf("[%s] %s\n", entry.ID, entry.Action)
    }
    if err := it.Err(); err != nil {
        fmt.Println("error:", err)
    }
}
```

```
[1] login
[2] update
[3] delete
[4] login
[5] logout
```

Five entries, page size of 2. The iterator fetched three pages internally (2 + 2 + 1). The client saw five calls to `Next()` and five calls to `Value()`. No cursor management, no page tracking, no buffer logic.

### Handling Failure Mid-Traversal

Set the API to fail when fetching the page starting at entry "3":

```go
api.failAt = "3"

it := NewAuditIterator(api)
for it.Next() {
    fmt.Printf("[%s] %s\n", it.Value().ID, it.Value().Action)
}
if err := it.Err(); err != nil {
    fmt.Println("error:", err)
}
```

```
[1] login
[2] update
error: API error: rate limited
```

The iterator delivered the first page (entries 1 and 2), then hit an error fetching the second page. `Next()` returned `false`, and `Err()` tells the client why. The client's loop didn't change -- the same code handles both success and failure. This is the `sql.Rows` contract in action: consume the loop, then check for errors.

Switch the backing store to a database cursor, a different API, or a file reader? Write a new iterator. The client loop above stays identical.

## Decorator Iterators: Filtering and Limiting

Once you have an iterator interface, you can wrap iterators with other iterators. This is the Decorator pattern applied to traversal, and it's one of the most practical extensions.

### A Filtering Iterator

Wraps any iterator and skips elements that don't match a predicate:

```go
type filterIterator[T any] struct {
    inner     Iterator[T]
    predicate func(T) bool
    current   T
}

func Filter[T any](it Iterator[T], pred func(T) bool) Iterator[T] {
    return &filterIterator[T]{inner: it, predicate: pred}
}

func (f *filterIterator[T]) Next() bool {
    for f.inner.Next() {
        if f.predicate(f.inner.Value()) {
            f.current = f.inner.Value()
            return true
        }
    }
    return false
}

func (f *filterIterator[T]) Value() T { return f.current }
func (f *filterIterator[T]) Err() error { return f.inner.Err() }
```

### A Limiting Iterator

Wraps any iterator and stops after N elements:

```go
type limitIterator[T any] struct {
    inner   Iterator[T]
    limit   int
    count   int
}

func Limit[T any](it Iterator[T], n int) Iterator[T] {
    return &limitIterator[T]{inner: it, limit: n}
}

func (l *limitIterator[T]) Next() bool {
    if l.count >= l.limit {
        return false
    }
    if l.inner.Next() {
        l.count++
        return true
    }
    return false
}

func (l *limitIterator[T]) Value() T  { return l.inner.Value() }
func (l *limitIterator[T]) Err() error { return l.inner.Err() }
```

### Composing Them

```go
it := NewAuditIterator(api)
it = Filter[AuditEntry](it, func(e AuditEntry) bool {
    return e.Action == "login"
})
it = Limit[AuditEntry](it, 10)

for it.Next() {
    fmt.Println(it.Value())
}
```

Read the first 10 login events from a paginated API. The API might have thousands of entries across dozens of pages, but the composed iterator stops fetching the moment it has 10 matches. Lazy evaluation is built in -- the paginated iterator only fetches pages as the filter requests more elements.

This composability is the real payoff of the pattern. Each decorator is small, testable, and reusable. They snap together in any combination.

## The Pattern in Go's Standard Library

If this interface shape looks familiar, it should. Go's standard library uses the same pattern extensively, just without calling it "Iterator."

**`database/sql.Rows`** -- a cursor over query results:

```go
rows, _ := db.Query("SELECT id, action FROM audit_log")
for rows.Next() {
    var id, action string
    rows.Scan(&id, &action)
}
if err := rows.Err(); err != nil { ... }
```

**`bufio.Scanner`** -- a line-by-line file reader:

```go
scanner := bufio.NewScanner(file)
for scanner.Scan() {
    fmt.Println(scanner.Text())
}
if err := scanner.Err(); err != nil { ... }
```

**`json.Decoder`** -- a streaming JSON parser:

```go
dec := json.NewDecoder(reader)
for dec.More() {
    var v MyType
    dec.Decode(&v)
}
```

All three follow the same shape: advance, access, check error. The data source varies wildly -- a database connection, a file handle, a network stream -- but the client code is nearly identical. That's the Iterator pattern doing its job.

## When to Use It

The Iterator pattern earns its place when **traversal itself is a design problem**:

- **Paginated or cursor-based data sources** where fetching the next batch is an operation with real cost. The iterator hides pagination, buffering, and cursor management.
- **Complex data structures** like trees, graphs, or nested collections where "the next element" isn't obvious. Different iterators can provide different traversal orders (DFS, BFS, in-order) over the same structure.
- **Streaming or lazy evaluation** where you can't load everything into memory. The iterator fetches or computes elements on demand.
- **Composable processing pipelines** where you want to chain filter, map, and limit operations without materializing intermediate results.

Skip it when:

- **You have a slice, map, or channel.** `for range` is simpler, more readable, and idiomatic. Wrapping a slice in an iterator adds ceremony for zero benefit.
- **Traversal is straightforward and unlikely to change.** If the data structure is a slice today and will be a slice next year, the abstraction isn't paying for itself.
- **There's only one consumer.** If traversal logic exists in exactly one place, a plain loop is clearer than an iterator.

## Pros and Cons

**Pros:**

- **Encapsulation** -- clients are shielded from the collection's internal structure. Switch from a slice to a database cursor, and client code doesn't change.
- **Composability** -- decorator iterators (filter, limit, map) snap together to form processing pipelines without materializing intermediate results.
- **Multiple concurrent traversals** -- each iterator instance carries its own state. Two goroutines can iterate over the same collection independently without coordination.
- **Lazy evaluation** -- elements are produced on demand. For large or infinite data sources, this means bounded memory usage regardless of total size.

**Cons:**

- **Boilerplate** -- even a simple iterator needs a struct, three methods, and a constructor. For trivial collections, this is more code than a `for range` loop.
- **Error handling design burden** -- where does the error surface? Inside `Next()`? A separate `Err()` method? The wrong choice leads to either swallowed errors or awkward APIs.
- **Debugging opacity** -- when a decorator chain misbehaves, the bug might be in the filter predicate, the limit count, or the underlying iterator. The indirection makes tracing harder than a flat loop.

## Best Practices

- **Follow the `sql.Rows` shape: `Next() bool`, `Value() T`, `Err() error`.** This is the most established iterator contract in Go. Developers recognize it instantly, and it handles the error case cleanly -- iterate until `Next()` returns false, then check `Err()` to distinguish completion from failure.
- **Make iterators safe to abandon.** Clients won't always consume every element. If your iterator holds resources (open files, database connections, HTTP responses), implement `io.Closer` so the client can release them early. `sql.Rows` does this.
- **Keep the iterator's constructor the only public entry point.** Export the `Iterator` interface, but keep the concrete iterator type unexported. Clients should receive iterators from a factory method on the collection, not construct them directly.
- **Don't modify the underlying collection during iteration.** An iterator that holds a pointer to a slice will produce undefined behavior if elements are added or removed mid-traversal. Document this contract, or snapshot the data at iterator creation time if mutation is expected.

## Common Mistakes

**Building an iterator when `for range` would do.** If the data is already a `[]T` and traversal is straightforward, wrapping it in an iterator adds a struct, three methods, and a constructor -- all to replicate what `for range` gives you for free. The pattern is for when traversal *itself* is complex, not for abstracting simple loops.

**Forgetting `Err()`.** An iterator over a paginated API or database cursor can fail mid-traversal. If `Next()` returns `false`, the client needs to know whether iteration completed successfully or was aborted by an error. Omitting `Err()` from the interface means errors get silently swallowed or require panicking inside `Next()`.

**Returning the same pointer from `Value()`.** If the iterator reuses a single struct to avoid allocations (a common optimization in database drivers), every call to `Value()` returns the same pointer. Clients that store the value and read it later get the wrong data. Either document the behavior clearly or return copies.

**Making `Value()` callable before `Next()`.** If the client calls `Value()` without calling `Next()` first, the result is a zero value -- or worse, a panic. Go convention (following `sql.Rows` and `bufio.Scanner`) is that `Next()` must be called before the first access and before each subsequent one. Make sure your iterator's zero state handles this gracefully rather than panicking.

## Final Thoughts

The Iterator pattern separates what a collection holds from how clients walk through it. That separation becomes valuable the moment traversal involves state -- cursors, page tokens, tree stacks, prefetch buffers. For simple slices, `for range` is the right tool and always will be. But for paginated APIs, database cursors, streaming data, and composite processing pipelines, the iterator gives clients a clean, uniform interface while the complexity stays hidden where it belongs.

Go already embraces this pattern in its standard library. `sql.Rows`, `bufio.Scanner`, and `json.Decoder` all follow the same shape: advance, access, check error. Building your own iterators on the same contract means your code feels familiar to any Go developer who opens the file.

The real power shows up when iterators compose. A paginated iterator wrapped in a filter wrapped in a limit produces exactly the elements you need, fetches only the pages required, and reads like a pipeline declaration. That's the pattern working at its best -- not replacing simple loops, but making complex traversal feel simple.

> **Use an iterator when traversal is the hard part. Use a loop when it isn't.**
