# Security and Authorization

## Contents

- Establish trust boundaries
- Separate authentication and authorization
- Model trusted identity
- Enforce module-owned authorization
- Propagate identity in-process
- Apply PostgreSQL row-level security
- Authenticate extracted services
- Manage secrets and sensitive data
- Secure Temporal and provider boundaries
- Record security audit events
- Test failure behavior
- Review the boundary

## Establish Trust Boundaries

Identify every boundary where untrusted data becomes trusted enough to influence a capability:

- Public HTTP or gRPC request
- Provider webhook
- Administrative CLI or internal endpoint
- Temporal signal, workflow, or activity payload
- In-process call from another capability
- Remote call from another deployed service
- Database session receiving row-level security context

Document who authenticates the caller, which claims are accepted, who authorizes the operation, and how identity reaches persistence. Treat transport encryption, network location, and possession of an identifier as insufficient authorization by themselves.

Default to denying access when required identity, role, tenant, scope, signature, or policy state is absent or malformed.

## Separate Authentication and Authorization

Let authentication prove who or what is calling. Let authorization decide whether that principal may perform a capability operation on a particular resource.

Perform authentication at the delivery boundary:

- Validate token signature, issuer, audience, expiry, and required claims.
- Verify webhook signatures against the raw request body before decoding or starting work.
- Authenticate administrative and service-to-service requests independently of public user tokens.
- Reject caller-supplied role, tenant, or user headers unless a trusted gateway creates and protects them under an explicit contract.

Perform coarse authorization at route-group conversion when it depends only on verified claims, such as requiring an administrator role. Perform resource- and state-dependent authorization in the owning service, inside the transaction when the decision depends on mutable data.

Do not place business authorization solely in middleware. A service must remain safe when called from HTTP, CLI, Temporal activities, tests, or another module.

## Model Trusted Identity

Use a small immutable `Sendable` identity value created only after verification. Distinguish principal kinds rather than representing every caller as an end user:

```swift
public enum Principal: Sendable, Equatable {
    case user(UserID)
    case administrator(AdminID)
    case service(ServiceIdentity)
    case system(SystemJob)
}
```

Add tenant, scopes, or delegated-user information only when the domain requires them. Keep raw JWTs, authorization headers, provider signatures, and SDK credential types out of domain commands.

Prefer explicit identity parameters or an immutable trusted execution context at service boundaries. Use task-local propagation only as scoped infrastructure plumbing; never let mutable identity leak between requests, tasks, transactions, or workflow executions.

Do not accept a user ID inside a command as proof that the caller owns that identity. Derive the actor from the trusted context and keep separately named subject/resource IDs when an administrator or service may act on another subject.

## Enforce Module-Owned Authorization

Let each capability own policies involving its resources and invariants. A caller module may establish identity and request an operation, but it must not reproduce the provider module's authorization rules.

Keep authorization close to the protected state:

```text
delivery authentication
    -> trusted principal
    -> owning service authorization
    -> module-owned transaction and repositories
```

Define typed forbidden or policy errors where callers need stable handling. Avoid leaking whether a protected resource exists when that distinction would disclose information; deliberately map forbidden and not-found behavior at the public boundary.

When one module calls another in-process, pass the principal or a narrowly defined authority context if the provider must authorize the original actor. Do not let the consumer assert a stronger role than it authenticated.

For privileged internal operations, expose a separately named service method or authority type rather than a boolean such as `isAdmin`. Keep construction of privileged authority in composition roots or verified boundary adapters.

## Propagate Identity In-Process

Preserve the distinction between:

- **Actor:** the principal requesting the business action.
- **Caller:** the module or service making the immediate call.
- **Subject:** the resource owner affected by the action.

Pass only the identity information the provider capability needs. Avoid passing full request contexts across modules because they couple domain services to transport and may accidentally grant unrelated claims.

Do not serialize an in-memory authorization object as a remote credential during extraction. Replace it with authenticated service identity plus explicitly validated delegated-user claims.

## Apply PostgreSQL Row-Level Security

Use RLS as defense in depth for tenant or user isolation, not as a substitute for service authorization.

Set verified identity and role values transaction-locally through the concrete database adapter. Ensure pooled connections cannot retain identity after the transaction. Define policies that return no protected rows when context is absent.

Use distinct database roles for migration, runtime, read-only diagnostics, and separately deployed services. Do not give an application role `BYPASSRLS`, schema ownership, or broad cross-capability privileges merely to simplify configuration.

Test ordinary users, cross-tenant access, administrators, internal workers, missing context, rollback, and pooled-connection reuse. Read [postgres-persistence.md](postgres-persistence.md) for transaction and policy structure.

## Authenticate Extracted Services

When a module becomes a remote service, authenticate both the immediate service caller and any delegated end-user identity required by policy.

Define:

- Service credential or workload identity issuer and audience
- Credential rotation and expiry
- Allowed calling services and operations
- Delegated-user representation and validation
- Replay protection where requests trigger valuable or non-idempotent work
- Deadline, cancellation, and authorization error mapping

Authorize the service principal before trusting delegated claims. Prefer short-lived workload credentials or platform identity over shared static secrets. If static secrets are unavoidable, scope and rotate them independently per caller.

Do not rely on private networking as authentication. Preserve least privilege when two services share a cluster, host, or service mesh.

## Manage Secrets and Sensitive Data

Decode secrets only in executable composition roots or dedicated secret-provider adapters. Pass configured clients or narrow credential values to the component that needs them. Never expose secrets through public module contracts, generated API schemas, domain entities, error descriptions, debug dumps, metrics labels, or health endpoints.

Classify at least:

- Signing and encryption keys
- Database and provider credentials
- Access and refresh tokens
- Passwords, reset tokens, and one-time codes
- Payment/provider payloads and personal data
- Webhook secrets and raw authorization headers

Redact by allowlisting safe telemetry fields rather than trying to blacklist every secret field. Avoid high-cardinality or personal identifiers in metric labels. Define retention and deletion behavior for persisted sensitive data and audit records according to project requirements.

Keep encryption-key ownership and rotation separate from encrypted application data. Do not invent custom cryptography when the platform or an established library provides the required primitive.

## Secure Temporal and Provider Boundaries

Assume workflow inputs, signals, activity inputs, activity results, failures, and memo/search attributes may be retained in Temporal history. Keep passwords, reset tokens, provider secrets, and sensitive raw payloads out of history unless an approved payload codec protects them with deliberate key ownership and rotation.

Prefer one owning activity that creates and consumes a secret, or store the secret externally and pass an opaque reference. Do not log complete activity payloads or failures containing sensitive provider responses.

Treat webhooks and Temporal signals as triggers, not authority over provider state. Verify webhook authenticity, deduplicate stable event IDs, and re-fetch authoritative state through the provider gateway before protected mutations.

Read [temporal-workflow-design.md](temporal-workflow-design.md) and [provider-integrations.md](provider-integrations.md) for durable execution and provider-state rules.

## Record Security Audit Events

Create durable audit records for security-relevant actions when accountability is required. Record the actor, caller, action, subject/resource identifier, outcome, timestamp, and safe correlation identifier. Record policy or reason codes only when they do not expose secrets.

Keep audit records distinct from diagnostic logs. Make them append-oriented, access-controlled, retention-aware, and resistant to ordinary application mutation. Do not claim an audit guarantee if logs can be sampled, dropped, or rewritten.

Avoid storing raw credentials, complete tokens, passwords, or sensitive request bodies in audit data.

## Test Failure Behavior

Test the policy owned by the application rather than token-library internals. Include:

- Missing, expired, malformed, wrong-issuer, and wrong-audience credentials
- Valid identity without the required role or resource ownership
- Cross-user and cross-tenant access
- Internal service without delegated authority
- Caller attempts to forge identity, role, or tenant metadata
- RLS behavior with absent context and pooled-connection reuse
- Webhook signature failure, duplicates, and replay
- Secret absence from logs, errors, API responses, and Temporal histories where observable
- Extracted-service credential rotation and old/new compatibility

Assert fail-closed outcomes and absence of protected side effects, not merely the returned status.

## Review the Boundary

Before shipping a new capability or extraction, verify:

- Every entry point has an explicit trust boundary.
- Authentication validates the intended issuer, audience, signature, and lifetime.
- The owning service enforces state-dependent authorization.
- Privileged operations use explicit authority rather than caller-controlled flags.
- Required identity reaches each transaction and missing context fails closed.
- Service-to-service calls authenticate the service and validate delegated identity separately.
- Secrets and sensitive payloads stay out of public contracts, logs, metrics, health output, and Temporal history.
- Database roles, provider keys, and service credentials follow least privilege.
- Security-relevant actions have an intentional audit policy.
- Tests prove denial paths produce no protected mutation.
