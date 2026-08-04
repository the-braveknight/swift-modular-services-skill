# Architecture Baseline

## Contents

- System shape
- Capability boundaries
- Target graph
- Allowed and forbidden dependencies
- Data ownership
- Deployment topology

## System Shape

Use a modular monolith as the default architecture:

```text
Executable composition root
├── API runtime
├── Worker runtime
├── Migration runtime
└── Administrative commands
        │
        ▼
Capability SPM targets
├── Core/platform abstractions
├── Users
├── Authentication
├── Entitlements
└── Payments
        │
        ▼
Module-owned persistence and provider adapters
```

Use the names above as examples, not required modules. Name actual targets after the product's business capabilities.

Keep API delivery separate from domain capability targets. Keep executable assembly separate from the API library so runtime lifecycle and concrete wiring do not leak into delivery or domain code.

## Capability Boundaries

Give each module:

- One cohesive business responsibility
- Ownership of its invariants and domain vocabulary
- Public commands, values, typed errors, service protocols, and workflow activities
- Exclusive ownership of its repositories and writable data
- Its own provider adapters when it talks to external systems

Do not create modules merely for shared email, HTTP, repositories, or other technical mechanisms. Prefer a narrow capability port in the business module that needs it, implemented by a provider-named adapter in that same module.

Do not split every module into `ModuleCore`, `ModuleContracts`, and `ModuleImplementation` targets. Start with one target. Introduce a separate contract target only when independent packaging, release, or client generation makes it necessary.

## Target Graph

Use `Package.swift` target dependencies to enforce module edges:

```text
Core            -> —
Users           -> Core
Authentication  -> Core, Users
Entitlements    -> Core
Payments        -> Core, Entitlements
API             -> required capability modules
CLI             -> API and every concrete module it assembles
```

Keep the graph acyclic. An edge means the consumer may import the provider module's public domain and port types. It does not authorize the consumer to reach the provider's adapters or persistence.

Use `Core` only for stable platform abstractions or primitives genuinely shared across capabilities. Do not move business behavior into `Core` to resolve a cycle.

## Allowed Dependencies

- Import another module's public domain entities, commands, errors, or enums when the target graph permits it.
- Inject another module's service protocol for synchronous in-process collaboration.
- Execute another module's activity container from a durable workflow.
- Construct all modules' concrete implementations in the executable composition root.
- Store another module's stable identifier without taking ownership of the referenced entity.

Example:

```text
Payments target depends on Entitlements
Payments workflow executes EntitlementActivities.UpsertEntitlement
EntitlementActivities wraps EntitlementService
EntitlementService alone reaches Entitlements repositories
```

No event publication is required for this collaboration.

## Forbidden Dependencies

- Never import another module's concrete service, repository, statement, database context, migration, or provider adapter from a capability module.
- Never let a controller or workflow call a repository.
- Never let a workflow call a domain or orchestration service directly.
- Never let an activity start workflows, coordinate multiple services, or contain business rules.
- Never include another module's repositories in a module's service context.
- Never read or write another module's tables directly.
- Avoid cross-module database foreign keys; enforce cross-capability integrity through the owning service boundary.
- Never create bidirectional target dependencies.

The executable composition root is the intentional exception for concrete construction. It may import adapters to assemble the application but must not contain business rules.

## Data Ownership

Let a capability own its schema or table set, migrations, statements, repositories, and service transaction context. Sharing a physical PostgreSQL server does not imply shared ownership.

Keep strong atomic invariants inside one module-owned transaction. For cross-module operations:

- Call the owning service protocol when ordinary in-process coordination is sufficient.
- Use a durable workflow plus module-owned activities when steps need retries, waiting, or recovery.
- Use messaging only when asynchronous delivery is a product or operational requirement.

Do not introduce cross-module foreign keys to simulate ownership. Carry stable identifiers and validate through public capability behavior where required.

## Deployment Topology

Treat topology as an adapter decision made after boundaries:

- **Modular monolith:** direct typed service ports and capability activities; one deployed application may expose several runtime modes.
- **Separate service:** remote client/server adapters surround the capability's public behavior; the service gets its own executable composition root and data credentials.
- **Asynchronous integration:** durable messages surround a real asynchronous contract; they do not replace clear ownership.

Keep modules in-process by default. Extract only for demonstrated scaling, isolation, security, availability, release, runtime, or organizational needs.
