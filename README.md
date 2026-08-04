# Swift Modular Services

`swift-modular-services` is a Codex skill for designing, creating, reviewing, and evolving modular Swift backends. It uses capability-based Swift Package Manager targets, explicit Domain/Ports/Adapters boundaries, typed service protocols, and executable composition roots.

The skill starts with a modular monolith and preserves the same business boundaries when a capability later needs to become a separately deployable HTTP or gRPC service. It does not introduce messaging, event publication, or distributed-system machinery unless the use case requires them.

## What it covers

- Capability ownership and acyclic SPM dependency graphs
- Domain, port, adapter, service, repository, and activity boundaries
- Direct typed collaboration between in-process modules
- PostgreSQL schema ownership, transactions, migrations, and RLS
- Contract-first OpenAPI delivery with Hummingbird
- Temporal workflows, activities, signals, retries, and compatibility
- API, worker, migration, CLI, and extracted-service composition roots
- Provider integrations, idempotency, and authoritative state
- Testing with Swift Testing and an unfiltered `swift test`
- Security, authorization, observability, operations, and staged evolution

## Install

Clone the repository and copy the skill package into your personal Codex skills directory:

```bash
git clone https://github.com/the-braveknight/swift-modular-services-skill.git
mkdir -p ~/.codex/skills/swift-modular-services
cp -R swift-modular-services-skill/SKILL.md \
  swift-modular-services-skill/agents \
  swift-modular-services-skill/references \
  ~/.codex/skills/swift-modular-services/
```

Restart Codex if the skill is not discovered immediately.

## Use

Invoke the skill explicitly:

```text
Use $swift-modular-services to design a modular Swift backend for subscriptions and entitlements.
```

```text
Use $swift-modular-services to add a Payments module to this backend and wire it to Entitlements through its public service protocol.
```

```text
Use $swift-modular-services to review whether Authentication is ready to become a separately deployable service.
```

The skill also triggers for requests involving modular Swift backend boundaries, cross-module collaboration, Temporal workflow design, contract-first APIs, runtime composition, and service extraction.

## Repository layout

```text
.
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── architecture-baseline.md
    ├── module-implementation.md
    ├── creation-workflows.md
    ├── separate-deployment.md
    ├── temporal-workflow-design.md
    ├── postgres-persistence.md
    ├── contract-first-api.md
    ├── testing-strategy.md
    ├── runtime-composition.md
    ├── provider-integrations.md
    ├── evolution-and-compatibility.md
    ├── security-and-authorization.md
    └── observability-and-operations.md
```

`SKILL.md` contains the core workflow and routes Codex to focused references only when they apply.

## Validate

Use the validator bundled with Codex's `skill-creator` skill:

```bash
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py \
  ~/.codex/skills/swift-modular-services
```
