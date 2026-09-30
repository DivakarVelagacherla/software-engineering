# Saga Pattern

## The One-Line Summary

> Saga replaces one distributed ACID transaction with a sequence of smaller local transactions. Each service commits locally and publishes an event. Failures trigger compensating transactions to undo previous steps.
> 

---

## The Problem

An order flow touches 3 services with separate databases — order, payment, inventory. You can't wrap them in one ACID transaction. 2PC locks resources across all services — one failure freezes everything.

---

## How Saga Works

**Happy path:**

```
Order Service → commits order → publishes "OrderCreated"
Payment Service → charges customer → publishes "PaymentCompleted"
Inventory Service → reserves stock → publishes "InventoryReserved"
```

**Failure path — payment fails:**

```
Order Service → commits order → "OrderCreated"
Payment Service → fails → publishes "PaymentFailed"
Order Service → receives "PaymentFailed" → runs compensating transaction → cancels order
```

**Compensating transaction:** The undo action for each step. If payment fails after inventory was reserved — inventory service releases the stock.

---

## Two Styles

### Choreography

Services react to each other's events independently. No central coordinator.

- Loosely coupled
- Hard to trace full flow
- Good for simple linear flows

### Orchestration

Central coordinator directs each step and handles compensation.

- Easy to see full flow
- Single point of coordination
- Good for complex conditional flows

---

## Saga vs 2PC

|  | 2PC | Saga |
| --- | --- | --- |
| **Locking** | Locks resources across all services | No locking — each service commits locally |
| **Failure** | Coordinator rolls back everything | Compensating transactions undo each step |
| **Availability** | Low — one failure freezes system | High — services work independently |
| **Consistency** | Strong | Eventual |

---

## Three-Question Ritual

**What problem does Saga solve?**

Distributed transactions across multiple services with separate databases. Instead of one ACID transaction with locking, each service commits locally and publishes events. Failures trigger compensating transactions.

**What breaks without Saga?**

2PC locks resources across all services. One service failure freezes the entire system waiting for the coordinator. At scale this is catastrophic.

**When NOT to use Saga?**

When all data lives in one database — one ACID transaction handles it. Saga complexity isn't needed.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Orchestration-based Saga | AWS Step Functions |
| Choreography-based Saga | Amazon EventBridge + SQS |
| Compensating transaction trigger | Lambda reacting to failure events |