# Event-Driven Architecture

## The One-Line Summary

> Event-driven architecture replaces direct service calls with events. Producers announce what happened, consumers react independently. Services are fully decoupled.
> 

---

## The Core Shift — Commands vs Events

**Command:** "PaymentService, charge this customer." Direct instruction to a specific service. Sender knows and cares about the receiver. Tightly coupled.

**Event:** "OrderPlaced." An announcement that something happened. Sender doesn't know or care who listens. Loosely coupled.

Event-driven architecture is built entirely around events, not commands.

---

## Why It Matters — Loose Coupling

With direct calls:

- Order service calls payment service, waits, calls inventory, waits, calls notification
- Payment service goes down — entire order flow fails
- Adding a new consumer requires changing the producer

With events:

- Order service publishes "OrderPlaced" and moves on
- Payment, inventory, notification all listen independently
- One consumer fails — others keep working, failed consumer processes events when it recovers
- Add a new consumer — producer never changes

---

## Two Styles

### Choreography

Services react to events independently. No central coordinator. Each service knows what events to listen to and what events to publish.

```
Order Service → "OrderPlaced"
                    ↓
            Payment Service → "PaymentCompleted"
                                    ↓
                        Inventory Service → "InventoryReserved"
```

**Advantages:** Fully decoupled, no single point of failure

**Disadvantages:** Hard to see full flow, hard to debug, complex branching logic spread across services

**Analogy:** Flash mob — everyone knows their part, no conductor. Beautiful when it works, chaos when it doesn't.

### Orchestration

A central orchestrator directs each service in sequence. Knows the full workflow.

```
Orchestrator → "ChargeCustomer" → Payment Service
             ← "PaymentCompleted"
Orchestrator → "ReserveInventory" → Inventory Service
```

**Advantages:** Easy to see full flow, easy to debug, handles complex conditional logic cleanly

**Disadvantages:** Orchestrator is tightly coupled to each service, single point of coordination

**Analogy:** Symphony orchestra — conductor directs everyone. Clear, controlled, easy to follow.

### When to use which

| Scenario | Use |
| --- | --- |
| Simple linear flow, services fully independent | Choreography |
| Complex conditional branching logic | Orchestration |
| Need full visibility of workflow in one place | Orchestration |
| Maximum decoupling, smaller system | Choreography |

---

## Two Types of Events

### Thin Event (Notification Event)

Just announces something happened. Minimal data.

```json
{ "event": "OrderPlaced", "orderId": "123" }
```

Consumer must call back to get full details. More network calls but smaller payload.

### Fat Event (Event-Carried State Transfer)

Contains all data the consumer needs. No callback required.

```json
{
  "event": "OrderPlaced",
  "orderId": "123",
  "userId": "456",
  "items": [...],
  "totalAmount": 99.99
}
```

Fewer network calls but larger payload. Risk: exposes data to consumers that don't need it.

**Most production systems prefer fat events** — avoid the callback round trip.

---

## Connection to Saga Pattern (Session 20)

- **Choreography-based Saga** — services react to each other's events, compensating transactions triggered by failure events
- **Orchestration-based Saga** — central coordinator directs each step, handles compensation centrally

---

## Three-Question Ritual

**What problem does event-driven architecture solve?**

Tight coupling between services where one failure cascades through the entire system. Events fully decouple producers from consumers — services can fail, restart, or be added without affecting each other.

**What breaks without it?**

One service failure blocks the entire chain. Users experience errors across the whole system even though only one component failed. Adding new consumers requires changing producers.

**When NOT to use it?**

When immediate response is required (synchronous request-response needed). Simple systems where debugging complexity isn't justified. When the team needs to trace a full workflow easily.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Event bus | Amazon EventBridge |
| Choreography | EventBridge + Lambda/SQS consumers |
| Orchestration | AWS Step Functions |
| Event streaming | Amazon Kinesis, Amazon MSK (Kafka) |
| Fan-out events | Amazon SNS |