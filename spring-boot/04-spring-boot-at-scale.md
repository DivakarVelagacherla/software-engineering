# Part IV — Spring Boot at Scale

## Chapter 14: Microservices Building Blocks

### Service-to-service communication

Choosing how services talk to each other depends on the interaction's shape:

- **Synchronous, simple calls** — `RestTemplate` (now in maintenance mode within the Spring
  ecosystem, but still widely seen) makes a request and blocks for the response, a
  straightforward two-way call.
- **Synchronous, cleaner client code** — **Feign Client** provides a declarative REST client:
  you declare an interface describing the remote API, and Feign generates the implementation,
  reducing boilerplate versus hand-writing `RestTemplate` calls.
- **Non-blocking, reactive** — **`WebClient`**, covered fully in Chapter 15, is the modern
  non-blocking alternative to `RestTemplate` for synchronous-style calls that shouldn't block a
  thread while waiting.
- **Asynchronous, decoupled in time** — **message brokers** (RabbitMQ, Kafka) let a service
  publish a message without needing the consumer to be available or responsive right now; the
  consumer processes it whenever it's ready. This decouples the caller from needing an immediate
  response at all, which is exactly right for workflows like order processing, where the
  user-facing request should return quickly and the actual processing can happen in the
  background.

### Spring Cloud, as an ecosystem

**Spring Cloud** is the Spring sub-ecosystem specifically addressing microservices concerns that
don't arise in a single-application context: coordinating many independently-deployed services,
distributing load across them, and managing configuration and secrets consistently across all of
them. A few of its pieces worth knowing individually:

- **Spring Cloud Config** — centralizes externalized configuration for many services behind a
  Config Server, so every service pulls consistent, centrally-managed settings rather than each
  managing its own configuration independently; sensitive values can be encrypted centrally
  rather than duplicated per-service.
- **Spring Cloud Gateway** — an API Gateway implementation, centralizing routing, security
  (via Spring Security integration), and monitoring (via Actuator) behind one front door for a
  microservices architecture, rather than every client needing to know about every individual
  service's location.
- **Spring Cloud Function** — lets you write business logic as plain Java functions that can be
  deployed as serverless functions on a cloud platform, abstracting away server management
  entirely — Spring's on-ramp into the serverless/FaaS world.

### Resilience against unreliable dependencies

Any service that calls other services over a network has to plan for those calls failing,
timing out, or being rate-limited. The standard toolkit, typically implemented via a library
like **Resilience4j**: a **circuit breaker** stops calling a dependency that's currently failing
(preventing a struggling downstream service from also dragging down everything that depends on
it — cascading failure), **retry with exponential backoff** recovers from transient, short-lived
failures without hammering a struggling service, **rate limiting** respects a dependency's own
capacity limits, **timeouts** prevent waiting indefinitely for something that may never respond,
and **caching** reduces call volume in the first place. Combined with real monitoring and
logging (Chapter 13) to actually detect when something's degraded, this is the standard shape of
"how you make a service that depends on other unreliable services actually reliable."

### Securing a microservices architecture

Security has to be applied *per-service*, not just at one network edge, since internal
service-to-service traffic is itself an attack surface. The standard pattern: a centralized
authentication service issues tokens (typically **JWT**) on login; every individual service
independently validates incoming tokens rather than trusting an upstream service's word for it;
all inter-service traffic uses SSL/TLS; and an API Gateway centralizes the security-adjacent
parts of request handling that would otherwise be duplicated across every service. We cover
authentication and authorization mechanics in full in Chapter 16.

---

## Chapter 15: Caching, Async, and Reactive Programming

### The Spring Cache abstraction

Spring's caching abstraction sits in front of expensive operations — most commonly database
reads — and remembers their results, so a repeated call with the same input returns the cached
result instead of redoing the work. Enabling it: add `spring-boot-starter-cache`, put
`@EnableCaching` on a configuration class, and annotate methods whose results should be cached
with `@Cacheable`; `@CacheEvict` and `@CachePut` manage invalidation and refresh explicitly. The
default provider is an in-memory `ConcurrentHashMap`-based cache — fine for a single instance,
but genuinely limited the moment an application runs as more than one instance: each instance
gets its own separate in-memory cache, meaning different instances can silently disagree about
cached data, and everything is lost on restart. **Redis** or **Hazelcast**, as *distributed*
caches shared across all instances, are the standard fix once an application scales beyond a
single instance.

**Cache eviction** and **cache expiration** are related but distinct: eviction removes entries to
free up space, under a policy like least-recently-used (a direct callback to the LRU-cache
material in the Core Java book's advanced collections chapter); expiration removes entries
because they've exceeded a time-to-live, for data freshness, independent of any space pressure.
The right invalidation strategy for frequently-changing data combines both: **event-driven
invalidation** (a data-change event triggers immediate cache invalidation for the affected entry,
so you never serve data known to be stale) as the primary mechanism, with a **TTL** as a backstop
for anything that changes without a corresponding event being fired.

### `@Async` and the proxy caveat, restated for real code

`@Async` runs a method on a background thread rather than blocking its caller, enabled via
`@EnableAsync` on a configuration class; the method can return `void` or a `Future`/
`CompletableFuture` for tracking completion or results. Because `@Async` is implemented through
the exact same AOP proxy mechanism covered in Chapter 4, the same self-invocation gotcha applies
here specifically and is worth restating in this practical context: calling an `@Async` method
from another method *in the same class* bypasses the proxy and runs synchronously, silently, with
no error. This single fact explains a large fraction of "why isn't my method actually running
asynchronously" bugs in real Spring Boot codebases.

### Reactive programming with WebFlux

**Spring WebFlux** is the non-blocking, reactive counterpart to the traditional (blocking,
thread-per-request) Spring MVC stack, built on Project Reactor's `Mono` (a single asynchronous
value or empty result) and `Flux` (an asynchronous stream of zero or more values). Building a
genuinely non-blocking API means using `WebClient` (not `RestTemplate`) for any outbound calls
and reactive repositories (`ReactiveCrudRepository`, not the standard blocking
`JpaRepository`) for data access — a controller returning `Mono`/`Flux` on top of a *blocking*
repository underneath doesn't actually get you WebFlux's scalability benefit, since the blocking
call still ties up a thread regardless of what the controller layer looks like on the surface.

**Backpressure** is the single most important reactive-programming concept beyond "it's
asynchronous," and it's worth understanding precisely rather than waving at it: the Reactive
Streams specification lets a *subscriber* tell the *publisher* how many items it's currently able
to handle, rather than the publisher simply pushing data as fast as it can produce it. This
flow-control negotiation between producer and consumer is exactly what prevents a fast data
source from overwhelming a slower consumer's memory or processing capacity — without it, a
sufficiently fast publisher and slow subscriber would eventually exhaust memory buffering data the
subscriber can't keep up with. This is the mechanism that makes WebFlux genuinely suitable for
high-concurrency, high-throughput scenarios (event-driven microservices, streaming data) without
requiring you to manually reason about buffering and flow control yourself.

---

## Chapter 16: Security

### Authentication versus authorization

These are two genuinely distinct concerns, commonly conflated in casual conversation but worth
keeping precisely separate: **authentication** is verifying *who* someone is (checking a
password, validating a token); **authorization** is deciding *what that verified identity is
allowed to do* (which endpoints, which data, which actions). A request can be correctly
authenticated and still be denied — that denial is authorization, not a failure of
authentication.

### Setting up Spring Security

The standard shape of securing a Spring Boot application: add the Spring Security starter
dependency; define a security configuration specifying which endpoints require authentication and
what login/logout flow to use; implement `UserDetailsService` to load user identity information
(commonly from a database); use a strong password encoder (`BCryptPasswordEncoder` is the
standard default) for any stored credentials; and use `@PreAuthorize` (or the broader
`@Secured`/`@PostAuthorize` family) for **method-level security** — fine-grained,
role/permission-based access control applied directly on service-layer methods, complementing
(not replacing) URL-level security configuration. Method security requires enabling it explicitly
on a configuration class (`@EnableMethodSecurity` in current Spring Security; older code may still
show `@EnableGlobalMethodSecurity`).

### Token-based authentication with JWT

For stateless APIs — and especially for microservices, where checking a shared session store on
every request across every service would be both slow and a scalability bottleneck — **JWT
(JSON Web Token)**-based authentication is the standard pattern: on login, the application issues
a token encoding the user's identity and permissions; every subsequent request carries that
token, and each service independently validates it (checking signature and claims) without
needing to re-query a central identity store per request. This is faster and scales better
than session-store lookups, at the cost of needing a real strategy for token revocation, since a
validly-signed JWT remains valid until it expires, regardless of what happens to the underlying
user account in the meantime.

### Session management in distributed systems

A traditional in-memory HTTP session is pinned to whichever server instance created it — which
breaks the moment a load balancer can route a user's subsequent requests to a *different*
instance across their session's lifetime. **Spring Session**, backed by a shared external store
(Redis, Hazelcast, or a JDBC-backed store), solves this by making session state available to
*every* instance rather than tied to one — any instance can serve any user's request and find
their session data in the shared store. This is required infrastructure, not an optional
optimization, as soon as an application runs behind a load balancer with more than one instance.

---

## Chapter 17: Scaling, Resilience, and Deployment

### Scaling strategies

The standard toolkit for handling increased load, roughly in order of how quickly each can be
applied: **horizontal scaling** (add more application instances, distribute traffic across them
via a **load balancer**) is usually the fastest lever to pull under acute pressure; decomposing a
monolith into **independently-scalable microservices** lets you scale only the specific parts
under load rather than the whole application uniformly; **caching** reduces database load for
frequently-accessed data (Chapter 15); **query and index optimization** addresses the database
itself directly; and **cloud auto-scaling** can adjust instance count automatically based on
real-time demand rather than requiring manual intervention.

### The CAP theorem — genuinely important conceptual grounding for distributed systems work

The **CAP theorem** states that a distributed system can only fully guarantee two of three
properties *simultaneously*: **Consistency** (every node sees the same data at the same time),
**Availability** (every request receives a response, success or failure, rather than hanging
indefinitely), and **Partition Tolerance** (the system keeps operating despite network partitions
— nodes becoming unable to communicate with each other). The practically important framing: since
network partitions are a real, unavoidable possibility in any genuinely distributed system,
Partition Tolerance isn't really an optional design choice you get to skip — you have to tolerate
partitions occurring. That leaves the *actual* real-world tradeoff as being between Consistency
and Availability specifically, when a partition happens: do you refuse to answer some requests
until the network heals (favoring Consistency), or do you answer with potentially-stale data on
both sides of the partition (favoring Availability)?

A concrete, defensible worked example: an e-commerce site under extreme load (a flash sale) might
deliberately choose Availability and Partition Tolerance over strict Consistency — accepting the
possibility of briefly showing stock that just sold out on a different instance — because keeping
the site responsive and online for everyone matters more than momentary perfect consistency, and
the inconsistency window is both brief and low-stakes relative to the alternative of the site
going down entirely.

### Transaction propagation across service boundaries

`@Transactional`'s basic behavior — an all-or-nothing boundary around a method — gets genuinely
more nuanced once a transaction spans multiple service calls, and the choice of **propagation
level** has real consequences worth understanding precisely, not just picking the default. Two
worth knowing specifically: **`REQUIRED`** (the default) joins an existing transaction if one is
already in progress, keeping everything atomic together — but this widens the transaction's scope
across everything nested inside it, which increases lock-contention and deadlock risk the more
that gets nested inside a single transactional scope. **`REQUIRES_NEW`** suspends any existing
transaction and starts an independent one, giving true isolation between the two units of work —
at the cost of higher resource consumption (more concurrent open transactions) and materially more
complicated rollback reasoning, since a failure in the inner `REQUIRES_NEW` transaction doesn't
automatically roll back the outer one it was suspended from.

For workflows that genuinely span multiple independently-deployed *services* (not just multiple
method calls within one service) — a distributed transaction spanning a true microservices
architecture — the modern standard answer is generally the **Saga pattern** rather than a single
distributed ACID transaction: a sequence of local transactions, each service committing its own
piece independently, with each step paired with a **compensating action** that can undo it if a
later step in the sequence fails. This trades strict atomicity (which becomes prohibitively
expensive and fragile across genuinely independent services) for a coordinated, eventually-
consistent sequence with an explicit undo path.

### Deployment models and zero-downtime strategy

Recall from Chapter 9 that Boot supports standalone JAR (the native default), WAR-to-external-
server, and Docker containerization as deployment options, each with different tradeoffs. For
deploying *updates* without user-visible downtime, the standard pattern is **blue-green
deployment**: maintain two identical production environments (call them blue and green); the new
version deploys to the currently-inactive one (green, if blue is live) and is fully tested there
while blue continues serving all real traffic; once green is verified, traffic is cut over to it
entirely; blue then becomes the immediate rollback target if anything goes wrong post-cutover,
since it's still fully intact and was, until moments ago, the known-good running version.

Containerized deployments typically build a Docker image (via a hand-written Dockerfile or Boot's
own `spring-boot:build-image`, per Chapter 9), push it to a registry (Docker Hub, or a private
registry like AWS ECR/Azure Container Registry for organization-controlled access), and manage
running containers through an orchestrator (Docker Compose for simple cases, Kubernetes for real
production scale). Best practices for the images themselves: keep base images small (Alpine-based
where possible, multi-stage builds so build-time tooling doesn't end up shipped in the final
runtime image), and externalize all environment-specific configuration via environment variables
rather than baking it into the image — keeping the same image genuinely deployable, unmodified,
across development, staging, and production.

---

## Chapter 18: Production War Stories and Debugging

This closing chapter mirrors the closing chapter of *Internals of Core Java* deliberately: less
"how the framework works," more "what actually happens when it's running in front of real users,
and what you do about it."

### Verifying a deployment actually worked

Immediately after any deployment, a layered verification approach catches problems before users
do: automated health checks and integration tests run first; monitoring tools watch
performance and error-rate metrics against predefined thresholds to catch anomalies quickly; and
for genuinely critical features, manual or user-acceptance testing supplements the automated
layers. The goal is treating "did the deploy script exit successfully" as necessary but not at
all sufficient evidence that the deployment actually worked correctly for real traffic.

### Rollback, planned in advance

When a critical bug surfaces shortly after a deployment, the ability to roll back quickly depends
entirely on having planned for it *before* the incident, not improvising during one: version
control and deployment tooling with genuine rollback support let you stop the current deployment,
reactivate the last known-good configuration, and restart services — and continuous monitoring
plus automated alerting is what actually catches the need for a rollback fast enough for it to
matter. Blue-green deployment (Chapter 17) is, among its other benefits, itself a rollback
strategy — the previous environment stays fully intact and immediately available as a fallback.

### Artifact storage and its own failure mode

Build artifacts (JARs, Docker images) are typically stored in a centralized repository manager —
Artifactory or Nexus for JARs, a container registry for Docker images — enabling versioned,
traceable deployments across every environment and team. This centralization is itself a
dependency worth planning around: if the primary artifact store becomes unreachable, deployments
can grind to a halt unless there's a secondary, kept-in-sync repository as a fallback, or critical
artifacts are cached locally/in a distributed cache for exactly this kind of outage.

### A concrete misconfiguration story, worth internalizing as a category of bug

A real, recurring category of production incident: a single misconfigured properties value — a
database connection timeout set too low, in one specific real case — causing frequent connection
drops specifically under high load, while working fine under light testing load where the
timeout's effects rarely surfaced. Found through error logs and monitoring, fixed by correcting
the property value and redeploying. The lesson worth generalizing: this class of bug isn't a code
defect at all — the code was correct — it's a configuration-value defect, and it's precisely the
kind of issue that code review structurally cannot catch (the code looks fine because it *is*
fine) and that only production-realistic load, combined with real monitoring, actually surfaces.
It's a direct, practical argument for taking configuration values as seriously as code during
review, and for load-testing configuration changes specifically, not just code changes.

### Debugging tools, connected back to Core Java

Nearly all the deep debugging tools from *Internals of Core Java*'s closing chapter apply
completely unchanged here, because a Spring Boot application is still, underneath everything
covered in this book, a running JVM process: heap dumps and `OutOfMemoryError` investigation
(Core Java Chapter 3), thread dumps for diagnosing deadlocks and stuck threads (Core Java Chapter
11), and the `equals()`/`hashCode()` contract's production performance implications (Core Java
Chapter 8) are exactly as relevant in a Spring Boot service as in any other Java process — Spring
doesn't replace the JVM's own operational characteristics, it runs on top of them. What this book
adds on top is the framework- and architecture-specific layer: Actuator's health and metrics
endpoints (Chapter 13), distributed tracing across service boundaries (Chapter 13), and the
deployment- and scaling-specific failure modes covered in this chapter and the last.

---

## Closing note

Spring's story, end to end, is really one continuous argument: complexity that has to exist
somewhere is better carried by a framework than scattered through business logic. Plain Java
without a framework forces every class to manage its own wiring; Spring's IoC container carries
that instead. Spring without Boot forces every project to manually reassemble the same
configuration, dependency alignment, and server setup; Spring Boot carries that instead. And
Spring Boot at scale — microservices, distributed transactions, observability across service
boundaries — pushes the same question one level further outward, into architecture and
operations, which is exactly where this book's final chapters end up. The throughline from
*Internals of Core Java* is unbroken: understand what a system is actually doing underneath its
convenient surface, and both the convenience and its limits stop being mysterious.
