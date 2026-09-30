# Event Sourcing

## The One-Line Summary

> Instead of storing current state, Event Sourcing stores every event that led to that state. You get complete history, audit trail, and the ability to reconstruct any past state.
> 

---

## The Problem With Traditional State Storage

Traditional database stores one row per entity:

```
account_id: 123
balance: 350
```

Every write overwrites the previous state. History is gone. When a bug causes a wrong balance, you have no way to investigate how it got there.

---

## What Event Sourcing Is

Instead of storing current state, store every event that caused state to change.

```
AccountOpened    → balance: $0
MoneyDeposited   → amount: $500
MoneyWithdrawn   → amount: $100
MoneyDeposited   → amount: $200
MoneyWithdrawn   → amount: $250
```

Current balance = replay all events = $0 + $500 - $100 + $200 - $250 = **$350**

**Key rule: Events are immutable. Never update or delete them.** If a mistake was made, append a corrective event ("TransactionReversed") — never edit the past.

---

## The Data Model

```sql
-- Event store
CREATE TABLE account_events (
    event_id      UUID PRIMARY KEY,
    account_id    VARCHAR(50),
    event_type    VARCHAR(50),      -- 'MoneyDeposited', 'MoneyWithdrawn'
    amount        DECIMAL(10,2),
    occurred_at   TIMESTAMP,
    metadata      JSONB
);

-- Snapshot store
CREATE TABLE account_snapshots (
    account_id    VARCHAR(50),
    balance       DECIMAL(10,2),
    snapshot_at   TIMESTAMP,
    last_event_id UUID              -- which event this snapshot covers up to
);
```

**Read flow:**

1. Fetch latest snapshot for `account_id`
2. Fetch all events after `last_event_id`
3. Replay events against snapshot balance → current balance

**Write flow:**

1. Validate command (sufficient funds?)
2. Append new event — never update existing rows
3. Background process updates read model / snapshot

---

## The Replay Mechanism

```python
balance = snapshot.balance  # start from latest snapshot
for event in events_after_snapshot:
    if event.event_type == 'MoneyDeposited':
        balance += event.amount
    elif event.event_type == 'MoneyWithdrawn':
        balance -= event.amount
return balance
```

Simple, deterministic, auditable. Same events always produce the same result.

---

## Snapshots — Solving the Performance Problem

Replaying 10 years of events on every read is too slow. Solution: periodic snapshots.

```
Snapshot at Jan 1: balance = $1,200
Events after Jan 1:
  MoneyDeposited → $500
  MoneyWithdrawn → $200

Current balance = $1,200 + $500 - $200 = $1,500
```

Snapshot frequency depends on event volume. High volume — snapshot often. Low volume — less often. Goal: keep the replay window small.

---

## Event Sourcing + CQRS

They almost always go together:

- **Event store** = write model
- **Snapshot/pre-computed state** = read model
- **Events** = sync mechanism between write and read

Background process replays events, updates snapshots and read model. Read side always has fast, pre-computed data ready.

---

## Time Travel — The Killer Feature

Since every event is stored, you can reconstruct system state at any point in time.

Real use cases:

- **Debugging** — "what was the account state at 3pm Tuesday when the bug occurred?"
- **Regulatory compliance** — 7 years of immutable transaction history
- **Dispute resolution** — replay exact transaction history to prove what happened
- **Analytics** — replay historical events to predict trends
- **A/B testing** — replay history through new code to see how it would have behaved

**Git is event sourcing** — every commit is an event, any past state is reconstructable.

---

## Three-Question Ritual

**What problem does Event Sourcing solve?**

Traditional databases overwrite state — history is lost. Event Sourcing stores every state change as an immutable event, giving complete history, audit trail, and the ability to reconstruct any past state.

**What breaks without it in a financial system?**

No audit trail, no debugging capability, no compliance. Can't investigate bugs, can't prove transaction history to regulators or customers disputing charges.

**When NOT to use Event Sourcing?**

Simple CRUD systems with no audit requirement. High write volume where storing every event is expensive. Small teams where the operational complexity of event stores and replay logic isn't justified.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Event store (append-only log) | Amazon DynamoDB, Amazon RDS, Amazon Kinesis (as event log) |
| Snapshot store | DynamoDB or RDS |
| Event streaming to read model | Amazon EventBridge, Amazon Kinesis |
| Read model | ElastiCache (Redis), Amazon OpenSearch |