# ACID vs BASE

## The One-Line Summary

> ACID and BASE are two opposing database philosophies. ACID prioritizes correctness and consistency. BASE prioritizes availability and scale. Pick based on what your data can tolerate.
> 

---

## The Connection to CAP Theorem

- **ACID = CP** — consistency over availability
- **BASE = AP** — availability over consistency

ACID and BASE aren't just properties — they represent fundamentally different design philosophies.

---

## ACID — Deep Dive

### A — Atomicity

**All or nothing. A transaction either fully completes or fully rolls back.**

Example: Transferring $500 from Account A to Account B.

- Debit Account A → success
- System crashes before crediting Account B
- On recovery — the debit rolls back. Account A gets $500 back. Neither operation happened.

The transaction is one indivisible unit. Never half-complete.

### C — Consistency

**Data always moves from one valid state to another. All rules and constraints are enforced.**

> Note: Different from CAP Consistency. ACID Consistency = data integrity rules. CAP Consistency = all nodes see the same data.
> 

Example: Account balance can never go below zero. A withdrawal of $1000 from a $500 account is rejected. The database stays valid.

### I — Isolation

**Concurrent transactions don't interfere with each other.**

Without isolation — **dirty reads**: Transaction 2 reads a value that Transaction 1 is in the middle of updating. Gets stale or invalid data.

With isolation — Transaction 1 locks the row. Transaction 2 waits. Once Transaction 1 commits, Transaction 2 reads the updated value.

**Isolation levels** (strictest to most relaxed):

- **Serializable** — transactions run as if sequential. No dirty reads. Slowest.
- **Repeatable Read, Read Committed, Read Uncommitted** — weaker isolation, better performance, some dirty reads allowed.

For interviews: Know Serializable is the strictest. Weaker levels trade correctness for performance.

### D — Durability

**Once committed, data survives crashes, power failures, anything.**

Mechanism: **Write-Ahead Log (WAL)** — before any change hits the actual data, the operation is first written to a log on disk. On recovery after a crash, the database replays completed transactions and rolls back incomplete ones.

> Same concept as Redis AOF (Session 9) — log every write before confirming success.
> 

---

## BASE — Deep Dive

### BA — Basically Available

System always responds — even during failures or partitions. Response may be stale but you always get one.

### S — Soft State

System state can change over time even without new input — because nodes are still syncing in the background. The state is "soft" — not yet settled.

Example: 3 Cassandra nodes. User updates profile picture. Node 1 has new picture. Nodes 2 and 3 still syncing. At this moment the system is in an inconsistent intermediate state — soft state.

### E — Eventual Consistency

Given enough time with no new updates, all nodes will agree on the same value. Not immediate — but guaranteed to converge.

---

## When to Use Which

| Use ACID when | Use BASE when |
| --- | --- |
| Data loss has real consequences | Brief inconsistency is acceptable |
| Money, inventory, reservations | Social feeds, user profiles, activity logs |
| Cross-row or cross-table transactions needed | Massive write throughput across many nodes |
| Correctness over speed | Availability over consistency |

---

## Common Databases

| Database | Philosophy | Type |
| --- | --- | --- |
| PostgreSQL, MySQL | ACID | SQL |
| Amazon Aurora | ACID | SQL |
| Cassandra | BASE | Wide-Column |
| DynamoDB | BASE (default) | Key-Value |
| Redis | BASE (used for caching) | Key-Value |
| CouchDB | BASE | Document |
| MongoDB | Configurable | Document |

---

## Three-Question Ritual

**What problem does ACID solve?**

Data corruption and inconsistency in concurrent, multi-step transactions. Guarantees data stays valid even when multiple operations happen simultaneously or the system crashes mid-transaction.

**What breaks without ACID in a payment system?**

Dirty reads, partial transactions, overdrafts. Without atomicity — crash mid-transfer leaves one account debited, other never credited. Without isolation — two transactions read the same balance and both succeed, overdrawing the account.

**When NOT to use ACID?**

When availability is the highest priority and brief inconsistency is acceptable — social media feeds, user profiles, activity logs, recommendations. Use BASE databases like Cassandra or DynamoDB instead.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| ACID database | Amazon RDS (PostgreSQL/MySQL), Amazon Aurora |
| BASE database | Amazon DynamoDB, Amazon Keyspaces (Cassandra) |
| Configurable | DynamoDB strongly consistent reads (optional) |