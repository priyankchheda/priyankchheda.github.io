+++
date = '2026-06-14'
title = 'Template Method Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about a data ingestion pipeline. Whether you're importing users from a CSV file, orders from a JSON API, or products from an XML feed, the overall workflow is the same: validate the source, parse the records, transform each record to your internal format, save to the database, and report results. The *structure* is fixed -- you always validate before parsing, always transform before saving. But the *implementation* of each step differs per source: CSV parsing is nothing like JSON decoding, and XML transformation has its own quirks.

Without structure, each importer rewrites the entire pipeline from scratch. The ordering is duplicated. One developer forgets the validation step. Another puts reporting before saving. The invariant -- "these steps happen in this order" -- isn't enforced anywhere.

The **Template Method** pattern solves this by defining the algorithm skeleton in one place and letting implementations fill in the variable steps. The pipeline structure is fixed: validate, parse, transform, save, report. Each data source provides its own implementation of the steps that vary. The ordering is guaranteed. The common flow is written once.

Go doesn't have abstract classes, but interfaces and composition achieve the same result idiomatically. In this article, we'll build a data ingestion framework where the pipeline is fixed and each source plugs in its own parsing and transformation logic.

## What Is the Template Method Pattern?

> "Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure."
> -- Gang of Four

The intent: **fix the overall algorithm structure while allowing specific steps to vary.** The "template" is the sequence of steps. Some steps are concrete (same for everyone). Some are abstract (each implementation provides its own version). The template method calls them in the right order -- implementations can't change the sequence, only the behavior of individual steps.

The key constraint: **the algorithm controls the flow.** Implementations don't call the template -- the template calls them. This is the "Hollywood Principle": don't call us, we'll call you.

Without this pattern, each data source reimplements the pipeline:

```go
func importCSV(path string) error {
    validate(path)
    records := parseCSV(path)
    for _, r := range records {
        transformed := transformCSVRecord(r)
        save(transformed)
    }
    report(len(records))
    return nil
}

func importJSON(url string) error {
    validate(url)
    records := fetchJSON(url)
    for _, r := range records {
        transformed := transformJSONRecord(r)
        save(transformed)
    }
    report(len(records))
    return nil
}
```

The structure is identical. Only the parsing and transformation differ. But the structure is *reimplemented* each time, with no guarantee that the next developer will keep the same order.

With the Template Method, the structure lives in one function and implementations plug into it.

## Core Components

Three participants, adapted to Go's idioms.

**Steps Interface** -- defines the customizable operations that each implementation must provide.

```go
type ImporterSteps interface {
    Validate(source string) error
    Parse(source string) ([]RawRecord, error)
    Transform(raw RawRecord) (Record, error)
    Name() string
}
```

**Template Function** -- the fixed algorithm skeleton. It calls the steps in order and handles the common logic (error handling, iteration, reporting).

```go
func RunImport(steps ImporterSteps, source string) error
```

**Concrete Implementations** -- each data source implements the Steps interface with its own parsing and transformation logic.

The flow:

```
Client calls RunImport(csvSteps, "data.csv")
  --> RunImport validates (via steps)
  --> RunImport parses (via steps)
  --> RunImport transforms each record (via steps)
  --> RunImport saves (common logic)
  --> RunImport reports (common logic)
```

The template function owns the flow. Implementations own the variable steps. Neither can change the other's responsibility.

## Code Walkthrough: A Data Ingestion Pipeline

We're building an import framework where the pipeline structure is fixed and each data source provides its own parsing and transformation.

### The Types

```go
type RawRecord map[string]string

type Record struct {
    ID    string
    Name  string
    Email string
}
```

`RawRecord` is the untyped intermediate form. `Record` is the clean, validated output.

### The Steps Interface

```go
type ImporterSteps interface {
    Validate(source string) error
    Parse(source string) ([]RawRecord, error)
    Transform(raw RawRecord) (Record, error)
    Name() string
}
```

Four methods. Each implementation provides source-specific logic for validation, parsing, and transformation. `Name()` identifies the importer for logging.

### The Template Function

This is the fixed algorithm -- the skeleton that never changes:

```go
func RunImport(steps ImporterSteps, source string) error {
    fmt.Printf("[%s] starting import from %s\n", steps.Name(), source)

    if err := steps.Validate(source); err != nil {
        return fmt.Errorf("[%s] validation failed: %w", steps.Name(), err)
    }
    fmt.Printf("[%s] validation passed\n", steps.Name())

    records, err := steps.Parse(source)
    if err != nil {
        return fmt.Errorf("[%s] parse failed: %w", steps.Name(), err)
    }
    fmt.Printf("[%s] parsed %d records\n", steps.Name(), len(records))

    var saved int
    for i, raw := range records {
        record, err := steps.Transform(raw)
        if err != nil {
            fmt.Printf("[%s] skipping record %d: %v\n", steps.Name(), i, err)
            continue
        }
        if err := saveRecord(record); err != nil {
            return fmt.Errorf("[%s] save failed at record %d: %w", steps.Name(), i, err)
        }
        saved++
    }

    fmt.Printf("[%s] import complete: %d/%d records saved\n", steps.Name(), saved, len(records))
    return nil
}

func saveRecord(r Record) error {
    fmt.Printf("  saved: {id=%s, name=%s, email=%s}\n", r.ID, r.Name, r.Email)
    return nil
}
```

The template owns: ordering, error handling, iteration, skip-on-error logic, reporting. It calls `steps.Validate`, `steps.Parse`, and `steps.Transform` at the right moments. Implementations can't reorder these calls or skip validation.

### Concrete Implementation: CSV Importer

```go
type CSVImporter struct{}

func (c *CSVImporter) Validate(source string) error {
    if !strings.HasSuffix(source, ".csv") {
        return fmt.Errorf("expected .csv file, got: %s", source)
    }
    return nil
}

func (c *CSVImporter) Parse(source string) ([]RawRecord, error) {
    return []RawRecord{
        {"id": "1", "name": "Alice", "email": "alice@example.com"},
        {"id": "2", "name": "Bob", "email": "bob@example.com"},
        {"id": "3", "name": "Charlie", "email": ""},
    }, nil
}

func (c *CSVImporter) Transform(raw RawRecord) (Record, error) {
    if raw["email"] == "" {
        return Record{}, fmt.Errorf("missing email for %s", raw["name"])
    }
    return Record{
        ID:    raw["id"],
        Name:  raw["name"],
        Email: raw["email"],
    }, nil
}

func (c *CSVImporter) Name() string { return "csv" }
```

### Concrete Implementation: JSON API Importer

```go
type JSONImporter struct{}

func (j *JSONImporter) Validate(source string) error {
    if !strings.HasPrefix(source, "https://") {
        return fmt.Errorf("source must be HTTPS URL, got: %s", source)
    }
    return nil
}

func (j *JSONImporter) Parse(source string) ([]RawRecord, error) {
    return []RawRecord{
        {"id": "101", "full_name": "Diana Prince", "contact": "diana@example.com"},
        {"id": "102", "full_name": "Bruce Wayne", "contact": "bruce@example.com"},
    }, nil
}

func (j *JSONImporter) Transform(raw RawRecord) (Record, error) {
    return Record{
        ID:    raw["id"],
        Name:  raw["full_name"],
        Email: raw["contact"],
    }, nil
}

func (j *JSONImporter) Name() string { return "json-api" }
```

Different field names (`full_name` vs `name`, `contact` vs `email`), different validation rules (file extension vs URL scheme), same pipeline structure.

### Running It

```go
func main() {
    fmt.Println("=== CSV Import ===")
    if err := RunImport(&CSVImporter{}, "users.csv"); err != nil {
        fmt.Println("FAILED:", err)
    }

    fmt.Println("\n=== JSON API Import ===")
    if err := RunImport(&JSONImporter{}, "https://api.example.com/users"); err != nil {
        fmt.Println("FAILED:", err)
    }
}
```

```
=== CSV Import ===
[csv] starting import from users.csv
[csv] validation passed
[csv] parsed 3 records
  saved: {id=1, name=Alice, email=alice@example.com}
  saved: {id=2, name=Bob, email=bob@example.com}
[csv] skipping record 2: missing email for Charlie
[csv] import complete: 2/3 records saved

=== JSON API Import ===
[json-api] starting import from https://api.example.com/users
[json-api] validation passed
[json-api] parsed 2 records
  saved: {id=101, name=Diana Prince, email=diana@example.com}
  saved: {id=102, name=Bruce Wayne, email=bruce@example.com}
[json-api] import complete: 2/2 records saved
```

Same pipeline, two different data sources. The CSV importer skips Charlie (missing email) -- the template's skip-on-error logic handles this generically. The JSON importer maps different field names. Neither implementation controls the flow -- `RunImport` does.

### Adding a Hook: Pre-Save Transformation

Sometimes implementations need an optional step. In Go, this is cleanly handled with an optional interface:

```go
type PreSaveHook interface {
    BeforeSave(r *Record)
}
```

The template checks for it:

```go
if hook, ok := steps.(PreSaveHook); ok {
    hook.BeforeSave(&record)
}
```

Implementations that want the hook implement it. Others don't. No empty methods, no forced implementation of optional behavior.

## Template Method in Go vs OOP Languages

In Java or C++, the Template Method uses an abstract base class with concrete and abstract methods. Subclasses inherit the template method and override the abstract steps.

Go doesn't have inheritance or abstract classes. Instead:

- The **template function** is a standalone function (not a method on a base struct).
- The **steps interface** replaces abstract methods.
- **Concrete implementations** are structs that satisfy the interface.
- **Hooks** are optional interfaces checked with type assertions.

This is actually *cleaner* than the inheritance version because the template function is explicit, the interface contract is clear, and there's no fragile base class problem. The Go version makes it obvious what's fixed (the function body) and what varies (the interface methods).

## Strategy vs Template Method

These two patterns both involve pluggable behavior, but they control flow differently.

**Template Method** fixes the algorithm structure and lets implementations fill in specific steps. The *template* controls when each step runs. Implementations can't reorder steps or skip them. The invariant is the *sequence*.

**Strategy** makes an entire algorithm interchangeable. The *context* delegates fully to the strategy -- it doesn't control the internal steps. Each strategy decides its own structure. The invariant is the *interface*, not the sequence.

The test: does the framework need to enforce step ordering? If yes (validate must happen before parse, transform must happen before save), it's Template Method. If each implementation is free to structure its work however it wants, it's Strategy.

## When to Use It

The Template Method fits when **multiple variants share the same algorithm structure but differ in specific steps**:

- **Data processing pipelines** (import, ETL, export) where the stages are fixed but the implementation per source/format varies.
- **Request handling frameworks** where authentication, parsing, validation, processing, and response formatting happen in a guaranteed order, but each handler provides its own processing logic.
- **Build/deploy systems** where the lifecycle (compile, test, package, deploy) is fixed but the tools used at each step vary per language or platform.
- **Testing frameworks** where setup, execute, assert, teardown always run in order, but each test provides its own execution and assertion logic.

Skip it when:

- **The algorithm structure itself varies.** If some importers need validation and others don't, or the step order differs per implementation, the template's rigidity works against you. Use Strategy instead.
- **There's only one implementation.** If you'll only ever have CSV import, the abstraction adds indirection without enabling reuse. Write the pipeline directly.
- **Steps are independent.** If the steps don't need to run in a specific order and can be composed freely, the Chain of Responsibility or middleware pattern is more flexible.

## Pros and Cons

**Pros:**

- **Guaranteed ordering** -- the algorithm structure is written once and enforced. No implementation can accidentally skip validation or reorder steps.
- **DRY orchestration** -- the common logic (error handling, iteration, reporting, skipping bad records) lives in one place. Implementations only provide what's unique to them.
- **Easy to add new variants** -- a new data source means a new struct implementing the steps interface. The template function doesn't change.
- **Testable at two levels** -- test the template with a mock implementation (verify ordering and error handling). Test each implementation independently (verify parsing and transformation logic).

**Cons:**

- **Rigid structure** -- the template enforces one way to run the algorithm. If a new requirement breaks the ordering (e.g., "parse before validate for this source"), the template either accommodates everyone or becomes a constraint.
- **Interface bloat for optional steps** -- if some implementations don't need certain steps, they still must implement the full interface (even if the method is a no-op). The optional interface pattern (hooks) mitigates this but adds type assertions.
- **Hidden control flow** -- a developer looking at a concrete implementation doesn't see the full picture. They must read the template function to understand when their methods are called and in what order.

## Best Practices

- **Make the template a function, not a method.** `RunImport(steps, source)` is clearer in Go than embedding the template in a base struct. There's no base class to inherit from -- a function with an interface parameter is the idiomatic Go equivalent.
- **Keep the steps interface minimal.** Only include methods that genuinely vary per implementation. If validation is the same for all importers, put it in the template directly -- don't force implementations to reimplement identical logic.
- **Use optional interfaces for hooks, not empty methods.** An implementation that doesn't need `BeforeSave` shouldn't implement a no-op `BeforeSave`. Use a type assertion in the template (`if hook, ok := steps.(PreSaveHook); ok`) to call it only on implementations that opt in.
- **Return errors from steps, handle them in the template.** The template decides the error policy (skip and continue, abort immediately, collect all errors). Implementations just report success or failure per step.

## Common Mistakes

**Putting orchestration logic inside implementations.** If `CSVImporter.Parse` calls `saveRecord` or reports progress, it's reached into the template's responsibility. Implementations provide data and behavior for individual steps -- they don't orchestrate the pipeline. The template owns the flow.

**Making the template too flexible.** A template that accepts options like `skipValidation: true` or `reverseOrder: true` is no longer enforcing a structure -- it's a configurable function. If different call sites need different orderings, the template's rigidity isn't appropriate. Use composition or Strategy instead.

**Conflating Template Method with Strategy.** If your "template" is a function that calls `steps.DoEverything()` and the implementation controls its own internal flow, that's Strategy, not Template Method. The distinguishing feature is that the *template* controls the step ordering, not the implementations.

**Not documenting when each step is called.** A developer implementing `ImporterSteps` needs to know: "Validate is called first with the source string. Parse is called next. Transform is called once per record after parsing." Without this documentation (or clear naming), they'll guess wrong about pre/post conditions.

## Final Thoughts

The Template Method pattern fixes an algorithm's structure while letting specific steps vary. The template owns the ordering, error policy, and orchestration. Implementations own the variable logic at each step. Neither can change the other's responsibility.

In Go, this translates to a function that accepts an interface -- the function is the template, the interface defines the variable steps. No base classes, no inheritance, no method overriding. Just a function calling interface methods in a fixed sequence. It's one of the most natural translations of a GoF pattern into Go's idioms.

The honest test: if multiple implementations share the same step sequence and you've seen developers accidentally reorder or skip steps, the Template Method centralizes that structure and prevents drift.

> **Fix the algorithm's skeleton. Let implementations fill in the blanks. Never let the blanks rearrange the skeleton.**
