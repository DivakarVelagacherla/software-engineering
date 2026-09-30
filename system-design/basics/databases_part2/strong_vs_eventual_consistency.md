# Strong vs Eventual Consistency

## The One-Line Summary

> Strong consistency guarantees every read gets the latest write. Eventual consistency guarantees all nodes will agree eventually, but may serve stale data in the meantime. Pick based on whether wrong data causes real harm.
> 

---

## The Core Difference

|  | Strong Consistency | Eventual Consistency |
| --- | --- | --- |
| **What it guarantees** | Every read returns the most recent write | All nodes will agree eventually, not immediately |
| **User experience** | Always correct data | May see stale data briefly |
| **Cost** | Higher latency, lower availability | Lower latency, higher availability |
| **During partition** | Rejects requests rather than serve stale data | Serves stale data, syncs when partition heals |

---

## Strong Consistency

Every read confirms with all nodes before responding. No stale data ever served.

**Cost:**

- **Latency** — more round trips between nodes before responding
- **Availability** — during a partition, requests are rejected rather than served

**Use when:** Wrong data causes real, irreversible harm — money, inventory, reservations.

---

## Eventual Consistency

Nodes respond immediately with what they have. All nodes sync in the background and eventually agree.

**Cost:**

- **Stale data** — users may see outdated information briefly

**What stale data looks like to users:**

- Post a tweet, refresh, your tweet doesn’t appear yet
- Update profile picture, friend sees old one for a few seconds
- Add item to cart on one device, not visible on another yet

**Use when:** Brief inconsistency is acceptable and availability matters more than perfect correctness.

---

## Read-Your-Writes Consistency

A middle ground. You always see your own writes immediately. Other users may not see them yet.

**Solves:** The most jarring eventual consistency problem — you post something and it disappears from your own view.

**Use cases:**

- Social media posts — you always see your own posts immediately
- Shopping cart — you always see items you just added
- Profile updates — you always see your own changes
- Comments — you always see your own comments

---

## Quick Decision Guide

| Scenario | Consistency Model | Why |
| --- | --- | --- |
| Bank balance | Strong | Wrong balance = overdrafts, financial loss |
| Instagram follower count | Eventual | Off by a few for a second — nobody cares |
| Flight seat booking | Strong | Overselling = passenger with no seat |
| Product star rating | Eventual | 4.2 vs 4.3 for a moment is acceptable |
| Stock price | Strong | Stale price = wrong trading decisions, financial loss |
| Social media feed | Eventual | Slight delay is acceptable |
| Shopping cart | Read-Your-Writes | You see your own adds, others may lag |

**The pattern:** Anything involving money, inventory, or irreversible actions — strong consistency. Everything else — eventual or read-your-writes.

---

## Conflict Resolution in Eventual Consistency

When two nodes accept conflicting writes during a partition, reconciliation is needed:

- **Last write wins** — most recent timestamp wins
- **Operational Transformation** — used by Google Docs, merges simultaneous edits intelligently
- **Application handles it** — conflict flagged, app logic decides
- **User resolves it** — rare, used in collaborative tools

---

## Three-Question Ritual

**What problem does eventual consistency solve?**

Strong consistency requires all nodes to agree before responding — slow and unavailable during partitions. Eventual consistency lets nodes respond immediately, keeping the system fast and always available at the cost of brief inconsistency.

**What breaks with eventual consistency for flight booking?**

Two users book the last seat simultaneously, both get confirmation. One arrives at the gate with no seat. Overselling caused by stale availability data.

**When to choose strong consistency?**

When wrong data causes real, irreversible harm — money, inventory, reservations, anything where incorrect data is worse than no data.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Strong consistency | Amazon RDS (PostgreSQL/MySQL) |
| Eventual consistency | Amazon DynamoDB (default reads) |
| Strong reads on demand | DynamoDB strongly consistent reads (2x cost) |
| Read-your-writes | DynamoDB — reads after writes on same partition key are strongly consistent by default |