# Part III — Building With Spring Boot

## Chapter 10: Building REST APIs

### The core annotation set

Five annotations cover the large majority of everyday REST API code in Spring Boot:
**`@RestController`** (a `@Controller` + `@ResponseBody` combination — every method's return
value is serialized directly into the response body as JSON/XML rather than resolved as a view
name), **`@RequestMapping`** (maps a URL path and HTTP method to a handler method; the
general-purpose form), and its HTTP-verb-specific shorthands **`@GetMapping`**,
**`@PostMapping`**, and their siblings **`@PutMapping`**/**`@DeleteMapping`**. `@PathVariable`
extracts dynamic segments from the URL path into method parameters; `@RequestParam` binds
query-string or form parameters; `@RequestBody` deserializes the raw HTTP request body (JSON/XML)
into a Java object; `@ResponseBody` (implied automatically by `@RestController`) serializes a
returned Java object directly into the response body.

`@Controller` versus `@RestController`: `@Controller` classically returns a *view name*, for
server-rendered pages; `@RestController` assumes every method returns *data*, making it the
default for building REST APIs specifically. `@RequestMapping` versus `@GetMapping`:
`@RequestMapping` is general-purpose and requires the HTTP method specified explicitly;
`@GetMapping` is a more concise, self-documenting shorthand specifically for GET requests.

### `ResponseEntity` and status-code discipline

`ResponseEntity<T>` gives full, explicit control over an HTTP response — status code, headers,
and body together (`new ResponseEntity<>(payload, HttpStatus.OK)`). Returning a plain object
instead is simpler, and Boot automatically wraps it in a `200 OK` response — the right default
for the common case, reserving `ResponseEntity` for when you genuinely need to customize the
response beyond that default (a specific non-200 status, custom headers). A concrete, commonly
misapplied example worth internalizing: `DELETE` endpoints should typically return `200 OK` (with
a response body), `204 No Content` (successful deletion, no body), or `404 Not Found` (nothing
existed to delete) — not a blanket `200` regardless of outcome.

### A real production bug pattern worth knowing: PUT versus POST

Using `POST` for an operation that should logically be **idempotent** (repeating the exact same
request produces the same end state, with no side effect from repetition) is a genuinely common,
concrete source of production bugs — a client retry, or a double-click on a submit button,
creates a duplicate record, because `POST` carries no idempotency guarantee. `PUT` is
idempotent by contract: the same `PUT` request repeated has the same effect as sending it once.
Choosing the correct verb isn't pedantry — it's the difference between a request that's safe to
retry and one that isn't, which matters enormously the moment any part of your system (a client,
a load balancer, a retry policy) might resend a request that already succeeded.

### Versioning and best practices

REST API versioning strategies, so an API can evolve without breaking existing clients: **URL
path** (`/api/v1/resource` — the most explicit and common), **query parameter**
(`?version=1`), **custom header**, or **media-type/content-negotiation**
(`Accept: application/vnd.example.v1+json`). Broader REST best practices worth treating as a
checklist: use the correct HTTP verb for the operation's actual semantics, keep requests
stateless, name resources clearly and consistently, handle errors with consistent status codes
and messages, secure endpoints with HTTPS and real input validation, and paginate large result
sets rather than returning unbounded collections.

### Validation

Spring Boot integrates the Jakarta Bean Validation API (Hibernate Validator underneath) directly
into the request-binding flow: annotate model fields with constraints (`@NotNull`, `@Size`,
`@Email`, and others), add `@Valid` on the corresponding controller method parameter, and Boot
automatically validates incoming data before your handler method body runs, short-circuiting into
an error response on failure. For validation logic that spans multiple fields — "field A must be
consistent with field B" — the pattern is a **custom class-level constraint**: define a new
annotation plus a `ConstraintValidator` implementation encapsulating the cross-field logic,
applied at the class/DTO level rather than per-field, keeping that logic encapsulated and
reusable everywhere the DTO is used.

Documenting the resulting API is commonly handled with **Swagger**, an open-source framework that
generates interactive, always-current documentation directly from the API's own definitions,
letting consumers explore and test endpoints from the documentation itself rather than a
separately-maintained (and inevitably stale) document.

---

## Chapter 11: Data Access and Multiple Databases

### Repositories, revisited at the Boot level

Chapter 5 introduced `CrudRepository` and `JpaRepository` as core-Spring-Data abstractions; Boot
adds auto-configuration on top, per Chapter 7 — if a JDBC driver and Spring Data JPA are on the
classpath, Boot wires up a `DataSource`, `EntityManagerFactory`, and `TransactionManager`
automatically, with zero manual configuration for the common single-database case.

### Multiple database connections

Connecting to more than one database in a single application steps outside what auto-
configuration handles for you by default — it requires explicit configuration: separate
`@Configuration` classes, each defining its own `DataSource`, `EntityManagerFactory`, and
`TransactionManager` beans for its specific database. `@Qualifier` at each injection point
distinguishes which database's beans a given repository or service should use;
`@Primary` on one `DataSource` marks it the default for any injection point that doesn't specify
a qualifier.

### Schema migrations

**Flyway** and **Liquibase** are the standard tools for managing database schema changes as
version-controlled, incremental scripts, applied automatically (typically at application
startup) in a defined, guaranteed order. This keeps schema state consistent and reproducible
across every environment an application runs in, replacing manual, ad-hoc DDL changes with a
tracked, repeatable process — genuinely important the moment more than one person or environment
needs to stay in sync on a database's structure.

**Zero-downtime schema migration** for a live production system follows a specific, disciplined
pattern worth naming explicitly — sometimes called "expand-migrate-contract": introduce the new
schema *alongside* the old one; have the application write to both simultaneously; backfill
existing data into the new schema; verify correctness; cut reads over to the new schema only once
verified; and only decommission the old schema after everything is confirmed working end to end.
Each step is independently reversible, which is the entire point — a single big-bang schema swap
gives you no safe rollback point if something's wrong.

### Pagination

Spring Data JPA's `Pageable`/`PageRequest` machinery handles paginated queries without hand-
written offset/limit logic: repository methods accept a `Pageable` parameter, the calling code
constructs a `PageRequest` (page number and size), and the result comes back as a `Page` object
carrying both the requested slice of data and useful metadata (total elements, total pages) —
letting an application efficiently work with large datasets a bounded slice at a time.

---

## Chapter 12: Testing Spring Boot Applications

### The test pyramid, applied to Spring Boot

A sensible testing strategy for a Boot application layers several distinct kinds of test, each
progressively more expensive and more end-to-end than the last:

1. **Unit tests** — isolated checks of individual components, using JUnit for assertions and
   Mockito to fake out dependencies, with no Spring context involved at all.
2. **Slice tests** — load *part* of the Spring context, scoped to one architectural layer:
   `@WebMvcTest` loads only the web layer (fast, focused controller testing); `@DataJpaTest`
   loads only the persistence layer.
3. **Integration tests** — `@SpringBootTest` loads the *entire* application context, verifying
   that every component works together correctly, in an environment close to a real running
   application.
4. **End-to-end tests** — automated tests simulating real user-facing flows against a fully
   deployed (or deploy-like) instance.
5. **Load/stress tests** — evaluate behavior specifically under heavy concurrent load, distinct
   from correctness testing.

Each layer trades speed for realism: unit tests are fast and narrow; `@SpringBootTest` is slow
but comprehensive. A healthy test suite leans heavily on the fast, narrow layers, using the
slower, broader ones sparingly, for what only they can actually verify.

### Mockito annotations, precisely

- **`@Mock`** — a plain Mockito annotation, creating a fully faked object with no real method
  bodies executing, entirely outside any Spring context. Used to isolate a unit under test from
  its dependencies in pure unit tests.
- **`@Spy`** — wraps a *real* instance; un-stubbed methods run their actual real code, and only
  explicitly-stubbed methods are overridden. Used for partial mocking, where you want most of an
  object's real behavior but need to control one specific method's result.
- **`@InjectMocks`** — takes a set of `@Mock`-created fakes and wires them into the actual
  class-under-test instance being constructed for the test, mirroring what DI would do in
  production, but entirely within the test.
- **`@MockBean`** — the Spring Boot-specific counterpart to `@Mock`: it creates a mock and
  injects it *into the running Spring application context*, replacing whatever real bean was
  there. Used in integration tests (`@SpringBootTest`) where you want the full context loaded,
  but need one specific bean (typically an external dependency — a third-party API client, a
  repository) replaced with a controllable fake, so the test doesn't depend on that external
  system actually being available or behaving deterministically.

### `@WebMvcTest` for controller unit tests

`@WebMvcTest` loads only the web layer, letting you test a controller in isolation: autowire
`MockMvc` to simulate HTTP requests and assert on responses without starting a real HTTP server,
and use Mockito (typically `@MockBean`) to fake out the service-layer dependencies the controller
calls — testing the controller's routing and request/response handling logic specifically,
without any of the layers beneath it needing to actually work.

Mocking whole external microservices during testing typically uses tools like WireMock (or
Mockito for simpler cases) to stand in fake HTTP responses for calls that would otherwise go to a
real, separately-deployed service — letting tests run fast and deterministically without that
other service needing to actually be up and reachable.

---

## Chapter 13: Actuator and Observability

### What Actuator provides

**Spring Boot Actuator** adds production-ready operational features to an application: a set of
built-in HTTP (or JMX) endpoints exposing health status, application info, runtime metrics,
active environment properties, and logger configuration, among others. Enabling it is a single
dependency, `spring-boot-starter-actuator` — the rest is auto-configured, per Chapter 7, and
endpoint exposure/visibility is then tunable through ordinary properties.

Concrete endpoints worth knowing by name, since they're the ones actually used day to day:
`/health` (overall and per-component health status), `/info` (general application metadata),
`/metrics` (memory usage, HTTP traffic, and other runtime metrics), `/env` (currently active
environment properties), and `/loggers` (view and even change logging levels at runtime, without
a redeploy).

### Security is not optional here

Actuator endpoints, if left open in production, can leak genuinely sensitive internal application
details — this is a security decision, not just an operational convenience, and needs to be
treated as such. Securing Actuator means: limiting which endpoints are web-exposed at all by
default (not every endpoint needs to be reachable over HTTP), requiring authentication via Spring
Security for the ones that are, using HTTPS, and considering a dedicated role (e.g.
`ACTUATOR_ADMIN`) restricting who can actually reach these endpoints even when authenticated.

### Custom health indicators

The built-in `/health` endpoint can be extended with application-specific checks by implementing
the `HealthIndicator` interface — a check that a specific downstream database is reachable, or
that a critical external API is currently responding — registered alongside Boot's built-in
checks, and surfaced through `management.endpoint.health.show-details=always` for full detail.
This lets Actuator's health signal actually reflect what "healthy" means for *your specific
system's* real dependencies, not just generic JVM/framework-level signals.

### Distributed tracing — and a version-specific fact worth getting exactly right

Once a single user request crosses multiple microservices, plain per-service logging stops being
enough to reconstruct what actually happened — you need **distributed tracing**, which propagates
a unique identifier across every service boundary a request touches. The core vocabulary: a
**`traceId`** identifies an entire request's journey across *every* service it passes through; a
**`spanId`** identifies *one specific unit of work* within a single service, as part of that
larger trace. A trace is composed of many spans — one or more per service the request touches —
and this "trace contains spans" relationship is the core mental model for reading and reasoning
about distributed traces.

The tooling here has genuinely changed, and it's worth being precise since older material
frequently references the outdated tool as current: **Spring Cloud Sleuth** was the standard
distributed-tracing tool for Spring Boot 2.x, but it is **not supported in Spring Boot 3.x**. The
replacement is **Micrometer Tracing**, which integrates with **OpenTelemetry** as the underlying
observability standard for tracing, metrics, and logging together. If you're working in Spring
Boot 3+, Sleuth references in older tutorials or Stack Overflow answers are simply out of date —
reach for Micrometer Tracing instead.

Getting all of this observability tooling in place — health checks, metrics, tracing — is what
makes the microservices patterns in the next part actually operable in production, rather than
just architecturally elegant on a whiteboard.

---

