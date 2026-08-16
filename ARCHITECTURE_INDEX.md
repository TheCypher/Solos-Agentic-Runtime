# Architecture Index

The documents below are the complete architecture specifications for the Solos Agent Runtime.

They are intended to be read as one system.

## 01 — Agent Definition and Agent Registry

**File:** `docs/architecture/01-agent-definition-and-agent-registry.md`

Defines:

- Agent Definition
- Agent Instance
- Agent Execution
- agent versioning
- registered assignment types
- capability eligibility
- delegation policy
- model policy
- evaluation policy
- registry lifecycle and emergency controls

Primary dependencies:
- shared runtime/domain contracts

Primary consumers:
- Assignment Protocol
- Chief of Staff
- Agent Executor

---

## 02 — Chief of Staff Architecture

**File:** `docs/architecture/02-chief-of-staff-architecture.md`

Defines:

- the principal intelligence
- objective ownership
- decision modes
- delegation
- planning
- Work Graph proposals
- approval decisions
- progress/user communication
- proactive behavior
- bounded reasoning loop

Depends heavily on:
- Work State
- Agent Registry
- Context Distribution
- Capability catalog
- Policies

---

## 03 — Assignment Protocol

**File:** `docs/architecture/03-assignment-protocol.md`

Defines:

- durable delegation contracts
- assignment lifecycle
- assignment revisions
- context references
- capability grants
- completion criteria
- evidence requirements
- blocked/revision/cancellation/supersession behavior
- result contracts

Depends on:
- Agent Registry
- Policies
- Context
- Model Routing
- Evaluation

---

## 04 — Capability and Tool Architecture

**File:** `docs/architecture/04-capability-and-tool-architecture.md`

Defines:

- Capability
- Operation
- Provider Adapter
- Provider Tool
- Capability Registry
- Capability Gateway
- authorization boundary
- normalized provider results
- idempotency
- reconciliation
- evidence production

Depends on:
- Policies
- Evidence
- provider integrations

---

## 05 — Context Distribution System

**File:** `docs/architecture/05-context-distribution-system.md`

Defines:

- context consumers
- context requests
- context policies
- provenance
- conversation windows
- memory/context distinctions
- freshness
- context budgets
- compression
- trust boundaries
- context expansion/versioning

Depends on:
- Work State
- memories
- accepted agent results
- external records
- policy/resource scope

---

## 06 — Work Graph and Execution Model

**File:** `docs/architecture/06-work-graph-and-execution-model.md`

Defines:

- Work Item
- Work Graph
- Work Nodes/Edges
- scheduler
- claims/leases
- parallelism
- waiting/resumption
- corrections
- graph versioning
- cancellation
- retries
- recovery
- completion

This is the durable execution backbone.

---

## 07 — Policies, Approvals, and Authority

**File:** `docs/architecture/07-policies-approvals-and-authority.md`

Defines:

- intent vs permission vs authority vs approval
- effective authority
- risk classes
- autonomy
- approval binding
- standing authority
- policy precedence
- pre-write checks
- revocation and kill switches

Consumed by:
- Assignment Protocol
- Capability Gateway
- Work Graph readiness
- external write execution

---

## 08 — Model Routing and Execution Budgets

**File:** `docs/architecture/08-model-routing-and-execution-budgets.md`

Defines:

- Fast / Balanced / Frontier model classes
- deterministic routing
- quality floors
- fallback vs retry vs escalation
- hierarchical budgets
- latency/cost/token accounting
- model health/circuit breaking

Consumed by:
- Chief of Staff
- Agent Executor
- model evaluators

---

## 09 — Evaluation and Evidence

**File:** `docs/architecture/09-evaluation-and-evidence.md`

Defines:

- Output vs Claim vs Evidence vs Evaluation vs Completion
- deterministic validation
- claim-to-evidence mapping
- fact/inference/hypothesis distinction
- accepted results
- external action evidence
- completion proof
- evaluation versioning

Consumed by:
- Assignment Protocol
- Work Graph
- Context Distribution
- Chief of Staff

---

## 10 — Initial Agent Set

**File:** `docs/architecture/10-initial-agent-set.md`

Defines the initial production organization:

- Chief of Staff
- Recipient Resolver
- Email Composer
- Revenue Analyst
- Schedule Analyst
- optional Research Specialist

Also explicitly defines which things should **not** become agents.

---

# Cross-Cutting Invariants

These rules cross every document:

```text
Runtime owns execution.
Chief of Staff owns outcome.
Specialists own bounded reasoning.

Agents do not own durable state.
Specialists do not delegate in v1.
Writes happen through deterministic capabilities.
Policies are enforced outside prompts.
Context is assembled, not inherited wholesale.
Work is durable and resumable.
Corrections preserve unaffected work.
Completion requires evidence.
Models are bounded resources.
```
