+++
date = '2026-02-21'
title = 'Facade Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'structural']
+++

Think about onboarding a new user in a SaaS application. Behind the scenes, it requires: creating a database record, provisioning a storage bucket, generating API credentials, sending a welcome email, setting up default permissions, and initializing the user's dashboard. Six subsystems, each with its own API, its own error handling, its own initialization sequence. Every handler that needs to "create a user" must orchestrate all six steps in the right order.

Without a facade, every call site reimplements this orchestration. The HTTP handler knows about the database, the storage service, the credential generator, the email sender, the permission system, and the dashboard service. Change the order (send email after dashboard instead of before) and you're editing five different files. Add a seventh step and you're modifying every function that creates users.

The **Facade pattern** wraps this complexity behind a single, clean entry point: `userService.Onboard(user)`. One method. The facade knows the subsystems, the order, the error handling. The client knows the facade. That's the entire relationship.

This is one of the simplest structural patterns -- and one of the most useful. Go makes it even simpler: a struct with subsystem fields and methods that orchestrate them. No interfaces required, no inheritance, no framework. In this article, we'll build a user onboarding facade, show how it simplifies client code, and draw the line between a healthy facade and a god object.

## What Is the Facade Pattern?

> "Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use."
> -- Gang of Four

The intent: **hide the complexity of a subsystem behind a simple, high-level API.** The facade doesn't add new functionality -- it just coordinates existing components so the client doesn't have to. The subsystems remain unchanged and independently usable. The facade is an *optional* convenience layer, not a mandatory gate.

Without a facade, user onboarding looks like this:

```go
func handleSignup(w http.ResponseWriter, r *http.Request) {
    user := parseRequest(r)

    dbUser, err := db.CreateUser(user.Email, user.Name)
    if err != nil { ... }

    bucket, err := storage.CreateBucket(dbUser.ID)
    if err != nil { ... }

    creds, err := auth.GenerateAPIKey(dbUser.ID)
    if err != nil { ... }

    err = permissions.SetDefaults(dbUser.ID, "free-tier")
    if err != nil { ... }

    err = email.SendWelcome(user.Email, creds.Key)
    if err != nil { ... }

    err = dashboard.Initialize(dbUser.ID)
    if err != nil { ... }

    respond(w, dbUser)
}
```

Six subsystem calls, each with error handling, in a specific order. This HTTP handler knows far too much. The same sequence is needed in the admin API, the CLI tool, and the batch import job. That's three copies of the same orchestration logic.

With a facade:

```go
func handleSignup(w http.ResponseWriter, r *http.Request) {
    user := parseRequest(r)
    result, err := onboarding.CreateUser(user.Email, user.Name)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    respond(w, result)
}
```

One call. The handler doesn't know about storage buckets, API keys, or permission defaults. The facade handles it.

## Core Components

Three participants -- the simplest structure of any GoF pattern.

**Facade** -- the simplified interface that coordinates subsystems. Exposes high-level methods that clients call.

```go
type OnboardingService struct {
    db          *UserDB
    storage     *StorageService
    auth        *AuthService
    permissions *PermissionService
    email       *EmailService
    dashboard   *DashboardService
}
```

**Subsystems** -- the actual components that do the work. They're independent, potentially complex, and don't know the facade exists.

**Client** -- interacts only with the facade. Doesn't import, reference, or configure subsystems directly.

The flow:

```
Client --> Facade.CreateUser() --> DB, Storage, Auth, Permissions, Email, Dashboard
```

The facade is the single entry point. Subsystems remain independently testable and reusable. The client stays simple.

## Code Walkthrough: A User Onboarding Service

We're building an onboarding facade that coordinates six subsystems to create a fully provisioned user account.

### The Subsystems

Each subsystem is a focused service with its own responsibilities:

```go
type UserDB struct{}

func (db *UserDB) Insert(email, name string) (string, error) {
    id := fmt.Sprintf("user_%d", time.Now().UnixNano()%10000)
    fmt.Printf("  [db] created user %s (%s)\n", id, email)
    return id, nil
}
```

```go
type StorageService struct{}

func (s *StorageService) CreateBucket(userID string) error {
    fmt.Printf("  [storage] provisioned bucket for %s\n", userID)
    return nil
}
```

```go
type AuthService struct{}

func (a *AuthService) GenerateAPIKey(userID string) (string, error) {
    key := "ak_" + userID + "_secret"
    fmt.Printf("  [auth] generated API key for %s\n", userID)
    return key, nil
}
```

```go
type PermissionService struct{}

func (p *PermissionService) SetDefaults(userID, tier string) error {
    fmt.Printf("  [permissions] set %s defaults for %s\n", tier, userID)
    return nil
}
```

```go
type EmailService struct{}

func (e *EmailService) SendWelcome(email, apiKey string) error {
    fmt.Printf("  [email] sent welcome to %s\n", email)
    return nil
}
```

```go
type DashboardService struct{}

func (d *DashboardService) Initialize(userID string) error {
    fmt.Printf("  [dashboard] initialized for %s\n", userID)
    return nil
}
```

Six subsystems, each doing one thing. They don't know about each other or the facade.

### The Facade

The onboarding service coordinates them:

```go
type OnboardingResult struct {
    UserID string
    APIKey string
}

type OnboardingService struct {
    db          *UserDB
    storage     *StorageService
    auth        *AuthService
    permissions *PermissionService
    email       *EmailService
    dashboard   *DashboardService
}

func NewOnboardingService(
    db *UserDB,
    storage *StorageService,
    auth *AuthService,
    permissions *PermissionService,
    email *EmailService,
    dashboard *DashboardService,
) *OnboardingService {
    return &OnboardingService{
        db:          db,
        storage:     storage,
        auth:        auth,
        permissions: permissions,
        email:       email,
        dashboard:   dashboard,
    }
}

func (s *OnboardingService) CreateUser(email, name string) (*OnboardingResult, error) {
    userID, err := s.db.Insert(email, name)
    if err != nil {
        return nil, fmt.Errorf("onboarding: db insert failed: %w", err)
    }

    if err := s.storage.CreateBucket(userID); err != nil {
        return nil, fmt.Errorf("onboarding: storage setup failed: %w", err)
    }

    apiKey, err := s.auth.GenerateAPIKey(userID)
    if err != nil {
        return nil, fmt.Errorf("onboarding: auth setup failed: %w", err)
    }

    if err := s.permissions.SetDefaults(userID, "free-tier"); err != nil {
        return nil, fmt.Errorf("onboarding: permissions failed: %w", err)
    }

    if err := s.email.SendWelcome(email, apiKey); err != nil {
        return nil, fmt.Errorf("onboarding: welcome email failed: %w", err)
    }

    if err := s.dashboard.Initialize(userID); err != nil {
        return nil, fmt.Errorf("onboarding: dashboard init failed: %w", err)
    }

    return &OnboardingResult{UserID: userID, APIKey: apiKey}, nil
}
```

One method. Six subsystem calls in the right order. Each error is wrapped with context. The facade returns a clean result struct. The client never touches subsystems directly.

### Using It

```go
func main() {
    svc := NewOnboardingService(
        &UserDB{},
        &StorageService{},
        &AuthService{},
        &PermissionService{},
        &EmailService{},
        &DashboardService{},
    )

    fmt.Println("--- Creating user ---")
    result, err := svc.CreateUser("alice@example.com", "Alice")
    if err != nil {
        fmt.Println("FAILED:", err)
        return
    }
    fmt.Printf("\nSuccess: UserID=%s, APIKey=%s\n", result.UserID, result.APIKey)
}
```

```txt
--- Creating user ---
  [db] created user user_4821 (alice@example.com)
  [storage] provisioned bucket for user_4821
  [auth] generated API key for user_4821
  [permissions] set free-tier defaults for user_4821
  [email] sent welcome to alice@example.com
  [dashboard] initialized for user_4821

Success: UserID=user_4821, APIKey=ak_user_4821_secret
```

The client wrote three lines. The facade orchestrated six subsystems. That's the pattern doing its job.

### When a Step Fails

If the storage service fails:

```go
func (s *StorageService) CreateBucket(userID string) error {
    return fmt.Errorf("bucket quota exceeded")
}
```

```txt
--- Creating user ---
  [db] created user user_4821 (alice@example.com)
FAILED: onboarding: storage setup failed: bucket quota exceeded
```

The facade stops at the first failure and returns a wrapped error with context. The client sees exactly what went wrong without knowing the internal structure.

## Facade vs Adapter vs Mediator

These three patterns all "sit between" client code and other components, but with different intents.

**Facade** simplifies access to a complex subsystem. The subsystems exist independently; the facade is an optional convenience. It doesn't change interfaces -- it just coordinates. "One easy method instead of ten hard ones."

**Adapter** makes an incompatible interface work with an expected one. It's about translation, not simplification. The adaptee has one shape; the client expects another. The adapter bridges the mismatch.

**Mediator** coordinates *peer objects* that communicate with each other. It controls interaction logic between components that are aware of each other through the mediator. Facade's subsystems don't interact; they're orchestrated top-down.

The test: if the subsystems are independent and the client just needs a simpler way to use them together, it's a Facade. If you're translating one interface to another, it's an Adapter. If components need to coordinate with each other (not just be called in sequence), it's a Mediator.

## When to Use It

The Facade pattern fits when **a subsystem is complex and the client needs a simpler entry point**:

- **Multi-step workflows** that are repeated across multiple call sites (onboarding, checkout, provisioning). The facade centralizes the sequence, and every call site gets it right.
- **Library or SDK simplification** where the underlying package has a large, flexible API but most users need one or two common workflows. The facade provides those workflows.
- **Decoupling layers** in a layered architecture. The HTTP layer depends on a service facade, not on six individual subsystem packages. Refactoring a subsystem doesn't ripple up to the handlers.
- **Legacy system wrapping** where a messy internal API needs a clean external interface for new consumers.

Skip it when:

- **The subsystem is already simple.** If using the subsystems directly is 2-3 calls with obvious ordering, a facade adds indirection for no benefit.
- **Clients need fine-grained control.** If different callers need different subsets of the workflow (one skips email, another skips dashboard), a rigid facade that does "everything" doesn't help. Consider making the facade methods more granular, or let clients use subsystems directly.
- **You're hiding a design problem.** A facade over a badly designed subsystem is a bandage. If the subsystem's API is confusing because it's poorly structured, fix the structure -- don't just wallpaper over it.

## Pros and Cons

**Pros:**

- **Reduces client coupling** -- the client depends on one facade, not six subsystem packages. Change a subsystem's API and only the facade updates, not every consumer.
- **Centralizes orchestration logic** -- the order of operations, error handling strategy, and step coordination live in one place. No copy-paste across handlers.
- **Improves readability** -- `svc.CreateUser(email, name)` is instantly understandable. The six-step implementation is discoverable but not in-your-face.
- **Subsystems remain independently usable** -- the facade is additive. Power users who need fine-grained control can still use subsystems directly. The facade doesn't restrict access; it offers a shortcut.

**Cons:**

- **Can become a god object** -- if the facade accumulates too many methods (CreateUser, DeleteUser, UpdateUser, SuspendUser, MigrateUser, ExportUser), it becomes the most coupled type in the system. Everything touches it.
- **Hides important complexity** -- if the orchestration has nuances that callers should understand (partial failure handling, compensating transactions), the facade's simplicity can be misleading.
- **Single point of change for unrelated reasons** -- if the email service API changes, the storage service API changes, and the permission model changes, the facade is modified for three unrelated reasons. It attracts changes from every subsystem it wraps.

## Best Practices

- **Keep the facade focused on one workflow.** `OnboardingService` handles onboarding. It doesn't also handle user deletion, password resets, or account migration. Separate workflows get separate facades. This prevents the god object problem.
- **Accept subsystem dependencies through the constructor.** `NewOnboardingService(db, storage, auth, ...)` makes dependencies explicit and testable. Don't create subsystems inside the facade -- inject them.
- **Wrap errors with step context.** `fmt.Errorf("onboarding: storage setup failed: %w", err)` tells the caller *which step* failed without exposing the subsystem's internal error types directly.
- **Don't force clients through the facade.** The facade is a convenience, not a gate. If a power user needs to call `storage.CreateBucket()` directly for a custom workflow, that should remain possible. The facade doesn't own the subsystems.

## Common Mistakes

**Letting the facade grow into a god object.** A single `UserService` that handles creation, deletion, updates, password resets, MFA setup, billing integration, and export has 15 methods touching 10 subsystems. It changes for every reason imaginable. The fix: split into focused facades -- `OnboardingService`, `AccountDeletionService`, `BillingService` -- each wrapping a specific workflow.

**Adding business logic to the facade.** A facade that validates email formats, computes subscription pricing, or decides which tier a user belongs to has taken on domain logic that belongs in the subsystems or a dedicated domain service. The facade *orchestrates*; it doesn't *decide*. Keep it to sequencing and delegation.

**Not handling partial failure.** The walkthrough stops on first error -- which is clean but incomplete for production. If `db.Insert` succeeds but `storage.CreateBucket` fails, the user record exists with no bucket. Production facades often need compensating actions (delete the user record on storage failure) or at minimum, logging/alerting about the inconsistent state. The simple "stop on error" approach is a starting point, not a complete solution.

**Creating a facade before the complexity exists.** If you have two subsystems called in sequence, wrapping them in a facade is premature abstraction. Let the complexity grow naturally. When you find yourself copying the same three-step orchestration across multiple handlers, *then* extract a facade. Don't pre-build facades for complexity that might never arrive.

## Final Thoughts

The Facade pattern wraps a complex subsystem behind a simple, high-level API. The subsystems do the real work. The facade decides the order and handles the plumbing. The client gets one clean method instead of six fragile steps.

Go makes this feel natural because a facade is just a struct with fields (the subsystems) and methods (the workflows). No interfaces required unless you need testability or swappability. No framework, no base class -- just composition and delegation.

The discipline is keeping facades focused. One workflow per facade. Orchestration, not logic. Delegation, not computation. The moment a facade starts making business decisions or growing to 20 methods, it's time to split.

> **Give complexity a simple front door -- but don't let the door become bigger than the house.**
