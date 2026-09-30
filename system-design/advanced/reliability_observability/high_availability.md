# High Availability

## The One-Line Summary

> High availability means minimizing downtime through redundancy at every layer — components, data centers, and regions. Every single point of failure is a risk.
> 

---

## The Nines — Know These for Interviews

| Uptime | Downtime per year | Downtime per month |
| --- | --- | --- |
| **99%** (2 nines) | 3.65 days | 7.3 hours |
| **99.9%** (3 nines) | 8.7 hours | 43.8 minutes |
| **99.99%** (4 nines) | 52.6 minutes | 4.4 minutes |
| **99.999%** (5 nines) | 5.26 minutes | 26 seconds |

5 nines is the gold standard for critical systems — banking, telecom, payments.

---

## What Causes Downtime

**Unplanned:** Server crashes, hardware failures, network outages, bugs, external dependency failures, DDoS.

**Planned:** Deployments, database upgrades, maintenance windows.

**The goal:** Eliminate both through redundancy and zero-downtime deployment strategies.

---

## Eliminating Single Points of Failure

Every component that can go down without a backup is a single point of failure.

| Component | Single point of failure? | Solution |
| --- | --- | --- |
| Load balancer | Yes — goes down, no traffic reaches servers | Active-passive or active-active LB pair |
| Application servers | No — if you have 3+ | Auto-scaling group, health checks |
| Database | Yes — goes down, entire app loses data access | Primary-replica with automatic failover |
| Redis cache | No — cache miss hits database, degrades but doesn't break | ElastiCache Multi-AZ for performance |

**Redundancy must be applied at every layer.** App servers redundant but single database = database is still the single point of failure.

---

## Multi-Region — Beyond One Data Center

Single data center redundancy isn't enough. Power outage, network cut, natural disaster can take an entire region down.

### Active-Passive

- One region handles all traffic
- Second region on standby, replicating data
- Failover: seconds to minutes when primary goes down
- Simpler and cheaper
- Risk: data loss if replication lagged at failover time

### Active-Active

- Both regions handle traffic simultaneously
- Users routed to nearest region — lower latency
- No failover needed — one region dies, other absorbs all traffic
- Complex — concurrent writes in both regions create consistency challenges
- More expensive — double infrastructure

### Which to use

| Component | Recommendation |
| --- | --- |
| Stateless services | Active-active — any region handles any request |
| Databases (critical) | Active-passive with fast automatic failover |
| Databases (non-critical) | Active-active with eventual consistency |

---

## The Four Pillars of High Availability

1. **Redundancy** — no single points of failure at any layer
2. **Replication** — data copied across multiple nodes and regions
3. **Multiple components** — load balancers, app servers, databases all have backups
4. **Multi-region** — survive entire data center failures

---

## When NOT to Chase Five Nines

- **Consistency over availability** — financial data can't be stale. CAP theorem: choose CP, accept lower availability.
- **Cost** — each additional nine roughly doubles infrastructure cost. Three nines (8.7 hrs/year) is acceptable for many systems at a fraction of the cost.
- **Per-component decision** — not every part of your system needs the same availability. Payment service needs 5 nines. Internal reporting dashboard needs 2 nines.

---

## Three-Question Ritual

**What does high availability mean and what does it require?**

Minimizing downtime (targeting 5 nines for critical systems) through redundancy at every layer — multiple components, replication, and multi-region deployment to eliminate every single point of failure.

**What breaks if only the application layer is redundant?**

The database becomes the single point of failure. All redundant app servers depend on one database — it goes down, entire system goes down.

**When NOT to design for five nines?**

When consistency matters more than availability (financial systems). When cost is prohibitive — each nine doubles infrastructure spend. Design per-component based on data criticality, not one SLA for everything.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Multi-AZ redundancy | RDS Multi-AZ, ElastiCache Multi-AZ |
| Auto-scaling app servers | EC2 Auto Scaling Groups |
| Multi-region active-passive | Route 53 failover routing |
| Multi-region active-active | Route 53 latency routing + Global Accelerator |
| Automatic database failover | RDS Multi-AZ, Aurora Global Database |
| Uptime monitoring | CloudWatch + Route 53 health checks |