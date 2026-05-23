+++
date = '2025-07-13'
title = 'Factory Method Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a document export system. Users can export reports as PDF, CSV, or Excel. The Simple Factory approach works at first: a `NewExporter("pdf")` function with a switch statement returns the right exporter. Then requirements grow. Each export format needs different initialization -- PDF needs page margins and font configuration, CSV needs delimiter settings, Excel needs sheet templates. The switch grows. Each format's setup logic is different enough that stuffing them all into one function gets messy. And every new format means modifying that function.

The deeper problem: the Simple Factory centralizes *all* creation decisions in one place. That works when the types are simple and the set is small. But when each type needs its own construction logic -- logic that only makes sense *for that type* -- you want to delegate. Let each exporter type own its own creation process. The caller says "give me an exporter" and the *creator* decides what to build and how to build it.

The **Factory Method** pattern does exactly this. Instead of one function with a switch, you have multiple creator types, each implementing the same factory interface but returning different products. Adding a new format means adding a new creator -- no existing code changes. The client works with the creator interface, never touching concrete types directly.

In this article, we'll build a document export system where each format has its own creator with custom initialization logic. Go's interfaces make the pattern natural, even without inheritance.

## What Is the Factory Method Pattern?

> "Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."
> -- Gang of Four

Reframed for Go: **define a creator interface with a factory method, and let each implementing struct decide which product to create.** The client calls the factory method on whatever creator it has, and gets back a product that satisfies the product interface. The client never knows or cares which concrete type it received.

The key difference from Simple Factory: **the decision of what to create is distributed, not centralized.** In a Simple Factory, one function owns all the creation logic. In Factory Method, each creator owns its own creation logic. This means adding a new product type doesn't require modifying any existing code -- you just add a new creator.

Without this pattern, export creation might live in one place:

```go
func NewExporter(format string) (Exporter, error) {
    switch format {
    case "pdf":
        return &PDFExporter{margins: defaultMargins, font: "Helvetica"}, nil
    case "csv":
        return &CSVExporter{delimiter: ','}, nil
    case "excel":
        return &ExcelExporter{template: loadTemplate("default")}, nil
    default:
        return nil, fmt.Errorf("unsupported format: %s", format)
    }
}
```

This function knows about PDF margins, CSV delimiters, and Excel templates. It's a dumping ground for every format's initialization details. The Factory Method pattern separates these so each format's creator handles only its own setup.

## Core Components

Four participants.

**Product** -- the interface that all created objects implement. The client depends on this.

```go
type Exporter interface {
    Export(data []byte) ([]byte, error)
    FileExtension() string
}
```

**ConcreteProduct** -- the actual implementations. PDF, CSV, Excel exporters, each with different internal state and behavior.

**Creator** -- the interface that declares the factory method. The client receives a creator and calls it.

```go
type ExporterCreator interface {
    CreateExporter() Exporter
}
```

**ConcreteCreator** -- each struct implements `CreateExporter()` and returns its specific product with its specific initialization.

The flow:

```txt
Client --> Creator.CreateExporter() --> Concrete Product (as Exporter interface) --> Client uses Exporter
```

The client picks a creator (or receives one via dependency injection), calls the factory method, and works with the product through the interface. Adding a new format means writing a new creator struct -- the client code doesn't change.

## Code Walkthrough: A Document Export System

We're building an export system where different formats require different initialization logic. Each creator knows how to set up its specific exporter.

### The Product Interface

```go
type Exporter interface {
    Export(data []byte) ([]byte, error)
    FileExtension() string
}
```

Two methods: export the data and tell the caller what file extension to use. Every exporter satisfies this contract.

### Concrete Products

**PDF Exporter** -- needs page margins and font configuration:

```go
type PDFExporter struct {
    marginMM int
    font     string
}

func (e *PDFExporter) Export(data []byte) ([]byte, error) {
    result := fmt.Sprintf("[PDF font=%s margin=%dmm] %s", e.font, e.marginMM, string(data))
    return []byte(result), nil
}

func (e *PDFExporter) FileExtension() string {
    return ".pdf"
}
```

**CSV Exporter** -- needs delimiter configuration:

```go
type CSVExporter struct {
    delimiter rune
}

func (e *CSVExporter) Export(data []byte) ([]byte, error) {
    result := fmt.Sprintf("[CSV delim='%c'] %s", e.delimiter, string(data))
    return []byte(result), nil
}

func (e *CSVExporter) FileExtension() string {
    return ".csv"
}
```

**Excel Exporter** -- needs a sheet template name:

```go
type ExcelExporter struct {
    template string
}

func (e *ExcelExporter) Export(data []byte) ([]byte, error) {
    result := fmt.Sprintf("[Excel template=%s] %s", e.template, string(data))
    return []byte(result), nil
}

func (e *ExcelExporter) FileExtension() string {
    return ".xlsx"
}
```

Each exporter has different fields and different initialization needs. This is exactly why a single factory function becomes unwieldy -- it has to know about all of these details.

### The Creator Interface

```go
type ExporterCreator interface {
    CreateExporter() Exporter
}
```

One method. Any struct that can produce an `Exporter` is a creator.

### Concrete Creators

Each creator encapsulates the initialization logic for its format:

```go
type PDFCreator struct {
    MarginMM int
    Font     string
}

func (c *PDFCreator) CreateExporter() Exporter {
    margin := c.MarginMM
    if margin == 0 {
        margin = 25
    }
    font := c.Font
    if font == "" {
        font = "Helvetica"
    }
    return &PDFExporter{marginMM: margin, font: font}
}
```

```go
type CSVCreator struct {
    Delimiter rune
}

func (c *CSVCreator) CreateExporter() Exporter {
    delim := c.Delimiter
    if delim == 0 {
        delim = ','
    }
    return &CSVExporter{delimiter: delim}
}
```

```go
type ExcelCreator struct {
    Template string
}

func (c *ExcelCreator) CreateExporter() Exporter {
    tmpl := c.Template
    if tmpl == "" {
        tmpl = "default"
    }
    return &ExcelExporter{template: tmpl}
}
```

Each creator handles its own defaults and configuration. The PDF creator knows about margins and fonts. The CSV creator knows about delimiters. They don't leak into each other or into a central function.

### The Client

The client receives a creator and uses it without knowing which format it's working with:

```go
func exportReport(creator ExporterCreator, data []byte) error {
    exporter := creator.CreateExporter()
    output, err := exporter.Export(data)
    if err != nil {
        return err
    }
    filename := "report" + exporter.FileExtension()
    fmt.Printf("Writing %s: %s\n", filename, string(output))
    return nil
}
```

`exportReport` doesn't import `PDFExporter`, `CSVExporter`, or `ExcelExporter`. It doesn't know about margins, delimiters, or templates. It works entirely through interfaces.

### Putting It Together

```go
func main() {
    data := []byte("Q4 Revenue: $1.2M")

    creators := []ExporterCreator{
        &PDFCreator{MarginMM: 20, Font: "Arial"},
        &CSVCreator{Delimiter: ';'},
        &ExcelCreator{},
    }

    for _, creator := range creators {
        if err := exportReport(creator, data); err != nil {
            fmt.Println("Error:", err)
        }
    }
}

// OUTPUT:
// Writing report.pdf: [PDF font=Arial margin=20mm] Q4 Revenue: $1.2M
// Writing report.csv: [CSV delim=';'] Q4 Revenue: $1.2M
// Writing report.xlsx: [Excel template=default] Q4 Revenue: $1.2M
```

Three different formats, three different creators, one client function. Adding a new format -- say, HTML -- means writing `HTMLExporter` and `HTMLCreator`. Nothing else changes.

## Factory Method vs Simple Factory

The distinction matters because the patterns solve different levels of the same problem.

**Simple Factory** centralizes creation in one function. You pass input, get back a product. The factory knows about all types. Adding a type means modifying the factory. Good for small, stable type sets.

**Factory Method** distributes creation across creator types. Each creator knows about one product. Adding a type means adding a creator. No existing code changes. Good for growing, extensible type sets.

The inflection point: when the factory function starts containing type-specific initialization logic (margins for PDF, delimiters for CSV), it's time to move to Factory Method. Each type's creator can own its own setup logic, defaults, and validation.

In Go terms: Simple Factory is one function with a switch. Factory Method is an interface with multiple implementations. Both return the same product interface, but the ownership of creation logic is different.

## Variations

### Function-Based Factory Method

For lightweight cases where a full creator struct feels heavy, use a function type:

```go
type ExporterFactory func() Exporter

func PDFFactory(margin int, font string) ExporterFactory {
    return func() Exporter {
        return &PDFExporter{marginMM: margin, font: font}
    }
}

func CSVFactory(delim rune) ExporterFactory {
    return func() Exporter {
        return &CSVExporter{delimiter: delim}
    }
}
```

Usage:

```go
factory := PDFFactory(25, "Helvetica")
exporter := factory()
```

This is more idiomatic Go for cases where the creator doesn't need state beyond what's captured in the closure. The interface-based approach is better when creators have methods beyond the factory method itself (e.g., `Validate()`, `Description()`).

### Registration-Based Selection

Combine Factory Method with a registry for runtime selection:

```go
var creators = map[string]ExporterCreator{
    "pdf":   &PDFCreator{MarginMM: 25, Font: "Helvetica"},
    "csv":   &CSVCreator{Delimiter: ','},
    "excel": &ExcelCreator{Template: "default"},
}

func GetCreator(format string) (ExporterCreator, error) {
    c, ok := creators[format]
    if !ok {
        return nil, fmt.Errorf("unsupported format: %s", format)
    }
    return c, nil
}
```

The registry maps names to creators. Each creator still owns its initialization logic. Adding a format means adding a map entry and a creator struct -- the `GetCreator` function stays unchanged.

## When to Use It

The Factory Method pattern fits when **creation logic varies per type and needs to be extensible without modifying existing code**:

- **Multiple product types with different initialization needs.** If PDF needs margins, CSV needs delimiters, and Excel needs templates, each creator encapsulates its own setup rather than dumping everything into one function.
- **Plugin-style architectures** where new types are added by third parties or separate teams. Each team writes their own creator without touching the core system.
- **Testing with mock creators.** Pass a `MockCreator` that returns a `MockExporter` and test the client's logic without real file I/O or network calls.
- **Frameworks and libraries** where the library defines the creator interface and users implement it for their specific types.

Skip it when:

- **All types have trivial, identical initialization.** If every exporter is just `&TypeExporter{}` with no configuration, a Simple Factory's switch statement is simpler and more direct.
- **The type set is small and stable.** Two or three types that will never change don't justify the extra abstraction layer.
- **You're the only consumer.** If there's no extensibility requirement and no testing benefit, the pattern adds boilerplate for zero return.

## Pros and Cons

**Pros:**

- **Open/Closed compliance** -- new product types are added by writing new creators. Existing creators, the product interface, and client code don't change.
- **Encapsulated initialization** -- each creator owns its type's configuration, defaults, and validation. No leaking of PDF margins into a function that also handles CSV delimiters.
- **Testability** -- inject a mock creator, get a mock product. Test client logic without real dependencies.
- **Polymorphic composition** -- a slice of `ExporterCreator` can contain PDF, CSV, Excel, and HTML creators. Loop over them, call `CreateExporter()`, and the right thing happens. No switch, no type assertions.

**Cons:**

- **More types per product** -- every new format needs both a product struct *and* a creator struct. For simple products, this doubling feels heavy.
- **Indirection** -- the client calls a creator to get a product, rather than getting the product directly. Tracing "where was this exporter created?" requires finding the specific creator.
- **Less natural in Go than OOP languages** -- the pattern was designed for class hierarchies with inheritance. In Go, you emulate it with interfaces and composition, which works well but doesn't have the language support (abstract methods, protected constructors) that makes it ergonomic in Java or C#.

## Best Practices

- **Keep creators focused on creation.** A creator's `CreateExporter()` method should construct and return. If it's also doing business logic, validation of input data, or I/O, it's doing too much. Configuration and defaults are fine; business logic is not.
- **Use the function-based variant for simple cases.** If the creator has no state beyond the factory method and no other methods, a `func() Exporter` closure is lighter than a struct with one method. Graduate to structs when the creator needs configuration fields or additional methods.
- **Combine with a registry for runtime selection.** Factory Method alone doesn't solve "which creator do I pick based on user input." A map of creators bridges the gap -- the map handles selection, the creator handles construction.
- **Return errors from the factory method when construction can fail.** `CreateExporter() (Exporter, error)` is more honest than panicking inside the creator. If loading a template file or connecting to a service can fail, surface it.

## Common Mistakes

**Confusing Factory Method with Simple Factory.** If your "factory method" is a standalone function with a switch statement, that's a Simple Factory. Factory Method specifically means a *method on a creator type* that subtype-specific implementations override. The distinction is distribution of responsibility, not just "a function that creates things."

**Creating god creators.** A creator that queries a database, fetches config from an API, validates business rules, and then constructs the product has taken on too many responsibilities. The creator should receive what it needs (via its own fields or constructor) and create the product. Heavy setup logic should happen before the creator is used, not inside `CreateExporter()`.

**Using Factory Method when types don't vary in creation logic.** If every creator's `CreateExporter()` is just `return &TypeExporter{}` with no configuration, defaults, or validation, you've added an abstraction layer that doesn't do anything. The pattern earns its keep when creation *logic* differs between types, not just when the returned *type* differs.

**Not providing a way to select creators at runtime.** Factory Method gives you polymorphic creation, but someone still needs to decide which creator the client gets. Without a registry, config-based selection, or dependency injection framework, the pattern just moves the switch statement from "which product" to "which creator" -- which isn't necessarily an improvement.

## Final Thoughts

The Factory Method pattern distributes creation responsibility so that each type owns its own construction logic. The client works with a creator interface, calls the factory method, and gets back a product -- without knowing or caring which concrete type is behind it. Adding new types means adding new creators, and existing code stays untouched.

In Go, the pattern maps onto interfaces and struct implementations. It's slightly more verbose than in languages with abstract classes and inheritance, but the result is the same: polymorphic creation without centralized switches. When combined with a registry for runtime selection, it gives you both the extensibility of distributed creators and the convenience of name-based lookup.

The honest test: if your Simple Factory's switch cases each have meaningfully different initialization logic, and you expect the type set to grow, Factory Method is the natural next step. If the switch is simple and stable, keep it simple.

> **Let each type decide how it's built. Your job is just to ask for one.**
