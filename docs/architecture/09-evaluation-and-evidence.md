# Evaluation and Evidence

## Purpose

The Evaluation and Evidence layer defines how Solos determines whether reasoning, actions, and outcomes are actually trustworthy.

It governs:

- how agent outputs are validated
- how factual claims are supported
- how deterministic checks are performed
- when model-based evaluation is justified
- how conflicting results are handled
- what evidence proves an external action occurred
- how evidence is attached to Work Items and Work Steps
- how completion claims are verified
- how unsupported or ambiguous results are rejected
- how Solos avoids false success confirmations

The core principle is:

> **No agent, capability, or model may prove its own success merely by claiming success.**

Completion must be supported by evidence and validated against explicit criteria.

---

# Foundational Distinction

The system must distinguish among:

```text
Output
Claim
Evidence
Evaluation
Acceptance
Completion
```

These are separate concepts.

---

# Output

An output is what an agent or capability returns.

Example:

```text
Revenue Analyst:
"July revenue increased 18%."
```

That is only an output.

It is not yet trusted.

---

# Claim

A claim is a factual assertion contained within an output.

Example:

```text
July revenue:
$12,450

Increase from June:
18%
```

Claims may require evidence.

---

# Evidence

Evidence is the source material or external proof supporting a claim or action.

Examples:

```text
normalized transaction records

provider message ID

calendar event ID

verified contact record

payment transaction ID

document passage

provider response
```

Evidence exists independently of model belief.

---

# Evaluation

Evaluation determines whether the output satisfies its contract.

It may check:

```text
schema validity

evidence support

calculation correctness

policy compliance

completion criteria

quality requirements

consistency
```

---

# Acceptance

A result becomes accepted only after required evaluation passes.

Accepted means:

> This result is trustworthy enough to be used by downstream work.

It does not necessarily mean the parent Work Item is complete.

---

# Completion

Completion means the user-level objective has satisfied its defined completion criteria.

Example:

```text
Agent drafted email:
accepted
```

does not mean:

```text
Email sent:
completed
```

---

# The Core Rule

> **A model's statement that something happened is never evidence that it happened.**

This applies to:

```text
email sent

appointment created

customer updated

payment refunded

message delivered

document saved

task completed
```

Only runtime and external evidence may establish those facts.

---

# Evaluation Pipeline

Conceptually:

```text
Result Submitted
      ↓
Schema Validation
      ↓
Policy Validation
      ↓
Evidence Validation
      ↓
Deterministic Validation
      ↓
Quality Evaluation if needed
      ↓
Completion-Criteria Check
      ↓
Accept / Revise / Reject / Escalate
```

Not every result requires every stage.

The evaluation policy defines which checks apply.

---

# Evaluation Policy

Every Agent Definition and Capability Operation should declare evaluation requirements.

```typescript
type EvaluationPolicy = {
  schemaValidation:
    | "required"
    | "optional";

  evidenceValidation:
    | "required"
    | "optional"
    | "none";

  deterministicChecks: EvaluationCheckDefinition[];

  modelEvaluation?: ModelEvaluationPolicy;

  chiefOfStaffReview:
    | "required"
    | "optional"
    | "none";

  humanReview:
    | "required"
    | "conditional"
    | "none";

  acceptancePolicy: AcceptancePolicy;
};
```

---

# Evaluation Result

All evaluation should produce a structured result.

```typescript
type EvaluationResult = {
  id: string;

  subjectType:
    | "agent_result"
    | "capability_result"
    | "work_node"
    | "work_item";

  subjectId: string;

  decision:
    | "accept"
    | "revise"
    | "reject"
    | "escalate";

  checks: EvaluationCheckResult[];

  evidenceValidated: EvidenceReference[];

  issues: EvaluationIssue[];

  evaluatedAt: string;
};
```

---

# Evaluation Order

Evaluation should prefer the least expensive reliable method.

Recommended order:

```text
1. Schema validation

2. Deterministic validation

3. Evidence validation

4. Policy validation

5. Chief of Staff judgment

6. Specialist/model evaluator

7. Human review
```

Do not invoke another model if deterministic checks can answer the question.

---

# Schema Validation

Structured agent results must first satisfy their declared schema.

Example:

```text
Revenue Analyst output requires:

summary
metrics
findings
evidence
limitations
```

If the output is malformed:

```text
do not pass it downstream
```

Possible recovery:

```text
deterministic repair where safe

targeted structured-output retry

model fallback

assignment failure
```

---

# Deterministic Evaluation

Deterministic checks should handle objective conditions.

Examples:

```text
Does the percentage recompute?

Does the email address match the verified contact?

Does every required field exist?

Does the date range match the assignment?

Does the draft recipient equal the resolved recipient?

Does the approval payload hash match?

Does the message ID exist?

Does the output exceed allowed limits?
```

These checks should not use an LLM.

---

# Calculation Validation

For numerical analysis, the runtime should recompute important derived numbers.

Example:

Agent claims:

```text
June:
$10,000

July:
$11,800

Increase:
28%
```

Runtime calculates:

```text
18%
```

Evaluation result:

```text
reject or revise
```

The model's prose quality cannot override incorrect arithmetic.

---

# Evidence Validation

Evidence validation determines whether claims are actually supported.

A factual finding may require:

```typescript
type EvidenceRequirement = {
  claimCategory: string;

  requiredEvidenceTypes: string[];

  minimumCount?: number;

  freshnessRequirement?: FreshnessRequirement;

  authorityRequirement?: EvidenceAuthority;
};
```

---

# Evidence Types

Evidence may include:

```text
User Evidence

Runtime Evidence

Provider Evidence

Database Evidence

Document Evidence

Accepted Agent Evidence

Derived Evidence
```

These should not have equal authority.

---

# User Evidence

A current user statement may be evidence of the user's intention, preference, or asserted fact.

Example:

```text
"Martin is my cofounder."
```

This is strong evidence of relationship context.

It is not necessarily external proof of Martin's current email address.

---

# Runtime Evidence

Runtime evidence proves internal state.

Examples:

```text
approval ID

Work Item state

assignment result

policy decision

operation record

cancellation event
```

This is authoritative for Solos runtime facts.

---

# Provider Evidence

Provider evidence proves external operations or records.

Examples:

```text
Gmail message ID

Google Calendar event ID

Square payment ID

provider delivery receipt
```

This is the primary evidence for external side effects.

---

# Document Evidence

Document evidence references source material.

Example:

```text
contract section
invoice line
email passage
report table
```

Claims should reference exact source sections where practical.

---

# Accepted Agent Evidence

An accepted agent result may be evidence for downstream reasoning.

Example:

```text
Revenue Analyst result:
accepted
```

The Email Composer may use its findings.

But the upstream result should preserve its underlying evidence references.

---

# Derived Evidence

Derived evidence represents deterministic transformation of source evidence.

Example:

```text
Revenue Change:
18%

Derived from:
June revenue evidence
July revenue evidence
calculation function v1
```

This makes analytical conclusions traceable.

---

# Evidence Record

Conceptually:

```typescript
type EvidenceRecord = {
  id: string;

  evidenceType: string;

  sourceType:
    | "user"
    | "runtime"
    | "provider"
    | "database"
    | "document"
    | "agent_result"
    | "derived";

  sourceId: string;

  provider?: string;

  externalResourceId?: string;

  observedAt: string;

  contentReference?: string;

  verificationStatus:
    | "verified"
    | "reconciled"
    | "user_asserted"
    | "derived"
    | "unverified";

  supersededBy?: string;
};
```

---

# Evidence Is Immutable

Once created, an evidence record should generally not be mutated.

If new information changes understanding:

```text
new evidence created
old evidence superseded
```

This preserves auditability.

---

# Evidence Freshness

Some evidence expires quickly.

Examples:

```text
calendar availability

inventory

account balance

current provider status
```

Others are stable:

```text
sent message ID

completed transaction ID

historical invoice

signed document
```

Evaluation policies should account for freshness.

---

# Evidence Authority

When evidence conflicts, source authority matters.

Example:

```text
Memory:
Martin's email = martin@old.com

Verified Gmail correspondence:
Martin's email = martin@new.com
```

The newer verified external evidence should dominate.

Evaluation should preserve both the conflict and resolution.

---

# Claim-to-Evidence Mapping

Important claims should explicitly map to evidence.

```typescript
type SupportedClaim = {
  claimId: string;

  statement: string;

  category: string;

  evidence: EvidenceReference[];

  confidence?: number;
};
```

This gives downstream agents traceable grounding.

---

# Unsupported Claims

If a specialist returns:

```text
"Revenue declined because customers disliked the new pricing."
```

but only revenue transactions exist, this may be:

```text
unsupported causal claim
```

The evaluator should either:

```text
reject

mark as hypothesis

require supporting evidence

request revision
```

It should not silently become fact.

---

# Facts vs Inferences

Results should distinguish:

```text
Observed Fact

Derived Fact

Inference

Hypothesis

Recommendation
```

Example:

```text
Observed:
Bookings fell 22%.

Derived:
Lost booking revenue was approximately $1,400.

Inference:
The booking decline likely explains most of the revenue decline.

Hypothesis:
The decline may relate to the price increase.
```

This distinction should survive downstream synthesis.

---

# Evaluation Issue Types

Conceptually:

```typescript
type EvaluationIssueCode =
  | "SCHEMA_INVALID"
  | "MISSING_EVIDENCE"
  | "UNSUPPORTED_CLAIM"
  | "CALCULATION_ERROR"
  | "STALE_EVIDENCE"
  | "CONFLICTING_EVIDENCE"
  | "POLICY_VIOLATION"
  | "ASSIGNMENT_INCOMPLETE"
  | "OUTPUT_OUT_OF_SCOPE"
  | "LOW_CONFIDENCE"
  | "QUALITY_INSUFFICIENT"
  | "COMPLETION_NOT_PROVEN";
```

---

# Acceptance Outcomes

Evaluation may produce four primary outcomes.

## Accept

The result satisfies requirements.

```text
result → accepted
```

Downstream work may consume it.

---

## Revise

The result is close but correctable.

Example:

```text
Draft is good but contains one unsupported revenue number.
```

Return targeted feedback.

Do not restart all upstream work.

---

## Reject

The result cannot be trusted.

Examples:

```text
wrong recipient

fabricated evidence

output outside assignment scope

severe calculation error
```

The result should not become eligible context.

---

## Escalate

Evaluation identifies a problem requiring stronger judgment.

Examples:

```text
specialist disagreement

ambiguous evidence

sensitive communication

high-risk decision
```

Escalation may invoke:

```text
Chief of Staff

stronger model

specialist evaluator

human
```

---

# Revision Feedback

Revision should be targeted.

```typescript
type EvaluationRevisionRequest = {
  assignmentId: string;

  issues: Array<{
    code: EvaluationIssueCode;
    field?: string;
    explanation: string;
  }>;

  retainedValidOutputs: string[];

  revisionBudget: ExecutionBudget;
};
```

Example:

```text
Do not redo revenue retrieval.

Correct the percentage calculation and rewrite the affected paragraph.
```

---

# Avoid Evaluation Loops

The runtime must prevent:

```text
agent
→ evaluator
→ revision
→ evaluator
→ revision
→ ...
```

Initial policy:

```text
maximum revision attempts:
1 or 2 depending assignment class
```

After that:

```text
escalate or fail
```

---

# Model Evaluators

A model evaluator is appropriate for subjective quality.

Examples:

```text
Does the email sound appropriately professional?

Does the summary preserve all material conclusions?

Is the recommendation internally coherent?

Is the tone too aggressive?
```

A model evaluator should receive:

```text
assignment

result

relevant evidence

evaluation rubric
```

Not unrelated context.

---

# Evaluator Independence

Where model evaluation is important, the evaluator should not simply receive:

```text
"the previous model said this is correct"
```

It should receive enough source information to judge independently.

But this should not turn into redundant full re-execution.

---

# Chief of Staff Review

The Chief of Staff should review outputs when the question is:

```text
Does this result satisfy the larger user objective?

Do several accepted results fit together?

Should we act on this information?

What remains?

How should this be communicated?
```

The Chief of Staff should not waste reasoning on checks already handled deterministically.

---

# Human Review

Human review should be reserved for cases requiring true human authority or judgment.

Examples:

```text
explicit user approval

high-consequence business commitments

financial authorization

sensitive external communication under policy
```

Human review is not a generic quality-control step.

---

# Capability Evaluation

Capabilities also require evaluation.

Example:

```text
email.send
```

Evaluation should check:

```text
provider returned valid result

external message ID exists

payload matches authorized payload

operation record persisted

idempotency state valid
```

The model is not involved.

---

# External Action States

External writes should distinguish:

```text
Requested

Authorized

Attempted

Provider Accepted

Verified

Completed
```

These states should never collapse into one generic:

```text
success
```

---

# Provider Acceptance vs Verification

Provider acceptance may be sufficient evidence for some operations.

Example:

```text
Gmail returns message ID
```

For others, stronger confirmation may be needed.

Example:

```text
payment status

scheduled appointment

delivery status
```

Each capability defines its proof threshold.

---

# Ambiguous Evidence

Example:

```text
send request timed out

no provider response
```

The evidence state is:

```text
ambiguous
```

Not:

```text
failed
```

and not:

```text
succeeded
```

The runtime must reconcile.

---

# Reconciliation Evidence

If reconciliation discovers the external action occurred:

```text
provider query finds message
```

create evidence:

```text
verificationStatus:
reconciled
```

This is still valid evidence.

The runtime should record how success was recovered.

---

# Completion Evidence

Work Item completion should reference objective-level evidence.

Example:

```text
Objective:
Send Martin July report.
```

Completion evidence:

```text
verified Martin identity

accepted July report

approved final draft

external email message ID
```

A Work Item should be able to explain why the runtime marked it complete.

---

# Completion Proof Object

Conceptually:

```typescript
type WorkCompletionProof = {
  workItemId: string;

  criteria: Array<{
    criterionId: string;
    satisfied: boolean;
    evidence: EvidenceReference[];
  }>;

  verifiedAt: string;

  status:
    | "complete"
    | "incomplete";
};
```

Completion state derives from this proof.

---

# Completion Must Fail Closed

If required evidence is missing:

```text
do not claim completion
```

Example:

```text
Agent:
"I sent the email."
```

But there is no provider evidence.

Result:

```text
Work Item remains incomplete.
```

The Chief of Staff may say:

> I don't have confirmation that the email was sent.

It must not fabricate certainty.

---

# False Success Prevention

The runtime should specifically prohibit treating these as completion evidence:

```text
model-generated success statement

planner outcome

tool request creation

queue delivery

internal bookkeeping success

agent result status alone

work-step intent

"ok: true" from an unrelated internal function
```

External actions require external or capability-specific evidence.

---

# Evidence Lineage

Evidence should preserve causal lineage.

Example:

```text
Square transactions
        ↓
Normalized revenue dataset
        ↓
Revenue analysis
        ↓
Email draft
        ↓
User approval
        ↓
Gmail send
```

This lets us trace the user's final outcome back to source data.

---

# Derived Artifact Lineage

A derived artifact should record dependencies.

```typescript
type ArtifactLineage = {
  artifactId: string;

  derivedFrom: EvidenceReference[];

  producedBy:
    | AgentExecutionReference
    | CapabilityOperationReference
    | DeterministicFunctionReference;

  createdAt: string;
};
```

This helps corrections propagate properly.

---

# Evidence Supersession

Example:

```text
Recipient evidence:
martin@work.com
```

User later confirms:

```text
Use martin@personal.com instead.
```

The old evidence does not disappear.

But the current Work Item should reference the new authoritative recipient.

Downstream evaluation must respect supersession.

---

# Evaluation and Context

Only accepted results should become normal downstream context.

```text
submitted
→ evaluated
→ accepted
→ context eligible
```

Rejected results should be excluded from normal reasoning unless explicitly supplied for diagnostic purposes.

---

# Evaluation and Work Graph

Work Nodes should complete only after required evaluation passes.

Example:

```text
Revenue Analyst execution finished
```

Node state:

```text
evaluating
```

Then:

```text
accepted
→ node completed
```

If revision required:

```text
node remains unresolved
```

---

# Evaluation and Model Budget

Evaluation must respect Model Routing rules.

Do not automatically double the number of model calls.

A good hierarchy is:

```text
schema

deterministic

evidence

policy

model only if necessary
```

This keeps reliability high without unnecessary latency.

---

# Evaluation Profiles

Different agent classes should have different profiles.

## Recipient Resolver

```text
Schema:
required

Evidence:
required

Deterministic:
validate email/contact record

Model evaluator:
not normally needed
```

---

## Revenue Analyst

```text
Schema:
required

Evidence:
required

Calculations:
deterministically recomputed

Causal claims:
must be supported or labeled inference

Chief of Staff:
reviews business significance
```

---

## Email Composer

```text
Schema:
required

Evidence:
facts must trace to accepted sources

Recipient:
must match resolved recipient

Tone:
may use model evaluation for high-risk content

Sending:
outside agent responsibility
```

---

## Schedule Analyst

```text
Schema:
required

Calendar evidence:
required

Current availability:
freshness checked

Writes:
none
```

---

# Quality Rubrics

For subjective evaluation, use explicit rubrics.

Example Email Quality Rubric:

```text
Accurate

Complete

Appropriate tone

No unsupported claims

No unnecessary disclosure

Aligned with objective

Clear next action
```

Rubrics should be versioned.

---

# Evaluation Versioning

Every evaluation should record:

```text
evaluation policy version

rubric version

deterministic validator version

model evaluator version when used
```

This supports regression analysis.

---

# Evidence Storage

Evidence should live in PostgreSQL as metadata and references.

Large source content may live elsewhere with stable references.

The database should store:

```text
evidence identity

source

provider

external IDs

verification state

timestamps

lineage

supersession

related Work Items
```

---

# Sensitive Evidence

Evidence may contain sensitive information.

The runtime should separate:

```text
evidence metadata
```

from:

```text
sensitive content
```

Agents receive only what their context policy permits.

---

# Evidence Retention

Retention policy should depend on:

```text
business requirements

provider terms

privacy requirements

audit value

user controls
```

Do not retain raw external content forever simply because it may be useful.

---

# Evaluation Observability

For every result, we should answer:

```text
What was evaluated?

Which checks ran?

Which checks failed?

What evidence was required?

What evidence was found?

What was unsupported?

Was revision requested?

Was escalation used?

Which evaluator was involved?

Why was the result accepted?
```

---

# Trust Metrics

Useful metrics include:

```text
agent-result acceptance rate

revision rate

rejection rate

unsupported-claim rate

calculation-error rate

missing-evidence rate

false-completion rate

reconciliation rate

ambiguous-operation rate

model-evaluator usage rate

human-approval rate

completion-evidence coverage
```

The most important reliability metric may be:

> **Percentage of claimed completed external actions with valid external evidence.**

That should approach 100%.

---

# Evaluation Tests

Tests should include:

```text
Agent claims success without evidence.

Wrong arithmetic with valid-looking prose.

Recipient differs from verified contact.

Email draft contains unsupported metric.

Stale calendar availability.

Provider operation ambiguous.

Reconciliation succeeds.

Reconciliation fails.

Rejected agent result attempts to enter downstream context.

Superseded evidence used by later assignment.

Approval payload differs from send payload.

Capability succeeds but Work Item criteria remain incomplete.

Model evaluator unavailable.

Deterministic evaluation sufficient without model.

Conflicting evidence sources.

Agent labels hypothesis as fact.

Work Item completion attempted with missing criterion.
```

---

# Example: Revenue Analysis

Revenue Analyst returns:

```text
July revenue:
$12,000

June revenue:
$10,000

Increase:
25%
```

Evidence supports the raw values.

Deterministic evaluator calculates:

```text
20%
```

Evaluation:

```text
decision:
revise

issue:
CALCULATION_ERROR
```

The result does not become downstream context.

---

# Example: Unsupported Explanation

Agent returns:

```text
Revenue dropped because customers disliked the new prices.
```

Evidence:

```text
transactions only
```

Evaluation:

```text
Observed revenue decline:
supported

Customer sentiment explanation:
unsupported
```

Revision may require:

```text
remove claim
```

or:

```text
label it as hypothesis
```

---

# Example: Email Send

Capability executes:

```text
email.send
```

Provider returns:

```text
messageId:
msg_123
```

Runtime stores:

```text
ExternalEvidence:
provider = Gmail
externalMessageId = msg_123
verification = provider_confirmed
```

Send Work Node completes.

The Chief of Staff may now truthfully tell the user:

> Sent.

---

# Example: Fake Success

Agent says:

> The email has been sent.

But runtime shows:

```text
no email.send operation

no provider message ID

no evidence
```

Evaluation:

```text
COMPLETION_NOT_PROVEN
```

The claim is rejected.

The Work Item remains active or failed depending on execution state.

---

# Example: Ambiguous Send

Provider connection times out.

Runtime:

```text
email.send:
ambiguous
```

No completion claim is allowed.

Reconciliation finds:

```text
msg_123
```

Evidence becomes:

```text
reconciled
```

Then the Work Node may complete.

---

# Example: Completion of Entire Work Item

Objective:

```text
Send Martin the July revenue report.
```

Completion criteria:

```text
Recipient resolved.

July revenue analysis accepted.

Final draft accepted.

Required approval satisfied.

Email send verified.
```

Runtime evaluates each.

Only when all are true:

```text
Work Item → completed
```

The Chief of Staff then communicates completion.

---

# Initial Evaluation Rules

For the first production implementation:

```text
All structured outputs:
schema validation required

All material factual claims:
source evidence where applicable

All calculations:
deterministic validation where practical

All external writes:
provider evidence required

All ambiguous writes:
reconciliation before retry

All accepted agent results:
evaluation record required

Rejected results:
excluded from downstream context

Specialist self-declared completion:
never sufficient

Work Item completion:
runtime-owned

Model evaluator:
only when deterministic evaluation is insufficient

Maximum evaluation revision loops:
bounded

Every completion:
must have explicit proof
```

---

# What We Are Not Building

We are not asking one model whether another model "looks right" for every task.

We are not treating confidence scores as proof.

We are not accepting unsupported agent claims.

We are not allowing specialist output to become fact automatically.

We are not treating tool invocation as evidence of external completion.

We are not treating provider timeouts as definite failures.

We are not treating queue success as action success.

We are not allowing completion without objective criteria.

We are building a system where trust comes from evidence and validation.

---

# The Standard

Every accepted result should answer:

```text
What does this claim?

What evidence supports it?

Is that evidence authoritative?

Is it fresh enough?

Can important parts be checked deterministically?

Does the result satisfy its contract?

What uncertainty remains?
```

Every completed action should answer:

```text
What action occurred?

Who authorized it?

Which provider performed it?

What external evidence proves it?

Was the exact approved payload used?
```

And every completed Work Item should answer:

```text
Which objective criteria were satisfied?

What evidence proves each one?

What limitations remain?
```

---

# One Sentence

> **The Evaluation and Evidence layer ensures that Solos never confuses model confidence, internal progress, or attempted execution with verified truth and completed work.**
