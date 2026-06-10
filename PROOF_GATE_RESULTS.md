# Proof Gate Results — Recorded, Not Claimed

> **Evidence class: SAMPLE RECORDED RUN — demonstrates the output contract; not proof from a live AWS deployment.**

Run: `idemgate-prod-us-east-1-20260610T084215Z-a91c7e2`  
Workflow: Stripe checkout → payment → fulfillment  
Invariant: **one customer intent = exactly one successful side effect**

## The 10-Second Proof

```text
INPUT                 AWS CONTROL                         RECORDED RESULT                     METRIC / ARTIFACT
100x same Stripe  ──▶ DynamoDB TransactWriteItems   ──▶ 1 SUCCEEDED + 99 BLOCKED       ──▶ DuplicateBlockedCount=99
event evt_01JY4M3G    table idemgate-prod-ledger         1 charge, 0 duplicate charges       tests/IG-ATTACK-001.json

50 concurrent     ──▶ DynamoDB conditional claim    ──▶ 1 winner + 49 blocked          ──▶ DuplicateBlockedCount=49
workers               intentKey checkout_ig_007          1 charge                           tests/IG-ATTACK-007.json

ALL INTENTS        ──▶ ledger + outcome-journal query ─▶ max successes = 1, lost = 0    ──▶ InvariantViolationCount=0
```

## Recorded Gate Scoreboard

| Gate | Input | AWS control | Recorded result | CloudWatch metric | Raw receipt | Verdict |
|---|---|---|---|---|---|---|
| 1. Duplicate Storm | `100x` identical Stripe `checkout.session.completed`, `evt_01JY4M3G` | DynamoDB `TransactWriteItems`, table `idemgate-prod-ledger` | `1 SUCCEEDED`, `99 DUPLICATE_BLOCKED`, `1 charge`, `0 duplicate charges` | `IdemGate/ProofGate · DuplicateBlockedCount · 99` | `artifacts/tests/IG-ATTACK-001.json` | **PASS** |
| 2. Out-of-Order | `payment_succeeded` before `order_created` | Step Functions state machine `idemgate-prod-saga`, legal-transition condition | `1 illegal transition blocked`, `0 fulfillments` | `IdemGate/ProofGate · IllegalTransitionBlockedCount · 1` | `artifacts/tests/IG-ATTACK-002.json` | **PASS** |
| 3. Timeout Retry | Lambda timeout after provider boundary, event received `2x` | Provider idempotency key `idemgate/checkout_ig_003` + ledger reconciliation | `1 prior charge returned`, `0 new charges` | `IdemGate/ProofGate · TimeoutReconciledCount · 1` | `artifacts/tests/IG-ATTACK-003.json` | **PASS** |
| 4. Poison Payload | `1x` malformed oversized JSON webhook | API Gateway size limit + verifier schema gate | `HTTP 400`, `0 ledger writes`, `0 side effects` | `IdemGate/ProofGate · PoisonPayloadRejectedCount · 1` | `artifacts/tests/IG-ATTACK-004.json` | **PASS** |
| 5. Partial Failure | Payment succeeds; fulfillment forced to fail | Step Functions compensation state | `1 refund compensation`, `0 silently completed orders` | `IdemGate/ProofGate · SagaCompensationCount · 1` | `artifacts/tests/IG-ATTACK-005.json` | **PASS** |
| 6. Replay Attack | Correctly signed Stripe webhook replayed `48h` late | HMAC timestamp gate, `300s` window | `HTTP 401`, `0 queue messages`, `0 ledger writes` | `IdemGate/ProofGate · ReplayRejectedCount · 1` | `artifacts/tests/IG-ATTACK-006.json` | **PASS** |
| 7. Concurrency Race | `50` workers submit one intent simultaneously | DynamoDB conditional intent claim | `1 winner`, `49 blocked`, `1 charge` | `IdemGate/ProofGate · DuplicateBlockedCount · 49` | `artifacts/tests/IG-ATTACK-007.json` | **PASS** |
| 8. Ledger Invariant | Query successful and non-success terminal intents | DynamoDB ledger + authoritative outcome journal | Safety: `max successes/intent = 1`; liveness: `lost accepted-success intents = 0`; unexpected non-success terminal successes: `0` | `IdemGate/ProofGate · InvariantViolationCount · 0` | `artifacts/tests/IG-ATTACK-008.json` | **PASS** |

## Promotion Receipt

```text
INVARIANT VERDICT: PASS
MAX SUCCESSFUL SIDE EFFECTS PER INTENT: 1
LOST ACCEPTED-SUCCESS INTENTS: 0
UNEXPECTED NON-SUCCESS TERMINAL OUTCOMES: 0
MANDATORY GATES: 8/8 PASS
MISSING EVIDENCE: 0
PROMOTION GATE: GO

BUNDLE: s3://idemgate-evidence-123456789012-us-east-1/runs/idemgate-prod-us-east-1-20260610T084215Z-a91c7e2/idemgate-evidence-bundle.json
SHA-256: 69745b38ca09243e91c77da28d8911cee05761f6f68ea9f330ea732640658d80
KMS SIGNATURE: verified after upload
S3 OBJECT LOCK: COMPLIANCE, retain until 2027-06-10T08:47:31Z
```

**VERDICT: GO — bundle signed, retrieved, verified, and Object Lock recorded.**
