# Evolution and Compatibility

## Contents

- Classify the contract
- Evolve module ports
- Evolve PostgreSQL schemas
- Evolve OpenAPI contracts
- Evolve Temporal workflows
- Evolve provider integrations
- Evolve remote service contracts
- Stage rollout and removal
- Review checklist

## Classify the Contract

Before changing a type or behavior, identify every persisted or independently released surface it affects:

- Swift module public API compiled in the same release
- Database schema and stored rows
- OpenAPI schema consumed by deployed clients
- Temporal workflow history, activity/signal names, and serialized payloads
- Provider metadata and external identifiers already stored remotely
- HTTP/gRPC/message contracts consumed by separate services
- Configuration keys and deployment manifests

An internal refactor and a persisted/distributed contract change require different migration plans. Do not assume recompiling the current package updates existing database rows, workflow histories, clients, or provider objects.

## Evolve Module Ports

Keep public ports behavior-oriented and owned by the provider module. When adding behavior:

- Prefer additive methods and domain values.
- Add a new command when a method would otherwise accumulate unrelated parameters.
- Update every explicit consumer target dependency and test mock.
- Preserve typed error meanings or introduce a deliberate new case.

When removing or changing behavior, migrate all in-process consumers in one coordinated package change unless the contract is independently packaged. Do not keep duplicate old/new ports indefinitely without a removal owner and date.

Avoid splitting a contract target solely to make a breaking change appear versioned. Introduce separate packaging only when independent compilation/release actually exists.

## Evolve PostgreSQL Schemas

Treat released migrations as append-only. Use expand/migrate/contract for changes that cannot be applied atomically with one deployment:

1. **Expand:** add nullable columns, new tables, indexes, or compatibility structures.
2. **Migrate:** backfill and verify existing data; deploy code able to use both forms when necessary.
3. **Contract:** remove old columns/constraints only after every reader/writer has moved and rollback no longer needs them.

Preserve migration ordering and module ownership. Create indexes concurrently or with an operational plan when table size makes locking material.

For new non-null fields, add a safe default or nullable phase, backfill, verify, then enforce the constraint. Do not invent placeholder domain data merely to satisfy a migration.

Update RLS policies, grants, and restricted roles alongside table changes. Test fail-closed behavior after every ownership/security migration.

Read [postgres-persistence.md](postgres-persistence.md) for adapter and migration structure.

## Evolve OpenAPI Contracts

Treat deployed clients as independent consumers even when server and generated Swift types build together.

Prefer additive changes:

- Add optional request fields with defined defaults.
- Add response fields only when clients tolerate unknown fields.
- Add endpoints rather than changing unrelated endpoint semantics.
- Keep existing enum values stable.

Treat these as potentially breaking:

- Removing or renaming fields/endpoints
- Making an optional field required
- Changing identifier representation
- Changing status or error semantics
- Adding enum cases to clients that decode exhaustively

Update the OpenAPI source first, regenerate types, update mappings/controllers, and run compatibility checks when available. Keep old routes/schemas through a deprecation window when clients release independently.

Distinguish optionality meanings. An optional PATCH field may mean "leave unchanged" even when the resulting domain value and database column are required. Do not make partial-update commands non-optional merely because stored state becomes non-null.

Read [contract-first-api.md](contract-first-api.md) for delivery structure.

## Evolve Temporal Workflows

Assume open workflow histories may replay old decisions against new code. Preserve:

- Workflow type/name
- Activity names
- Signal names
- Task-queue routing
- Serialized input and signal compatibility
- Deterministic command ordering

Do not change a workflow's control flow in a way that emits different commands for an existing history without the SDK's supported patch/version mechanism. Add new optional payload fields with backward-compatible decoding when possible.

When renaming or replacing an activity, keep the old activity registered until no open history can schedule it. When moving workers/task queues, plan how existing executions continue to find their workflow and activity implementations.

Once new histories reference a new workflow/activity/signal name or patch branch, do not roll back to a worker binary that lacks it. Keep one dual-compatible worker or use the platform's supported worker-version routing, then prefer a forward fix.

Test replay or representative open histories for compatibility-sensitive changes. Add workflow behavior tests for new paths without deleting coverage of old persisted paths prematurely.

Read [temporal-workflow-design.md](temporal-workflow-design.md) for workflow rules and race handling.

## Evolve Provider Integrations

Treat provider metadata keys, idempotency keys, customer/source mappings, and stored external IDs as released contracts. Objects already created at the provider will not gain new metadata automatically.

When changing a metadata convention:

1. Decode both old and new forms.
2. Write only the new form after deployment.
3. Backfill provider state only when necessary and operationally safe.
4. Remove old decoding after the historical population expires or is migrated.

Preserve idempotency-key derivation for the same logical operation. A new key shape can cause duplicate provider mutations on retry.

Handle provider API/version upgrades inside the gateway. Keep domain-facing behavior stable where possible and add contract tests for changed request/response mapping.

Read [provider-integrations.md](provider-integrations.md) for gateway ownership.

## Evolve Remote Service Contracts

Version independently deployed HTTP/gRPC/message contracts according to actual compatibility needs. Define a supported client/server skew window and deploy additive server support before clients use it.

For gRPC/protobuf-style schemas:

- Never reuse removed field numbers.
- Add fields compatibly.
- Preserve method and message semantics.
- Reserve removed names/numbers where supported.

For events:

- Preserve event meaning; create a new event when semantics change materially.
- Include stable event identity and schema/version information.
- Keep consumers idempotent and tolerant of expected version skew.

Do not coordinate a breaking change through hope or deployment timing alone. Use parallel endpoints/methods, compatibility adapters, or a staged migration.

## Stage Rollout and Removal

Use a reversible rollout:

1. Add backward-compatible capability.
2. Deploy providers/servers/readers that understand both forms.
3. Enable new writers or callers gradually.
4. Observe correctness, errors, latency, and compatibility metrics.
5. Backfill or reconcile persisted state.
6. Drill rollback while both forms remain supported.
7. Remove the old form only after the observation and rollback window.

Use feature flags or routing switches when they materially reduce risk. Assign one authoritative writer throughout data migrations. Avoid indefinite dual-write paths without conflict resolution and an explicit removal plan.

Update architecture documentation, API contracts, migration registries, worker registration, configuration examples, and tests in the same change where appropriate.

## Review Checklist

Before handoff, answer:

- Which persisted or independently deployed contracts changed?
- Can old data, clients, workflow histories, and provider objects still be read?
- What must deploy first?
- Is every write path idempotent during retries and rollout?
- Is there exactly one authoritative writer?
- What telemetry proves the migration is safe?
- What are the rollback trigger and procedure?
- When and how will compatibility code be removed?
- Which tests exercise old and new forms?
