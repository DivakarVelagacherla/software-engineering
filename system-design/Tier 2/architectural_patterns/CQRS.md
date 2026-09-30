# Command Query Responsibility Segregation (CQRS)

## The One-Line Summary

> CQRS separates your read model from your write model so each can be optimized independently. Writes go to a normalized database. Reads go to a pre-computed, denormalized store.
> 

---

## The Problem

One database handles both reads and writes. At scale:

- Complex read queries (joining 5 tables for a product page) compete with writes
- Indexing speeds up reads but slows down writes
- Schema optimized for writes is wrong for reads
- Can't optimize for both simultaneously — the database becomes the bottleneck for everything

---

## What CQRS Is

**Command Query Responsibility Segregation** — plain English: separate write responsibility from read responsibility.

- **Command** — anything that changes state. Place order, add review, update stock.
- **Query** — anything that reads state. Get product page, get reviews, get analytics.

Each gets its own dedicated model, optimized for its specific needs.

---

## The Two Models

### Write Model (Command Side)

- **Database:** Normalized, relational — PostgreSQL
- **Optimized for:** Data integrity, ACID transactions, business rules
- **Schema:** Designed around how data is stored correctly

### Read Model (Query Side)

- **Database:** Denormalized, pre-computed — Redis, Elasticsearch, or read-optimized DB
- **Optimized for:** Speed, serving UI directly with no joins needed
- **Schema:** Designed around what the UI needs to display

**Key difference from simple replication:** Replication copies the same schema. CQRS read model is a completely different structure — pre-joined, pre-aggregated, ready to serve.

**Example — product page read model:**

```json
{
  "productId": "123",
  "name": "iPhone 15",
  "avgRating": 4.7,
  "reviewCount": 1234,
  "stockAvailable": true,
  "relatedProducts": [...],
  "recentActivity": [...]
}
```

One read, no joins, instant response.

---

## Keeping the Read Model in Sync

Write model publishes events when state changes. A background listener updates the read model.

```
User places order
    ↓
Write model (PostgreSQL) — saves order
    ↓
Publishes "OrderPlaced" event
    ↓
Event handler updates read model
    ↓
Read model (Redis) — pre-computed view updated
```

**Not polling — event-driven.** Each service publishes events on change. Read model updater reacts instantly. This is why CQRS and event-driven architecture almost always appear together.

---

## The Consistency Tradeoff

Read model is **eventually consistent** — updated milliseconds after the write, not instantly.

**Not all queries go to the read model:**

- Eventually consistent queries (product page, feed, analytics) → read model
- Critical queries needing fresh data (payment confirmation, account balance) → write model directly

---

## When to Use CQRS

| Use CQRS when | Avoid CQRS when |
| --- | --- |
| Read and write requirements conflict | System is simple, queries are straightforward |
| Complex read views joining many tables | Small team — two models adds operational overhead |
| Read traffic vastly outnumbers writes | Strong consistency required everywhere |
| Need pre-computed analytics or dashboards | Eventual consistency is unacceptable for all queries |

---

## Three-Question Ritual

**What problem does CQRS solve?**

The conflict between read and write requirements on the same database. Reads need denormalized, pre-joined fast data. Writes need normalized, ACID-compliant data. CQRS separates them so each is optimized independently.

**What breaks without CQRS at scale?**

Complex read queries compete with writes on the same database. Performance degrades for both. Can't optimize one without hurting the other. Database becomes the bottleneck.

**When NOT to use CQRS?**

Simple systems with straightforward queries. Small teams where two models add unnecessary complexity. When strong consistency is required for all queries — eventual consistency of the read model makes it pointless.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Write model | Amazon RDS (PostgreSQL) |
| Read model | ElastiCache (Redis), Amazon OpenSearch |
| Event sync mechanism | Amazon EventBridge, Amazon Kinesis, Amazon MSK |
| Pre-computed views | DynamoDB (denormalized documents) |