# Testing Strategy

## Contents

- Test ownership
- Meaningful coverage
- Suite organization
- Mocks and support values
- Assertions and parameterization
- Domain and service tests
- Persistence, API, and Temporal tests
- Concurrency and suite isolation
- Verification workflow

## Test Ownership

Keep tests with the production target that owns the behavior:

```text
Tests/<Module>Tests/
├── <ProductionObject>Tests.swift
├── Mocks/
└── Support/
```

Keep orchestration-service tests in their capability's test target. Keep API transport tests in the API test target. Do not create horizontal test targets such as `ServiceTests` or `OrchestrationTests` that weaken capability ownership.

Declare every directly imported module and test product explicitly in the test target. Do not rely on transitive dependencies.

## Meaningful Coverage

Test behavior owned by the production object:

- Business rules and branches
- State transitions and invariants
- Input normalization and domain/API mapping
- Ordering, compensation, and required side effects
- Intentional error translation
- Authentication and authorization boundaries
- Idempotency and duplicate handling
- Controller-owned request, response, and status mapping

Apply this deletion test: if removing the production behavior would still leave the test passing, the test is probably not useful.

Do not add tests that only prove:

- Swift returns a dependency's value unchanged.
- `try await` propagates an unexpected error.
- A mock stores and returns its own data.
- A thin activity forwards one call.
- Framework decoding or generic problem formatting works in every controller.
- A Temporal call completed without an observable business assertion.

## Suite Organization

Keep suite files focused on setup, the production call, and assertions. Put reusable test doubles in `Mocks/` and deterministic domain examples in `Support/`.

Name tests and suites by observable behavior. Parameterize cases when they share the same execution path and differ only in inputs and expected results.

Keep test state local to each test whenever possible so suites can run in parallel. Use serialization only around a genuinely shared resource such as one managed Temporal test server.

## Mocks and Support Values

Make mocks minimal implementations of production protocols:

- Name them `MockX`, not `TestX`.
- Capture arguments in directly readable properties.
- Return configurable values or errors needed by the suite.
- Use `fatalError("Not used by …")` for protocol methods outside the tested path.
- Avoid test-only call-record/input wrapper types unless ordering across many calls is itself the behavior.
- Do not add getter methods solely to inspect mock state.
- Do not test the mock.

Define reusable examples as static properties on production domain types when the repository follows that convention:

```swift
extension User {
    static let reader = User(...)
}
```

Keep timestamps, UUIDs, provider IDs, and tokens deterministic. Avoid fixture namespaces that obscure the domain type being constructed.

## Assertions and Parameterization

Use `#expect` for ordinary assertions and `#require` when later assertions depend on an optional or condition:

```swift
let user = try #require(response.user)
#expect(user.email == "reader@example.com")
```

Use typed thrown-error assertions and compare the real error:

```swift
let error = await #expect(
    throws: UserServiceError.self,
    performing: { try await service.get(id: missingId) }
)
#expect(error == .userNotFound)
```

Do not introduce generic `TestError` values when the production object emits a meaningful typed error. Record an explicit issue when an exhaustive error assertion cannot be expressed directly.

## Domain and Service Tests

Test pure domain values without mocks. Test services with focused repository/context/provider doubles when the service owns:

- Validation or normalization
- A transaction boundary
- Branching based on repository/provider results
- Cross-module port collaboration
- Required call order or side effects
- Repository-to-service error translation

Do not unit-test an implementation detail that is already enforced more reliably by a database constraint or generated transport schema. Test the service meaning separately from adapter enforcement.

## Persistence, API, and Temporal Tests

Read the specialized reference for boundary-specific coverage:

- [postgres-persistence.md](postgres-persistence.md) for constraints, transactions, statement mapping, concurrency, migrations, and RLS.
- [contract-first-api.md](contract-first-api.md) for router tests, generated payloads, identity, status, and problem mapping.
- [temporal-workflow-design.md](temporal-workflow-design.md) for signals, children, retries, idempotency, authoritative reads, replay, and managed test-server usage.

Use real adapters only when the test needs to prove their translation or integration behavior. Use real activity containers around service mocks when testing workflow code so the workflow-facing contract remains authentic.

Keep representative end-to-end tests for composition and critical paths, but do not duplicate every service branch through HTTP, PostgreSQL, and Temporal.

## Concurrency and Suite Isolation

Prefer parallel tests with independent state. When a suite owns shared infrastructure:

- Scope serialization to that suite rather than the entire test target.
- Give database tests isolated schemas, transactions, or identifiers.
- Avoid global mutable mocks and singleton state.
- Shut down lifecycle-managed clients and servers deterministically.

Express required isolation through Swift Testing suite traits and scoped resource ownership. For example, keep one managed Temporal test server under its integration suite and mark only that suite `.serialized`; leave unrelated suites parallel.

Design the package so the complete suite runs correctly in one invocation:

```bash
swift test
```

Do not use filtered test-target invocations as a substitute for fixing shared state, lifecycle cleanup, port conflicts, or suite-level serialization. Add process-level separation only when a repository documents an unavoidable tool or infrastructure limitation; keep the normal developer and CI verification command full-suite.

## Verification Workflow

During implementation:

1. Add focused tests in the target that owns the changed behavior.
2. Add or update adjacent consumer/provider coverage when a public port changed.
3. Add API or worker integration coverage when delivery or workflow wiring changed.
4. Run package/build checks to catch dependency and generated-code issues.
5. Run the complete test suite so cross-target interactions and shared-resource isolation are exercised.
6. Run repository formatting and linting when configured.

Common checks are:

```bash
swift package describe
swift build
swift test
```

Review the target graph and imports in addition to tests; runtime tests cannot enforce every architectural boundary inside one SPM target.
