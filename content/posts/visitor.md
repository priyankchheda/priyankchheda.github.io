+++
date = '2026-06-14'
title = 'Visitor Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about an infrastructure-as-code tool that manages a collection of resources: servers, databases, load balancers, DNS records. The resource types are stable -- they rarely change. But the operations on them keep growing. First you need a cost calculator. Then a compliance auditor. Then a Terraform exporter. Then a dependency graph builder. Then a capacity planner.

Each new operation means modifying every resource type -- adding `EstimateCost()`, `Audit()`, `ExportTF()`, `Dependencies()` to Server, Database, LoadBalancer, and DNSRecord. Four resource types, five operations, twenty methods to write. And every time you add operation number six, you're touching four files that were already stable and tested.

The **Visitor pattern** flips the relationship. Instead of putting every operation inside every resource, you put each operation in its own visitor object. The visitor knows how to operate on each resource type. Resources just accept visitors -- they don't know or care what operation is being performed. Adding a new operation means writing one new visitor. No resource types change.

This is one of the more complex GoF patterns, and one of the few where Go's lack of method overloading makes the implementation feel slightly different from the textbook version. In this article, we'll build an infrastructure cost calculator, compliance auditor, and Terraform exporter -- each as a separate visitor over the same resource collection.

## What Is the Visitor Pattern?

> "Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates."
> -- Gang of Four

The intent: **separate operations from the objects they operate on, so new operations can be added without modifying the object hierarchy.** The objects (elements) have a fixed structure. The operations (visitors) are the axis of change. Each visitor implements the operation for every element type. Elements just "accept" the visitor and let it do its work.

The mechanism is **double dispatch**. When client code calls `resource.Accept(visitor)`, two things are determined: which resource type (resolved by the `Accept` method dispatch), and which visitor type (resolved inside `Accept` when it calls `visitor.VisitServer(self)` or `visitor.VisitDatabase(self)`). The combination determines the exact behavior.

Without the pattern, operations accumulate on the elements:

```go
type Server struct { /* ... */ }
func (s *Server) EstimateCost() float64 { /* ... */ }
func (s *Server) Audit() []Finding { /* ... */ }
func (s *Server) ExportTF() string { /* ... */ }
func (s *Server) Dependencies() []string { /* ... */ }
```

Every new operation means four new methods (one per resource type). Every new resource type means adding methods for every existing operation. The growth is multiplicative.

With Visitor, elements are thin and operations live separately:

```go
func (s *Server) Accept(v Visitor) { v.VisitServer(s) }
```

One method per element. Operations grow independently.

## Core Components

Four participants.

**Visitor** -- the interface declaring a visit method for each element type.

```go
type Visitor interface {
    VisitServer(s *Server)
    VisitDatabase(d *Database)
    VisitLoadBalancer(lb *LoadBalancer)
}
```

**ConcreteVisitor** -- implements the operation for each element type. Each visitor is one operation (cost calculation, auditing, export).

**Element** -- the interface declaring `Accept(Visitor)`. Every resource type implements this.

```go
type Resource interface {
    Accept(v Visitor)
    Name() string
}
```

**ConcreteElement** -- the actual resource types. Each implements `Accept` by calling the appropriate visit method on the visitor, passing itself.

The flow (double dispatch):

```
Client calls resource.Accept(visitor)
  --> Server.Accept(v) calls v.VisitServer(self)
    --> CostVisitor.VisitServer(s) calculates server cost
```

Two decisions: which element's `Accept` runs (first dispatch), which visitor method gets called (second dispatch).

## Code Walkthrough: Infrastructure Resource Visitors

We're building a system where infrastructure resources can be visited by different operations -- cost estimation, compliance auditing, and Terraform export -- without the resources knowing about any of these operations.

### The Element Interface and Types

```go
type Resource interface {
    Accept(v Visitor)
    Name() string
}

type Server struct {
    Hostname string
    CPUs     int
    MemoryGB int
    Region   string
}

func (s *Server) Accept(v Visitor) { v.VisitServer(s) }
func (s *Server) Name() string     { return s.Hostname }

type Database struct {
    Engine    string
    StorageGB int
    Replicas  int
    Region    string
}

func (d *Database) Accept(v Visitor) { v.VisitDatabase(d) }
func (d *Database) Name() string     { return d.Engine + "-db" }

type LoadBalancer struct {
    Algorithm string
    Backends  int
    Region    string
}

func (lb *LoadBalancer) Accept(v Visitor) { v.VisitLoadBalancer(lb) }
func (lb *LoadBalancer) Name() string     { return "lb-" + lb.Region }
```

Each resource type has one tiny `Accept` method that calls the appropriate visitor method. That's the only pattern-specific code on the elements. The resources carry data; they don't contain operations.

### The Visitor Interface

```go
type Visitor interface {
    VisitServer(s *Server)
    VisitDatabase(d *Database)
    VisitLoadBalancer(lb *LoadBalancer)
}
```

One method per element type. Every visitor must know how to handle every resource type.

### Visitor 1: Cost Estimator

```go
type CostEstimator struct {
    total float64
}

func (c *CostEstimator) VisitServer(s *Server) {
    cost := float64(s.CPUs)*50.0 + float64(s.MemoryGB)*5.0
    fmt.Printf("  server %s: $%.2f/mo\n", s.Hostname, cost)
    c.total += cost
}

func (c *CostEstimator) VisitDatabase(d *Database) {
    cost := float64(d.StorageGB)*0.10 + float64(d.Replicas)*200.0
    fmt.Printf("  database %s: $%.2f/mo\n", d.Engine, cost)
    c.total += cost
}

func (c *CostEstimator) VisitLoadBalancer(lb *LoadBalancer) {
    cost := float64(lb.Backends) * 15.0
    fmt.Printf("  lb %s: $%.2f/mo\n", lb.Region, cost)
    c.total += cost
}

func (c *CostEstimator) Total() float64 {
    return c.total
}
```

The cost estimator accumulates a total as it visits each resource. Different resource types have different pricing formulas. The formula for servers (`CPUs*50 + MemoryGB*5`) is completely separate from the formula for databases (`StorageGB*0.10 + Replicas*200`).

### Visitor 2: Compliance Auditor

```go
type ComplianceAuditor struct {
    findings []string
}

func (a *ComplianceAuditor) VisitServer(s *Server) {
    if s.MemoryGB < 4 {
        a.findings = append(a.findings, fmt.Sprintf("%s: memory below minimum (4GB required)", s.Hostname))
    }
}

func (a *ComplianceAuditor) VisitDatabase(d *Database) {
    if d.Replicas < 2 {
        a.findings = append(a.findings, fmt.Sprintf("%s: insufficient replicas (minimum 2)", d.Engine))
    }
}

func (a *ComplianceAuditor) VisitLoadBalancer(lb *LoadBalancer) {
    if lb.Backends < 2 {
        a.findings = append(a.findings, fmt.Sprintf("lb-%s: single backend is a SPOF", lb.Region))
    }
}

func (a *ComplianceAuditor) Findings() []string {
    return a.findings
}
```

Same resources, completely different operation. The auditor checks rules per type and collects findings.

### Visitor 3: Terraform Exporter

```go
type TerraformExporter struct {
    output strings.Builder
}

func (t *TerraformExporter) VisitServer(s *Server) {
    fmt.Fprintf(&t.output, "resource \"aws_instance\" %q {\n  instance_type = \"custom-%d-%d\"\n  region = %q\n}\n\n",
        s.Hostname, s.CPUs, s.MemoryGB*1024, s.Region)
}

func (t *TerraformExporter) VisitDatabase(d *Database) {
    fmt.Fprintf(&t.output, "resource \"aws_rds_instance\" %q {\n  engine = %q\n  storage = %d\n  replicas = %d\n}\n\n",
        d.Engine+"-db", d.Engine, d.StorageGB, d.Replicas)
}

func (t *TerraformExporter) VisitLoadBalancer(lb *LoadBalancer) {
    fmt.Fprintf(&t.output, "resource \"aws_lb\" %q {\n  algorithm = %q\n  target_count = %d\n}\n\n",
        "lb-"+lb.Region, lb.Algorithm, lb.Backends)
}

func (t *TerraformExporter) Export() string {
    return t.output.String()
}
```

A third operation on the same resources. No resource type was modified to add cost estimation, compliance auditing, or Terraform export.

### Running It

```go
func main() {
    resources := []Resource{
        &Server{Hostname: "api-1", CPUs: 4, MemoryGB: 16, Region: "us-east-1"},
        &Server{Hostname: "worker-1", CPUs: 8, MemoryGB: 32, Region: "us-east-1"},
        &Database{Engine: "postgres", StorageGB: 500, Replicas: 1, Region: "us-east-1"},
        &LoadBalancer{Algorithm: "round-robin", Backends: 3, Region: "us-east-1"},
    }

    fmt.Println("=== Cost Estimate ===")
    cost := &CostEstimator{}
    for _, r := range resources {
        r.Accept(cost)
    }
    fmt.Printf("  TOTAL: $%.2f/mo\n", cost.Total())

    fmt.Println("\n=== Compliance Audit ===")
    audit := &ComplianceAuditor{}
    for _, r := range resources {
        r.Accept(audit)
    }
    for _, f := range audit.Findings() {
        fmt.Printf("  FINDING: %s\n", f)
    }

    fmt.Println("\n=== Terraform Export ===")
    tf := &TerraformExporter{}
    for _, r := range resources {
        r.Accept(tf)
    }
    fmt.Print(tf.Export())
}
```

```
=== Cost Estimate ===
  server api-1: $280.00/mo
  server worker-1: $560.00/mo
  database postgres: $250.00/mo
  lb us-east-1: $45.00/mo
  TOTAL: $1135.00/mo

=== Compliance Audit ===
  FINDING: postgres: insufficient replicas (minimum 2)

=== Terraform Export ===
resource "aws_instance" "api-1" {
  instance_type = "custom-4-16384"
  region = "us-east-1"
}

resource "aws_instance" "worker-1" {
  instance_type = "custom-8-32768"
  region = "us-east-1"
}

resource "aws_rds_instance" "postgres-db" {
  engine = "postgres"
  storage = 500
  replicas = 1
}

resource "aws_lb" "lb-us-east-1" {
  algorithm = "round-robin"
  target_count = 3
}
```

Three operations, four resource types, zero modifications to the resource structs after the initial `Accept` method. Adding a fourth operation (dependency grapher, capacity planner, documentation generator) means writing one new visitor struct. The resources stay untouched.

## Why Double Dispatch Matters

In Go (and most languages), a method call dispatches on one type -- the receiver. When you call `resource.Accept(visitor)`, Go resolves which `Accept` to call based on the concrete type of `resource` (Server, Database, or LoadBalancer). That's single dispatch.

Inside `Accept`, the element calls back into the visitor: `v.VisitServer(self)`. Now Go resolves which `VisitServer` to call based on the concrete type of `v` (CostEstimator, ComplianceAuditor, or TerraformExporter). That's the second dispatch.

The combination of both dispatches determines the exact behavior: "cost estimation for a server" or "compliance audit for a database." Without the `Accept` indirection, you'd need type switches:

```go
switch r := resource.(type) {
case *Server:
    visitor.VisitServer(r)
case *Database:
    visitor.VisitDatabase(r)
}
```

The `Accept` method eliminates this switch by letting each element type route to the correct visitor method.

## When to Use It

The Visitor pattern fits when **the element types are stable but operations on them keep growing**:

- **AST processing** in compilers or linters where node types (expressions, statements, declarations) are fixed but operations (type checking, optimization, code generation, pretty printing) proliferate.
- **Infrastructure tooling** where resource types are defined once but operations (cost, audit, export, validate, graph) are added over time.
- **Document processing** where document element types (paragraph, heading, image, table) are stable but output formats (HTML, PDF, Markdown, LaTeX) grow.
- **Reporting and analytics** where data model entities are fixed but reports, aggregations, and visualizations change frequently.

Skip it when:

- **Element types change frequently.** Adding a new element type (e.g., a new resource `CDN`) means adding a method to *every* visitor. If you add elements more often than operations, Visitor makes changes harder, not easier.
- **The operation count is small and stable.** If there are only two operations and they won't grow, putting methods directly on the elements is simpler and more readable.
- **Elements need private access.** Visitors operate on elements' public API. If the operation needs deep access to private state, the Visitor either breaks encapsulation or the element must expose too much.

## Pros and Cons

**Pros:**

- **Open for new operations** -- add a new visitor without touching any element type. The resource structs are closed for modification but open for extension through visitors.
- **Operation logic is cohesive** -- all cost calculation logic lives in `CostEstimator`, not scattered across four element types. Each visitor is one focused concern.
- **Accumulating state across elements** -- visitors can carry state (like `total` in CostEstimator or `findings` in ComplianceAuditor) that accumulates as they visit multiple elements. Methods on elements can't easily do this.
- **Separates data from operations** -- elements hold data and structure. Visitors hold algorithms. Clean separation of responsibilities.

**Cons:**

- **Adding new element types is expensive** -- a new element means adding a method to every existing visitor. If you have 10 visitors and add one element type, that's 10 methods to write.
- **Visitor interface grows with elements** -- the `Visitor` interface has one method per element type. Five element types means five methods, each visitor must implement all five, and the interface is coupled to the full element hierarchy.
- **Double dispatch is non-obvious** -- the `Accept`/`Visit` indirection confuses developers who encounter it for the first time. The control flow isn't linear, and understanding "who calls whom" requires tracing through two layers.

## Best Practices

- **Keep elements focused on data, not behavior.** The element's `Accept` method should be the only pattern-specific code on it. If elements start accumulating visitor-specific helpers, the separation is eroding.
- **Use concrete visitor types, not the Visitor interface, in client code.** Clients create `&CostEstimator{}`, use it, then call `cost.Total()`. The `Visitor` interface is for the elements' `Accept` method, not for the client's API. Visitors often have result methods (`Total()`, `Findings()`, `Export()`) that aren't on the interface.
- **Consider returning results from Visit methods when appropriate.** The classic pattern uses `void` visit methods with accumulated state. In Go, returning values (`VisitServer(s *Server) float64`) can be cleaner for simple operations -- though it complicates the interface when visitors produce different return types.
- **Use type switches instead of Visitor when the element set is tiny.** For 2-3 types with simple operations, a `switch r := resource.(type)` is more readable than the full Accept/Visit ceremony. Visitor earns its keep at scale (5+ types, growing operations).

## Common Mistakes

**Adding Accept to interfaces that shouldn't have it.** If your elements are already behind a rich interface with meaningful methods, cramming `Accept(Visitor)` into that interface pollutes it with pattern machinery. Consider a separate `Visitable` interface that elements optionally satisfy, or use type switches for the small cases.

**Visitors that modify elements.** The pattern works best when visitors *read* elements and *produce results* (calculations, strings, findings). A visitor that mutates elements blurs the line between the Visitor pattern and direct method calls, and makes reasoning about state changes harder.

**Not providing a traversal mechanism.** The pattern defines how visitors operate on *individual* elements, but not how they're applied across a *collection*. In our walkthrough, the client loops over `[]Resource`. For tree structures (ASTs, file systems), you need a separate traversal mechanism (recursion, iterator) that calls `Accept` on each node.

**Over-applying Visitor to dynamic element hierarchies.** If your element types change every sprint (new resource types, new AST nodes), Visitor forces you to update every visitor on every change. The pattern assumes elements are *stable*. If they're not, Strategy or type switches are more maintainable.

## Final Thoughts

The Visitor pattern separates operations from the objects they operate on. Elements accept visitors. Visitors implement the operation for each element type. New operations are new visitors -- no elements change. It's the right tool when the data structure is stable and operations on it keep growing.

Go's lack of method overloading means the visitor interface lists each element type explicitly (`VisitServer`, `VisitDatabase`, `VisitLoadBalancer`). This is more verbose than languages with overloading, but it's also more explicit -- you can see exactly which element types the visitor handles by reading the interface.

The honest test: if you find yourself adding the same method to multiple types repeatedly (cost calculation today, audit tomorrow, export next week), and those types rarely change, Visitor centralizes each operation in one place instead of scattering it across every type.

> **When operations grow faster than types, let visitors carry the behavior and elements carry the data.**
