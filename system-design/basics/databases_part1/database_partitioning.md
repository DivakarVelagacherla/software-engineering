# Database Partitioning

## The One-Line Summary

> Partitioning splits a large table into smaller chunks so queries only scan what they need instead of billions of rows.
> 

---

## The Problem Without Partitioning

A single `orders` table with 10 billion rows accumulated over 10 years. Even with indexes, queries slow down because:

- The index itself becomes massive
- Full scans touch too much data
- Every query competes for the same table resources

Replication helps with read traffic but doesn't solve the problem of the table being too large. Partitioning does.

---

## What Partitioning Is

Partitioning splits one large table into smaller chunks called **partitions**, all managed by one database instance. The database handles routing automatically — you query the `orders` table and the database figures out which partition to look in.

> Partitioning vs Sharding: Partitioning stays on one database instance. Sharding splits across multiple servers. (Session 15)
> 

---

## Partition Key — The Most Important Decision

The partition key determines how data is split. **It must match your most common query patterns.**

For an orders table, most queries have a time filter — "orders from last month", "orders in Q3", "orders in the last 90 days." So timestamp is a better partition key than ID — time-based queries only scan the relevant time partitions, not the entire table.

---

## The Three Partitioning Strategies

### 1. Range Partitioning

Split by a range of values. January orders in partition 1, February in partition 2, etc.

**Best for:** Time-based queries — queries only scan relevant partitions

**Problem: Hot partition** — all new writes go to the current month's partition. Older partitions sit untouched. One partition becomes a bottleneck.

### 2. Hash Partitioning

Run the partition key through a hash function. Result determines which partition. Distributes data evenly across all partitions.

**Best for:** Write-heavy systems where even distribution matters

**Problem:** Time-based queries must scan all partitions — the database doesn't know which partitions contain last month's orders when data is hash-distributed.

### 3. List Partitioning

Split by specific values. US orders in partition 1, India in partition 2, UK in partition 3.

**Best for:** Geographic or categorical splits where queries filter by region or category.

---

## Range vs Hash — The Core Tradeoff

|  | Range Partitioning | Hash Partitioning |
| --- | --- | --- |
| **Time-based queries** | Fast — scans only relevant partitions | Slow — scans all partitions |
| **Write distribution** | Hot partition — all writes go to current partition | Even — writes spread across all partitions |
| **Use when** | Read query speed is the priority | Write throughput is the bottleneck |

**In practice for orders tables:** Most systems use range partitioning by time and manage the hot partition problem by adding replication on top.

---

## Three-Question Ritual

**What problem does partitioning solve?**

A single table grows too large — billions of rows make full table scans slow even with indexes. Partitioning splits the table into smaller chunks so queries only scan the relevant partition.

**What breaks without partitioning?**

Queries become slow even with indexing as the table grows. The index itself becomes massive, and every query touches too much data.

**When to use hash over range partitioning?**

When there's no meaningful time series to partition on, or when write throughput is so high that a hot partition becomes a bottleneck and even distribution matters more than query speed.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Table partitioning | Amazon RDS (PostgreSQL native partitioning) |
| Auto-partitioned time series | Amazon Timestream |
| Hash-distributed tables | Amazon Redshift (distribution keys) |