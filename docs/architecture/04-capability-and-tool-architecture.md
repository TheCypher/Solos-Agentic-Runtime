# Capability and Tool Architecture

## Purpose

The Capability and Tool Architecture defines how Solos interacts with the outside world.

It governs how the Chief of Staff and specialist agents:

- discover what actions are available
- receive permission to use those actions
- request actions
- execute actions safely
- interact with provider APIs
- validate inputs and outputs
- handle retries
- enforce approvals
- prevent duplicate side effects
- persist evidence
- report verified results back to the runtime

The core principle is:

> **Agents reason in business capabilities. Provider adapters execute provider-specific tools.**

Agents should not reason directly in terms of Gmail endpoints, Square request objects, or provider SDK methods.

They should reason in terms of what Solos can do.

---

# Foundational Distinction

We need to separate four concepts:

```text
Capability
Operation
Provider Adapter
Provider Tool
```

## Capability

A business-level ability that Solos exposes internally.

Examples:

```text
Email
Calendar
Customers
Appointments
Revenue
Payments
Messaging
Documents
```

A capability describes **what Solos can do**.

---

## Operation

A specific action within a capability.

Examples:

```text
email.search
email.read
email.create_draft
email.send

calendar.list_events
calendar.find_availability
calendar.create_event
calendar.update_event

customers.find
customers.read
customers.create

revenue.get_summary
revenue.get_transactions
```

Operations are the callable contracts agents and workflows interact with.

---

## Provider Adapter

The component that translates a Solos operation into provider-specific behavior.

Example:

```text
Email Capability
      ↓
Gmail Adapter
      ↓
Gmail API
```

Future:

```text
Email Capability
      ↓
Microsoft Graph Adapter
      ↓
Outlook
```

The rest of the runtime does not need to know which provider was used.

---

## Provider Tool

The actual external API call, SDK operation, RPC, MCP method, or provider request.

Examples:

```text
gmail.users.messages.send

gmail.users.drafts.create

google.calendar.events.insert

square.customers.search

square.bookings.create
```

These are implementation details at the edge of the system.

---

# The Architecture

```text
Chief of Staff / Specialist / Workflow
                │
                ▼
        Capability Request
                │
                ▼
        Policy + Authorization
                │
                ▼
        Capability Executor
                │
                ▼
        Provider Adapter
                │
                ▼
        Provider API / Tool
                │
                ▼
        Normalized Result
                │
                ▼
      Evidence + Runtime State
```

The agent never directly owns the provider call.

The runtime controls the path.

---

# Why This Separation Matters

Without a capability layer, agents become coupled to providers.

For example:

```text
Agent knows Gmail
Agent knows Square
Agent knows Sendblue
Agent knows Google Calendar
```

Then provider-specific behavior leaks everywhere:

- prompts
- tool schemas
- retries
- error handling
- provider naming
- IDs
- authentication assumptions
- data formats

Instead:

```text
Agent knows:
"send an email"

Capability knows:
what sending an email means inside Solos

Adapter knows:
how Gmail performs that operation
```

This keeps the architecture stable as providers change.

---

# Capability Definition

Every capability should be registered with an explicit definition.

Conceptually:

```typescript
type CapabilityDefinition = {
  id: string;
  name: string;
  version: string;

  description: string;

  operations: CapabilityOperationDefinition[];

  riskClass: CapabilityRiskClass;

  supportedProviders: ProviderDefinition[];

  policy: CapabilityPolicy;

  status:
    | "draft"
    | "testing"
    | "active"
    | "deprecated"
    | "disabled";
};
```

Capabilities should be versioned just like agents.

---

# Operation Definition

Each operation must have a typed contract.

```typescript
type CapabilityOperationDefinition<
  TInput = unknown,
  TResult = unknown
> = {
  id: string;

  capability: string;
  operation: string;

  inputSchema: SchemaReference<TInput>;
  resultSchema: SchemaReference<TResult>;

  accessType: "read" | "write";

  sideEffectClass:
    | "none"
    | "reversible"
    | "consequential"
    | "financial"
    | "destructive";

  approvalPolicy: ApprovalPolicy;

  idempotencyPolicy: IdempotencyPolicy;

  retryPolicy: RetryPolicy;

  evidencePolicy: EvidencePolicy;

  timeoutPolicy: TimeoutPolicy;
};
```

The definition tells the runtime what the operation means and how it must be governed.

---

# Capability Registry

The Capability Registry is the authoritative catalog of actions available inside Solos.

It should answer:

```text
What capabilities exist?

What operations exist?

What inputs do they require?

Are they reads or writes?

What risk level do they have?

Which providers can fulfill them?

What approval policy applies?

What evidence proves success?

Can they safely be retried?

Which agents are eligible to use them?
```

A conceptual interface:

```typescript
interface CapabilityRegistry {
  getCapability(
    capability: string,
    version?: string
  ): Promise<CapabilityDefinition | null>;

  getOperation(
    capability: string,
    operation: string
  ): Promise<CapabilityOperationDefinition | null>;

  listAvailableOperations(
    context: CapabilityDiscoveryContext
  ): Promise<CapabilityCatalogEntry[]>;

  validateRequest(
    request: CapabilityRequest
  ): Promise<CapabilityValidationResult>;
}
```

---

# Capability Discovery

Agents should not receive every available capability on every invocation.

The runtime should expose only the capabilities relevant to the current assignment.

Example:

```text
Revenue Analyst receives:

revenue.get_summary
revenue.get_transactions
customers.read_aggregate
```

Not:

```text
email.send
calendar.create_event
payments.refund
messaging.send
customers.delete
```

The available tool surface should be deliberately narrow.

This improves:

- reliability
- model focus
- latency
- security
- tool-selection accuracy
- auditability

---

# Capability Catalog Entry

The Chief of Staff or specialist should receive a concise description.

Example:

```typescript
type CapabilityCatalogEntry = {
  capability: string;
  operation: string;

  description: string;

  access: "read" | "write";

  requiredInputs: string[];

  resultSummary: string;

  approvalRequired: boolean;
};
```

Example:

```text
email.create_draft

Creates an unsent email draft for a resolved recipient.

Requires:
recipient
subject
body

Access:
write

Approval:
not required to create draft
```

The model does not need Gmail's complete API schema.

---

# Capability Request

Agents do not call provider tools.

They submit a structured Capability Request.

```typescript
type CapabilityRequest<TInput = unknown> = {
  id: string;

  workItemId: string;
  workStepId?: string;

  assignmentId?: string;
  agentInstanceId?: string;

  capability: string;
  operation: string;

  input: TInput;

  requestedBy: CapabilityRequester;

  capabilityGrantId: string;

  idempotencyKey?: string;

  createdAt: string;
};
```

This request becomes part of the runtime history.

---

# The Execution Gateway

Every capability call passes through one centralized execution boundary.

Conceptually:

```text
Capability Request
      ↓
Capability Gateway
      ↓
Authorization
      ↓
Policy Check
      ↓
Approval Check
      ↓
Input Validation
      ↓
Idempotency Check
      ↓
Provider Resolution
      ↓
Adapter Execution
      ↓
Result Normalization
      ↓
Evidence Validation
      ↓
Persistence
```

No agent should bypass this path.

---

# Authorization

The Capability Gateway calculates effective authority.

```text
Agent eligibility
      ∩
Assignment grant
      ∩
User authorization
      ∩
Provider connection scope
      ∩
Business policy
      ∩
Runtime policy
      ∩
Approval state
      =
Allowed operation
```

A request must fail closed.

If authority cannot be proven, the action should not execute.

---

# Read vs Write

The runtime should treat reads and writes differently.

## Read operations

Examples:

```text
search emails
read calendar
retrieve revenue
find customer
list appointments
```

Reads are generally lower risk.

But they still require:

- authorization
- resource scoping
- privacy controls
- auditability

---

## Write operations

Examples:

```text
send email
create appointment
update customer
issue refund
delete event
send text
```

Writes need stronger controls.

Potential requirements:

```text
explicit permission

approval

idempotency

precondition checks

evidence

reconciliation

higher observability

stricter retries
```

---

# Risk Classification

Every operation should have a risk class.

```typescript
type CapabilityRiskClass =
  | "read_only"
  | "low_risk_write"
  | "reversible_write"
  | "external_communication"
  | "financial"
  | "destructive"
  | "high_consequence";
```

Example:

```text
calendar.list_events
→ read_only

email.create_draft
→ low_risk_write

calendar.create_event
→ reversible_write

email.send
→ external_communication

payments.refund
→ financial

customer.delete
→ destructive
```

Risk classification should drive policy automatically.

---

# Approval Policy

Approval should be attached to operations, not improvised in prompts.

```typescript
type ApprovalPolicy =
  | {
      mode: "none";
    }
  | {
      mode: "always";
    }
  | {
      mode: "conditional";
      policyId: string;
    };
```

Examples:

```text
email.read
Approval: none

email.create_draft
Approval: none

email.send
Approval: conditional or always

payments.refund
Approval: always initially

calendar.delete_event
Approval: conditional

messaging.send_external
Approval: conditional
```

The Policy Engine resolves conditional cases.

---

# Approval Context

Approval decisions may depend on:

```text
operation risk

user preferences

recipient

amount

business policy

whether the content was previously approved

whether the action is reversible

whether the user explicitly asked for the action

whether the target changed after approval
```

Approval should bind to the exact action payload.

For example:

```text
Approved:

Send email
To: Martin
Subject: July report
Body hash: abc123
```

If the recipient or body changes materially, the old approval should not automatically remain valid.

---

# Deterministic Writes

Consequential actions should generally be deterministic capability operations.

Example:

```text
Email Composer
→ produces draft

Chief of Staff
→ decides draft should be sent

Runtime
→ checks approval

Email Capability
→ executes send
```

We do not need a "Send Email Agent."

The reasoning has already happened.

Sending is execution.

---

# Provider Resolution

A capability may have multiple providers.

Example:

```text
email.send
```

Possible adapters:

```text
Gmail
Microsoft Graph
future provider
```

Provider resolution should be deterministic based on:

```text
user connection

account selection

resource ownership

provider capability support

business policy

availability

fallback policy
```

The model should not arbitrarily choose a provider.

---

# Provider Adapter Contract

A provider adapter translates Solos contracts into provider requests.

```typescript
interface EmailProviderAdapter {
  search(
    input: SearchEmailInput,
    context: ProviderExecutionContext
  ): Promise<SearchEmailResult>;

  createDraft(
    input: CreateDraftInput,
    context: ProviderExecutionContext
  ): Promise<CreateDraftResult>;

  send(
    input: SendEmailInput,
    context: ProviderExecutionContext
  ): Promise<SendEmailResult>;
}
```

The adapter owns:

- provider request formatting
- provider authentication
- provider SDK usage
- provider-specific pagination
- provider errors
- provider identifiers
- provider rate-limit handling
- provider response translation

It does not own business policy.

---

# Normalized Results

Provider results should be translated into Solos-owned result types.

Bad:

```typescript
Gmail.Users.Messages.SendResponse
```

Better:

```typescript
type SentEmailResult = {
  provider: "gmail";

  accountId: string;

  externalMessageId: string;
  externalThreadId?: string;

  recipients: string[];

  sentAt: string;

  evidence: EvidenceReference[];
};
```

The runtime should not depend on provider response structures.

---

# Evidence

Every meaningful write must produce evidence.

Evidence is what allows Solos to distinguish:

```text
requested
attempted
accepted
completed
verified
```

A generalized evidence record:

```typescript
type ExternalEvidence = {
  id: string;

  capability: string;
  operation: string;

  provider: string;

  evidenceType: string;

  externalResourceId?: string;
  externalOperationId?: string;

  observedAt: string;

  metadata: Record<string, unknown>;

  verificationStatus:
    | "provider_confirmed"
    | "reconciled"
    | "partially_verified"
    | "unverified";
};
```

Evidence should be persisted independently from agent output.

---

# Completion Proof

Each operation should define what proves success.

Example:

```text
email.create_draft
Success evidence:
provider draft ID
```

```text
email.send
Success evidence:
provider message ID + provider acceptance
```

```text
calendar.create_event
Success evidence:
provider event ID
```

```text
payments.refund
Success evidence:
refund transaction ID + provider status
```

```text
customer.find
Success evidence:
returned normalized record references
```

The capability definition declares this policy.

---

# Tool Result vs Objective Completion

The runtime must distinguish:

```text
Capability succeeded.
```

from:

```text
Work Item succeeded.
```

Example:

```text
email.create_draft succeeded
```

does not mean:

```text
Send Martin the email
```

is complete.

The draft is only evidence that one step succeeded.

Work Item completion is evaluated separately.

---

# Idempotency

Every consequential operation must have an idempotency strategy.

Conceptually:

```typescript
type IdempotencyPolicy =
  | {
      mode: "provider";
    }
  | {
      mode: "runtime_key";
    }
  | {
      mode: "reconcile_before_retry";
    }
  | {
      mode: "non_retryable";
    };
```

Example:

```text
Operation:
email.send

Idempotency key:
workItem + workStep + finalPayloadHash
```

The same logical send should not occur twice simply because a worker retried.

---

# Idempotency Records

The runtime should persist operation attempts.

```typescript
type CapabilityOperationRecord = {
  id: string;

  capabilityRequestId: string;

  idempotencyKey?: string;

  capability: string;
  operation: string;

  inputHash: string;

  state:
    | "prepared"
    | "executing"
    | "succeeded"
    | "failed"
    | "ambiguous"
    | "cancelled";

  provider?: string;

  externalOperationId?: string;
  externalResourceId?: string;

  createdAt: string;
  completedAt?: string;
};
```

Before executing a write, the runtime checks whether an equivalent operation already exists.

---

# Ambiguous Outcomes

One of the most important cases is:

```text
Runtime sends request.

Provider completes action.

Connection drops.

Runtime does not receive success response.
```

The runtime cannot safely assume:

```text
failed
```

and retry.

That can create duplicate real-world actions.

Instead:

```text
executing
→ ambiguous
→ reconcile
```

---

# Reconciliation

Capabilities must define reconciliation behavior where possible.

Example:

```text
email.send
→ search provider for known draft/message metadata
→ confirm whether message exists
→ recover provider ID
→ mark operation succeeded
```

Example:

```text
calendar.create_event
→ search for event by known external metadata
→ verify exact date/time/title
```

Example:

```text
payment refund
→ query refund state using provider request ID
```

Reconciliation should be provider-specific but governed by the capability layer.

---

# Retry Policy

Retries should be operation-aware.

Conceptually:

```typescript
type RetryPolicy = {
  maxAttempts: number;

  retryOn: CapabilityErrorClass[];

  backoff: BackoffPolicy;

  requireReconciliationBeforeRetry: boolean;
};
```

Examples:

```text
Network timeout on read
→ retry
```

```text
429 rate limit
→ retry according to provider policy
```

```text
invalid recipient
→ do not retry
```

```text
OAuth revoked
→ pause and request reconnection
```

```text
unknown result after write
→ reconcile before retry
```

---

# Capability Error Model

Provider-specific errors should normalize into runtime error classes.

```typescript
type CapabilityError =
  | "AUTHENTICATION_REQUIRED"
  | "AUTHORIZATION_DENIED"
  | "INVALID_INPUT"
  | "RESOURCE_NOT_FOUND"
  | "RATE_LIMITED"
  | "PROVIDER_UNAVAILABLE"
  | "TIMEOUT"
  | "AMBIGUOUS_RESULT"
  | "CONFLICT"
  | "POLICY_DENIED"
  | "APPROVAL_REQUIRED"
  | "UNSUPPORTED_OPERATION"
  | "INTERNAL_ERROR";
```

Agents should not reason over raw provider exceptions.

---

# Capability Result

Every execution returns a common envelope.

```typescript
type CapabilityResult<T = unknown> = {
  requestId: string;

  capability: string;
  operation: string;

  status:
    | "succeeded"
    | "failed"
    | "blocked"
    | "ambiguous";

  result?: T;

  evidence: EvidenceReference[];

  error?: CapabilityErrorDetail;

  providerMetadata?: ProviderMetadataReference;

  startedAt: string;
  completedAt: string;
};
```

This is what agents, workflows, and the runtime consume.

---

# Capability Calls Inside Agent Execution

When an agent requests a capability:

```text
Agent
→ Capability Request
```

the model invocation should pause while the runtime executes the request.

Conceptually:

```text
Agent reasoning
      ↓
tool request
      ↓
runtime authorization
      ↓
capability execution
      ↓
normalized result
      ↓
agent continues reasoning
```

The agent receives only the normalized result and evidence appropriate to its context.

---

# Capability Calls Must Be Persisted

Every call should record:

```text
who requested it

which Work Item it belonged to

which assignment it belonged to

which agent instance requested it

what operation was requested

what input was used

which authorization applied

which provider executed it

what happened

what evidence resulted

how long it took
```

No important tool call should exist only inside an LLM trace.

---

# Capability Grants

Capability access is issued through grants.

```typescript
type CapabilityGrant = {
  id: string;

  subjectType:
    | "chief_of_staff"
    | "agent_assignment"
    | "workflow"
    | "runtime";

  subjectId: string;

  capability: string;

  operations: string[];

  resourceScope?: ResourceScope;

  access: "read" | "write";

  approvalPolicy: ApprovalPolicy;

  expiresAt?: string;
};
```

Grants are contextual.

An Email Composer may be eligible to read relevant thread context for one assignment but have no access to unrelated messages.

---

# Resource Scoping

Permissions should be narrower than entire accounts whenever practical.

Examples:

```text
specific email thread

specific date range

specific customer

specific business

specific calendar

specific transaction set

specific draft

specific recipient
```

Example:

```typescript
{
  capability: "email",
  operations: ["read_thread"],
  resourceScope: {
    threadIds: ["thread_123"]
  }
}
```

The specialist should not receive inbox-wide access when one thread is sufficient.

---

# Provider Credentials

Agents must never receive provider credentials.

Credentials belong to the provider execution layer.

```text
Agent
→ capability
→ adapter
→ credential broker
→ provider
```

Credential handling should remain completely outside model context.

This includes:

- OAuth access tokens
- refresh tokens
- API keys
- provider secrets
- webhook secrets

---

# Secrets Boundary

No prompt or agent context should contain:

```text
API keys

OAuth tokens

database passwords

provider secrets

webhook signing keys
```

Provider authentication is infrastructure.

Not reasoning context.

---

# Capability Policies

A capability may contain business-level policies.

Example Email Capability:

```text
Do not send to unresolved recipient.

Do not send if approval binding is stale.

Do not send a superseded draft.

Do not treat provider acceptance as delivery confirmation
unless the provider supports that distinction.
```

Example Payment Capability:

```text
Require explicit amount.

Require verified transaction.

Require approval.

Reject refund amount above original transaction.

Require idempotency.
```

These rules should live in policy code, not solely in prompts.

---

# Preconditions

Writes should support explicit preconditions.

Example:

```typescript
type CapabilityPrecondition = {
  type: string;
  expected: unknown;
};
```

For an email send:

```text
recipient still equals approved recipient

draft hash still equals approved content

Work Item is not cancelled

approval still valid
```

For an appointment:

```text
time slot still available

customer identity still valid

calendar unchanged where relevant
```

The runtime checks preconditions immediately before action.

---

# Postconditions

Capabilities should also define expected postconditions.

Example:

```text
email.send

Postcondition:
provider message ID exists
```

```text
calendar.create_event

Postcondition:
created event can be retrieved
```

Where feasible, postconditions give stronger verification than a single API response.

---

# Read Caching

Some read capabilities may support caching.

Examples:

```text
business service catalog

user profile

provider metadata

stable contacts
```

But caching must never blur freshness requirements.

The capability definition should declare:

```text
cacheable

maximum age

freshness required for writes
```

Example:

```text
Revenue summary for analysis:
may allow short cache.

Appointment availability before booking:
must be fresh.
```

---

# Capability Composition

Capabilities should be composable without becoming workflows themselves.

Example:

```text
email.create_draft
```

may internally require:

```text
validate recipient
normalize content
select provider
create provider draft
persist evidence
```

Those are implementation steps.

The agent still sees one business operation.

A capability should expose a coherent atomic business action.

---

# Capabilities vs Workflows

This distinction is important.

## Capability

Represents an ability:

```text
send email
find customer
create appointment
```

## Workflow

Represents a procedure:

```text
prepare and send weekly report

follow up on overdue invoice

run morning business brief
```

A workflow may use multiple capabilities and specialist assignments.

Capabilities should not become hidden workflow engines.

---

# Capabilities vs Agents

Another critical distinction:

## Agent

Used when judgment is required.

```text
Draft sensitive customer response.
```

## Capability

Used when an operation is known.

```text
Create draft in Gmail.
```

An agent decides **what should be written**.

A capability determines **how the draft is created**.

---

# Capability Versioning

Capability definitions should be versioned.

Example:

```text
email.send v1
```

Later:

```text
email.send v2
```

may introduce:

- stronger evidence
- new provider support
- new approval policy
- different idempotency semantics

Running Work Items should bind to a compatible operation version where necessary.

---

# Provider Adapter Versioning

Provider adapters should also be versioned independently.

Example:

```text
gmail-adapter 2.3.1
```

This allows us to identify whether a regression originated from:

```text
agent logic

capability contract

provider adapter

provider API
```

---

# Observability

For every capability operation, we should know:

```text
What requested the action?

Why was it allowed?

What grant authorized it?

Was approval required?

Which approval covered it?

Which provider executed it?

How long did it take?

How many attempts occurred?

Was reconciliation required?

What external evidence exists?

What did the provider return?

Did the operation affect Work Item completion?
```

Useful identifiers:

```text
traceId
workItemId
workStepId
assignmentId
agentInstanceId
capabilityRequestId
capabilityOperationId
providerOperationId
evidenceId
approvalId
```

---

# Latency Measurement

Capability latency should be decomposed.

Example:

```text
authorization

approval resolution

queue delay

provider connection

provider execution

reconciliation

persistence
```

This helps avoid blaming model latency for provider problems.

---

# Capability Testing

Every operation should have tests for:

```text
valid input

invalid input

missing authorization

missing approval

expired approval

provider unavailable

rate limiting

timeout

duplicate execution

ambiguous write result

reconciliation success

reconciliation failure

provider malformed response

provider auth expiration

resource no longer exists

policy rejection

successful evidence creation
```

Write operations require especially strong failure-path testing.

---

# Fake Capability Layer

The runtime test suite should support fake capabilities.

Example:

```typescript
class FakeEmailCapability {
  drafts = [];
  sent = [];

  createDraft(...) { ... }

  send(...) { ... }
}
```

This lets us test:

```text
Chief of Staff
→ Assignment
→ Capability
→ Evidence
→ Completion
```

without contacting Gmail.

---

# Example: Read Email

User:

> What did Martin say in his last email?

Flow:

```text
1. Chief of Staff identifies simple retrieval.

2. Runtime resolves relevant email-read capability.

3. Capability grant permits read.

4. Email Capability selects connected provider.

5. Gmail Adapter searches relevant correspondence.

6. Provider results normalize into Solos email records.

7. Evidence references are persisted.

8. Chief of Staff summarizes relevant content.
```

No Email Agent is necessary unless interpretation becomes complex.

---

# Example: Draft Email

User:

> Write Martin an email with the July numbers.

Flow:

```text
1. Recipient Resolver identifies Martin.

2. Revenue Analyst prepares July analysis.

3. Email Composer writes the message.

4. Chief of Staff accepts the draft.

5. Runtime calls email.create_draft.

6. Gmail Adapter creates provider draft.

7. Provider draft ID becomes evidence.

8. Chief of Staff tells the user the draft is ready.
```

---

# Example: Send Email

User:

> Send it.

Flow:

```text
1. Runtime resolves the referenced draft.

2. Chief of Staff proposes send action.

3. Policy Engine checks:
   recipient
   content
   approval
   Work Item state
   authorization

4. Runtime calculates idempotency key.

5. Email Capability executes send.

6. Gmail Adapter sends the exact approved draft.

7. Provider returns message ID.

8. Evidence is persisted.

9. Work Step becomes complete.

10. Parent Work Item completion is evaluated.

11. Chief of Staff confirms verified success.
```

---

# Example: Timeout After Send

```text
1. Gmail receives send request.

2. Gmail sends message.

3. Runtime connection times out.

4. Capability Operation becomes:
   ambiguous

5. Runtime does not blindly retry.

6. Gmail Adapter performs reconciliation.

7. Message is found.

8. External message ID is recovered.

9. Operation becomes:
   succeeded

10. Evidence is persisted.

11. No duplicate email is sent.
```

This behavior must be architectural, not accidental.

---

# Example: Approval Becomes Invalid

User approves:

```text
Send July report to Martin A.
```

Then corrects:

```text
Actually use Martin's personal email.
```

The previous approval was bound to:

```text
Martin A work address
+
specific draft payload
```

Changing the recipient invalidates that approval.

The new send operation must receive a new valid approval if policy requires it.

---

# Initial Capability Set

For the first version, I would keep the capability surface deliberately small.

## Email

```text
email.search
email.read
email.create_draft
email.send
```

## Contacts

```text
contacts.search
contacts.read
```

## Calendar

```text
calendar.list_events
calendar.find_availability
calendar.create_event
calendar.update_event
```

## Revenue

```text
revenue.get_summary
revenue.get_transactions
```

## Customers

```text
customers.search
customers.read
```

## Work Management

Internal capabilities:

```text
work.read
work.request_approval
work.cancel
work.get_status
```

Start small.

Add operations only because a real product responsibility requires them.

---

# Initial Write Policy

For the first version, I would be conservative.

```text
Specialist agents:
Read-only by default.

Chief of Staff:
May propose writes.

Runtime:
Executes writes.

External communication:
Requires policy evaluation.

Financial operations:
Require explicit approval.

Destructive actions:
Disabled or require strong approval.

All consequential writes:
Require evidence and idempotency strategy.
```

This gives us a strong trust foundation.

---

# What We Are Not Building

We are not giving raw provider APIs directly to models.

We are not attaching every tool to every agent.

We are not allowing provider SDKs to define our business architecture.

We are not relying on prompts for authorization.

We are not blindly retrying external writes.

We are not treating a successful model tool call as evidence of real-world success.

We are not allowing provider credentials into model context.

We are not allowing agents to bypass the runtime execution gateway.

We are building a governed capability layer between intelligence and the real world.

---

# The Standard

Every Solos capability operation must be:

```text
Typed
Authorized
Scoped
Policy-controlled
Observable
Idempotent where required
Retry-aware
Provider-independent
Evidence-producing
Auditable
```

If an external action cannot satisfy those properties, it should not be exposed to agents.

---

# One Sentence

> **The Capability and Tool Architecture is the controlled execution boundary that converts agent intent into authorized, provider-independent, verifiable real-world action.**
