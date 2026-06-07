+++
date = '2026-02-27'
title = 'Interpreter Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Picture a monitoring system. At first, alerting is simple: fire an alert if CPU usage exceeds 90%. One threshold, one check, one `if` statement. Then the ops team wants compound alerts -- CPU above 90% *and* memory above 80%. Then region-specific rules. Then rules that come from a YAML config file and change without redeploying. Before long, the alerting logic is a tangled `if/else` tree that nobody wants to touch, and every new rule means another deploy.

The **Interpreter pattern** takes a different approach. Instead of encoding each rule as control flow, you model the grammar itself -- the `AND`, `OR`, `GreaterThan`, `LessThan` building blocks -- as objects. Each object knows how to evaluate itself. Compose them into a tree, call `Interpret()` on the root, and the evaluation cascades down recursively. The tree is the rule. The rule is data. And data can come from anywhere.

This is one of the more specialized patterns in the GoF catalog. Most codebases won't need it. But when the problem is right -- dynamic rules, small grammar, varying contexts -- it replaces hundreds of lines of brittle conditionals with a handful of composable structs. In this article, we'll build a metrics alerting rule engine in Go, then extend it to load rules from configuration. Go's interfaces and recursive composition make the whole thing surprisingly compact.

## What Is the Interpreter Pattern?

> "Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language."<br>- Gang of Four

"Language" here doesn't mean Go or Python. It means any small, structured set of rules: boolean expressions like `cpu > 90 AND memory > 80`, arithmetic like `price * quantity - discount`, validation rules like `field != "" OR hasDefault`. If there's a grammar -- a set of rules for how pieces combine -- the Interpreter pattern can model it.

The core idea: **each grammar rule becomes a struct, and each struct knows how to evaluate itself.** Terminal expressions (the leaves) read values from a context. Non-terminal expressions (the branches) combine their children's results. Call `Interpret()` on the root, and the whole tree evaluates through recursion. No central evaluator function. No type switches. The objects *are* the interpreter.

One thing that trips people up: the Interpreter pattern is not about parsing. Turning the string `"cpu > 90 AND memory > 80"` into an object tree is a separate problem. The pattern starts *after* you have the tree. How you build it -- manually in code, from JSON config, from a parser -- is up to you.

Without this pattern, rule evaluation usually looks like this:

```go
func shouldAlert(metrics Metrics) bool {
    switch alertRule {
    case "high_cpu":
        return metrics.CPU > 90
    case "high_cpu_and_memory":
        return metrics.CPU > 90 && metrics.Memory > 80
    case "disk_critical_or_cpu":
        return metrics.Disk > 95 || metrics.CPU > 90
    }
    return false
}
```

Every new rule is a new case. Rules can't be composed at runtime. Testing means exercising the entire switch. The Interpreter pattern replaces this with small, composable objects that each handle one piece of the grammar.

## Core Components

Four participants.

**Expression** -- the interface. Every grammar rule implements it.

```go
type Expression interface {
    Interpret(ctx Context) bool
}
```

**TerminalExpression** -- a leaf node that reads from the context and returns a result. It doesn't contain other expressions. "Is CPU above 90?" is a terminal expression.

**NonTerminalExpression** -- a branch node that holds child expressions and combines their results. AND, OR, NOT are all non-terminal.

**Context** -- the data used during evaluation. Metric values, user attributes, environment variables. The context changes between evaluations; the expression tree typically doesn't.

Visually, the rule `(cpu > 90 AND memory > 80) OR disk > 95` becomes:

```
          OR
        /    \
      AND    disk>95
     /   \
 cpu>90  memory>80
```

Each node is an `Expression`. Calling `Interpret()` on the `OR` node triggers the entire tree recursively. Non-terminals delegate to their children. Terminals read from the context. Results bubble up.

If this structure reminds you of the Composite pattern, that's not a coincidence. The Interpreter pattern uses Composite internally -- `Expression` is the Component, terminal expressions are Leaves, non-terminals are Composites. The difference is intent: Composite is about uniform part-whole treatment, Interpreter is about evaluating grammar. But the recursive tree structure is identical.

## Code Walkthrough: A Metrics Alert Engine

We're building an alerting rule engine. Rules like "alert if CPU is above 90 and memory is above 80" are represented as expression trees and evaluated against live metric snapshots.

The context holds metric values:

```go
type Context map[string]float64
```

The expression interface:

```go
type Expression interface {
    Interpret(ctx Context) bool
}
```

### Terminal Expressions

A **threshold expression** that checks if a metric exceeds a value:

```go
type GreaterThan struct {
    metric    string
    threshold float64
}

func (g *GreaterThan) Interpret(ctx Context) bool {
    v, ok := ctx[g.metric]
    if !ok {
        return false
    }
    return v > g.threshold
}
```

A missing metric evaluates to `false` rather than panicking. In a monitoring system, a metric might not be reported yet, or a rule might reference a metric that only some hosts emit. Defensive defaults matter here.

A **LessThan** for the opposite direction:

```go
type LessThan struct {
    metric    string
    threshold float64
}

func (l *LessThan) Interpret(ctx Context) bool {
    v, ok := ctx[l.metric]
    if !ok {
        return false
    }
    return v < l.threshold
}
```

Both terminals are self-contained. They don't know about AND, OR, or any other expression type. They answer one question about the context and nothing more.

### Non-Terminal Expressions

**AND** -- true only if both children are true:

```go
type And struct {
    left  Expression
    right Expression
}

func (a *And) Interpret(ctx Context) bool {
    return a.left.Interpret(ctx) && a.right.Interpret(ctx)
}
```

**OR** -- true if either child is true:

```go
type Or struct {
    left  Expression
    right Expression
}

func (o *Or) Interpret(ctx Context) bool {
    return o.left.Interpret(ctx) || o.right.Interpret(ctx)
}
```

**NOT** -- inverts its child:

```go
type Not struct {
    expr Expression
}

func (n *Not) Interpret(ctx Context) bool {
    return !n.expr.Interpret(ctx)
}
```

The fields are `Expression`, not concrete types. `And` doesn't care whether its children are `GreaterThan`, `Not`, or another `And`. This is what makes the pattern composable -- any expression plugs into any slot.

### Evaluating Alerts

The rule: alert if CPU is above 90 *and* memory is above 80, *or* if disk is above 95.

```go
func main() {
    rule := &Or{
        left: &And{
            left:  &GreaterThan{"cpu", 90},
            right: &GreaterThan{"memory", 80},
        },
        right: &GreaterThan{"disk", 95},
    }

    snapshot := Context{
        "cpu":    92.5,
        "memory": 85.0,
        "disk":   60.0,
    }

    fmt.Println(rule.Interpret(snapshot))
}
```

```
true
```

The evaluation walks the tree: `Or` asks `And` and `disk > 95`. `And` asks `cpu > 90` (true) and `memory > 80` (true). `And` returns true. `disk > 95` returns false. `true OR false` is true. Alert fires.

Same rule, different snapshot:

```go
snapshot2 := Context{
    "cpu":    45.0,
    "memory": 30.0,
    "disk":   97.0,
}
fmt.Println(rule.Interpret(snapshot2))
```

```
true
```

CPU and memory are fine, but disk is critical. The OR branch catches it. Same rule, different data, different evaluation path.

And a quiet system:

```go
snapshot3 := Context{
    "cpu":    45.0,
    "memory": 30.0,
    "disk":   60.0,
}
fmt.Println(rule.Interpret(snapshot3))
```

```
false
```

No alert. The rule object didn't change across any of these evaluations. The context did.

### Building Rules from Configuration

Building trees by hand in `main()` demonstrates the pattern, but the real power is that rules can come from *outside your code*. Here's a minimal rule builder that turns a JSON-like config into an expression tree:

```go
type RuleConfig struct {
    Op       string       `json:"op"`
    Metric   string       `json:"metric,omitempty"`
    Value    float64      `json:"value,omitempty"`
    Children []RuleConfig `json:"children,omitempty"`
}

func BuildRule(cfg RuleConfig) (Expression, error) {
    switch cfg.Op {
    case "gt":
        return &GreaterThan{cfg.Metric, cfg.Value}, nil
    case "lt":
        return &LessThan{cfg.Metric, cfg.Value}, nil
    case "and", "or":
        if len(cfg.Children) < 2 {
            return nil, fmt.Errorf("%s requires 2 children, got %d", cfg.Op, len(cfg.Children))
        }
        left, err := BuildRule(cfg.Children[0])
        if err != nil {
            return nil, err
        }
        right, err := BuildRule(cfg.Children[1])
        if err != nil {
            return nil, err
        }
        if cfg.Op == "and" {
            return &And{left, right}, nil
        }
        return &Or{left, right}, nil
    case "not":
        if len(cfg.Children) < 1 {
            return nil, fmt.Errorf("not requires 1 child, got 0")
        }
        child, err := BuildRule(cfg.Children[0])
        if err != nil {
            return nil, err
        }
        return &Not{child}, nil
    default:
        return nil, fmt.Errorf("unknown op: %s", cfg.Op)
    }
}
```

Now rules live in a config file:

```json
{
    "op": "or",
    "children": [
        {
            "op": "and",
            "children": [
                {"op": "gt", "metric": "cpu", "value": 90},
                {"op": "gt", "metric": "memory", "value": 80}
            ]
        },
        {"op": "gt", "metric": "disk", "value": 95}
    ]
}
```

Parse the JSON, call `BuildRule`, and you get the same expression tree -- without touching Go code. The ops team edits a config file, the rule engine picks it up, and alerting behavior changes without a deploy. This is where the Interpreter pattern earns its keep in production.

### Extending the Grammar

Need an `Equals` check? Add one struct:

```go
type Equals struct {
    metric string
    value  float64
}

func (e *Equals) Interpret(ctx Context) bool {
    v, ok := ctx[e.metric]
    if !ok {
        return false
    }
    return v == e.value
}
```

Add a `case "eq"` to `BuildRule`. No other code changes. `And`, `Or`, `Not`, `GreaterThan`, `LessThan` -- none of them know or care that `Equals` exists. It implements `Expression` and slots into any tree.

## The Pattern Hiding in Go's Standard Library

Go's own `go/ast` package is the Interpreter pattern at scale. When the Go compiler parses source code, it builds an abstract syntax tree where every node -- `BinaryExpr`, `UnaryExpr`, `BasicLit`, `Ident` -- implements the `ast.Node` interface. Tools like `go vet`, `gofmt`, and `golangci-lint` walk this tree and evaluate expressions against it.

The structure is exactly what we built: an interface, terminal nodes (literals, identifiers), non-terminal nodes (binary expressions, function calls), and recursive evaluation. The grammar is just much larger. That's the Interpreter pattern operating at the heart of Go's toolchain.

## When to Use It

The pattern fits when you have a **small, stable grammar** evaluated dynamically:

- **Alerting and monitoring rules** -- threshold conditions composed with AND/OR logic, loaded from config.
- **Access control policies** -- "allow if role is admin OR (role is editor AND resource is owned)."
- **Feature flag targeting** -- "enable for premium users in EU with account age > 90 days."
- **Validation rules** composed from reusable pieces rather than hardcoded per endpoint.

Skip it when:

- **The grammar is large or evolving rapidly.** Dozens of expression types with precedence rules and operator overloading means you need a parser generator, not the Interpreter pattern.
- **Performance is critical.** Recursive evaluation through many small objects has overhead. For hot-path evaluation at high frequency, a compiled or stack-based approach will be faster.
- **A simple conditional would suffice.** If the rule is `if cpu > 90` and it will never get more complex, the pattern adds indirection for nothing.

The decision heuristic: **are you modeling rules, or just writing logic?** Dynamic, composable, reusable rules that change without deploys -- Interpreter. Static, one-off conditions -- plain code.

## Pros and Cons

**Pros:**

- **Extensibility** -- new grammar rules are new structs. Existing expressions don't change. One of the cleanest demonstrations of the Open/Closed Principle.
- **Testability** -- each expression is a tiny, independent unit. Test `GreaterThan` with a simple context, test `And` with stub children, test the full tree for integration.
- **Dynamic composition** -- rules built at runtime from config, a database, or an API. The same tree structure works whether hardcoded or assembled from JSON.
- **Separation of rule definition from evaluation** -- the tree is built once and evaluated many times against different contexts. Rule structure and runtime data stay cleanly separated.

**Cons:**

- **Struct proliferation** -- every grammar rule needs its own type. Fifteen operators means fifteen structs, each small and similar-looking.
- **Recursion overhead** -- deep trees mean deep call stacks. Fine for most rule engines, but worth watching with very large trees or very high evaluation frequency.
- **Debugging indirection** -- stepping through recursive evaluation in a debugger means bouncing across many `Interpret()` calls. Adding a `String()` method to each expression for logging helps, but it's inherently harder to trace than a flat function.

## Best Practices

- **Keep the context passive.** It holds values, nothing more. Evaluation logic inside the context splits interpretation across two places and makes both harder to reason about.
- **Make expressions immutable.** Once constructed, an expression shouldn't change. This makes them safe to share across goroutines and reuse across evaluations without defensive copying.
- **Handle missing keys gracefully.** A rule might reference a metric the context doesn't have -- maybe the host hasn't reported it yet. Returning `false` is usually safer than panicking, especially when rules come from external config.
- **Add `String()` for debuggability.** Expressions that can describe themselves (e.g., `"cpu > 90.00"`, `"(cpu > 90.00 AND memory > 80.00)"`) make logging and debugging dramatically easier when tracing why a particular evaluation returned what it did.
- **Validate the tree at construction time, not evaluation time.** If `BuildRule` receives an `"and"` with zero children, catch it there. Don't let a malformed tree surface as a nil pointer during `Interpret()`.

## Common Mistakes

**Confusing interpretation with parsing.** The Interpreter pattern evaluates a tree. It doesn't parse `"cpu > 90 AND memory > 80"` from a string. Parsing is a separate, often harder, problem. Mixing both into a single layer produces code that's neither a good parser nor a clean interpreter. Build your parser (or config loader, or JSON deserializer) separately, and feed its output into the expression tree.

**Building a central evaluator with type switches.** This defeats the pattern entirely:

```go
func eval(expr Expression, ctx Context) bool {
    switch e := expr.(type) {
    case *And:
        return eval(e.left, ctx) && eval(e.right, ctx)
    case *GreaterThan:
        return ctx[e.metric] > e.threshold
    }
    return false
}
```

Every new expression type means modifying this function. The whole point is that each expression evaluates itself -- distributing the logic so the system is extensible without central modification.

**Letting the grammar outgrow the pattern.** Interpreter works beautifully for 5-10 expression types. Somewhere around 20-30, the struct count gets unwieldy, tree construction becomes its own problem, and you start wanting operator precedence, parenthesis grouping, and error recovery. That's a parser's job. Knowing when to graduate from Interpreter to a proper DSL framework is part of using the pattern well.

**Coupling non-terminals to concrete types.** Writing `left *GreaterThan` instead of `left Expression` in an `And` struct. This is the most common structural mistake, and it destroys composability. Non-terminals should only know about the `Expression` interface.

## Final Thoughts

The Interpreter pattern is a specialist tool. Most codebases won't need it, and reaching for it when a simple `if` statement would do is worse than not knowing it exists. But for the problem it's designed for -- evaluating dynamic, composable rules against varying contexts -- it's remarkably clean.

The rules become data. The grammar becomes a set of small structs. Evaluation becomes recursive traversal. And extending the system means adding a struct and a `case` in the builder, not rewriting a switch statement that was already too long.

Go is a particularly good fit here. Interfaces are implicit, so expressions don't need to declare anything beyond their `Interpret()` method. Composition is the default, so recursive trees feel natural. And the pattern shows up in Go's own toolchain -- `go/ast` is exactly this structure, applied to the grammar of the Go language itself.

The constraint worth remembering is scope. Small grammar, stable rules, dynamic evaluation -- that's the sweet spot. The moment the grammar starts looking like a language of its own, the pattern has done its job and it's time to move on.

> **Model rules as a language when they behave like one -- but know when the language needs a real parser.**
