# Separate Deployment

## Contents

- Decide whether to extract
- Preserve the capability boundary
- Add a remote adapter
- Separate runtime and data ownership
- Handle remote failures
- Use messaging conditionally
- Migrate safely

## Decide Whether to Extract

Keep a module in-process unless separate deployment solves a concrete problem:

- Independent scaling profile
- Availability or failure isolation
- Security or compliance isolation
- Independent release cadence
- Different runtime or infrastructure requirements
- Clear operational/team ownership

Separate deployment adds latency, partial failure, version skew, authentication, observability, deployment coordination, and data migration. Do not extract only because a module appears architecturally independent.

## Preserve the Capability Boundary

Before extraction, require:

- Consumers use only the module's public service/activity contracts.
- No consumer imports its adapters.
- No external code reads or writes its tables.
- The module owns its schema, migrations, and data credentials.
- Cross-module transactions have an explicit alternative.
- The module can be assembled and tested without consumer adapters.

Preserve provider-owned business semantics. Do not expose repository-shaped RPCs or database rows.

## Add a Remote Adapter

Transform the boundary as follows:

```text
Before
Consumer -> provider service protocol -> provider service

After
Consumer -> client port -> HTTP/gRPC client
                         -> versioned transport contract
                         -> server controller/handler
                         -> provider service protocol -> provider service
```

Keep domain commands, results, and capability errors conceptually stable, but use separate transport DTOs when serialization and compatibility require them. Add a dedicated contract target/package only when independent generation or release requires it.

Do not pretend a remote call is local. Model deadline exceeded, unavailable, authentication failure, cancellation, and incompatible responses at the client boundary.

If an existing workflow called a module-owned activity in the same worker, choose deliberately:

- Keep the activity with the workflow worker and let it call the new remote client.
- Move the activity to the extracted service's worker if workflow/task-queue ownership also moves.
- Replace the collaboration with a child workflow only when durable ownership and task routing justify it.

## Separate Runtime and Data Ownership

Add an executable composition root for the extracted service. Let it construct:

- Configuration
- Database pool and owned repositories
- Provider gateways
- Domain/application services
- HTTP/gRPC server handlers
- Workflow activities/workers it owns
- Health, readiness, logging, metrics, and graceful shutdown

Give the service exclusive write credentials. Sharing the same physical PostgreSQL cluster temporarily is acceptable; sharing mutable table ownership is not.

Remove cross-module foreign keys, joins, and transactions before extraction. Replace cross-capability atomicity with explicit durable orchestration only where the business process requires it.

## Handle Remote Failures

For each synchronous operation, define:

- Deadline and cancellation
- Authentication and authorization
- Capability-error to transport-status mapping
- Retryable versus terminal failures
- Idempotency for commands that may be replayed
- Contract compatibility and version-skew policy
- Trace/correlation propagation and metrics

Read [security-and-authorization.md](security-and-authorization.md) for service identity and delegated-user rules. Read [observability-and-operations.md](observability-and-operations.md) for cross-process telemetry and operational acceptance criteria.

Retry only transient operations that are safe or idempotent. Avoid deep synchronous call chains.

Test contract mapping, deadlines, cancellation, authentication, idempotent replay, version compatibility, dependency outages, and composition of both client and server executables.

## Use Messaging Conditionally

Do not add events as part of extraction by default. Use a durable message when:

- The producer should complete while the consumer is unavailable.
- Multiple independently changing consumers need the same fact.
- Eventual consistency is acceptable.
- A local projection is justified by remote read volume or availability requirements.

When messaging is selected, define event ownership, schema version, stable event ID, delivery guarantee, ordering scope, deduplication, retries, dead letters, replay, and reconciliation. Use an outbox only when a state write and publication must be coordinated.

## Migrate Safely

Read [evolution-and-compatibility.md](evolution-and-compatibility.md) before changing released transport schemas, Temporal histories, database ownership, or deployment order during extraction.

Use a reversible sequence:

1. Clean the in-process boundary and remove concrete cross-imports.
2. Route every consumer through the provider's public port.
3. Establish exclusive data ownership and remove external table access.
4. Add an in-process client adapter if a consumer-facing seam is useful.
5. Define the remote contract and explicit failure semantics.
6. Add remote client and server adapters.
7. Deploy the new service with exactly one authoritative writer.
8. Shadow or canary safe traffic and compare results, latency, and errors.
9. Switch traffic under explicit rollback criteria.
10. Remove the in-process implementation path only after the observation window succeeds.

If data moves, backfill, capture changes, verify counts and invariants, then switch reads/writes in the order required by the replication and rollback plan. Never leave two active writers without an explicit authority and conflict policy.

Define measurable cutover gates: acceptable mismatch rate, latency/error objectives, reconciliation drift, rollback drill success, and an observation window covering normal and peak behavior.
