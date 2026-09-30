# Database Sharding

## The One-Line Summary

> Sharding splits your database across multiple servers to handle write volume and data size that exceeds what one server can handle. It's the last resort — exhaust every other option first.
> 

---

## When Do You Need Sharding?

Exhaust these options first, in order:

1. **Vertical scaling** — bigger server
2. **Caching** — reduce database load
3. **Replication** — scale reads with replicas
4. **Partitioning** — split large tables on one server
5. **Sharding** — only when everything else fails

Sharding solves: write throughput and data volume that exceeds a single server's capacity.

---

## What Sharding Is

Sharding splits data across multiple database servers called **shards**. Each shard holds a subset of the data. Together they hold everything.

```
orders table (10 billion rows)
├── Shard 1 — user_id 1 to 25M
├── Shard 2 — user_id 25M to 50M
├── Shard 3 — user_id 50M to 75M
└── Shard 4 — user_id 75M to 100M
```

---

## The Shard Key — The Most Critical Decision

The shard key determines which row goes to which server. **It must match your most common query pattern.**

**Example:** Sharding orders by `user_id`

- All orders for one user live on one shard
- Query "show me all orders for user 123" hits exactly one shard
- This is called **data locality** — related data lives together

### Shard Key Options

| Type | Example | Use when |
| --- | --- | --- |
| Single column | `user_id`, `order_id` | Simple, most common |
| Composite | `(region, user_id)` | Geographic distribution + data locality |
| Directory-based | Lookup table maps data to shards | Maximum flexibility, used by Pinterest, Airbnb |

> Start simple. Add complexity only when needed.
> 

---

## The Four Big Problems With Sharding

### 1. Hot Shards

If 80% of traffic comes from your top 1000 power users, the shards holding those users get overwhelmed. Uneven data distribution defeats the purpose of sharding.

**Solution:** Choose a shard key with high cardinality and even distribution. Avoid keys that cluster popular data on one shard.

### 2. Resharding

You start with 4 shards using `user_id % 4`. You add a 5th shard — formula becomes `user_id % 5`. Almost every row needs to move to a new shard. With billions of rows, this is a massive, expensive, risky operation.

**Solution:** Consistent Hashing (Session 16) — minimizes data movement when adding/removing shards.

### 3. Cross-Shard Queries (Scatter-Gather)

Query: `SELECT * FROM orders WHERE product_id = 999`

If sharded by `user_id`, the database has no idea which shard has orders for product 999. It must hit all shards, collect results, and merge them in application code. Slow, expensive, gets worse with more shards.

**Solution:** Design shard key to match query patterns. Accept that some queries will be cross-shard.

### 4. No Cross-Shard Transactions

ACID transactions across multiple shards are nearly impossible. If an operation needs to update data on Shard 1 and Shard 2 atomically, you have a distributed transaction problem.

**Solution:** Design data model to avoid cross-shard transactions. Or use a database that handles this automatically.

---

## Manual Sharding vs Managed Databases

Most companies avoid manual sharding. Instead they use databases that handle sharding automatically:

| Database | What it does |
| --- | --- |
| **Cassandra** | Automatic sharding, massive write scale, eventual consistency |
| **DynamoDB** | Automatic sharding, managed by AWS |
| **CockroachDB** | Automatic sharding with ACID transactions |

**Migrating Postgres to Cassandra:** Solves sharding automatically but you lose SQL, joins, and ACID. Only worth it if your data doesn't need the relational model.

**Rule:** If data needs ACID (payments, inventory) — stay on Postgres. If not (activity logs, user history, messages) — Cassandra handles scale automatically.

---

## Real Companies Using Sharding

| Company | How they shard |
| --- | --- |
| **WhatsApp** | Messages sharded by `chat_id` — data locality per conversation |
| **Uber** | Trip data sharded by region/city |
| **Twitter** | Tweets sharded by `user_id` |
| **Pinterest** | Directory-based sharding on MySQL by `user_id` |
| **Netflix** | Uses Cassandra for user data — automatic sharding |

---

## Three-Question Ritual

**What problem does sharding solve?**

Write throughput and data volume that exceeds what a single database server can handle, after all other scaling options are exhausted.

**What breaks without sharding at that scale?**

Writes become the bottleneck. The primary database is overwhelmed, write latency increases, and the entire app slows down or goes down.

**When NOT to shard?**

When replication, caching, indexing, and partitioning can handle your scale. Sharding introduces significant operational complexity — hot shards, resharding, cross-shard queries, no cross-shard transactions. Only accept this complexity when there is no other choice.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Manual sharding | Amazon RDS (you manage shard logic yourself) |
| Automatic sharding | Amazon DynamoDB |
| Cassandra-compatible | Amazon Keyspaces |
| Distributed SQL with auto-sharding | Amazon Aurora with sharding extensions |