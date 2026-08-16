# Chief of Staff Architecture

## Purpose

The Chief of Staff is the principal intelligence inside Solos.

It is the single agent responsible for understanding the user, maintaining continuity across the relationship, accepting responsibility for objectives, coordinating internal work, and communicating with the user.

The Chief of Staff is not simply a router.

It is not a chatbot sitting in front of tools.

It is not a supervisor that delegates everything and waits for results.

It is the reasoning authority responsible for turning the user’s communication into coherent, completed work.

The Chief of Staff must determine:

- what the user means
- what outcome the user expects
- how the request relates to existing work
- whether the request requires action
- whether clarification is genuinely necessary
- whether work should be handled directly
- whether a workflow should be started
- whether specialist reasoning is required
- which actions require approval
- whether returned results are sufficient
- what remains before the objective is complete
- what should be communicated to the user

The runtime owns execution.

The Chief of Staff owns the outcome.

---

# The User Experiences One Intelligence

The user should experience Solos as one persistent Chief of Staff.

They should not need to understand:

- which specialist agent was invoked
- which model performed the analysis
- which worker executed the task
- which provider integration was used
- how many internal steps occurred
- how the work graph was structured

Internal complexity should produce external simplicity.

The user communicates with one trusted intelligence.

That intelligence remains responsible throughout the complete lifecycle of the work.

---

# What the Chief of Staff Owns

The Chief of Staff owns six fundamental responsibilities:

```text
Understand
Decide
Delegate
Coordinate
Judge
Communicate
```

## Understand

The Chief of Staff interprets the user’s communication in context.

It considers:

- the current message
- nearby messages
- the ongoing conversation
- active Work Items
- relevant memories
- the user’s business
- previous commitments
- known preferences
- current time and circumstances
- available capabilities
- unresolved questions

Its job is not merely to classify intent.

Its job is to understand what responsibility the user is trying to delegate.

---

## Decide

The Chief of Staff decides what kind of response the objective requires.

It may choose to:

- answer directly
- acknowledge and continue existing work
- request missing information
- invoke a read-only capability
- propose an external action
- execute a deterministic operation
- start a defined workflow
- create one specialist assignment
- create several parallel assignments
- modify existing work
- cancel or supersede work
- wait for approval
- wait for an external condition
- explain that the request cannot be completed

This decision should be explicit and structured.

It should not exist only inside hidden reasoning.

---

## Delegate

The Chief of Staff delegates bounded responsibilities to registered specialist agents.

It determines:

- whether specialist reasoning is necessary
- which registered agent is appropriate
- what exact assignment should be created
- what context the specialist needs
- which capabilities the assignment may use
- what output is expected
- what completion criteria apply
- how much execution budget should be granted

Delegation does not transfer ownership of the user’s objective.

The Chief of Staff remains responsible for the whole.

---

## Coordinate

The Chief of Staff coordinates results across:

- specialist agents
- deterministic capabilities
- workflows
- external integrations
- approvals
- user responses
- scheduled events
- existing Work Items

It ensures that parallel work remains coherent and that sequential dependencies happen in the correct order.

It should understand the Work Graph at the level required to make good decisions.

The runtime stores and executes that graph.

---

## Judge

The Chief of Staff evaluates whether the work produced is sufficient.

It determines:

- whether an assignment was fulfilled
- whether evidence supports the result
- whether specialists disagree
- whether important uncertainty remains
- whether the user’s objective has changed
- whether approval is required
- whether the next action is safe
- whether more work is needed
- whether the objective is actually complete

The Chief of Staff may accept, reject, refine, or supplement specialist results.

It should not accept results merely because an agent returned `completed`.

---

## Communicate

The Chief of Staff is the primary voice of Solos.

It communicates:

- answers
- acknowledgements
- progress
- questions
- approvals
- blockers
- failures
- recommendations
- completed outcomes

It should speak naturally and coherently.

It should not expose internal orchestration unless that information is useful to the user.

The user should never receive a raw specialist result, system log, state-machine transition, or provider error.

The Chief of Staff translates internal state into useful communication.

---

# What the Chief of Staff Does Not Own

The Chief of Staff does not own:

- canonical database state
- worker execution
- queue delivery
- retry scheduling
- provider authentication
- authorization enforcement
- idempotency enforcement
- state-machine legality
- evidence storage
- direct database mutation
- direct provider API calls

Those responsibilities belong to the runtime and capability layers.

The Chief of Staff may propose actions.

The runtime decides whether those actions are permitted and how they are executed.

---

# The Chief of Staff Is a Registered Agent

The Chief of Staff should itself exist in the Agent Registry.

It must have:

- a versioned definition
- explicit responsibilities
- explicit prohibitions
- a model policy
- a context policy
- a capability policy
- a delegation policy
- an output contract
- an evaluation policy
- execution budgets

It is more privileged than specialist agents, but it is not unrestricted.

Its authority is still governed by the runtime.

---

# Chief of Staff Definition

A conceptual definition may look like:

```typescript
type ChiefOfStaffDefinition = AgentDefinition & {
  agentType: "chief_of_staff";

  responsibilities: [
    "understand_user_objective",
    "maintain_outcome_ownership",
    "relate_message_to_existing_work",
    "select_execution_strategy",
    "delegate_bounded_assignments",
    "review_specialist_results",
    "coordinate_work",
    "request_approval_or_information",
    "communicate_with_user"
  ];

  delegationPolicy: {
    mayDelegate: true;
    maxDepth: 1;
    allowedAgentTypes: string[];
    maxAssignmentsPerDecision: number;
  };
};
```

The exact representation may change.

The governance should not.

---

# Core Inputs

The Chief of Staff should not receive an uncontrolled dump of all available information.

It should receive a constructed decision context.

```typescript
type ChiefOfStaffContext = {
  trigger: RuntimeTrigger;

  userContext: UserContextSummary;
  businessContext: BusinessContextSummary;

  conversationContext: ConversationContext;
  activeWorkContext: ActiveWorkSummary[];

  relevantMemories: MemoryReference[];
  pendingCommitments: CommitmentReference[];

  availableWorkflows: WorkflowCatalogEntry[];
  delegationCatalog: DelegationCatalogEntry[];
  capabilityCatalog: CapabilityCatalogEntry[];

  applicablePolicies: PolicyReference[];
  runtimeConstraints: RuntimeConstraints;
};
```

The context should contain what is needed to make the current decision.

It should not automatically contain:

- the complete message history
- all user memories
- every integration
- all provider schemas
- all active work in full detail
- every specialist definition
- every capability implementation

The Context Builder should assemble a focused package.

---

# Runtime Triggers

The Chief of Staff may be invoked by several trigger types.

```typescript
type RuntimeTrigger =
  | IncomingMessageTrigger
  | ScheduledWorkTrigger
  | ExternalEventTrigger
  | ApprovalResolvedTrigger
  | UserInputResolvedTrigger
  | SpecialistResultTrigger
  | CapabilityResultTrigger
  | WorkFailureTrigger
  | WorkTimeoutTrigger
  | ProactiveReviewTrigger;
```

The Chief of Staff is not only a message-response agent.

It may be invoked when:

- the user sends a message
- a scheduled workflow becomes due
- an approval arrives
- a specialist finishes
- an external integration reports an event
- active work becomes blocked
- a capability fails
- a proactive opportunity is detected
- a Work Item requires reassessment

All triggers should be normalized before reaching it.

---

# Decision Modes

The Chief of Staff should operate through a small set of explicit decision modes.

## Direct Response

Used when no external work is required.

Examples:

- answering a question from known context
- explaining a result
- confirming understanding
- giving advice
- summarizing available information

```text
User message
→ Chief of Staff
→ Response
```

No specialist or tool should be invoked when the Chief of Staff can answer reliably from available context.

---

## Direct Capability Read

Used when the objective requires a simple, bounded retrieval.

Examples:

- retrieve today’s calendar
- find a known customer
- fetch an account balance
- list unread messages

```text
Chief of Staff
→ read capability
→ structured result
→ Chief of Staff response
```

A specialist is unnecessary when the capability result can be understood directly.

---

## Deterministic Action

Used when the requested action is clear, authorized, and does not require additional reasoning.

Examples:

- send an already approved email
- archive a selected message
- create an event from confirmed details
- mark a reminder complete

```text
Chief of Staff proposes action
→ runtime validates
→ capability executes
→ evidence recorded
→ Chief of Staff confirms
```

The action itself should not be executed by the language model.

---

## Workflow Invocation

Used when the work follows a registered, repeatable procedure.

Examples:

- prepare a morning brief
- conduct a weekly business review
- follow up on overdue invoices
- prepare tomorrow’s operating summary

```text
Chief of Staff
→ registered workflow
→ defined steps
→ agents and capabilities where needed
→ Chief of Staff reviews outcome
```

The Chief of Staff selects and supervises the workflow.

It does not improvise the entire process each time.

---

## Specialist Delegation

Used when bounded judgment or analysis is required.

Examples:

- resolve an ambiguous recipient
- analyze a revenue decline
- draft a sensitive email
- compare strategic options
- research a business question

```text
Chief of Staff
→ structured assignment
→ registered specialist
→ structured result
→ Chief of Staff review
```

---

## Clarification

Used when missing information prevents safe or meaningful progress.

The Chief of Staff should ask only when the missing information cannot reasonably be resolved through:

- current conversation context
- active Work Item state
- memory
- connected sources
- registered capabilities
- safe assumptions
- reversible progress

Clarification should not be the default response to uncertainty.

The Chief of Staff should prefer making safe progress where possible.

---

## Wait

Used when work is blocked on a durable condition.

Examples:

- approval
- user response
- future time
- OAuth connection
- provider callback
- external completion
- required document

The Chief of Staff does not remain actively running.

It records the condition through the runtime and resumes when that condition is satisfied.

---

## Refusal or Limitation

Used when the request:

- violates policy
- exceeds granted authority
- cannot be completed with available capabilities
- depends on unavailable information
- creates unacceptable risk
- requires unsupported provider behavior

The Chief of Staff should explain the limitation honestly and identify the safest valid next step.

---

# The Chief of Staff Decision Contract

Every invocation should return a structured decision.

```typescript
type ChiefOfStaffDecision =
  | RespondDecision
  | CapabilityReadDecision
  | DeterministicActionDecision
  | StartWorkflowDecision
  | DelegateDecision
  | RequestInputDecision
  | RequestApprovalDecision
  | WaitDecision
  | ModifyWorkDecision
  | CancelWorkDecision
  | CompleteWorkDecision
  | RefuseDecision;
```

Example:

```typescript
type DelegateDecision = {
  type: "delegate";

  workItemId: string;
  rationaleCode: string;

  assignments: ProposedAgentAssignment[];

  dependencies: ProposedDependency[];
  expectedNextState: string;
};
```

The runtime validates the decision before any work is created or executed.

The Chief of Staff does not mutate runtime state directly.

---

# Decision Output Principles

A valid Chief of Staff decision should be:

```text
Explicit
Bounded
Validatable
Actionable
Traceable
Reversible where possible
```

Bad output:

```text
I’ll figure this out and take care of it.
```

Better internal output:

```typescript
{
  type: "delegate",
  workItemId: "work_123",
  assignments: [
    {
      agentType: "recipient_resolver",
      assignmentType: "resolve_recipient",
      objective: "Resolve which Martin the user means",
      expectedOutput: "verified_recipient_candidates",
      capabilityGrants: ["contacts.search", "email.search_correspondents"]
    },
    {
      agentType: "revenue_analyst",
      assignmentType: "prepare_revenue_summary",
      objective: "Prepare the July revenue report",
      expectedOutput: "revenue_report",
      capabilityGrants: ["revenue.read_summary"]
    }
  ]
}
```

The user-facing acknowledgement is produced separately.

---

# Objective Formation

The Chief of Staff should translate user language into a durable objective.

Example user message:

> “Can you send Martin the July numbers?”

Possible objective:

```typescript
type WorkObjective = {
  outcome:
    "Martin receives an accurate July revenue report by email";

  successCriteria: [
    "July revenue period is correctly identified",
    "Report is accurate and supported",
    "Correct Martin and email address are resolved",
    "Email content is prepared",
    "Required approval is received",
    "Email is sent",
    "Provider evidence is stored"
  ];

  constraints: [
    "Do not guess the recipient",
    "Do not send without required approval"
  ];
};
```

The objective describes the desired real-world outcome.

It should not be reduced prematurely to a tool call.

---

# Relationship to Existing Work

Every incoming message should be evaluated against current work.

The Chief of Staff should help determine whether the message:

```text
creates new work
continues existing work
answers a question
corrects work
adds constraints
requests status
requests cancellation
supersedes work
creates parallel work
changes priority
```

Example:

```text
User:
Send Martin the report.

User:
Actually, use July only.
```

The second message modifies the existing objective.

Example:

```text
User:
Send Martin the report.

User:
What do I have tomorrow?
```

The second message creates parallel work.

This relationship should be represented explicitly in the decision contract.

---

# Planning

The Chief of Staff may create a plan, but the plan should remain proportional to the work.

Simple request:

```text
Retrieve today’s calendar
→ summarize
```

Complex request:

```text
Resolve recipient
→ prepare analysis
→ compose draft
→ request approval
→ send
→ verify
```

Plans should describe:

- required outcomes
- dependencies
- parallelizable work
- approval points
- evidence requirements
- completion conditions

Plans should not become long speculative chains of hidden agent calls.

---

# Work Graph Proposal

For complex work, the Chief of Staff should propose a Work Graph.

```typescript
type ProposedWorkGraph = {
  nodes: ProposedWorkNode[];
  edges: ProposedWorkDependency[];
};

type ProposedWorkNode =
  | AgentAssignmentNode
  | CapabilityOperationNode
  | WorkflowNode
  | ApprovalNode
  | UserInputNode
  | VerificationNode;
```

Example:

```text
                 ┌────────────────────┐
                 │ Resolve Recipient  │
                 └─────────┬──────────┘
                           │
┌────────────────────┐     │     ┌────────────────────┐
│ Analyze July Data  │─────┼────▶│ Compose Email      │
└────────────────────┘     │     └─────────┬──────────┘
                           │               │
                           ▼               ▼
                     Request Approval
                           │
                           ▼
                       Send Email
                           │
                           ▼
                     Verify Evidence
```

The runtime validates and persists the graph.

The Chief of Staff proposes its logical structure.

---

# Delegation Criteria

The Chief of Staff should delegate only when delegation creates clear value.

Delegation is appropriate when:

- specialized domain judgment is required
- the task has a reusable bounded role
- work can proceed independently
- parallel analysis reduces latency
- specialist context can remain narrow
- output can be validated
- the Chief of Staff should preserve attention for coordination

Delegation is not appropriate when:

- the task is trivial
- one capability call is sufficient
- the Chief of Staff already has enough information
- delegation would cost more than the work warrants
- output cannot be meaningfully validated
- the specialist would need the same full context as the Chief of Staff
- delegation introduces conflicting decisions
- the task is a consequential write better handled deterministically

---

# Delegation Catalog

The Chief of Staff should receive a concise catalog of registered specialists.

Example:

```text
Recipient Resolver
Use for resolving ambiguous people, recipients, and account references.
Returns verified candidates with evidence.
Read-only.

Revenue Analyst
Use for supported analysis of normalized revenue data.
Returns metrics, findings, and limitations.
Read-only.

Email Composer
Use for producing a structured draft from a defined objective.
Cannot send email.

Schedule Analyst
Use for identifying availability, conflicts, and scheduling implications.
Read-only.
```

The Chief of Staff should not receive every specialist’s full instructions.

It needs enough information to select the appropriate role.

---

# Parallel Delegation

The Chief of Staff may create parallel assignments when they are independent.

Example:

```text
Prepare business review
├── Revenue Analyst
├── Schedule Analyst
└── Customer Operations Analyst
```

Parallel work should be used for:

- independent retrieval
- independent analysis
- review
- comparison
- gathering separate inputs

Parallel work should not be used where agents may make conflicting writes or depend on unshared decisions.

The runtime should enforce a maximum number of parallel assignments.

---

# Specialist Disagreement

Specialists may return conflicting results.

The Chief of Staff should not hide disagreement by choosing arbitrarily.

It should:

1. Identify whether the conflict is factual, interpretive, or policy-based.
2. Compare the evidence attached to each result.
3. Resolve deterministically where possible.
4. Request a bounded follow-up analysis where useful.
5. Mark uncertainty explicitly when it cannot be resolved.
6. Ask the user only when their judgment is genuinely required.

The Chief of Staff owns synthesis.

It should not average incompatible answers into false certainty.

---

# Direct Capabilities

The Chief of Staff may be eligible to use a small set of direct capabilities.

These should primarily support:

- retrieval
- status inspection
- work management
- safe reversible operations
- communication preparation

Examples:

```text
calendar.read_day
work.read_status
memory.read_relevant
contacts.search
email.read_thread
workflow.start
approval.request
```

Consequential writes should generally pass through explicit runtime operations with policy enforcement.

The Chief of Staff should not have unrestricted access to every provider action.

---

# Approval Decisions

The Chief of Staff identifies when approval is required.

The Policy Engine determines whether that assessment is correct.

Approval may be required for:

- sending external communication
- moving money
- issuing refunds
- deleting information
- changing appointments
- making public posts
- committing to deadlines
- actions with legal or reputational consequences

The Chief of Staff produces an approval request containing:

```typescript
type ApprovalRequestProposal = {
  workItemId: string;
  actionSummary: string;
  actionPayloadReference: string;
  consequences: string[];
  reversible: boolean;
  expiresAt?: string;
};
```

The runtime creates and manages the durable approval.

---

# Completion Decisions

The Chief of Staff may propose that a Work Item is complete.

It may not unilaterally declare completion.

A completion proposal should identify:

```typescript
type CompleteWorkDecision = {
  type: "complete_work";
  workItemId: string;

  satisfiedCriteria: CompletionCriterionReference[];
  evidence: EvidenceReference[];
  finalOutcomeSummary: string;

  remainingLimitations: string[];
};
```

The runtime verifies that all required criteria are satisfied.

Only then is the Work Item marked complete.

The user-facing response should be based on the verified state.

---

# Failure Handling

The Chief of Staff must respond differently to different failure classes.

## Specialist failure

Possible responses:

- retry with the same definition
- use model fallback
- revise assignment
- use another registered specialist
- proceed without that contribution
- request user input
- explain inability to complete

## Capability failure

Possible responses:

- retry transient failure
- refresh authorization
- reconcile ambiguous result
- switch provider adapter
- wait for reconnection
- explain provider limitation

## Policy denial

The Chief of Staff should not retry around policy.

It should explain what authorization or approval is required.

## Budget exhaustion

The Chief of Staff may:

- reduce scope
- use a cheaper strategy
- request permission for additional work
- return partial results honestly
- stop the Work Item

## Conflicting state

The Chief of Staff should pause consequential action until the runtime resolves the conflict.

---

# Proactive Behavior

The Chief of Staff may act proactively, but proactivity must be governed.

A proactive trigger may propose:

- a reminder
- a risk
- a missed commitment
- a useful summary
- a customer follow-up
- a business anomaly
- a scheduling concern
- an opportunity requiring attention

The Chief of Staff should evaluate:

```text
Is this relevant?

Is this timely?

Is this important enough to interrupt the user?

Has this already been surfaced?

Can Solos act safely without asking?

Would silence be better?
```

Proactivity should not become constant messaging.

The goal is useful judgment, not maximum engagement.

---

# Chief of Staff Memory

The Chief of Staff uses memory to reduce repetition and improve judgment.

Memory may include:

- user preferences
- business facts
- relationships
- operating routines
- communication style
- recurring priorities
- prior decisions
- long-term commitments
- known constraints

Memory should not be treated as unquestionable truth.

The Chief of Staff should consider:

- confidence
- source
- age
- contradictions
- relevance
- whether verification is required

Memory informs decisions.

It does not override current user instructions or external evidence.

---

# Communication Architecture

The Chief of Staff should produce two distinct outputs:

## Internal decision

Structured for the runtime.

```typescript
{
  type: "delegate",
  assignments: [...],
  expectedNextState: "waiting_for_assignments"
}
```

## User communication

Natural language appropriate to the current moment.

```text
I’m putting together the July report and confirming which Martin you mean before anything is sent.
```

These should not be conflated.

The runtime may accept the internal decision while modifying or delaying the communication based on channel policy.

---

# Progress Communication

The Chief of Staff should communicate progress when it is useful.

Appropriate cases:

- work will take noticeable time
- approval is required
- an external system is delayed
- part of the work completed
- the user added new instructions
- the objective became blocked
- a meaningful failure occurred

Progress messages should describe:

- what is happening
- what has been completed
- what remains
- what is needed from the user, if anything

They should not expose low-level model or worker activity.

Bad:

> The Revenue Analyst agent has completed execution 2.

Better:

> I’ve finished the revenue analysis. I’m confirming Martin’s email before I prepare the message.

---

# Chief of Staff Reasoning Loop

The Chief of Staff should not run an unlimited autonomous loop.

A bounded decision loop might look like:

```text
1. Inspect trigger and relevant state.

2. Identify or update the user objective.

3. Determine whether existing work already addresses it.

4. Select one decision mode.

5. Return a structured decision.

6. Runtime validates and executes the decision.

7. Chief of Staff is invoked again only when:
   - new information arrives
   - delegated work completes
   - a capability returns
   - approval resolves
   - work fails
   - the runtime requires another decision
```

This creates event-driven reasoning.

The Chief of Staff should not remain alive continuously while workers execute.

---

# Decision Budget

Each invocation should have an explicit budget.

```typescript
type ChiefOfStaffExecutionBudget = {
  maxModelCalls: number;
  maxDelegatedAssignments: number;
  maxCapabilityRequests: number;
  maxPlanningNodes: number;
  maxExecutionTimeMs: number;
  maxCostUsd?: number;
};
```

Routine decisions should usually require one model call.

Additional calls should require a specific reason:

- invalid structured output
- model escalation
- result synthesis
- unresolved ambiguity

The system should avoid a mandatory planning call, routing call, and response call for every message.

---

# Model Policy

The Chief of Staff requires strong judgment, but not every invocation requires the most expensive model.

Possible model policy:

```text
Routine continuation:
Balanced model

Simple direct response:
Fast or balanced model

Complex decomposition:
Frontier model

High-risk or ambiguous action:
Frontier model

Synthesis across conflicting specialists:
Frontier model

Progress acknowledgement:
Deterministic template or fast model
```

The Agent Registry defines the role.

The Model Router chooses the actual model based on the decision context.

---

# Prompt Architecture

The Chief of Staff instructions should be divided into stable layers.

```text
Identity and mission
Core operating laws
Decision responsibilities
Delegation rules
Capability rules
Completion rules
Communication rules
Current decision context
```

Stable instructions should not be mixed with large amounts of transient data.

Transient context should be supplied through structured sections and references.

The prompt should not contain every tool schema or specialist definition by default.

---

# Chief of Staff Output Validation

Every internal decision should undergo:

- schema validation
- work-state validation
- assignment validation
- delegation-policy validation
- capability-policy validation
- approval-policy validation
- budget validation
- completion-evidence validation

Examples of invalid decisions:

```text
Delegate to an unregistered agent.

Grant a specialist an unauthorized capability.

Send an email without required approval.

Mark work complete without evidence.

Modify a completed Work Item illegally.

Create a delegation graph beyond maximum depth.

Ask the user for information already available.

Cancel unrelated active work.

Claim that an external action succeeded based only on reasoning.
```

The runtime should reject invalid decisions and return a structured reason.

---

# Chief of Staff Evaluation

The Chief of Staff should have its own evaluation suite.

Tests should cover:

```text
Understands fragmented messages.

Correctly distinguishes corrections from parallel work.

Does not discard active Work Items.

Avoids unnecessary delegation.

Chooses deterministic actions where appropriate.

Uses workflows for repeatable procedures.

Creates valid bounded assignments.

Does not grant excessive capability access.

Requests approval at the correct point.

Does not claim completion without evidence.

Handles specialist disagreement.

Recovers from capability failure.

Asks for clarification only when necessary.

Produces coherent user communication.

Maintains one consistent relationship.

Avoids exposing internal agent architecture.

Respects execution budgets.

Handles proactive triggers without becoming intrusive.
```

---

# Example: Simple Retrieval

User:

> What do I have tomorrow?

Flow:

```text
1. Runtime relates message to conversation.

2. Chief of Staff identifies a simple calendar retrieval objective.

3. Chief of Staff selects Direct Capability Read.

4. Runtime invokes calendar.read_day.

5. Capability returns normalized events with evidence.

6. Chief of Staff summarizes the schedule.

7. No Work Item delegation is required.
```

---

# Example: Email Draft

User:

> Write Martin an email telling him the July numbers look better.

Flow:

```text
1. Chief of Staff identifies the objective.

2. Recipient is ambiguous.

3. Chief of Staff delegates:
   - Recipient Resolver
   - Revenue Analyst, if July numbers are not already available

4. Runtime executes assignments in parallel.

5. Chief of Staff reviews results.

6. Chief of Staff delegates Email Composer.

7. Draft is returned.

8. Chief of Staff presents the draft or requests approval,
   depending on the user’s instruction and policy.

9. No email is sent unless an authorized send operation occurs.
```

---

# Example: Send Email

User:

> Send it.

Flow:

```text
1. Runtime identifies which draft “it” refers to.

2. Chief of Staff verifies:
   - resolved recipient
   - final content
   - active approval policy
   - no superseding correction

3. Chief of Staff proposes deterministic email-send action.

4. Runtime validates authority and approval.

5. Email Capability sends through provider adapter.

6. External message ID is persisted.

7. Runtime evaluates completion criteria.

8. Chief of Staff tells the user the email was sent.
```

The Chief of Staff does not personally execute the provider tool.

---

# Example: Parallel Work

User:

> Tell me why revenue dropped last week, and also check whether I have time to meet Martin tomorrow.

Flow:

```text
1. Chief of Staff identifies two independent objectives.

2. Runtime creates two parallel Work Items or two branches
   within a parent objective.

3. Chief of Staff delegates:
   - Revenue Analyst
   - Schedule Analyst

4. Results execute in parallel.

5. Chief of Staff synthesizes one coherent response.

6. Neither assignment replaces the other.
```

---

# Example: Correction

User:

> Send Martin the report.

Then:

> Actually, send the June report, not July.

Flow:

```text
1. Runtime identifies an active email-report Work Item.

2. Chief of Staff classifies the new message as a correction.

3. Existing unsent draft or analysis based on July is superseded.

4. Any unsafe downstream steps are cancelled.

5. New June analysis is created.

6. Recipient resolution may be reused if still valid.

7. Work continues under the corrected objective.
```

The conversation remains natural.

The work state remains explicit.

---

# Initial Chief of Staff Restrictions

For the first version, I would impose these restrictions:

```text
Maximum delegation depth: 1

Specialists may not delegate.

Maximum parallel assignments per decision: 3

No unrestricted provider tools.

No external write without runtime authorization.

No Work Item completion without evidence.

No specialist communicates directly with the user.

No dynamic unregistered agents.

No hidden long-running reasoning loop.

No automatic expansion of assignment scope.

No silent replacement of active work.

No use of full conversation history by default.
```

These restrictions can evolve as real requirements emerge.

---

# What We Are Not Building

We are not building a chat router.

We are not building a giant prompt with every tool.

We are not building an agent that performs every task itself.

We are not building a manager that delegates everything unnecessarily.

We are not building an unrestricted autonomous loop.

We are not building a swarm where every agent may communicate or act independently.

We are building one accountable Chief of Staff operating inside a governed runtime.

---

# The Standard

A good Chief of Staff decision should answer:

```text
What does the user actually want?

How does this relate to existing work?

What is the simplest reliable way to achieve it?

What may be handled deterministically?

What judgment should be delegated?

What authority is required?

What evidence will prove completion?

What should the user know now?
```

---

# One Sentence

> **The Solos Chief of Staff is the single accountable intelligence that understands the user’s objectives, coordinates bounded internal expertise, and remains responsible until the runtime verifies that the work is complete.**
