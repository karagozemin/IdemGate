# IdemGate Architecture

IdemGate's correctness target is a business invariant, not queue delivery semantics:

> **One customer intent = exactly one successful side effect — no matter how many times the infrastructure retries, reorders, or replays.**

SQS FIFO deduplication reduces duplicate work. The DynamoDB ledger, conditional state transitions, downstream idempotency keys, and deployed attack evidence form the correctness boundary.

## High-Level Architecture

```mermaid
flowchart LR
    S[Webhook source] -->|HMAC + timestamp| API[API Gateway]
    API --> V[Verifier + normalizer Lambda]
    V -->|conditional claim| D[(DynamoDB idempotency ledger)]
    V -->|eventId / ordering key| Q[SQS FIFO]
    Q --> SF[Step Functions saga]
    SF --> A[Idempotent side-effect adapters]
    A --> J[(Authoritative outcome journal)]
    SF --> EB[EventBridge]
    Q --> DLQ[SQS DLQ]
    CW[CloudWatch / X-Ray / EMF] --- V
    CW --- SF
    CW --> SNS[SNS alarms]
    H[Attack harness] --> API
    H --> P[Invariant proof gate]
    D --> P
    J --> P
    CW --> P
    P -->|hash + KMS sign| E[(S3 Object Lock evidence)]
```

## Happy Path

```mermaid
sequenceDiagram
    participant Source
    participant API as API Gateway
    participant Verify as Verifier Lambda
    participant Ledger as DynamoDB Ledger
    participant Queue as SQS FIFO
    participant Saga as Step Functions
    participant Effect as Side-effect Provider
    Source->>API: signed raw webhook
    API->>Verify: raw body + signature + timestamp
    Verify->>Verify: constant-time HMAC, window, schema, operationHash
    Verify->>Ledger: conditional IN_PROGRESS claim(intentKey)
    Ledger-->>Verify: acquired
    Verify->>Queue: eventId + messageGroupId
    Queue->>Saga: normalized intent
    Saga->>Effect: execute with downstream idempotency key
    Effect-->>Saga: one result
    Saga->>Ledger: conditional SUCCEEDED + resultHash
    Saga-->>Source: prior/stored result on duplicate
```

## Attack And Promotion Pipeline

```mermaid
flowchart TD
    A[Deploy isolated proof environment] --> B[Run 8 mandatory attacks]
    B --> C[Collect ledger, journal, metrics, ARNs, HTTP results]
    C --> D{All observations present?}
    D -->|No| N[NO-GO, exit non-zero]
    D -->|Yes| I{Invariant query: count <= 1?}
    I -->|No| N
    I -->|Yes| H[Canonicalize + SHA-256 + KMS sign]
    H --> O[Upload S3 Object Lock]
    O --> V{Retrieve and verify hash, signature, retention}
    V -->|No| N
    V -->|Yes| G[GO]
    N --> E[Preserve failure evidence]
    G --> E2[Preserve passing evidence]
```

## Scenario, Control, Proof

```mermaid
flowchart LR
    DS[Duplicate storm] --> C1[Conditional intent claim] --> P1[1 journal success, 99 blocked]
    OO[Out of order] --> C2[Conditional legal transition] --> P2[0 wrong fulfillments]
    TO[Timeout retry] --> C3[Provider idempotency key + reconcile] --> P3[1 provider result]
    PP[Poison payload] --> C4[Ingress schema and size limits] --> P4[Clean ledger / isolated DLQ]
    PF[Partial failure] --> C5[Saga compensation / NEEDS_REPAIR] --> P5[Terminal state + alarm]
    RA[Replay attack] --> C6[HMAC timestamp window] --> P6[Rejected before queue]
    CR[Concurrency race] --> C7[DynamoDB conditional write] --> P7[Exactly 1 winner]
    IC[Invariant check] --> C8[Ledger + outcome journal query] --> P8[0 violating intents]
```

## Component Responsibilities

| Component | Responsibility | Recorded proof | Pillars |
|---|---|---|---|
| API Gateway | Bounded signed-webhook ingress, throttling, size limits | HTTP result and access log | Security, Performance Efficiency |
| Verifier Lambda | Raw-body HMAC, timestamp window, normalization, operation hash | Rejection metric, trace, request ID | Security, Reliability |
| DynamoDB ledger | Atomic intent ownership and legal transitions | Item snapshot and condition/transaction result | Reliability, Security |
| SQS FIFO + DLQ | Ordered retry transport and poison isolation | receive counts, DLQ metrics | Reliability, Operational Excellence |
| Step Functions saga | Durable orchestration and compensation | execution ARN and terminal state | Reliability, Operational Excellence |
| Side-effect adapter + journal | Provider idempotency and authoritative outcome count | downstream key, result hash, journal count | Reliability |
| EventBridge | Post-commit internal fanout | matched/failed invocation metrics | Reliability, Performance Efficiency |
| CloudWatch / X-Ray / SNS | Metrics, traces, alarms, notification | datapoints and alarm state | Operational Excellence |
| KMS / Secrets Manager | Secret and evidence-signing protection | key ARN and verification result | Security |
| Attack harness | Deployed adversarial scenarios | verbatim redacted inputs and observations | Reliability, Operational Excellence |
| S3 Object Lock | Immutable evidence retention | object version, mode, retention date | Security, Sustainability |

## Well-Architected Mapping

| Pillar | IdemGate design |
|---|---|
| Operational Excellence | Automated attacks, machine gate, dashboards, runbooks, rollback, deterministic evidence |
| Security | HMAC timestamp verification, least privilege, KMS, Secrets Manager, encryption, redaction, Object Lock |
| Reliability | Conditional ledger writes, downstream idempotency, FIFO/DLQ, saga compensation, invariant query |
| Performance Efficiency | Serverless scaling, DynamoDB on-demand, asynchronous work, explicit ordering groups |
| Cost Optimization | Default USD 200/month ceiling, budgets, TTL, retention controls, on-demand services |
| Sustainability | Event-driven execution, ephemeral proof environments, TTL, right-sized retention |

## Failure Semantics

- `IN_PROGRESS` means one worker owns a bounded lease; it is not permission for another worker to repeat an ambiguous side effect.
- `SUCCEEDED` is terminal and returns the prior result for matching inputs.
- `FAILED` is safe only when no successful side effect remains.
- `NEEDS_REPAIR` stops automation when reconciliation or compensation requires a human.
- A reused `intentKey` with a different `operationHash` is a tampering/conflict error.
- Any missing proof field or unverifiable outcome count makes promotion `NO-GO`.

