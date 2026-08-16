# Agent Definition and Agent Registry

## Purpose

The Agent Registry defines what agents are allowed to exist inside Solos.

It is the authoritative catalog of every approved agent role, its responsibilities, its authority, its inputs, its outputs, and the conditions under which it may operate.

Agents are not created by writing a prompt and attaching tools.

Every agent must exist as a versioned, governed definition inside the registry.

The registry ensures that the runtime can answer:

- What is this agent responsible for?
- What work may it accept?
- What work must it refuse?
- What information may it receive?
- What capabilities may it use?
- May it delegate work?
- Which model should execute it?
- How much time and compute may it consume?
- What output must it return?
- How is its result evaluated?
- Which version produced a result?

The Agent Registry is part of the runtime control plane.

It is not merely a prompt library.

---

# Foundational Distinction

The system must distinguish among three concepts:

```text
Agent Definition
Agent Instance
Agent Execution
```

## Agent Definition

A durable, versioned description of an approved role.

Example:

```text
Revenue Analyst
Version 1.2.0
```

The definition describes what the role is, what it may do, and how it must behave.

## Agent Instance

A temporary participation of an Agent Definition in one specific assignment.

An instance belongs to:

```text
one Work Item
one Assignment
one exact Agent Definition version
```

It is not a permanent autonomous process.

## Agent Execution

One computational attempt to execute an Agent Instance.

An execution records:

```text
attempt number
model/provider
context package
usage
result
failure
timing
```

A single Agent Instance may have multiple executions because of retry, fallback, or revision.

---

# What an Agent Is

An agent is a bounded reasoning role operating inside the Solos Runtime.

An agent consists of:

```text
Identity
+ responsibility
+ accepted assignments
+ prohibited work
+ context policy
+ capability eligibility
+ delegation policy
+ model policy
+ execution policy
+ output contract
+ evaluation policy
+ version
```

An agent is not simply:

```text
name
+ prompt
+ tools
```

---

# Agent Definition

A conceptual Agent Definition might look like this:

```typescript
type AgentDefinition = {
  id: string;
  agentType: string;
  version: string;

  displayName: string;
  description: string;

  purpose: string;
  responsibilities: string[];
  prohibitedResponsibilities: string[];

  acceptedAssignmentTypes: AssignmentTypeDefinition[];

  inputContract: SchemaReference;
  outputContract: SchemaReference;

  contextPolicy: AgentContextPolicy;
  capabilityPolicy: AgentCapabilityPolicy;
  delegationPolicy: AgentDelegationPolicy;

  modelPolicy: AgentModelPolicy;
  executionPolicy: AgentExecutionPolicy;
  evaluationPolicy: AgentEvaluationPolicy;

  instructions: AgentInstructionSet;

  status:
    | "draft"
    | "testing"
    | "active"
    | "deprecated"
    | "disabled";

  createdAt: string;
  activatedAt?: string;
  deprecatedAt?: string;
};
```

---

# Agent Identity

Every agent has a stable `agentType`.

Examples:

```text
chief_of_staff
recipient_resolver
email_composer
revenue_analyst
schedule_analyst
```

The agent type identifies the responsibility.

The version identifies the exact behavioral contract.

Example:

```text
revenue_analyst@1.2.0
```

Every execution must record the exact version that ran.

This gives us reproducibility and auditability.

---

# Versioning

An agent definition should use semantic versioning conceptually.

```text
Patch
Prompt tuning, model fallback adjustment, internal non-contract changes.

Minor
Backward-compatible behavior or capability improvements.

Major
Responsibility, authority, assignment contract, or output-contract changes.
```

A running assignment must remain bound to the exact definition version selected when it was created.

The registry must never silently switch a running assignment to a different version.

---

# Purpose

Every agent definition should contain a concise purpose.

Example:

```text
Revenue Analyst

Analyze normalized business revenue data and return
supported explanations of revenue performance.
```

The purpose should answer:

> Why does this role exist?

If the purpose is too broad to describe clearly, the agent is probably too broad.

---

# Responsibilities

An agent definition must explicitly state what the role owns.

Example:

```text
Revenue Analyst responsibilities:

- calculate and compare revenue metrics
- identify meaningful changes
- identify anomalies
- explain supported findings
- distinguish facts from hypotheses
```

Responsibilities should describe reasoning ownership.

---

# Prohibited Responsibilities

Every specialist must also state what it must not do.

Example:

```text
Revenue Analyst prohibited responsibilities:

- issue refunds
- mutate financial transactions
- send financial reports externally
- invent missing figures
- communicate directly with the user
- mark the parent Work Item complete
```

Negative boundaries are first-class architecture.

An agent without explicit prohibited responsibilities is too loosely defined.

---

# Accepted Assignment Types

Agents do not accept arbitrary natural-language delegation.

They accept registered assignment types.

Example Revenue Analyst assignment types:

```text
compare_revenue_periods
identify_revenue_anomalies
summarize_revenue_performance
```

The runtime validates:

```text
Assignment Type
      ↓
Accepted by Agent Definition?
      ↓
yes / no
```

before an instance can be created.

---

# Input and Output Contracts

Every agent needs structured contracts.

Natural-language instructions may accompany the contract.

They do not replace it.

Example result envelope:

```typescript
type AgentResult<TOutput> = {
  assignmentId: string;
  agentInstanceId: string;
  executionId: string;

  agentType: string;
  agentVersion: string;

  status:
    | "completed"
    | "blocked"
    | "needs_input"
    | "failed";

  output?: TOutput;

  findings: Finding[];
  proposedActions: ProposedAction[];
  evidence: EvidenceReference[];
  unresolvedQuestions: AgentQuestion[];

  capabilityCalls: CapabilityCallReference[];

  usage: AgentUsage;

  completedAt: string;
};
```

Structured output gives the runtime something it can validate.

---

# Context Policy

An Agent Definition declares what kinds of context the role is eligible to receive.

```typescript
type AgentContextPolicy = {
  allowedContextTypes: (
    | "assignment"
    | "work_item_summary"
    | "selected_messages"
    | "business_profile"
    | "selected_memories"
    | "external_records"
    | "policy_documents"
    | "prior_agent_results"
  )[];

  maxConversationMessages: number;
  maxMemoryItems: number;

  allowFullConversation: boolean;
  allowSensitiveData: boolean;

  requiredContextTypes: string[];
};
```

The Context Builder enforces this policy.

The Chief of Staff cannot override data-access restrictions simply by putting more information into an assignment.

---

# Capability Policy

The Agent Definition defines capability eligibility.

The assignment defines the actual capability grant.

Effective permissions are the intersection:

```text
Agent Definition eligibility
∩ Assignment grants
∩ User authorization
∩ Connected account permissions
∩ Runtime policy
```

No layer can expand permissions beyond the layer above it.

Example:

```text
Revenue Analyst definition may be eligible for:

revenue.read_summary
revenue.read_transactions
customers.read_aggregate
```

A specific assignment might grant only:

```text
revenue.read_summary
```

The agent receives only that operation.

---

# Delegation Policy

Delegation must be explicit.

```typescript
type AgentDelegationPolicy = {
  mayDelegate: boolean;
  allowedAgentTypes: string[];
  maxDepth: number;
  maxAssignmentsPerExecution: number;
};
```

Initial policy:

```text
Chief of Staff:
may delegate
max depth 1

Specialists:
may not delegate
```

If a specialist discovers that another specialist would be useful, it returns a proposed dependency or blocked result.

The Chief of Staff/runtime decides whether to create the separate assignment.

---

# Model Policy

Agent identity is not tied to one model.

```typescript
type AgentModelPolicy = {
  preferredModelClass:
    | "fast"
    | "balanced"
    | "frontier";

  allowedProviders: string[];
  allowedModels?: string[];

  fallbackModelClasses: string[];

  requiresStructuredOutput: boolean;
  requiresToolCalling: boolean;
  requiresVision: boolean;

  escalationPolicy?: {
    onLowConfidence: boolean;
    onInvalidOutput: boolean;
    onHighComplexity: boolean;
    maxEscalations: number;
  };
};
```

The Model Router chooses the actual provider/model at runtime.

---

# Execution Policy

Each definition has a bounded execution policy.

```typescript
type AgentExecutionPolicy = {
  maxModelCalls: number;
  maxCapabilityCalls: number;
  maxExecutionTimeMs: number;

  allowParallelCapabilityCalls: boolean;
  maxParallelCapabilityCalls: number;

  retryPolicy: RetryPolicy;
  budgetPolicy: BudgetPolicy;

  requiresHumanApprovalBeforeExecution: boolean;
};
```

An agent may not increase its own execution budget.

---

# Evaluation Policy

Every agent declares how its result must be evaluated.

```typescript
type AgentEvaluationPolicy = {
  schemaValidation: "required";

  evidenceValidation:
    | "required"
    | "optional"
    | "none";

  minimumEvidenceCount?: number;
  confidenceThreshold?: number;

  evaluator:
    | "deterministic"
    | "chief_of_staff"
    | "specialist_evaluator"
    | "human";

  rejectUnsupportedClaims: boolean;
};
```

The result is not automatically trusted simply because the agent returned `completed`.

---

# Instructions

The Agent Definition contains versioned instructions describing:

```text
role identity
operating behavior
reasoning responsibility
failure behavior
constraints
output expectations
```

Prompts are not security.

If instructions say:

```text
Do not send email.
```

the runtime should also withhold `email.send`.

The architecture must remain safe even when a model does not follow its prompt.

---

# Agent Registry Responsibilities

The Agent Registry must:

```text
register definitions
validate definitions
version definitions
activate versions
deprecate versions
disable versions
resolve exact versions
resolve active versions
validate assignment compatibility
expose allowed delegation targets
provide definitions to execution
retain historical versions
```

---

# Registry Interface

Conceptually:

```typescript
interface AgentRegistry {
  register(
    definition: AgentDefinition
  ): Promise<RegisteredAgentDefinition>;

  get(
    agentType: string,
    version: string
  ): Promise<AgentDefinition | null>;

  getActive(
    agentType: string
  ): Promise<AgentDefinition | null>;

  listActive(): Promise<AgentDefinition[]>;

  validateAssignment(
    agentType: string,
    assignment: AgentAssignment
  ): Promise<AssignmentValidationResult>;

  activate(
    agentType: string,
    version: string
  ): Promise<void>;

  deprecate(
    agentType: string,
    version: string
  ): Promise<void>;

  disable(
    agentType: string,
    version: string
  ): Promise<void>;
}
```

---

# Definition Lifecycle

```text
draft
→ testing
→ active
→ deprecated
→ disabled
```

## Draft

Definition is being developed.

No production assignments.

## Testing

Definition may execute in controlled evaluation environments.

## Active

Definition may receive new production assignments.

## Deprecated

Existing historical/running assignments remain valid.

No new assignments should use the version.

## Disabled

Execution is prohibited.

Used for emergency shutdown or serious defects.

---

# Registry Storage

I would initially keep agent definitions source-controlled.

Example:

```text
packages/
  agents/
    chief-of-staff/
      v1.0.0.ts
      instructions.md
      schemas.ts
      evaluations.ts

    revenue-analyst/
      v1.0.0.ts
      instructions.md
      schemas.ts
      evaluations.ts
```

Runtime activation metadata may live in PostgreSQL.

I would not begin with a production admin UI that edits arbitrary agent prompts.

Agent behavior is production code.

It should receive the same review discipline as production code.

---

# Agent Instantiation Flow

The Chief of Staff does not start a model directly.

It proposes an assignment.

The Assignment Manager performs:

```text
1. Resolve requested agent type.

2. Load active Agent Definition.

3. Validate requesting agent's delegation rights.

4. Validate target agent.

5. Validate assignment type.

6. Validate assignment input.

7. Intersect capability grants with policy.

8. Apply execution budget.

9. Bind exact Agent Definition version.

10. Create durable Agent Instance.

11. Create Agent Execution.

12. Dispatch execution.
```

---

# Agent Instance

Conceptually:

```typescript
type AgentInstance = {
  id: string;

  workItemId: string;
  assignmentId: string;

  agentType: string;
  agentDefinitionVersion: string;

  status:
    | "created"
    | "queued"
    | "running"
    | "waiting"
    | "completed"
    | "failed"
    | "cancelled";

  capabilityGrants: CapabilityGrant[];
  executionBudget: ExecutionBudget;

  createdBy:
    | "chief_of_staff"
    | "workflow"
    | "runtime";

  createdAt: string;
  completedAt?: string;
};
```

An Agent Instance is not an always-running process.

It is a durable record of one agent role participating in one assignment.

---

# Agent Execution

```typescript
type AgentExecution = {
  id: string;

  agentInstanceId: string;
  attemptNumber: number;

  modelProvider: string;
  modelId: string;

  status:
    | "queued"
    | "running"
    | "succeeded"
    | "failed"
    | "timed_out"
    | "cancelled";

  contextPackageId: string;

  startedAt?: string;
  completedAt?: string;

  usage?: AgentUsage;

  error?: AgentExecutionError;
};
```

Retries or fallbacks create new execution attempts rather than overwriting history.

---

# Execution Flow

```text
1. Agent Instance becomes ready.

2. Executor loads exact Agent Definition.

3. Context Builder constructs context.

4. Policy Engine calculates effective grants.

5. Model Router selects model.

6. Agent Execution record created.

7. Model receives:
   - instructions
   - assignment
   - bounded context
   - granted capability descriptions
   - output schema
   - execution limits

8. Agent reasons and may request capabilities.

9. Every capability call passes runtime authorization.

10. Capability returns structured result + evidence.

11. Agent returns structured AgentResult.

12. Result Validator checks:
    - schema
    - evidence
    - policy
    - budget
    - completion requirements

13. Result persisted.

14. Parent Work Graph updated.

15. Chief of Staff receives accepted result.

16. Agent Instance becomes completed, blocked, or failed.
```

---

# Agent Selection Catalog

The Chief of Staff should not receive every full Agent Definition.

It should receive a concise Delegation Catalog.

```typescript
type DelegationCatalogEntry = {
  agentType: string;
  purpose: string;

  acceptedAssignmentTypes: string[];

  requiredInputs: string[];
  expectedOutputs: string[];

  limitations: string[];
};
```

Example:

```text
Recipient Resolver
Resolve person/contact references.
Returns verified candidates.
Read-only.
Does not send messages.

Revenue Analyst
Analyze normalized revenue data.
Returns supported metrics/findings.
Read-only.
Does not mutate financial data.

Email Composer
Creates structured email drafts.
Cannot send.
```

This gives the Chief of Staff enough information to delegate without filling its context with every specialist's implementation details.

---

# Registry Validation Rules

A definition should be rejected if it has:

```text
no output schema

unrestricted capabilities

unlimited delegation depth

no prohibited responsibilities

write authority without approval policy

no execution budget

parent Work Item completion ownership

provider-specific raw API behavior embedded directly into the role

direct user communication without explicit policy

no evaluation policy
```

---

# Emergency Controls

The registry must support operational controls:

```text
disable one agent version

disable an entire agent type

prevent new assignments

remove capability eligibility

force model policy

allow existing assignments to finish

cancel existing assignments where necessary

select fallback definition
```

This is necessary for safe production operation.

---

# Initial Agent Definitions

The registry should remain small initially.

## Chief of Staff

Purpose:

```text
Own the user objective, coordinate work, delegate bounded assignments,
review accepted results, and communicate with the user.
```

May delegate.

May not bypass policy.

May not fabricate completion.

## Recipient Resolver

Purpose:

```text
Resolve ambiguous people/contact references using permitted read-only sources.
```

No writes.

No direct user communication.

## Email Composer

Purpose:

```text
Produce a structured email draft using accepted facts and a resolved recipient.
```

Cannot send.

## Revenue Analyst

Purpose:

```text
Produce supported analysis of normalized business revenue data.
```

Read-only.

## Schedule Analyst

Purpose:

```text
Interpret schedule, conflicts, and available windows.
```

Read-only initially.

Consequential calendar actions remain deterministic capabilities.

---

# Example: Recipient Resolver Definition

Conceptually:

```typescript
const recipientResolver: AgentDefinition = {
  id: "recipient_resolver",
  agentType: "recipient_resolver",
  version: "1.0.0",

  displayName: "Recipient Resolver",

  description:
    "Resolves ambiguous references to people and communication destinations.",

  purpose:
    "Return verified recipient candidates for a bounded reference.",

  responsibilities: [
    "resolve person references",
    "search permitted contact sources",
    "compare candidate evidence",
    "surface unresolved ambiguity"
  ],

  prohibitedResponsibilities: [
    "send communication",
    "modify contacts",
    "guess when evidence is insufficient",
    "communicate with the user",
    "mark Work Items complete"
  ],

  acceptedAssignmentTypes: [
    "resolve_recipient"
  ],

  inputContract: "ResolveRecipientInput",
  outputContract: "RecipientResolutionResult",

  contextPolicy: {
    allowedContextTypes: [
      "assignment",
      "selected_messages",
      "selected_memories",
      "external_records"
    ],

    maxConversationMessages: 8,
    maxMemoryItems: 8,

    allowFullConversation: false,
    allowSensitiveData: true,

    requiredContextTypes: [
      "assignment"
    ]
  },

  capabilityPolicy: {
    eligibleCapabilities: [
      "contacts.search",
      "contacts.read",
      "email.search_correspondents"
    ],

    defaultAccess: "read_only"
  },

  delegationPolicy: {
    mayDelegate: false,
    allowedAgentTypes: [],
    maxDepth: 0,
    maxAssignmentsPerExecution: 0
  },

  modelPolicy: {
    preferredModelClass: "fast",
    allowedProviders: [],
    fallbackModelClasses: ["balanced"],
    requiresStructuredOutput: true,
    requiresToolCalling: true,
    requiresVision: false,

    escalationPolicy: {
      onLowConfidence: true,
      onInvalidOutput: true,
      onHighComplexity: false,
      maxEscalations: 1
    }
  },

  executionPolicy: {
    maxModelCalls: 1,
    maxCapabilityCalls: 4,
    maxExecutionTimeMs: 15000,

    allowParallelCapabilityCalls: true,
    maxParallelCapabilityCalls: 2,

    retryPolicy: {},
    budgetPolicy: {},

    requiresHumanApprovalBeforeExecution: false
  },

  evaluationPolicy: {
    schemaValidation: "required",
    evidenceValidation: "required",
    minimumEvidenceCount: 1,
    evaluator: "deterministic",
    rejectUnsupportedClaims: true
  },

  instructions: {
    instructionSetId: "recipient_resolver_v1"
  },

  status: "active",

  createdAt: "..."
};
```

The exact schema will evolve during implementation.

The important point is that responsibility, authority, context, execution, and evaluation all exist independently from prompt wording.

---

# Database Concepts

Likely durable concepts include:

```text
agent_definitions
agent_definition_versions
agent_definition_status

agent_assignments

agent_instances
agent_executions
agent_results

agent_capability_grants
agent_delegation_events
agent_evaluation_results
agent_usage_events
```

The precise tables can be refined during schema design.

---

# Observability

For every execution, the runtime should be able to answer:

```text
Why was this agent selected?

Which Agent Definition version executed?

What assignment did it receive?

What context was included?

What context was excluded?

Which capabilities were granted?

Which capabilities were called?

Which model executed it?

How many tokens/model calls were used?

What result did it return?

What evidence supported that result?

What validations ran?

How did the result affect the Work Item?
```

---

# Testing

Every Agent Definition should ship with an evaluation suite.

Tests should cover:

```text
valid assignment

invalid assignment

missing context

ambiguous input

capability denial

unsupported action request

malformed output

unsupported claims

low confidence

budget exhaustion

provider failure

prompt injection in retrieved content

attempt to exceed responsibility

attempt to communicate with user

attempt to mark Work Item complete
```

Before activating a new version, compare it with the currently active version.

---

# What We Are Not Building

We are not building:

```text
an unrestricted prompt marketplace

a Chief of Staff that invents arbitrary autonomous agents

an agent with every tool

prompt-defined security

agent names as architecture

agents owning durable state

specialists deciding the user's objective is complete
```

---

# The Standard

A valid Solos agent must be:

```text
Necessary
Bounded
Versioned
Typed
Permissioned
Observable
Evaluated
Replaceable
```

If a proposed agent cannot satisfy those requirements, it should not exist.

---

# One Sentence

> **The Agent Registry is the authoritative control plane that defines which reasoning roles may exist inside Solos and the exact conditions under which they may participate in user work.**
