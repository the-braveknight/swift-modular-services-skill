# Runtime Composition

## Contents

- Composition-root responsibility
- Runtime modes
- Construction order
- Configuration ownership
- Lifecycle and shutdown
- Logging and observability
- Health, readiness, and startup
- Migration runtime
- Extracted-service runtime
- Testing composition

## Composition-Root Responsibility

Keep concrete construction in executable targets. A composition root may import adapters from every module it assembles; capability modules must not construct or discover one another's implementations.

Let the composition root:

- Decode external configuration
- Bootstrap logging and observability
- Construct infrastructure clients
- Construct module services and provider adapters
- Inject public service protocols across modules
- Wrap services in activity containers
- Assemble API routes or workers
- Register lifecycle-managed services
- Start the selected runtime

Keep business rules, request mapping, provider translation, and database queries out of the composition root.

## Runtime Modes

Use one executable with subcommands or separate executables according to deployment needs. Model each runtime as a distinct composition path:

- API server
- Temporal/background worker
- Migration runner
- Administrative command
- Independently deployable capability service

Do not construct every dependency in every runtime. An API may need request authentication and controllers; a worker may need workflow activities and provider clients; a migration runner needs migration credentials but no HTTP server.

Keeping construction paths separate allows API and worker processes to scale, secure, and deploy independently even when they share one executable product.

## Construction Order

Construct dependencies from the outside inward:

```text
configuration
    -> logging/telemetry
    -> infrastructure clients
    -> provider adapters and repositories
    -> service contexts and domain services
    -> orchestration services and activity containers
    -> API server or worker
    -> lifecycle service group
```

Share a lifecycle-safe client only when the client library supports concurrent reuse and both consumers have the same configuration and shutdown owner. Otherwise construct separate clients deliberately.

Inject protocols into consumers after constructing the provider service. Do not use global registries or service locators to resolve dependencies at runtime.

## Configuration Ownership

Let each configurable type own a nested or associated pure `Configuration: Sendable` value in its module:

```swift
public struct Configuration: Sendable {
    public let timeout: Duration
    public let endpoint: String
}
```

Keep configuration-reading libraries out of capability targets. Add executable-side extensions that translate a scoped configuration reader into the module's configuration value:

```text
CLI/Configuration/
CLI/API/Configuration/
CLI/Worker/Configuration/
```

Use scoped keys to prevent collisions between modules/providers. Mark secrets as secrets in the configuration library, fail fast when required values are absent, and never log secret values.

When a runtime supports selectable providers, decode and validate only the selected provider's required configuration. Do not require credentials for an inactive provider. Construct shared SDK clients only for selected adapters, and permit different capabilities to select providers independently when their ports are separately owned.

Provide defaults only for safe operational values. Require credentials, signing keys, provider secrets, and production endpoints explicitly when absence would make the runtime insecure or misleading.

## Lifecycle and Shutdown

Register long-running clients and servers with the runtime's lifecycle mechanism:

- Database clients and pools
- HTTP servers
- Temporal workers
- Log processors/exporters
- Queue consumers
- Scheduled background services

Handle process termination signals and propagate graceful shutdown. Stop accepting new work, allow bounded in-flight completion where supported, flush telemetry, and close clients in dependency-safe order.

Do not launch unstructured background tasks from module initializers. Let the composition root own task lifetime and failure propagation.

Treat unexpected termination of a required lifecycle service as runtime failure. Do not silently keep serving when the database pool, worker, or critical telemetry pipeline fails to initialize.

## Logging and Observability

Bootstrap the logging system once before constructing components that capture loggers. Give each runtime a stable logger label and pass loggers into services/adapters that own meaningful events.

Use structured metadata with stable camel-case keys such as:

- `userId`
- `workflowId`
- `sessionId`
- `providerEventId`
- `module`

Log business milestones and unexpected failures, not secrets or complete provider payloads. Keep provider-specific metadata interpretation in the provider gateway.

When tracing is available, propagate request/workflow correlation across HTTP, Temporal activities, and remote services. Emit metrics for latency, error rate, retry volume, queue lag, workflow failure, database saturation, and reconciliation drift where operationally relevant.

Read [observability-and-operations.md](observability-and-operations.md) when defining telemetry fields, propagation, objectives, alerts, dashboards, redaction, or incident diagnostics.

## Health, Readiness, and Startup

Distinguish:

- **Liveness:** the process and event loop are functioning.
- **Readiness:** the runtime can accept its assigned work.
- **Dependency diagnostics:** detailed status for operators, not necessarily a public endpoint.

Fail startup when required configuration is invalid or a mandatory component cannot be constructed. Decide deliberately whether temporary dependency unavailability prevents readiness or is handled by client retries.

Do not expose secrets, connection strings, provider responses, or internal topology through health endpoints.

## Migration Runtime

Use a dedicated migration command or executable with elevated migration credentials. Construct only configuration, logging, the database client, and the ordered migration registry.

Register module-owned migrations in dependency order. Start the database client under lifecycle management, apply migrations, report completion, and shut down cleanly.

Do not run migrations implicitly in every API or worker replica unless the deployment strategy explicitly coordinates one migration authority.

## Extracted-Service Runtime

Give an extracted service its own composition root, configuration scope, database credentials, delivery server, health/readiness behavior, logging identity, and graceful shutdown.

Keep its domain/application target reusable. Let the standalone executable select remote server adapters, while monolith composition may select an in-process adapter during migration.

Read [separate-deployment.md](separate-deployment.md) for boundary and data extraction and [evolution-and-compatibility.md](evolution-and-compatibility.md) for rollout ordering.

## Testing Composition

Add lightweight smoke tests for assembly that is difficult to verify statically:

- Required configuration maps to module configuration values.
- The API can register all routes with supplied protocols.
- The worker registers every referenced workflow and activity.
- The migration registry contains new migrations in the intended order.
- Shutdown completes without leaking owned test services.

Do not duplicate service business tests in composition tests. Prefer factories or closure-scoped application construction that ensures lifecycle cleanup.
