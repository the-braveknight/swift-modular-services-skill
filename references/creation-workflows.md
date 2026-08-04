# Creation Workflows

## Contents

- Create a whole backend
- Add a module
- Add an entity or use case
- Add cross-module collaboration
- Add durable orchestration
- Add an API surface
- Test and verify

## Create a Whole Backend

1. Initialize or inspect the Swift package before adding source structure.
2. List business capabilities, invariants, owned data, and runtime entry points.
3. Create an acyclic target graph with one target per capability.
4. Add a minimal shared `Core` target only for stable platform primitives or abstractions.
5. Add an API library target that depends on required capability modules.
6. Add one executable composition target with API, worker, migration, and administrative subcommands or construction paths.
7. Give each capability its own test target.
8. Implement one complete vertical slice before scaffolding every capability.

Representative manifest shape:

```swift
targets: [
    .target(name: "Core"),
    .target(name: "Users", dependencies: ["Core"]),
    .target(name: "Authentication", dependencies: ["Core", "Users"]),
    .target(name: "Entitlements", dependencies: ["Core"]),
    .target(name: "Payments", dependencies: ["Core", "Entitlements"]),
    .target(name: "API", dependencies: ["Users", "Authentication", "Entitlements", "Payments"]),
    .executableTarget(name: "CLI", dependencies: ["API", "Users", "Authentication", "Entitlements", "Payments"]),
    .testTarget(name: "UsersTests", dependencies: ["Users"]),
    .testTarget(name: "PaymentsTests", dependencies: ["Payments"])
]
```

Adapt names and dependencies to the product. Do not copy example capabilities blindly.

## Add a Module

1. State the module's responsibility, invariants, owned data, public ports, and dependencies.
2. Confirm the proposed edge does not create a target cycle.
3. Add one SPM target and one test target.
4. Create `Domain/`, `Ports/`, and `Adapters/` folders required by the first vertical slice.
5. Add module-owned schema creation and migrations when it persists data.
6. Add service protocol, service implementation, repository/context ports, and adapters.
7. Add API or worker delivery only for actual entry points.
8. Wire concrete dependencies in the executable composition root.
9. Document the module and dependency graph in the repository's architecture guide.
10. Build and run the new module's tests.

Do not create a shared infrastructure module to hold the new module's providers. Keep its capability adapters with the module.

## Add an Entity or Use Case

Implement from ownership outward:

1. Add the domain entity/value and supporting top-level enums.
2. Add create/update commands when several input fields travel together.
3. Add typed repository and service errors.
4. Add or extend the repository protocol.
5. Add the repository to the owning service context.
6. Follow [postgres-persistence.md](postgres-persistence.md) for migrations, statements, concrete repositories, transaction contexts, and error mapping.
7. Add service behavior and deliberate error translation.
8. Add an activity method only if a workflow must call the behavior.
9. Add orchestration/workflow behavior only if durability is required.
10. Add API schemas, mappings, problem mapping, and controller route when exposed over HTTP.
11. Add focused tests for the owned behavior.

Keep transaction-sensitive behavior inside the owning service/context. Do not pre-validate with another module's database access.

## Add Cross-Module Collaboration

For an immediate in-process call:

1. Make the provider module's service protocol and required domain inputs/results public.
2. Add a one-way SPM dependency from consumer to provider.
3. Inject the provider service protocol into the consumer service or orchestration service.
4. Construct the concrete provider service in the composition root.
5. Pass only the protocol into the consumer.
6. Test the consumer with a focused mock of the provider port.

For collaboration inside a durable workflow:

1. Let the provider module define an `XActivities` container in `Ports/Activities/`.
2. Wrap exactly one provider service and expose only required operations.
3. Make activity inputs `Codable` and `Sendable` as required by the workflow engine.
4. Add the provider module as a target dependency of the workflow-owning module.
5. Execute the provider's activity from the workflow.
6. Register the activity implementation in the worker composition root.

Example:

```text
Payments workflow
    -> EntitlementActivities.upsertEntitlement(command)
    -> EntitlementServiceProtocol
    -> Entitlements-owned context/repositories
```

Do not introduce an event merely to avoid a valid one-way target dependency.

## Add Durable Orchestration

Use a durable workflow when a process requires waiting, retries, recovery, signals, child workflows, or compensation. Follow [temporal-workflow-design.md](temporal-workflow-design.md) for ownership, activity shapes, determinism, inputs, retries, signals, child workflows, worker registration, and testing.

## Add an API Surface

Follow [contract-first-api.md](contract-first-api.md) for contract changes, generated transport types, domain mapping, controllers, request contexts, error responses, route composition, and tests.

## Test and Verify

Follow [testing-strategy.md](testing-strategy.md) for test ownership, meaningful coverage, mocks, support values, assertions, specialized integration boundaries, scoped serialization, and full-suite verification. Also review imports and `Package.swift` for forbidden edges that Swift access control cannot express inside a single target.
