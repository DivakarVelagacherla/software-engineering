# Service Discovery

## The One-Line Summary

> Service discovery lets services find each other by name instead of hardcoded IP addresses. In AWS, it's largely abstracted away by ALB, API Gateway, and Cloud Map.
> 

---

## The Problem

Microservice A needs to call Microservice B synchronously. B runs on 3 ECS instances with dynamic IPs. Auto-scaling adds a 4th instance. One instance crashes.

If A has B's IP hardcoded:

- Crashed instance IP is now invalid — calls fail
- New instance IP is unknown — never gets traffic
- Every deployment requires manually updating IPs across all services

**Service discovery:** Services register themselves. Others look them up by name. IPs can change freely.

> Note: Event-driven architectures (SQS, Kinesis) don't need service discovery — services never call each other directly. Service discovery is for synchronous service-to-service calls.
> 

---

## Two Approaches

### Client-Side Discovery

Calling service queries a service registry directly, picks an instance, calls it.

```
Order Service → Registry: "Where is Payment Service?"
Registry → Order Service: "10.0.0.6, 10.0.0.7, 10.0.0.8"
Order Service → picks one → calls Payment Service
```

- Client handles load balancing logic
- More control, more complexity in every client

### Server-Side Discovery

Calling service calls a load balancer or router. Router queries registry and forwards.

```
Order Service → Load Balancer: "Call Payment Service"
Load Balancer → Registry: "Where is Payment Service?"
Load Balancer → forwards to healthy instance
```

- Client knows nothing about instances
- Simpler clients, centralized routing logic
- **AWS ALB is server-side discovery**

---

## Service Discovery in AWS — Already Abstracted

In AWS you're using service discovery without knowing it:

| AWS Service | How it handles discovery |
| --- | --- |
| **ECS + ALB** | Tasks register/deregister automatically. Call stable DNS name, ALB routes to healthy instance. |
| **API Gateway + Lambda** | Lambda ARN is stable. No IP addresses. AWS handles routing. |
| **AWS Cloud Map** | Explicit service registry. Services register, others look up by name. Used for more control. |
| **EKS/Kubernetes** | Built-in DNS. Every service gets stable name like `payment-service.default.svc.cluster.local`. |
| **Route 53** | DNS-based service discovery. Health checks remove unhealthy endpoints automatically. |

**Self-managed tools (non-AWS):**

- **Consul** — HashiCorp, popular for hybrid/multi-cloud
- **Eureka** — Netflix open source, Java/Spring ecosystem
- **etcd** — Kubernetes uses this under the hood

---

## For Interviews

When designing microservices with synchronous calls:

- Mention services need a way to find each other
- In AWS — ALB + ECS handles it automatically
- In Kubernetes — built-in DNS
- Only need explicit tools (Consul, Eureka) when self-managing outside cloud

---

## Three-Question Ritual

**What problem does service discovery solve?**

Services can't have hardcoded IP addresses — instances come and go with auto-scaling, deployments, and failures. Service discovery provides stable name-based lookup so services find each other regardless of IP changes.

**What breaks without it?**

Hardcoded IPs become stale instantly. Service calls fail when instances crash or scale. Every deployment requires manually updating IPs across all services — impossible at scale.

**When NOT needed?**

Event-driven architectures where services communicate through queues — no direct calls. Managed AWS services where ALB, API Gateway, and Lambda ARN handle routing automatically.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Server-side discovery | Application Load Balancer (ALB) |
| Explicit service registry | AWS Cloud Map |
| DNS-based discovery | Route 53 + health checks |
| Kubernetes service discovery | EKS built-in DNS |