+++
date = '2026-04-21'
title = 'Memento Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a configuration management tool. Users edit config values through a CLI or web UI -- changing database URLs, toggling feature flags, adjusting rate limits. Each change gets applied immediately. Then something breaks. The new database URL has a typo. The rate limit is too aggressive. The user needs to roll back to the previous configuration, or maybe two versions back.

The obvious approach is exposing the config struct's fields and manually copying them before each change. But that breaks encapsulation -- external code now knows the exact shape of the config's internals. And managing a history of copies manually, with proper undo/redo semantics, gets messy fast.

The **Memento pattern** solves this cleanly. The object that changes (the **Originator**) knows how to snapshot its own state into a sealed package (the **Memento**). A separate object (the **Caretaker**) holds the history of snapshots. When a rollback is needed, the caretaker hands a memento back to the originator, which restores itself. Nobody else ever sees or touches the internal state.

In this article, we'll build a configuration manager with full undo/redo support, including a diff-based variation for memory efficiency. Go's unexported fields make encapsulation natural, and the pattern maps cleanly onto struct snapshots.

## What Is the Memento Pattern?

> "Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later."
> -- Gang of Four

The intent is straightforward: **save an object's state at a point in time so you can restore it later, without exposing how that state is structured.** The key constraint is "without violating encapsulation" -- the saved state should be opaque to everything except the object that created it.

Without this pattern, undo typically looks like this:

```go
type Config struct {
    DBHost     string
    DBPort     int
    RateLimit  int
    DebugMode  bool
}

var history []Config

func saveState(c Config) {
    history = append(history, c)
}
```

It works for a flat struct. But it exposes every field to whoever manages the history. If `Config` gains private fields, computed state, or nested structures that need deep copying, the external snapshot breaks. And the moment multiple callers manage their own histories, state management scatters across the codebase.

The Memento pattern centralizes this: the config object creates its own snapshot, the snapshot is opaque to outsiders, and a caretaker manages the history without knowing what's inside.

## Core Components

Three participants.

**Originator** -- the object whose state changes and needs to be saved/restored. It creates mementos and restores itself from them.

```go
type ConfigManager struct {
    dbHost    string
    dbPort    int
    rateLimit int
    debug     bool
}
```

**Memento** -- a snapshot of the originator's state at a specific point in time. It's opaque to outsiders -- they can hold it but not read or modify it.

```go
type ConfigMemento struct {
    dbHost    string
    dbPort    int
    rateLimit int
    debug     bool
}
```

**Caretaker** -- manages a collection of mementos. Stores them, retrieves them for undo/redo, but never inspects or modifies their contents.

```go
type History struct {
    undoStack []ConfigMemento
    redoStack []ConfigMemento
}
```

The flow:

```
User changes config --> Originator creates Memento --> Caretaker stores it
User requests undo  --> Caretaker retrieves Memento --> Originator restores from it
```

The caretaker never opens the memento. The originator never manages the history. Each has one job.

## Code Walkthrough: A Configuration Manager with Undo/Redo

We're building a configuration manager where each change is automatically snapshotted, and users can undo and redo through the history.

### The Memento

The memento is a simple struct with unexported fields. In Go, unexported fields are the natural encapsulation mechanism -- code outside the package can hold a `ConfigMemento` but can't read or modify its contents.

```go
type ConfigMemento struct {
    dbHost    string
    dbPort    int
    rateLimit int
    debug     bool
}
```

### The Originator

The `ConfigManager` holds the live configuration and knows how to snapshot and restore itself:

```go
type ConfigManager struct {
    dbHost    string
    dbPort    int
    rateLimit int
    debug     bool
}

func NewConfigManager(host string, port int, rateLimit int, debug bool) *ConfigManager {
    return &ConfigManager{
        dbHost:    host,
        dbPort:    port,
        rateLimit: rateLimit,
        debug:     debug,
    }
}

func (c *ConfigManager) SetDatabase(host string, port int) {
    c.dbHost = host
    c.dbPort = port
}

func (c *ConfigManager) SetRateLimit(limit int) {
    c.rateLimit = limit
}

func (c *ConfigManager) SetDebug(on bool) {
    c.debug = on
}

func (c *ConfigManager) Save() ConfigMemento {
    return ConfigMemento{
        dbHost:    c.dbHost,
        dbPort:    c.dbPort,
        rateLimit: c.rateLimit,
        debug:     c.debug,
    }
}

func (c *ConfigManager) Restore(m ConfigMemento) {
    c.dbHost = m.dbHost
    c.dbPort = m.dbPort
    c.rateLimit = m.rateLimit
    c.debug = m.debug
}

func (c *ConfigManager) String() string {
    return fmt.Sprintf("db=%s:%d rate=%d debug=%v", c.dbHost, c.dbPort, c.rateLimit, c.debug)
}
```

`Save()` creates a memento by copying all fields. `Restore()` replaces all fields from a memento. The memento is a value type (not a pointer), so the caretaker gets an independent copy -- modifying the config after saving doesn't affect the snapshot.

### The Caretaker with Undo/Redo

The caretaker manages a stack-based history that supports both undo and redo:

```go
type History struct {
    undoStack []ConfigMemento
    redoStack []ConfigMemento
}

func NewHistory() *History {
    return &History{}
}

func (h *History) Push(m ConfigMemento) {
    h.undoStack = append(h.undoStack, m)
    h.redoStack = nil
}

func (h *History) Undo() (ConfigMemento, bool) {
    if len(h.undoStack) < 2 {
        return ConfigMemento{}, false
    }
    current := h.undoStack[len(h.undoStack)-1]
    h.undoStack = h.undoStack[:len(h.undoStack)-1]
    h.redoStack = append(h.redoStack, current)
    return h.undoStack[len(h.undoStack)-1], true
}

func (h *History) Redo() (ConfigMemento, bool) {
    if len(h.redoStack) == 0 {
        return ConfigMemento{}, false
    }
    m := h.redoStack[len(h.redoStack)-1]
    h.redoStack = h.redoStack[:len(h.redoStack)-1]
    h.undoStack = append(h.undoStack, m)
    return m, true
}
```

`Push` adds a new snapshot and clears the redo stack -- once a new change happens after an undo, you can't redo the old path. `Undo` pops the current state, moves it to redo, and returns the previous state. `Redo` reverses an undo.

The key design choice: `Undo` requires at least 2 entries because the top of the stack is always the *current* state. You undo *to* the entry below it.

### Putting It Together

```go
func main() {
    config := NewConfigManager("localhost", 5432, 100, false)
    history := NewHistory()

    history.Push(config.Save())

    config.SetDatabase("prod-db.internal", 5432)
    history.Push(config.Save())
    fmt.Println("After DB change:", config)

    config.SetRateLimit(50)
    history.Push(config.Save())
    fmt.Println("After rate limit:", config)

    config.SetDebug(true)
    history.Push(config.Save())
    fmt.Println("After debug on: ", config)

    if m, ok := history.Undo(); ok {
        config.Restore(m)
        fmt.Println("After undo 1:   ", config)
    }

    if m, ok := history.Undo(); ok {
        config.Restore(m)
        fmt.Println("After undo 2:   ", config)
    }

    if m, ok := history.Redo(); ok {
        config.Restore(m)
        fmt.Println("After redo:     ", config)
    }
}
```

```
After DB change: db=prod-db.internal:5432 rate=100 debug=false
After rate limit: db=prod-db.internal:5432 rate=50 debug=false
After debug on:  db=prod-db.internal:5432 rate=50 debug=true
After undo 1:    db=prod-db.internal:5432 rate=50 debug=false
After undo 2:    db=prod-db.internal:5432 rate=100 debug=false
After redo:      db=prod-db.internal:5432 rate=50 debug=false
```

Four states saved, two undos, one redo. The configuration object handles its own snapshotting. The history manages the stack. The client code (`main`) never accesses the config's internal fields directly -- it works entirely through `SetX`, `Save`, and `Restore`.

## Encapsulation: Why the Memento Is Opaque

In Go, encapsulation works through unexported fields. Since `ConfigMemento` has only unexported fields (`dbHost`, `dbPort`, `rateLimit`, `debug`), code outside the package can hold a `ConfigMemento` value but can't read or modify any of its contents.

This is the pattern's central constraint. The caretaker stores mementos but is structurally prevented from tampering with them. Compare this to the naive approach of passing a `Config` struct with exported fields -- anyone in the chain can modify the "snapshot" and corrupt the history.

One nuance: in our walkthrough, everything lives in `package main`, so the caretaker *technically* has access to the memento's unexported fields -- they're in the same package. In a real codebase, the originator and memento would live in one package (e.g., `config`) while the caretaker lives in another (e.g., `history`). That's when Go's visibility rules actually enforce the opacity. In a single-package example the protection is conventional; in production code, package boundaries make it structural.

In languages without package-level visibility (like Java), the Memento typically uses a nested class or a narrow interface to achieve opacity. In Go, unexported fields plus package separation do this naturally and idiomatically.

## Variation: Diff-Based Mementos

For objects with large state -- a document with thousands of lines, a game world with complex entity data -- storing full snapshots at every change burns memory fast. A practical variation stores only the *difference* between states.

```go
type ConfigDiff struct {
    field    string
    oldValue any
}

type DiffMemento struct {
    diffs []ConfigDiff
}
```

The originator records what changed rather than copying everything:

```go
func (c *ConfigManager) SetRateLimitWithDiff(limit int) DiffMemento {
    diff := DiffMemento{
        diffs: []ConfigDiff{{field: "rateLimit", oldValue: c.rateLimit}},
    }
    c.rateLimit = limit
    return diff
}

func (c *ConfigManager) RestoreDiff(m DiffMemento) {
    for _, d := range m.diffs {
        switch d.field {
        case "rateLimit":
            c.rateLimit = d.oldValue.(int)
        case "dbHost":
            c.dbHost = d.oldValue.(string)
        }
    }
}
```

This trades simplicity for efficiency. Full-state mementos are O(state size) per snapshot regardless of how much changed. Diff mementos are O(changes) per snapshot. For a config manager with 50 fields where most changes touch one field, the difference matters.

The tradeoff: diff-based undo must be applied in strict reverse order. You can't jump to an arbitrary point in history without replaying all intermediate diffs. Full-state mementos allow random access to any snapshot. Choose based on your access patterns.

## When to Use It

The Memento pattern fits when you need **state snapshots with rollback, while keeping the object's internals private**:

- **Undo/redo systems** -- text editors, configuration tools, form wizards, drawing applications. Any place where users expect to reverse their actions.
- **Transaction rollback** -- applying a series of changes to an object and reverting all of them if any step fails. Similar to the Command pattern's undo, but focused on state snapshots rather than inverse operations.
- **Checkpointing in long-running processes** -- saving progress at intervals so a failure doesn't lose everything. Migration tools, batch processors, workflow engines.
- **State debugging and auditing** -- recording the state of an object at key points for later inspection. Useful in complex systems where reproducing a bug requires knowing the exact state sequence.

Skip it when:

- **The object's state is trivially copyable and already public.** If the struct has all exported fields and copying it is fine, a plain `history = append(history, config)` is simpler and clearer.
- **State changes are better expressed as operations than snapshots.** If each change has a natural inverse (increment/decrement, add/remove), the Command pattern with undo is usually cleaner than storing full snapshots.
- **Memory is constrained and state is large.** If full snapshots are too expensive and diff-based mementos are too complex for the use case, consider event sourcing or append-only logs instead.

## Pros and Cons

**Pros:**

- **Encapsulation preserved** -- the object's internal state stays private. The caretaker holds mementos but can't read or modify them, preventing accidental corruption of history.
- **Clean separation of concerns** -- the originator manages its state, the caretaker manages history, and the two don't bleed into each other. Each is independently testable.
- **Arbitrary rollback** -- with full-state mementos, you can restore to any point in history, not just the immediately previous state. Jump back five versions if needed.
- **Simple to implement in Go** -- unexported struct fields provide natural encapsulation. Value-type mementos give you automatic deep copies for flat structs.

**Cons:**

- **Memory cost** -- every snapshot stores a complete copy of the state. For objects with large state or frequent changes, this adds up. Ten thousand snapshots of a 1KB config is 10MB.
- **No partial restore** -- full-state mementos are all-or-nothing. You can't undo just the database change while keeping the rate limit change. The entire state rolls back together.
- **Deep copy complexity** -- if the originator holds pointers, slices, or maps, the memento must deep-copy them. A shallow copy creates shared references between the live object and the snapshot, corrupting history when the live object changes.

## Best Practices

- **Return mementos as value types, not pointers.** `Save() ConfigMemento` (value) gives the caretaker an independent copy automatically. `Save() *ConfigMemento` (pointer) shares memory with the originator -- modifying the originator could corrupt the snapshot.
- **Deep-copy reference types inside `Save()`.** If the originator holds a `map[string]string` or `[]Item`, the memento must copy the underlying data, not just the slice header or map reference. Otherwise the snapshot and the live object share the same backing array.
- **Clear the redo stack on new changes.** Once the user makes a new edit after undoing, the old redo path is invalid. Forgetting to clear it leads to incoherent state when redo is pressed.
- **Limit history size in production.** An unbounded history slice grows forever. Cap it at N entries, or implement a ring buffer that drops the oldest snapshots when the limit is reached.
- **Keep the memento in the same package as the originator.** Go's encapsulation is package-level. If the memento lives in a different package with exported fields, the opacity guarantee breaks. Same package, unexported fields -- that's the combination that makes it work.

## Common Mistakes

**Storing pointers instead of values in the memento.** If the originator has a `Tags []string` field and the memento copies the slice header, both the live object and the snapshot point to the same underlying array. Modifying `Tags` after saving corrupts the snapshot. The fix is explicit: `copy(m.tags, o.tags)` or `slices.Clone(o.tags)`.

**Making the caretaker responsible for deciding when to save.** The caretaker should store and retrieve mementos -- period. The *originator* or the *client* decides when a snapshot is worth taking (before a destructive change, at a checkpoint, on explicit user action). If the caretaker auto-saves on every method call, you get excessive snapshots of intermediate states.

**Treating Memento as a replacement for event sourcing.** Memento stores state snapshots. Event sourcing stores the sequence of *changes*. For systems that need to replay history, audit who changed what, or derive multiple views from the same events, Memento is the wrong tool. It answers "what was the state at time T?" but not "what sequence of actions led to time T?"

**Forgetting that undo/redo has UX implications.** If `Save()` is called at the wrong granularity -- every keystroke instead of every logical action -- undo becomes unusably granular. If it's called too rarely, undo skips over changes the user wanted to roll back individually. The pattern provides the mechanism; the call site determines the user experience.

## Final Thoughts

The Memento pattern gives objects the ability to save and restore their own state, while keeping that state invisible to the rest of the system. It's a focused tool -- when you need undo/redo, checkpointing, or rollback, it provides clean mechanics without breaking encapsulation.

Go makes the pattern feel natural. Unexported fields provide package-level opacity by default. Value-type mementos give automatic copy semantics for flat structs. And the separation between originator, memento, and caretaker maps cleanly onto Go's preference for small, composable types with clear responsibilities.

The main judgment call is granularity: when to snapshot, how many to keep, and whether full-state or diff-based mementos fit your access patterns. Get that right, and the pattern stays lightweight. Get it wrong, and you're burning memory on snapshots nobody uses or missing the ones they need.

> **Save state when it matters, restore it when it breaks, and never let the outside world peek inside the snapshot.**
