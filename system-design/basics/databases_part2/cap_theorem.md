# CAP Theorem

## The One-Line Summary

> CAP Theorem says during a network partition, you can only choose one: consistency or availability. This is a design framework, not a technology.
> 

---

## First — Clear Up the Terminology

**Database Partitioning:** Splitting a large table into smaller chunks for performance. Nothing to do with CAP.

**Database Sharding:** Splitting data across multiple servers for scale. Nothing to do with CAP.

**Network Partition:** When the network connection between two servers breaks. The servers are fine, the data is fine — but they can’t communicate with each other. Like two offices with broken phone lines — both still open, but can’t share information.

---

## The Three Properties

| Letter | Property | What it means |
| --- | --- | --- |
| **C — Consistency** | Every read gets the most recent write | All nodes return the same data at the same time |
| **A — Availability** | Every request gets a response | System always responds, even if data might be stale |
| **P — Partition Tolerance** | System keeps working during network partitions | Not optional — networks fail, you must tolerate partitions |

**The real choice:** Since Partition Tolerance is mandatory, the decision is always **CP vs AP** — when a network partition occurs, do you prioritize Consistency or Availability?

---

## CP vs AP — The Core Decision

### CP — Consistency + Partition Tolerance

*“I’d rather give you no answer than a wrong answer.”*

- System returns an error or times out during a partition
- No stale data ever served
- Users may get errors but data is always correct
- **Use for:** Banking, payments, inventory, anything where wrong data is worse than no data

### AP — Availability + Partition Tolerance

*“I’d rather give you something than nothing, even if it’s slightly old.”*

- System keeps responding during a partition
- May serve stale data until partition heals
- Users always get a response
- **Use for:** Social media, search, recommendations, user profiles — slightly stale data is acceptable, downtime is not

---

## The ATM Analogy

- **CP ATM during outage:** “Sorry, I can’t process your request.” Frustrating but safe — no overdrafts possible.
- **AP ATM during outage:** “Here’s your $200.” But the main system doesn’t know yet — hit another ATM and you might withdraw again.

---

## Eventual Consistency in AP Systems

AP systems are **eventually consistent** — during a partition, nodes serve stale data. Once the partition heals and nodes sync up, all nodes agree on the same data.

**Write conflicts:** If two nodes accept conflicting writes during a partition (e.g. both approve a $200 withdrawal), reconciliation is needed when the partition heals:

- **Last write wins** — most recent timestamp wins
- **Application handles it** — conflict flagged, app logic decides
- **User resolves it** — used in collaborative tools like Google Docs

This is why AP is dangerous for payments — conflicts can’t always be cleanly resolved after the fact.

---

## CAP Is Per-Component, Not Per-Application

The same application can use different consistency models for different components:

| Component | Choice | Database |
| --- | --- | --- |
| Payment service | CP — consistency non-negotiable | PostgreSQL |
| User profile service | AP — stale profile picture is fine | Cassandra |
| Activity feed | AP — slight delay acceptable | Cassandra |
| Inventory system | CP — overselling is catastrophic | PostgreSQL |

> Mentioning per-component CAP decisions in interviews signals senior-level thinking.
> 

---

## Real Databases and Their CAP Choice

| Database | Choice | Why |
| --- | --- | --- |
| PostgreSQL, MySQL | CP | ACID — consistency non-negotiable |
| Cassandra | AP | Always available, eventually consistent |
| DynamoDB | AP by default | Configurable per operation |
| HBase | CP | Strong consistency, may reject during partition |
| Redis | CP by default | Consistency over availability |
| CouchDB | AP | Always responds, syncs when partition heals |

---

## CAP Applies Beyond Databases

CAP theorem applies to any distributed system — CDNs, message queues, microservices. CDNs are AP by nature — edge servers serve stale content when they can’t reach the origin.

---

## Three-Question Ritual

**What problem does CAP theorem solve?**

It’s a framework for making design decisions in distributed systems. When a network partition occurs, it forces you to explicitly choose between consistency and availability rather than assuming you can have both.

**What breaks if you choose AP for payments?**

Both servers can approve conflicting transactions during a partition — overdrafts, double spending, unresolvable write conflicts. Wrong data is worse than no data for money.

**When to choose AP over CP?**

When availability matters more than perfect consistency — social media, user profiles, recommendations, activity feeds. Slightly stale data is acceptable, downtime is not.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| CP database | Amazon RDS (PostgreSQL/MySQL) |
| AP database | Amazon DynamoDB (default), Amazon Keyspaces (Cassandra) |
| Configurable per operation | DynamoDB strongly consistent reads (optional, costs 2x) |