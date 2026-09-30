# Distributed Transactions

## The One-Line Summary

> Distributed transactions maintain atomicity across multiple services with separate databases. Two approaches: 2PC (strong consistency, blocking) and Saga (eventual consistency, no blocking).
> 

---

## The Problem

A user places an order. Three things must happen atomically:

1. Deduct from inventory
2. Charge payment
3. Create order record

Each lives in a **different service with its own database** (microservices architecture). You can't wrap them in one SQL transaction.

**What breaks without handling this:**

- Payment charged but order never created
- Inventory deducted but payment failed
- Retries cause duplicate charges

---

## Why Database Per Service?

In microservices, each service owns its own database. This is deliberate:

- **Autonomy** — teams change schemas without breaking other teams
- **Independent scaling** — inventory might need Cassandra, payments need PostgreSQL
- **Isolation** — one database going down doesn't take everything down

Tradeoff: distributed transactions become your problem.

---

## Approach 1 — Two Phase Commit (2PC)

### How it works

**Phase 1 — Prepare:**

```
Coordinator → Inventory: "Can you deduct 1 iPhone? Lock it."
Coordinator → Payment: "Can you charge $999? Lock it."
Coordinator → Order: "Can you create order #123? Lock it."

All three: "Yes, ready. Resources locked."
```

Each service locks its resources and writes the pending operation to its WAL. Does not commit yet.

**Phase 2 — Commit (if all said yes):**

```
Coordinator → All: "Commit now."
All three: Commit and release locks.
```

**Phase 2 — Rollback (if anyone said no):**

```
Coordinator → All: "Rollback. Undo everything."
```

### The Coordinator

A separate service or component that orchestrates the two phases. Can be:

- A dedicated coordination service
- The initiating service acting as coordinator
- A distributed transaction manager (XA protocol)

### Problems with 2PC

- **Blocking** — all services lock resources between Phase 1 and Phase 2. If coordinator crashes, services are frozen waiting for instructions.
- **Single point of failure** — coordinator crash mid-transaction requires complex recovery.
- **Latency** — two network round trips per transaction.

---

## Approach 2 — Saga Pattern

Break the transaction into a sequence of smaller local transactions. Each service does its part and publishes an event. The next service picks up the event.

**Happy path:**

```
1. Inventory deducted → publishes "inventory reserved"
2. Payment charged → publishes "payment completed"
3. Order created → publishes "order confirmed"
```

**If step 3 fails — compensating transactions:**

```
Order service → publishes "order failed"
Payment service → refunds the charge
Inventory service → restores the stock
```

No locking. No coordinator. Each service acts independently and undoes its own work if needed.

---

## 2PC vs Saga

|  | 2PC | Saga |
| --- | --- | --- |
| **Consistency** | Strong — all or nothing | Eventual — compensating transactions |
| **Blocking** | Yes — locks resources across services | No — services act independently |
| **Failure handling** | Coordinator rolls back everything | Compensating transactions undo each step |
| **Coordinator** | Required — single point of failure | Not needed |
| **Scale** | Poor — more services = more blocking | Good — scales across many services |
| **Use when** | Strong consistency required, few services | Microservices, eventual consistency acceptable |

**In practice:** Saga dominates modern microservices. 2PC is rare.

---

## Idempotency — Handling Retries Safely

Saga uses retry logic. Without idempotency, retries cause duplicate operations — customer charged twice, inventory deducted twice.

**Solution: Idempotency keys**

- Client generates a unique key per transaction attempt (UUID + user ID + timestamp)
- Server stores processed keys
- If same key arrives again — return previous result, don't process again

**Real world:** Stripe requires an idempotency key on every payment API call. Retry as many times as you want — the charge happens exactly once.

---

## Three-Question Ritual

**What problem do distributed transactions solve?**

Maintaining atomicity across multiple services with separate databases. Ensures either all steps succeed or the system recovers cleanly — without a shared database to wrap everything in one ACID transaction.

**What breaks without distributed transaction handling?**

Inconsistency — payment charged but order never created. Duplicate operations — retries without idempotency keys charge customers twice.

**When to choose 2PC over Saga?**

When strong consistency is required and only a few services are involved. Saga scales better but is eventually consistent — 2PC is blocking but guarantees atomicity.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Saga orchestration | AWS Step Functions — orchestrates multi-service workflows with compensation |
| Event-driven Saga | Amazon EventBridge + SQS — choreography-based Saga |
| Idempotency | Built into AWS SDK retries with idempotency tokens |