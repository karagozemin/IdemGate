# IdemGate Evidence Bundle

**Evidence class:** `SAMPLE_NOT_DEPLOYMENT_PROOF`  
**Run ID:** `idemgate-prod-us-east-1-20260610T084215Z-a91c7e2`  
**Region:** `us-east-1`  
**Workflow:** Stripe checkout → payment → fulfillment

## Invariant Verdict

> **PASS — one customer intent = exactly one successful side effect**

| Machine check | Recorded value |
|---|---:|
| Mandatory gates passed | `8 / 8` |
| Violating intent keys | `0` |
| Maximum successful side effects per intent | `1` |
| Lost accepted-success intents | `0` |
| Unexpected successes for rejected/compensated/repair intents | `0` |
| Missing evidence fields | `0` |
| Promotion gate | **GO** |

## Proof Receipt

See [PROOF_GATE_RESULTS.md](PROOF_GATE_RESULTS.md) for the complete `INPUT → AWS CONTROL → RESULT → METRIC → ARTIFACT` scoreboard.

The strongest recorded chain:

```text
100 identical Stripe events
  → DynamoDB TransactWriteItems on idemgate-prod-ledger
  → 1 SUCCEEDED + 99 DUPLICATE_BLOCKED
  → CloudWatch IdemGate/ProofGate DuplicateBlockedCount = 99
  → authoritative outcome journal = 1 charge
  → evidence/tests/IG-ATTACK-001.json
  → PASS
```

## Attestation

| Field | Recorded value |
|---|---|
| Bundle | `idemgate-evidence-bundle.json` |
| Canonical payload SHA-256 | `69745b38ca09243e91c77da28d8911cee05761f6f68ea9f330ea732640658d80` |
| KMS signing algorithm | `RSASSA_PSS_SHA_256` |
| Post-upload signature verification | `true` |
| S3 Object Lock mode | `COMPLIANCE` |
| Retain until | `2027-06-10T08:47:31Z` |

This is a realistic sample artifact. A live IdemGate run must replace all illustrative identifiers and attestations with observations retrieved from the deployed AWS environment.
