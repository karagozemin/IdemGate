# DoraHacks Form Content

All text below is copy-paste-ready. Character-limited fields include verified counts.

## Profile Tab

**BUIDL name**

IdemGate

**Vision** (238/256 characters)

IdemGate turns retries into recorded proof: it attacks AWS money and webhook workflows with duplicates, races, replays, timeouts, poison, and partial failures, then blocks promotion unless one customer intent produced exactly one outcome.

**Category**

Other

**Is this an AI Agent?**

No

**Logo note**

Upload `assets/idemgate-logo.png` (480x480 PNG, under 2 MB).

## Team Information

Solo builder combining distributed-systems engineering, AWS architecture, and product storytelling. I built IdemGate to move reliability reviews from “the pipeline looks safe” to a machine-enforced, immutable record that the business invariant survived real attacks.

## Details Tab

Paste the complete contents of `SUBMISSION.md`. It includes the verbatim master prompt, prerequisites, use case, expected outcome, troubleshooting, evidence summary, AWS services, and all six Well-Architected pillars.

## Submission Tab

**Prompt Title**

IdemGate: Exactly-Once Business Outcomes for Money & Webhook Workflows on AWS

**Description & Use Case**

IdemGate is a proof-gated master prompt for building production AWS payment, order, subscription, deployment, and webhook workflows. It makes an AI coding agent implement idempotency by construction, attack the deployed workflow with eight failure scenarios, record immutable evidence, and block promotion unless one customer intent produced exactly one successful side effect. This is not another event-driven pipeline prompt. It is a business-invariant proof gate.

**Category dropdown**

Code Development

**AWS Services Used**

Amazon API Gateway; AWS Lambda and Powertools Idempotency behavior; Amazon DynamoDB; Amazon SQS FIFO and DLQ; AWS Step Functions; Amazon EventBridge; Amazon CloudWatch, X-Ray, and EMF; AWS KMS; AWS Secrets Manager; Amazon S3 Object Lock; Amazon SNS; AWS CodeBuild; Terraform or AWS CDK.

**Example Output** (866/960 characters)

INVARIANT VERDICT: PASS — one customer intent = exactly one successful side effect. PROMOTION GATE: GO. Deployed attack run: 8/8 PASS. Duplicate storm: 100 deliveries produced 1 charge; 99 conditional acquisitions blocked. Out-of-order payment: 0 incorrect fulfillments. Timeout retry: provider idempotency reconciliation found 1 charge; no second charge. Poison payload: rejected; ledger clean. Partial failure: refund compensation recorded. Old signed replay: rejected before enqueue. Concurrency race: 50 workers, 1 winner, 49 blocked. Ledger invariant query: 0 violating intent keys; maximum successful side effects per intent = 1. Evidence: canonical JSON + Markdown, SHA-256 content hash, KMS signature verified after upload, S3 Object Lock COMPLIANCE retention recorded. Path: s3://idemgate-evidence-ACCOUNT-us-east-1/runs/RUN_ID/idemgate-evidence-bundle.json

**Installation Steps**

1. Copy the fenced prompt from `MASTER_PROMPT.md` into Kiro, Claude Code, Amazon Q Developer, or another terminal-capable coding agent.
2. Provide an AWS sandbox account/region, workflow type, business intent key, downstream sandbox, and approved cost/retention limits.
3. Approve the Phase 0 contract, then let the agent run `bootstrap`, `plan`, `deploy`, `test`, `evidence`, and `gate`.
4. Treat only a verified deployed-run evidence bundle with an all-PASS `GO` decision as proof.

**Use Case Examples**

- One checkout creates exactly one charge during duplicate delivery and timeout retries.
- One payment-success webhook fulfills exactly one order despite replay or reordering.
- One subscription renewal creates exactly one invoice.
- One GitHub webhook creates exactly one deployment.
- One Stripe event creates exactly one ledgered state transition.

**Troubleshooting Tips**

- Any duplicate side effect, missing observation, failed signature, unlocked evidence object, or unverifiable count is `NO-GO`.
- Reject an `intentKey` reused with a different operation hash.
- Reconcile the downstream provider idempotency key before repairing an expired `IN_PROGRESS` lease.
- Mark uncompensated partial failure `NEEDS_REPAIR`; alarm and stop automation.
- Never redrive a DLQ until the poison cause and idempotency path are verified.
