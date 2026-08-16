# Contributing to the Solos Agent Runtime

## Development Philosophy

This repository is architecture-first.

The goal is not to maximize code output. The goal is to implement a runtime whose components remain coherent when developed independently.

## Before Starting Work

1. Read `README.md`.
2. Read `AGENTS.md`.
3. Read the component's full architecture document.
4. Read the documents for direct dependencies.
5. Create or claim an implementation issue.
6. Identify whether the work changes a shared architectural contract.

## Branches

Recommended branch naming:

```text
feat/agent-registry
feat/work-graph
feat/context-builder
fix/assignment-cancellation
test/email-send-reconciliation
docs/architecture-clarification
```

Do not push implementation work directly to `main`.

## Pull Request Scope

Prefer one coherent component or behavior per PR.

Good:

```text
Implement Agent Registry lifecycle and version resolution
```

Bad:

```text
Implement registry + rewrite context + change Work Item states + refactor provider layer
```

Small, coherent PRs make parallel architecture work safer.

## Architecture Changes

Implementation will occasionally reveal that an architecture contract needs revision.

Do not hide that revision inside code.

Open an Architecture Change issue when changing:

- component ownership
- shared domain semantics
- lifecycle states
- dependency direction
- capability authority
- evidence requirements
- completion semantics
- delegation policy
- context trust boundaries

Document:

1. Current architecture.
2. Problem discovered.
3. Proposed change.
4. Components affected.
5. Migration/compatibility implications.

## Testing Expectations

Every component should test:

- expected behavior
- invalid input
- failure
- retry
- cancellation
- supersession where relevant
- process recovery where relevant
- authorization denial where relevant
- evidence requirements where relevant

Core runtime tests should avoid real provider dependencies.

## Provider Integration Tests

Provider adapters may use dedicated integration tests, but domain correctness should not depend on live providers.

## Shared Contracts

Shared types should live in a canonical package once implementation begins.

Do not define private copies of shared contracts inside multiple packages.

## Database Changes

Schema changes should:

- preserve ownership boundaries
- use explicit migrations
- avoid provider-specific concepts in domain tables where possible
- preserve auditability for important state transitions

## Observability

New execution behavior should be traceable through stable identifiers such as:

- Work Item ID
- Work Graph ID
- Work Node ID
- Assignment ID
- Agent Instance ID
- Agent Execution ID
- Capability Request ID
- Approval ID
- Evidence ID

## Review Questions

Reviewers should ask:

- Does this match the architecture?
- Does it introduce a hidden second source of truth?
- Does a model own something the runtime should own?
- Can retries duplicate a side effect?
- Can a delayed job execute after cancellation?
- Is completion actually evidenced?
- Are permissions enforced in code?
- Does this make parallel implementation harder?
