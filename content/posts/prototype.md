+++
date = '2025-09-11'
title = 'Prototype Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a load testing tool. It sends thousands of HTTP requests per second, each slightly different -- different query parameters, different headers, different payload fields. But the base request is always the same: same host, same method, same authentication headers, same timeout configuration. Constructing each request from scratch means repeating all that setup logic thousands of times. The setup isn't just verbose -- it might involve reading config files, computing auth signatures, or resolving DNS. Doing it once and cloning with modifications is faster, simpler, and less error-prone.

The **Prototype pattern** formalizes this idea. Keep a fully-initialized object (the prototype), and when you need a new one, clone it instead of building from scratch. The clone is independent -- modify it freely without affecting the original. The client doesn't need to know the concrete type or its construction details. It just calls `Clone()`.

In Go, this maps naturally onto value semantics and explicit copy methods. A struct dereference (`copy := *original`) gives you a shallow copy for free. For structs with slices, maps, or pointers, you write a `Clone()` method that handles the deep copy. No reflection frameworks, no serialization magic -- just explicit, readable code.

In this article, we'll build a request template system for an HTTP load tester, explore shallow vs deep cloning, add a prototype registry for managing templates, and cover the performance trade-offs that make cloning worthwhile.

## What Is the Prototype Pattern?

> "Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype."
> -- Gang of Four

The intent: **create new objects by copying an existing one, rather than constructing from scratch.** The prototype is a fully-initialized template. When you need a new instance, you clone it and customize the clone. The original stays untouched, ready for the next clone.

This matters when construction is expensive (parsing config, loading data, computing state), when you need many similar objects with minor variations, or when the concrete type isn't known at compile time and you need to duplicate whatever you're given.

Without the pattern, creating request variants looks like this:

```go
func makeRequest(path string, token string) *http.Request {
    req, _ := http.NewRequest("POST", "https://api.prod.internal"+path, nil)
    req.Header.Set("Authorization", "Bearer "+token)
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Request-ID", uuid.New().String())
    req.Header.Set("X-Trace-ID", generateTraceID())
    // ... 10 more headers
    return req
}
```

Every call repeats the full setup. If the base configuration changes (new required header, different content type), every call site changes. And if setup involves computation (generating trace IDs, signing requests), that work is repeated unnecessarily.

With a prototype:

```go
baseRequest := buildBaseRequest()

clone := baseRequest.Clone()
clone.URL.Path = "/users"
clone.Header.Set("X-Request-ID", uuid.New().String())
```

Build the expensive base once. Clone it cheaply. Customize the clone. The base stays stable.

## Core Components

Three participants.

**Prototype** -- the interface declaring the clone capability.

```go
type Prototype interface {
    Clone() Prototype
}
```

**ConcretePrototype** -- the actual object with state that can be cloned. Implements `Clone()` with proper deep-copy semantics for its fields.

**Client** -- uses prototypes without knowing their concrete type. Asks for a clone, customizes it, uses it.

Optionally, a **Prototype Registry** stores named prototypes so the client can request clones by key without holding direct references to prototypes.

The flow:

```txt
Client --> Prototype.Clone() --> Independent copy --> Client customizes and uses
```

The client never constructs the object from scratch. It always starts from a clone.

## Code Walkthrough: An HTTP Request Template System

We're building a request template system for a load tester. Base requests are configured once (host, auth, common headers), then cloned and customized per test scenario.

### The Prototype Interface

```go
type RequestTemplate interface {
    Clone() RequestTemplate
    SetPath(path string)
    SetBody(body []byte)
    SetHeader(key, value string)
    String() string
}
```

### The Concrete Prototype

A request template with headers (a map) and a body (a slice) -- both reference types that need deep copying:

```go
type HTTPRequestTemplate struct {
    Method  string
    Host    string
    Path    string
    Headers map[string]string
    Body    []byte
}

func NewHTTPRequestTemplate(method, host string) *HTTPRequestTemplate {
    return &HTTPRequestTemplate{
        Method:  method,
        Host:    host,
        Headers: make(map[string]string),
    }
}

func (r *HTTPRequestTemplate) SetPath(path string) {
    r.Path = path
}

func (r *HTTPRequestTemplate) SetBody(body []byte) {
    r.Body = body
}

func (r *HTTPRequestTemplate) SetHeader(key, value string) {
    r.Headers[key] = value
}

func (r *HTTPRequestTemplate) String() string {
    return fmt.Sprintf("%s %s%s headers=%d body=%d bytes",
        r.Method, r.Host, r.Path, len(r.Headers), len(r.Body))
}
```

### The Clone Method

This is where the pattern lives. Headers is a map, Body is a slice -- both must be deep-copied:

```go
func (r *HTTPRequestTemplate) Clone() RequestTemplate {
    headers := make(map[string]string, len(r.Headers))
    for k, v := range r.Headers {
        headers[k] = v
    }

    var body []byte
    if r.Body != nil {
        body = make([]byte, len(r.Body))
        copy(body, r.Body)
    }

    return &HTTPRequestTemplate{
        Method:  r.Method,
        Host:    r.Host,
        Path:    r.Path,
        Headers: headers,
        Body:    body,
    }
}
```

Value-type fields (`Method`, `Host`, `Path`) are copied automatically by assignment. Reference-type fields (`Headers`, `Body`) are explicitly deep-copied. The clone is completely independent -- modifying its headers or body doesn't affect the original.

### Using It

```go
func main() {
    base := NewHTTPRequestTemplate("POST", "https://api.prod.internal")
    base.SetHeader("Authorization", "Bearer secret-token")
    base.SetHeader("Content-Type", "application/json")
    base.SetHeader("X-Client", "load-tester-v2")

    fmt.Println("Base:", base)

    userReq := base.Clone()
    userReq.SetPath("/users")
    userReq.SetBody([]byte(`{"name":"alice"}`))
    userReq.SetHeader("X-Request-ID", "req-001")
    fmt.Println("User:", userReq)

    orderReq := base.Clone()
    orderReq.SetPath("/orders")
    orderReq.SetBody([]byte(`{"item":"widget","qty":5}`))
    orderReq.SetHeader("X-Request-ID", "req-002")
    fmt.Println("Order:", orderReq)

    fmt.Println("Base unchanged:", base)
}
```

```txt
Base: POST https://api.prod.internal headers=3 body=0 bytes
User: POST https://api.prod.internal/users headers=4 body=16 bytes
Order: POST https://api.prod.internal/orders headers=4 body=25 bytes
Base unchanged: POST https://api.prod.internal headers=3 body=0 bytes
```

The base template was configured once with three headers. Each clone got its own path, body, and request ID header. The base stayed at three headers and zero body bytes -- completely unaffected by the clones. That's the deep copy doing its job.

## Prototype Registry: Managing Templates by Name

In a load testing tool, you'd have multiple base templates: one for authenticated endpoints, one for public endpoints, one for admin routes. A registry stores these by name and clones on demand:

```go
type TemplateRegistry struct {
    templates map[string]RequestTemplate
}

func NewTemplateRegistry() *TemplateRegistry {
    return &TemplateRegistry{templates: make(map[string]RequestTemplate)}
}

func (r *TemplateRegistry) Register(name string, tmpl RequestTemplate) {
    r.templates[name] = tmpl
}

func (r *TemplateRegistry) Get(name string) (RequestTemplate, error) {
    tmpl, ok := r.templates[name]
    if !ok {
        return nil, fmt.Errorf("template not found: %s", name)
    }
    return tmpl.Clone(), nil
}
```

`Get` always returns a clone, never the original. The registry's prototypes stay pristine.

```go
func main() {
    registry := NewTemplateRegistry()

    authTemplate := NewHTTPRequestTemplate("POST", "https://api.prod.internal")
    authTemplate.SetHeader("Authorization", "Bearer secret")
    authTemplate.SetHeader("Content-Type", "application/json")
    registry.Register("authenticated", authTemplate)

    publicTemplate := NewHTTPRequestTemplate("GET", "https://api.prod.internal")
    publicTemplate.SetHeader("Content-Type", "application/json")
    registry.Register("public", publicTemplate)

    req, _ := registry.Get("authenticated")
    req.SetPath("/orders")
    fmt.Println(req)

    req2, _ := registry.Get("public")
    req2.SetPath("/health")
    fmt.Println(req2)
}

// OUTPUT:
// POST https://api.prod.internal/orders headers=2 body=0 bytes
// GET https://api.prod.internal/health headers=1 body=0 bytes
```

Adding a new template category means registering it -- no switch statements, no factory functions to modify.

## Shallow vs Deep Cloning

The critical implementation decision in the Prototype pattern is what gets copied and what gets shared.

**Shallow clone** copies the struct's fields by value. For primitive types (`int`, `string`, `bool`), this creates independent copies. For reference types (`map`, `slice`, `*T`), it copies the *pointer* -- both original and clone point to the same underlying data. Modifying the clone's map modifies the original's map.

**Deep clone** creates new instances of all reference-type fields. The clone is fully independent. No shared state.

In Go, a struct assignment (`copy := *original`) is always a shallow copy. That's fine for structs with only value-type fields. The moment you have a map, slice, or pointer field, you need explicit deep-copy logic in your `Clone()` method.

The walkthrough's `Clone()` demonstrates this: `Method`, `Host`, and `Path` (all strings -- value types in Go) are copied by assignment. `Headers` (map) and `Body` (slice) are explicitly deep-copied with make + loop/copy. Skip the deep copy, and your clones share data with the original -- a bug that's subtle, intermittent, and painful to debug.

## When to Use It

The Prototype pattern fits when **you need many similar objects and construction is either expensive or repetitive**:

- **Expensive initialization** where the base object requires file parsing, network calls, computation, or multi-step setup. Clone the result instead of repeating the work.
- **Template-based creation** where you maintain preconfigured objects (request templates, document layouts, test fixtures) and spawn variants by cloning and customizing.
- **Runtime type flexibility** where you don't know the concrete type at compile time but need to duplicate whatever you have. The `Clone()` interface lets you copy any prototype polymorphically.
- **Registry-based systems** where a set of prototypes is stored by key and cloned on demand -- avoiding scattered factory logic.

Skip it when:

- **Construction is trivial.** If creating the object is just `&MyStruct{field: value}` with no expensive setup, cloning adds complexity for zero speed benefit.
- **Objects have no shared base state.** If every instance is completely different with no common template, there's nothing to clone from.
- **You need immutable objects.** If the "prototype" should never be modified and clones aren't customized, you might just share the original (Flyweight pattern) rather than cloning it.

## Pros and Cons

**Pros:**

- **Avoids expensive re-initialization** -- construct the complex object once, clone it cheaply afterward. For objects requiring file I/O, network calls, or heavy computation during setup, cloning can be orders of magnitude faster.
- **Decouples client from concrete types** -- the client works with the `Prototype` interface. It can clone any object without knowing its specific type or construction details.
- **Simplifies creation of variants** -- start from a known-good base and modify only what differs. Less repetitive code, fewer opportunities for misconfiguration.
- **Supports dynamic type sets** -- new prototype types can be registered at runtime without modifying existing code. Good for plugin systems and extensible architectures.

**Cons:**

- **Deep cloning is error-prone** -- forgetting to deep-copy a map, slice, or pointer field creates shared state between original and clone. The bug is silent until concurrent modification or unexpected mutation reveals it.
- **Clone maintenance burden** -- every time you add a field to the struct, you must update `Clone()`. Forget, and the new field either gets zero-valued in clones (shallow miss) or shared (deep miss).
- **Not always faster than construction** -- for simple structs, the overhead of deep-copying slices and maps can exceed the cost of just creating a new instance. Measure before assuming cloning wins.

## Best Practices

- **Document whether Clone() is shallow or deep.** A reader of your code should know immediately whether clones share state with the original. If the answer varies by field (some shared, some independent), document that per-field.
- **Never return the original from a registry.** `registry.Get()` should always return `Clone()`, never the prototype itself. Returning the original lets callers mutate the registry's prototype, corrupting future clones.
- **Test clone independence explicitly.** Write tests that clone an object, modify the clone's reference-type fields, and assert the original is unaffected. This catches shallow-copy bugs that won't surface until production.
- **Use value-type fields where possible to reduce deep-copy complexity.** If a field can be a `string` instead of `*string`, or a fixed-size array instead of a slice, the shallow copy is automatically correct and you avoid one more field in your `Clone()` method.

## Common Mistakes

**Forgetting to update Clone() when fields are added.** A struct gains a new `Tags []string` field. The `Clone()` method, written months ago, doesn't copy it. Clones get a nil `Tags` (if the field is a slice) or share the original's tags (if the prototype has them). The fix: treat `Clone()` as a contract that must be reviewed on every struct change. Some teams add a test that compares field counts between the struct and the clone method to catch drift.

**Using serialization for deep copy in hot paths.** `json.Marshal` / `json.Unmarshal` or `gob.Encode` / `gob.Decode` produce correct deep copies but are dramatically slower than explicit field-by-field copying. They're fine for setup-time or infrequent operations. In a loop that runs thousands of times per second, write the explicit `Clone()`.

**Cloning when construction is cheap.** If your struct has three `string` fields and no reference types, `NewThing(a, b, c)` is simpler, clearer, and no slower than maintaining a prototype and calling `Clone()`. The Prototype pattern earns its keep when construction involves work beyond just setting fields -- file I/O, computation, multi-step initialization. Without that, it's overhead.

**Sharing mutable prototypes across goroutines without synchronization.** If the prototype itself is mutable (e.g., the registry modifies it), concurrent cloning can race. Either make prototypes immutable after registration, or protect the clone operation with a mutex. Immutable prototypes are the cleaner solution -- once registered, never modified.

## Final Thoughts

The Prototype pattern gives you a way to create objects by copying rather than constructing. Clone an expensive-to-build template, customize the copy, and avoid repeating the setup work. The original stays clean, ready for the next clone.

Go makes the shallow case trivial (`copy := *original`) and the deep case explicit (loop over maps, copy slices, clone nested structs). There's no magic and no hidden behavior -- which aligns perfectly with Go's philosophy of explicitness and clarity. The `Clone()` method is the contract: it tells you exactly what independence guarantee you get.

The pattern shines when objects are expensive to build, when many variants share a common base, or when you need to duplicate objects polymorphically without knowing their concrete type. For simple structs with cheap construction, skip it -- a plain constructor is clearer. But when the template-and-clone workflow fits, it eliminates repetition and centralizes the "known-good base" in one place.

> **Build the expensive thing once, clone it cheaply, and never hand out the original.**
