# Module Implementation

## Contents

- Canonical layout
- Service taxonomy
- Component call rules
- Port design
- Focused implementation references
- Composition and configuration
- Swift constraints

## Canonical Layout

Use this structure for a substantial capability:

```text
Sources/<Module>/
├── Domain/
│   ├── Entities/
│   ├── Commands/
│   ├── Events/             # only when events exist
│   └── Errors/
├── Ports/
│   ├── Services/
│   ├── Repositories/
│   ├── Contexts/
│   ├── Security/           # only for replaceable security capabilities
│   └── Activities/         # only for workflow-facing operations
└── Adapters/
    ├── Services/
    ├── Orchestration/      # API-triggered durable coordination
    ├── Repositories/
    ├── Contexts/
    ├── Statements/
    │   └── <Entity>/
    ├── Migrations/
    │   └── <Entity>/
    ├── Extensions/
    └── Workflows/          # only for durable workflows
```

Create directories only when they contain a real concept. A small module may use fewer subdirectories while preserving the same dependency direction.

Keep shared HTTP delivery in a separate API target:

```text
Sources/API/
├── Controllers/
├── Contexts/
├── Middlewares/
├── Security/
├── Problems/
└── Extensions/            # API/domain mapping
```

Keep executable assembly separate:

```text
Sources/CLI/
├── API/
├── Worker/
├── Migrate/
├── Configuration/
└── <AdministrativeCommand>/
```

## Service Taxonomy

Choose the component kind before naming or placing it:

| Kind | Responsibility | Public shape | Concrete | Location |
|---|---|---|---|---|
| Domain service | Enforce use cases and business rules; use owned repositories | `XServiceProtocol` | `XService` | `Ports/Services`, `Adapters/Services` |
| Orchestration service | Coordinate services or start/signal durable workflows; no repository access | `XServiceProtocol` | `XService` | `Ports/Services`, `Adapters/Orchestration` |
| Provider gateway | Translate one external provider into a domain-shaped API | `XServiceProtocol` | `XService` | `Ports/Services`, `Adapters/Services` |
| Swappable capability | Perform one narrow replaceable capability | capability noun without `Protocol` | `<Provider><Capability>` | `Ports/Services`, `Adapters/Services` |
| Activity container | Expose selected service operations to workflows; no logic | `XActivities` | same type wrapping one service | `Ports/Activities` |
| Workflow | Coordinate durable steps through activities | `XWorkflow` | workflow implementation | `Adapters/Workflows` |
| Repository | Persist one module's domain state | `XRepository` | `PostgresXRepository` or provider equivalent | `Ports/Repositories`, `Adapters/Repositories` |
| Service context | Group repositories and transaction behavior for one service/module | `XServiceContext` | `PostgresXServiceContext` | `Ports/Contexts`, `Adapters/Contexts` |
| Controller | Decode, call one service, encode | route-facing type | controller implementation | `API/Controllers` |

Use `XService`/`XServiceProtocol` for domain, orchestration, and provider services. Use a capability-named port without `Protocol` when implementations are provider-swappable, such as `UserEmailSender` implemented by `ResendUserEmailSender`.

Read [provider-integrations.md](provider-integrations.md) before adding or changing an external provider gateway, capability adapter, provider metadata convention, webhook, or idempotent provider operation.

## Component Call Rules

Use this call graph:

```text
HTTP request                              Durable worker
     │                                         │
Controller                                Workflow
     │ one service call                        │ execute activity
     ├── Domain service                        ▼
     ├── Gateway service                  XActivities
     └── Orchestration service                 │ one wrapped service
             │ start/signal workflow           ▼
             └──────────────────────────> Domain/gateway service
                                                   │
                                                   ▼
                                         repository or provider
```

Enforce these rules:

1. Let a controller call one domain, gateway, or orchestration service. Move coordination of multiple services into an orchestration service.
2. Let a domain service call its owned repositories through its context. It may call another module's injected service protocol when the dependency graph permits it.
3. Let an orchestration service call domain/gateway services and start or signal workflows. Do not let it touch repositories directly.
4. Let a workflow call activities and child workflows only. Keep all I/O inside activities.
5. Let an activity wrap exactly one domain/gateway/capability service and expose only methods workflows need.
6. Let repositories call their own statements and remain private to the owning service context.
7. Keep domain and provider services unaware of whether the caller is HTTP or a workflow activity.

When using Temporal, allow Temporal imports only in orchestration services, activity containers, and workflows. Keep ordinary domain/gateway services Temporal-free.

## Port Design

Let the providing module own its service protocol. Use domain entities, commands, values, and typed errors in that port:

```swift
public protocol EntitlementServiceProtocol: Sendable {
    func upsertEntitlement(command: UpsertEntitlementCommand) async throws -> Entitlement
    func deleteEntitlement(sourceId: String) async throws
}
```

Let consumers import and call this public contract; do not define a mirror repository-shaped port in the consumer.

Use repository `find` for optional lookup and service `get` when absence is a typed failure. Prefer behavior-oriented operations over generic CRUD when invariants require it.

Use commands for multi-field create/update inputs. Keep supporting domain enums as top-level types when that matches repository convention.

Keep HTTP schemas, workflow payload wrappers, database rows, provider SDK types, and domain values distinct when their reasons to change differ. Translate at their adapters.

## Focused Implementation References

- Keep persistence inside the owning module's adapters and transaction context. Read [postgres-persistence.md](postgres-persistence.md) before adding or changing schemas, migrations, statements, repositories, service contexts, transactions, database error mapping, or row-level security.
- Keep controllers as single-call transport adapters and keep API transport types separate from domain values. Read [contract-first-api.md](contract-first-api.md) before adding or changing OpenAPI schemas, generated types, controllers, request contexts, middleware, problem mapping, or API tests.
- Keep durable sequencing in workflows and I/O in activities. Read [temporal-workflow-design.md](temporal-workflow-design.md) before adding or changing orchestration services, workflows, activities, signals, child workflows, retries, or worker registration.

## Composition and Configuration

Construct concrete dependencies only in executable composition roots. Read [runtime-composition.md](runtime-composition.md) before adding or changing API, worker, migration, administrative, or standalone-service runtimes; configuration bridges; service lifecycle; logging; health; or startup behavior.

## Swift Constraints

- Make cross-task public contracts `Sendable`.
- Use actors for genuinely shared mutable state, not as default service wrappers.
- Keep non-sendable transaction state scoped to the operation that owns it.
- Define cancellation, lifecycle, and graceful shutdown in executable targets.
- Preserve repository naming, file-header, import, and access-control conventions.
- Use package targets to enforce module edges and code review/tests to enforce the internal `Domain/Ports/Adapters` direction, because directories alone are not compiler boundaries.
