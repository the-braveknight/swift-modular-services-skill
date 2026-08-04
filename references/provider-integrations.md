# Provider Integrations

## Contents

- Choose the port shape
- Keep ownership local
- Build an anti-corruption layer
- Own metadata conventions
- Design idempotent operations
- Handle webhooks
- Re-fetch authoritative state
- Translate failures
- Configure and compose providers
- Test integrations

## Choose the Port Shape

Choose between two public shapes:

| Shape | Use when | Port | Adapter |
|---|---|---|---|
| Provider gateway | One provider exposes several related domain operations and is the intended implementation | `XServiceProtocol` | `XService` |
| Swappable capability | One narrow capability may have multiple providers or local implementations | capability noun without `Protocol` | `<Provider><Capability>` |

Examples:

- A payments provider gateway may expose customer, checkout, subscription, and verification operations behind `PaymentProviderServiceProtocol`.
- Email delivery may use `UserEmailSender`, implemented by `ResendUserEmailSender` or `ConsoleUserEmailSender`.

Do not create a shared provider/infrastructure module merely because several capabilities use the same vendor. Let each capability own the port and adapter for the behavior it needs. Share a lifecycle-safe SDK client in the composition root when appropriate, not business-facing adapter types.

## Keep Ownership Local

Place the provider port in the capability's `Ports/Services/` and its implementation in `Adapters/Services/`. Keep provider configuration as a pure value owned by the adapter or gateway.

Let domain/orchestration services and activities call the public provider port. Do not let controllers, workflows, or repositories call the vendor SDK directly.

Keep provider gateways free of unrelated database access and Temporal orchestration. If an operation coordinates provider calls with module persistence, put that coordination in a domain/orchestration service or workflow and reach each side through its port/activity.

## Build an Anti-Corruption Layer

Translate vendor SDK values into domain-shaped types at the gateway:

```text
domain command/value
    -> provider gateway
    -> vendor request
    -> vendor response
    -> domain provider-edge value/error
```

Do not leak vendor request/response types into domain services, workflow inputs, API schemas, or persistence models. Group provider-edge domain values under a provider-named subfolder only when that keeps the external boundary visibly distinct from the core business domain.

Use domain terminology in public methods. Let the gateway synthesize vendor-specific parameters, headers, metadata, expansions, and request shapes.

## Own Metadata Conventions

Keep provider metadata and external-reference conventions private to the gateway. Define keys once, write them during creation, and decode them during retrieval in the same adapter.

Do not make workflows or controllers know raw metadata keys. Expose typed properties such as `userId`, `packageId`, or `sourceId` on the anti-corruption model.

Treat missing metadata according to its mutability:

- Missing immutable metadata required for an invariant: typed non-retryable failure.
- Missing optional provider data: represent it explicitly and let the owning service decide.
- Malformed untrusted metadata: reject or ignore according to the public contract; never silently reinterpret it.

## Design Idempotent Operations

Every provider mutation that may be retried needs a stable idempotency strategy:

- Derive keys from stable domain identifiers, workflow IDs, source IDs, or tokens.
- Reuse the same key across retries of the same logical operation.
- Never generate a random key inside each retry attempt.
- Store the resulting provider identifier when later reconciliation requires it.

Examples of stable key shapes:

```text
customer-{userId}
refund-{purchaseId}
welcome-email-{userId}
```

Treat idempotency as part of the business operation, not a controller concern. Test the actual header/key passed to the provider adapter when the SDK seam permits it.

When the provider offers no idempotency guarantee, document the weaker semantics explicitly:

- Accept at-least-once delivery and possible duplicates after an ambiguous timeout, or add a module-owned delivery ledger/reconciliation process.
- Recognize that a ledger cannot eliminate the interval where the provider accepted a request but its response was lost.
- Do not claim exactly-once delivery when the provider cannot prove it.
- Drain, version, or route in-flight durable activities deliberately when switching providers; an idempotency key from one provider has no meaning to another.

## Handle Webhooks

Verify the provider signature against the raw body before decoding or starting work. Distinguish verification failure from payload-decoding failure.

Keep the HTTP handler thin:

```text
raw body + signature
    -> provider gateway verification/classification
    -> orchestration service
    -> start/signal durable workflow
    -> acceptance response
```

Classify provider events into a small domain-facing event enum. Keep unknown events safely ignored or acknowledged according to the provider retry contract.

Return success only after the system has durably accepted required work. Let verification and durable-acceptance failures propagate as non-success when the provider must retry.

Expect duplicate, delayed, and out-of-order webhooks. Use stable provider event IDs, workflow IDs, database constraints, and idempotent activities to make replay safe.

## Re-fetch Authoritative State

Treat webhook and client-return payloads as triggers, not proof of current provider state, when the provider supports authoritative retrieval.

Re-fetch live state through the provider gateway/activity before granting entitlements, shipping goods, revoking access, or performing another consequential mutation. This protects against forged client data, incomplete webhook payloads, and stale ordering.

Handle every provider transition that can make the process actionable. Do not close a durable workflow on a transient state if a later event may complete the operation without a re-arming path.

## Translate Failures

Map vendor failures into meaningful provider-edge errors:

- Authentication/configuration failure
- Signature verification failure
- Invalid or unsupported response
- Rate limiting or temporary unavailability
- Provider-declared validation failure
- Missing authoritative resource

Preserve the distinction between retryable transport/provider failures and non-retryable immutable validation errors. Do not expose raw provider payloads, SDK error objects, request headers, or secrets through public API errors.

Classify timeouts after request transmission as potentially ambiguous when the provider may have accepted the operation. Decide whether retrying may duplicate the effect and surface that risk in the owning workflow/service policy.

Let workflows classify retry behavior at the activity boundary. Let HTTP delivery map public capability errors without revealing vendor internals.

## Configure and Compose Providers

Let the adapter own a pure `Configuration: Sendable` containing only required values. Decode environment variables and secrets in the executable composition root, then construct the SDK client and adapter there.

Use least-privilege provider credentials. Fail fast on missing credentials or structurally invalid configuration. Never log keys, webhook secrets, tokens, or complete sensitive payloads.

Share one SDK/HTTP client only when concurrent reuse and lifecycle ownership are supported. Register lifecycle-managed clients with the runtime service group.

## Test Integrations

Test provider-independent service behavior with a mock port. Test adapter behavior where translation matters:

- Domain command to provider request mapping
- Metadata and idempotency keys
- Provider response to anti-corruption type mapping
- Signature verification and event classification
- Retryable versus terminal error mapping
- Missing or malformed authoritative fields
- Duplicate webhook/provider retry behavior

Use provider sandbox/integration tests only for behavior that cannot be proven at a stable local seam. Keep secrets out of fixtures and test output.
