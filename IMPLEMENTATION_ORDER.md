# Recommended Implementation Order

This file defines a practical implementation sequence for the architecture.

It does **not** replace any architecture document.

## Principle

Stabilize shared contracts first, then build components in parallel where their boundaries are already defined.

---

# Phase 0 — Repository and Shared Foundations

Create the implementation repository/package structure.

Recommended initial packages:

```text
packages/
  contracts/
  runtime-core/
  agents/
  assignments/
  capabilities/
  context/
  policies/
  model-routing/
  evaluation/
  observability/
  testing/
```

Establish:

- canonical ID types
- shared domain contracts
- runtime event envelope
- test clock
- fake queue
- fake model provider
- fake capability/provider adapters
- error taxonomy

**Gate:** individual components must not invent incompatible duplicate domain contracts.

---

# Phase 1 — Parallel Foundations

These can begin in parallel once shared contracts stabilize.

## Track A — Agent Registry

Implement:

- Agent Definition
- version resolution
- lifecycle
- validation
- delegation/capability eligibility metadata

Architecture:
`01-agent-definition-and-agent-registry.md`

## Track B — Capability Layer

Implement:

- Capability Registry
- Capability Gateway
- provider adapter interfaces
- normalized results
- idempotency primitives
- reconciliation contracts

Architecture:
`04-capability-and-tool-architecture.md`

## Track C — Policy and Approval Layer

Implement:

- Policy Engine
- effective authority
- approval records
- payload binding
- revocation
- pre-write authorization

Architecture:
`07-policies-approvals-and-authority.md`

## Track D — Model Routing

Implement:

- model catalog
- model classes
- deterministic router
- provider/model health
- fallback/escalation
- budgets/usage

Architecture:
`08-model-routing-and-execution-budgets.md`

## Track E — Evaluation and Evidence Foundation

Implement:

- Evidence Store
- evidence references
- deterministic validators
- evaluation results
- completion-proof primitives

Architecture:
`09-evaluation-and-evidence.md`

---

# Phase 2 — Work Graph and Execution Core

Implement the durable execution backbone.

Order:

1. Work Item
2. Work Graph + versions
3. Work Nodes + Edges
4. readiness calculation
5. scheduler
6. worker claim/lease
7. dispatch references
8. waiting/resumption
9. retries
10. cancellation
11. supersession
12. dependency lineage

Architecture:
`06-work-graph-and-execution-model.md`

**Integration gate:** Work Graph must execute fake deterministic capability nodes without any model.

---

# Phase 3 — Assignment Runtime

Implement:

- Assignment Manager
- assignment validation
- Agent Instance
- Agent Execution
- immutable execution contracts
- revisions
- blocked states
- result submission
- evaluation integration

Architecture:
`03-assignment-protocol.md`

**Integration gate:** a fake specialist can be scheduled through the Work Graph and produce an accepted result.

---

# Phase 4 — Context Distribution

Implement:

- Chief of Staff context packages
- specialist context packages
- local conversational reference resolution
- active Work Item summaries
- provenance
- freshness
- context budgeting
- immutable package versions
- expansion

Architecture:
`05-context-distribution-system.md`

---

# Phase 5 — Chief of Staff

Implement structured Chief of Staff decisions.

Initial useful decision modes:

```text
respond
capability read
deterministic action proposal
delegate
request user input
request approval
wait
modify work
cancel work
complete-work proposal
```

Architecture:
`02-chief-of-staff-architecture.md`

Avoid adding a mandatory router or planner model before the Chief of Staff.

---

# Phase 6 — Initial Specialist Agents

Recommended order:

1. Recipient Resolver
2. Email Composer
3. Revenue Analyst
4. Schedule Analyst

Optional later:
- Research Specialist

Architecture:
`10-initial-agent-set.md`

---

# Phase 7 — Vertical Integration

Implement thin end-to-end flows instead of waiting for every subsystem to be feature-complete.

## Slice 1 — Calendar Read

```text
message
→ Chief of Staff
→ calendar capability
→ response
```

Validates:

- inbound work
- context
- model routing
- capability reads
- response path

## Slice 2 — Resolve + Draft

```text
message
→ recipient resolution
→ email composition
→ draft result
```

Validates:

- Assignment Protocol
- Registry
- Context Distribution
- accepted result flow

## Slice 3 — Revenue Report Send

```text
message
→ recipient resolution
→ revenue analysis
→ join
→ email composition
→ approval
→ deterministic send
→ evidence
→ completion
```

Validates almost the full architecture.

## Slice 4 — Correction

While July work is active:

```text
Actually June.
```

Expected:

- recipient result reused
- July-dependent work superseded
- June analysis created
- stale draft/approval invalidated

## Slice 5 — Cancellation

After approval but before send:

```text
Don't send it.
```

Expected:

- delayed/queued send cannot execute
- runtime rechecks canonical state

## Slice 6 — Ambiguous Provider Write

Provider completes a write but runtime times out.

Expected:

- ambiguous state
- reconciliation
- no blind duplicate retry
- evidence recovered where possible

---

# Merge Gates

## Gate 1 — Shared semantics

No duplicate definitions of core concepts.

## Gate 2 — Deterministic runtime

Work Graph functions without a model.

## Gate 3 — Specialist execution

One assignment can run end-to-end.

## Gate 4 — Policy

Unauthorized write cannot execute.

## Gate 5 — Approval

Required approval is payload-bound and durable.

## Gate 6 — Evidence

External write cannot complete parent objective without proof.

## Gate 7 — Correction

Corrections invalidate affected descendants only.

## Gate 8 — Recovery

Waiting and in-progress state survive process restart.

---

# Parallel Team Guidance

A practical split might be:

```text
Engineer A
Agent Registry + initial agent definitions

Engineer B
Capabilities + provider adapters

Engineer C
Work Graph + scheduler + workers

Engineer D
Policy + approvals + evidence

Engineer/Codex track
Context + model routing
```

Assignment ownership can change.

Shared contracts and architecture cannot drift independently.
