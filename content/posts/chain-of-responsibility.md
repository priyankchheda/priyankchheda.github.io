+++
date = '2026-01-28'
title = 'Chain of Responsibility Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'behavioral']
+++

Think about an HTTP request hitting your API server. Before the actual business logic runs, several things need to happen: authentication verifies the token, authorization checks permissions, rate limiting ensures the client hasn't exceeded their quota, request validation confirms the payload is well-formed, and logging records the request for observability. Each check can either pass the request along or reject it outright.

The naive approach is a nested `if/else` tree inside the handler. Authentication passes? Check authorization. Authorization passes? Check rate limit. Rate limit passes? Validate the payload. Each new concern adds another nesting level, and the handler becomes a 200-line function that does six different things.

The **Chain of Responsibility** pattern replaces that nesting with a pipeline. Each concern becomes its own handler -- a small, focused unit that decides to process the request, pass it along, or stop the chain. Handlers link together, and a request flows through them in sequence. Adding a new check means adding a handler to the chain. Removing one means unlinking it. The business logic at the end doesn't know or care what happened upstream.

If this sounds exactly like HTTP middleware, that's because it is. Go's `net/http` middleware stacks are the Chain of Responsibility pattern in its most idiomatic form. In this article, we'll build both a struct-based chain and a functional middleware chain, show how Go's standard library uses this pattern, and explore the trade-offs between the two approaches.

## What Is the Chain of Responsibility Pattern?

> "Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it."
>-- Gang of Four

The core idea: **a request passes through a sequence of handlers, each of which can process it, modify it, or stop it.** The sender doesn't know which handler will ultimately deal with the request, or how many handlers exist in the chain. It just hands the request to the first handler and trusts the chain to do its job.

Without this pattern, request processing looks like this:

```go
func handleRequest(req Request) Response {
    if !authenticate(req) {
        return unauthorizedResponse()
    }
    if !authorize(req) {
        return forbiddenResponse()
    }
    if !rateLimit(req) {
        return tooManyRequestsResponse()
    }
    if !validate(req) {
        return badRequestResponse()
    }
    return processBusinessLogic(req)
}
```

Every concern lives in one function. Adding CORS handling means modifying this function. Adding request tracing means modifying this function. Adding a circuit breaker means modifying this function. The function grows, responsibilities tangle, and testing any single concern means running through the entire chain.

With the pattern, each concern is its own handler:

```txt
Request --> Auth --> RateLimit --> Validate --> BusinessLogic --> Response
```

Each handler has one job. The chain is configured externally. Adding or removing handlers doesn't touch any existing handler's code.

## Core Components

Two concepts: the handler interface and the chain linking.

**Handler** -- defines the contract for processing a request. Each handler can either handle the request itself, pass it to the next handler, or do both (process something and continue).

```go
type Handler interface {
    Handle(req *Request) error
    SetNext(h Handler)
}
```

**ConcreteHandler** -- implements the Handler interface. Contains one focused piece of logic and a reference to the next handler in the chain.

The flow:

```txt
Client --> Handler A --> Handler B --> Handler C --> (end)
                |              |              |
            handle or      handle or      handle or
            pass along     pass along     pass along
```

Each handler independently decides whether to continue the chain or stop it. The client only interacts with the first handler.

## Code Walkthrough: An HTTP Request Pipeline

We're building a request processing pipeline with authentication, rate limiting, and validation -- the kind of chain you'd put in front of an API handler.

### The Request Type

```go
type Request struct {
    Token   string
    ClientIP string
    Body    string
    Headers map[string]string
}
```

### Approach 1: Struct-Based Chain (Classic GoF)

Define the handler interface:

```go
type Handler interface {
    Handle(req *Request) error
    SetNext(h Handler)
}
```

A base handler reduces boilerplate -- it holds the `next` reference and provides a helper to forward the request:

```go
type BaseHandler struct {
    next Handler
}

func (b *BaseHandler) SetNext(h Handler) {
    b.next = h
}

func (b *BaseHandler) Forward(req *Request) error {
    if b.next != nil {
        return b.next.Handle(req)
    }
    return nil
}
```

Now the concrete handlers. Each embeds `BaseHandler` and implements `Handle`:

**AuthHandler** -- verifies the request has a valid token:

```go
type AuthHandler struct {
    BaseHandler
}

func (h *AuthHandler) Handle(req *Request) error {
    if req.Token == "" {
        return fmt.Errorf("auth: missing token")
    }
    if req.Token != "valid-token" {
        return fmt.Errorf("auth: invalid token")
    }
    fmt.Println("  [auth] token verified")
    return h.Forward(req)
}
```

**RateLimitHandler** -- checks if the client has exceeded their quota:

```go
type RateLimitHandler struct {
    BaseHandler
    blocked map[string]bool
}

func NewRateLimitHandler(blocked []string) *RateLimitHandler {
    m := make(map[string]bool)
    for _, ip := range blocked {
        m[ip] = true
    }
    return &RateLimitHandler{blocked: m}
}

func (h *RateLimitHandler) Handle(req *Request) error {
    if h.blocked[req.ClientIP] {
        return fmt.Errorf("rate limit: client %s is blocked", req.ClientIP)
    }
    fmt.Println("  [rate limit] client allowed")
    return h.Forward(req)
}
```

**ValidationHandler** -- ensures the request body isn't empty:

```go
type ValidationHandler struct {
    BaseHandler
}

func (h *ValidationHandler) Handle(req *Request) error {
    if req.Body == "" {
        return fmt.Errorf("validation: empty request body")
    }
    fmt.Println("  [validation] body is valid")
    return h.Forward(req)
}
```

### Building and Using the Chain

```go
func main() {
    auth := &AuthHandler{}
    rateLimit := NewRateLimitHandler([]string{"10.0.0.99"})
    validation := &ValidationHandler{}

    auth.SetNext(rateLimit)
    rateLimit.SetNext(validation)

    req := &Request{
        Token:    "valid-token",
        ClientIP: "10.0.0.1",
        Body:     `{"action": "create"}`,
    }

    fmt.Println("--- Valid request ---")
    if err := auth.Handle(req); err != nil {
        fmt.Println("REJECTED:", err)
    } else {
        fmt.Println("ACCEPTED")
    }

    fmt.Println("\n--- Bad token ---")
    req.Token = "wrong"
    if err := auth.Handle(req); err != nil {
        fmt.Println("REJECTED:", err)
    }

    fmt.Println("\n--- Blocked IP ---")
    req.Token = "valid-token"
    req.ClientIP = "10.0.0.99"
    if err := auth.Handle(req); err != nil {
        fmt.Println("REJECTED:", err)
    }
}
```

```txt
--- Valid request ---
  [auth] token verified
  [rate limit] client allowed
  [validation] body is valid
ACCEPTED

--- Bad token ---
REJECTED: auth: invalid token

--- Blocked IP ---
  [auth] token verified
REJECTED: rate limit: client 10.0.0.99 is blocked
```

Each handler stops the chain by returning an error. The caller gets a clear reason for rejection. No handler downstream of the failure point executes.

### Approach 2: Functional Middleware Chain (Idiomatic Go)

The struct-based approach works, but Go developers will recognize a more idiomatic pattern -- function-based middleware. This is exactly how `net/http` middleware works.

```go
type HandlerFunc func(req *Request) error
type Middleware func(next HandlerFunc) HandlerFunc
```

A `Middleware` takes a handler and returns a new handler that wraps it. Each middleware does its work, then optionally calls `next`.

**Logging middleware:**

```go
func LoggingMiddleware(next HandlerFunc) HandlerFunc {
    return func(req *Request) error {
        fmt.Printf("  [log] %s %s\n", req.ClientIP, req.Body)
        return next(req)
    }
}
```

**Auth middleware:**

```go
func AuthMiddleware(next HandlerFunc) HandlerFunc {
    return func(req *Request) error {
        if req.Token != "valid-token" {
            return fmt.Errorf("auth: unauthorized")
        }
        fmt.Println("  [auth] verified")
        return next(req)
    }
}
```

**Rate limit middleware:**

```go
func RateLimitMiddleware(blocked map[string]bool) Middleware {
    return func(next HandlerFunc) HandlerFunc {
        return func(req *Request) error {
            if blocked[req.ClientIP] {
                return fmt.Errorf("rate limit: blocked")
            }
            return next(req)
        }
    }
}
```

**Composing the chain:**

```go
func Chain(handler HandlerFunc, middlewares ...Middleware) HandlerFunc {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}
```

The middlewares wrap in reverse order so the first middleware in the list runs first.

**Usage:**

```go
func main() {
    businessLogic := func(req *Request) error {
        fmt.Println("  [handler] processing:", req.Body)
        return nil
    }

    blocked := map[string]bool{"10.0.0.99": true}

    pipeline := Chain(
        businessLogic,
        LoggingMiddleware,
        AuthMiddleware,
        RateLimitMiddleware(blocked),
    )

    req := &Request{
        Token:    "valid-token",
        ClientIP: "10.0.0.1",
        Body:     `{"action":"create"}`,
    }

    if err := pipeline(req); err != nil {
        fmt.Println("ERROR:", err)
    }
}
```

```txt
  [log] 10.0.0.1 {"action":"create"}
  [auth] verified
  [handler] processing: {"action":"create"}
```

This is the same pattern, different shape. The functional version has less boilerplate, no interfaces, no `SetNext` wiring -- just function composition. It's the approach you'll find in nearly every Go HTTP framework.

## The Pattern in Go's Standard Library

Go's `net/http` package uses this pattern directly. The `http.Handler` interface is the handler, and middleware wraps handlers:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
```

`http.Handler` is the `Handler` interface. `http.HandlerFunc` is the function adapter. Middleware functions take a handler and return a handler. The chain is built by nesting: `logging(auth(rateLimit(businessLogic)))`. This is the Chain of Responsibility pattern, and it handles millions of requests in production Go services every day.

## When to Use It

The pattern fits when **a request needs to pass through multiple processing steps, and the set of steps should be configurable**:

- **HTTP/RPC middleware** -- authentication, authorization, rate limiting, logging, tracing, CORS, compression. The classic use case.
- **Validation pipelines** where each validator checks one aspect of the input and either passes or rejects, and new validators are added as requirements grow.
- **Event processing** where events flow through filters, transformers, and enrichers before reaching their final handler.
- **Approval workflows** where a request escalates through levels (support tiers, management approvals) until someone handles it.

Skip it when:

- **There's only one handler.** A direct function call is simpler. The pattern adds value when there are multiple handlers that can be composed differently.
- **Execution order doesn't need to change.** If the steps are always the same, always in the same order, and will never grow, a simple sequential function is clearer.
- **You need guaranteed handling.** If the request *must* be processed and silent pass-through is a bug, the pattern's "handlers can choose to pass" semantics work against you. Use a pipeline with mandatory stages instead.

## Pros and Cons

**Pros:**

- **Single Responsibility** -- each handler does one thing. Auth checks auth. Rate limiting checks rates. Validation validates. Testing each in isolation is straightforward.
- **Open/Closed compliance** -- adding a new handler (circuit breaker, request ID injection, CORS) doesn't modify any existing handler or the business logic.
- **Flexible ordering and composition** -- the same handlers can be arranged differently for different routes. An admin API might skip rate limiting. A public API might add extra validation.
- **Matches Go idioms** -- the functional middleware pattern (`func(next Handler) Handler`) is so natural in Go that most developers use it without knowing it has a GoF name.

**Cons:**

- **Debugging indirection** -- when a request is rejected, tracing *which* handler rejected it requires either good error messages or logging in each handler. The flow is spread across multiple files.
- **Silent pass-through risk** -- if no handler processes the request and there's no final fallback, the request silently succeeds (or silently fails) with no processing. Design requires a terminal handler.
- **Order sensitivity** -- auth before rate limiting behaves differently than rate limiting before auth. The chain's correctness depends on handler ordering, which is configured externally and can be misconfigured.

## Best Practices

- **Return errors, don't just print.** Handlers that `fmt.Println("failed")` and return silently swallow errors. Return an `error` so the caller knows the chain was interrupted and why.
- **Wrap in reverse order for functional chains.** `Chain(handler, mw1, mw2, mw3)` should iterate in reverse so `mw1` wraps outermost and runs first. Getting this wrong means middleware runs in unexpected order.
- **Always have a terminal handler.** The last handler in the chain should be the actual business logic, not another check. If the chain ends without a terminal handler, requests pass through the entire chain and nothing happens.
- **Make handlers independent.** A handler should not hold a reference to another handler (other than `next`). Inter-handler communication should happen through the request object or context, not through direct coupling.
- **Use `context.Context` for cross-cutting data in Go.** Instead of stuffing user IDs or trace IDs into the request struct, use Go's context. Middleware can enrich the context, and downstream handlers read from it. This follows Go convention and works with the standard library.

## Common Mistakes

**Forgetting to call `next` (or `Forward`).** A handler that does its work and returns without calling the next handler silently breaks the chain. This is the most common bug -- especially when the handler has multiple return paths and one of them skips the `next` call. Every path through the handler should either return an error (intentionally stopping the chain) or call next (continuing it).

**Putting business logic in a middleware handler.** Middleware should handle cross-cutting concerns: auth, logging, rate limiting, tracing. The moment a middleware is computing order totals or querying a database for business data, it's doing the wrong job. Keep middleware thin and focused on request lifecycle, not domain logic.

**Not handling the "no handler processed the request" case.** If every handler in the chain passes and there's no terminal handler, the request completes with an empty response or no action. In the struct-based approach, `BaseHandler.Forward` returns `nil` when there's no next handler -- which the caller interprets as success. Always terminate the chain with a handler that does real work or explicitly returns an error.

**Depending on handler execution order for correctness.** If handler B only works correctly because handler A ran first (e.g., B reads a value that A sets on the request), that's an implicit dependency between handlers. Document it clearly, or better yet, make each handler defensive about what it expects on the request. A handler that panics because a previous handler didn't set a field is a chain that breaks when reordered.

## Final Thoughts

The Chain of Responsibility pattern takes a monolithic request processor and splits it into a pipeline of focused, composable handlers. Each handler has one concern, the chain is configured externally, and new handlers slot in without modifying existing code.

Go embraces this pattern more visibly than most languages. The functional middleware style -- `func(next Handler) Handler` -- is the standard approach for HTTP servers, gRPC interceptors, and event pipelines. It doesn't feel like a "design pattern" in Go; it feels like how you write software. Which is probably the highest compliment a pattern can receive.

The discipline is minimal: return errors instead of swallowing them, always call next unless you're intentionally stopping the chain, and keep handlers independent. Get those right, and the pattern scales from three middleware to thirty without losing clarity.

> **Give each concern its own handler, link them into a chain, and let the request flow through until someone says stop.**
