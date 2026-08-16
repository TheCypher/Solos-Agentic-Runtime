# Policies, Approvals, and Authority

## Purpose

The Policies, Approvals, and Authority layer defines **what Solos is allowed to do**.

It governs:

- what the Chief of Staff may do autonomously
- what specialist agents may access
- which capabilities may execute
- when user approval is required
- how risky actions are classified
- how permissions are scoped
- how approvals are bound to exact actions
- how authority expires or is revoked
- how business rules override agent intent
- how policy violations are prevented
- how every authorization decision is recorded

The core principle is:

> **Models may request actions. Policy determines whether those actions are allowed.**

No agent can grant itself authority.

No prompt can override policy.

---

# Foundational Distinction

The system must distinguish among:

```text
Intent
Permission
Authority
Approval
Execution
```

These are not the same thing.

---

# Intent

Intent describes what an agent wants to do.

Example:

```text
Send the July report to Martin.
```

Intent alone gives no permission.

---

# Permission

Permission describes whether a role is generally eligible to perform an operation.

Example:

```text
Chief of Staff may propose email.send.

Revenue Analyst may not.
```

Permission is role-level eligibility.

---

# Authority

Authority describes whether this specific execution is allowed under current circumstances.

It may depend on:

- user identity
- business
- connected account
- resource
- assignment
- operation
- risk level
- current Work Item
- time
- approval state

Authority is contextual.

---

# Approval

Approval is explicit consent for an action that policy says requires user authorization.

Example:

```text
Send this exact email
to this exact recipient
with this exact content.
```

Approval does not mean unlimited permission to perform related actions later.

---

# Execution

Execution is the actual capability operation.

It may occur only after authority has been established.

---

# The Core Rule

> **Intent never implies authority.**

Even if the Chief of Staff confidently decides:

```text
This email should be sent.
```

the runtime must still verify:

```text
Is sending allowed?

Is this recipient permitted?

Is approval required?

Is approval valid?

Has the action changed?

Is the Work Item still active?

Is the provider connection authorized?

Has this operation already happened?
```

Only then may execution proceed.

---

# Policy Engine

The Policy Engine is the authoritative runtime component responsible for evaluating authorization and approval requirements.

Conceptually:

```text
Action Request
     ↓
Identity
     ↓
Agent / Workflow Authority
     ↓
Assignment Scope
     ↓
User Authorization
     ↓
Business Policy
     ↓
Risk Policy
     ↓
Resource Policy
     ↓
Approval Policy
     ↓
Runtime Safety Policy
     ↓
Decision
```

The result should be deterministic wherever possible.

---

# Policy Decision

Every evaluated action should produce a structured decision.

```typescript
type PolicyDecision =
  | {
      decision: "allow";
      effectiveAuthority: EffectiveAuthority;
    }
  | {
      decision: "approval_required";
      approvalRequirement: ApprovalRequirement;
    }
  | {
      decision: "deny";
      reasonCode: PolicyDenialCode;
      reason: string;
    }
  | {
      decision: "allow_with_constraints";
      effectiveAuthority: EffectiveAuthority;
      constraints: PolicyConstraint[];
    };
```

Policy decisions should never exist only in logs or model reasoning.

---

# Authority Hierarchy

Authority should be narrowed through multiple layers.

```text
System policy
      ↓
Business policy
      ↓
User authorization
      ↓
Role eligibility
      ↓
Assignment / workflow grant
      ↓
Resource scope
      ↓
Approval
      ↓
Runtime preconditions
      =
Effective authority
```

Each layer may restrict authority.

No lower layer may expand authority beyond a higher layer.

---

# Effective Authority

Conceptually:

```typescript
type EffectiveAuthority = {
  subject: AuthoritySubject;

  capability: string;
  operations: string[];

  businessId: string;
  userId: string;

  accountScope?: string[];
  resourceScope?: ResourceScope;

  constraints: AuthorityConstraint[];

  approvalId?: string;

  issuedAt: string;
  expiresAt?: string;
};
```

This object describes exactly what the runtime may execute.

---

# Authority Subjects

Authority may be granted to:

```text
Chief of Staff

Specialist Assignment

Registered Workflow

Runtime Recovery Process

Internal System Process
```

A specialist should generally receive authority through its assignment.

Not simply because the agent role exists.

---

# Agent Eligibility

The Agent Definition determines maximum possible authority.

Example:

```text
Revenue Analyst

Eligible:
revenue.read_summary
revenue.read_transactions

Never eligible:
email.send
payments.refund
calendar.delete_event
```

Even if the Chief of Staff requests:

```text
Give Revenue Analyst permission to send this email.
```

the runtime must reject it.

Delegation cannot increase an agent's registered authority.

---

# Assignment Authority

The assignment narrows authority further.

Example:

```text
Revenue Analyst Definition:

Eligible:
revenue.read_summary
revenue.read_transactions
customers.read_aggregate
```

Specific assignment:

```text
Granted:
revenue.read_summary

Not granted:
revenue.read_transactions
customers.read_aggregate
```

Effective authority:

```text
revenue.read_summary only
```

---

# User Authorization

A user's connected services define another authority boundary.

Example:

```text
Solos has access to:

Business Gmail
Personal Google Calendar
Square Business Account
```

A Work Item may only use accounts the user has authorized.

The runtime should not treat:

```text
Google connected
```

as equivalent to:

```text
every Google account and resource authorized.
```

---

# Business Authority

For users with multiple businesses or workspaces:

```text
Business A
Business B
Personal
```

authority must remain scoped.

A Work Item for Business A should not automatically access Business B.

The business context should be resolved before external operations execute.

---

# Resource Authority

Permissions should be scoped to specific resources whenever practical.

Examples:

```text
one email thread

one recipient

one calendar

one appointment

one customer

one transaction

one draft

one date range
```

Example:

```typescript
{
  capability: "email",
  operation: "send",
  resourceScope: {
    draftId: "draft_123",
    recipient: "martin@example.com"
  }
}
```

Narrow authority reduces the damage from mistakes.

---

# Risk Classification

Every capability operation should have a risk class.

Initial classes:

```typescript
type ActionRiskClass =
  | "informational"
  | "read_sensitive"
  | "low_risk_write"
  | "reversible_write"
  | "external_communication"
  | "financial"
  | "destructive"
  | "high_consequence";
```

---

# Informational

Examples:

```text
calculate revenue percentage

summarize already-loaded data

inspect Work Item state
```

Generally no approval required.

---

# Sensitive Read

Examples:

```text
read email

read customer information

read financial transactions

read private calendar
```

Usually no per-action approval after connection, but requires:

```text
explicit account authorization

proper resource scope

auditability

privacy policy enforcement
```

---

# Low-Risk Write

Examples:

```text
create unsent draft

save internal note

create internal task
```

Often autonomous.

---

# Reversible Write

Examples:

```text
create calendar event

update appointment

add label

archive email
```

May be autonomous under user policy where easily reversible.

---

# External Communication

Examples:

```text
send email

send text

send customer message

publish communication
```

Higher reputational risk.

Requires stronger approval policy.

---

# Financial

Examples:

```text
refund payment

issue credit

charge payment

transfer money
```

Initially should require explicit user approval.

---

# Destructive

Examples:

```text
delete customer

delete appointment permanently

delete records

remove provider data
```

Initially require explicit approval or remain disabled.

---

# High-Consequence

Examples may include actions involving:

```text
legal commitments

employment decisions

high-value financial commitments

public statements

major business account changes
```

These should have the strongest policy restrictions.

---

# Default Autonomy Model

Solos should not begin with maximum autonomy.

Autonomy should expand through demonstrated trust.

A reasonable initial model:

```text
Reads:
Autonomous within authorized scope.

Internal analysis:
Autonomous.

Draft creation:
Autonomous.

External sends:
Approval according to user policy.

Calendar creation/update:
Conditional approval.

Financial writes:
Explicit approval.

Destructive actions:
Explicit approval or disabled.

High-consequence actions:
Explicit approval.
```

---

# User Autonomy Preferences

Users should eventually be able to configure how much authority Solos has.

Example:

```text
Email sends:

Always ask

Ask for new recipients only

Ask for sensitive messages only

Send routine messages autonomously
```

The policy system must translate these preferences into enforceable rules.

Not merely prompt instructions.

---

# Autonomy Profiles

Conceptually:

```typescript
type AutonomyProfile = {
  email: CommunicationAuthorityPolicy;
  calendar: CalendarAuthorityPolicy;
  payments: FinancialAuthorityPolicy;
  customers: CustomerAuthorityPolicy;
  messaging: CommunicationAuthorityPolicy;
};
```

Examples:

```text
Conservative

Balanced

High autonomy
```

But the underlying authority should remain operation-specific.

A simple UI profile should compile into explicit policies.

---

# Policy Precedence

Policy conflicts need deterministic resolution.

A recommended precedence:

```text
Platform safety policy

Legal / compliance restrictions

Business policy

User explicit current instruction

User stored autonomy preference

Workflow policy

Agent role policy

Assignment grant

Model proposal
```

Higher layers override lower ones.

A model can never override policy.

---

# Explicit User Instruction

A current explicit instruction may grant authority where policy permits.

Example:

```text
User:
Send that email to Martin.
```

This may count as authorization for:

```text
this email

this recipient

this Work Item
```

But should not automatically grant:

```text
future emails

different content

different recipients

different Work Items
```

User instructions should be scoped.

---

# Approval Model

An approval should be a durable object.

```typescript
type ApprovalRequest = {
  id: string;

  workItemId: string;
  workNodeId: string;

  capability: string;
  operation: string;

  actionSummary: string;

  actionPayloadReference: string;
  payloadHash: string;

  riskClass: ActionRiskClass;

  requestedBy: string;

  status:
    | "pending"
    | "approved"
    | "denied"
    | "expired"
    | "revoked"
    | "cancelled";

  requestedAt: string;
  resolvedAt?: string;
  expiresAt?: string;
};
```

Approval should be persisted.

It should survive deployments and worker restarts.

---

# Approval Must Bind to the Action

Approval should be bound to the exact material action.

For email:

```text
Recipient

Subject

Body

Attachments

Sending account
```

For appointment:

```text
Customer

Date

Time

Duration

Service

Location
```

For financial action:

```text
Transaction

Amount

Currency

Recipient

Reason
```

The runtime should create a canonical payload representation and hash it.

---

# Payload Binding

Example:

```text
Approval:

Capability:
email.send

Recipient:
martin@example.com

Subject:
July Revenue Report

Body hash:
8fa21...

Account:
business@gmail.com
```

If a material part changes:

```text
recipient changes

body changes materially

attachment changes

account changes
```

the approval becomes stale.

---

# Approval Scope

Approvals may have different scopes.

## Single Action

```text
Approve this exact email.
```

Most restrictive.

---

## Bounded Batch

```text
Approve sending these 12 appointment reminders.
```

Should bind to an explicit batch.

---

## Temporary Policy Grant

Example:

```text
For the next hour, send routine appointment confirmations
without asking.
```

More advanced and should be tightly scoped by:

```text
operation

recipient class

time

business

risk level

maximum action count
```

---

# Standing Authority

Eventually, users may grant standing authority.

Example:

```text
You can automatically confirm appointments
that meet these conditions.
```

This should be represented as a policy grant.

```typescript
type StandingAuthorityGrant = {
  id: string;

  userId: string;
  businessId: string;

  capability: string;
  operations: string[];

  conditions: PolicyCondition[];

  limits: AuthorityLimit[];

  createdAt: string;
  expiresAt?: string;

  status:
    | "active"
    | "revoked"
    | "expired";
};
```

Standing authority should never be hidden inside memory.

---

# Action Limits

Authority can include quantitative limits.

Examples:

```text
maximum refund: $50

maximum one-time spend: $25

maximum 10 outbound messages per hour

maximum 5 automatic appointment changes per day
```

Conceptually:

```typescript
type AuthorityLimit =
  | MonetaryLimit
  | FrequencyLimit
  | QuantityLimit
  | TimeWindowLimit
  | RecipientLimit;
```

This allows safe incremental autonomy.

---

# Approval Expiration

Approvals should expire.

Reasons:

```text
external state may change

user intent may change

recipient may change

price may change

schedule may change
```

A send approval might remain valid for a reasonable bounded period.

A payment authorization may require much shorter validity.

Expiration should be policy-specific.

---

# Approval Revocation

The user must be able to revoke pending approval.

Example:

```text
Actually don't send that.
```

Runtime:

```text
Approval → revoked

dependent send node → cancelled
```

If the operation already occurred, revocation cannot undo reality.

Any reversal must be separate work.

---

# Approval Communication

The Chief of Staff should present approvals in human terms.

Bad:

> Approve capability operation email.send with payload hash 8427?

Better:

> I have the July report ready for Martin at his personal email. Want me to send it?

The runtime still binds the approval to the exact action underneath.

---

# Approval Resolution

The user's response should map to the durable approval.

Example:

```text
Chief of Staff:
Want me to send it?

User:
Yes.
```

The runtime should resolve:

```text
which pending approval

which action

which Work Item
```

before approving.

A generic "yes" should not accidentally approve multiple unrelated actions.

---

# Ambiguous Approval

If more than one pending approval could match:

```text
Email send approval

Calendar change approval
```

the runtime should not guess.

The Chief of Staff should clarify which action the user means.

---

# Denials

Approval denial should be explicit runtime state.

```text
Approval → denied
```

Dependent work should react according to the graph.

Example:

```text
Compose Email    completed
Approval         denied
Send             cancelled
```

The Work Item may still remain useful if the user only wants the draft.

---

# Policy Conditions

Conditional policies may evaluate facts such as:

```text
recipient is known

recipient is internal

amount below threshold

message classified routine

event is reversible

action directly requested by user

recipient was previously approved

content has not changed

business hours

resource owner matches business
```

Conceptually:

```typescript
type PolicyCondition = {
  attribute: string;
  operator:
    | "equals"
    | "not_equals"
    | "less_than"
    | "greater_than"
    | "contains"
    | "in";

  value: unknown;
};
```

---

# Policy Rules

A policy rule might look like:

```typescript
type PolicyRule = {
  id: string;

  appliesTo: {
    capability: string;
    operation?: string;
  };

  conditions: PolicyCondition[];

  effect:
    | "allow"
    | "deny"
    | "require_approval";

  priority: number;
};
```

Rules should be machine-enforceable.

---

# Example: Email Send Policy

```text
IF:
operation = email.send

AND:
recipient is already verified

AND:
user explicitly requested this exact send

AND:
message content has not materially changed

THEN:
allow
```

Alternatively:

```text
IF:
recipient is new

THEN:
require approval
```

And:

```text
IF:
recipient unresolved

THEN:
deny
```

---

# Example: Refund Policy

```text
IF:
operation = payments.refund

THEN:
require approval
```

Additional rule:

```text
IF:
refund > original transaction amount

THEN:
deny
```

Additional rule:

```text
IF:
transaction not verified

THEN:
deny
```

---

# Authority Escalation

An agent may request more authority.

It should never grant it to itself.

Example:

```text
Revenue Analyst wants transaction-level data.
```

It returns:

```typescript
{
  status: "blocked",
  requiredResolution: [{
    type: "authority_request",
    capability: "revenue.read_transactions",
    reason: "Needed to identify service-level decline."
  }]
}
```

The runtime evaluates whether the assignment can receive that grant.

---

# Capability Escalation

The Authority layer should determine whether escalation is:

```text
automatically allowed

requires Chief of Staff judgment

requires user approval

prohibited
```

This provides controlled progressive access.

---

# Delegation Authority

The Chief of Staff may delegate only to registered agents allowed by its delegation policy.

Example:

```text
Chief of Staff
→ Revenue Analyst
```

Allowed.

```text
Revenue Analyst
→ Email Sender
```

Not allowed in the initial architecture.

The runtime enforces delegation authority independently from prompts.

---

# Agent Cannot Pass Authority

Authority should not automatically propagate between agents.

Example:

```text
Chief of Staff has authority to propose email sends.
```

If it delegates to Email Composer:

```text
Email Composer receives draft-related permissions only.
```

It does not inherit Chief of Staff's email-send authority.

---

# Least Authority

The system should follow the principle:

> **Every participant receives only the minimum authority required for its current responsibility.**

Not:

> Give the agent everything it might possibly need.

Least authority improves:

```text
security

reliability

tool selection

debugging

blast-radius containment
```

---

# Runtime Preconditions

Policy authorization at planning time is not sufficient.

Consequential actions should recheck authority immediately before execution.

Example:

```text
Approved email send.
```

Before send:

```text
Work Item still active?

Approval still valid?

Recipient unchanged?

Draft unchanged?

Connected account still authorized?

Operation not already executed?

Policy still permits action?
```

If any precondition fails:

```text
do not execute
```

---

# Policy Changes During Work

Policy may change while a Work Item waits.

Example:

```text
Monday:
email send permitted with standing authority.

Tuesday:
user revokes automatic sending.

Wednesday:
waiting Work Item resumes.
```

The action must use Wednesday's valid policy state.

Old authority should not be assumed.

---

# Authority Revocation

Authority may be revoked because of:

```text
user preference change

OAuth revocation

business policy update

security event

agent disablement

capability disablement

account removal
```

Revocation should prevent future operations immediately.

Running operations should be cancelled where possible.

---

# Emergency Disablement

The runtime should support kill switches.

Examples:

```text
Disable email.send globally.

Disable all financial writes.

Disable one provider adapter.

Disable one agent type.

Disable autonomous sends for one business.

Disable one user's external writes.
```

Emergency controls should not require code changes where possible.

---

# Policy Denials

Denials should be typed.

```typescript
type PolicyDenialCode =
  | "AGENT_NOT_ELIGIBLE"
  | "ASSIGNMENT_NOT_AUTHORIZED"
  | "USER_NOT_AUTHORIZED"
  | "BUSINESS_SCOPE_MISMATCH"
  | "RESOURCE_SCOPE_MISMATCH"
  | "APPROVAL_REQUIRED"
  | "APPROVAL_EXPIRED"
  | "APPROVAL_PAYLOAD_CHANGED"
  | "ACTION_DISABLED"
  | "RISK_LIMIT_EXCEEDED"
  | "STANDING_AUTHORITY_MISSING"
  | "PROVIDER_SCOPE_INSUFFICIENT"
  | "POLICY_PROHIBITED";
```

This gives the Chief of Staff meaningful recovery options.

---

# Policy Denial Is Not Failure

Example:

```text
email.send
→ approval required
```

This is not a system error.

It is a normal state transition.

Graph:

```text
Send Email
→ blocked_by_policy
→ Approval Node
→ Send Email resumes
```

Policy conditions should integrate naturally with the Work Graph.

---

# Authority and Work Graph

Policy should affect node readiness.

Example:

```text
Send Email Node
```

becomes ready only when:

```text
dependencies complete

approval satisfied

authority valid

preconditions satisfied
```

This avoids dispatching work that cannot legally execute.

---

# Approval Nodes May Be Inserted Automatically

The Chief of Staff may propose:

```text
Compose → Send
```

But policy may require:

```text
Compose → Approval → Send
```

The runtime should be able to insert the required approval node automatically.

Security should not depend on the Chief of Staff remembering to request approval.

---

# Policy Enforcement Locations

Policy should be evaluated at several boundaries.

## Assignment Creation

Can this agent receive this authority?

---

## Capability Request

Is this subject allowed to request this operation?

---

## Work Node Readiness

Are required approvals and authority present?

---

## Immediately Before Write

Are all current preconditions still valid?

---

## Completion

Does evidence prove the authorized action occurred?

Defense should be layered.

---

# Prompt Policy vs Runtime Policy

Prompts may include behavioral guidance:

> Never send an email without approval.

That is useful.

But real enforcement is:

```text
Agent has no direct provider access.

Capability Gateway requires valid authority.

Policy Engine requires approval.

Send operation refuses execution without approval.
```

The rule should remain true even if the model ignores the prompt.

---

# User Preferences Are Not Security Rules

Example memory:

```text
User usually likes me to send routine emails automatically.
```

That may inform policy configuration.

It should not itself grant authority.

Standing permission must exist as a durable policy grant.

Memory describes preference.

Policy grants authority.

---

# Audit Trail

Every authorization decision should record:

```text
who requested action

which role requested it

which Work Item

which assignment

which capability

which resource

which policy rules applied

whether approval was required

which approval authorized it

which constraints were applied

final policy decision

timestamp
```

This allows us to explain exactly why an operation occurred.

---

# Policy Decision Record

Conceptually:

```typescript
type PolicyDecisionRecord = {
  id: string;

  workItemId?: string;
  workNodeId?: string;
  assignmentId?: string;

  subject: AuthoritySubject;

  capability: string;
  operation: string;

  resourceScope?: ResourceScope;

  evaluatedRules: PolicyRuleReference[];

  decision:
    | "allowed"
    | "denied"
    | "approval_required"
    | "allowed_with_constraints";

  approvalId?: string;

  constraints: PolicyConstraint[];

  evaluatedAt: string;
};
```

---

# Dry-Run Support

Some capabilities should support preview or dry-run modes.

Examples:

```text
show email before sending

show calendar change before applying

show proposed refund

show customer update
```

This can improve trust and approval quality.

Dry-run is not a substitute for authorization.

It is an additional product mechanism.

---

# Reversibility

Policy should consider whether actions are reversible.

Example:

```text
create calendar event:
reversible

send email:
not meaningfully reversible

refund:
may be irreversible

delete record:
potentially destructive
```

Autonomy may reasonably be higher for reversible actions than irreversible ones.

---

# Reversal Is New Work

If an action has already executed, undoing it should be represented as another explicit operation.

Example:

```text
Calendar event created.
```

User:

```text
Undo that.
```

Runtime creates:

```text
calendar.delete_event
```

with its own:

```text
policy

authorization

evidence

Work Node
```

History remains intact.

---

# Financial Authority

Financial capabilities deserve a dedicated policy domain.

Initial recommendations:

```text
No financial write without explicit approval.

Every amount explicit.

Every currency explicit.

Every transaction verified.

No inferred recipient.

No amount greater than user's approved amount.

Every operation idempotent.

Every operation evidence-producing.

Standing financial authority disabled initially.
```

Financial autonomy can evolve only after strong production evidence.

---

# Communication Authority

Communication policy should account for:

```text
recipient familiarity

message sensitivity

message content

user's direct instruction

channel

recipient count

external vs internal

business impact
```

Examples:

```text
Draft:
autonomous.

Send to known cofounder after explicit user command:
may be allowed.

Send unsolicited message to 500 customers:
requires explicit approval.

Publish public statement:
requires explicit approval.
```

---

# Calendar Authority

Calendar policy may distinguish:

```text
read availability

create event

modify event

cancel event

invite external attendee
```

These have different consequences.

Example:

```text
Read calendar:
autonomous.

Create personal reminder:
autonomous.

Move client appointment:
approval or explicit instruction.

Cancel client appointment:
explicit approval.
```

---

# Policy Context

The Policy Engine should evaluate structured facts.

It should not require a language model for normal authorization.

Example:

```typescript
type PolicyContext = {
  subject: AuthoritySubject;

  userId: string;
  businessId: string;

  capability: string;
  operation: string;

  riskClass: ActionRiskClass;

  resourceScope?: ResourceScope;

  actionPayloadHash?: string;

  explicitUserInstruction?: UserInstructionReference;

  standingAuthority?: StandingAuthorityGrant[];

  approvals?: ApprovalReference[];

  providerScopes: string[];

  currentTime: string;
};
```

Models may assist with classifications such as:

```text
Is this communication sensitive?
```

But final authorization should remain deterministic.

---

# Model-Assisted Policy Classification

Some conditions require interpretation.

Example:

```text
Is this email routine or sensitive?
```

A bounded classifier may return:

```typescript
{
  classification: "sensitive",
  confidence: 0.91,
  reasons: [...]
}
```

Then deterministic policy uses that classification:

```text
IF sensitive
THEN approval required
```

The classifier does not authorize the send.

---

# Conservative Uncertainty

When policy classification is uncertain:

```text
uncertain risk
```

should generally move toward:

```text
more restrictive behavior
```

For example:

```text
confidence low
→ require approval
```

rather than:

```text
confidence low
→ assume safe
```

---

# Initial Policy Architecture

I would implement policies in code/configuration first.

Example structure:

```text
packages/
  policies/
    email/
    calendar/
    payments/
    customers/
    delegation/
    approvals/
    risk/
```

Policies should be:

```text
version-controlled

tested

reviewable

typed

deterministic
```

We should not begin with a generic user-editable policy language.

---

# Policy Versioning

Policy decisions should record policy version.

Example:

```text
email_send_policy v1.3
```

This allows us to answer:

> Why was this email allowed on August 5 but would be blocked today?

Policy evolution should remain auditable.

---

# Policy Testing

Every policy should have tests.

Examples:

```text
Known recipient + approved message → allow.

Unknown recipient → require resolution.

Recipient changed after approval → reject approval.

Body materially changed → approval invalid.

Expired approval → require approval.

Agent lacking eligibility → deny.

Assignment lacking grant → deny.

OAuth scope missing → deny.

Financial amount above limit → deny.

User revokes authority while waiting → deny.

Duplicate operation already succeeded → do not execute again.

Cancelled Work Item → deny write.

Disabled capability → deny.
```

---

# Security Tests

We should explicitly test attempts such as:

```text
Agent requests capability outside grant.

Agent fabricates approval ID.

Agent attempts to increase its own scope.

Retrieved email instructs agent to bypass approval.

Chief of Staff proposes disabled capability.

Old approval reused for modified payload.

Cross-business account accessed.

Cross-user resource accessed.

Standing authority used after expiration.

Cancelled work attempts delayed write.
```

The system should reject all of them without relying on model judgment.

---

# Example: Send Martin an Email

User:

> Send Martin the July report.

Potential flow:

```text
1. Chief of Staff forms objective.

2. Recipient Resolver verifies Martin.

3. Revenue Analyst produces report.

4. Email Composer produces draft.

5. Chief of Staff proposes email.send.

6. Policy Engine evaluates:
   - recipient verified
   - capability allowed
   - Work Item active
   - user authority
   - communication risk
   - approval policy

7. If approval required:
   Approval Node inserted.

8. User approves.

9. Approval binds exact recipient + content.

10. Send node becomes ready.

11. Immediately before execution:
    policy and payload revalidated.

12. Email capability sends.

13. Evidence persisted.

14. Work Item completion evaluated.
```

---

# Example: Recipient Changes After Approval

Approval:

```text
Send to:
martin@solos.com
```

User:

> Actually send it to his personal email.

The runtime detects:

```text
recipient changed
```

Therefore:

```text
old approval → stale
```

The send node cannot execute under the old approval.

A new approval may be required.

---

# Example: Specialist Attempts Write

Revenue Analyst attempts:

```text
email.send
```

Runtime checks:

```text
Agent Definition:
not eligible

Assignment:
not granted
```

Result:

```text
deny
```

No provider request occurs.

---

# Example: User Cancels While Waiting

Current state:

```text
Email Draft        completed
Approval           approved
Send Email         queued
```

User:

> Don't send it.

Runtime:

```text
Work Item cancellation requested

Send node → cancelled where possible

Authority reevaluated

Work Item no longer active
```

Even if the queue delivers the old send job later:

```text
precondition:
Work Item active?
→ false

execution denied
```

---

# Example: Standing Authority

User has configured:

```text
Automatically send routine appointment confirmations
to existing customers.
```

Proposed send:

```text
Existing customer

Appointment confirmation

No sensitive information

One recipient
```

Policy:

```text
standing authority matches
→ allow
```

Another message:

```text
Promotional blast to 300 customers
```

does not match standing authority.

```text
→ approval required
```

---

# Initial Authority Rules

For the first production version, I would establish:

```text
Specialists:
read-only by default

Chief of Staff:
may propose actions, not bypass policy

Writes:
performed only by Capability Gateway

External communication:
explicit policy evaluation required

Financial actions:
explicit approval required

Destructive actions:
explicit approval or disabled

Agent delegation:
registered agents only

Delegation depth:
1

Provider credentials:
never exposed to agents

Approvals:
bound to exact material payload

Expired/stale approvals:
invalid

Cancelled Work Items:
cannot execute new writes

Policy changes:
take effect before execution

Every policy decision:
persisted
```

---

# What We Are Not Building

We are not using prompts as our security model.

We are not allowing agents to decide their own permissions.

We are not assuming user intent gives unlimited authority.

We are not treating memory as authorization.

We are not giving every connected integration unrestricted access.

We are not allowing approvals to cover materially changed actions.

We are not allowing urgent requests to bypass policy.

We are not trusting queue delivery to prove an action is still authorized.

We are not making one global "autonomous mode" switch that ignores operation risk.

We are building explicit, contextual, enforceable authority.

---

# The Standard

Every consequential action should be able to answer:

```text
Who requested this?

What Work Item does it serve?

What role requested it?

What authority allowed it?

What resource may it affect?

What policy applied?

Was approval required?

What exactly was approved?

Is that approval still valid?

Are current preconditions still satisfied?

What evidence will prove the result?
```

If the runtime cannot answer those questions, the action should not execute.

---

# One Sentence

> **The Policies, Approvals, and Authority layer ensures that Solos can become increasingly autonomous without ever allowing model intent to become unchecked real-world authority.**
