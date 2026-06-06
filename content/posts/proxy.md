+++
date = '2026-04-25'
title = 'Proxy Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about a third-party API client. It makes HTTP requests to an external service -- a geocoding API, a payment gateway, a machine learning inference endpoint. The client works fine, but production reveals problems: the API has rate limits (429 errors if you exceed them), responses for the same input are identical (wasting quota on redundant calls), and certain callers shouldn't have access to certain endpoints (security boundaries).

You could modify the API client directly -- add rate limiting, caching, and access control inside it. But the client isn't yours to modify (it's a third-party SDK), or it does one thing well and shouldn't grow into a god object, or you need different combinations of these controls in different contexts.

The **Proxy pattern** solves this by placing a stand-in object between the client code and the real service. The proxy implements the same interface as the real service, so callers can't tell the difference. Internally, the proxy intercepts each call and adds its own logic: check the cache before forwarding, enforce rate limits before calling the real API, verify permissions before allowing access. The real service stays unchanged.

In this article, we'll build an API client proxy that adds caching, rate limiting, and access control around a geocoding service. Go's interfaces make the proxy transparent -- the caller works with the same contract regardless of whether it's talking to the real service or a proxy stack.

## What Is the Proxy Pattern?

> "Provide a surrogate or placeholder for another object to control access to it."
> -- Gang of Four

The intent: **interpose an object between the client and a real service to control how and when the real service is accessed.** The proxy implements the same interface as the real service, making it a drop-in replacement. The client doesn't know (or care) that it's talking to a proxy.

Unlike Decorator (which adds behavior), the Proxy's focus is *control*: access control, caching to avoid unnecessary work, lazy initialization to defer expensive creation, rate limiting to prevent overuse. The proxy decides whether to forward the request, when to forward it, and what to do with the result.

Without a proxy, protection logic scatters into every call site:

```go
func getCoordinates(city string) (*Location, error) {
    if !hasPermission(currentUser, "geocode") {
        return nil, ErrForbidden
    }
    if cached := cache.Get(city); cached != nil {
        return cached, nil
    }
    if rateLimiter.Exceeded() {
        return nil, ErrRateLimited
    }
    result, err := geocodingAPI.Lookup(city)
    if err == nil {
        cache.Set(city, result)
    }
    return result, err
}
```

Access control, caching, and rate limiting are mixed with the business logic of getting coordinates. Every function that calls the geocoding API repeats some variation of this. The proxy centralizes it: callers just call `service.Lookup(city)` and the proxy handles the rest transparently.

## Core Components

Three participants.

**Subject** -- the interface that both the real service and the proxy implement.

```go
type GeocodingService interface {
    Lookup(city string) (*Location, error)
}
```

**RealSubject** -- the actual service that does the work (makes HTTP calls, queries databases, etc.).

**Proxy** -- implements the same interface, holds a reference to the real subject, and adds control logic around each call.

The flow:

```txt
Client --> Proxy.Lookup() --> [caching? rate limit? access check?] --> RealService.Lookup()
```

The client calls the proxy. The proxy decides whether and how to forward to the real service. The real service does the actual work. All three share the same interface.

## Code Walkthrough: A Geocoding API with Proxy Controls

We're building a geocoding service with three layers of proxy control: caching (avoid redundant API calls), rate limiting (stay within quota), and access control (restrict by caller role).

### The Interface and Real Service

```go
type Location struct {
    Lat  float64
    Lon  float64
    City string
}

type GeocodingService interface {
    Lookup(city string) (*Location, error)
}
```

The real service makes the actual API call:

```go
type RealGeocodingService struct{}

func (s *RealGeocodingService) Lookup(city string) (*Location, error) {
    fmt.Printf("  [api] calling external geocoding API for %q\n", city)
    // Simulate API call
    locations := map[string]*Location{
        "london":    {Lat: 51.5074, Lon: -0.1278, City: "London"},
        "tokyo":     {Lat: 35.6762, Lon: 139.6503, City: "Tokyo"},
        "new york":  {Lat: 40.7128, Lon: -74.0060, City: "New York"},
    }
    loc, ok := locations[city]
    if !ok {
        return nil, fmt.Errorf("city not found: %s", city)
    }
    return loc, nil
}
```

### Proxy 1: Caching

Stores results and returns cached values for repeated lookups:

```go
type CachingProxy struct {
    service GeocodingService
    cache   map[string]*Location
    mu      sync.RWMutex
}

func NewCachingProxy(service GeocodingService) *CachingProxy {
    return &CachingProxy{
        service: service,
        cache:   make(map[string]*Location),
    }
}

func (p *CachingProxy) Lookup(city string) (*Location, error) {
    p.mu.RLock()
    if loc, ok := p.cache[city]; ok {
        p.mu.RUnlock()
        fmt.Printf("  [cache] hit for %q\n", city)
        return loc, nil
    }
    p.mu.RUnlock()

    loc, err := p.service.Lookup(city)
    if err != nil {
        return nil, err
    }

    p.mu.Lock()
    p.cache[city] = loc
    p.mu.Unlock()
    fmt.Printf("  [cache] stored %q\n", city)
    return loc, nil
}
```

The caching proxy implements `GeocodingService`. It checks the cache first, forwards to the wrapped service on miss, and stores the result. Thread-safe with `sync.RWMutex`.

### Proxy 2: Rate Limiting

Rejects requests when the quota is exceeded:

```go
type RateLimitProxy struct {
    service   GeocodingService
    calls     int
    maxCalls  int
    mu        sync.Mutex
}

func NewRateLimitProxy(service GeocodingService, maxCalls int) *RateLimitProxy {
    return &RateLimitProxy{
        service:  service,
        maxCalls: maxCalls,
    }
}

func (p *RateLimitProxy) Lookup(city string) (*Location, error) {
    p.mu.Lock()
    if p.calls >= p.maxCalls {
        p.mu.Unlock()
        return nil, fmt.Errorf("rate limit exceeded (%d/%d calls used)", p.calls, p.maxCalls)
    }
    p.calls++
    p.mu.Unlock()

    return p.service.Lookup(city)
}
```

Counts calls and rejects when the limit is hit. In production this would use a sliding window or token bucket, but the proxy mechanics are the same.

### Proxy 3: Access Control

Restricts access based on caller role:

```go
type AccessControlProxy struct {
    service      GeocodingService
    allowedRoles map[string]bool
}

func NewAccessControlProxy(service GeocodingService, roles []string) *AccessControlProxy {
    allowed := make(map[string]bool)
    for _, r := range roles {
        allowed[r] = true
    }
    return &AccessControlProxy{
        service:      service,
        allowedRoles: allowed,
    }
}

func (p *AccessControlProxy) Lookup(city string) (*Location, error) {
    role := getCurrentUserRole()
    if !p.allowedRoles[role] {
        return nil, fmt.Errorf("access denied: role %q cannot use geocoding", role)
    }
    return p.service.Lookup(city)
}

func getCurrentUserRole() string {
    return "engineer"
}
```

Checks the caller's role before forwarding. Unauthorized roles get a clear error without the real service ever being called.

### Stacking Proxies

Because all proxies implement `GeocodingService`, they compose naturally:

```go
func main() {
    real := &RealGeocodingService{}
    cached := NewCachingProxy(real)
    limited := NewRateLimitProxy(cached, 5)
    secured := NewAccessControlProxy(limited, []string{"engineer", "admin"})

    fmt.Println("--- First lookup ---")
    loc, err := secured.Lookup("london")
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Result: %s (%.4f, %.4f)\n", loc.City, loc.Lat, loc.Lon)
    }

    fmt.Println("\n--- Second lookup (cached) ---")
    loc, err = secured.Lookup("london")
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Result: %s (%.4f, %.4f)\n", loc.City, loc.Lat, loc.Lon)
    }
}
```

```txt
--- First lookup ---
  [api] calling external geocoding API for "london"
  [cache] stored "london"
Result: London (51.5074, -0.1278)

--- Second lookup (cached) ---
  [cache] hit for "london"
Result: London (51.5074, -0.1278)
```

The first call flows through all layers: access check passes, rate limit increments, cache misses, real API is called, result is cached. The second call hits the cache and never reaches the real API. The client code is identical for both -- it just calls `secured.Lookup("london")`.

The proxy stack:

```txt
Client --> AccessControlProxy --> RateLimitProxy --> CachingProxy --> RealGeocodingService
```

Each layer adds one control. The real service stays untouched. Swap the order, remove a layer, add a new one -- the interface contract stays the same.

## Types of Proxies

The pattern manifests differently depending on what kind of control is needed:

**Protection Proxy** -- controls access based on permissions. Our `AccessControlProxy` is this type. Checks credentials before forwarding.

**Caching Proxy** -- stores results to avoid repeating expensive operations. Our `CachingProxy` returns cached data instead of calling the real service.

**Virtual Proxy (Lazy Initialization)** -- defers creation of the real object until it's actually needed. Useful when the real service is expensive to initialize but might never be called.

**Remote Proxy** -- represents a remote object locally. The proxy handles serialization, network calls, and deserialization. gRPC clients are essentially remote proxies.

**Logging Proxy** -- records method calls for debugging or auditing. Intercepts every call, logs it, then forwards.

All five share the same structural pattern: same interface, wrap the real object, add control logic around delegation.

## Proxy vs Decorator

These patterns are structurally identical (both wrap an object behind the same interface) but differ in intent.

**Proxy** controls *access* to the real object. It decides *whether* to forward, *when* to forward, and *what to do instead* if it doesn't forward (return cached data, reject with error, defer creation). The proxy often *prevents* calls to the real object.

**Decorator** adds *behavior* around the real object. It always forwards the call -- it just does something extra before or after. Logging, timing, and metrics are decorator territory.

The test: does the wrapper sometimes *not* call the real object? If yes (cache hit, access denied, rate limited), it's a Proxy. If it always calls the real object and just adds something around it, it's a Decorator.

In practice, the line blurs. A caching proxy that always calls through on miss and adds cache-store logic on the response looks decorator-like. The distinction is useful for understanding intent, not for strict classification.

## When to Use It

The Proxy pattern fits when **you need to control access to an object without modifying it or its callers**:

- **Caching expensive operations** where repeated calls with the same input should return stored results. API call quotas, database queries, computation results.
- **Rate limiting external dependencies** to stay within quotas or prevent overwhelming downstream services.
- **Access control** where certain callers or contexts shouldn't reach the real service. Permission checks, role-based restrictions, tenant isolation.
- **Lazy initialization** where the real object is expensive to create and might not be needed. Database connections opened on first query, heavy computation deferred until results are requested.
- **Logging and auditing** where every access to a sensitive service must be recorded without modifying the service itself.

Skip it when:

- **You control the real service and can modify it directly.** If adding caching or access control inside the service is simpler and clearer, do that. Proxies add indirection that's only justified when the real service shouldn't change.
- **The control logic is always the same for every caller.** If every single consumer needs caching and rate limiting in the same configuration, embedding it in the service (or using middleware) might be more direct than a proxy stack.
- **The interface is large.** A proxy for a 15-method interface must implement all 15 methods, most just delegating. The boilerplate can become unwieldy.

## Pros and Cons

**Pros:**

- **Transparent to clients** -- the proxy implements the same interface. Callers don't know whether they're hitting the real service or a proxy. Swapping is seamless.
- **Composable layers** -- proxies stack. Cache inside rate limit inside access control. Each adds one concern. Reorder or remove layers without touching the others.
- **Real service stays unchanged** -- access control, caching, and rate limiting live in proxies, not in the service itself. The service does its job; proxies handle the politics.

**Cons:**

- **Increased latency for proxy logic** -- each proxy layer adds method calls, map lookups, and lock acquisitions. For high-frequency paths, this overhead matters.
- **Complexity in proxy interactions** -- when three proxies stack, understanding the order of operations (does access check happen before or after cache lookup?) requires reading the composition code carefully.
- **Stale cache risk** -- a caching proxy that doesn't expire entries serves stale data indefinitely. TTL, invalidation, and capacity management are real concerns the proxy must handle.

## Best Practices

- **Stack proxies in the right order.** Access control should be outermost (reject unauthorized requests before any work). The relative order of caching and rate limiting depends on what you're measuring: if the rate limit tracks API quota, put cache outside rate limit (`Cache(RateLimit(Real))`) so cache hits don't count. If the rate limit tracks client request volume, put it outside cache.
- **Keep each proxy focused on one concern.** A `CachingRateLimitAccessControlProxy` is a god object. Three separate proxies composed together are testable, removable, and reorderable.
- **Use `sync.RWMutex` for caching proxies.** Reads (cache hits) are far more frequent than writes (cache stores). An RWMutex allows concurrent reads while blocking only for writes, reducing contention.
- **Return the same interface type from constructors.** `NewCachingProxy(service GeocodingService) GeocodingService` (returning the interface, not `*CachingProxy`) makes the proxy truly transparent and composable without type assertions.

## Common Mistakes

**Caching errors.** If a `Lookup("unknown-city")` fails and the proxy caches the error, subsequent calls for that city fail immediately even if the service recovers. Only cache *successful* results unless you deliberately implement negative caching with short TTLs.

**Breaking the interface contract.** If the real service's `Lookup` always returns a non-nil `Location` on nil error, the proxy must maintain that guarantee. A caching proxy that returns `nil, nil` (cache miss without forwarding) violates the contract and causes nil pointer dereferences in callers.

**Proxy ordering that defeats the purpose.** If the rate limit proxy sits outside the caching proxy (as in our walkthrough: `RateLimit(Cache(Real))`), every cache hit still passes through the rate limiter and counts against the quota. For a system where you want rate limiting to track only *actual API calls*, the cache should be outermost: `Cache(RateLimit(Real))`. That way, cache hits return before reaching the rate limiter, and only misses count against the limit. Ordering is a design decision with real consequences -- think about what each layer is meant to protect.

**Forgetting that proxies are stateful.** A caching proxy accumulates entries. A rate limit proxy tracks call counts. In long-running services, this state grows unbounded unless you add TTL expiration, capacity limits, or periodic resets. A proxy without lifecycle management is a slow memory leak.

## Final Thoughts

The Proxy pattern places a controlled intermediary between clients and a real service. The proxy implements the same interface, so it's invisible to callers. It adds access control, caching, rate limiting, or lazy initialization without touching the service or the client. Each concern gets its own proxy, and proxies compose by wrapping each other.

Go's interfaces make proxies transparent by default -- implement the methods and you're a valid stand-in. No registration, no class hierarchy, no framework. A struct with a service field and the right method signatures is a working proxy.

The discipline is keeping proxies focused and ordered correctly. One concern per proxy. Access control outside, caching inside. Clear error propagation. Bounded state with lifecycle management. Get those right and the proxy stack is a clean, composable control layer that the real service never knows exists.

> **Control access without changing the thing you're protecting. Stand between, don't stand inside.**
