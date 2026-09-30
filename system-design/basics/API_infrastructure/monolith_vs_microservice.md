# Monolith vs Microservices

## The One-Line Summary

> Start with a monolith. Split into microservices when team size, scaling needs, or failure isolation justify the complexity. Never start with microservices.
> 

---

## Monolith — Everything in One

One codebase, one deployment, one database. All components live together.

**Advantages:**

- Simple to build and run locally
- No network calls between components — just function calls
- One deployment, one log file, easy to debug
- Fast to ship on day one
- No distributed systems complexity

**When it breaks down:**

- **Deployment coupling** — 50 engineers, payments bug blocks orders team deployment
- **Scaling inefficiency** — catalog needs 10x scale, payments doesn't. Must scale entire app.
- **Technology lock-in** — ML team wants Python, core is Java. Everyone uses same stack.
- **Reliability** — notification memory leak crashes entire app including payments

---

## Microservices — Independent Services

Each component is its own service with its own codebase, deployment, and database.

**Advantages:**

- **Independent deployability** — orders team deploys without waiting for payments team
- **Independent scaling** — scale catalog 10x without touching payments
- **Failure isolation** — notification bug doesn't kill payments
- **Technology flexibility** — each service picks its own language and stack
- **Team autonomy** — each team owns their service end to end

**The complexity microservices introduce** (all covered in earlier sessions):

- **Distributed transactions (Session 20)** — Saga pattern, 2PC, idempotency
- **API Gateway (Session 26)** — single entry point for 10 services
- **Message queues (Session 22)** — async communication between services
- **Service discovery** — services need to find each other over the network
- **Distributed tracing** — trace a request across 5 services when something fails

> Every session in Week 5 and 6 exists largely because microservices introduced complexity that monoliths don't have.
> 

---

## The Stepping Stone — Modular Monolith

Same single codebase and deployment, but internally organized into clear modules with strict boundaries. Modules communicate only through defined interfaces.

```
Traditional Monolith → Modular Monolith → Microservices
(spaghetti code)       (clean boundaries)   (independent services)
```

**Advantage:** When you're ready to split, each module becomes a service. Boundary work is already done. No network complexity yet.

**Most teams should stay here longer than they think.**

---

## When to Use Which

| Scenario | Use |
| --- | --- |
| Day one, small team, moving fast | Monolith |
| Growing team, clean internal structure | Modular Monolith |
| Different components need different scale | Microservices |
| Large engineering org, team autonomy needed | Microservices |
| Failure in one component must not affect others | Microservices |

**When NOT to use microservices:**

- Small team — complexity overhead kills velocity
- Uniform scaling needs — no benefit to splitting
- Acceptable shared failure risk — isolation isn't critical
- Early stage — you don't know your domain boundaries yet, splitting too early creates wrong boundaries

**Famous principle:** Amazon, Netflix, Twitter all started as monoliths. Split only when the pain of staying together exceeded the pain of splitting.

---

## Monorepo vs Polyrepo — Separate Concept

**Monorepo —** all code in one repository. **Monolith —** all code deployed as one application. These are independent decisions.

|  | Monorepo | Polyrepo |
| --- | --- | --- |
| **Shared libraries** | Update once, all services get it automatically | Update library + update every service repo separately |
| **Code visibility** | Any engineer can search all code | Siloed per team |
| **Tooling needed** | Sophisticated build systems (Bazel, Buck) | Standard Git workflows |
| **Best for** | Large orgs with tooling investment | Most startups and mid-size companies |

**Google's Piper:** 2 billion lines of code, 86TB, one repo. Microservices architecture inside. Requires custom tooling — Piper (version control), Bazel (builds), custom code search.

**Key insight:** Monorepo + microservices is valid (Google, Meta). Polyrepo + microservices is valid (most startups). They're orthogonal choices.

---

## Three-Question Ritual

**What problem do microservices solve?**

Independent scaling of components, failure isolation so one bug doesn't crash everything, and team autonomy so large engineering orgs can deploy independently without coordination.

**What breaks when you adopt microservices too early?**

Operational complexity explodes before you have scale to justify it — distributed transactions, service discovery, API gateways, distributed tracing. Premature optimization that kills velocity.

**When NOT to use microservices?**

Small team, uniform scaling needs across all components, when failure isolation isn't critical, and when you don't yet know your domain boundaries well enough to split correctly.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Monolith deployment | EC2, Elastic Beanstalk |
| Microservices containers | Amazon ECS, Amazon EKS |
| Serverless microservices | AWS Lambda |
| Service discovery | AWS Cloud Map, App Mesh |
| Distributed tracing | AWS X-Ray |