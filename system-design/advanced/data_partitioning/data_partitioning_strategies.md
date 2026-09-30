# Data Partitioning Strategies

## The One-Line Summary

> Partitioning splits a large table into smaller chunks so queries only scan the relevant partition. The partition key choice determines whether it helps or hurts.
> 

---

## Quick Recap

**Partitioning** — splitting a large table into smaller virtual partitions within the same database cluster. Transparent to the application — you query the table normally, the database routes to the right partition automatically.

**Sharding** — splitting data across multiple database clusters. Application must know which cluster to query.

---

## The Three Strategies

### 1. Range Partitioning

Split by a range of values. Orders by month, users by ID range.

```
Partition 1: orders Jan 2024
Partition 2: orders Feb 2024
Partition 3: orders Mar 2024
```

**Best for:** Time-series data, natural ordering, range queries are common.

**Problem — Data Skew:** If 80% of orders happen in Nov-Dec (holiday season), those partitions are overloaded while others sit idle. Hot partition problem.

**Solutions for skew:**

- Smaller partitions for hot ranges — split Nov/Dec into weekly partitions
- Sub-partitioning — partition by month, then by category within month
- Switch to hash if skew is severe

### 2. Hash Partitioning

Hash the partition key, result determines which partition. Guarantees even distribution.

**Best for:** Even distribution needed, point lookups by key, no range queries.

**Problem — Range Queries:** A query like "orders from last month" must hit ALL partitions and aggregate — scatter-gather. The hash destroyed time ordering, so no partition can be skipped.

### 3. Directory-Based Partitioning

A lookup table (directory) maps each record to its partition. Flexible — any logic, any assignment.

```
User 1-1000     → Partition A (small customers grouped)
User 1001-5000  → Partition B (small customers grouped)
User 5001       → Partition C (enterprise customer, dedicated)
User 5002       → Partition D (enterprise customer, dedicated)
```

**Best for:** Hot/cold data separation, multi-tenant SaaS, custom routing logic.

**Real example — multi-tenant SaaS:** 10,000 small customers + 5 large enterprise customers generating 70% of traffic. Give each enterprise customer their own dedicated partition. Group small customers in shared partitions. Hot customers isolated, no interference.

**Problems:**

- Directory lookup overhead — every query hits directory first
- Directory is a single point of failure — must be highly available

---

## Transparent to Application

```sql
SELECT * FROM orders WHERE user_id = 123
-- Database internally routes to correct partition
-- Application code never changes
```

Unlike sharding, partitioning is completely transparent. No application-level routing logic needed.

---

## Decision Framework

| Strategy | Use when |
| --- | --- |
| **Range** | Time-series data, natural ordering, range queries are primary |
| **Hash** | Even distribution needed, point lookups, no range queries |
| **Directory** | Hot/cold separation, multi-tenant isolation, custom routing needed |

---

## Three-Question Ritual

**What problem does partitioning solve?**

Large tables make even indexed queries slow because the index becomes massive. Partitioning keeps chunks small so queries scan only the relevant partition, not billions of rows.

**What breaks with the wrong partition key?**

Scatter-gather queries — database must hit all partitions and aggregate. Or hot partitions — one partition overloaded while others idle. Either defeats the purpose and adds complexity for no benefit.

**When NOT to partition?**

Small tables where full scans are fast enough. Unpredictable query patterns that don’t align with any single partition key — partitioning helps one pattern at the expense of others.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Range partitioning | PostgreSQL native partitioning on RDS |
| Hash partitioning | PostgreSQL hash partitioning, DynamoDB partition key |
| Directory-based | DynamoDB with custom partition key design |
| Time-series partitioning | Amazon Timestream (built-in time partitioning) |