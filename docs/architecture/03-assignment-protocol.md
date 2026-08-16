# Assignment Protocol

## Purpose

The Assignment Protocol defines how work is delegated from the Chief of Staff to specialist agents.

It governs the full lifecycle of delegated reasoning:

```text
propose
→ validate
→ create
→ schedule
→ execute
→ return
→ evaluate
→ accept
```

It also governs exceptional states:

```text
blocked
revised
cancelled
superseded
failed
expired
```

The Assignment Protocol exists because delegation must never be an informal conversation between models.

The Chief of Staff does not simply tell another agent:

> “Handle this.”

It creates a bounded responsibility with explicit inputs, authority, expected outputs, completion criteria, and execution limits.

The runtime governs that responsibility until it is resolved.

---

# Foundational Principle

> **An assignment is a contract, not a prompt.**

Natural language may describe the objective.

But the runtime requires a structured contract around that objective.

An assignment must answer:

- What needs to be accomplished?
- Which Work Item does it serve?
- Which agent role may perform it?
- What information is available?
- What capabilities may be used?
- What must not be done?
- What result must be returned?
- What evidence is required?
- What dependencies exist?
- What is the execution budget?
- What constitutes success?
- What constitutes being blocked?
- Who created the assignment?
- Can it be revised?
- Can it be cancelled?

---

# Assignment Ownership

Assignments have three distinct ownership relationships.

```text
Work Item
    ↓
Chief of Staff owns outcome
    ↓
Assignment
    ↓
Specialist owns bounded responsibility
```

The specialist owns the assignment while it is executing.

It does not own the parent Work Item.

The Chief of Staff remains responsible for the user's broader objective.

The runtime remains responsible for durable state and execution legality.

---

# Assignment Definition

A conceptual assignment contract might look like:

```typescript
type AgentAssignment<TInput = unknown, TOutput = unknown> = {
  id: string;

  workItemId: string;
  workStepId?: string;

  agentType: string;
  agentVersion?: string;

  assignmentType: string;

  objective: string;

  input: TInput;
  expectedOutput: SchemaReference<TOutput>;

  contextReferences: ContextReference[];

  capabilityGrants: CapabilityGrant[];
  constraints: AssignmentConstraint[];

  completionCriteria: CompletionCriterion[];
  evidenceRequirements: EvidenceRequirement[];

  dependencies: AssignmentDependency[];

  executionBudget: ExecutionBudget;

  priority: WorkPriority;

  createdBy: AssignmentCreator;
  createdAt: string;

  expiresAt?: string;
};
```

This is the durable contract.

The prompt given to the specialist is generated from this contract and its Agent Definition.

---

# Assignment Type

Every assignment belongs to a registered assignment type.

Examples:

```text
resolve_recipient
compare_revenue_periods
draft_email
analyze_schedule
research_question
review_communication
categorize_customers
```

The assignment type defines its semantic purpose.

Example:

```typescript
type ResolveRecipientAssignment = {
  type: "resolve_recipient";

  reference: string;

  candidateSources: Array<
    | "contacts"
    | "email"
    | "calendar"
    | "memory"
  >;

  requiredConfidence: number;
};
```

The runtime validates that:

```text
requested assignment type
        ↓
is accepted by
        ↓
registered agent definition
```

The model does not decide whether this relationship is legal.

---

# Assignment Objective

Every assignment needs one bounded objective.

Good:

> Identify which Martin the user intends to email and return verified recipient candidates.

Bad:

> Help with Martin.

Good:

> Compare July revenue with June and identify the three largest supported changes.

Bad:

> Analyze the business.

Good:

> Draft a friendly email explaining the July revenue results using the supplied analysis.

Bad:

> Handle the email.

A useful test is:

> Could we objectively determine whether the specialist fulfilled this assignment?

If not, the assignment is too vague.

---

# Assignment Input

Inputs should be typed whenever practical.

Example:

```typescript
type DraftEmailInput = {
  objective: string;

  recipient: {
    name: string;
    email: string;
    evidenceReference: string;
  };

  relevantFacts: FactReference[];

  communicationContext: MessageReference[];

  constraints: {
    tone?: string;
    maxLength?: number;
    requiredPoints?: string[];
    prohibitedClaims?: string[];
  };
};
```

The runtime should avoid giving the specialist a giant unstructured context blob and asking it to infer everything.

Structure reduces ambiguity.

---

# Context References

Assignments should reference context rather than duplicating canonical state.

Example:

```typescript
type ContextReference =
  | MessageReference
  | MemoryReference
  | WorkItemReference
  | ExternalRecordReference
  | AgentResultReference
  | DocumentReference
  | BusinessContextReference;
```

Example assignment:

```text
Objective:
Analyze July revenue.

References:
revenue_dataset_492
business_profile_18
memory_service_pricing_29
```

The Context Builder resolves those references into the bounded context package used during execution.

This prevents stale copies of important information from spreading throughout the system.

---

# Constraints

Assignments should explicitly include relevant constraints.

```typescript
type AssignmentConstraint =
  | DoNotEstimateConstraint
  | ReadOnlyConstraint
  | NoExternalCommunicationConstraint
  | DateRangeConstraint
  | ResourceScopeConstraint
  | UserPreferenceConstraint
  | PolicyConstraint
  | OutputLengthConstraint
  | ConfidentialityConstraint;
```

Examples:

```text
Do not estimate missing revenue.

Do not send communication.

Use transactions only between July 1 and July 31.

Do not access unrelated customer records.

Do not treat memory as verified external evidence.
```

Constraints exist independently of prompts.

Runtime enforcement should be used wherever technically possible.

---

# Capability Grants

Every assignment carries its effective capability grants.

Example:

```typescript
type CapabilityGrant = {
  capability: string;

  operations: string[];

  access: "read" | "write";

  resourceScope?: ResourceScope;

  accountIds?: string[];

  approvalPolicy?: ApprovalPolicy;

  expiresAt?: string;
};
```

Example:

```text
Revenue Analyst assignment

Granted:
revenue.read_summary
revenue.read_transactions

Not granted:
payments.refund
email.send
customers.update
```

The specialist cannot request capabilities outside the assignment grant.

Even if its Agent Definition would normally permit them.

---

# Effective Authority

Actual authority should be calculated through intersection.

```text
Agent Definition eligibility
        ∩
Assignment grants
        ∩
User authorization
        ∩
Connected-provider permissions
        ∩
Business policy
        ∩
Runtime policy
        =
Effective authority
```

Every layer can reduce authority.

No layer can expand authority beyond the layers above it.

---

# Completion Criteria

Assignments need explicit completion criteria.

Example:

```typescript
type CompletionCriterion = {
  id: string;
  description: string;

  validation:
    | "schema"
    | "evidence"
    | "deterministic"
    | "chief_of_staff"
    | "human";
};
```

Recipient Resolver:

```text
1. At least one candidate returned.

2. Every candidate has verified contact evidence.

3. Confidence score provided.

4. Ambiguity explicitly identified when threshold is not met.
```

Revenue Analyst:

```text
1. Requested periods analyzed.

2. Required metrics calculated.

3. Every numerical claim references evidence.

4. Missing data is explicitly identified.

5. Output validates against schema.
```

The specialist may believe it is finished.

The runtime determines whether the criteria are satisfied.

---

# Evidence Requirements

Some assignments require evidence.

```typescript
type EvidenceRequirement = {
  claimType: string;
  minimumEvidence: number;
  acceptedEvidenceTypes: string[];
};
```

Example:

```text
Recipient identity:
Verified contact record or verified correspondence.

Revenue number:
Normalized transaction or revenue record.

Calendar availability:
Calendar event or availability result.
```

Evidence should reference persisted artifacts.

Not model statements.

---

# Assignment Dependencies

Assignments may depend on other work.

```typescript
type AssignmentDependency = {
  dependsOnId: string;

  condition:
    | "completed"
    | "accepted"
    | "specific_output_available";

  requiredOutputPath?: string;
};
```

Example:

```text
Recipient Resolver ─────┐
                       ├──→ Email Composer
Revenue Analyst ────────┘
```

Email Composer should not execute until both required inputs exist.

The dependency graph belongs to the runtime.

The specialist should not poll other agents.

---

# Assignment Lifecycle

I would define the assignment lifecycle explicitly:

```text
proposed
→ validated
→ ready
→ queued
→ running
→ result_submitted
→ evaluating
→ accepted
```

Exceptional states:

```text
blocked
revision_required
failed
cancelled
superseded
expired
rejected
```

---

# Proposed

The Chief of Staff or workflow has proposed the assignment.

Nothing has executed.

Example:

```text
agentType: revenue_analyst
assignmentType: compare_revenue_periods
```

The runtime must validate it.

---

# Validated

The runtime confirms:

- agent exists
- version is eligible
- assignment type is supported
- input contract validates
- permissions are valid
- delegation policy allows creation
- execution budget is legal
- dependencies are valid

If validation fails, the assignment never becomes executable.

---

# Ready

All required dependencies are satisfied.

The assignment may now execute.

---

# Queued

Execution has been dispatched.

The queue is responsible only for delivery.

The assignment remains canonical in PostgreSQL.

---

# Running

An Agent Instance has claimed the assignment and an Agent Execution is active.

The runtime records:

```text
agentInstanceId
executionId
attemptNumber
model
start time
lease
budget
```

---

# Result Submitted

The specialist has returned its structured result.

This does not mean the assignment succeeded.

The result still requires evaluation.

---

# Evaluating

The runtime performs required checks:

```text
schema validation
evidence validation
policy validation
capability-call validation
budget validation
completion criteria
specialized evaluation
```

---

# Accepted

The assignment contract has been satisfied.

Its output may now be used by dependent work.

Accepted does not mean the parent Work Item is complete.

---

# Blocked

The specialist cannot continue without something external.

Examples:

```text
missing required user information

missing authorization

unavailable external record

unresolved ambiguity

required upstream assignment incomplete
```

Blocked result:

```typescript
type BlockedAgentResult = {
  status: "blocked";

  reasonCode: string;
  reason: string;

  requiredResolution: ResolutionRequirement[];

  completedPartialWork?: unknown;
};
```

The runtime persists the partial result where useful.

The Chief of Staff decides how to resolve the blocker.

---

# Needs User Input

This should be a specific blocked condition.

Example:

```text
Two verified Martins exist and there is insufficient evidence
to identify which one the user means.
```

The specialist should not message the user directly.

It returns:

```typescript
{
  status: "blocked",
  reasonCode: "AMBIGUOUS_RECIPIENT",
  requiredResolution: [{
    type: "user_input",
    questionIntent: "choose_recipient",
    options: [...]
  }]
}
```

The Chief of Staff decides how to phrase the question.

---

# Revision Required

Sometimes an assignment result is usable but does not fully satisfy the contract.

Example:

```text
Email draft contains unsupported revenue claim.
```

The evaluator may return:

```typescript
type RevisionRequest = {
  assignmentId: string;

  issues: EvaluationIssue[];

  retainedEvidence: EvidenceReference[];
  revisionBudget: ExecutionBudget;
};
```

The assignment can be executed again with targeted feedback.

This is different from starting a brand-new assignment.

---

# Failure

Failure means the assignment could not fulfill its responsibility.

Examples:

```text
model repeatedly failed structured output

required provider unavailable

budget exhausted

evaluation repeatedly rejected result

internal execution error
```

A failure should include a typed failure class.

```typescript
type AssignmentFailure =
  | "MODEL_FAILURE"
  | "CAPABILITY_FAILURE"
  | "POLICY_FAILURE"
  | "BUDGET_EXHAUSTED"
  | "INVALID_OUTPUT"
  | "EVALUATION_FAILURE"
  | "DEPENDENCY_FAILURE"
  | "INTERNAL_FAILURE";
```

The Chief of Staff decides whether to:

- retry
- revise
- substitute
- reduce scope
- proceed partially
- ask the user
- fail the parent Work Item

---

# Cancellation

Assignments must be cancellable.

Cancellation may originate from:

```text
user
Chief of Staff
workflow
runtime policy
parent Work Item cancellation
superseding correction
```

Example:

```text
User:
Send Martin the July report.

Then:
Never mind, don't send anything.
```

Any incomplete email-related assignments should be cancelled where appropriate.

A cancellation request must propagate through the Work Graph.

---

# Superseded

Superseded is different from cancelled.

It means newer work replaces the assignment.

Example:

```text
Analyze July revenue.
```

User corrects:

```text
Actually, June.
```

The July assignment becomes:

```text
superseded
```

The historical assignment remains visible.

A newly created June assignment becomes authoritative.

This gives the runtime a complete causal history.

---

# Assignment Revision

Not every correction should create an entirely unrelated assignment.

A revision may create a new assignment revision:

```typescript
type AssignmentRevision = {
  assignmentId: string;
  revisionNumber: number;

  changedFields: string[];

  reason:
    | "user_correction"
    | "evaluation_feedback"
    | "updated_dependency"
    | "chief_of_staff_refinement";

  createdAt: string;
};
```

For auditability, previously executed inputs should not be silently mutated.

The runtime should preserve the earlier version and record the revision.

---

# Assignment Immutability

Once an assignment begins execution, its execution contract should be effectively immutable.

If material changes occur:

```text
objective changes
recipient changes
date range changes
permissions change
output requirements change
```

create a revision or successor assignment.

Do not mutate the contract underneath a running specialist.

This prevents race conditions and impossible debugging.

---

# Agent Acceptance

A specialist should be able to reject an assignment when it does not fit its registered responsibility.

Example:

```typescript
type AssignmentAcceptance =
  | {
      accepted: true;
    }
  | {
      accepted: false;
      reasonCode:
        | "UNSUPPORTED_ASSIGNMENT"
        | "MISSING_REQUIRED_INPUT"
        | "INSUFFICIENT_AUTHORITY"
        | "INVALID_SCOPE";
    };
```

Most validation should happen before execution.

But the agent execution layer should still fail safely when reality differs from the contract.

---

# Specialist Output

A common result envelope:

```typescript
type AgentResult<TOutput = unknown> = {
  assignmentId: string;
  agentInstanceId: string;
  executionId: string;

  status:
    | "completed"
    | "blocked"
    | "failed";

  output?: TOutput;

  findings: Finding[];

  evidence: EvidenceReference[];

  proposedActions: ProposedAction[];

  unresolvedQuestions: AgentQuestion[];

  capabilityCalls: CapabilityCallReference[];

  usage: AgentUsage;
};
```

Specialists may propose actions.

They should not silently perform consequential actions unless the assignment explicitly grants them and runtime policy permits it.

---

# Proposed Actions

A specialist may discover that additional work would be useful.

Example Revenue Analyst:

> Revenue declined because appointment volume fell. It may be useful to inspect cancellations.

The specialist should not create new assignments itself in the initial architecture.

It returns:

```typescript
type ProposedAction = {
  type:
    | "create_assignment"
    | "call_capability"
    | "request_user_input"
    | "request_approval";

  reason: string;

  proposal: unknown;
};
```

The Chief of Staff evaluates the proposal.

This preserves delegation depth of one.

---

# Communication Isolation

Specialists should not communicate directly with the user.

They may return suggested communication:

```typescript
{
  proposedUserMessage:
    "I found two contacts named Martin..."
}
```

But only the Chief of Staff communicates.

This ensures:

- consistent voice
- coherent context
- no conflicting explanations
- no accidental exposure of internal architecture
- centralized judgment

---

# Execution Budget

Each assignment has its own budget.

```typescript
type ExecutionBudget = {
  maxModelCalls: number;

  maxCapabilityCalls: number;

  maxParallelCapabilityCalls: number;

  maxInputTokens?: number;
  maxOutputTokens?: number;

  maxCostUsd?: number;

  maxExecutionTimeMs: number;

  maxRetries: number;
};
```

The specialist may not increase its own budget.

If the budget becomes insufficient, it returns a blocked or failed state.

The Chief of Staff may propose a larger follow-up assignment where justified.

---

# Timeouts

Different concepts need separate timeouts.

```text
Model call timeout

Capability call timeout

Agent execution timeout

Assignment deadline

External waiting deadline
```

A model timing out should not automatically cause the entire assignment to expire.

A waiting-for-user assignment may remain valid for days.

The runtime should model these independently.

---

# Retries

Agent retries should be intentional.

Safe retry examples:

```text
model provider temporary failure
structured output parse failure
read-only capability timeout
```

Unsafe retry examples:

```text
provider may have completed a write

email may already have sent

payment may already have processed
```

Capability-level write behavior must use reconciliation and idempotency.

The Assignment Protocol should never blindly retry consequential actions.

---

# Result Evaluation

Evaluation should happen after every result submission.

Possible validators:

```text
Schema Validator
Evidence Validator
Policy Validator
Capability Validator
Deterministic Evaluator
Specialist Evaluator
Chief of Staff Review
Human Review
```

Evaluation produces:

```typescript
type AssignmentEvaluation = {
  assignmentId: string;

  decision:
    | "accept"
    | "revise"
    | "reject"
    | "escalate";

  issues: EvaluationIssue[];

  evidenceValidated: EvidenceReference[];

  evaluatedAt: string;
};
```

---

# Deterministic Evaluation

Where possible, results should be checked without another model.

Examples:

```text
Does the email exist?

Does the recipient match verified contact data?

Do revenue calculations recompute?

Does every finding have evidence?

Did the requested date range match?

Does output conform to schema?
```

Model-based evaluation should be used for subjective qualities such as:

```text
tone
clarity
strategic quality
sensitivity
reasoning completeness
```

Even then, deterministic checks should remain separate.

---

# Assignment Results Become Inputs

An accepted result becomes a durable artifact that other work may reference.

Example:

```text
assignment:
revenue_analysis_44

accepted result:
revenue_report_82
```

Then:

```text
Email Composer
context reference:
revenue_report_82
```

The Email Composer should not need to rerun the revenue analysis.

This prevents repeated work and inconsistent conclusions.

---

# Assignment Dependency Graph

The runtime should explicitly represent assignment dependencies.

Example:

```text
               Resolve Recipient
                       │
                       ▼
Revenue Analysis → Email Composer
                       │
                       ▼
                 Approval Step
                       │
                       ▼
                  Send Email
```

A dependency may represent:

```text
must complete first

may run in parallel

needs output field

blocked until approval

cancel if parent is superseded
```

This eventually becomes part of the larger Work Graph architecture.

---

# Parallel Assignments

Assignments may execute concurrently when independent.

Example:

```text
           Chief of Staff
           /            \
          /              \
Recipient Resolver    Revenue Analyst
          \              /
           \            /
            Email Composer
```

The runtime controls parallelism.

The Chief of Staff proposes logical independence.

Specialists should never independently synchronize with each other.

They communicate through accepted runtime results.

---

# Assignment Priority

Assignments should have explicit priority.

```typescript
type WorkPriority =
  | "background"
  | "normal"
  | "high"
  | "urgent";
```

Priority affects scheduling.

It should not override:

```text
security
approval
policy
dependency
resource ownership
```

“Urgent” does not mean “skip safety checks.”

---

# Assignment Expiration

Some assignments become invalid over time.

Example:

```text
Find availability for tomorrow afternoon.
```

If tomorrow has passed, the assignment should not execute later.

Assignments may therefore include:

```typescript
{
  expiresAt: "..."
}
```

When expired:

```text
assignment → expired
```

The Chief of Staff reevaluates the parent objective if necessary.

---

# Parent Work Changes

Assignments exist within Work Items.

If the parent Work Item changes materially, the runtime should determine which assignments remain valid.

Example:

```text
Parent:
Prepare and send July report.

User changes:
Don't send it. Just show me the numbers.
```

Potential effects:

```text
Revenue Analyst:
still valid

Recipient Resolver:
no longer necessary

Email Composer:
cancel

Send operation:
cancel
```

The Chief of Staff proposes the revised objective.

The runtime reconciles the Work Graph.

---

# Assignment Events

Every important transition should create an event.

Examples:

```text
assignment.proposed

assignment.validated

assignment.ready

assignment.queued

assignment.started

assignment.capability_called

assignment.result_submitted

assignment.blocked

assignment.revised

assignment.evaluation_failed

assignment.accepted

assignment.cancelled

assignment.superseded

assignment.failed
```

These events support:

- debugging
- observability
- audit history
- analytics
- latency analysis
- reliability measurement

---

# Database Concepts

The precise schema comes later, but we will likely need:

```text
agent_assignments

agent_assignment_revisions

agent_assignment_dependencies

agent_assignment_constraints

agent_assignment_capability_grants

agent_instances

agent_executions

agent_results

agent_result_evidence

agent_evaluations

agent_assignment_events
```

Important relationships:

```text
Work Item
  ↓
Assignment
  ↓
Agent Instance
  ↓
Agent Execution
  ↓
Agent Result
  ↓
Evaluation
```

---

# Example: Recipient Resolution

User:

> Send Martin the report.

Chief of Staff proposes:

```typescript
{
  agentType: "recipient_resolver",

  assignmentType: "resolve_recipient",

  objective:
    "Identify which Martin the user intends to receive the report.",

  input: {
    reference: "Martin"
  },

  capabilityGrants: [
    "contacts.search",
    "email.search_correspondents"
  ],

  completionCriteria: [
    "candidate identity verified",
    "email address supported by evidence",
    "ambiguity explicitly identified"
  ]
}
```

Result:

```typescript
{
  status: "completed",

  output: {
    candidates: [{
      name: "Martin ...",
      email: "...",
      confidence: 0.97
    }]
  },

  evidence: [...]
}
```

Runtime evaluates.

Assignment becomes:

```text
accepted
```

The result becomes available to downstream work.

---

# Example: Ambiguous Recipient

The same assignment returns:

```typescript
{
  status: "blocked",

  output: {
    candidates: [
      {
        name: "Martin A",
        confidence: 0.53
      },
      {
        name: "Martin B",
        confidence: 0.47
      }
    ]
  },

  unresolvedQuestions: [{
    type: "recipient_selection"
  }]
}
```

The Recipient Resolver does not guess.

It does not text the user.

The Chief of Staff asks:

> I found two Martins. Do you mean Martin from Solos or Martin from the client account?

When the user responds, the Work Item resumes.

---

# Example: User Correction

Current assignment:

```text
Revenue Analyst:
Analyze July revenue.
```

User:

> Actually June, not July.

Runtime flow:

```text
1. Incoming message relates to existing Work Item.

2. Chief of Staff identifies a correction.

3. July assignment is marked superseded.

4. Running execution receives cancellation where possible.

5. June assignment is created.

6. Any downstream work depending on July output is invalidated.

7. Independent assignments remain intact.
```

Nothing unrelated disappears.

---

# Example: Evaluation Failure

Email Composer returns:

> Revenue increased 24%.

But revenue evidence supports only 14%.

Evaluation:

```typescript
{
  decision: "revise",

  issues: [{
    code: "UNSUPPORTED_NUMERIC_CLAIM",
    path: "body",
    expected: "14%",
    received: "24%"
  }]
}
```

The runtime creates a revision execution.

The Chief of Staff does not see the draft as trustworthy until the result is accepted.

---

# Example: Cancellation

User:

> Never mind, don't send anything.

Current graph:

```text
Revenue Analyst       accepted
Recipient Resolver    accepted
Email Composer        running
Approval              pending
Send Email            waiting
```

Runtime may transition:

```text
Email Composer        cancelled
Approval              cancelled
Send Email            cancelled
```

Previously accepted read-only analysis remains in historical state.

Nothing is sent.

---

# Initial Constraints

For the first version, I would establish:

```text
Maximum delegation depth:
1

Specialists can create assignments:
No

Specialist direct user communication:
No

Assignment mutation after execution begins:
No

Consequential write permissions for specialists:
Generally no

Maximum parallel specialist assignments:
3 per Chief of Staff decision

Every assignment:
Must have a registered type

Every assignment:
Must have an output contract

Every assignment:
Must have an execution budget

Every accepted factual result:
Must satisfy its evidence policy

Every assignment transition:
Must be persisted
```

This gives us a strict system initially.

We can loosen individual rules only when real product requirements justify it.

---

# What We Are Not Building

We are not sending arbitrary prompts between agents.

We are not letting agents silently invent their own responsibilities.

We are not allowing specialists to spawn unbounded subagents.

We are not using model conversation as durable task state.

We are not mutating running assignments invisibly.

We are not treating agent output as automatically correct.

We are not allowing specialists to communicate around the Chief of Staff.

We are building a durable delegation protocol.

---

# The Standard

Every assignment must be:

```text
Bounded
Typed
Authorized
Contextualized
Budgeted
Traceable
Evaluated
Cancellable
Versioned
```

An assignment that cannot satisfy those properties should not execute.

---

# One Sentence

> **The Assignment Protocol converts delegation from an informal model interaction into a durable, typed, governed contract between the Chief of Staff, specialist intelligence, and the Solos Runtime.**
