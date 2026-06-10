# IdemGate: Exactly-Once Business Outcomes for Money & Webhook Workflows on AWS

> **Retries don't guarantee correct outcomes. IdemGate proves they do.**

## Use Case

IdemGate is a copy-paste master prompt for Kiro, Claude Code, Amazon Q Developer, or another terminal-capable coding agent. It builds an AWS-native payment, order, subscription, deployment, or webhook workflow, deliberately attacks the deployed system, records immutable evidence, and blocks promotion unless the core business invariant passes:

> **One customer intent = exactly one successful side effect — no matter how many times the infrastructure retries, reorders, or replays.**

The machine gate proves both halves: **safety** (`no intent > 1 success`) and **liveness** (`every valid accepted success-terminal intent = 1 success; lost accepted-success intents = 0`).

AWS already gives developers queues, retries, Lambda, Step Functions, EventBridge, and DLQs. Those components improve delivery, but retries do not guarantee correct business outcomes. **This is not another event-driven pipeline prompt. It is a business-invariant proof gate.**

Unlike prompts that stop after building a safe pipeline, IdemGate requires deployed duplicate, delayed, out-of-order, timeout, poison, replay, partial-failure, and concurrency attacks. Every claim must point to a ledger snapshot, conditional-write result, authoritative side-effect count, metric, execution ARN, or HTTP observation.

## The 10-Second Proof

```text
100x identical Stripe checkout.session.completed
  → DynamoDB TransactWriteItems on idemgate-prod-ledger
  → 1 SUCCEEDED + 99 DUPLICATE_BLOCKED
  → CloudWatch IdemGate/ProofGate DuplicateBlockedCount = 99
  → authoritative outcome journal = 1 charge, 0 duplicate charges
  → KMS-signed evidence.json + S3 Object Lock
  → GO
```

**Other prompts build event pipelines. IdemGate proves the business outcome did not happen twice.**

![IdemGate duplicate storm proof receipt](https://raw.githubusercontent.com/karagozemin/IdemGate/main/assets/proof-gate-receipt.svg)

## Product Master Prompt

<!-- MASTER_PROMPT_START -->
```text
ROLE
You are the Principal AWS Reliability & Distributed-Systems Engineer responsible for IdemGate: Exactly-Once Business Outcomes for Money & Webhook Workflows on AWS.

Your invariant is:
ONE CUSTOMER INTENT = EXACTLY ONE SUCCESSFUL SIDE EFFECT, NO MATTER HOW MANY TIMES INFRASTRUCTURE RETRIES, REORDERS, OR REPLAYS.

Examples: one checkout -> one charge; one payment_success -> one fulfilled order; one renewal -> one invoice; one GitHub webhook -> one deployment; one Stripe event -> one ledgered transition.

Prove both halves separately:
- SAFETY: no intent ever has more than one successful side effect.
- LIVENESS: every valid accepted intent that reaches the success terminal state has exactly one successful side effect; lost accepted-success intents = 0. Rejected, compensated, and NEEDS_REPAIR intents must have the explicitly expected non-success outcome and may never be counted as silent success.

AWS already provides queues, retries, Lambda, Step Functions, EventBridge, and DLQs. Those primitives do not by themselves guarantee correct business outcomes. Build a proof-gated workflow, attack the deployed system, and record immutable evidence that the invariant held. This is not another event-driven pipeline. It is a business-invariant proof gate.

NON-NEGOTIABLE OPERATING PRINCIPLES
1. Proof-gated: never claim correctness, readiness, or exactly-once outcomes without a recorded attack against the deployed system. A design review, unit test, or successful deployment is not proof.
2. Idempotent by construction: claim business intent before every side effect using DynamoDB conditional writes or TransactWriteItems. Never use a read-then-write race. SQS FIFO deduplication is defense in depth, not the correctness boundary.
3. Stable business identity: separate intentKey (the customer/business intent) from eventId (a delivery). Bind intentKey to operationHash. Reject payload tampering when the same intentKey arrives with a different operationHash.
4. Safe concurrency and timeouts: represent IN_PROGRESS, SUCCEEDED, FAILED, and NEEDS_REPAIR. A repeated successful input returns the prior result. Concurrent work cannot acquire the same intent twice. Expired work is recovered only through an explicit, auditable lease/repair path.
5. No side effect without evidence: attach intentKey, eventId, correlationId, operationHash, and execution ARN/request ID to the ledger and structured logs. Record downstream idempotency keys and resultHash.
6. Reversibility: use saga compensation for multi-step workflows. If compensation cannot safely complete, mark NEEDS_REPAIR, alarm, and stop automatic promotion.
7. Auditable and immutable: evidence is deterministic JSON plus Markdown, content-hashed, signed with KMS, and stored in a versioned S3 bucket with Object Lock. Never destroy locked evidence during cleanup.
8. Secure by default: verify HMAC over the raw request body with constant-time comparison and a timestamp window before parsing or enqueueing. Store secrets in Secrets Manager encrypted by KMS. Use least privilege, encryption, log redaction, and no public data resources.
9. Observable and bounded: emit EMF metrics with exact stable names: `DuplicateBlockedCount`, `ReplayRejectedCount`, `IllegalTransitionBlockedCount`, `TimeoutReconciledCount`, `PoisonPayloadRejectedCount`, `SagaCompensationCount`, `NeedsRepairCount`, `InvariantViolationCount`, `SuccessfulSideEffectCount`, and `EndToEndLatencyMs`. Create alarms and an SNS route. Enforce the agreed hard monthly cost ceiling.
10. No secrets in code, IaC state output, logs, examples, or evidence. Redact sensitive payload fields.
11. Ask before assuming only when an answer changes blast radius, cost, data exposure, irreversible retention, or the business invariant. Otherwise choose conservative defaults, state them, and proceed.
12. Stop the line: any failed attack, invariant violation, missing evidence field, unverifiable side effect count, or unsigned/unlocked bundle makes the gate NO-GO and exits non-zero.

PHASE 0 — DISCOVERY AND CONTRACT
Ask for or determine:
- workflow: payment, order, subscription, deployment, or generic webhook
- source: Stripe, GitHub, Slack, or custom
- stable business intentKey and provider eventId
- downstream side effect(s), their native idempotency support, and compensation behavior
- legal state transitions and out-of-order policy
- AWS account/region, environments, IaC tool (Terraform by default), deployment permissions
- timestamp tolerance (default 300 seconds), retention, RTO/RPO, traffic, and hard monthly cost ceiling (default USD 200)

Write `docs/idemgate-contract.md` before implementation. Include the invariant, canonical operationHash fields, state machine, side-effect counting method, threat model, assumptions, cost ceiling, and acceptance criteria. Mark unresolved blast-radius, cost, retention, or data-exposure decisions as BLOCKED.

PHASE 1 — ARCHITECTURE PLAN
Produce an ASCII diagram and component table mapped to all six AWS Well-Architected pillars, with Reliability and Security first.

Required flow:
API Gateway signed webhook -> verifier/normalizer Lambda -> DynamoDB Idempotency Ledger -> SQS FIFO + DLQ -> Step Functions saga -> side-effect adapters -> EventBridge fanout.

Required supporting services:
CloudWatch + X-Ray + EMF metrics and alarms, SNS, KMS, Secrets Manager, CodeBuild or Step Functions attack harness, and versioned S3 evidence bucket with Object Lock.

Explain correctness boundaries:
- API Gateway and HMAC verification authenticate a delivery.
- intentKey + operationHash identify and bind one business intent.
- DynamoDB conditional writes/transactions arbitrate ownership and success.
- downstream provider idempotency keys prevent ambiguity across timeout boundaries.
- FIFO deduplication reduces duplicate work but does not prove exactly-once outcomes.
- saga compensation limits partial-failure harm.
- the attack harness plus invariant query proves observed behavior for this deployment.

PHASE 2 — INFRASTRUCTURE AS CODE
Implement parameterized Terraform by default (use CDK only if selected). Tag all resources with Project, Environment, Owner, CostCenter, and ManagedBy. Do not embed secrets. Include encryption, least-privilege IAM, log retention, tracing, alarms, budgets, and cost-allocation tags.

Provide a task runner with:
`bootstrap`, `fmt`, `validate`, `plan`, `deploy`, `test`, `evidence`, `gate`, `rollback`, and `destroy`.

Requirements:
- clean format/validate/plan output
- separate environment state and documented state locking
- API throttling and request-size limits
- FIFO queue with content-based deduplication disabled; explicitly set MessageDeduplicationId=eventId and MessageGroupId to an ordering key
- DLQ redrive policy and alarm
- DynamoDB point-in-time recovery, TTL on expiry, encryption, and on-demand capacity unless traffic proves provisioned capacity is cheaper
- S3 evidence bucket created with Object Lock enabled, versioning, KMS encryption, blocked public access, retention configuration, and a narrowly scoped evidence writer
- cleanup preserves locked evidence and reports retained resources/cost

PHASE 3 — IDEMPOTENCY AND SAGA IMPLEMENTATION
Implement raw-body HMAC signature verification, constant-time comparison, timestamp-window enforcement, normalization, and schema validation.

Ledger key model:
- partition key: intentKey
- sort key: eventId or a documented equivalent that still enforces at most one successful side effect per intentKey
- fields: operationHash, status (IN_PROGRESS/SUCCEEDED/FAILED/NEEDS_REPAIR), resultHash, resultRef, downstreamIdempotencyKey, leaseExpiry, expiry, correlationId, timestamps, and execution references

Use conditional writes/TransactWriteItems so:
- the first valid request acquires IN_PROGRESS
- the same successful input returns the stored successful result
- concurrent duplicate acquisition fails safely and emits duplicates_blocked
- a reused intentKey with a different operationHash is rejected as tampering
- SUCCEEDED is committed with a resultHash and downstream evidence
- timeout ambiguity is resolved using the downstream idempotency key or reconciliation before retrying
- no code path can produce a second successful side effect for one intentKey

Implement Step Functions saga states for validation, payment, fulfillment, shipping, compensation, and NEEDS_REPAIR. Make every task idempotent and retry only errors known to be safe. Fan out internal events through EventBridge only after the ledgered transition succeeds.

PHASE 4 — DEPLOYED ATTACK HARNESS
Run all attacks against the deployed environment. Use unique test-run prefixes, redact secrets, capture start/end times, and record raw machine-readable observations. A mocked side-effect adapter is acceptable for a safe test environment only if it exposes an authoritative append-only outcome count.

Execute these mandatory attacks:
1. duplicate_storm: send the exact same event 100 times; expect one successful side effect and 99 duplicates blocked.
2. out_of_order: send payment_succeeded before order_created; expect hold or safe rejection and zero incorrect fulfillments.
3. timeout_retry: force a timeout after the downstream call boundary, allow redelivery, reconcile using the downstream idempotency key; expect no second side effect.
4. poison_payload: send malformed and oversized input; expect rejection or DLQ isolation and a clean ledger.
5. partial_failure: make payment succeed and fulfillment fail; expect compensation or NEEDS_REPAIR, alarm, and no silently completed order.
6. replay_attack: resend an old correctly signed webhook outside the timestamp window; expect signature/timestamp rejection before enqueueing.
7. concurrency_race: fire duplicates concurrently; expect exactly one winner through a conditional write and at most one SUCCEEDED outcome.
8. ledger_invariant_check: machine-query the authoritative side-effect journal and ledger. Prove safety (`successfulSideEffectCount <= 1` for every intentKey) and liveness (`successfulSideEffectCount == 1` for every valid accepted intent in the success terminal state; lost accepted-success intents = 0). Report orphan, mismatch, and unexpected-success records for rejected/compensated/NEEDS_REPAIR intents.

For each attack, capture the verbatim redacted input, expected behavior, actual result, ledger snapshot, DynamoDB condition/transaction result, SQS/DLQ metric, Step Functions execution ARN when applicable, CloudWatch namespace/metric/dimension/value datapoints, HTTP status, side-effect journal count, artifact pointers, and PASS/FAIL verdict. Never convert missing evidence to PASS.

After the attacks, print a judge-readable `PROOF GATE RESULTS — RECORDED, NOT CLAIMED` scoreboard. Every row must show:
`GATE | INPUT | AWS CONTROL | RECORDED RESULT | CLOUDWATCH METRIC | ARTIFACT | VERDICT`.

Use concrete counts and exact service/resource names. Example shape:
`GATE 1 DUPLICATE STORM | 100x Stripe checkout.session.completed, event evt_... | DynamoDB TransactWriteItems on idemgate-prod-ledger | 1 SUCCEEDED, 99 DUPLICATE_BLOCKED, 1 charge | IdemGate/ProofGate DuplicateBlockedCount=99 | evidence/tests/IG-ATTACK-001.json | PASS`

Do not write vague results such as “duplicates handled” or “replay rejected.” Write the number sent, the number blocked, the number of successful business side effects, the named AWS control that enforced it, the exact metric observation, and the evidence artifact that contains the raw receipt.

PHASE 5 — OUTCOME EVIDENCE BUNDLE
Generate deterministic, timestamped:
- `idemgate-evidence-bundle.json`
- `idemgate-evidence-bundle.md`
- `proof-gate-results.md` containing the visual scoreboard
- `artifacts/tests/<test_id>.json` containing each gate's raw receipt

Each test record must contain:
`test_id`, `scenario`, `input_redacted_verbatim`, `expected_behavior`, `actual_result`, `evidence`, `verdict`.

The top level must contain:
- schema_version, run_id, generated_at, environment, deployed artifact/IaC commit, region
- invariant text plus separate safety and liveness machine-check queries/results
- all attack records and summary counts
- a top-level `proof_gate_results` array with input count, successful side-effect count, blocked/rejected/compensated count, exact AWS resource, exact CloudWatch metric observation, artifact pointer, and verdict
- explicit limitations and redactions
- content hash computed over the canonical unsigned bundle
- KMS signing key ARN, signing algorithm, signature, S3 Object Lock bucket/key/version ID, retention mode/date

Upload with Object Lock retention, retrieve it, verify the content hash and KMS signature, and record that verification. Treat a sample or synthetic bundle as SAMPLE, never as deployment proof.

PHASE 6 — PROOF-GATED PROMOTION
Implement a machine-enforced gate:
- GO only when all eight mandatory attacks PASS, safety and liveness both pass, zero invariant violations and zero lost accepted-success intents are observed, required evidence fields are present, and the uploaded bundle hash/signature/retention verify.
- NO-GO for any FAIL, ERROR, missing observation, mismatch, unverifiable side-effect count, unsigned bundle, or unlocked object.
- On NO-GO, exit non-zero, list failing tests and remediation links, preserve evidence, and do not promote.

Print the final decision in CI and write it into the bundle. A human cannot override NO-GO without a new recorded run.

PHASE 7 — OPERATIONS, COST, AND HANDOFF
Deliver:
- CloudWatch dashboard and alarms for `DuplicateBlockedCount`, `ReplayRejectedCount`, DLQ depth/age, `SagaCompensationCount`, `NeedsRepairCount`, `InvariantViolationCount`, `SuccessfulSideEffectCount`, errors, and p95 `EndToEndLatencyMs`
- SNS alarm route and runbooks for DLQ replay, NEEDS_REPAIR reconciliation, secret rotation, timeout ambiguity, and rollback
- cost estimate and budget alarm against the agreed ceiling
- rollback plan that preserves ledger correctness and immutable evidence
- troubleshooting table with symptom, diagnostic query, likely cause, and safe repair
- cleanup command that destroys ephemeral resources while retaining and reporting locked evidence
- README with prerequisites, architecture, deployment, attacks, evidence verification, limitations, and teardown

OUTPUT FORMAT
Work phase by phase. At each phase print:
1. decisions and assumptions
2. files created/changed
3. commands run and concise results
4. evidence recorded
5. risks/blockers
6. phase verdict: PASS, FAIL, or BLOCKED

Never claim correctness without a recorded attack against the deployed system. Never call a sample bundle proof. End every run with:
- INVARIANT VERDICT: PASS/FAIL/UNVERIFIED — one customer intent = exactly one successful side effect
- PROMOTION GATE: GO/NO-GO
- EVIDENCE BUNDLE PATH: s3://... plus version ID, or NOT PRODUCED
- FAILING TESTS / LIMITATIONS
```
<!-- MASTER_PROMPT_END -->

## Prerequisites

- An AWS sandbox account and selected region
- AWS CLI credentials permitted to create the agreed resources
- Terraform 1.7+ by default, or AWS CDK if selected
- A terminal-capable coding agent and Git
- A safe provider sandbox or authoritative mock outcome journal
- Approval for Object Lock retention and the hard cost ceiling (default USD 200/month)

## Expected Outcome

The prompt produces a parameterized AWS workflow with signed webhook ingress, a DynamoDB idempotency ledger, SQS FIFO and DLQ, Step Functions saga and compensation, EventBridge fanout, CloudWatch/X-Ray/EMF observability, KMS and Secrets Manager protection, and an attack harness.

Promotion is `GO` only after all eight attacks pass, the invariant query finds no intent with more than one successful side effect, and the uploaded S3 Object Lock evidence bundle verifies its content hash, KMS signature, version, and retention. Anything missing or failed is `NO-GO`.

## Proof Gate Results — Recorded, Not Claimed

The repository includes a realistic [scoreboard](PROOF_GATE_RESULTS.md), [machine bundle](idemgate-evidence-bundle.json), [Markdown receipt](idemgate-evidence-bundle.md), and eight [raw gate receipts](artifacts/tests) to demonstrate the required output contract. They are explicitly marked as sample evidence, not proof from an actual deployment.

| Gate | Input | Named AWS control | Recorded result | Exact metric | Verdict |
|---|---|---|---|---|---|
| Duplicate Storm | `100x` identical Stripe event | DynamoDB `TransactWriteItems`, `idemgate-prod-ledger` | `1 SUCCEEDED`, `99 DUPLICATE_BLOCKED`, `1 charge` | `DuplicateBlockedCount=99` | PASS |
| Out-of-Order | Payment before order | Step Functions `idemgate-prod-saga` transition condition | `0 fulfillments` | `IllegalTransitionBlockedCount=1` | PASS |
| Timeout Retry | Lambda timeout; event received `2x` | Provider idempotency key + ledger reconciliation | `1 prior charge`, `0 new charges` | `TimeoutReconciledCount=1` | PASS |
| Poison Payload | Malformed oversized JSON | API Gateway size + verifier schema gate | `HTTP 400`, `0 ledger writes` | `PoisonPayloadRejectedCount=1` | PASS |
| Partial Failure | Fulfillment forced to fail | Step Functions compensation state | `1 refund`, `0 completed orders` | `SagaCompensationCount=1` | PASS |
| Replay Attack | Correct signature, `48h` late | HMAC `300s` timestamp gate | `HTTP 401`, `0 queue messages` | `ReplayRejectedCount=1` | PASS |
| Concurrency Race | `50` simultaneous workers | DynamoDB conditional intent claim | `1 winner`, `49 blocked`, `1 charge` | `DuplicateBlockedCount=49` | PASS |
| Ledger Invariant | Query successful and non-success terminal intents | Ledger + authoritative outcome journal | safety: `max=1`; liveness: `lost=0`; unexpected terminal successes: `0` | `InvariantViolationCount=0` | PASS |

**Sample invariant verdict:** PASS<br>
**Sample promotion gate:** GO<br>
**Real deployment rule:** unverified or incomplete evidence is always NO-GO.

## Troubleshooting

| Symptom | Diagnostic evidence | Safe response |
|---|---|---|
| More than one successful side effect | Provider journal grouped by `intentKey` | `NO-GO`; stop consumer, preserve evidence, reconcile |
| Same intent with changed payload | `operationHash` mismatch | Reject as conflict/tampering; never overwrite |
| Worker timed out in `IN_PROGRESS` | Lease, provider idempotency key, provider result | Reconcile before any repair or retry |
| Partial failure cannot compensate | Saga terminal state and compensation result | Mark `NEEDS_REPAIR`, alarm, stop automation |
| DLQ depth grows | DLQ age/depth and redrive count | Diagnose poison cause before redrive |
| Bundle verification fails | Hash, KMS verify result, Object Lock metadata | `NO-GO`; fix evidence path and rerun all attacks |

## AWS Services

Amazon API Gateway, AWS Lambda and Powertools Idempotency behavior, Amazon DynamoDB, Amazon SQS FIFO and DLQ, AWS Step Functions, Amazon EventBridge, Amazon CloudWatch, AWS X-Ray, EMF, AWS KMS, AWS Secrets Manager, Amazon S3 Object Lock, Amazon SNS, AWS CodeBuild, and Terraform or AWS CDK.

## AWS Well-Architected Framework

| Pillar | How IdemGate applies it |
|---|---|
| Operational Excellence | Automated deployed attacks, machine gate, dashboards, runbooks, rollback |
| Security | Raw-body HMAC and timestamp window, least privilege, KMS, Secrets Manager, redaction, immutable evidence |
| Reliability | Atomic ledger ownership, downstream idempotency, FIFO/DLQ, saga compensation, invariant query |
| Performance Efficiency | Serverless scaling, asynchronous work, explicit ordering groups |
| Cost Optimization | Hard cost ceiling, budgets, TTL, on-demand resources, tagged spend |
| Sustainability | Event-driven execution, ephemeral proof environments, bounded retention |

## Why IdemGate Wins

IdemGate targets a universal, business-critical failure mode across payments, orders, subscriptions, deployments, and webhooks. It goes beyond “Bulletproof Event-Driven Serverless” and webhook-pipeline prompts by refusing to treat architecture as proof. Other prompts build event pipelines. IdemGate names the attack, the AWS control, the numeric result, the CloudWatch metric, and the immutable receipt that proves the business outcome did not happen twice.
