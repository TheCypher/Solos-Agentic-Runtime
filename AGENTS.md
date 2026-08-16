# AGENTS.md

This file defines the implementation rules for Codex and any AI coding agent working in this repository.

## Source of Truth

The full architecture documents in:

```text
docs/architecture/
```

are authoritative.

Do not summarize them into a smaller replacement specification and implement from that replacement.

Read the relevant document in full.

## Mission

Implement the Solos Agent Runtime exactly according to the architecture, one bounded component at a time, while preserving compatibility across the whole runtime.

## Clean-Slate Requirement

This is a new implementation.

Do not copy or reuse implementation code, database schemas, agent loops, prompts, abstractions, workflows, tests, or integration architecture from the previous Solos runtime.

Historical systems may be consulted only to understand requirements or failure cases.

## Non-Negotiable Architectural Invariants

### Runtime ownership

The runtime owns:

- durable state
- Work Items and Work Graphs
- execution
- retries
- scheduling
- approvals
- policy enforcement
- cancellation
- idempotency
- evidence
- completion

### Chief of Staff ownership

The Chief of Staff owns:

- user understanding
- objective formation
- delegation
- coordination
- synthesis
- communication

### Specialist ownership

Specialists own only the bounded reasoning responsibility expressed by an assignment.

They do not own:

- parent Work Items
- user communication
- canonical state
- unrestricted tools
- completion

## Implementation Rules

### 1. Read before coding

Before modifying a component, read:

- its full architecture document
- the architecture documents of its direct dependencies
- this file
- `CONTRIBUTING.md`

### 2. Do not invent duplicate domain concepts

If a shared concept already exists, use it.

Examples:

- Work Item
- Work Graph
- Work Node
- Agent Definition
- Assignment
- Agent Instance
- Agent Execution
- Capability
- Approval
- Evidence
- Context Package

If the implementation requires changing shared semantics, raise an architecture change rather than silently creating a competing abstraction.

### 3. Models do not enforce guarantees

Do not rely on prompts to enforce:

- permission
- approval
- idempotency
- retry limits
- delegation depth
- evidence
- completion
- state-machine legality

These must be enforced by runtime code.

### 4. Do not call provider APIs from agents

Agents interact with Solos-owned capabilities.

Provider-specific implementation belongs in adapters.

### 5. Do not put provider credentials in model context

Never expose:

- API keys
- OAuth access tokens
- refresh tokens
- database secrets
- webhook secrets

to models.

### 6. Writes require evidence

A consequential external write is not complete because:

- the model asked for it
- a job was queued
- a provider request was attempted
- an agent returned `completed`

Use the capability-specific evidence requirements from the architecture.

### 7. Preserve bounded execution

Do not add:

- infinite model loops
- unrestricted agent delegation
- mandatory router models
- mandatory planner models
- mandatory evaluator models
- autonomous agent creation

unless the architecture is explicitly changed.

### 8. Preserve cancellation and supersession

Delayed jobs must reload current state before executing.

A queued write must not execute after its Work Item has been cancelled or superseded.

### 9. Build deterministic tests

Core orchestration must be testable with fake:

- models
- capabilities
- providers
- queues
- clocks

### 10. Implement failure paths

A component is incomplete unless it handles its documented:

- failure
- retry
- cancellation
- supersession
- timeout
- recovery

behavior.

## Definition of Done for a Component

A component is complete only when:

- its public contract matches the architecture
- its ownership boundaries are respected
- happy path passes
- documented failure paths pass
- cancellation/supersession works
- retries are bounded
- relevant state is durable
- required events/telemetry exist
- tests cover acceptance behavior
- downstream components can consume it without hidden architecture changes

## Pull Requests

A PR should implement one coherent architectural unit.

If implementation requires changing architecture, state that explicitly in the PR and link the corresponding architecture-change issue.
