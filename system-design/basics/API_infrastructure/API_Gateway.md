# API Gateway

## The One-Line Summary

> An API Gateway is a single entry point for all your services. It handles authentication, rate limiting, routing, logging, and more — so your individual services don't have to.
> 

---

## The Problem Without an API Gateway

With 10 microservices, a mobile app talks directly to each one. Problems:

- **Duplicated logic** — every service implements its own auth, rate limiting, logging
- **Tight coupling** — client knows every service's address and port. One service moves, client breaks.
- **Multiple round trips** — mobile app makes separate calls to user service, order service, etc.
- **No central control** — can't enforce policies across all services from one place

---

## What an API Gateway Is

A single entry point that sits in front of all your services and handles cross-cutting concerns.

**Analogy:** Reception desk in a large office building. Visitors don't walk directly into every department. Reception verifies ID, checks appointments, logs visits, directs to the right floor, limits visitors per hour. Your services are the departments.

---

## What an API Gateway Does

### 1. Request Routing

One URL for the client. Gateway routes to the right service.

```
api.myapp.com/users  → User Service
api.myapp.com/orders → Order Service
api.myapp.com/payments → Payment Service
```

### 2. Authentication & Authorization

Gateway verifies JWT token or API key before forwarding. Invalid token → rejected immediately. Service never sees it.

### 3. Rate Limiting

Tracks requests per client. Exceeds limit → returns 429. Service never gets overwhelmed.

### 4. SSL Termination

HTTPS decrypted at gateway. Plain HTTP forwarded internally. Services don't manage SSL certificates.

### 5. Request Aggregation

Mobile app needs data from user service AND order service. Gateway fans out to both, merges responses, returns one response. One network call instead of two — critical for slow mobile networks.

### 6. Caching

Gateway caches common responses. Service never called for cached data.

### 7. Load Balancing

Distributes requests across multiple instances of the same service.

### 8. Logging & Monitoring

Centralized request logging across all services from one place.

---

## API Gateway vs Load Balancer

|  | Load Balancer | API Gateway |
| --- | --- | --- |
| **Primary job** | Distribute traffic across instances of same service | Single entry point — routing, auth, rate limiting, logging |
| **Knows about services?** | No | Yes |
| **Auth** | No | Yes |
| **Rate limiting** | No | Yes |
| **Request aggregation** | No | Yes |
| **Caching** | No | Yes |

**They work together:**

```
Client → API Gateway → Load Balancer → Service instances
```

Gateway handles cross-cutting concerns and routes to right service. Load balancer distributes across instances.

---

## How It's Implemented

**Managed services (AWS API Gateway, Kong, Apigee):**

All configuration — no code. Set rate limits, auth, caching via console or config files.

```yaml
# AWS CloudFormation example
MyApiStage:
  ThrottlingRateLimit: 1000      # requests per second
  ThrottlingBurstLimit: 2000     # max concurrent
  CachingEnabled: true
```

**Code only needed for custom logic:**

- Lambda authorizer — custom auth logic
- Request transformation — custom payload mapping
- Complex rate limiting rules

**AWS options:**

| Service | Use when |
| --- | --- |
| API Gateway (REST) | Full featured — auth, rate limiting, caching, transformation |
| API Gateway (HTTP) | Simpler, cheaper, faster — basic routing and auth |
| ALB | Just routing and load balancing, no full API management |

---

## The Tradeoffs

- **Single point of failure** — gateway goes down, everything goes down. Mitigate with multiple instances across availability zones.
- **Added latency** — extra hop for every request. Negligible for most apps, critical for ultra-low latency systems.
- **Operational complexity** — one more component to manage, monitor, and scale.

---

## Three-Question Ritual

**What problem does an API Gateway solve?**

Duplicated cross-cutting concerns across all services — auth, rate limiting, logging. Centralizes them in one place so services focus purely on business logic.

**What breaks without it?**

Every service implements its own auth and rate limiting. Client tightly coupled to every service address. No central control point for policies. Multiple round trips from client for data from multiple services.

**When NOT to use it?**

Single service systems where overhead isn't justified. Ultra-low latency systems where the extra hop is unacceptable (e.g. high frequency trading internal calls).

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| API Gateway | Amazon API Gateway (REST or HTTP) |
| Custom auth | Lambda Authorizer |
| Rate limiting | API Gateway Usage Plans |
| Logging | CloudWatch Logs integration |
| Caching | API Gateway Cache |
| SSL termination | ACM Certificate on API Gateway |