# Messaging queues

## The One-Line Summary

> A message queue decouples producers from consumers, enabling asynchronous processing so services don't need to be available simultaneously and the system stays resilient when individual services fail.
> 

---

## The Problem Without Message Queues

User places an order. System synchronously calls:

1. Email service
2. SMS service
3. Warehouse service
4. Invoice service

**Problems:**

- User waits for all operations before seeing "Order confirmed" — slow experience
- Email service goes down — entire order flow fails
- Tight coupling — microservices behaving like a monolith

---

## How Message Queues Work

1. User places order
2. Order service saves order, drops message on queue — "Order #123 placed"
3. Order service immediately returns "Order confirmed" to user
4. Background consumers process the message independently:
    - Email service → sends confirmation email
    - SMS service → sends SMS
    - Warehouse service → notifies warehouse
    - Invoice service → generates invoice

Producer doesn't know or care what happens next. It drops and moves on.

**Key terminology:**

- **Producer** — service that puts messages on the queue
- **Consumer** — service that reads messages from the queue
- **Message** — the data being passed
- **Queue** — the buffer that holds messages until consumers are ready

---

## Fan-Out Pattern — SNS + SQS

With a plain queue, only one consumer gets each message. To send one message to multiple services:

```
Order Service → SNS Topic ("order placed")
                    ├── SQS Queue → Email Service
                    ├── SQS Queue → SMS Service
                    ├── SQS Queue → Warehouse Service
                    └── SQS Queue → Invoice Service
```

**SNS** — Simple Notification Service. Broadcasts one message to multiple SQS queues simultaneously.

**SQS** — Simple Queue Service. Each service has its own queue and processes at its own pace.

---

## Key Queue Features

### Message Visibility Timeout

When a consumer picks up a message, it becomes invisible to other consumers for a set period. If processing succeeds — consumer deletes the message. If consumer crashes — timeout expires, message becomes visible again for another consumer.

Prevents duplicate processing while protecting against message loss.

### Batching

Consumer reads multiple messages at once (up to 10 in SQS). More efficient, fewer network round trips.

### Message Retention

How long a message stays in the queue if unconsumed. SQS default: 4 days. Max: 14 days. After that, auto-deleted.

### Dead Letter Queue (DLQ)

After n failed processing attempts, message moves to DLQ instead of being lost or retried forever. Engineers inspect, fix the issue, and reprocess.

Without DLQ: poison pill messages (always failing) block the queue forever.

### FIFO vs Standard Queue

|  | Standard Queue | FIFO Queue |
| --- | --- | --- |
| **Ordering** | Best effort — not guaranteed | Strict first in first out |
| **Delivery** | At least once — duplicates possible | Exactly once — no duplicates |
| **Throughput** | Unlimited | 300 msg/sec (3,000 with batching) |
| **Use when** | Order doesn't matter, high throughput | Order matters, no duplicates allowed |

FIFO: Financial transactions, order processing.

Standard: Email notifications, analytics events.

### Long Polling vs Short Polling

- **Short polling** — consumer asks "any messages?" Server responds immediately even if empty. Wastes API calls.
- **Long polling** — consumer waits up to 20 seconds for a message. Fewer empty responses, cheaper. **Always prefer long polling.**

### Message Size

SQS max: **256KB**. For larger payloads — store data in S3, put S3 reference in the queue message. Consumer fetches from S3.

### Delay Queues

Messages hidden for a configurable delay before becoming visible. Use case: send follow-up email 24 hours after signup.

---

## Three-Question Ritual

**What problem does a message queue solve?**

Tight synchronous coupling between services. Decouples producers from consumers so services don't need to be available simultaneously, failures don't cascade, and users don't wait for background operations.

**What breaks without a message queue?**

One service failure cascades through the entire chain. Users wait for all background operations. Microservices behave like a monolith.

**When NOT to use a message queue?**

- When you need an immediate response (username availability check)
- Simple, fast operations where queue overhead isn't justified
- Early stage systems where operational complexity isn't warranted

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Message queue | Amazon SQS |
| Fan-out to multiple queues | Amazon SNS + SQS |
| Dead letter queue | SQS DLQ (built-in feature) |
| FIFO queue | SQS FIFO Queue |
| Large message payloads | SQS Extended Client + S3 |