---
name: swift-modular-services
description: Design, create, review, and evolve modular Swift backends using capability-based Swift Package Manager targets, Domain/Ports/Adapters module structure, typed service protocols, module-owned PostgreSQL persistence, contract-first APIs, Temporal workflows and activities, and executable composition roots. Use when creating a whole Swift backend, adding a backend module or use case, defining or reviewing module boundaries and dependency graphs, wiring cross-module collaboration, designing durable workflows, building a modular monolith, or adapting an existing module into a separately deployable HTTP/gRPC service without changing domain ownership.
---

# Swift Modular Services

## Purpose

Build capability modules with stable public contracts, then compose them in one process or place a network adapter across the same boundary when separate deployment is justified. Use the modular-monolith structure as the baseline; treat remote services, messaging, and event publication as conditional extensions.

## Source of Truth

Apply guidance in this order:

1. Follow the target repository's `AGENTS.md`, `CLAUDE.md`, README, package manifest, and established source/test conventions.
2. Preserve existing architectural decisions unless the user explicitly requests a redesign.
3. Use this skill's baseline for greenfield work or gaps not answered by the repository.
4. Introduce distributed-system patterns only when the requested topology requires them.

Do not impose event publishing, separate contract targets, brokers, outboxes, sagas, or projections on an in-process backend that does not need them.

## Baseline Invariants

- Define modules by business capability, not technical layer.
- Represent each capability as one SPM target by default.
- Keep the target dependency graph explicit and acyclic.
- Structure each substantial capability as `Domain/`, `Ports/`, and `Adapters/`.
- Keep domain values independent of HTTP, persistence, configuration, workflow engines, and provider SDKs.
- Let a module expose public domain types and ports; keep its adapters as internal plumbing.
- Allow a consumer module to depend on another module's public `Domain/` and `Ports/` contracts.
- Never let a consumer reach another module's repository, database context, concrete provider adapter, or tables.
- Let each module own its persistence schema, migrations, statements, repositories, and transaction boundary.
- Assemble concrete implementations in executable composition roots.
- Prefer direct typed service-port calls across in-process modules.
- From a durable workflow, call an activity owned by the target module; never call its concrete service or repository directly.
- Add events only for a real asynchronous consumer or fan-out requirement.

Read [architecture-baseline.md](references/architecture-baseline.md) before designing boundaries, reviewing imports, or choosing a dependency direction.

## Workflow

### 1. Inspect Before Designing

Inspect repository instructions, `Package.swift`, `Sources/`, `Tests/`, runtime entry points, persistence ownership, configuration wiring, public API contracts, and existing imports. Identify the actual conventions before suggesting new targets or folders.

For greenfield work, identify:

- Business capabilities and invariants
- Owned data and transaction boundaries
- HTTP, worker, migration, and administrative runtimes
- Cross-capability calls and workflow interactions
- Expected deployment topology now and later

### 2. Draw the Capability Graph

Record for each module:

- Responsibility and invariants
- Domain types and commands it owns
- Tables or external provider state it writes
- Public service protocols and workflow activities it exposes
- Other modules whose public contracts it consumes
- Runtime modes that construct it

Draw the SPM target graph before implementation. Resolve cycles by moving responsibility, introducing an orchestration owner, or merging one consistency boundary—not by importing adapters in both directions.

### 3. Choose the Collaboration Path

Use the narrowest path that matches the existing runtime:

1. **Same module:** call the owning service.
2. **Cross-module, in-process:** inject the provider module's service protocol at the composition root.
3. **Cross-module, inside a durable workflow:** execute the provider module's activity container.
4. **Cross-process, immediate result:** use a versioned HTTP/gRPC adapter with explicit remote-failure semantics.
5. **Cross-process, asynchronous need:** use a durable message only when independent delivery or fan-out is required.

Do not describe a network call as if it had local latency and failure behavior.

### 4. Implement a Complete Vertical Slice

Implement in ownership order:

1. Domain entity/value, command, and typed errors
2. Repository and service ports
3. Service implementation and owned transaction context
4. Migration, statements, and repository adapter when persistence is needed
5. Provider adapter when an external capability is needed
6. Activity container or orchestration/workflow only when the use case requires durable execution
7. HTTP/CLI delivery adapter
8. Composition-root wiring
9. Focused module and delivery tests

Create only folders required by real concepts. Read [module-implementation.md](references/module-implementation.md) for the canonical layout, type taxonomy, call rules, persistence, API, configuration, and composition patterns.

Read the focused implementation reference whenever the slice uses that boundary:

- [temporal-workflow-design.md](references/temporal-workflow-design.md) for orchestration services, workflows, activities, signals, child workflows, retries, worker registration, and Temporal tests.
- [postgres-persistence.md](references/postgres-persistence.md) for schemas, transactions, contexts, migrations, statements, repository errors, and row-level security.
- [contract-first-api.md](references/contract-first-api.md) for OpenAPI schemas, generated types, Hummingbird controllers, request contexts, error mapping, route composition, and API tests.
- [testing-strategy.md](references/testing-strategy.md) for Swift Testing structure, meaningful coverage, mocks, assertions, integration boundaries, scoped serialization, and full-suite verification.
- [runtime-composition.md](references/runtime-composition.md) for API/worker/migration/CLI construction, configuration bridges, lifecycle, logging, health, and startup behavior.
- [provider-integrations.md](references/provider-integrations.md) for provider gateways, swappable capabilities, anti-corruption types, idempotency, webhooks, and authoritative state.
- [evolution-and-compatibility.md](references/evolution-and-compatibility.md) for database, OpenAPI, Temporal, provider, and remote-contract evolution and staged rollout.
- [security-and-authorization.md](references/security-and-authorization.md) for identity trust boundaries, authentication, module-owned authorization, RLS, service identities, secrets, audit records, and extracted-service security.
- [observability-and-operations.md](references/observability-and-operations.md) for structured telemetry, context propagation, metrics and objectives, health, alerting, reconciliation, redaction, and incident diagnostics.

### 5. Use the Appropriate Creation Recipe

Read [creation-workflows.md](references/creation-workflows.md) when creating:

- A complete backend package
- A new capability module
- An entity or use case
- Cross-module collaboration
- A workflow-backed process
- An API surface
- Tests and architecture verification

### 6. Extract Only at a Clean Port Boundary

Keep a module in-process unless independent scaling, availability, security, release cadence, runtime needs, or ownership justifies separate deployment.

Before extraction, require all consumers to use public ports, eliminate external access to the module's persistence, and give it an independent composition root. Replace the in-process call adapter with a remote client/server pair while preserving the capability's business semantics.

Read [separate-deployment.md](references/separate-deployment.md) before creating a standalone service, adding HTTP/gRPC between modules, moving data, or adding asynchronous messaging.

### 7. Verify

Run repository-specific formatting and builds, then run the complete test suite with an unfiltered `swift test` unless the repository explicitly defines another full-suite command. Verify:

- The SPM graph remains acyclic.
- Only permitted module dependencies were added.
- No module imports another module's adapter types.
- Workflows call activities, not services or repositories.
- Activities wrap one service and contain no business logic.
- Controllers perform transport mapping and one service call.
- Each repository and table has one owning module.
- Concrete construction and environment decoding stay in composition roots.
- Authentication establishes identity at delivery boundaries; owning services enforce resource authorization and fail closed when required identity is absent.
- Correlation crosses HTTP, Temporal, provider, and remote-service boundaries without exposing secrets or sensitive payloads.
- Tests exercise owned behavior rather than framework or mock behavior.

## Design Output

When asked for architecture rather than implementation, provide:

1. Capability ownership map
2. Acyclic SPM target graph
3. Public service/activity contracts per edge
4. Module layout and persistence ownership
5. API, worker, migration, and CLI composition roots
6. Initial deployment topology and reasons
7. Implementation or extraction sequence
8. Boundary risks and verification criteria

When asked to implement, make the smallest complete vertical change and preserve repository conventions throughout.
