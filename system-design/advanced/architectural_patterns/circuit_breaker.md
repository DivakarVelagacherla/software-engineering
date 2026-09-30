# Circuit Breaker

## The One-Line Summary

> A circuit breaker detects when a downstream service is failing and stops sending requests to it, preventing cascading failures and giving the dependency time to recover.
> 

---

## The Problem — Cascading Failures

Payment service calls Stripe. Stripe has an outage. Every request times out after 30 seconds. 1000 users trying to pay — 1000 threads blocked, each waiting 30 seconds. Memory exhausted, connection pool drained, payment service crashes. Services depending on payment service crash too.

**One external dependency's outage cascades through the entire system.**

Retrying makes it worse — you hammer an already struggling service, making recovery harder.

---

## The Circuit Breaker Pattern

Named after electrical circuit breakers — trips when too much current flows, cuts power before wires overheat. Reset when problem is fixed.

### Three States

**1. Closed (normal operation)**

All requests flow through to the downstream service. Circuit breaker monitors failure rate. Below threshold — stays closed.

**2. Open (failure detected)**

Failure rate exceeds threshold (e.g. 50% of requests failing). Circuit opens. All requests immediately return an error — no waiting for timeout, no thread blocking. Downstream gets breathing room to recover.

**3. Half-Open (testing recovery)**

After configured wait (e.g. 60 seconds), allows one test request through:

- Succeeds → circuit closes, normal operation resumes
- Fails → circuit opens again, waits another 60 seconds

---

## Handling Users When Circuit Is Open

Opening the circuit protects your service — but users still need a response:

| Strategy | How |
| --- | --- |
| **Fail fast** | Immediate clear error: "Payment temporarily unavailable, try again shortly" |
| **Fallback** | "Order placed, payment processed shortly" — queue for when Stripe recovers |
| **Retry queue** | Push to queue, process when circuit closes, notify user on success |
| **Alternative provider** | Route to backup payment processor (PayPal, Braintree) |

> In interviews: always follow "I'd use a circuit breaker" with "and here's what happens to the user experience when it opens." That shows maturity.
> 

---

## Implementation — It's a Library

Not a separate service. Embedded in your application code.

**Java — Resilience4j:**

```java
CircuitBreaker cb = CircuitBreaker.ofDefaults("stripe");
Supplier<PaymentResponse> call = CircuitBreaker
    .decorateSupplier(cb, () -> stripeClient.charge(request));
try {
    return call.get();
} catch (CallNotPermittedException e) {
    return fallbackPayment(request); // circuit open
}
```

**Configuration:**

```yaml
failure-rate-threshold: 50        # open at 50% failure rate
wait-duration-in-open-state: 60s  # wait before half-open
sliding-window-size: 10           # track last 10 requests
permitted-calls-in-half-open: 1   # 1 test request in half-open
```

**When to use library vs service mesh:**

- **Library (Resilience4j)** — you control the service code. Most common. Lambda, ECS — just add the dependency.
- **AWS App Mesh** — infrastructure layer handles circuit breaking for all services centrally. No code changes. For complex microservices architectures.

---

## Three-Question Ritual

**What problem does a circuit breaker solve?**

Prevents cascading failures. When a downstream dependency fails, the circuit opens and stops your service from hammering it — protecting your service and giving the downstream time to recover.

**What breaks without it?**

One external dependency's outage causes cascading failures. Threads pile up waiting for timeouts, memory exhausts, your service crashes, upstream services crash too.

**When NOT to use it?**

Internal calls within a reliable system where cascading failure isn't a risk. When the operation is so critical that failing fast is worse than waiting.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Library-based circuit breaker | Resilience4j inside ECS/Lambda |
| Infrastructure-level circuit breaking | AWS App Mesh |
| Health-based routing | API Gateway with Lambda health checks |
| Resilience monitoring | AWS Resilience Hub |