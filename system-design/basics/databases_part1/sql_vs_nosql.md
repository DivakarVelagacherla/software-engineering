# SQL vs NoSQL

## The One-Line Summary

> SQL gives you strict structure and guarantees. NoSQL gives you flexibility and horizontal scale. Pick based on what your data needs, not what's trendy.
> 

---

## Why NoSQL Was Invented

SQL databases were built in the 1970s assuming all data lives on **one machine**. In the 2000s, companies like Google, Amazon, and Facebook hit a wall — their data was growing faster than any single machine could handle. Joins across machines are painful, transactions across machines are nearly impossible. NoSQL was invented to solve this: store and retrieve massive data across many machines without the relational model getting in the way.

---

## SQL — The Relational Model

**Core idea:** Data lives in tables. Tables are linked by foreign keys. The database enforces that relationships are always valid.

**Example:** An `orders` table has a `user_id` foreign key. The database guarantees every order points to a real user. You can never have an orphan order.

**What SQL gives you — ACID:**

| Letter | What it means | Why it matters |
| --- | --- | --- |
| **A — Atomicity** | All or nothing. A transaction either fully completes or fully rolls back. | A bank transfer debits AND credits — or neither happens. Never half. |
| **C — Consistency** | Data always moves from one valid state to another. | No money is created or destroyed during a transfer. |
| **I — Isolation** | Concurrent transactions don't interfere with each other. | Two users trying to book the last flight seat — only one succeeds. The other is blocked until the first completes. |
| **D — Durability** | Once committed, data survives crashes and restarts. | Your payment record doesn't disappear if the server reboots. |

**The scaling problem:** SQL was designed for one machine. Splitting a SQL database across machines breaks joins and makes ACID transactions extremely hard. This is SQL's fundamental horizontal scaling limitation.

---

## NoSQL — The Non-Relational Model

**Core idea:** Trade strict consistency and structure for speed, scale, and flexibility.

**What NoSQL gives you — BASE (instead of ACID):**

| Letter | What it means |
| --- | --- |
| **BA — Basically Available** | System always responds, even if some data is slightly stale |
| **S — Soft state** | Data may be in a transitional state between updates |
| **E — Eventual consistency** | All nodes will eventually agree on the same data — just not instantly |

**When eventual consistency is fine:** Like counts, view counts, activity feeds. Nobody cares if the like count is 999 vs 1000 for half a second.

**When eventual consistency is NOT fine:** Payments, inventory, seat reservations. Correctness is non-negotiable.

---

## The Four NoSQL Types

### 1. Document Stores

**What it stores:** JSON-like documents. Each document can have different fields — no fixed schema required.

**Best for:** Flexible, nested data where structure varies between records.

**Examples:** MongoDB, CouchDB

**Use case:** Product catalog (a shirt has size/color, a laptop has RAM/storage — different fields per product)

### 2. Key-Value Stores

**What it stores:** Just a key and a value. The simplest NoSQL type.

**Best for:** Fast lookups where you always know the exact key.

**Examples:** DynamoDB, Redis

**Use case:** Session storage, caching, feature flags

### 3. Wide-Column Stores

**What it stores:** Rows where each row can have completely different columns. No wasted space for missing data.

**Best for:** Massive write-heavy workloads and time series data.

**Examples:** Cassandra, HBase

**Use case:** Netflix viewing history, Uber trip data, IoT sensor readings

**Why it's fast:** You append new columns (e.g. timestamps) rather than inserting new rows. Reads across one user's history are extremely fast.

### 4. Graph Databases

**What it stores:** Nodes (entities) and edges (relationships between them). Relationships are first-class citizens.

**Best for:** Traversing connections between entities — friends of friends, recommendations.

**Examples:** Neo4j

**Use case:** Social networks (LinkedIn connections), recommendation engines

---

## Quick Decision Guide

| Scenario | Pick this |
| --- | --- |
| Payments, banking, reservations | SQL — ACID is non-negotiable |
| User profiles, product catalogs | Document — MongoDB |
| Session storage, caching | Key-Value — DynamoDB or Redis |
| Social graph, recommendations | Graph — Neo4j |
| Time series, logs, IoT, analytics | Wide-Column — Cassandra |
| You need joins and referential integrity | SQL |
| You need to scale writes to millions/sec | NoSQL (Cassandra or DynamoDB) |

---

## Polyglot Persistence

**The insight:** You don't have to pick just one. Real production systems often use multiple databases — each chosen for what it does best.

**Example — Instagram:**

- User accounts → **PostgreSQL** (ACID, relational)
- Posts and comments → **MongoDB** (flexible, nested)
- Sessions → **Redis** (fast, temporary)
- Social graph → **Neo4j** (who follows whom)
- Analytics → **Cassandra** (massive write volume)

> In interviews, saying "I'd use polyglot persistence here" signals you think in tradeoffs, not absolutes.
> 

---

## Three-Question Ritual

**What problem does NoSQL solve?**

Storing and retrieving massive data across many machines — without the constraints of the relational model that make horizontal scaling painful.

**What breaks if you use NoSQL for payments?**

ACID breaks. Atomicity: a transfer could debit one account without crediting the other. Isolation: two transactions could both read the same balance and both succeed, causing an overdraft.

**When NOT to use SQL?**

When you need to scale writes massively across many machines, when your data has no meaningful relationships, or when your data structure is flexible and changes frequently.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| SQL database | Amazon RDS (PostgreSQL, MySQL) or Amazon Aurora |
| Key-Value NoSQL | Amazon DynamoDB |
| In-memory Key-Value | Amazon ElastiCache (Redis) |
| Wide-Column | Amazon Keyspaces (Cassandra-compatible) |