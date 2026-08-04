# Temporal Workflow Design

## Contents

- Decide when to use Temporal
- Assign ownership
- Separate orchestration layers
- Design activities
- Design workflow inputs and state
- Preserve determinism
- Choose execution patterns
- Avoid start races and lost wakeups
- Configure timeouts, retries, and errors
- Define IDs, signals, and authoritative reads
- Register the worker
- Test workflows

## Decide When to Use Temporal

Use a Temporal workflow when the process needs one or more of:

- Durable waiting across minutes, hours, or days
- Automatic retry of transient external failures
- Recovery after process restarts
- Signals from webhooks, users, or other workflows
- Ordered coordination across modules or providers
- Child workflows with explicit lifecycle behavior
- Compensation or reconciliation across non-atomic steps

Keep an ordinary synchronous service call when the operation is short, transactional, and does not require durable state. Do not put all business logic into workflows merely because Temporal is available.

## Assign Ownership

Place a workflow in the module whose domain process it governs:

- A checkout/fulfillment workflow belongs to Payments.
- An email-verification lifecycle belongs to Users.
- A password-reset notification lifecycle belongs to Authentication.

Let the workflow call another module only through the other module's activity container. For example:

```text
Payments/FulfillmentWorkflow
    -> EntitlementActivities.upsertEntitlement
    -> EntitlementServiceProtocol
    -> Entitlements-owned persistence
```

The workflow owner sequences the process. The activity owner retains the meaning, validation, and persistence of its capability operation.

## Separate Orchestration Layers

Keep two forms of orchestration distinct:

| Form | Location | Called by | May call | Must not |
|---|---|---|---|---|
| Orchestration service | `Adapters/Orchestration/` | Controller or another orchestration service | Domain/gateway services and `TemporalClient` start/signal APIs | Run inside a workflow or access repositories directly |
| Workflow | `Adapters/Workflows/` | Temporal worker | Activities, timers, conditions, signals, child workflows | Call services, repositories, HTTP clients, databases, or environment APIs directly |

Let an API controller call one orchestration service. Let that service start, execute, or signal the workflow. Never call an orchestration service from workflow code, and never start a workflow from an activity.

## Design Activities

Place activity containers in the providing module's `Ports/Activities/` directory:

```swift
@ActivityContainer
public struct EntitlementActivities {
    private let service: any EntitlementServiceProtocol

    public init(service: any EntitlementServiceProtocol) {
        self.service = service
    }

    @Activity
    public func upsertEntitlement(
        command: UpsertEntitlementCommand
    ) async throws -> Entitlement {
        try await service.upsertEntitlement(command: command)
    }
}
```

Enforce these activity rules:

- Wrap exactly one service.
- Expose only operations a workflow needs.
- Keep the body as transport adaptation and one service call.
- Keep business rules, multi-service coordination, and workflow starts out of activities.
- Use domain values and commands where they are already `Codable` and `Sendable`.
- Translate activity-specific payloads at the activity boundary when needed.

Use consistent activity input shapes:

- Zero arguments: no input parameter.
- One primitive or identifier: pass it directly.
- Two or more unrelated values: define a nested `<Method>Input: Codable, Sendable`.
- Existing domain command: pass the command directly rather than wrapping it again.

Keep activity method names aligned with service behavior: `get`, `find`, `list`, `create`, `update`, `upsert`, and `delete` plus the domain noun.

## Design Workflow Inputs and State

Make workflow input `Codable` and `Sendable`. Prefer stable identifiers and immutable request decisions:

```swift
public struct Input: Codable, Sendable {
    public let sessionId: String
}
```

Fetch mutable or authoritative state through activities when the workflow needs it. Do not copy large mutable provider or database records into workflow input merely to avoid an activity.

Keep secrets and sensitive one-time values out of workflow history when possible. Workflow inputs, signals, activity inputs, and activity results are normally recorded in history unless a configured payload codec/encryption layer says otherwise.

Do not return a secret from one activity and pass it into another activity while claiming it never entered history. Instead, choose one safe shape:

- Create and consume/send the secret inside one owning activity.
- Store it in a protected external system and pass an opaque reference.
- Use an approved encrypted payload codec with deliberate key ownership and rotation.

Avoid persisting secrets as workflow state or logging payloads. Verify the actual SDK/history behavior rather than relying on comments.

Store only durable coordination state on the workflow value, such as:

- A completion signal
- A queue of pending events
- A state-machine phase
- Stable identifiers needed after replay

Use `mutating` workflow methods and signal handlers when workflow state changes.

## Preserve Determinism

Workflow code must replay to the same decisions. Therefore:

- Perform database, HTTP, file, provider, and other external I/O only in activities.
- Use workflow context time rather than wall-clock APIs.
- Use workflow timers/timeouts rather than sleeping tasks.
- Avoid random UUIDs and nondeterministic collection/order behavior in workflow decisions.
- Use workflow-compatible logging through the context.
- Do not read environment variables or mutable global state from workflow code.
- Treat changes to workflow code with open histories as compatibility-sensitive.

When a code change may alter replay decisions, use the workflow engine's supported versioning or patching strategy and test representative histories. Do not casually rename workflows, activities, signals, or payload fields used by running histories.

## Choose Execution Patterns

### Interactive workflow

Use `executeWorkflow` when an API caller needs the workflow result, such as a checkout URL. Bound total latency with appropriate activity `scheduleToCloseTimeout` values.

### Background single-shot workflow

Use `startWorkflow` when the caller only needs durable acceptance. Give activities a bounded attempt duration and a deliberate retry policy.

### Signal-driven workflow

Use one authoritative starter for a signal-driven workflow. Use `signalWithStart` when the external caller owns creation of a top-level workflow and the signal may arrive before an ordinary start. Store the first accepted signal or queue signals according to the business rule, then wait with a workflow condition bounded by an in-workflow timer/timeout.

Do not trust a webhook or client-return signal as authoritative provider state. Treat it as a trigger and re-fetch live state through an activity before mutating business state.

### Child workflow

Use a child workflow when a durable sub-process has its own identity and lifecycle. Choose deliberately whether the parent awaits completion or starts a detached child. Set parent-close behavior explicitly when the child must outlive the parent.

### Long-running re-arming workflow

Use `continueAsNew` when a workflow repeatedly handles signals or cycles and its history would grow indefinitely. For concurrent signals, enqueue events and process them serially when ordering matters.

## Avoid Start Races and Lost Wakeups

Do not let two paths compete to create the same logical workflow. Choose one pattern:

1. **Parent-owned child:** the parent starts the child; external callers signal it only after its start is guaranteed or through a race-safe mechanism supported by the SDK.
2. **Externally owned top-level workflow:** orchestration services and webhooks use `startWorkflow`/`signalWithStart`; no parent later starts a child with the same workflow ID.
3. **Distinct executions:** parent and external entry points use different IDs and coordinate through an explicit supported link.

Never let `signalWithStart` create a top-level workflow using an ID that a parent may later use for `startChildWorkflow` unless the SDK explicitly supports attaching to that existing execution and the behavior is tested. A child-start conflict can fail the parent after irreversible external work has already happened.

Implement signal conflict semantics in code, not comments:

```swift
@WorkflowSignal
public mutating func setCompletion(input: Completion) {
    if completion == nil {
        completion = input // first writer wins
    }
}
```

Append to a queue instead when every signal must be processed. Do not overwrite state unconditionally when the documented rule is first-writer-wins.

Do not close successfully on a transient provider state if a later webhook or signal is expected to make the operation actionable. Choose one of:

- Keep waiting and re-fetch after a relevant signal.
- Use a durable timer and poll authoritative state.
- Handle every provider transition that can complete the process and re-arm through `continueAsNew` when appropriate.

Test delayed success, duplicate triggers, signal-before-start ordering, and every race between external starts/signals and child starts.

## Configure Timeouts, Retries, and Errors

Choose activity options from the caller's latency and recovery requirements:

- Interactive request path: cap total activity time so the API does not wait indefinitely.
- Background workflow: use `startToCloseTimeout` per attempt and a retry policy with a meaningful maximum or server policy.
- External email/provider delivery: allow a wider schedule-to-close window when retries are valuable.
- Signal wait: bound abandoned business processes with an in-workflow timer/timeout such as `context.timeout` when the domain has an expiry.

Distinguish timeout scopes:

- Activity `startToClose`: one activity attempt.
- Activity `scheduleToClose`: the whole scheduled activity including retries.
- In-workflow timer/timeout: a deterministic business wait.
- Workflow run/execution timeout: server-side lifetime limits for a run or execution chain.

Choose the narrowest scope that expresses the intended behavior; do not use the terms interchangeably.

Classify failures deliberately:

- Retry transient network, provider availability, and contention failures when the operation is safe.
- Mark immutable validation failures, missing required provider metadata, and unsupported states as non-retryable.
- Ensure retried writes are idempotent through stable source IDs, unique constraints, or idempotency keys.
- Let cancellation propagate; do not catch it and reinterpret it as an ordinary timeout or success.

Do not copy timeout numbers from another workflow without considering latency, provider behavior, and user expectations.

## Define IDs, Signals, and Authoritative Reads

Use stable, meaningful workflow IDs derived from domain identifiers:

```text
checkout-{userId}-{packageId}
fulfillment-{sessionId}
subscription-{subscriptionId}-{eventId}
```

Choose ID reuse and conflict policies to encode duplicate behavior. Use `signalWithStart` when either the signal or initial start may arrive first.

Keep workflow IDs and task queues consistent with the target repository. In an established project, preserve inline ID patterns, task-queue names, activity names, and signal names used by live histories. For greenfield work, define a clear convention before the first deployment.

Make signals small and intent-focused. When the trigger payload is not authoritative, signal only the identifiers or discriminator required to wake the workflow, then re-fetch through activities.

## Register the Worker

Construct ordinary services first, wrap them in activity containers, then register all activity containers and workflow types in the worker composition root:

```text
database/provider clients
    -> repositories and services
    -> XActivities(service: service)
    -> TemporalWorker(activityContainers: ..., workflows: ...)
    -> ServiceGroup lifecycle
```

The worker composition root is allowed to construct concrete adapters from multiple modules. Workflows still depend only on public domain/activity contracts.

Register lifecycle-managed provider/database clients beside the worker and implement graceful shutdown. Ensure every activity referenced by a registered workflow is registered on the expected task queue.

## Test Workflows

Test ordinary domain and orchestration-service branches without a Temporal server when they make no Temporal request. Use a workflow test server only to execute production workflow code and observe durable behavior.

Focus integration coverage on:

- Activity sequencing that enforces business behavior
- Signals, first-writer or queue semantics, and bounded waits
- Child-workflow behavior
- Retry/non-retryable classification when observable
- Idempotent replay of writes or duplicate triggers
- Authoritative provider re-fetch before mutation
- Compatibility of changed workflows with representative open histories

Register real activity containers around focused service mocks so the test exercises the same workflow-facing boundary. Do not test an activity merely to prove it forwards a value. Do not query workflow listings merely to prove that a call occurred.

When the repository owns one managed Temporal test server, keep its integration suite serialized while leaving unrelated unit tests parallel. Make the test-server trait own setup and teardown so the complete package suite succeeds in one unfiltered `swift test` invocation.
