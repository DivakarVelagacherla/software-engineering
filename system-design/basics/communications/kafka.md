# Kafka

## The One-Line Summary

> Kafka is a distributed event log — not just a message queue. Messages are retained, replayable, and consumed independently by multiple consumer groups simultaneously.
> 

---

## Kafka vs SQS — The Fundamental Difference

**SQS:** Message delivered → consumer processes it → message deleted. Gone forever.

**Kafka:** Message written to log → consumers read it → message stays. Any consumer can re-read from any point in time.

**The scenario that reveals the difference:**

Analytics team found a bug. They need to reprocess all orders from the last 30 days. With SQS — impossible, messages are deleted. With Kafka — replay from any offset.

---

## Core Kafka Concepts

### Topics

Messages are organized into **topics** — categories like "orders", "payments", "user-signups".

Think of a topic like a TV channel. Producers broadcast, consumers watch. Multiple consumers watch the same channel simultaneously without affecting each other.

### Partitions

A topic is split into **partitions** — multiple ordered logs on different servers for parallelism and scale.

```
Topic: "orders"
├── Partition 0 — messages 0, 3, 6, 9...
├── Partition 1 — messages 1, 4, 7, 10...
└── Partition 2 — messages 2, 5, 8, 11...
```

- Messages within a partition are **strictly ordered**
- Messages across partitions are **not ordered**

**Partition key:** Determines which partition a message goes to.

- Same key → always same partition → guaranteed ordering for that key
- No key → round-robin distribution

**Why partition key matters:** All events for order #123 (placed → payment confirmed → shipped) must go to the same partition. Otherwise consumers process them out of order — shipping before payment is confirmed.

### Consumer Offsets

Each consumer group tracks its own **offset** — its position in each partition. Independent per group.

- Analytics group at offset 200
- Warehouse group at offset 500
- Fraud detection at offset 600

If a consumer crashes and restarts — it picks up from its last saved offset. No messages lost, no duplicates.

### Consumer Groups

A group of consumers working together to consume a topic. Each partition assigned to exactly one consumer within the group.

```
Topic: orders (3 partitions)
Consumer Group: warehouse-service
├── Consumer 1 → Partition 0
├── Consumer 2 → Partition 1
└── Consumer 3 → Partition 2
```

**Rule:** Number of consumers in a group should never exceed number of partitions. Extra consumers sit idle as standby — take over on rebalance if another consumer crashes.

**Parallelism:** 3 consumers + 3 partitions = 3x throughput. To scale consumption — increase partitions first, then add consumers.

---

## Message Replay — Kafka's Killer Feature

Any consumer group can rewind to any offset and reprocess messages. Enables:

- **Bug fixes** — reprocess historical data after fixing analytics bug
- **New services** — new fraud detection service replays all historical orders to build initial state
- **Auditing** — reconstruct what happened at any point in time
- **Disaster recovery** — consumer lost state, replay from last known good offset

This is why Kafka is an **event log and source of truth** — not just a message transport.

---

## AWS Options for Kafka-style Streaming

| Service | What it is | Use when |
| --- | --- | --- |
| **SQS** | Simple queue, no replay | Simple async messaging, no history needed |
| **Kinesis** | AWS-proprietary Kafka-like streaming | AWS-native, moderate scale, up to 365 day retention |
| **MSK** | Fully managed real Kafka | Need Kafka ecosystem (Connect, Streams) without managing infra |
| **Self-hosted Kafka** | Kafka on EC2 | Maximum control and flexibility |

**Kinesis vs Kafka:**

|  | Kafka / MSK | Kinesis |
| --- | --- | --- |
| **Retention** | Configurable — forever | 24hrs default, max 365 days |
| **Throughput** | Very high | 1MB/sec write per shard |
| **Ecosystem** | Huge — Kafka Connect, Streams | AWS ecosystem only |
| **Ops overhead** | Higher (even with MSK) | Fully managed |

---

## Three-Question Ritual

**What problem does Kafka solve that SQS cannot?**

Message replay — consumers can reprocess historical messages from any offset. Also handles massive throughput with multiple independent consumer groups reading the same topic simultaneously.

**What breaks without consumer offsets?**

Consumers lose their position on crash/restart — either reprocess everything from the beginning (duplicates) or miss messages they never processed.

**When NOT to use Kafka?**

Single consumer, low message frequency, no need for replay or long retention. Kafka requires significant infrastructure — SQS is fully managed with zero ops overhead for simpler use cases.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Kafka (self-managed) | Apache Kafka on EC2 |
| Managed Kafka | Amazon MSK |
| AWS-native streaming | Amazon Kinesis Data Streams |
| Kinesis analytics | Amazon Kinesis Data Analytics |