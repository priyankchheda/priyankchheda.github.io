+++
date = '2026-06-07'
title = 'Command Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a deployment tool. It runs a database migration, rebuilds a service, restarts the container. Three function calls. Straightforward enough -- until someone asks for rollback. Now the tool needs to know how to reverse each step. Then someone wants to queue deployments. Then audit logging. Suddenly the runner is tangled with undo logic, queue management, and logging, and every new requirement means rewriting the same function.

This is the exact kind of mess the **Command pattern** was designed to prevent. The idea is deceptively simple: take an action, wrap it in an object, and let that object be passed around like any other piece of data. Once an action is an object, you can store it in a slice, send it over a channel, reverse it, combine it with other actions, or replay it a week later. You can't do any of that with a hardcoded function call.

Go makes this pattern particularly clean. First-class functions mean you can start with a bare `func()` and only graduate to structs when you actually need state or undo. No frameworks, no abstract base classes, no ceremony. In this article, we'll build a database migration pipeline that uses the Command pattern to handle execution, rollback, and composition -- and then explore how commands flow through channels for concurrent processing.

## What Is the Command Pattern?

> "Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations."<br>- Gang of Four

The restaurant analogy is overused, but it holds up. A customer tells the waiter what they want. The waiter writes it on an order slip and carries it to the kitchen. The chef cooks the food. The waiter doesn't know how to cook. The chef doesn't know who ordered. The order slip -- the command -- connects them while keeping them completely independent.

In software terms: the **Client** creates a **Command** that knows which **Receiver** to call. The **Invoker** triggers the command without knowing or caring what happens behind the `Execute()` call. Control flows in one direction, and nobody talks to anyone they don't need to.

Without this separation, here's what a deploy tool looks like:

```go
type Runner struct {
    db     *Database
    server *Server
}

func (r *Runner) Deploy() {
    r.db.Migrate()
    r.server.Build()
    r.server.Restart()
}
```

Clean, sure. But `Runner` is now coupled to both `Database` and `Server`. Adding rollback means `Runner` has to understand how to reverse each step. Adding a new step means modifying `Runner`. It violates Open/Closed, it violates Single Responsibility, and it turns into a god object fast.

The Command pattern breaks this apart. The runner holds a list of commands. Each command knows one thing and how to undo it. The runner just iterates.

## Core Components

Five participants, each doing one job.

**Command** -- the interface.

```go
type Command interface {
    Execute() error
    Undo() error
    String() string
}
```

**ConcreteCommand** -- binds a specific action to a specific receiver. This is where "what to do" meets "who does it."

**Receiver** -- the object doing the actual work. It has no idea it's being called through a command.

**Invoker** -- triggers commands, optionally tracks history. Knows nothing about the business logic behind `Execute()`.

**Client** -- wires receivers into commands and hands commands to the invoker. This is usually `main()`.

The flow is always: `Client --> Invoker --> Command --> Receiver`. The invoker never touches the receiver. The receiver never knows a command exists. That separation is the whole point.

## Code Walkthrough: A Migration Pipeline

We're building a deployment pipeline. It runs database migrations and service restarts, and if anything fails partway through, it rolls everything back automatically.

Start with the receiver -- a database that tracks its schema version:

```go
type Database struct {
    name    string
    version int
}

func (d *Database) MigrateTo(version int) error {
    fmt.Printf("  [%s] migrating v%d -> v%d\n", d.name, d.version, version)
    d.version = version
    return nil
}

func (d *Database) Version() int {
    return d.version
}
```

The concrete command wraps a single migration step:

```go
type MigrateCommand struct {
    db          *Database
    toVersion   int
    fromVersion int
}

func (c *MigrateCommand) Execute() error {
    c.fromVersion = c.db.Version()
    return c.db.MigrateTo(c.toVersion)
}

func (c *MigrateCommand) Undo() error {
    return c.db.MigrateTo(c.fromVersion)
}

func (c *MigrateCommand) String() string {
    return fmt.Sprintf("migrate %s to v%d", c.db.name, c.toVersion)
}
```

One thing worth noting: `Execute()` snapshots the current version *before* it runs. Undo needs to know where to revert to, and that information only exists at execution time. Hardcoding the "from" version at construction time is a common mistake -- it breaks the moment commands are reordered or retried.

Add a second receiver to make it feel more real:

```go
type Service struct {
    name    string
    running bool
}

func (s *Service) Restart() error {
    fmt.Printf("  [%s] restarting...\n", s.name)
    s.running = true
    return nil
}

func (s *Service) Stop() error {
    fmt.Printf("  [%s] stopping...\n", s.name)
    s.running = false
    return nil
}

type RestartCommand struct {
    service    *Service
    wasRunning bool
}

func (c *RestartCommand) Execute() error {
    c.wasRunning = c.service.running
    return c.service.Restart()
}

func (c *RestartCommand) Undo() error {
    if !c.wasRunning {
        return c.service.Stop()
    }
    return c.service.Restart()
}

func (c *RestartCommand) String() string {
    return fmt.Sprintf("restart %s", c.service.name)
}
```

Now the invoker. The pipeline runs commands in sequence, and if any step fails, it rolls back everything that already succeeded -- in reverse order:

```go
type Pipeline struct {
    executed []Command
}

func (p *Pipeline) Run(commands ...Command) error {
    for _, cmd := range commands {
        fmt.Printf("-> executing: %s\n", cmd)
        if err := cmd.Execute(); err != nil {
            fmt.Printf("x  failed: %s (%v)\n", cmd, err)
            if rbErr := p.rollback(); rbErr != nil {
                return fmt.Errorf("pipeline failed at [%s]: %w (rollback also failed: %v)", cmd, err, rbErr)
            }
            return fmt.Errorf("pipeline failed at [%s]: %w", cmd, err)
        }
        p.executed = append(p.executed, cmd)
    }
    fmt.Println("pipeline complete")
    return nil
}

func (p *Pipeline) rollback() error {
    fmt.Println("  rolling back...")
    var rollbackErr error
    for i := len(p.executed) - 1; i >= 0; i-- {
        cmd := p.executed[i]
        fmt.Printf("  <- undoing: %s\n", cmd)
        if err := cmd.Undo(); err != nil {
            fmt.Printf("  rollback failed: %s (%v)\n", cmd, err)
            rollbackErr = err
        }
    }
    p.executed = nil
    return rollbackErr
}
```

The pipeline doesn't know what a database migration is. It doesn't know what a service restart is. It calls `Execute()` and, when things go wrong, `Undo()`. You could slot in a command that sends a Slack notification and the pipeline wouldn't care.

Wire it all up:

```go
func main() {
    db := &Database{name: "users-db", version: 1}
    api := &Service{name: "api-server", running: true}

    pipeline := &Pipeline{}
    err := pipeline.Run(
        &MigrateCommand{db: db, toVersion: 2},
        &MigrateCommand{db: db, toVersion: 3},
        &RestartCommand{service: api},
    )
    if err != nil {
        fmt.Println("Deploy failed:", err)
    }
}
```

```txt
-> executing: migrate users-db to v2
  [users-db] migrating v1 -> v2
-> executing: migrate users-db to v3
  [users-db] migrating v2 -> v3
-> executing: restart api-server
  [api-server] restarting...
pipeline complete
```

If the second migration fails, the pipeline automatically rolls back the first. The calling code doesn't deal with partial failure at all.

## Macro Commands

Sometimes one command should trigger many. A "full deploy" that runs migrations, rebuilds, and restarts. This is the Composite pattern applied to commands.

```go
type MacroCommand struct {
    label    string
    commands []Command
}

func (m *MacroCommand) Execute() error {
    for _, cmd := range m.commands {
        if err := cmd.Execute(); err != nil {
            return err
        }
    }
    return nil
}

func (m *MacroCommand) Undo() error {
    for i := len(m.commands) - 1; i >= 0; i-- {
        if err := m.commands[i].Undo(); err != nil {
            return err
        }
    }
    return nil
}

func (m *MacroCommand) String() string { return m.label }
```

`MacroCommand` itself satisfies `Command`. The pipeline sees one command; internally it's five. Undo reverses them in the right order. And because it's just another `Command`, you can nest macros, pass them through channels, or wrap them in decorators.

## Commands Over Channels

This is where Go makes the pattern shine in a way most languages can't easily match. Commands are objects. Channels carry objects. A concurrent task executor falls out in about 15 lines:

```go
func worker(id int, jobs <-chan Command, results chan<- error) {
    for cmd := range jobs {
        fmt.Printf("  worker %d: %s\n", id, cmd)
        results <- cmd.Execute()
    }
}
```

Spin up a few goroutines, push commands into the channel, collect results. The workers don't know what they're executing. Add a retry wrapper, a timeout decorator, a logging layer -- they all compose because they all implement the same interface.

If this feels familiar, it should. Go's `http.Handler` and `http.HandlerFunc` use the exact same structure. `Handler` is the Command interface. `HandlerFunc` is a function adapter. Middleware wraps handlers -- decorator commands. The router is the invoker. The Command pattern, handling millions of requests, baked right into the standard library.

## Idiomatic Go: Pick Your Level of Ceremony

Go gives you a spectrum. Where you land on it matters more than whether you "used the pattern correctly."

**Plain functions** work when you don't need undo, state, or metadata:

```go
type Action func() error

func runAll(actions ...Action) error {
    for _, a := range actions {
        if err := a(); err != nil {
            return err
        }
    }
    return nil
}
```

This is still the Command pattern. Actions are decoupled from the runner. You can store them, reorder them, retry them.

**Function type with interface adapter** bridges the gap when some callers need an interface. Say your simpler commands only need `Execute()`:

```go
type SimpleCommand interface {
    Execute() error
}

type CommandFunc func() error

func (f CommandFunc) Execute() error { return f() }
```

Now closures satisfy `SimpleCommand` wherever it's expected. This is the `http.HandlerFunc` trick -- `HandlerFunc` adapts a plain function to satisfy the `Handler` interface. For our pipeline's full `Command` interface (with `Undo()` and `String()`), you'd still need structs. But plenty of systems only need `Execute()`, and this adapter keeps those cases clean.

**Struct-based commands** are for when you need to remember things -- state for undo, timestamps for logging, parameters for retry.

The rule: **start with functions. Move to structs when state enters the picture.** Not because a book said so, but because the code demanded it.

## When to Use It

The Command pattern earns its keep when actions need to be **more than fire-and-forget**. Specifically: when you need undo/redo, when actions get queued or scheduled for later, when you want to compose multiple actions into a single unit, or when you're building audit logs that capture intent rather than just side effects.

Skip it when a direct function call does the job. If there's no undo, no queue, no behavioral variability, and no composition -- you're adding indirection for nothing. Wrapping `processOrder(order)` in a `ProcessOrderCommand` struct just because "we use the Command pattern here" is textbook over-engineering.

The honest litmus test: **if you can't name a concrete benefit the pattern gives you today, you don't need it yet.**

## Pros and Cons

**Pros:**

- **Composability** -- commands share an interface, which means macro commands, decorator commands, queues, and channel-based workers all just work. Build one piece and it snaps into everything else.
- **Testability** -- test the pipeline with stub commands. Test commands with mock receivers. Each layer is verifiable in isolation without spinning up the whole system.
- **Open/Closed compliance** -- need to add a `NotifySlackCommand` to the deploy pipeline? Write the struct, pass it in. The pipeline, the database, and the server don't change.
- **Natural undo model** -- each command owns its inverse operation and the state needed to perform it. The invoker just manages a stack. No centralized "undo registry" needed.
- **Deferred execution** -- because commands are objects, they can be created now and executed later. This is what makes queuing, scheduling, and retry possible without any special infrastructure.

**Cons:**

- **Type proliferation** -- every distinct action typically needs its own struct. In a large system, that's a lot of small types that each do very little individually.
- **Indirection tax** -- when someone asks "what happens when we deploy?", they trace through Pipeline, MacroCommand, MigrateCommand, and Database. With a direct call, it's one function. Debuggability is real, and the Command pattern works against it.
- **Undo is deceptively hard** -- the pattern gives you the *structure* for undo, but the *logic* is still your problem. Toggling a boolean is trivial. Rolling back a migration that three other migrations depend on is a different beast entirely, and the pattern won't save you from that complexity.

## Best Practices

- **Snapshot state in `Execute()`, not the constructor.** The state you need for undo might not exist yet when the command is created -- especially if commands are queued. `Execute()` is the only place where you're guaranteed the world is in the right state.
- **Keep the invoker thin.** It calls `Execute()`, tracks history, and handles errors. That's it. The moment the invoker has an `if` branch for a specific command type, you've reintroduced the coupling the pattern was supposed to eliminate.
- **Return errors from `Execute()`.** The textbook signature returns nothing. Real operations fail. Designing for errors from the start avoids a painful refactor later when you realize you've been silently swallowing failures.
- **Undo in reverse order.** When rolling back a sequence of commands, go last-to-first. If migration B depends on migration A having run, undoing A first leaves B in a broken state.
- **One interface is enough.** You need a `Command` interface. You almost certainly don't need `Receiver` and `Invoker` interfaces too. Go's implicit interface satisfaction means the receiver doesn't need to declare anything special -- it's just a struct with methods.
- **Let the receiver own domain logic.** Commands delegate; they don't replace. A `MigrateCommand` that contains SQL queries directly has merged two responsibilities. When the migration logic changes, you should be editing the `Database` type, not the command.

## Common Mistakes

**Sharing mutable commands across goroutines.** If a command captures state in `Execute()` (like our `fromVersion` field), passing the same command instance to multiple goroutines is a data race. Either create a fresh command per goroutine, or protect shared state with a mutex. The `-race` flag will catch this, but only if you remember to run it.

**Fat command interfaces.** It's tempting to keep adding methods -- `Validate()`, `Priority()`, `Timeout()` -- to the base `Command` interface. But then every command has to implement methods it doesn't care about. In our pipeline example, we bundled `Execute()`, `Undo()`, and `String()` together because every command in that system genuinely needs all three. That's fine for a focused use case. The mistake is doing it at the wrong level of abstraction -- a shared library or framework-level `Command` interface should be minimal, with optional capabilities expressed as separate interfaces that the invoker can type-assert against.

**Ignoring Go's functions.** Plenty of Go codebases have 30 command structs where 25 of them could have been closures. If your command has no state and no undo, `func() error` is the right tool. The `http.HandlerFunc` adapter exists precisely because Go's standard library authors recognized this.

**Silently swallowing undo failures.** When rollback fails, logging and continuing is sometimes the right call -- but silently ignoring the error is never correct. A failed undo means the system is in a partially rolled-back state. At minimum, surface it to the caller so they can decide whether to panic, alert, or retry.

## Final Thoughts

The Command pattern takes actions and gives them the same powers as data: you can store them, move them, compose them, and reverse them. Go's implicit interfaces and first-class functions let the pattern scale from a single closure all the way up to a concurrent pipeline with undo, macros, and audit logging.

But the best advice is also the most boring: don't reach for the pattern until you feel the problem it solves. A direct function call is easier to read, easier to debug, and easier to explain to the next person who opens the file. The Command pattern earns its complexity when actions need structure and control. Until then, a function is just fine.

> **Turn actions into objects when behavior needs structure. Keep them as functions when it doesn't.**
