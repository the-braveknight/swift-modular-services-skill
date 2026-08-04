# Contract-First API Delivery

## Contents

- API target and composition
- Contract source of truth
- Generated types and mapping
- Controllers
- Request contexts and security
- Error responses
- Route composition
- Testing

## API Target and Composition

Keep the HTTP API as a library delivery target that imports public capability contracts. Let the executable composition root construct concrete services and assemble the API. Do not put lifecycle management, environment decoding, database construction, or provider-client construction in the API library.

Use Hummingbird as a transport adapter when it is the selected framework; preserve the same controller/service boundary with another framework.

## Contract Source of Truth

When the repository is OpenAPI-first, update the OpenAPI document before writing controller code. Treat it as the authoritative source for:

- Paths and operations
- Request and response schemas
- Authentication requirements
- Status codes and error shapes
- Reusable identifiers and enums

Run the configured Swift OpenAPI generator through the normal build/plugin workflow. Do not hand-write parallel request/response structs that can drift from generated schemas.

Keep the contract synchronized when adding, changing, or removing routes. Do not edit generated Swift sources directly.

## Generated Types and Mapping

Use generated request/response types at the HTTP boundary and domain types inside services. Add explicit adapter extensions:

```text
API/Extensions/
├── Requests/<Request>+Command.swift
├── Responses/<Response>+Domain.swift
└── Shared/<Enum>+Domain.swift
```

Use request extensions to produce domain commands. Use response extensions to map domain entities into generated response schemas. Keep enum conversion explicit when API and domain representations may evolve independently.

Do not return domain entities directly from controllers. Do not make domain entities conform to HTTP response protocols merely for convenience. Do not expose persistence-only audit or internal fields unless the contract explicitly requires them.

## Controllers

Keep each controller method as a transport adapter:

```text
decode generated request
    -> derive trusted identity/context
    -> call one service
    -> map domain result to generated response
```

Let a controller depend on a service protocol, not repositories or concrete adapters. Move coordination of two or more services into an orchestration service so the controller still makes one call.

Parse path/query parameters using framework facilities and map them into domain values before the service call. Let the authenticated request context supply user identity; never trust a client-provided user ID for an authenticated ownership operation.

Keep webhook handlers thin: obtain raw bytes and signature, call one verification/orchestration service, and return the contractually correct acceptance status. Perform provider signature verification before decoding or starting durable work.

## Request Contexts and Security

Define request contexts by authorization surface, for example:

- Public context: no authenticated identity required.
- Authenticated-user context: verified token and user claims.
- Admin context: verified token plus an explicit role check during context conversion.

Apply authentication and authorization middleware at route-group boundaries. Keep resource-specific authorization in the owning service when it depends on domain state.

When RLS is used, bridge verified request identity into a transaction-scoped database context in middleware/database adapters. Ensure missing identity fails closed and task-local state cannot leak between requests.

Keep secrets and provider keys out of generated API schemas and logs. Use restricted provider credentials appropriate to the operation.

Read [security-and-authorization.md](security-and-authorization.md) when designing token validation, authorization ownership, service identities, audit records, or extracted-service authentication.

## Error Responses

Map typed domain/application errors to one project-wide problem/error representation. Keep one conformance or mapper per error type when that is the repository convention.

Choose statuses by public semantics:

- Invalid request/domain value: `400` or contract-selected validation status
- Unauthenticated: `401`
- Authenticated but forbidden: `403`
- Missing owned resource: `404`
- Uniqueness/state conflict: `409`
- Accepted durable work: `202` when processing continues asynchronously

Do not expose database errors, provider payloads, stack traces, or secrets in HTTP responses. Preserve retryable provider/Temporal failures as non-success responses when the external sender must retry.

## Route Composition

Group routes by capability and authorization surface. Keep user and admin surfaces distinct when their contexts and response visibility differ.

Let an API assembly type accept service protocols as dependencies and register controllers/routes. Let the executable composition root build those concrete services and pass them in.

Keep API schema resources and generator configuration attached to the API target. Keep executable configuration bridges in the CLI/composition target.

## Testing

Use the framework's router testing support for controller-owned behavior. Test:

- Request-to-command mapping
- Trusted identity overriding or excluding client-provided identity
- Domain-to-response mapping
- Intended status and problem mapping
- Route authentication/authorization boundaries
- Raw webhook signature/body forwarding

Build request bodies from generated schemas rather than hard-coded JSON, except where the contract requires raw bytes. Test malformed decoding or generic problem formatting only once at the framework/middleware boundary, not in every controller suite.

Do not duplicate every domain service test through HTTP. Use representative API tests for transport wiring and keep business-rule coverage in the owning module test target.
