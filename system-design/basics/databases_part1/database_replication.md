# Database Replication

## The One-Line Summary

> Replication copies your database across multiple nodes — one primary handles writes, multiple replicas handle reads. Solves read scaling and eliminates single point of failure.
> 

---

## The Problem Without Replication

One database handles all reads and writes. As traffic grows:

- **Performance degrades** — reads and writes compete for the same resources
- **Single point of failure** — database goes down, entire app goes down
- **No redundancy** — no failover, no backup

Most web apps are **90% reads, 10% writes**. Replication is the natural solution — separate read and write traffic across multiple nodes.

---

## Primary-Replica Replication

**Primary (Master):** Handles all writes. Single source of truth.

**Replicas (Slaves):** Handle all reads. Stay in sync with the primary.

```
App → writes → Primary
App → reads  → Replica 1
                Replica 2
                Replica 3
```

> Replicas can have their own indexes optimized for read queries without affecting write performance on the primary.
> 

---

## Synchronous vs Asynchronous Replication

|  | Synchronous | Asynchronous |
| --- | --- | --- |
| **How it works** | Primary waits for replica to confirm before returning success | Primary returns success immediately, syncs to replicas in background |
| **Write speed** | Slower — waits for replica confirmation | Faster — returns immediately |
| **Consistency** | Strong — replica always up to date | Eventual — replica may lag behind (replication lag) |
| **Data loss risk** | None | Yes — if primary crashes before sync completes |
| **Use when** | Financial data, critical records | Most web apps where slight lag is acceptable |

**Replication lag** — the delay between a write hitting the primary and appearing on replicas. Usually milliseconds, but can grow under heavy load.

---

## Failover — What Happens When Primary Goes Down

1. Health checks detect primary is down
2. System promotes the most up-to-date replica to primary
3. Write traffic redirected to new primary
4. Old replicas start syncing from new primary

**Hard problems with failover:**

- **Which replica gets promoted?** The one with least replication lag — but detecting this automatically is complex
- **Write gap** — seconds where no primary exists, writes may fail or queue
- **Split brain** — if old primary comes back online, two nodes think they're primary. Dangerous.

**In practice:** Use managed solutions — AWS RDS Multi-AZ handles promotion, health checks, and failover automatically.

---

## Scaling Reads with Replicas

Read traffic doubled? Add more replicas. Horizontal scaling for reads is almost unlimited — just keep adding replicas.

This is why replication is usually the **first scaling move** for read-heavy systems — simpler than sharding, handles most read scaling problems.

---

## When NOT to Use Replication

- **Write-heavy systems** — all writes still go to one primary. Replication doesn't help write bottlenecks. Use sharding instead.
- **Early stage apps** — low traffic doesn't justify the operational complexity of managing multiple nodes.

---

## Three-Question Ritual

**What problem does replication solve?**

A single database becomes a bottleneck for both reads and writes, and is a single point of failure. Replication separates read and write traffic across multiple nodes and provides redundancy through failover.

**What breaks without replication?**

Single point of failure — one database goes down, entire app goes down. Also, read and write traffic compete on one node, degrading performance for everyone as traffic grows.

**When NOT to use replication?**

Write-heavy systems — replication only scales reads. Early stage apps where complexity isn't justified yet.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Primary-Replica replication | Amazon RDS Read Replicas |
| Automatic failover | Amazon RDS Multi-AZ |
| Managed replication at scale | Amazon Aurora (up to 15 read replicas, automatic failover) |