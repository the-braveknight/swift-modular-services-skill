# PostgreSQL Persistence

## Contents

- Ownership and layout
- Database abstraction and transactions
- Service contexts
- Schemas and migrations
- Statements and repositories
- Error mapping
- Cross-module integrity
- Row-level security
- Testing

## Ownership and Layout

Keep PostgreSQL details in the owning module's adapters:

```text
Ports/
├── Repositories/
└── Contexts/
Adapters/
├── Contexts/
├── Repositories/
├── Statements/<Entity>/
├── Migrations/<Entity>/
└── Extensions/
```

Let the module own its schema or table namespace, migrations, statements, repositories, database mappings, and write transactions. Sharing a PostgreSQL cluster does not imply shared table ownership.

## Database Abstraction and Transactions

Use a small transaction abstraction whose operation receives a module-specific context:

```swift
public protocol Database<Context>: Sendable {
    associatedtype Context: Sendable

    func withTransaction<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T
}
```

Let the concrete PostgreSQL database open, commit, and roll back transactions and construct the context from the transaction connection. Keep business services generic over the database abstraction when that improves testability and transaction consistency.

Put all writes that protect one module invariant in the same service-owned transaction. Do not hold a database transaction open across a Temporal wait, remote service call, or external user interaction.

## Service Contexts

Define one context protocol per service or cohesive transaction boundary. List only repositories owned by that module:

```swift
public protocol EntitlementServiceContext: Sendable {
    var entitlementRepository: any EntitlementRepository { get }
    var deviceRepository: any DeviceRepository { get }
}
```

Let the concrete PostgreSQL context construct repositories over the same transaction connection. Never add another module's repository to the context. Inject the other module's service protocol separately at the service/composition boundary.

## Schemas and Migrations

Prefer one PostgreSQL schema per capability when the project uses schema ownership. Create:

- A module schema migration
- One table migration per entity
- Separate policy or alteration migrations when required

Register migrations in an explicit migration composition root and preserve ordering. Create roles and schemas before dependent tables; create parent tables before intra-module children.

Keep migrations append-only after release unless the repository explicitly supports reversible editing. Do not hide production schema changes in repository initialization code.

Grant runtime roles only the permissions required by their runtime. Keep migration credentials separate from restricted API/service credentials when the deployment model supports it.

## Statements and Repositories

Use one prepared statement/query type per operation when that is the repository convention:

```text
Create<Entity>Statement
Get<Entity>Statement
Get<Entity>By<X>Statement
List<Entity>Statement
Update<Entity>Statement
Delete<Entity>Statement
Consume<Entity>Statement
```

Return typed domain entities or a named row projection, not anonymous tuples crossing layers. Keep SQL column and enum mappings in adapter extensions rather than making domain values conform directly to database protocols.

Let a repository compose only its own statements. Let services access repositories through their transaction context; do not call repositories from controllers, workflows, activities belonging to another module, or other modules.

Use database constraints for uniqueness, ownership, ranges, and intra-module referential integrity. Avoid pre-validation round trips when the write constraint can decide atomically.

## Error Mapping

Catch the database driver's typed error and map only known SQLSTATE and constraint names:

- Unique violation: map the specific constraint to a typed duplicate/conflict error.
- Foreign-key violation: map only an owned intra-module relationship.
- Check violation: map a known domain constraint.
- No row returned from required update: map to not found.
- Unknown database failure: rethrow unchanged.

Translate repository errors into service errors only where the service intentionally owns a different public meaning. Do not collapse every database error into a generic domain failure.

Make delete idempotent only when that is the public behavior. Otherwise distinguish not found deliberately.

## Cross-Module Integrity

Do not create cross-module foreign keys or join another module's tables from production service code. Store stable identifiers as plain values. Validate through the owning module's public service when immediate validation is required.

For a cross-module durable process, coordinate module-owned transactions through workflow activities. Do not attempt one database transaction across independently owned capability contexts.

## Row-Level Security

Use PostgreSQL RLS when database-enforced tenant/user isolation is required. Apply a fail-closed design:

- Set trusted user and role values inside the transaction using `SET LOCAL` or equivalent.
- Scope request identity with task-local or explicit request context only for the transaction lifetime.
- Return no protected rows when identity context is absent.
- Test user, admin, internal-service, and missing-context behavior.
- Keep worker/service principals explicit; do not pretend internal automation is an end-user or admin.

Let authentication middleware establish verified identity. Let the concrete database adapter translate that trusted identity into transaction-local PostgreSQL settings. Domain services should not know SQL session-variable details.

When extracting a service, recreate trusted identity at the remote server boundary and give the service exclusive database credentials. Never accept arbitrary caller-provided role metadata without authentication.

Read [security-and-authorization.md](security-and-authorization.md) for identity trust boundaries, module-owned authorization, internal-service principals, secret handling, and audit requirements.

## Testing

Unit-test service behavior with an in-memory/mock `Database<Context>` only when it proves business rules or transaction orchestration. Use PostgreSQL integration tests for behavior that depends on:

- Constraints and SQLSTATE mapping
- Prepared-statement row mapping
- Transaction rollback/atomicity
- Concurrent writes and capacity enforcement
- RLS isolation and fail-closed context
- Migration ordering or compatibility

Do not test the database driver itself. Test the adapter behavior and invariants the application owns.
