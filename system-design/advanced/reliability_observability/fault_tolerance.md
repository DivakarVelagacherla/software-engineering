# Fault Tolerance

## The One-Line Summary

> Fault tolerance means the system continues operating through failures with zero user impact. High availability means recovering quickly. Fault tolerance is the harder, more expensive goal.
> 

---

## HA vs Fault Tolerance

|  | High Availability | Fault Tolerance |
| --- | --- | --- |
| **Goal** | Recover quickly from failures | Continue operating through failures |
| **User impact** | Brief downtime during failover | Zero downtime, users never notice |
| **Example** | Database replica promoted in 30 seconds | System keeps working while database is down |
| **Cost** | Lower | Higher |

---

## The Five Fault Tolerance Techniques

### 1. Redundancy

Multiple instances of every component. One fails — others absorb load instantly. No downtime window. (Covered in Session 36)

### 2. Graceful Degradation

When a component fails, system keeps working at reduced capacity rather than failing completely.

**Examples:**

- Netflix recommendation service down → show generic popular movies instead of error page
- YouTube network slow → automatically drop from 4K to 720p instead of buffering
- Search autocomplete down → show search box without suggestions instead of broken page

Users get degraded experience but system never completely fails.

### 3. Bulkhead Pattern

Named after watertight compartments in a ship — one floods, others stay dry.

**Problem:** One shared thread pool for all services. Stripe is slow — payment threads fill up waiting. User service requests also blocked even though user service is fine.

**Fix:** Separate thread pools (and connection pools) per service.

- Payment service → 50 dedicated threads
- User service → 50 dedicated threads
- Stripe being slow only affects payment threads. User service unaffected.

**With Circuit Breaker:** Bulkhead isolates the failure to one pool. Circuit breaker stops new requests entering that pool when dependency is down. Together — cascading failures prevented completely.

### 4. Timeouts

Every network call must have a timeout. If Stripe doesn't respond in 5 seconds — fail fast.

Without timeouts: threads block forever, resources drain, system collapses.

**Always set timeouts on:** database calls, external API calls, inter-service calls.

### 5. Retry with Exponential Backoff + Jitter

When a call fails, retry — but wait longer between each attempt.

```
Attempt 1 — immediate
Attempt 2 — wait 1 second
Attempt 3 — wait 2 seconds
Attempt 4 — wait 4 seconds
Attempt 5 — wait 8 seconds
```

**Why exponential, not fixed interval?** Gives the failing service breathing room to recover. Fixed interval retries hammer an already struggling service.

**Jitter:** Add a small random delay on top of backoff. Prevents thundering herd — 1000 clients retrying at the exact same millisecond would overwhelm a recovering service.

---

## Thundering Herd Problem

1000 clients all retry simultaneously — 1000 requests per second hit an already struggling service. Makes recovery impossible. Exponential backoff + jitter solves this by spreading retries out.

---

## Three-Question Ritual

**What's the difference between HA and Fault Tolerance?**

HA = recover quickly, brief downtime acceptable. Fault Tolerance = continue operating through failures, zero user impact. Fault tolerance is harder and more expensive.

**What breaks without fault tolerance in microservices?**

Cascading failures. One slow dependency consumes shared resources, blocking unrelated services. No bulkheads means one failure floods everything.

**When NOT to implement all five techniques?**

Small systems with low traffic and limited budget. A simple internal tool with 10 users doesn't need bulkheads and exponential backoff. Implement proportional to your scale and risk.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Redundancy | EC2 Auto Scaling, Multi-AZ RDS |
| Graceful degradation | CloudFront serving cached content when origin is down |
| Bulkhead (thread isolation) | Resilience4j Bulkhead inside ECS/Lambda |
| Timeouts | AWS SDK default timeouts, API Gateway timeout config |
| Retry + exponential backoff | AWS SDK built-in retry with backoff, SQS visibility timeout |
| Thundering herd prevention | SQS jitter on visibility timeout, EventBridge retry jitter |