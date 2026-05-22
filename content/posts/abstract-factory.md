+++
date = '2025-07-19'
title = 'Abstract Factory Design Pattern in Go'
series = ['design patterns']
tags = ['golang', 'design-patterns', 'creational']
+++

Think about a cloud infrastructure tool that provisions resources across AWS, GCP, and Azure. Each provider needs a compute instance, a storage bucket, and a network configuration. These resources must be compatible within the same provider -- an AWS security group can't attach to a GCP instance. The tool needs to create all three resources for whichever provider the user selects, and guarantee they work together.

The obvious approach scatters provider checks everywhere: `if provider == "aws"` before every resource creation, repeated across every function that provisions infrastructure. Add a fourth provider and you're updating dozens of files. Worse, nothing prevents accidentally mixing an AWS compute instance with a GCP network config -- the type system doesn't enforce it.

The **Abstract Factory** pattern solves both problems at once. A single factory interface declares methods for creating each resource type. Each provider gets its own factory implementation that produces compatible resources. The client code works through the factory interface -- swap the factory, and the entire family of resources changes together. No scattering, no mixing, no modification of existing code.

In this article, we'll build a cloud provisioner with AWS and GCP factories, explore how the pattern prevents cross-provider mixing, and show the Go-idiomatic functional variation for lighter use cases.

## What Is the Abstract Factory Pattern?

> "Provide an interface for creating families of related or dependent objects without specifying their concrete classes."
> -- Gang of Four

The intent: **create groups of related objects that must work together, through a single factory interface, so the entire family can be swapped by swapping the factory.** The client code never references concrete types. It works through product interfaces and asks the factory to produce them.

The key insight that separates this from Factory Method: Factory Method creates *one* product at a time, and each creator type handles one product. Abstract Factory creates *multiple* products that belong together as a family. The factory interface has multiple creation methods, one per product type in the family.

Without this pattern, provisioning looks like this:

```go
func provision(provider string) {
    var compute Compute
    var storage Storage
    var network Network

    switch provider {
    case "aws":
        compute = &EC2Instance{region: "us-east-1"}
        storage = &S3Bucket{region: "us-east-1"}
        network = &AWSNetwork{vpc: "vpc-default"}
    case "gcp":
        compute = &GCEInstance{zone: "us-central1-a"}
        storage = &GCSBucket{project: "my-project"}
        network = &GCPNetwork{subnetwork: "default"}
    }

    compute.Start()
    storage.Create()
    network.Configure()
}
```

Every function that provisions resources repeats this switch. Add Azure and you modify them all. Nothing structurally prevents `&EC2Instance{}` from appearing alongside `&GCPNetwork{}` -- the compiler won't catch it.

With an Abstract Factory:

```go
func provision(factory CloudFactory) {
    compute := factory.CreateCompute()
    storage := factory.CreateStorage()
    network := factory.CreateNetwork()

    compute.Start()
    storage.Create()
    network.Configure()
}
```

The factory guarantees compatibility. The client function doesn't know or care which provider it's using.

## Core Components

Five participants.

**AbstractFactory** -- the interface declaring creation methods for each product type in the family.

```go
type CloudFactory interface {
    CreateCompute() Compute
    CreateStorage() Storage
    CreateNetwork() Network
}
```

**ConcreteFactory** -- implements the factory interface for a specific family. All products it creates are compatible with each other.

**AbstractProduct** -- interfaces for each product type. The client depends on these.

```go
type Compute interface {
    Start() error
    Stop() error
    String() string
}

type Storage interface {
    Create() error
    String() string
}

type Network interface {
    Configure() error
    String() string
}
```

**ConcreteProduct** -- the actual implementations, tied to a specific family (AWS, GCP).

**Client** -- works with the factory and product interfaces. Never references concrete types.

The flow:

```
Client --> CloudFactory.CreateCompute() --> AWS/GCP Compute (as Compute interface)
       --> CloudFactory.CreateStorage() --> AWS/GCP Storage (as Storage interface)
       --> CloudFactory.CreateNetwork() --> AWS/GCP Network (as Network interface)
```

Swap the factory, swap the family. The client code stays identical.

## Code Walkthrough: A Cloud Resource Provisioner

We're building a tool that provisions compute, storage, and network resources for different cloud providers. Each provider's resources are guaranteed to be compatible.

### Product Interfaces

```go
type Compute interface {
    Start() error
    Stop() error
    String() string
}

type Storage interface {
    Create() error
    String() string
}

type Network interface {
    Configure() error
    String() string
}
```

Three product types. Each has behavior methods and a `String()` for display.

### AWS Family

```go
type EC2Instance struct {
    region       string
    instanceType string
}

func (e *EC2Instance) Start() error {
    fmt.Printf("  [aws] starting EC2 %s in %s\n", e.instanceType, e.region)
    return nil
}

func (e *EC2Instance) Stop() error {
    fmt.Printf("  [aws] stopping EC2 %s\n", e.instanceType)
    return nil
}

func (e *EC2Instance) String() string {
    return fmt.Sprintf("EC2{%s, %s}", e.instanceType, e.region)
}
```

```go
type S3Bucket struct {
    region string
    name   string
}

func (s *S3Bucket) Create() error {
    fmt.Printf("  [aws] creating S3 bucket %s in %s\n", s.name, s.region)
    return nil
}

func (s *S3Bucket) String() string {
    return fmt.Sprintf("S3{%s, %s}", s.name, s.region)
}
```

```go
type AWSNetwork struct {
    vpcID  string
    region string
}

func (n *AWSNetwork) Configure() error {
    fmt.Printf("  [aws] configuring VPC %s in %s\n", n.vpcID, n.region)
    return nil
}

func (n *AWSNetwork) String() string {
    return fmt.Sprintf("VPC{%s, %s}", n.vpcID, n.region)
}
```

All three share a region -- they're designed to work together within the same AWS deployment.

### GCP Family

```go
type GCEInstance struct {
    zone        string
    machineType string
}

func (g *GCEInstance) Start() error {
    fmt.Printf("  [gcp] starting GCE %s in %s\n", g.machineType, g.zone)
    return nil
}

func (g *GCEInstance) Stop() error {
    fmt.Printf("  [gcp] stopping GCE %s\n", g.machineType)
    return nil
}

func (g *GCEInstance) String() string {
    return fmt.Sprintf("GCE{%s, %s}", g.machineType, g.zone)
}
```

```go
type GCSBucket struct {
    project string
    name    string
}

func (g *GCSBucket) Create() error {
    fmt.Printf("  [gcp] creating GCS bucket %s in project %s\n", g.name, g.project)
    return nil
}

func (g *GCSBucket) String() string {
    return fmt.Sprintf("GCS{%s, %s}", g.name, g.project)
}
```

```go
type GCPNetwork struct {
    subnetwork string
    project    string
}

func (n *GCPNetwork) Configure() error {
    fmt.Printf("  [gcp] configuring subnetwork %s in project %s\n", n.subnetwork, n.project)
    return nil
}

func (n *GCPNetwork) String() string {
    return fmt.Sprintf("Subnet{%s, %s}", n.subnetwork, n.project)
}
```

GCP resources share a project -- again, designed for internal compatibility.

### The Factory Interface and Implementations

```go
type CloudFactory interface {
    CreateCompute() Compute
    CreateStorage() Storage
    CreateNetwork() Network
}
```

**AWS Factory:**

```go
type AWSFactory struct {
    Region string
}

func (f *AWSFactory) CreateCompute() Compute {
    return &EC2Instance{region: f.Region, instanceType: "t3.medium"}
}

func (f *AWSFactory) CreateStorage() Storage {
    return &S3Bucket{region: f.Region, name: "app-data"}
}

func (f *AWSFactory) CreateNetwork() Network {
    return &AWSNetwork{vpcID: "vpc-0abc123", region: f.Region}
}
```

**GCP Factory:**

```go
type GCPFactory struct {
    Project string
    Zone    string
}

func (f *GCPFactory) CreateCompute() Compute {
    return &GCEInstance{zone: f.Zone, machineType: "e2-medium"}
}

func (f *GCPFactory) CreateStorage() Storage {
    return &GCSBucket{project: f.Project, name: "app-data"}
}

func (f *GCPFactory) CreateNetwork() Network {
    return &GCPNetwork{subnetwork: "default", project: f.Project}
}
```

Each factory encapsulates provider-specific configuration (region for AWS, project+zone for GCP) and produces resources that are guaranteed compatible. The AWS factory will never accidentally produce a GCP bucket.

### The Client

```go
func provisionEnvironment(factory CloudFactory) {
    compute := factory.CreateCompute()
    storage := factory.CreateStorage()
    network := factory.CreateNetwork()

    fmt.Printf("Provisioning: %s, %s, %s\n", compute, storage, network)
    network.Configure()
    storage.Create()
    compute.Start()
}
```

The client doesn't import AWS or GCP packages. It works entirely through interfaces. Testing means passing a mock factory. Switching providers means changing one line.

### Running It

```go
func main() {
    fmt.Println("--- AWS Environment ---")
    provisionEnvironment(&AWSFactory{Region: "us-east-1"})

    fmt.Println("\n--- GCP Environment ---")
    provisionEnvironment(&GCPFactory{Project: "my-project", Zone: "us-central1-a"})
}
```

```
--- AWS Environment ---
Provisioning: EC2{t3.medium, us-east-1}, S3{app-data, us-east-1}, VPC{vpc-0abc123, us-east-1}
  [aws] configuring VPC vpc-0abc123 in us-east-1
  [aws] creating S3 bucket app-data in us-east-1
  [aws] starting EC2 t3.medium in us-east-1

--- GCP Environment ---
Provisioning: GCE{e2-medium, us-central1-a}, GCS{app-data, my-project}, Subnet{default, my-project}
  [gcp] configuring subnetwork default in project my-project
  [gcp] creating GCS bucket app-data in project my-project
  [gcp] starting GCE e2-medium in us-central1-a
```

Same client function, two completely different infrastructure stacks. No conditionals, no provider-specific imports in the client, no risk of cross-provider mixing.

## Functional Variation: Lighter Factories

For systems where the full interface-based approach feels heavy, Go's first-class functions offer a lighter alternative:

```go
type CloudFactoryFunc func() (Compute, Storage, Network)

func awsFactory(region string) CloudFactoryFunc {
    return func() (Compute, Storage, Network) {
        return &EC2Instance{region: region, instanceType: "t3.medium"},
            &S3Bucket{region: region, name: "app-data"},
            &AWSNetwork{vpcID: "vpc-0abc123", region: region}
    }
}
```

Usage:

```go
factory := awsFactory("us-east-1")
compute, storage, network := factory()
```

This skips the interface ceremony while preserving the core guarantee: one function call produces a compatible family. The tradeoff is that you lose the ability to pass a `CloudFactory` interface to generic client code -- the function signature *is* the contract, not a named interface. Good for internal code, less good for library APIs.

## When to Use It

The Abstract Factory pattern fits when **you need to create groups of objects that must be used together, and the family can be swapped as a unit**:

- **Multi-provider systems** -- cloud infrastructure, payment gateways, notification channels. Each provider needs multiple compatible objects (connections, configs, clients) that must come from the same family.
- **Platform-dependent code** -- desktop apps targeting Windows/Mac/Linux, mobile apps targeting iOS/Android. Each platform has its own implementations of shared abstractions (file system, notifications, UI components).
- **Test/production swapping** -- a "real" factory for production and a "mock" factory for tests that produces in-memory fakes of the entire resource family. One swap changes everything.
- **Themed or branded configurations** -- a "premium" factory produces premium-tier resources, a "free" factory produces free-tier resources, and the client code doesn't branch on tier.

Skip it when:

- **You only need one product type.** If there's no "family" -- just one object to create -- Factory Method or Simple Factory is simpler.
- **Products don't need to be compatible.** If mixing a GCP bucket with an AWS instance is actually valid in your system, the consistency guarantee is a constraint, not a benefit.
- **The system is small and stable.** Two providers with three resources each means 6 concrete types, 3 interfaces, and 2 factories. For a script that only ever targets one provider, that's a lot of indirection for nothing.

## Pros and Cons

**Pros:**

- **Family consistency guaranteed** -- the factory produces only compatible objects. Cross-family mixing is structurally impossible through the factory interface.
- **Single swap point** -- changing the entire family means changing one factory assignment. The client code, the provisioning logic, the error handling -- none of it changes.
- **Open/Closed compliance** -- adding Azure means writing an `AzureFactory` and Azure product types. No existing code is modified.
- **Testability** -- inject a `MockFactory` that returns in-memory fakes of all products. Test the provisioning logic without hitting real cloud APIs.

**Cons:**

- **Boilerplate scales multiplicatively** -- every new product type means a new method on *every* factory. Three providers with four product types means twelve concrete types, four interfaces, and three factories. It adds up.
- **Rigid family boundaries** -- if a legitimate use case requires one AWS product with one GCP product (hybrid cloud), the pattern actively prevents it. You'd need to break out of the factory to accomplish it.
- **Adding a product to the family breaks all factories** -- if you add `CreateLoadBalancer()` to `CloudFactory`, every existing concrete factory must implement it. This is the flip side of consistency -- the interface change cascades.

## Best Practices

- **Keep factories stateless or configuration-only.** Factories should hold provider configuration (region, project, credentials) but not mutable state. They create products; they don't manage lifecycles.
- **Use a registry for runtime factory selection.** Rather than `if provider == "aws"` scattered in main, register factories by name and look them up. This keeps the selection point centralized and extensible.

```go
var factories = map[string]CloudFactory{
    "aws": &AWSFactory{Region: "us-east-1"},
    "gcp": &GCPFactory{Project: "my-project", Zone: "us-central1-a"},
}

func GetFactory(provider string) (CloudFactory, error) {
    f, ok := factories[provider]
    if !ok {
        return nil, fmt.Errorf("unsupported provider: %s", provider)
    }
    return f, nil
}
```

- **Return errors from product creation methods when creation can fail.** `CreateCompute() (Compute, error)` is more honest than panicking inside the factory. Cloud resource creation fails; the interface should reflect that.
- **Consider the functional variant for internal code.** If you're the only consumer and don't need a named interface for testing or dependency injection, a factory function reduces boilerplate significantly.

## Common Mistakes

**Adding the factory interface before you have two families.** If you only support AWS today and "might add GCP later," you're paying the abstraction cost now for a benefit that may never materialize. Start with direct construction, and introduce the factory when the second family actually arrives. Premature abstraction is real.

**Putting business logic in the factory.** A factory that validates resource limits, checks quotas, or applies billing rules has taken on too many responsibilities. The factory creates and configures. Validation and business rules belong in the client or in a separate layer. Keep factories about *construction*, not *policy*.

**Not evolving the product interfaces.** Over time, AWS compute might need capabilities GCP doesn't have. If you keep expanding the `Compute` interface to accommodate one provider's features, the other providers get forced to implement no-op methods. Watch for interface bloat and consider splitting into base interfaces plus provider-specific extensions.

**Forgetting that interface changes cascade.** Adding `CreateDNS()` to `CloudFactory` means *every* concrete factory must implement it immediately -- even providers that don't have DNS support yet. Plan the interface carefully upfront, or accept that adding products will touch all factories. There's no free lunch here.

## Final Thoughts

The Abstract Factory pattern creates families of objects that are guaranteed to work together, through a single swappable interface. Change the factory, change the family. The client code stays clean, the products stay compatible, and new families slot in without modifying existing code.

In Go, the pattern maps onto a factory interface with multiple methods, each returning a product interface. It's more verbose than languages with abstract classes and inheritance, but the explicitness is the point -- you can see exactly what each factory produces and what each product does, without hidden dispatching.

The honest test: if your system needs multiple providers (or themes, or tiers) that each produce a coordinated set of objects, Abstract Factory is the right tool. If you only have one family today and aren't sure about tomorrow, start simple. The pattern is easy to introduce when the second family arrives; introducing it preemptively costs you boilerplate for speculation.

> **When objects must come as a family, give them a family factory -- and let the client stay blissfully unaware of which family it got.**
