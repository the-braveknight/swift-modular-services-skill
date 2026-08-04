# Observability and Operations

## Contents

- Define operational ownership
- Use structured telemetry
- Propagate correlation and traces
- Instrument module and runtime boundaries
- Choose metrics and objectives
- Design health and readiness
- Alert on actionable conditions
- Reconcile durable and provider state
- Protect telemetry data
- Diagnose incidents
- Test observability behavior
- Review operational readiness

## Define Operational Ownership

Make each runtime and capability diagnosable without reading its implementation. Assign ownership for:

- API, worker, migration, and administrative runtimes
- Capability-specific business operations
- Temporal workflows and task queues
- PostgreSQL pools, schemas, and migrations
- Provider gateways and webhooks
- Extracted service endpoints and remote clients

Define who responds, which dashboard or query answers the first incident questions, and which dependency owns each failure. Do not create telemetry that has no operational consumer or retention purpose.

Keep diagnostic telemetry, security audit records, and business analytics separate. They have different reliability, access, sampling, and retention requirements.

## Use Structured Telemetry

Bootstrap logging, tracing, and metrics once in the executable composition root. Give every runtime a stable service name, environment, version, and instance identity through resource attributes rather than repeating them in every message.

Use structured logs with stable field names. Prefer fields such as:

- `module`
- `operation`
- `requestId`
- `traceId`
- `workflowId`
- `workflowRunId`
- `providerEventId`
- `errorType`
- `attempt`
- `outcome`

Use one naming convention across runtimes. Emit a concise event name and fields instead of interpolating essential data into prose that cannot be queried reliably.

Log state transitions and unexpected failures at their owning boundary. Avoid logging the same error at every layer; add context once, then preserve the underlying typed error and trace.

Choose levels consistently:

- Debug: local diagnostic detail safe to suppress in production.
- Info: meaningful lifecycle or business milestone.
- Warning: degraded or unexpected behavior that recovered or needs attention.
- Error: failed operation or invariant requiring investigation.

Do not use error logs for expected validation, not-found, or authorization denials unless their rate or context is operationally significant.

## Propagate Correlation and Traces

Create or accept a request correlation identifier at the public delivery boundary. Propagate standard trace context through supported HTTP/gRPC headers and provider SDK instrumentation. Preserve correlation across:

```text
HTTP request
    -> controller and service
    -> Temporal workflow start/signal
    -> workflow and activities
    -> provider or remote service call
    -> webhook or reconciliation follow-up
```

Use trace context for causal latency analysis and a stable business/process identifier for work that outlives one trace. Do not assume an original HTTP span remains open for a workflow that completes hours later.

Attach workflow IDs, provider object/event IDs, and domain operation IDs as links or safe structured fields. Never use trace or request IDs as authorization, deduplication, or business identity.

When accepting caller-supplied correlation IDs, validate length and character constraints before logging or forwarding them. Generate a safe replacement when invalid.

## Instrument Module and Runtime Boundaries

Instrument boundaries where ownership, latency, or failure semantics change:

- HTTP/gRPC server operation
- Orchestration-service workflow start, execute, or signal
- Temporal activity execution
- PostgreSQL query/transaction aggregate
- Provider gateway operation
- Remote service client operation
- Webhook verification and durable acceptance
- Reconciliation cycle

Avoid tracing every internal function. Let automatic framework instrumentation cover generic transport/database spans, then add capability attributes and events only where they explain application behavior.

For in-process cross-module calls, retain the current trace and identify the provider module or operation. After extraction, preserve the same conceptual operation name while adding client/server spans and remote failure attributes.

Do not attach full SQL, request bodies, provider payloads, tokens, email addresses, or other sensitive/high-cardinality values by default.

## Choose Metrics and Objectives

Start from user- or process-visible outcomes. For each important entry point or durable process, define an objective and the measurements that explain misses.

Use low-cardinality dimensions such as operation, module, outcome class, provider, task queue, or error category. Never label metrics by user ID, workflow ID, request ID, email, provider object ID, raw route, or error message.

Measure where relevant:

- Request rate, latency, and error ratio by normalized operation
- Workflow started, completed, failed, timed out, and stuck duration
- Activity attempts, retries, terminal failures, and schedule-to-start latency
- Task-queue or consumer backlog and oldest-item age
- Database pool wait, saturation, transaction latency, and rollback rate
- Provider latency, availability, throttling, and webhook delay
- Reconciliation drift, repair attempts, and unresolved age
- Deployment version and readiness transitions

Distinguish accepted durable work from completed business work. An HTTP `202` or successful workflow start is not proof that fulfillment, delivery, or reconciliation succeeded.

Prefer histograms for latency and duration. Define units and stable bucket strategy. Avoid counters whose meaning changes between releases.

## Design Health and Readiness

Keep probes narrow:

- **Liveness:** confirm the process can continue running; avoid dependency calls that cause restart storms.
- **Readiness:** confirm the instance can accept its assigned work.
- **Startup:** allow bounded initialization where the platform supports a separate startup probe.
- **Dependency diagnostics:** expose detailed operator-only state separately.

An API may be unready when it cannot serve required database-backed operations. A worker may be unready until its Temporal connection and required activity registrations are established. Decide whether optional provider failure should reduce readiness or only degrade a specific operation.

Do not expose secrets, connection strings, stack traces, database topology, provider payloads, or internal hostnames through public health endpoints. Keep expensive end-to-end synthetic checks outside high-frequency platform probes.

## Alert on Actionable Conditions

Alert on sustained user impact, exhausted recovery capacity, or an invariant needing intervention. Good candidates include:

- Objective burn rate or sustained error/latency increase
- Workflow terminal failures or oldest-open age beyond the business limit
- Retry storms, task-queue backlog, or schedule-to-start delay
- Database saturation or migration failure
- Provider webhook silence when traffic is expected
- Reconciliation drift or unresolved repair age
- Readiness loss across enough instances to reduce capacity
- Security-significant denial or signature-failure anomaly

Do not page on every exception, retry, or single-instance restart. Include the affected service/module, environment, symptom, safe correlation hints, dashboard/query, and first diagnostic action in alert metadata or its runbook.

Use separate severity and notification paths for immediate user impact, approaching capacity, and informational anomalies.

## Reconcile Durable and Provider State

Design reconciliation for processes spanning PostgreSQL, Temporal, and external providers. Treat it as a normal reliability mechanism, not an emergency script.

Define:

- Which system is authoritative for each fact
- The stable identifier connecting records
- Expected convergence time
- Safe idempotent repair operation
- Maximum automatic repair attempts
- Escalation and manual-resolution state
- Metrics for detected, repaired, and unresolved drift

Re-fetch authoritative provider state before repair. Make reconciliation safe to rerun and prevent two reconcilers from competing through leases, workflow IDs, database constraints, or another explicit ownership mechanism.

Retain enough safe evidence to explain a repair without persisting raw sensitive payloads unnecessarily.

## Protect Telemetry Data

Treat telemetry as production data. Apply access controls, retention, deletion, and regional/compliance requirements appropriate to what it contains.

Use an allowlist for recorded headers and attributes. Redact authorization headers, cookies, passwords, reset tokens, one-time codes, signing material, connection strings, and provider secrets. Review exception descriptions because SDK errors may contain request or response payloads.

Hashing a low-entropy secret does not make it safe. Prefer omission or a deliberately scoped opaque identifier. Avoid personal data in span names, metric labels, log messages, and workflow search attributes.

Keep sampling decisions compatible with incident needs. Preserve errors and rare critical workflows deliberately, but do not claim audit completeness from sampled traces or best-effort logs.

Read [security-and-authorization.md](security-and-authorization.md) for secret classification and security audit records.

## Diagnose Incidents

Make the first investigation path answer:

1. Which user-visible operation or durable process is affected?
2. When did impact begin, and which versions or instances changed?
3. Is the failure in delivery, a capability service, Temporal, PostgreSQL, a provider, or a remote dependency?
4. Are retries recovering, accumulating backlog, or amplifying load?
5. Which stable workflow/provider/domain identifiers can locate one representative execution safely?
6. Is reconciliation possible, and what prevents duplicate repair?

Prefer drill-down from objectives to operation metrics, traces, structured logs, workflow history, and safe database/provider diagnostics. Preserve causal errors and normalized error categories so operators do not depend on brittle message searches.

For every critical process, document a bounded recovery action: retry, resume, reconcile, drain, roll back, disable an optional integration, or escalate for manual resolution. Ensure the action is idempotent and authorized.

## Test Observability Behavior

Do not assert every log line. Test contracts whose failure would hide or misrepresent incidents:

- Correlation and trace context propagation across custom adapters
- Normalized route/operation names and bounded metric cardinality
- Workflow/activity/provider identifiers present where needed
- Secret and sensitive-field redaction
- Readiness transitions for required dependencies
- Reconciliation metrics and unresolved-state reporting
- Typed error-to-outcome categorization
- Telemetry exporter shutdown and flush when owned by the runtime

Use an in-memory exporter or captured logger for focused tests. Keep telemetry failures from changing business outcomes unless observability is explicitly part of the operation's guarantee, such as a durable security audit record.

## Review Operational Readiness

Before shipping a capability, workflow, provider integration, or extracted service, verify:

- Runtime and capability ownership are identifiable.
- Important user and durable-process outcomes have measurable objectives.
- Correlation crosses delivery, Temporal, provider, and remote boundaries.
- Metrics use normalized low-cardinality dimensions.
- Logs and traces record useful transitions without secrets or raw sensitive payloads.
- Liveness and readiness reflect the runtime's assigned work without causing restart storms.
- Alerts identify sustained actionable conditions and link to a first diagnostic path.
- Cross-system state has an authoritative source and reconciliation strategy.
- One representative execution can be followed safely from entry point to final outcome.
- Shutdown flushes owned telemetry within a bounded period.
