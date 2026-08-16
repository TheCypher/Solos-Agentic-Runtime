# Initial Agent Set

## Purpose

The Initial Agent Set defines the smallest useful collection of specialist agents that should exist beneath the Solos Chief of Staff in the first production version.

The goal is not to create an impressive organization chart.

The goal is to create only those specialist roles that provide clear value through:

- narrower context
- better domain reasoning
- reusable expertise
- independent parallel work
- stronger evaluation
- clearer responsibility boundaries

The core principle is:

> **Create an agent only when bounded reasoning creates meaningful value that deterministic code, a capability, or a workflow cannot provide more reliably.**

The initial Solos organization should be deliberately small.

---

# Foundational Rule

Before creating any specialist, ask:

```text
Does this task require judgment?

Is the responsibility reusable?

Can it be bounded?

Can its output be validated?

Does specialization improve reliability or latency?

Would a workflow or capability be simpler?
```

If the answer is no, it should probably not be an agent.

---

# The Initial Organization

The first production organization should look roughly like:

```text
                     User
                      │
                      ▼
              ┌────────────────┐
              │ Chief of Staff │
              └───────┬────────┘
                      │
          ┌───────────┼───────────┬───────────┐
          ▼           ▼           ▼           ▼
   Recipient      Email       Revenue      Schedule
   Resolver       Composer    Analyst      Analyst
          │           │           │           │
          └───────────┴───────────┴───────────┘
                      │
                      ▼
              Capabilities
                      │
                      ▼
              Provider Adapters
```

Potentially one additional specialist:

```text
Research Specialist
```

But I would make that optional for v1 unless research is clearly part of the initial product experience.

---

# Chief of Staff

The Chief of Staff remains the principal intelligence.

It owns:

```text
user relationship

objective formation

message interpretation

Work Item ownership

delegation

coordination

synthesis

approval decisions

completion judgment

user communication
```

It is not one specialist among many.

It is the accountable owner of the whole outcome.

---

# What Should Stay With the Chief of Staff

The Chief of Staff should directly handle:

```text
simple questions

simple follow-ups

conversation continuity

corrections

cancellations

status requests

simple capability reads

workflow selection

approval communication

progress communication

result synthesis

final user-facing communication
```

We should not delegate these merely because specialist agents exist.

---

# Specialist 1: Recipient Resolver

## Purpose

Resolve ambiguous references to people, recipients, businesses, accounts, or contact destinations.

Examples:

```text
"Martin"

"my accountant"

"the customer from yesterday"

"send it to her personal email"

"the other Sarah"
```

This is a very important specialist because incorrect entity resolution can cause high-impact mistakes.

---

# Responsibilities

The Recipient Resolver should:

```text
identify candidate entities

search permitted contact sources

compare candidates with conversational context

use verified historical correspondence

return confidence and evidence

surface unresolved ambiguity
```

---

# Prohibited Responsibilities

It must not:

```text
send communication

create contacts

modify contacts

choose an ambiguous recipient without sufficient evidence

communicate with the user

mark Work Items complete
```

---

# Capabilities

Typical read-only capabilities:

```text
contacts.search

contacts.read

email.search_correspondents

calendar.search_attendees

memory.read_relevant
```

---

# Output

```typescript
type RecipientResolutionResult = {
  status:
    | "resolved"
    | "ambiguous"
    | "not_found";

  candidates: Array<{
    entityId?: string;
    name: string;
    address?: string;
    confidence: number;
    evidence: EvidenceReference[];
  }>;

  recommendedCandidateId?: string;

  unresolvedReason?: string;
};
```

---

# Why This Should Be an Agent

Entity resolution often requires semantic judgment across:

```text
conversation

relationships

email history

contact records

contextual references
```

Simple deterministic lookup is not always sufficient.

But the result remains bounded and evidence-backed.

That makes this a strong agent role.

---

# Specialist 2: Email Composer

## Purpose

Create high-quality written communication from a defined objective and trusted context.

Examples:

```text
draft an email

rewrite a customer response

prepare a follow-up

write a concise update

turn analysis into a professional message
```

---

# Responsibilities

The Email Composer should:

```text
understand communication objective

use verified recipient identity

incorporate supplied facts

match requested tone

preserve required details

avoid unsupported claims

return structured draft
```

---

# Prohibited Responsibilities

It must not:

```text
resolve unknown recipients independently

send email

modify external records

invent financial facts

create new business commitments without instruction

communicate directly with the user
```

---

# Capabilities

Potentially:

```text
email.read_thread

memory.read_relevant
```

Mostly read-only.

It should often receive upstream artifacts rather than perform broad retrieval itself.

---

# Output

```typescript
type EmailDraftResult = {
  recipient: ResolvedRecipientReference;

  subject: string;

  body: string;

  factsUsed: EvidenceReference[];

  unresolvedQuestions: string[];

  sensitivity:
    | "routine"
    | "important"
    | "sensitive";
};
```

---

# Why This Should Be an Agent

Writing communication is inherently judgment-heavy.

Tone, prioritization, framing, concision, and relationship context matter.

It is reusable and bounded.

It should therefore be an agent.

---

# Why "Email Sender" Should Not Be an Agent

Sending an email requires no reasoning once the exact message is approved.

It should be:

```text
email.send capability
```

not:

```text
Email Sender Agent
```

The architecture should remain:

```text
Email Composer
→ accepted draft
→ policy/approval
→ deterministic email.send
→ provider evidence
```

---

# Specialist 3: Revenue Analyst

## Purpose

Analyze normalized financial and operational revenue data and return supported business insights.

Examples:

```text
How much did I make last week?

Why was July lower than June?

Which services are driving revenue?

Where did revenue drop?

What changed this month?
```

---

# Responsibilities

The Revenue Analyst should:

```text
calculate requested metrics

compare time periods

identify important changes

identify anomalies

group results where useful

distinguish fact from inference

explain limitations

attach supporting evidence
```

---

# Prohibited Responsibilities

It must not:

```text
issue refunds

move money

modify transactions

provide definitive tax advice

provide unsupported causal explanations as fact

send reports externally

communicate with the user directly
```

---

# Capabilities

Read-only:

```text
revenue.get_summary

revenue.get_transactions

customers.read_aggregate

appointments.read_aggregate

business.read_service_catalog
```

---

# Output

```typescript
type RevenueAnalysisResult = {
  summary: string;

  metrics: RevenueMetric[];

  findings: SupportedFinding[];

  anomalies: RevenueAnomaly[];

  hypotheses: Hypothesis[];

  limitations: string[];

  evidence: EvidenceReference[];
};
```

---

# Why This Should Be an Agent

The arithmetic should be deterministic.

But determining:

```text
what matters

what changed

what deserves attention

how multiple trends relate

what the business implication may be
```

requires judgment.

Therefore:

```text
data retrieval + arithmetic
→ deterministic

interpretation + synthesis
→ Revenue Analyst
```

This separation is important.

---

# Specialist 4: Schedule Analyst

## Purpose

Reason about calendars, appointments, availability, timing constraints, and scheduling implications.

Examples:

```text
Can I meet Martin tomorrow?

What's my day look like?

Where do I have conflicts?

When can I fit this client in?

Which appointment should I move?
```

---

# Responsibilities

The Schedule Analyst should:

```text
analyze events

identify conflicts

interpret availability

compare possible time windows

consider business hours

consider appointment duration

identify scheduling constraints

return supported options
```

---

# Prohibited Responsibilities

Initially, it must not:

```text
create events

cancel events

reschedule clients

invite attendees

send messages

modify provider calendars
```

Those should remain deterministic operations behind policy.

---

# Capabilities

Read-only:

```text
calendar.list_events

calendar.find_availability

appointments.list

business.read_hours
```

---

# Output

```typescript
type ScheduleAnalysisResult = {
  requestedWindow: DateRange;

  availableWindows: TimeWindow[];

  conflicts: ScheduleConflict[];

  recommendedOptions: TimeWindow[];

  constraints: string[];

  evidence: EvidenceReference[];
};
```

---

# Why This Should Be an Agent

Basic availability can often be deterministic.

But questions like:

```text
"What's the best time?"

"Can I fit this around my current appointments?"

"Which option would be least disruptive?"
```

require judgment.

The capability should calculate availability.

The Schedule Analyst should interpret it.

---

# Why "Calendar Manager" Should Not Exist Initially

A broad Calendar Manager Agent would probably have:

```text
read calendar

create events

change events

delete events

reason about schedules

contact attendees
```

That is too much responsibility.

Instead:

```text
Schedule Analyst
→ reasoning

Calendar Capability
→ deterministic write
```

This is safer and easier to debug.

---

# Specialist 5: Research Specialist

## Status

Optional for initial production.

I would include it only if early Solos product workflows require meaningful external or document research.

---

# Purpose

Answer bounded questions requiring retrieval and synthesis across multiple external sources.

Examples:

```text
research competitor pricing

compare software options

find local regulations

review a supplied document set

prepare background for a meeting
```

---

# Responsibilities

```text
retrieve relevant sources

rank source quality

extract relevant facts

identify disagreement

synthesize findings

cite evidence

state uncertainty
```

---

# Prohibited Responsibilities

```text
take external action

send communication

modify business systems

present unsupported facts

expand research scope indefinitely
```

---

# Output

```typescript
type ResearchResult = {
  answer: string;

  findings: SupportedFinding[];

  sources: EvidenceReference[];

  uncertainties: string[];

  followUpQuestions: string[];
};
```

---

# Why This May Be an Agent

Research is open-ended enough to benefit from specialization.

But it can also become:

```text
expensive

slow

unbounded

context-heavy
```

So I would not make it part of every workflow.

---

# Agents We Should Not Build Yet

The most important part of the Initial Agent Set may be what we deliberately do not create.

---

# No Router Agent

We should not create:

```text
Intent Router Agent
```

whose only job is:

```text
user message
→ classify
→ Chief of Staff
```

The Chief of Staff should understand the message itself.

A deterministic helper or lightweight classifier may exist where useful.

But no mandatory model hop should sit before the Chief of Staff.

---

# No Planner Agent

We should not create:

```text
Planner Agent
```

by default.

The Chief of Staff owns objective decomposition.

A separate planner introduces:

```text
extra latency

split responsibility

duplicated reasoning

context divergence
```

If highly complex workflows eventually justify specialized planning, we can revisit it.

---

# No Tool-Selection Agent

We should not build:

```text
Tool Router Agent
```

The capability surface should already be bounded by:

```text
Agent Definition

Assignment

Capability Grants

Capability Registry
```

The active agent can choose among the few relevant capabilities it receives.

---

# No Memory Agent in the Execution Path

We should not require a dedicated Memory Agent every turn.

Memory retrieval should mostly be a subsystem of the Context Distribution System.

Potential memory extraction or consolidation processes may run asynchronously or as bounded background work.

But they should not become mandatory orchestration hops.

---

# No Verification Agent by Default

We should not create:

```text
Verifier Agent
```

for every result.

Evaluation should begin with:

```text
schemas

deterministic validation

evidence checks

policy checks
```

A model evaluator may be used only when subjective evaluation is required.

---

# No Send Agent

Do not create:

```text
Email Sender Agent

Text Sender Agent

Calendar Writer Agent
```

These should be capabilities.

Reasoning and action must remain separate.

---

# No Generic "Business Agent"

A specialist such as:

```text
Business Agent
```

is too broad.

It would duplicate the Chief of Staff and gradually accumulate every capability.

Avoid vague agents.

---

# No Customer Operations Agent Yet

A Customer Operations specialist may eventually become valuable.

Potential responsibilities:

```text
customer segmentation

customer history analysis

follow-up prioritization

relationship analysis
```

But initial needs may be adequately covered by:

```text
Chief of Staff

Recipient Resolver

Revenue Analyst

capabilities
```

We should add it when repeated workflows demonstrate a real boundary.

---

# No Marketing Agent Yet

Similarly:

```text
Marketing Agent
```

is too broad for v1.

Marketing eventually may decompose into:

```text
Campaign Analyst

Content Composer

Audience Analyst
```

But only after product usage creates a clear need.

---

# No Autonomous Agent Factory

The Chief of Staff should not invent:

```text
"Let me create a Cancellation Analysis Agent."
```

It should select from registered roles.

If an unusual task doesn't fit a specialist:

```text
Chief of Staff may reason directly
```

or:

```text
use a general Research Specialist
```

where appropriate.

We do not need a new agent for every noun.

---

# Workflows, Not Agents

Many recurring business behaviors should be workflows.

Examples:

```text
Morning Brief

Weekly Revenue Report

Appointment Follow-Up

Overdue Invoice Follow-Up

Daily Schedule Review

Customer Re-engagement

Weekly Business Review
```

A workflow coordinates:

```text
triggers

agents

capabilities

approvals

waits

verification
```

It does not require a permanent "Morning Brief Agent."

---

# Example: Morning Brief

Workflow:

```text
Scheduled Trigger
       │
       ├── Calendar Capability
       │
       ├── Revenue Capability
       │
       ├── Email Capability
       │
       └── Active Work Retrieval
                  │
                  ▼
            Chief of Staff
                  │
                  ▼
             Brief Output
```

Potentially a specialist may assist with analysis.

But there is no reason to create:

```text
Morning Brief Agent
```

unless that role eventually needs substantial unique reasoning.

---

# Example: Send Martin Revenue Report

User:

> Send Martin the July report.

Internal organization:

```text
                     Chief of Staff
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
      Recipient Resolver      Revenue Analyst
               │                     │
               └──────────┬──────────┘
                          ▼
                    Email Composer
                          │
                          ▼
                    Policy Engine
                          │
                          ▼
                       Approval
                          │
                          ▼
                  email.send capability
                          │
                          ▼
                       Gmail
```

Notice:

```text
3 agents reason

1 deterministic operation sends
```

This is exactly the architecture we want.

---

# Example: "What Do I Have Tomorrow?"

User:

> What do I have tomorrow?

We should not invoke Schedule Analyst automatically.

Possible flow:

```text
Chief of Staff
→ calendar.list_events
→ summarize
```

If user asks:

> Can I fit a 90-minute meeting with Martin tomorrow without making the day too packed?

Then:

```text
Chief of Staff
→ Schedule Analyst
→ availability interpretation
→ response
```

Specialization is invoked only when it adds value.

---

# Example: "How Much Did I Make Last Week?"

Simple case:

```text
Chief of Staff
→ revenue.get_summary
→ response
```

No Revenue Analyst required.

Complex:

> Why was last week so much worse than the week before?

Then:

```text
Chief of Staff
→ Revenue Analyst
→ supported analysis
→ response
```

This distinction protects latency and cost.

---

# Agent Invocation Policy

An agent should be invoked when at least one of these is true:

```text
specialized interpretation is required

the output requires substantial synthesis

the assignment can execute independently

narrow context improves reliability

the responsibility recurs frequently

the result has a clear evaluation contract
```

An agent should usually not be invoked when:

```text
one deterministic capability answers the question

the Chief of Staff can answer directly

the action is purely mechanical

the workflow is already defined

delegation would duplicate reasoning

the output cannot be meaningfully validated
```

---

# Agent Granularity

Agent roles should be neither too broad nor too narrow.

Too broad:

```text
Business Agent
Operations Agent
Communication Agent
```

Too narrow:

```text
Tuesday Revenue Percentage Agent
Subject Line Agent
Calendar Conflict Agent
```

Good granularity describes a reusable reasoning responsibility:

```text
Recipient Resolver
Email Composer
Revenue Analyst
Schedule Analyst
Research Specialist
```

---

# Agent Boundaries

A good specialist has:

```text
one coherent domain

one recognizable responsibility

a bounded input contract

a bounded output contract

limited capability access

clear failure behavior

clear evaluation
```

If responsibilities cannot be summarized in one or two sentences, the agent may be too broad.

---

# Agent Authority Summary

Initial recommendation:

```text
Chief of Staff
Broad read eligibility
May propose writes
May delegate
User-facing
No policy bypass

Recipient Resolver
Read-only
No delegation
No user communication

Email Composer
Read-only
No send
No delegation
No user communication

Revenue Analyst
Read-only
No financial writes
No delegation
No user communication

Schedule Analyst
Read-only
No calendar writes
No delegation
No user communication

Research Specialist
Read-only external/document retrieval
No delegation
No user communication
```

This creates a very safe first organization.

---

# Agent Communication Rules

Specialists should not directly talk to each other.

Bad:

```text
Revenue Analyst ↔ Email Composer
```

Instead:

```text
Revenue Analyst
      ↓
Accepted Result Artifact
      ↓
Runtime
      ↓
Email Composer
```

Communication happens through:

```text
assignments

accepted results

context artifacts

Work Graph dependencies
```

Not freeform inter-agent conversation.

---

# Agent Lifecycle

Every specialist follows the same lifecycle:

```text
Registered Definition
      ↓
Assignment Created
      ↓
Instance Created
      ↓
Context Built
      ↓
Execution
      ↓
Capability Calls if permitted
      ↓
Structured Result
      ↓
Evaluation
      ↓
Accepted / Revised / Rejected
      ↓
Instance Ends
```

Agents are ephemeral.

The organization is durable.

---

# Specialist State

Specialists should not maintain private long-lived state.

If a specialist discovers something important:

```text
result artifact

evidence

memory proposal

work update
```

must be persisted through the runtime.

Then the agent instance may disappear.

---

# Memory Writing

No specialist should directly write global memory.

Instead:

```text
Specialist
→ proposes memory candidate
→ Memory subsystem evaluates
→ accepted memory persisted
```

Example:

Email Composer notices:

```text
User prefers very concise investor emails.
```

It may propose this as a memory candidate.

It should not silently alter durable user memory.

---

# Agent Model Defaults

An initial model strategy could be:

```text
Chief of Staff
Balanced default
Frontier for high complexity

Recipient Resolver
Fast default
Balanced escalation

Email Composer
Balanced default
Frontier for sensitive/high-stakes content

Revenue Analyst
Balanced default
Frontier escalation

Schedule Analyst
Fast or Balanced depending reasoning depth

Research Specialist
Balanced/Frontier depending scope
```

The Model Router chooses actual models dynamically.

---

# Parallel Execution

The initial organization supports useful parallelism.

Example:

```text
             Chief of Staff
             /            \
            /              \
Recipient Resolver      Revenue Analyst
            \              /
             \            /
              Email Composer
```

This is safe because the first two agents are read-only and independent.

The write happens only after their results are accepted.

---

# Agent Failure Isolation

A specialist failure should not collapse the entire system.

Example:

```text
Recipient Resolver:
completed

Revenue Analyst:
failed
```

The Chief of Staff may:

```text
retry Revenue Analyst

use direct revenue capability

reduce analysis scope

ask user

continue with partial result
```

The successful recipient resolution remains reusable.

---

# Agent Replacement

Because agents are governed by contracts, implementations should be replaceable.

Example:

```text
Revenue Analyst v1
→ replaced by
Revenue Analyst v2
```

Downstream systems still consume:

```text
RevenueAnalysisResult
```

This allows us to improve prompts, models, and internal strategies without redesigning the runtime.

---

# Agent Version Rollout

New agent versions should pass:

```text
offline evals

behavioral regression tests

shadow execution where useful

limited rollout

production metrics
```

before becoming active defaults.

The registry controls activation.

---

# Agent Metrics

Each specialist should have measurable performance.

Examples:

```text
assignment acceptance rate

successful completion rate

evaluation rejection rate

revision rate

latency

model calls

capability calls

cost

evidence completeness

escalation rate

user-visible downstream failures
```

An agent should justify its existence through measurable value.

---

# Agent Deletion Criteria

Agents should also be removable.

If an agent:

```text
adds latency

rarely improves outcomes

duplicates Chief of Staff reasoning

causes frequent evaluation failures

has no clear reusable boundary
```

we should remove it.

The architecture should not accumulate permanent agent roles simply because they once seemed useful.

---

# Initial Agent Registry

The first active registry might contain:

```text
chief_of_staff
v1.0.0

recipient_resolver
v1.0.0

email_composer
v1.0.0

revenue_analyst
v1.0.0

schedule_analyst
v1.0.0
```

Optional:

```text
research_specialist
v1.0.0
```

That is enough to build meaningful multi-agent behavior without creating unnecessary complexity.

---

# Phase 1 Organization

For the first production release:

```text
Chief of Staff

Recipient Resolver

Email Composer

Revenue Analyst

Schedule Analyst
```

Read-heavy specialists.

Deterministic writes.

Delegation depth of one.

---

# Phase 2 Potential Specialists

Only after production evidence:

```text
Customer Analyst

Business Performance Analyst

Document Analyst

Research Specialist

Communication Reviewer
```

These should be introduced only because repeated tasks justify them.

---

# Phase 3 Potential Coordinators

Much later, if Solos handles significantly larger objectives:

```text
Meeting Preparation Coordinator

Customer Lifecycle Coordinator

Financial Operations Coordinator
```

These could potentially manage sub-assignments.

But that would require revisiting:

```text
delegation depth

authority

budget

context sharing

write coordination
```

We should not start there.

---

# The Initial Agent Organization in Full

```text
┌──────────────────────────────────────────────────────┐
│                    Solos Runtime                     │
│                                                      │
│                 ┌────────────────┐                   │
│                 │ Chief of Staff │                   │
│                 └───────┬────────┘                   │
│                         │                            │
│          bounded Agent Assignments                   │
│                         │                            │
│     ┌───────────────────┼────────────────────┐       │
│     │                   │                    │       │
│     ▼                   ▼                    ▼       │
│ Recipient          Email Composer       Revenue      │
│ Resolver                                Analyst      │
│     │                   │                    │       │
│     └───────────────────┼────────────────────┤       │
│                         │                    │       │
│                         ▼                    ▼       │
│                  Schedule Analyst    Research*       │
│                                         optional     │
│                                                      │
│              accepted structured results             │
│                         │                            │
│                         ▼                            │
│                 Chief of Staff                       │
│                         │                            │
│                         ▼                            │
│           Policies / Work Graph / Runtime            │
│                         │                            │
│                         ▼                            │
│                  Capabilities                        │
│                         │                            │
│                         ▼                            │
│                Provider Adapters                     │
└──────────────────────────────────────────────────────┘
```

The organization is asymmetric by design.

The Chief of Staff is the only generalist.

Everyone else is narrow.

---

# Design Principle: One Brain, Many Bounded Experts

The product should not feel like:

```text
a swarm of autonomous agents
```

It should feel like:

```text
one exceptional Chief of Staff
with access to specialized internal expertise
```

This is closer to how a real high-functioning Chief of Staff operates.

The principal owns:

```text
judgment

priorities

coordination

communication
```

Specialists provide:

```text
focused expertise
```

The runtime provides:

```text
discipline
```

---

# What the User Sees

User:

> Send Martin the July numbers and make it concise.

Internally:

```text
Recipient Resolver
Revenue Analyst
Email Composer
Policy Engine
Email Capability
Gmail Adapter
Evidence
```

Externally:

> Sent Martin the July report. I kept it concise.

That contrast is the product.

Internal complexity.

External simplicity.

---

# What We Are Not Building

We are not building an agent for every capability.

We are not building an agent for every workflow.

We are not building a separate planner, router, verifier, and tool-selector by default.

We are not building a hierarchy of autonomous managers.

We are not allowing agents to freely communicate with each other.

We are not allowing specialists to inherit broad Chief of Staff authority.

We are not allowing specialists to own user relationships.

We are not creating agent roles because they sound sophisticated.

We are building the smallest useful organization of bounded experts beneath one accountable Chief of Staff.

---

# The Standard

Every proposed new agent must answer:

```text
What unique reasoning responsibility does it own?

Why can't the Chief of Staff handle this directly?

Why isn't this a deterministic capability?

Why isn't this a workflow?

Can the responsibility be bounded?

Can the result be evaluated?

Does narrow context improve performance?

Will this role be reused often enough to justify existing?

What authority does it actually require?
```

If those questions cannot be answered clearly, do not create the agent.

---

# Initial Production Set

My recommendation:

```text
Required:

Chief of Staff
Recipient Resolver
Email Composer
Revenue Analyst
Schedule Analyst
```

Optional:

```text
Research Specialist
```

Everything else should initially remain:

```text
Chief of Staff reasoning

deterministic capabilities

registered workflows

runtime infrastructure
```

---

# One Sentence

> **The initial Solos agent organization should consist of one accountable Chief of Staff supported by a very small number of read-heavy, bounded specialists whose outputs are structured, evaluated, and converted into real-world action only through the runtime.**
