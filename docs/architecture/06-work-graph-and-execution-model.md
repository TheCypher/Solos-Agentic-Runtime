# Work Graph and Execution Model

## Purpose

The Work Graph and Execution Model defines how Solos represents and executes the work required to fulfill a user objective.

It connects everything we have designed so far:

- Chief of Staff decisions
- specialist assignments
- deterministic capability operations
- workflows
- approvals
- user-input waits
- verification
- retries
- cancellations
- scheduling
- evidence
- completion

The Work Graph answers:

> **What must happen, in what order, under what conditions, before this objective can be considered complete?**

The runtime owns this graph.

Agents may propose work.

Agents may contribute results.

Agents do not own execution order or durable workflow state.

---

# Foundational Principle

> **The Work Item owns the objective. The Work Graph owns the path to completion.**

Example:

```text
Work Item:
Send Martin the July revenue report.
```

The Work Graph may contain:

```text
Resolve Martin ──────────────┐
                             ▼
Analyze July Revenue ──→ Compose Email
                             │
                             ▼
                      Request Approval
                             │
                             ▼
                         Send Email
                             │
                             ▼
                      Verify Evidence
```

The Work Item says **what outcome Solos accepted responsibility for**.

The Work Graph says **what must happen to accomplish it**.

---

# Work Item vs Work Graph

These concepts should remain separate.

## Work Item

Represents a user-level responsibility.

```typescript
type WorkItem = {
  id: string;

  objective: WorkObjective;

  status: WorkItemStatus;

  priority: WorkPriority;

  createdBy: WorkItemCreator;

  completionCriteria: CompletionCriterion[];

  currentGraphVersion: number;

  createdAt: string;
  completedAt?: string;
};
```

Examples:

```text
Send Martin the July report.

Find me a time to meet Sarah next week.

Prepare my morning business brief.

Figure out why revenue dropped last month.
```

---

## Work Graph

Represents the executable structure supporting that objective.

```typescript
type WorkGraph = {
  id: string;

  workItemId: string;

  version: number;

  nodes: WorkNode[];
  edges: WorkEdge[];

  status:
    | "draft"
    | "active"
    | "completed"
    | "superseded"
    | "cancelled";

  createdAt: string;
};
```

A Work Item may have multiple graph versions over its lifetime because the user's objective may evolve.

---

# The Work Graph Is Durable

The Work Graph must exist outside any model context.

If:

```text
worker crashes

server deploys

model provider fails

queue redelivers

user disappears for two days

approval arrives tomorrow
```

the runtime must still know exactly:

- what has completed
- what is currently running
- what is blocked
- what is waiting
- what failed
- what should happen next
- what remains before completion

The graph is canonical runtime state.

---

# Work Node Types

The graph should use a small set of explicit node types.

```typescript
type WorkNode =
  | AgentAssignmentNode
  | CapabilityOperationNode
  | WorkflowNode
  | ApprovalNode
  | UserInputNode
  | WaitNode
  | VerificationNode
  | DecisionNode
  | JoinNode;
```

We should resist creating dozens of node types initially.

Each node represents a distinct execution responsibility.

---

# Agent Assignment Node

Used when bounded reasoning is required.

```typescript
type AgentAssignmentNode = {
  type: "agent_assignment";

  id: string;

  assignmentId: string;

  agentType: string;

  status: NodeStatus;
};
```

Example:

```text
Analyze July Revenue
→ Revenue Analyst
```

The node completes only after the Assignment Protocol accepts the specialist result.

---

# Capability Operation Node

Used for a known deterministic operation.

```typescript
type CapabilityOperationNode = {
  type: "capability_operation";

  id: string;

  capability: string;
  operation: string;

  inputReference: string;

  approvalRequirement?: string;

  evidenceRequirements: EvidenceRequirement[];

  status: NodeStatus;
};
```

Examples:

```text
email.send

calendar.create_event

email.create_draft

customers.read
```

The node completes when the capability result satisfies its success and evidence policy.

---

# Workflow Node

A Workflow Node invokes a registered reusable workflow.

Example:

```text
Run Morning Business Brief
```

The workflow may itself expand into an internal graph.

Conceptually:

```text
Parent Work Graph
       │
       ▼
Workflow Node
       │
       ▼
Child Work Graph
```

The parent does not need every internal workflow detail flattened into its own graph.

---

# Approval Node

Represents a durable approval requirement.

```typescript
type ApprovalNode = {
  type: "approval";

  id: string;

  approvalRequestId: string;

  status:
    | "waiting"
    | "approved"
    | "denied"
    | "expired"
    | "cancelled";
};
```

The graph pauses at this branch until the approval resolves.

No worker remains running.

---

# User Input Node

Represents required information from the user.

Example:

```text
Which Martin do you mean?
```

```typescript
type UserInputNode = {
  type: "user_input";

  id: string;

  questionIntent: string;

  expectedResponseContract?: SchemaReference;

  resolutionState:
    | "waiting"
    | "resolved"
    | "cancelled"
    | "expired";
};
```

The Chief of Staff controls the communication.

The Work Graph controls the waiting state.

---

# Wait Node

Used when work depends on time or an external condition.

Examples:

```text
wait until Monday at 8 AM

wait for provider callback

wait until invoice is overdue

wait until scheduled follow-up date
```

```typescript
type WaitNode = {
  type: "wait";

  id: string;

  condition: WaitCondition;

  status:
    | "waiting"
    | "satisfied"
    | "expired"
    | "cancelled";
};
```

The runtime should persist the condition rather than keeping a process asleep.

---

# Verification Node

Used when completion requires explicit verification beyond an operation response.

Examples:

```text
Verify email exists.

Verify calendar event was created.

Verify recipient still matches approval.

Recompute revenue calculation.

Reconcile ambiguous provider result.
```

Verification should be deterministic where possible.

---

# Decision Node

Used when the Chief of Staff must make another judgment after new information becomes available.

Example:

```text
Revenue Analyst returns:
revenue fell 23%

Schedule Analyst returns:
three appointment cancellations occurred

→ Chief of Staff decides what follow-up is required
```

A Decision Node represents:

```text
runtime must invoke Chief of Staff again
```

It does not represent an open-ended agent loop.

---

# Join Node

Used to synchronize parallel branches.

Example:

```text
Recipient Resolver ────┐
                       │
Revenue Analyst ───────┼──→ Join → Email Composer
                       │
Business Context ──────┘
```

The Join Node may require:

```text
all dependencies

any dependency

minimum successful count

specific required outputs
```

Example:

```typescript
type JoinPolicy =
  | {
      type: "all";
    }
  | {
      type: "any";
    }
  | {
      type: "minimum";
      count: number;
    };
```

---

# Work Edges

Edges define relationships between nodes.

```typescript
type WorkEdge = {
  fromNodeId: string;
  toNodeId: string;

  condition:
    | "on_success"
    | "on_failure"
    | "on_approved"
    | "on_denied"
    | "on_resolved"
    | "on_timeout"
    | "always";
};
```

The graph should represent execution semantics explicitly.

Not through hidden agent assumptions.

---

# Node Lifecycle

A common node lifecycle might be:

```text
pending
→ ready
→ queued
→ running
→ completed
```

Exceptional states:

```text
blocked
waiting
failed
cancelled
superseded
expired
skipped
```

Not every node uses every state.

---

# Pending

The node exists but one or more dependencies are unsatisfied.

Example:

```text
Email Composer:
pending

Reason:
Revenue analysis not yet accepted.
```

---

# Ready

All required dependencies and preconditions are satisfied.

The node is eligible for execution.

---

# Queued

The runtime has dispatched the node for processing.

The queue does not own node state.

PostgreSQL does.

---

# Running

A worker has claimed the node and execution is active.

---

# Waiting

The node is intentionally inactive until an external condition resolves.

Examples:

```text
approval

user input

time

external callback
```

Waiting is not failure.

---

# Completed

The node's completion criteria are satisfied.

For consequential operations, this generally requires evidence.

---

# Failed

The node cannot complete under its current execution strategy.

The runtime must determine whether to:

```text
retry

revise

replace

branch to recovery

ask the Chief of Staff

fail the parent objective
```

---

# Superseded

A newer version of the work has replaced this node.

Example:

```text
Analyze July Revenue
```

becomes superseded after:

```text
Actually June.
```

Historical state remains intact.

---

# Skipped

A node may become unnecessary because another branch resolved the objective.

Example:

```text
Search Contacts
Search Email History
```

If verified contact resolution succeeds through Contacts and the graph uses an `any` policy, Email History may be skipped.

---

# Graph Construction

The Chief of Staff may propose the logical structure of a Work Graph.

Example:

```typescript
type ProposedWorkGraph = {
  nodes: ProposedWorkNode[];
  dependencies: ProposedWorkDependency[];
};
```

The runtime then validates:

- node types
- agent registrations
- capability permissions
- dependencies
- graph cycles
- execution budgets
- approval requirements
- maximum parallelism
- delegation depth
- Work Item compatibility

The Chief of Staff proposes.

The runtime materializes.

---

# Graph Validation

Before activation, the runtime should reject graphs that contain:

```text
unknown agents

unknown capabilities

illegal dependency cycles

unbounded branches

unauthorized writes

missing approval nodes

impossible completion criteria

invalid output references

delegation beyond permitted depth

nodes without execution budgets
```

Where a cycle is genuinely needed, it should be modeled as an explicit bounded retry or revision mechanism—not an arbitrary graph cycle.

---

# No Unbounded Cycles

The initial Work Graph should be a directed acyclic graph wherever possible.

Instead of:

```text
Agent A
→ Agent B
→ Agent A
→ Agent B
→ ...
```

use:

```text
Assignment
→ Evaluation
→ Revision Attempt 1
→ Evaluation
→ Revision Attempt 2
→ fail/escalate
```

Loops should have:

```text
explicit reason

maximum count

budget

exit condition
```

---

# Scheduler

The Scheduler determines which nodes become executable.

Conceptually:

```text
Work Graph changes
      ↓
Scheduler evaluates graph
      ↓
Find ready nodes
      ↓
Apply priority + concurrency policy
      ↓
Dispatch execution
```

The Scheduler should not reason about business meaning.

It evaluates runtime conditions.

---

# Ready-Node Evaluation

A node becomes ready when:

```text
dependencies satisfied

node not cancelled

node not superseded

required inputs available

required authority valid

required approval already satisfied where applicable

execution budget available

Work Item still active

time constraints still valid
```

These conditions should be evaluated transactionally where practical.

---

# Worker Execution

Workers execute bounded nodes.

A worker should:

```text
1. Receive node reference.

2. Load canonical node state.

3. Attempt to claim node.

4. Confirm it is still executable.

5. Load required execution contract.

6. Execute bounded responsibility.

7. Persist result/evidence.

8. Transition node state.

9. Emit runtime event.

10. Trigger scheduler reevaluation.
```

The worker should not decide what the entire Work Item should do next.

---

# Claims and Leases

Workers need explicit ownership.

```typescript
type NodeLease = {
  nodeId: string;

  claimedBy: string;

  claimedAt: string;

  expiresAt: string;

  attemptNumber: number;
};
```

Only one valid worker should own a node execution at a time.

If the worker crashes:

```text
lease expires
→ node becomes reclaimable
```

Claims reduce duplicate concurrent execution.

They do not replace operation idempotency.

---

# Queue Semantics

Queue messages should contain references.

Example:

```typescript
type ExecuteNodeMessage = {
  workItemId: string;
  workGraphId: string;
  nodeId: string;
  dispatchId: string;
};
```

Do not place full mutable workflow state in the queue payload.

Worker:

```text
receives reference
→ loads current state from PostgreSQL
```

This prevents stale queue payloads from becoming authoritative.

---

# Parallelism

Independent nodes should execute concurrently when useful.

Example:

```text
               ┌→ Recipient Resolver ─┐
Chief of Staff ┤                      ├→ Email Composer
               └→ Revenue Analyst ────┘
```

Parallelism reduces latency.

It should not increase ambiguity.

---

# Parallelism Rules

Parallel execution is appropriate when branches:

```text
do not mutate the same external resource

do not depend on unshared implicit decisions

can produce independently verifiable results

have explicitly declared dependencies
```

Parallel execution is dangerous when:

```text
two branches may send communication

two branches may update the same appointment

two branches may make incompatible decisions

one branch relies on unstated assumptions from another
```

The principle remains:

> **Parallelize intelligence. Coordinate consequential action.**

---

# Concurrency Limits

The runtime should impose limits such as:

```text
maximum parallel nodes per Work Item

maximum parallel specialist assignments

maximum provider concurrency

maximum writes against one resource

per-user execution limits

global provider limits
```

Parallelism should be controlled infrastructure, not model enthusiasm.

---

# Resource Locks

Some operations may require resource-level serialization.

Example:

```text
appointment_42
```

Two active operations should not simultaneously:

```text
reschedule appointment_42

cancel appointment_42
```

A logical resource lock or transactional precondition may be required.

Conceptually:

```typescript
type ResourceClaim = {
  resourceType: string;
  resourceId: string;

  operationType: string;

  workItemId: string;

  expiresAt: string;
};
```

Use only where the operation truly requires serialization.

---

# Sequential Execution

Some operations must happen in order.

Example:

```text
Resolve Recipient
→ Compose Draft
→ Approval
→ Send
→ Verify
```

A downstream node should receive accepted upstream outputs through durable references.

It should not re-derive them.

---

# Join Semantics

When parallel work rejoins, the runtime should explicitly define what is required.

Example:

```text
Revenue Analysis:
completed

Recipient Resolution:
blocked
```

Email Composer cannot become ready if both are required.

The Join Node remains pending.

The Chief of Staff may be invoked to resolve the blocker.

---

# Partial Success

Not every parallel branch must always succeed.

Example:

```text
Prepare business health summary

Revenue Analyst        completed
Schedule Analyst       completed
Customer Analyst       failed
```

The graph policy may say:

```text
all three required
```

or:

```text
minimum two required
```

or:

```text
Revenue required, others optional
```

This must be defined explicitly.

---

# Waiting

Waiting should be a first-class execution state.

Example:

```text
Compose Draft
→ Approval
→ Send
```

While approval is pending:

```text
Work Item:
waiting_for_approval

Approval Node:
waiting

Send Node:
pending
```

No worker remains alive.

No repeated model calls are required.

---

# Resumption

When the awaited event arrives:

```text
approval

user response

scheduled time

external callback
```

the runtime:

```text
1. Persists the triggering event.

2. Resolves the matching waiting node.

3. Updates node state.

4. Revalidates dependent state.

5. Rebuilds fresh context where necessary.

6. Invokes Scheduler.

7. Continues eligible work.
```

Resumption should be event-driven.

---

# User Input Matching

A user response may satisfy a waiting User Input Node.

Example:

```text
Chief of Staff:
Do you mean Martin Smith or Martin Jones?

User:
Smith.
```

The runtime must associate:

```text
"Smith"
```

with the unresolved Work Item.

It should not treat the message as an unrelated new request.

Once resolved:

```text
User Input Node → completed
Recipient resolution continues
```

---

# Approval Matching

Approvals should reference exact action state.

When an approval arrives:

```text
approvalId
approved payload hash
related Work Item
related node
```

The runtime confirms that the proposed action has not materially changed.

If it has:

```text
approval invalid
→ new approval required
```

---

# Scheduled Work

A Work Graph may begin or resume at a future time.

Example:

```text
Morning Brief
→ every weekday at 8 AM
```

The schedule should create a runtime trigger.

The runtime then creates or resumes a Work Item.

Scheduling is not equivalent to running an agent continuously.

---

# Time-Based Nodes

Examples:

```text
wait until invoice is 3 days overdue

follow up tomorrow morning

check again after provider maintenance window
```

Time conditions should be persisted and queryable.

---

# Corrections

User corrections may change the Work Graph.

Example:

```text
Send Martin the July report.

Actually June.
```

Current graph:

```text
Resolve Martin      completed
Analyze July        running
Compose Email       pending
Approval            pending
Send                pending
```

The runtime should determine impact.

New graph:

```text
Resolve Martin      reuse completed result
Analyze July        superseded
Analyze June        ready
Compose Email       replaced
Approval            invalidated if already created
Send                still pending behind new graph
```

Only affected branches should change.

---

# Dependency Lineage

Every derived artifact should record what it depends on.

Example:

```text
July date range
      ↓
Revenue Analysis
      ↓
Draft
      ↓
Approval
```

If the root input changes:

```text
July → June
```

the runtime can identify every descendant that is no longer valid.

This is how corrections propagate safely.

---

# Graph Versioning

Material changes to work should create a new Work Graph version.

Example:

```text
Work Graph v1
Send July report.
```

User corrects:

```text
Work Graph v2
Send June report.
```

Version 1 remains historically visible.

Version 2 becomes active.

This gives us:

```text
auditability

causal history

debugging

replay

correction tracking
```

---

# Reuse Across Graph Versions

Not every completed node should be repeated.

The runtime may reuse an accepted output when:

```text
its inputs remain valid

its evidence remains valid

its context remains relevant

its expiration has not passed

its dependencies were unaffected
```

Example:

```text
Resolved Martin
```

may remain valid when:

```text
July → June
```

The Revenue Analysis cannot.

---

# Cancellation

A Work Item must be cancellable.

Example:

```text
Never mind. Don't send it.
```

The runtime should:

```text
mark Work Item cancellation requested

cancel queued nodes

attempt to cancel running nodes

cancel waiting approvals

prevent future nodes from becoming ready

preserve completed historical work

prevent new external writes
```

Already completed external actions cannot be magically undone.

If reversal is supported, reversal should be separate explicit work.

---

# Cancellation Propagation

The graph should know which descendants must be cancelled.

Example:

```text
Draft
→ Approval
→ Send
```

If Draft is cancelled before completion:

```text
Approval → skipped/cancelled

Send → cancelled
```

Independent branches may or may not be cancelled depending on parent objective.

---

# Running Node Cancellation

Cancellation during active execution is best effort.

The runtime should distinguish:

```text
cancellation requested

execution stopped before side effect

side effect already occurred

outcome ambiguous
```

For consequential operations, reconciliation may be required before reporting the result.

---

# Retries

Retries belong to bounded nodes.

A retry should not replay the entire Work Graph.

Example:

```text
Revenue Analyst model timeout
```

Retry:

```text
Revenue Analyst node
```

Not:

```text
resolve recipient again
rebuild everything
restart Work Item
```

---

# Retry Policy

Each executable node should define:

```typescript
type NodeRetryPolicy = {
  maxAttempts: number;

  retryableErrors: string[];

  backoff: BackoffPolicy;

  requireReconciliation?: boolean;
};
```

Agent, capability, and workflow node types may use different retry rules.

---

# Retry Isolation

Completed upstream work should remain completed when a downstream node retries.

Example:

```text
Resolve Recipient    completed
Revenue Analysis     completed
Email Composer       failed
```

Retry only:

```text
Email Composer
```

This improves:

```text
latency

cost

consistency

provider load

debuggability
```

---

# Recovery Paths

Some failures should trigger explicit recovery branches.

Example:

```text
email.send
   │
   ├─ success → verify
   │
   ├─ auth failure → reconnect OAuth
   │
   └─ ambiguous result → reconcile
```

Recovery logic should be represented explicitly where it is predictable.

The Chief of Staff should be invoked only when judgment is required.

---

# Reconciliation Node

For ambiguous external writes:

```text
Send Email
→ ambiguous
→ Reconcile Email Send
```

Then:

```text
found → completed

not found → retry if safe

cannot determine → Chief of Staff / user escalation
```

This prevents duplicate side effects.

---

# Failure Escalation

A Work Node failure does not automatically mean the Work Item fails.

Example:

```text
Schedule Analyst failed.
```

The Chief of Staff may choose:

```text
retry

use direct calendar capability

use alternate specialist

continue without result

ask user

fail that branch only
```

The runtime surfaces failure.

The Chief of Staff judges its effect on the objective.

---

# Work Item Status

The Work Item should have a small, meaningful lifecycle.

Possible states:

```text
created

active

waiting_for_user

waiting_for_approval

waiting_for_external

completed

failed

cancelled
```

The Work Item status is derived from the graph and runtime decisions.

It should not mirror every internal node state.

---

# Work Item Completion

The Work Item completes only when its objective-level completion criteria are satisfied.

Example:

```text
Objective:
Martin receives July report.
```

Criteria:

```text
correct report prepared

correct Martin resolved

approved content used

email sent

provider evidence recorded
```

Even if every reasoning node says `completed`, the Work Item remains incomplete until all required real-world conditions are satisfied.

---

# Completion Evaluation

Conceptually:

```typescript
type WorkCompletionEvaluation = {
  workItemId: string;

  allRequiredNodesSatisfied: boolean;

  completionCriteria: Array<{
    criterionId: string;
    satisfied: boolean;
    evidence: EvidenceReference[];
  }>;

  blockers: string[];
};
```

The runtime performs deterministic checks where possible.

The Chief of Staff may provide judgment for subjective criteria.

---

# Completed Does Not Mean Perfect

A Work Item may complete with limitations.

Example:

```text
Report sent successfully.

One unavailable revenue category was excluded.
```

Completion may include:

```typescript
type CompletionOutcome = {
  status: "completed";

  summary: string;

  evidence: EvidenceReference[];

  limitations: string[];

  followUps?: SuggestedFollowUp[];
};
```

Limitations should be communicated honestly.

---

# Work Graph and Workflows

Registered workflows should use the same execution primitives.

A workflow should not introduce a second parallel execution architecture.

Example:

```text
Morning Brief Workflow

Schedule Retrieval
Revenue Retrieval
Email Review
     ↓
Brief Synthesis
     ↓
Deliver
```

These should ultimately map onto:

```text
Work Nodes
Edges
Assignments
Capabilities
Waits
Evidence
```

This gives the system one execution model.

---

# Parent and Child Work Items

Some objectives may create meaningful sub-objectives.

Example:

```text
Prepare investor update.
```

May include:

```text
Analyze revenue

Analyze customer growth

Review product milestones
```

Initially, I would prefer nodes and assignments within one Work Item.

Child Work Items should be used only where the sub-objective:

```text
has independent lifecycle

may outlive the parent

has separate user-visible responsibility

needs independent priority or cancellation
```

Do not create a Work Item for every tiny step.

---

# Work Graph Mutation

The graph may evolve during execution.

New information may require:

```text
additional assignment

new verification

new approval

new user-input step

recovery branch
```

Graph mutation should be explicit.

It should create:

```text
graph revision

runtime event

causal reason
```

No invisible plan mutation.

---

# Who May Modify the Graph

Graph changes may be proposed by:

```text
Chief of Staff

registered workflow

runtime recovery logic

user correction

policy engine
```

Specialists should not directly mutate the graph.

They return proposed actions.

The Chief of Staff or runtime evaluates those proposals.

---

# Work Graph Events

Every significant event should be persisted.

Examples:

```text
work_item.created

work_graph.created

work_graph.activated

node.created

node.ready

node.queued

node.claimed

node.started

node.completed

node.failed

node.waiting

node.cancelled

node.superseded

graph.revised

approval.resolved

user_input.resolved

work_item.completed

work_item.cancelled
```

The event history should allow us to reconstruct what happened.

---

# Event Log vs Current State

We should preserve both:

```text
current state
```

for efficient runtime execution,

and:

```text
event history
```

for auditability and debugging.

We do not necessarily need full event sourcing as the persistence architecture.

But important state transitions should be recorded as structured events.

---

# Observability

For any Work Item, we should be able to display:

```text
Objective

Current status

Current graph version

Completed nodes

Running nodes

Waiting nodes

Failed nodes

Upcoming nodes

Agent assignments

Capability operations

Approvals

Evidence

Retries

Corrections

Graph revisions

Total latency

Total model usage

Total capability usage
```

An engineer should be able to look at one Work Item and understand exactly why it is where it is.

---

# Critical Path

The runtime should eventually calculate the critical path through the graph.

Example:

```text
Recipient Resolver: 2s
Revenue Analyst: 8s
Email Composer: 3s
Approval: 45s
Send: 1s
```

This tells us where the real completion latency occurred.

It also lets us distinguish:

```text
model latency

provider latency

queue latency

user waiting

approval waiting
```

---

# Execution Metrics

Useful metrics include:

```text
Work Item completion rate

Work Item completion latency

nodes per Work Item

model calls per Work Item

capability calls per Work Item

parallelism rate

retry rate

cancellation rate

correction rate

graph revisions per Work Item

waiting time

provider ambiguity rate

reconciliation rate

false-completion rate

Work Items requiring user clarification
```

The architecture should make these measurable from the start.

---

# Work Graph Testing

The graph engine should be extensively testable without model providers.

Test scenarios:

```text
simple sequential graph

parallel branches

join after parallel work

one branch fails

one branch blocked

approval wait and resume

user-input wait and resume

worker crash

duplicate queue delivery

expired worker lease

node retry

capability ambiguous result

reconciliation recovery

user correction

graph supersession

partial branch reuse

Work Item cancellation

cancellation during running operation

approval becomes stale

scheduled continuation

deployment during waiting state

specialist result revision

completion without required evidence rejected
```

---

# Deterministic Test Clock

Tests should use a controllable clock.

This is necessary for:

```text
leases

timeouts

deadlines

schedules

approval expiration

waiting

retries

backoff
```

No core runtime test should depend on real wall-clock delays.

---

# Example: Send Martin the July Report

Work Item:

```text
Objective:
Martin receives an accurate July revenue report.
```

Graph:

```text
               ┌─────────────────────┐
               │ Resolve Recipient   │
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │                     │
               │    Join Inputs      │◀────────────┐
               │                     │             │
               └──────────┬──────────┘             │
                          ▲                        │
                          │                        │
               ┌──────────┴──────────┐             │
               │ Analyze July Revenue│─────────────┘
               └─────────────────────┘

                          │
                          ▼
               ┌─────────────────────┐
               │    Compose Email    │
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │ Request Approval    │
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │     Send Email      │
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │ Verify Evidence     │
               └──────────┬──────────┘
                          │
                          ▼
                       Complete
```

---

# Example: Rapid User Correction

Current graph:

```text
Resolve Martin      completed

Analyze July        running

Compose Email       pending
```

User:

> Actually June.

Runtime:

```text
1. Chief of Staff identifies correction.

2. Work Graph v1 becomes superseded.

3. Analyze July receives cancellation request.

4. Work Graph v2 created.

5. Resolve Martin output reused.

6. Analyze June becomes ready.

7. Compose Email depends on June analysis.

8. No July-derived draft may execute.
```

No unrelated work is lost.

---

# Example: Parallel Request

User:

> Tell me why revenue dropped, and see if I can meet Martin tomorrow.

Runtime may create two Work Items:

```text
Work Item A:
Explain revenue decline.

Work Item B:
Determine availability to meet Martin tomorrow.
```

Their graphs execute independently.

A failure or correction in one does not destroy the other.

The Chief of Staff may combine their user-facing updates when convenient.

---

# Example: Approval Wait

Graph:

```text
Compose Email      completed

Approval           waiting

Send Email         pending
```

The system may remain in this state overnight.

No model call is running.

No worker is reserved.

When approval arrives:

```text
Approval → approved

Send Email → ready

Scheduler dispatches Send Email
```

---

# Example: Provider Ambiguity

Graph:

```text
Send Email
    ↓
ambiguous result
    ↓
Reconcile Send
```

Reconciliation finds message.

```text
Send Email       completed

Reconciliation   completed

Verify Evidence  ready
```

No duplicate email occurs.

---

# Example: Partial Cancellation

User:

> Don't send the report, but still show me the numbers.

Current graph:

```text
Revenue Analysis   completed

Email Composer     running

Approval           pending

Send               pending
```

Updated objective:

```text
Show user revenue analysis.
Do not send externally.
```

Runtime:

```text
Revenue Analysis   retained

Email Composer     cancelled

Approval           cancelled

Send               cancelled
```

The useful completed work survives.

---

# Initial Architecture Rules

For the first implementation, I would establish:

```text
Work Graph:
Directed and acyclic by default

Graph changes:
Versioned

Maximum active graph depth:
Bounded

Maximum parallel specialist nodes:
3 initially

Specialist graph mutation:
Not allowed

Specialist delegation:
Not allowed

Consequential writes:
Serialized where resource conflict is possible

Waiting:
Durable, never held in a live worker

Queue payloads:
References only

Canonical graph state:
PostgreSQL

Worker execution:
Lease-based

Retries:
Node-specific

Corrections:
Invalidate affected descendants only

Completion:
Requires objective criteria + evidence

Graph cycles:
Forbidden except explicit bounded retry/revision constructs

Every transition:
Persisted and observable
```

---

# Database Concepts

The exact schema comes later, but likely concepts include:

```text
work_items

work_item_objectives

work_item_completion_criteria

work_graphs

work_graph_versions

work_nodes

work_edges

work_node_dependencies

work_node_attempts

work_node_leases

work_node_outputs

work_node_evidence

work_graph_events

wait_conditions

resource_claims
```

These should integrate with:

```text
agent_assignments

capability_operations

approvals

agent_results

external_evidence
```

---

# What We Are Not Building

We are not allowing agents to hold workflows in memory.

We are not treating a model-generated plan as durable execution state.

We are not restarting entire objectives because one step failed.

We are not running long-lived workers while waiting for users.

We are not blindly replaying completed operations.

We are not letting parallel agents independently mutate shared state.

We are not silently changing plans after user corrections.

We are not letting queue payloads become the source of truth.

We are not declaring a Work Item complete because the final model said "done."

We are building a durable execution graph whose state can always explain what Solos has done, what it is doing, what it is waiting for, and what remains.

---

# The Standard

Every Work Graph should be:

```text
Durable
Explicit
Bounded
Observable
Interruptible
Resumable
Recoverable
Versioned
Evidence-aware
Dependency-aware
```

And every Work Item should always be able to answer:

```text
What outcome did we accept?

What has completed?

What is happening now?

What are we waiting for?

What failed?

What changed?

What remains?

What evidence proves the work?
```

---

# One Sentence

> **The Work Graph is the durable execution plan that allows Solos to coordinate agents, capabilities, approvals, waits, and verification while preserving the user's objective through parallel work, interruptions, failures, corrections, and time.**
