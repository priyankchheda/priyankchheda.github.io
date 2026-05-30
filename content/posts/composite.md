+++
date = '2025-11-27'
title = 'Composite Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about a deployment pipeline. Some steps are simple -- run a migration, restart a service, send a notification. Other steps are *groups* of steps -- "deploy backend" means run migrations, build the binary, restart the service, and run smoke tests. And groups can contain other groups -- "deploy everything" contains "deploy backend," "deploy frontend," and "update DNS."

The client that triggers the pipeline shouldn't care whether it's executing a single step or a deeply nested group of steps. It calls `Execute()` on whatever it has, and the right thing happens. A single step runs itself. A group runs its children. A group containing groups recurses naturally. The interface is the same at every level.

The **Composite pattern** makes this possible. It lets you treat individual objects and groups of objects through the same interface. The tree structure -- steps containing steps containing steps -- is handled uniformly. No type switches, no "is this a leaf or a container" checks, no special-casing. One interface, recursive composition, and the client stays blissfully unaware of the depth.

Go's interfaces and struct composition make this pattern clean. In this article, we'll build a deployment pipeline where tasks and task groups share the same interface, compose recursively, and execute uniformly.

## What Is the Composite Pattern?

> "Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly."
> -- Gang of Four

The intent: **let clients work with single objects and groups of objects through the same interface, hiding the structural complexity.** A leaf does its own work. A composite delegates to its children. The client calls the same method on both and doesn't care which one it's talking to.

Without this pattern, tree-structured execution requires type checks everywhere:

```go
func execute(step interface{}) {
    switch s := step.(type) {
    case *SingleTask:
        s.Run()
    case *TaskGroup:
        for _, child := range s.Tasks {
            execute(child)
        }
    }
}
```

Every function that interacts with the pipeline needs this switch. Add a new type (conditional task, parallel group) and every switch needs updating. The client must always know what it's dealing with.

With the Composite pattern, the client just calls:

```go
step.Execute()
```

Whether `step` is a single task or a group of 50 nested tasks, it works. The interface hides the structure.

## Core Components

Three participants.

**Component** -- the interface shared by both individual objects and groups.

```go
type Task interface {
    Execute() error
    Name() string
}
```

**Leaf** -- a single object that does actual work. It implements the interface directly, with no children.

**Composite** -- a group that holds a collection of Components (which can be leaves or other composites). It implements the interface by delegating to its children.

The structure:

```
Task (interface)
 ├── SingleTask (leaf) -- does work
 └── TaskGroup (composite) -- delegates to children
       ├── SingleTask
       ├── SingleTask
       └── TaskGroup (nested composite)
             ├── SingleTask
             └── SingleTask
```

The client operates on `Task`. It never knows whether it holds a leaf or a tree.

## Code Walkthrough: A Deployment Pipeline

We're building a deployment pipeline where individual tasks and groups of tasks share the same interface. Groups can contain other groups, forming an arbitrarily deep tree.

### The Component Interface

```go
type Task interface {
    Execute() error
    Name() string
}
```

Two methods. Every pipeline step -- whether it's "restart nginx" or "deploy entire production stack" -- satisfies this.

### The Leaf: SingleTask

A single unit of work:

```go
type SingleTask struct {
    name   string
    action func() error
}

func NewTask(name string, action func() error) *SingleTask {
    return &SingleTask{name: name, action: action}
}

func (t *SingleTask) Execute() error {
    fmt.Printf("  [task] %s\n", t.name)
    return t.action()
}

func (t *SingleTask) Name() string {
    return t.name
}
```

Simple: it has a name and an action. `Execute()` runs the action. No children, no delegation.

### The Composite: TaskGroup

A group of tasks that executes its children sequentially:

```go
type TaskGroup struct {
    name     string
    children []Task
}

func NewTaskGroup(name string) *TaskGroup {
    return &TaskGroup{name: name}
}

func (g *TaskGroup) Add(tasks ...Task) *TaskGroup {
    g.children = append(g.children, tasks...)
    return g
}

func (g *TaskGroup) Execute() error {
    fmt.Printf("[group] %s (%d steps)\n", g.name, len(g.children))
    for _, child := range g.children {
        if err := child.Execute(); err != nil {
            return fmt.Errorf("%s: %w", g.name, err)
        }
    }
    return nil
}

func (g *TaskGroup) Name() string {
    return g.name
}
```

`TaskGroup` implements `Task` -- same interface as `SingleTask`. But internally it holds a slice of `Task` and iterates over them. Children can be single tasks or other groups. The recursion is structural: a group's `Execute` calls each child's `Execute`, which might itself be a group.

`Add` returns `*TaskGroup` for fluent construction, but the group is used as a `Task` once composed.

### Building the Pipeline

```go
func main() {
    migrate := NewTask("run migrations", func() error {
        fmt.Println("    applying schema changes...")
        return nil
    })

    build := NewTask("build binary", func() error {
        fmt.Println("    compiling go build...")
        return nil
    })

    restart := NewTask("restart service", func() error {
        fmt.Println("    systemctl restart api...")
        return nil
    })

    smokeTest := NewTask("smoke test", func() error {
        fmt.Println("    curl health endpoint...")
        return nil
    })

    deployBackend := NewTaskGroup("deploy backend").Add(
        migrate, build, restart, smokeTest,
    )

    updateDNS := NewTask("update DNS", func() error {
        fmt.Println("    pointing A record...")
        return nil
    })

    notifyTeam := NewTask("notify team", func() error {
        fmt.Println("    posting to #deploys...")
        return nil
    })

    deployAll := NewTaskGroup("deploy everything").Add(
        deployBackend, updateDNS, notifyTeam,
    )

    if err := deployAll.Execute(); err != nil {
        fmt.Println("FAILED:", err)
    }
}
```

```txt
[group] deploy everything (3 steps)
[group] deploy backend (4 steps)
  [task] run migrations
    applying schema changes...
  [task] build binary
    compiling go build...
  [task] restart service
    systemctl restart api...
  [task] smoke test
    curl health endpoint...
  [task] update DNS
    pointing A record...
  [task] notify team
    posting to #deploys...
```

`deployAll` is a group containing a group and two single tasks. The client calls `deployAll.Execute()` -- one line -- and the entire tree executes recursively. The nested group (`deployBackend`) runs its four children before the pipeline continues with `updateDNS` and `notifyTeam`.

### Error Handling: Stopping on Failure

If the build step fails:

```go
build := NewTask("build binary", func() error {
    return fmt.Errorf("compilation error: missing import")
})
```

```txt
[group] deploy everything (3 steps)
[group] deploy backend (4 steps)
  [task] run migrations
    applying schema changes...
  [task] build binary
FAILED: deploy everything: deploy backend: compilation error: missing import
```

The error bubbles up through the tree. `TaskGroup.Execute()` wraps each error with the group's name, creating a clear path: `deploy everything: deploy backend: compilation error`. The pipeline stops at the first failure -- no subsequent steps execute.

### The Visualization

```
deployAll (TaskGroup)
├── deployBackend (TaskGroup)
│   ├── migrate (SingleTask)
│   ├── build (SingleTask)
│   ├── restart (SingleTask)
│   └── smokeTest (SingleTask)
├── updateDNS (SingleTask)
└── notifyTeam (SingleTask)
```

The client calls `Execute()` on the root. The tree handles itself.

## Adding Parallel Execution

A natural extension: some groups should run their children concurrently instead of sequentially.

```go
type ParallelGroup struct {
    name     string
    children []Task
}

func NewParallelGroup(name string) *ParallelGroup {
    return &ParallelGroup{name: name}
}

func (g *ParallelGroup) Add(tasks ...Task) *ParallelGroup {
    g.children = append(g.children, tasks...)
    return g
}

func (g *ParallelGroup) Execute() error {
    fmt.Printf("[parallel] %s (%d steps)\n", g.name, len(g.children))
    errs := make(chan error, len(g.children))
    for _, child := range g.children {
        go func(t Task) {
            errs <- t.Execute()
        }(child)
    }
    for range g.children {
        if err := <-errs; err != nil {
            return fmt.Errorf("%s: %w", g.name, err)
        }
    }
    return nil
}

func (g *ParallelGroup) Name() string {
    return g.name
}
```

`ParallelGroup` satisfies `Task`. It looks identical to `TaskGroup` from the outside. But internally it launches goroutines. You can mix sequential and parallel groups freely:

```go
deployAll := NewTaskGroup("deploy everything").Add(
    NewParallelGroup("deploy services").Add(backendDeploy, frontendDeploy),
    updateDNS,
    notifyTeam,
)
```

Backend and frontend deploy concurrently. DNS update waits for both to finish. The tree structure composes naturally.

## When to Use It

The Composite pattern fits when **you have a tree structure where individual items and groups should be treated uniformly**:

- **Pipeline/workflow systems** where steps can be grouped, nested, and executed through a single `Execute()` call regardless of depth.
- **File system operations** where files and directories share operations (size, permissions, search) and directories recurse into their contents.
- **UI component trees** where containers hold widgets, widgets can be containers themselves, and rendering/layout applies uniformly.
- **Permission systems** where individual permissions and permission groups are evaluated through the same `HasAccess()` interface.
- **Pricing/discount engines** where individual line items and bundles of items both support `Total()` and compose into arbitrary nesting.

Skip it when:

- **The structure is flat.** If you'll never have groups within groups -- just a list of items -- a simple slice is clearer than a composite tree.
- **Leaves and composites have genuinely different operations.** If groups need `Add`/`Remove` but leaves don't, forcing both into one interface creates methods that don't make sense on leaves. Consider separating the interfaces.
- **You need random access by index.** Composite trees are traversed, not indexed. If clients need `GetChild(3)` or `FindByName("step-7")`, the uniform interface doesn't help -- you need tree-specific navigation.

## Pros and Cons

**Pros:**

- **Uniform client code** -- the client calls `Execute()` on a leaf or a tree of 1000 nodes. Same code, same interface. No type switches, no depth checks.
- **Natural recursion** -- the composite's `Execute()` calls each child's `Execute()`, which might recursively call further children. The tree structure handles itself.
- **Open/Closed for new node types** -- add `ParallelGroup`, `ConditionalTask`, `RetryTask` -- each implements `Task` and slots into existing trees without modifying existing code.
- **Flexible composition at runtime** -- build pipelines dynamically based on config, feature flags, or user input. The tree structure is data, not hardcoded control flow.

**Cons:**

- **Overly general interface** -- the shared interface must make sense for both leaves and composites. Methods like `Add()` on a leaf are meaningless. Go doesn't force you to include them (unlike some OOP languages), but the design tension is real.
- **Hard to constrain** -- it's difficult to restrict what types a composite can contain. "A deploy group should only contain deploy tasks, not notification tasks" -- the type system doesn't enforce this when everything is just `Task`.
- **Debugging nested trees** -- when a deeply nested step fails, understanding which level and which branch requires clear error wrapping or logging at every level.

## Best Practices

- **Wrap errors with the group name at each level.** `fmt.Errorf("%s: %w", g.name, err)` creates a breadcrumb trail: `deploy everything: deploy backend: build binary: compilation error`. Without this, a failure deep in the tree produces an error with no context about *where* in the tree it happened.
- **Keep the Component interface minimal.** Only include methods that genuinely apply to both leaves and composites. `Execute()` and `Name()` work for both. `Add()` is specific to composites and should stay on `TaskGroup`, not on `Task`.
- **Return `*TaskGroup` from `Add()` for fluent construction, but use `Task` for composition.** Construct groups fluently (`group.Add(a, b, c)`), but pass them around as `Task`. This keeps construction ergonomic while preserving the uniform interface for execution.
- **Make leaf execution simple and self-contained.** A `SingleTask` should do one thing. If a leaf needs setup, work, and teardown, it might actually be a group of three leaves.

## Common Mistakes

**Putting `Add`/`Remove` on the Component interface.** In strict GoF, the Component interface includes child management methods so the client can treat leaves and composites identically even for tree manipulation. But in Go, this leads to leaves implementing no-op `Add` methods or panicking. Better approach: keep `Add` on the composite type only, and use the shared interface (`Task`) only for operations that apply to both.

**Forgetting that execution order is an implicit contract.** A `TaskGroup` executes children in the order they were added. If "build" must happen before "restart," that ordering is enforced only by the order of `Add()` calls -- not by the type system. Document the expected ordering, or use named stages if ordering is critical.

**Making composites too smart.** A `TaskGroup` that retries failed children, skips certain steps based on conditions, and logs metrics has taken on too many responsibilities. Keep the composite focused on delegation. Retry logic should be a decorator around a task. Conditional execution should be a `ConditionalTask` leaf. Metrics should be a separate concern.

**Using Composite when a flat list would suffice.** If your "pipeline" is always a flat sequence of steps with no nesting, `[]Task` with a loop is simpler and more obvious than a composite tree. The pattern earns its keep when nesting is real and variable -- groups within groups. Without that, it's over-engineering.

## Final Thoughts

The Composite pattern lets you build tree structures where every node -- leaf or branch -- satisfies the same interface. The client calls one method on the root, and the tree handles itself recursively. Individual tasks and groups of tasks are interchangeable. Depth is hidden. Composition is the architecture.

Go makes this clean through interfaces and slices. The component is an interface. Leaves implement it directly. Composites hold a `[]Component` and delegate. No abstract classes, no inheritance hierarchies, no visitor infrastructure -- just an interface, a slice, and a `for` loop that calls the interface method on each child.

The pattern works best when the nesting is real -- pipelines with sub-pipelines, directories with sub-directories, permission groups with sub-groups. If your structure is flat, keep it flat. But when the tree is genuine, Composite turns complex recursive structures into a single `Execute()` call.

> **Treat one and many the same. Let the tree handle itself.**
