# Model Routing and Execution Budgets

## Purpose

The Model Routing and Execution Budget system defines **which intelligence Solos uses, when it uses it, how much computation a piece of work is allowed to consume, and when the runtime should escalate, retry, degrade, or stop.**

This layer exists because a multi-agent system can become slow and expensive very quickly if every decision becomes:

```text
router model
→ planner model
→ Chief of Staff
→ specialist
→ evaluator
→ synthesizer
→ final response
```

That is exactly what we do **not** want.

The goal is not to use the smartest possible model for every operation.

The goal is:

> **Use the minimum intelligence required to reliably make the current decision.**

Model selection is an infrastructure decision.

It should not become another uncontrolled reasoning loop.

---

# Foundational Principle

> **Model calls are expensive reasoning operations, not free function calls.**

Every model call has a cost in:

```text
latency
tokens
money
failure probability
context construction
provider dependency
system complexity
```

Therefore, every call should justify its existence.

---

# Another Critical Principle

> **Do not create a model call merely to decide which model call to make.**

The Model Router should primarily be deterministic.

We should not require:

```text
routing LLM
→ selected agent LLM
```

for every interaction.

That creates fixed latency before useful reasoning begins.

Instead:

```text
Agent Definition
+
Assignment Type
+
Risk
+
Complexity
+
Required Capabilities
+
Runtime Conditions
        ↓
Deterministic Model Router
        ↓
Selected Model
```

A model-assisted routing decision may exist for genuinely ambiguous cases.

It should be the exception.

Not the default.

---

# Models Are Execution Resources

Agents and models must remain separate concepts.

```text
Agent:
Revenue Analyst

Model:
Whatever approved model best executes this
particular Revenue Analyst assignment.
```

Never:

```text
Revenue Analyst = Model X
```

Agent identity defines:

```text
responsibility

authority

input contract

output contract

behavior

evaluation
```

Model policy defines:

```text
what level of intelligence is appropriate
```

The Model Router determines:

```text
which actual model executes now
```

---

# The Architecture

```text
Agent / Chief of Staff Execution
              │
              ▼
       Execution Profile
              │
              ▼
          Model Policy
              │
              ▼
         Model Router
              │
        ┌─────┼─────┐
        ▼     ▼     ▼
      Fast Balanced Frontier
        │     │     │
        └─────┼─────┘
              ▼
       Provider Adapter
              │
              ▼
          Model Call
              │
              ▼
      Structured Result
              │
              ▼
       Usage + Evaluation
```

---

# Model Classes

The runtime should reason about **model classes**, not provider marketing names.

Initially, I would define three primary classes.

```text
Fast

Balanced

Frontier
```

Potentially later:

```text
specialized
vision
embedding
audio
```

But those should exist only when real requirements justify them.

---

# Fast Model Class

Used for highly bounded work where:

```text
input is narrow

output is strongly typed

judgment requirements are low

failure is easy to detect

latency matters

results are inexpensive to retry
```

Examples:

```text
simple classification

structured extraction

entity candidate ranking

message categorization

simple rewriting

basic summarization

low-ambiguity recipient ranking
```

Fast models should not be given broad strategic responsibility simply because they are cheaper.

---

# Balanced Model Class

The default for meaningful but bounded reasoning.

Examples:

```text
email composition

moderate business analysis

schedule analysis

conversation interpretation

routine Chief of Staff decisions

specialist synthesis

bounded planning
```

This will probably handle a large percentage of Solos reasoning.

---

# Frontier Model Class

Reserved for work where stronger reasoning materially improves correctness.

Examples:

```text
complex Chief of Staff decomposition

high ambiguity

conflicting specialist results

complex causal analysis

high-risk communication

strategic reasoning

difficult multi-step synthesis

recovery from failed lower-tier reasoning
```

The frontier model should not become the default simply because it performs best.

---

# Capability Requirements

Some executions require specific model functionality.

For example:

```text
structured output

tool calling

long context

vision

large output

low latency

strong reasoning
```

The Model Router must eliminate models that cannot satisfy required capabilities.

Conceptually:

```typescript
type ModelRequirement = {
  structuredOutput: boolean;
  toolCalling: boolean;

  vision?: boolean;

  minimumContextWindow?: number;
  minimumOutputTokens?: number;

  maximumLatencyClass?: string;
};
```

---

# Agent Model Policy

Every Agent Definition defines its allowed model strategy.

```typescript
type AgentModelPolicy = {
  preferredClass:
    | "fast"
    | "balanced"
    | "frontier";

  allowedClasses: Array<
    | "fast"
    | "balanced"
    | "frontier"
  >;

  requirements: ModelRequirement;

  escalationPolicy: EscalationPolicy;

  fallbackPolicy: FallbackPolicy;

  maximumModelCallsPerExecution: number;
};
```

Example:

```text
Recipient Resolver

Preferred:
fast

Allowed:
fast
balanced

Frontier:
not normally required

Escalate:
when ambiguity remains above threshold
```

---

# Chief of Staff Model Policy

The Chief of Staff requires a more dynamic model policy.

Routine interaction:

```text
balanced
```

Simple communication:

```text
fast or balanced
```

Complex objective decomposition:

```text
frontier
```

High-risk decision:

```text
frontier
```

Conflicting specialist results:

```text
frontier
```

Simple continuation after a capability result:

```text
balanced
```

The Chief of Staff does not always require frontier reasoning.

---

# Execution Profile

Before model selection, the runtime should construct an Execution Profile.

```typescript
type ModelExecutionProfile = {
  role:
    | "chief_of_staff"
    | "specialist"
    | "evaluator"
    | "classifier"
    | "communication";

  agentType?: string;
  assignmentType?: string;

  complexity:
    | "low"
    | "medium"
    | "high";

  ambiguity:
    | "low"
    | "medium"
    | "high";

  risk:
    | "low"
    | "medium"
    | "high";

  contextTokensEstimated: number;

  outputTokensEstimated: number;

  requirements: ModelRequirement;

  latencyPriority:
    | "interactive"
    | "normal"
    | "background";

  budgetRemaining: ExecutionBudgetRemaining;
};
```

This becomes the primary input to the router.

---

# Complexity Should Be Grounded

Complexity should not mean:

```text
"This feels difficult."
```

It should derive from observable properties where possible.

Examples:

```text
number of objectives

number of dependencies

amount of conflicting evidence

number of entities

required reasoning depth

number of specialist results to synthesize

size of relevant context

whether planning is required

whether the output is open-ended
```

---

# Risk and Complexity Are Different

A task may be:

```text
simple + high risk
```

Example:

```text
Send a $5,000 refund.
```

Or:

```text
complex + low risk
```

Example:

```text
Analyze six months of revenue and identify trends.
```

Risk may justify stronger evaluation or human approval.

It does not necessarily justify more model calls.

---

# Deterministic Model Router

Conceptually:

```typescript
interface ModelRouter {
  select(
    profile: ModelExecutionProfile,
    policy: AgentModelPolicy,
    runtimeState: ModelRuntimeState
  ): Promise<ModelSelection>;
}
```

A selection:

```typescript
type ModelSelection = {
  modelClass:
    | "fast"
    | "balanced"
    | "frontier";

  provider: string;
  modelId: string;

  reasonCodes: string[];

  timeoutMs: number;

  maxOutputTokens: number;
};
```

The decision should be traceable.

---

# Example Routing Rules

Conceptually:

```text
IF
role = recipient_resolver
AND ambiguity = low

THEN
fast
```

```text
IF
role = chief_of_staff
AND complexity = high

THEN
frontier
```

```text
IF
risk = high
AND reasoning materially affects action

THEN
frontier
```

```text
IF
context exceeds model capacity

THEN
exclude model
```

```text
IF
provider currently unhealthy

THEN
exclude provider
```

These should be deterministic rules.

---

# Provider Independence

The runtime should depend on an internal Model Provider interface.

```typescript
interface ModelProvider {
  generate(
    request: ModelRequest
  ): Promise<ModelResponse>;

  stream?(
    request: ModelRequest
  ): AsyncIterable<ModelChunk>;
}
```

OpenRouter may be one adapter.

Future adapters may include direct provider APIs.

The runtime should not know OpenRouter-specific request structures.

---

# Model Catalog

The system should maintain an internal Model Catalog.

```typescript
type ModelCatalogEntry = {
  id: string;

  provider: string;
  providerModelId: string;

  class:
    | "fast"
    | "balanced"
    | "frontier";

  capabilities: ModelCapabilities;

  contextWindow: number;
  maximumOutputTokens: number;

  costProfile: ModelCostProfile;

  latencyProfile: ModelLatencyProfile;

  reliabilityProfile: ModelReliabilityProfile;

  status:
    | "active"
    | "degraded"
    | "disabled";
};
```

Models become runtime resources.

Not hardcoded strings spread throughout agents.

---

# Model Health

Model selection should account for current provider health.

Track:

```text
success rate

timeout rate

invalid structured output rate

tool-call failure rate

latency

rate-limit events

provider outages
```

A normally preferred model may temporarily be excluded.

---

# Circuit Breaking

If a provider or model becomes unreliable:

```text
failure threshold exceeded
→ circuit opens
→ stop routing new work there
```

After a recovery period:

```text
limited probes
→ healthy
→ restore traffic
```

This avoids repeatedly burning latency on known failing infrastructure.

---

# Fallback

Fallback means:

> The selected model could not successfully execute, so use another approved model.

Example:

```text
Balanced Model A
      ↓ timeout
Balanced Model B
      ↓ unavailable
Frontier Model
```

Fallback should follow registered policy.

---

# Fallback Is Not Retry

These are different.

## Retry

Same model.

```text
Model A
→ transient failure
→ Model A again
```

## Fallback

Different model or provider.

```text
Model A
→ failure
→ Model B
```

The runtime should distinguish them for observability.

---

# Model Failure Classes

Normalize model failures:

```typescript
type ModelFailureClass =
  | "TIMEOUT"
  | "RATE_LIMITED"
  | "PROVIDER_UNAVAILABLE"
  | "INVALID_STRUCTURED_OUTPUT"
  | "TOOL_CALL_INVALID"
  | "CONTEXT_TOO_LARGE"
  | "OUTPUT_TRUNCATED"
  | "CONTENT_POLICY"
  | "AUTHENTICATION"
  | "UNKNOWN";
```

Different failures require different strategies.

---

# Invalid Structured Output

If the model produces malformed structured output:

```text
first attempt
→ deterministic parse / repair if safe
```

If still invalid:

```text
one constrained retry
```

Then:

```text
fallback or fail
```

Do not run five increasingly desperate model calls.

---

# Escalation

Escalation differs from fallback.

Fallback occurs because execution failed.

Escalation occurs because the current intelligence level appears insufficient.

Example:

```text
Fast Recipient Resolver
→ two equally plausible candidates
→ balanced model
```

Or:

```text
Balanced Chief of Staff
→ unresolved conflicting evidence
→ frontier model
```

---

# Escalation Signals

Useful escalation signals:

```text
low confidence

multiple valid interpretations

conflicting evidence

evaluation failure

high-impact ambiguity

complex dependency graph

specialist disagreement

repeated invalid output

explicit agent policy
```

Escalation should be bounded.

---

# No Infinite Escalation

A policy may define:

```typescript
type EscalationPolicy = {
  allowed: boolean;

  triggers: EscalationTrigger[];

  escalationClasses: string[];

  maxEscalations: number;
};
```

Initial recommendation:

```text
Maximum escalation:
1 per execution
```

A specialist should not repeatedly climb through models indefinitely.

---

# Confidence

Model-reported confidence should not be blindly trusted.

Where confidence matters, combine:

```text
model confidence

evidence quality

deterministic ambiguity measures

evaluation result

number of plausible candidates
```

For example, Recipient Resolver confidence can partially derive from:

```text
one exact verified contact match
vs
three similarly plausible contacts
```

not merely:

```text
model says 0.92
```

---

# Execution Budgets

Every meaningful unit of work should have a finite resource budget.

Budgets exist at several levels:

```text
Work Item

Chief of Staff decision

Work Node

Agent Assignment

Agent Execution

Capability execution
```

Lower-level budgets must fit within higher-level budgets.

---

# Hierarchical Budgeting

Conceptually:

```text
Work Item Budget
       │
       ├── Chief of Staff
       │
       ├── Assignment A
       │      ├── Model Calls
       │      └── Capabilities
       │
       ├── Assignment B
       │      ├── Model Calls
       │      └── Capabilities
       │
       └── Final Synthesis
```

A child may consume only part of its parent's available budget.

---

# Budget Dimensions

Budgets should not be defined only in dollars.

They should include:

```typescript
type ExecutionBudget = {
  maxModelCalls: number;

  maxCapabilityCalls: number;

  maxDelegatedAssignments: number;

  maxParallelExecutions: number;

  maxInputTokens?: number;
  maxOutputTokens?: number;

  maxModelCostUsd?: number;
  maxTotalCostUsd?: number;

  maxWallTimeMs?: number;

  deadlineAt?: string;
};
```

Different tasks may care about different dimensions.

---

# Budget Is a Ceiling

A budget does not mean:

```text
Use 4 model calls because 4 are available.
```

It means:

```text
Never exceed 4 calls.
```

The preferred execution remains:

```text
minimum required calls
```

---

# Work Item Budget

A Work Item may have an overall budget.

Example:

```typescript
{
  maxModelCalls: 8,
  maxCapabilityCalls: 12,
  maxDelegatedAssignments: 3,
  maxParallelExecutions: 3,
  maxWallTimeMs: 120000
}
```

Simple Work Items should have much smaller budgets.

---

# Dynamic Budget Assignment

The Chief of Staff may propose complexity.

The runtime then selects an appropriate budget class.

```text
trivial
small
normal
complex
background
```

Example:

```text
"What do I have today?"

Budget:
1 reasoning call
1 calendar capability call
```

Compared with:

```text
"Analyze last quarter and prepare an investor update."

Budget:
multiple bounded assignments
larger context
parallel retrieval
frontier synthesis
```

---

# Budget Classes

Conceptually:

```typescript
type ExecutionBudgetClass =
  | "micro"
  | "standard"
  | "complex"
  | "background";
```

## Micro

```text
1 model call

1-2 capability calls

no delegation
```

## Standard

```text
1-2 Chief of Staff calls

1 specialist if required

limited capabilities
```

## Complex

```text
multiple bounded specialists

parallel reads

frontier synthesis

larger execution window
```

## Background

Used for:

```text
scheduled analysis

large research jobs

non-interactive workflows
```

May allow more time while still remaining bounded.

---

# Latency Budgets

Because Solos is text-native, latency must be treated as a product constraint.

Different phases have different budgets.

```text
ingestion latency

first-response latency

decision latency

specialist latency

capability latency

final-completion latency
```

We should never treat:

```text
"the whole thing took 42 seconds"
```

as enough information.

---

# Interactive vs Background Work

The runtime should know whether work is:

```text
interactive

normal

background
```

Interactive work should prioritize:

```text
low latency

minimal model calls

quick acknowledgement

parallel independent reads
```

Background work may prioritize:

```text
cost efficiency

deeper analysis

larger context

slower providers
```

---

# First Response Is Different From Completion

Text-native UX requires distinguishing:

```text
time to meaningful response
```

from:

```text
time to completed work
```

Example:

```text
User:
Send Martin the report.
```

Solos may quickly respond:

> I’m putting the report together and confirming Martin’s email before I send anything.

Then execution continues.

We should not force the complete workflow into one synchronous request simply to produce one reply.

---

# Avoid Mandatory Reasoning Stages

There should not be a universal pipeline like:

```text
router
→ planner
→ outcome declaration
→ executor
→ verifier
→ response writer
```

for every message.

Different requests need different paths.

Simple:

```text
Chief of Staff
→ answer
```

Simple retrieval:

```text
Chief of Staff
→ capability
→ response
```

Complex:

```text
Chief of Staff
→ specialists
→ synthesis
→ action
```

Architecture should support variable depth.

---

# One-Call Principle

For routine Chief of Staff decisions, our target should be:

> **One meaningful reasoning call whenever one call can reliably decide the next action.**

Not:

```text
one call for routing

one for planning

one for execution

one for wording
```

The Chief of Staff's structured output should often contain both:

```text
internal decision
+
user-facing communication intent
```

where appropriate.

---

# Deterministic Work Should Use Zero Model Calls

Examples:

```text
resume approved workflow

execute previously approved send

retry safe provider read

schedule known Work Node

evaluate deterministic evidence

reconcile provider operation

calculate arithmetic metric
```

These should not invoke a model simply because the overall product is AI.

---

# Specialist Call Budget

Specialists should usually be optimized around:

```text
one model call
```

with optional:

```text
bounded capability calls during that execution
```

Additional model calls should be reserved for:

```text
structured-output correction

approved escalation

targeted revision
```

---

# Evaluation Budget

Not every result needs another LLM judging it.

Evaluation hierarchy:

```text
schema validation first

deterministic checks second

evidence checks third

Chief of Staff review where appropriate

model evaluator only when necessary
```

This avoids doubling model cost for every assignment.

---

# Model Evaluators

Use model-based evaluators for subjective questions:

```text
Is this email appropriately sensitive?

Does this synthesis accurately represent several nuanced analyses?

Is the communication clear and professional?
```

Do not use them unnecessarily for:

```text
Does 100 + 50 = 150?

Does provider message ID exist?

Does recipient match approval payload?

Does schema validate?
```

---

# Critical-Path Budgeting

Parallel branches should be evaluated by their effect on total completion time.

Example:

```text
Recipient Resolver: 2s

Revenue Analyst: 8s
```

Running sequentially:

```text
10s
```

Running safely in parallel:

```text
~8s
```

The scheduler should exploit safe parallelism.

But adding three redundant agents to analyze the same thing is not useful parallelism.

---

# Redundant Reasoning

We should generally avoid:

```text
Agent A analyzes

Agent B independently analyzes same thing

Agent C votes

Chief of Staff synthesizes
```

unless the work has a demonstrated need for adversarial or independent evaluation.

Redundancy should be intentional.

Not a default reliability strategy.

---

# Cost Accounting

Every model execution should record:

```text
input tokens

output tokens

cached tokens where applicable

model

provider

estimated cost

actual cost where known

latency

assignment

Work Item
```

This allows us to calculate:

```text
cost per user message

cost per Work Item

cost per completed outcome

cost per agent type

cost per workflow

cost per successful external action
```

The most important metric is eventually:

> **Cost per reliably completed user outcome.**

Not simply cost per token.

---

# Usage Record

Conceptually:

```typescript
type ModelUsageRecord = {
  executionId: string;

  workItemId?: string;
  assignmentId?: string;

  provider: string;
  modelId: string;

  inputTokens: number;
  outputTokens: number;

  estimatedCostUsd: number;

  latencyMs: number;

  result:
    | "success"
    | "failure"
    | "fallback"
    | "escalated";

  recordedAt: string;
};
```

---

# Budget Reservation

For parallel work, the runtime may reserve budget before dispatch.

Example:

```text
Work Item has $0.10 remaining.

Three assignments each could consume $0.08.
```

We should not blindly dispatch all three.

The runtime may:

```text
reserve expected maximum

reduce assignments

choose cheaper models

execute sequentially

request higher budget
```

This prevents concurrency from defeating budget controls.

---

# Budget Exhaustion

Budget exhaustion is a normal runtime condition.

Possible behavior:

```text
stop optional work

use deterministic fallback

return partial result

reduce scope

use cheaper model if policy permits

ask Chief of Staff to reprioritize

request additional authorization where appropriate
```

Never silently continue beyond the budget.

---

# Budget Exhaustion Result

```typescript
type BudgetExhausted = {
  dimension:
    | "model_calls"
    | "capability_calls"
    | "tokens"
    | "cost"
    | "wall_time"
    | "delegation";

  consumed: number;

  limit: number;

  partialResultReferences: string[];
};
```

The Chief of Staff can then make an informed decision.

---

# Graceful Degradation

When ideal intelligence is unavailable, Solos should degrade predictably.

Example:

```text
frontier provider unavailable
```

Possible policy:

```text
use approved balanced fallback
```

But if the work is high-risk and policy says frontier judgment is mandatory:

```text
do not downgrade

wait or explain limitation
```

Not every failure should be hidden behind a weaker model.

---

# Quality Floor

Each agent may define a minimum model class.

Example:

```text
High-Risk Communication Reviewer:

minimum:
frontier
```

If no qualifying model is available:

```text
assignment blocks
```

It should not silently run using a cheap model.

---

# Cost Floor vs Quality Floor

We optimize between:

```text
quality

latency

cost
```

But **quality requirements are constraints**, not just preferences.

Conceptually:

```text
First:
meet correctness requirements.

Then:
among qualifying choices, optimize latency and cost.
```

---

# Provider Failover

If several providers expose equivalent approved models:

```text
Provider A unavailable
→ Provider B
```

Provider failover should preserve:

```text
model class

required capabilities

structured-output contract

context requirements
```

The provider adapter normalizes differences.

---

# Execution Deadlines

A Work Item or assignment may have a deadline.

Examples:

```text
respond interactively within a useful window

finish background report before 8 AM

send reminder before appointment
```

The Model Router should consider remaining time.

A slow frontier model may be inappropriate when:

```text
deadline is 3 seconds away
```

unless quality policy requires it.

---

# Time Remaining

Conceptually:

```typescript
type ExecutionBudgetRemaining = {
  modelCallsRemaining: number;

  capabilityCallsRemaining: number;

  delegatedAssignmentsRemaining: number;

  tokenBudgetRemaining?: number;

  costBudgetRemaining?: number;

  wallTimeRemainingMs?: number;
};
```

Every execution receives the remaining budget.

---

# Context Budget and Model Selection

Context size affects model selection.

If an assignment requires:

```text
70k tokens
```

a model that supports only:

```text
32k
```

is ineligible.

However, the preferred solution should often be:

```text
better context selection
```

rather than:

```text
always choose the largest context model
```

The Context Distribution System and Model Router should work together.

---

# Output Budget

Models should receive explicit output limits.

A Recipient Resolver may need:

```text
300 tokens
```

not:

```text
8,000 tokens
```

A research synthesis may need much more.

Output limits reduce:

```text
latency

cost

rambling

unexpected behavior
```

---

# Token Budgeting by Role

Example initial policies:

```text
Recipient Resolver:
small input
small output

Email Composer:
moderate input
moderate output

Revenue Analyst:
moderate/large input
structured moderate output

Chief of Staff:
focused broader input
short-to-moderate output

Background Research:
larger budget where justified
```

Real limits should come from production measurement.

---

# Model Versioning

Every execution should record exact:

```text
provider

model

model version when available

routing policy version

agent version

context-builder version
```

This lets us diagnose behavior changes.

---

# Routing Policy Versioning

The Model Router itself should be versioned.

Example:

```text
model_routing_policy v1.4
```

If performance changes after routing updates, we should know.

---

# Model Selection Observability

For every model call, we should answer:

```text
Why did we need a model call?

Why this agent?

Why this model class?

Why this model?

Why this provider?

Was escalation used?

Was fallback used?

How much budget remained?

What latency occurred?

What did the call cost?

Did evaluation accept the result?
```

---

# Routing Metrics

Track:

```text
model calls per user turn

model calls per completed Work Item

model calls per agent type

percentage fast / balanced / frontier

escalation rate

fallback rate

retry rate

structured-output failure rate

model latency

provider latency

cost per Work Item

cost per completed outcome

context tokens per model call

unnecessary-model-call rate
```

The last metric is particularly important.

---

# Latency Attribution

For each Work Item:

```text
ingestion

queue

context construction

model

capability

waiting

approval

user response

provider

verification
```

should be measurable independently.

Otherwise we will repeat the mistake of blaming the model for latency that belongs elsewhere.

---

# Example: Simple Calendar Question

User:

> What do I have tomorrow?

Desired execution:

```text
Chief of Staff
→ balanced/fast decision
→ calendar.read
→ Chief of Staff response
```

Potential budget:

```text
Model calls:
1-2 maximum

Capability calls:
1

Specialists:
0
```

No routing model.

No planner.

No evaluator model.

---

# Example: Recipient Resolution

User:

> Send Martin the report.

Recipient Resolver executes:

```text
Preferred:
fast model

Capability calls:
contacts.search
email.search_correspondents

Model calls:
1
```

If one verified Martin exists:

```text
complete
```

If two plausible candidates remain:

```text
escalate once to balanced
```

If still ambiguous:

```text
blocked
→ user clarification
```

Not:

```text
keep calling models until one guesses
```

---

# Example: Complex Revenue Analysis

Assignment:

> Explain why revenue fell 28% compared with last month.

Execution profile:

```text
complexity:
high

risk:
low

latency:
normal
```

Possible routing:

```text
Revenue Analyst
→ balanced model initially
```

If evidence is complex and the result fails evaluation:

```text
frontier escalation
```

Budget:

```text
maximum model calls:
2

maximum capability calls:
4
```

---

# Example: High-Risk Email

User asks Solos to draft and send a sensitive customer dispute response.

Routing:

```text
Email Composer:
balanced/frontier depending complexity

Sensitive Communication Review:
frontier if policy requires

Send:
deterministic capability
```

We do not need the frontier model to perform the actual send.

The model handles judgment.

The runtime handles action.

---

# Example: Provider Failure

Selected model:

```text
Balanced Model A
```

Provider times out.

Routing policy:

```text
retry:
0 or 1 depending error

fallback:
Balanced Model B
```

If successful:

```text
continue
```

The user does not need to know which provider failed unless it materially affects the work.

---

# Example: Frontier Unavailable

Complex strategic assignment requires:

```text
minimum model class:
frontier
```

All frontier providers unavailable.

The runtime should not silently use Fast.

Result:

```text
assignment blocked:
QUALIFYING_MODEL_UNAVAILABLE
```

The Chief of Staff may:

```text
wait

reduce scope

return a limited partial answer

explain the temporary limitation
```

---

# Example: Budget Pressure

Work Item:

```text
Analyze business performance and prepare a summary.
```

Three optional specialist analyses remain.

Budget can support only one.

The Chief of Staff should decide which contributes most to the objective.

It should not dispatch all three and hope the budget survives.

---

# Initial Routing Strategy

For the first implementation, I would intentionally keep this simple.

```text
Model classes:
Fast / Balanced / Frontier

Router:
Deterministic

Mandatory router model:
None

Chief of Staff default:
Balanced

Chief of Staff complex/high-risk:
Frontier

Specialists:
Definition-specific preferred class

Escalation:
Maximum 1

Fallback:
Within approved policy

Specialist model calls:
Target 1

Routine Chief of Staff decision:
Target 1

Deterministic operations:
0 model calls

Model evaluator:
Only when deterministic evaluation is insufficient

Provider health:
Included in routing

Every model call:
Usage + latency recorded

Every execution:
Finite budget
```

We should earn additional complexity through measured requirements.

---

# Initial Budget Philosophy

I would not hardcode arbitrary production token or dollar numbers before measuring the new runtime.

Instead, initially define:

```text
call limits

delegation limits

parallelism limits

wall-time limits

output limits
```

Then instrument actual:

```text
tokens

cost

latency

success
```

After sufficient real workloads, set cost and token ceilings using observed distributions.

---

# Hard Initial Limits

Some limits should exist immediately.

```text
Maximum specialist delegation depth:
1

Maximum specialists created from one Chief of Staff decision:
3

Maximum specialist model calls per normal execution:
1

Maximum escalation attempts:
1

Unlimited agent loops:
forbidden

Mandatory planning call:
none

Mandatory routing call:
none

Mandatory evaluator model:
none

Provider retries:
bounded

Every execution:
deadline or timeout

Every Work Item:
finite model-call budget
```

---

# Optimization Order

When improving the system, optimize in this order:

```text
1. Correctness

2. Reliability

3. Unnecessary model calls

4. Latency

5. Context size

6. Cost

7. Sophistication
```

Cost should not be reduced by making the system unreliable.

But adding expensive reasoning should also require measurable value.

---

# What We Are Not Building

We are not using one expensive frontier model for everything.

We are not creating a router LLM call before every useful model call.

We are not chaining planner, router, evaluator, and writer models by default.

We are not tying agent identity to provider models.

We are not allowing agents to choose unlimited compute.

We are not allowing unlimited retries.

We are not allowing unlimited escalation.

We are not spending model tokens on deterministic work.

We are not silently downgrading work below its required quality floor.

We are not optimizing cost without measuring completed outcomes.

We are building a controlled intelligence allocation system.

---

# The Standard

Every model execution should answer:

```text
Why is reasoning required?

Why is this agent responsible?

Why is this model class sufficient?

Why was this actual model selected?

What budget governs it?

How many calls remain?

What happens if it fails?

What triggers escalation?

What prevents runaway execution?

Was the result worth the latency and cost?
```

If we cannot answer those questions, the model call should be questioned.

---

# One Sentence

> **The Model Routing and Execution Budget system gives Solos the minimum intelligence necessary for each responsibility while placing hard boundaries around latency, cost, retries, delegation, and computational complexity.**
