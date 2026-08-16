# Solos Agent Runtime Architecture

This repository is the canonical architecture specification for the **Solos Agent Runtime**.

Solos is building an AI Chief of Staff for solo business owners: one persistent intelligence that understands the user, remembers what matters, coordinates specialized reasoning, acts across connected software, and remains responsible until work is verifiably complete.

> **Conversation is the interface. Completed work is the product.**

This repository defines the runtime architecture that makes that possible.

---

## Status

**Architecture:** Core orchestration architecture complete  
**Implementation:** Ready to begin  
**Implementation model:** Clean-slate, contract-first, component-by-component

The architecture documents in `docs/architecture/` are the source of truth.

They should not be summarized, replaced, or independently reinterpreted during implementation.

---

## Core Principle

> **The runtime owns execution. The Chief of Staff owns the outcome. Specialists own bounded reasoning.**

The language model may reason, plan, interpret, and propose.

The runtime must guarantee:

- durable state
- execution
- authorization
- approvals
- retries
- idempotency
- waiting and resumption
- evidence
- cancellation
- completion

---

## Repository Structure

```text
.
├── README.md
├── AGENTS.md
├── CONTRIBUTING.md
├── IMPLEMENTATION_ORDER.md
├── ARCHITECTURE_INDEX.md
├── .editorconfig
├── .gitignore
│
├── docs/
│   └── architecture/
│       ├── 01-agent-definition-and-agent-registry.md
│       ├── 02-chief-of-staff-architecture.md
│       ├── 03-assignment-protocol.md
│       ├── 04-capability-and-tool-architecture.md
│       ├── 05-context-distribution-system.md
│       ├── 06-work-graph-and-execution-model.md
│       ├── 07-policies-approvals-and-authority.md
│       ├── 08-model-routing-and-execution-budgets.md
│       ├── 09-evaluation-and-evidence.md
│       └── 10-initial-agent-set.md
│
└── .github/
    ├── pull_request_template.md
    └── ISSUE_TEMPLATE/
        ├── component-implementation.md
        └── architecture-change.md
```

---

## Architecture Documents

The core runtime is defined across ten components.

### 1. Agent Definition and Agent Registry

Defines which reasoning roles are allowed to exist inside Solos, how they are versioned, what assignments they accept, and what capabilities they may receive.

[Read the document](docs/architecture/01-agent-definition-and-agent-registry.md)

### 2. Chief of Staff Architecture

Defines the single accountable intelligence responsible for understanding the user, forming objectives, delegating bounded work, coordinating results, and communicating outcomes.

[Read the document](docs/architecture/02-chief-of-staff-architecture.md)

### 3. Assignment Protocol

Defines the durable typed contract used to delegate work from the Chief of Staff to specialist agents.

[Read the document](docs/architecture/03-assignment-protocol.md)

### 4. Capability and Tool Architecture

Defines the execution boundary between agent intent and real-world provider actions.

[Read the document](docs/architecture/04-capability-and-tool-architecture.md)

### 5. Context Distribution System

Defines how every reasoning process receives the smallest authoritative, relevant, and permitted context necessary for its responsibility.

[Read the document](docs/architecture/05-context-distribution-system.md)

### 6. Work Graph and Execution Model

Defines the durable graph used to execute, pause, resume, retry, cancel, parallelize, and complete work.

[Read the document](docs/architecture/06-work-graph-and-execution-model.md)

### 7. Policies, Approvals, and Authority

Defines who may do what, how authority is scoped, when approval is required, and how model intent is prevented from becoming unchecked real-world authority.

[Read the document](docs/architecture/07-policies-approvals-and-authority.md)

### 8. Model Routing and Execution Budgets

Defines how Solos chooses models, escalates intelligence, handles fallbacks, and bounds latency, cost, retries, and computational complexity.

[Read the document](docs/architecture/08-model-routing-and-execution-budgets.md)

### 9. Evaluation and Evidence

Defines how results are validated, how claims are grounded, and what proof is required before Solos may say work is complete.

[Read the document](docs/architecture/09-evaluation-and-evidence.md)

### 10. Initial Agent Set

Defines the smallest production organization of bounded specialists beneath the Chief of Staff.

[Read the document](docs/architecture/10-initial-agent-set.md)

---

## First-Principles Architectural Rules

Implementation must preserve these invariants:

1. **The user experiences one Chief of Staff.**
2. **Agents never own canonical runtime state.**
3. **Specialists receive bounded assignments, not entire conversations.**
4. **Specialists do not delegate in v1.**
5. **Consequential writes are deterministic capability operations.**
6. **Provider APIs are hidden behind Solos-owned capability contracts.**
7. **Provider credentials never enter model context.**
8. **PostgreSQL is the canonical durable runtime state.**
9. **Queue messages contain references, not canonical mutable state.**
10. **Waiting is durable state; workers do not remain alive while waiting.**
11. **Corrections supersede only affected work.**
12. **External writes require idempotency and evidence.**
13. **Agent output is not automatically trusted.**
14. **Work Item completion belongs to the runtime.**
15. **No mandatory router → planner → evaluator model chain exists for every message.**
16. **Model calls are bounded resources.**
17. **Parallelize intelligence; coordinate consequential action.**
18. **The old Solos runtime is not an implementation source.**

---

## Clean-Slate Rule

The new runtime is a clean-slate implementation.

Do **not** copy or reuse:

- runtime source code
- database schemas
- prompts
- workflows
- abstractions
- agent loops
- integration architecture
- tests

from the old Solos runtime.

The old system may be consulted only as historical evidence of requirements, product behavior, and failure modes.

The architecture in this repository defines the new implementation.

---

## How to Use This Repository

### For engineers

Before implementing a component:

1. Read this `README.md`.
2. Read `AGENTS.md`.
3. Read `ARCHITECTURE_INDEX.md`.
4. Read the full architecture document for the component.
5. Read the architecture documents for its direct dependencies.
6. Create or claim a GitHub issue for that component.
7. Implement behind shared contracts.
8. Add tests for normal, failure, retry, cancellation, and recovery behavior.
9. Open a pull request using the repository template.

### For Codex

Start with `AGENTS.md`.

When implementing one component, Codex should treat that component's architecture file as authoritative while also respecting every cross-component contract referenced by it.

Codex must not silently redesign adjacent systems.

---

## Parallel Implementation

The architecture was intentionally designed so multiple engineers can work in parallel.

However, parallel work is safe only when shared contracts are established first.

Recommended high-level order:

```text
Shared contracts and runtime primitives
        ↓
Registry / Capabilities / Policies / Model Routing / Evidence
        ↓
Work Graph and execution
        ↓
Assignment runtime + Context Distribution
        ↓
Chief of Staff
        ↓
Initial specialist agents
        ↓
End-to-end workflows
```

See [IMPLEMENTATION_ORDER.md](IMPLEMENTATION_ORDER.md) for the recommended sequence and integration gates.

---

## Architectural Change Policy

Implementation discoveries may require architecture changes.

That is expected.

But architecture changes must be explicit.

Do not silently change a shared semantic contract inside one component implementation.

If a change affects:

- ownership
- shared types
- lifecycle semantics
- authority
- completion
- evidence
- dependency direction
- agent responsibilities

open an **Architecture Change** issue first and document the reasoning.

---

## What Success Looks Like

The architecture is working when a user can say something like:

> Send Martin the July report.

and Solos can reliably:

```text
understand objective
→ resolve Martin
→ analyze July
→ compose message
→ obtain required approval
→ execute one deterministic send
→ verify provider evidence
→ tell the user it is complete
```

while also correctly handling:

- corrections
- interruptions
- cancellations
- retries
- provider timeouts
- ambiguous external outcomes
- model failures
- application deployments
- long-running waits

without losing the user's objective.

---

## License

No license is included yet.

Choose and add the repository license deliberately before making the repository public.
