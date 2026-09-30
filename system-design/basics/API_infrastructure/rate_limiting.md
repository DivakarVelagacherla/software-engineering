# Rate Limiting

## The One-Line Summary

> Rate limiting controls how many requests a client can make in a given time window, protecting your servers from abuse, DDoS attacks, and runaway clients.
> 

---

## The Problem Without Rate Limiting

One abusive client sends 10,000 requests per second. Server gets overloaded and crashes. Every legitimate user is affected. One bad actor takes down the entire system.

---

## What to Track

To rate limit a client you need:

1. **Who** is making the request — client ID, API key, or IP address
2. **How many requests** they've made in a given time window

**Where to store this:** Redis — fast in-memory reads/writes, shared across all gateway instances, atomic increment operations.

---

## The Three Algorithms

### 1. Fixed Window

Divide time into fixed buckets. Each client gets N requests per bucket.

```
Window 1: 12:00:00 - 12:01:00 → 100 requests allowed
Window 2: 12:01:00 - 12:02:00 → 100 requests allowed
```

**Problem — Boundary Attack:**

Client sends 100 requests at 12:00:59 (end of window 1) and 100 more at 12:01:01 (start of window 2). Both within limits — but 200 requests in 2 seconds. Server sees a burst twice the intended limit.

**Use when:** Simplicity matters, boundary attacks are acceptable risk.

### 2. Sliding Window

For each request, look back exactly N seconds and count requests in that rolling period. No fixed boundary — window moves with time.

**Advantage:** Eliminates boundary attack.

**Problem:** Must store timestamp of every individual request. At millions of requests/sec, memory is expensive. Fixed window just stores one counter.

**Use when:** Accuracy matters more than memory efficiency.

### 3. Token Bucket (most common in production)

Each client has a bucket with max capacity (e.g. 100 tokens). Tokens added at fixed rate (e.g. 10/sec). Each request costs 1 token. Empty bucket → request rejected.

**Analogy:** Prepaid phone plan. You get 10 minutes added every hour. Use them anytime. Run out — calls blocked until refill. Unused minutes accumulate to plan limit.

**Advantage:** Handles legitimate bursts gracefully. Client inactive for a while — bucket fills up, can burst. But sustained rate is enforced.

**Use when:** Need to allow legitimate bursts while enforcing sustained rate limits.

---

## Algorithm Comparison

|  | Fixed Window | Sliding Window | Token Bucket |
| --- | --- | --- | --- |
| **Complexity** | Simple | Medium | Medium |
| **Memory** | Low — one counter | High — all timestamps | Low — token count + timestamp |
| **Burst handling** | Poor — boundary attack | Good | Best — allows legitimate bursts |
| **Use when** | Simple systems | Accuracy critical | Production default |

---

## Distributed Rate Limiting

**The problem:** 5 API gateway instances, client sends 20 requests split across all 5. Each instance sees only 4 requests — under the limit. Together — 20 requests, over the limit.

**The solution:** Centralized Redis. All gateway instances read and write to the same Redis counter.

```
Gateway 1 → Redis INCR client_123 → returns 4
Gateway 2 → Redis INCR client_123 → returns 8
Gateway 3 → Redis INCR client_123 → returns 12
```

Redis `INCR` is atomic — no race conditions, accurate count regardless of which gateway handles the request.

**Full architecture:**

```
Client → Load Balancer → API Gateway instances (all sharing Redis)
                              → Load Balancer → Service instances
```

---

## Redis as Single Point of Failure

Centralized Redis needs high availability:

- **Redis Cluster** — data sharded across multiple nodes
- **Redis Sentinel** — automatic failover
- **AWS ElastiCache** — managed Redis with built-in failover

**If Redis goes down — fail open or fail closed?**

|  | Fail Open | Fail Closed |
| --- | --- | --- |
| **Behavior** | Allow all requests through | Reject all requests |
| **Risk** | Abusive clients overwhelm backend | Legitimate users blocked |
| **Use when** | User-facing services — don't punish legitimate users | Security-sensitive APIs where abuse is catastrophic |

**Most production systems fail open** with immediate alerting to on-call engineers.

---

## Three-Question Ritual

**What problem does rate limiting solve?**

Protects servers from abusive or runaway clients so legitimate users aren't impacted. One bad actor can't take down the entire system.

**What breaks without rate limiting?**

Servers get overloaded from DDoS attacks, abusive scrapers, or runaway client bugs. System crashes, all legitimate users affected.

**When NOT to use rate limiting?**

Low traffic internal systems with trusted clients where abuse isn't a concern and added complexity isn't justified.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| API-level rate limiting | API Gateway Usage Plans and throttling |
| DDoS protection | AWS WAF + AWS Shield |
| Distributed counter | ElastiCache (Redis) |
| IP-based rate limiting | AWS WAF rate-based rules |