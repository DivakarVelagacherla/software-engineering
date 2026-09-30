# Everything About Spring and Spring Boot

*A continuation of Internals of Core Java — how the framework that sits on top of the language actually works*

---

## How this book connects to the last one

*Internals of Core Java* covered the language and the JVM: objects, classes, reflection,
generics, collections, concurrency. This book is the layer built on top of that foundation.
Spring isn't a new language — it's a very large, very deliberate application of the concepts
from that book. The IoC container is built on reflection and classloading (Chapter 1 and
Chapter 14 of the last book). Spring AOP is built on dynamic proxies (also Chapter 14).
`@Transactional` and `@Async` only work because of how Java resolves method calls through those
proxies, which is really a question about polymorphism and dispatch (Chapter 5). Spring's own
internals lean on the Singleton, Factory, and Proxy design patterns (Chapter 15). None of that
is a coincidence — Spring is, in a real sense, a systematic demonstration of what those Core
Java ideas are *for*.

So this book assumes you've read the first one, or know its material, and it will point back to
specific chapters rather than re-explain them. What it adds is everything Spring and Spring Boot
build on top: dependency injection as an architectural discipline, a container that manages
object lifecycles for you, a mechanism for cross-cutting concerns, and — in the second half —
Spring Boot's answer to the operational pain that plain Spring accumulated over a decade of
real-world use.

The structure follows the arc you'd actually want to understand it in: first, what problem
Spring was solving and how it solves it under the hood; then, what new problems *Spring itself*
created at scale, and how Spring Boot was built specifically to solve those; then a deep,
practical tour of building and running real systems with Spring Boot, ending — like the last
book — with production debugging.

---

## Table of Contents

**Part I — Why Spring Exists**
1. The Problem Before Spring
2. Inversion of Control and the IoC Container
3. Dependency Injection in Depth
4. Aspect-Oriented Programming and Spring AOP
5. Data Access, Transactions, and the Rest of the Spring Ecosystem

**Part II — Why Spring Boot Exists**
6. The Pain Points of Plain Spring
7. Auto-Configuration: How Spring Boot Actually Decides What to Configure
8. Starters and Dependency Management
9. The Embedded Server and the New Deployment Model

**Part III — Building With Spring Boot**
10. Building REST APIs
11. Data Access and Multiple Databases
12. Testing Spring Boot Applications
13. Actuator and Observability

**Part IV — Spring Boot at Scale**
14. Microservices Building Blocks
15. Caching, Async, and Reactive Programming
16. Security
17. Scaling, Resilience, and Deployment
18. Production War Stories and Debugging

---

# Part I — Why Spring Exists

## Chapter 1: The Problem Before Spring

Spring exists because writing enterprise Java the "plain" way produces a specific, recurring
kind of pain: classes that manually construct every object they depend on, tightly coupling
each class to concrete implementations of its collaborators rather than to abstractions. Recall
the Dependency Inversion Principle from the Core Java book's SOLID chapter — high-level modules
shouldn't depend directly on low-level implementation details, both should depend on
abstractions. Plain, unassisted Java code violates this constantly, not because developers don't
know better, but because *someone* has to actually construct the concrete objects somewhere, and
without a framework, that responsibility ends up scattered through the codebase, usually right
where it's least convenient — inside the classes that should just be consuming their
dependencies, not manufacturing them.

This has compounding costs. Testing gets harder, because a class that constructs its own
database connection inside its constructor can't easily have that connection swapped for a test
double. Change gets harder, because swapping one implementation for another means hunting down
every `new ConcreteThing()` call site instead of changing one wiring point. And enterprise Java
specifically — before Spring — leaned on heavyweight standards like EJB (Enterprise JavaBeans)
that tried to solve transaction management, remote invocation, and lifecycle management, but did
so with enormous ceremony: verbose interfaces, deployment descriptors, and a programming model
that made simple things complicated in the name of handling distributed, transactional edge
cases most applications never actually needed.

Spring's founding insight was that most of what EJB was trying to provide — transaction
management, object lifecycle, cross-cutting concerns like logging and security — could be
delivered through **plain Java objects** ("POJOs") managed by a lightweight container, without
forcing every class to implement heavyweight framework interfaces or extend framework base
classes. The framework, not your business logic, should carry the ceremony. This is the through-
line for everything else in this part of the book: Spring's core mechanisms (the IoC container,
dependency injection, AOP) exist specifically to let you write plain objects that focus on
business logic, while the framework handles wiring, lifecycle, and cross-cutting concerns
*around* those objects rather than *inside* them.

---

## Chapter 2: Inversion of Control and the IoC Container

### What "inversion of control" actually inverts

In ordinary, non-framework code, a class that needs a collaborator typically constructs it
directly: `class OrderService { private PaymentGateway gateway = new StripeGateway(); }`. The
class is in control of creating its own dependencies. **Inversion of Control (IoC)** flips this:
the *framework* (or "container") takes control of that flow instead, constructing objects and
handing dependencies to classes that need them, rather than those classes constructing their own
dependencies. **Dependency Injection (DI)** is the specific technique that implements this
inversion — a class declares what it needs (typically via constructor parameters), and the
container supplies concrete instances of those needs from outside.

The payoff mirrors the Dependency Inversion Principle directly: your `OrderService` can depend
on a `PaymentGateway` interface, entirely unaware of whether it's talking to Stripe, a mock, or
something else — the container decides which concrete implementation to hand it, and that
decision lives in one place (configuration), not scattered across every class that happens to
need a `PaymentGateway`.

### What a Spring Bean actually is

A **Spring Bean** is simply an object whose creation and lifecycle are managed by the Spring
container rather than by application code calling `new` directly. Beans are the unit of
everything the container does: it creates them, wires their dependencies, and (as we'll cover in
the next chapter) manages their lifecycle from construction through destruction. "Managed by the
container" is the entire distinction between a bean and any other Java object — architecturally
they're just POJOs, exactly as Chapter 1's founding insight intended.

### The container itself: BeanFactory and ApplicationContext

Spring provides two IoC container implementations, one a strict superset of the other:

- **`BeanFactory`** is the basic container — it handles bean creation and dependency wiring, and
  little else. It's lightweight and suited to memory-constrained scenarios, but rarely used
  directly in modern applications.
- **`ApplicationContext`** is the container almost everyone actually uses. It's built on top of
  `BeanFactory`'s core capabilities and adds a substantial amount more: event propagation (so
  beans can publish and listen for application-level events — the same event mechanism covered
  in the Spring Boot chapters later), tighter AOP integration, internationalization support, and
  web-context awareness for web applications.

Under the hood, the container's job — reading bean definitions, instantiating objects, and
wiring their dependencies — is a direct, large-scale application of **reflection** from the Core
Java book's Chapter 14. When the container encounters a class it needs to instantiate, it uses
reflection to inspect constructors and fields, decide which dependencies are needed, locate
matching beans elsewhere in the context, and invoke the constructor (or set the fields)
reflectively rather than through code you wrote by hand. This is exactly the same
inspect-then-construct-and-wire pattern the Core Java book described for a minimal hand-rolled
dependency injection framework built on `@Inject` field scanning — Spring's container is that
same idea, matured into a full production framework.

### `@Configuration` and `@Bean`: declaring beans explicitly

`@Configuration` marks a class as a source of bean definitions. `@Bean`, placed on a method
inside such a class, tells the container that the method's return value should be registered
and managed as a bean — the container calls that method (and handles its dependencies, if the
method itself takes parameters) to obtain the instance. This is the explicit, code-based way to
tell the container "here is an object you should manage," most commonly reached for when you
need to construct a bean from a third-party class you don't own and therefore can't annotate
directly (`@Component` requires modifying the class itself; `@Bean` doesn't).

---

## Chapter 3: Dependency Injection in Depth

### Constructor, setter, and field injection

Spring supports three ways to actually deliver dependencies into a bean, and the difference
between them isn't stylistic — it has real correctness implications:

- **Constructor injection** supplies all dependencies as constructor parameters at the moment of
  object creation. The object is guaranteed to be fully, validly constructed the instant it
  exists — there's no window where it's half-wired, and required dependencies are structurally
  impossible to omit, since the compiler enforces the constructor's parameter list. This also
  naturally supports declaring dependency fields as `final` (immutability, straight out of the
  Core Java book's memory and thread-safety chapters), and it makes a class's dependencies
  visible and testable without needing the Spring container at all — you can just call the
  constructor directly in a unit test.
- **Setter injection** supplies dependencies via setter methods after construction, allowing them
  to be optional or changed later, at the cost of a window where the object exists but isn't yet
  fully wired.
- **Field injection** (`@Autowired` directly on a field) is the most concise but the most
  discouraged in practice — it hides a class's real dependencies from anyone reading its
  constructor, makes the class harder to instantiate outside the Spring container (for testing),
  and provides none of constructor injection's immutability benefits.

**Constructor injection is the recommended default**, precisely because it's the only one of the
three that makes an incompletely-wired object structurally impossible to create. The one
legitimate exception is breaking a genuine circular dependency (below) — where constructor
injection's very strictness is what makes the cycle unsatisfiable in the first place.

`@Autowired` triggers Spring's automatic dependency resolution — locating a matching bean by
type (and by name/qualifier when there's ambiguity) and injecting it without you writing manual
lookup code. It can be applied to constructors, setters, or fields; as covered above, constructor
placement is the default choice.

### Resolving ambiguity: `@Qualifier` and `@Primary`

If more than one bean of the same type exists in the container, `@Autowired` alone is
genuinely ambiguous — Spring cannot guess which one you mean, and the container throws
`NoUniqueBeanDefinitionException` at startup rather than silently guessing. This is a hard,
loud failure, not a subtle wrong-bean bug — a deliberately safe design choice. Two annotations
resolve the ambiguity:

- **`@Qualifier("beanName")`**, placed at the injection point, explicitly names which bean to
  use — precise, per-usage control.
- **`@Primary`**, placed on the bean definition itself, marks it as the default choice whenever
  an injection point doesn't specify a qualifier — a fallback rather than a per-usage override.

### Stereotypes: `@Component` and its specializations

`@Component` is Spring's generic stereotype — any class annotated with it (or discovered via
`@ComponentScan`, which tells the container which packages to search for annotated classes) gets
automatically registered as a bean, with no manual registration required. `@Service`,
`@Repository`, and `@Controller`/`@RestController` are all specializations of `@Component`: they
register a bean identically under the hood, but each signals the *role* that class plays —
business logic, data access, or web request handling, respectively. Technically interchangeable
with plain `@Component`, but the specializations carry real value: they make a codebase's
architecture legible at a glance, and `@Repository` specifically adds one genuine functional
difference beyond convention — automatic translation of persistence-layer exceptions into
Spring's unified `DataAccessException` hierarchy, so calling code doesn't need to know or care
whether a given repository is backed by JPA, JDBC, or something else.

### Bean scopes

A bean's **scope** governs how many instances of it exist and how long each one lives:

- **Singleton** (the default) — exactly one shared instance for the entire application context.
  Used for stateless services, shared configuration, and shared resources.
- **Prototype** — a fresh instance every time the bean is requested. Used when a bean needs
  per-use or per-caller state.
- **Request** — one instance per HTTP request (web applications only).
- **Session** — one instance per user session (web applications only).
- **Global Session** — one instance per global session, a portlet-era edge case rarely seen in
  modern applications.

A genuinely important correctness note follows directly from the singleton default: **singleton
beans are not automatically thread-safe.** Because a singleton is shared across every concurrent
request/thread hitting the application, any mutable state it holds is exactly the kind of shared
mutable state the Core Java book's concurrency chapters warned about — it needs explicit
protection (synchronization, thread-safe data structures) or, far more idiomatically in Spring,
it should simply be designed **stateless**, with no mutable instance fields at all, so there's
nothing to race on in the first place. Most well-designed Spring service beans are stateless for
exactly this reason.

### Bean lifecycle

Beans move through a lifecycle — creation, dependency injection, initialization callbacks, use,
and eventual destruction callbacks — managed entirely by the container. Understanding this
lifecycle matters in large applications for two practical reasons: it's how you correctly hook
resource setup and teardown (opening a connection pool at startup, releasing it at shutdown), and
it's frequently the actual root cause when debugging startup-order or dependency-resolution
issues — many confusing "why isn't my bean ready yet" bugs are really lifecycle-ordering bugs.

### Circular dependencies

A **circular dependency** occurs when Bean A requires Bean B to be constructed, and Bean B
simultaneously requires Bean A — with constructor injection specifically, this is *unsatisfiable*:
neither bean can be fully constructed first, because each one's constructor needs the other
already built. The container detects this at startup and fails to start the application, rather
than hanging — this is a hard failure at context-startup time, not a runtime deadlock in the
threading sense, despite how the situation is sometimes loosely described.

Three ways to resolve it, in order of how much they actually fix versus paper over the
underlying design issue:

1. **Switch to setter or field injection** for one side of the cycle — this lets the container
   instantiate a bean's bare shell before its dependencies are set, breaking the strict
   construction-order requirement constructor injection imposes. This is the pragmatic, immediate
   fix, and the one legitimate common exception to "always prefer constructor injection."
2. **Use `@Lazy`** on one side to defer that dependency's actual initialization until it's first
   used, rather than at startup — breaks the cycle by delaying when the mutual dependency is
   actually resolved.
3. **Redesign to remove the cycle entirely** — extract a shared interface or a third
   collaborator that both classes can depend on instead of depending on each other directly. This
   is the real fix: a circular dependency is very often a signal that two classes are more
   tightly coupled than they should be, and the first two options are legitimate short-term
   tools, not substitutes for reconsidering the design.

---

## Chapter 4: Aspect-Oriented Programming and Spring AOP

### The problem AOP solves

Some concerns don't belong to any single class's core responsibility, but cut across many
classes at once: logging, security checks, transaction management. Implementing these inline,
scattered through every method that needs them, duplicates the same boilerplate everywhere and
tangles core business logic together with orthogonal concerns. **Aspect-Oriented Programming
(AOP)** modularizes these "cross-cutting concerns" into a separate unit — an *aspect* — defined
once and applied at specified points across the application, keeping the core logic focused on
what it's actually supposed to do.

AOP's own vocabulary is precise and worth having exactly right: a **join point** is a specific
point during program execution — a method call, for instance — where an aspect *could*
potentially apply. A **pointcut** is an expression that selects a *set* of join points where
advice should actually be applied. The distinction: a join point is a location that exists;
a pointcut is the query that picks which of those locations actually get the aspect's behavior.
"Advice" is the actual code that runs at a matched join point (before, after, or around it).

AOP's honest tradeoff, worth stating plainly rather than glossing over: it genuinely improves
code cleanliness and reduces duplication, but it does so by making control flow *less visible* at
the call site — a method annotated `@Transactional` doesn't show, in its own body, that a
transaction boundary and rollback logic are wrapped around it. This can make execution harder to
trace and debug, especially for developers unfamiliar with the codebase's aspects. It's a real
cost, not a hypothetical one.

### How Spring actually implements this: dynamic proxies

Here is the mechanism, and it connects directly back to the Core Java book's Chapter 14 on
dynamic proxies. When you annotate a bean's method with something like `@Transactional` or
`@Async`, Spring doesn't rewrite your class's bytecode. Instead, at context-startup time, it
generates a **proxy object** wrapping your real bean — using `java.lang.reflect.Proxy` for
interface-based beans, or a subclassing-based proxy (CGLIB) for concrete classes without
interfaces. Every call that goes *through the proxy* is intercepted: the proxy runs the aspect's
advice (start a transaction, log the call, check a security constraint) and then delegates to
your actual method, exactly the mechanism the Core Java book described generically for
`InvocationHandler`-based interception — Spring's AOP support is that same generic tool, applied
specifically to enable declarative, annotation-driven cross-cutting behavior.

This explains a real, extremely common gotcha, worth walking through explicitly because it trips
up developers at every experience level: **`@Async`, `@Transactional`, `@Cacheable`, and every
other proxy-backed Spring annotation only take effect on calls that go *through the proxy* —
meaning calls made from *outside* the bean.** If a method inside a bean calls another
`@Async`-annotated method *on itself* (`this.someAsyncMethod()`), that call happens directly on
the real object, bypassing the proxy entirely — the annotation is silently ignored, and the call
runs synchronously with no error or warning. This is not a bug in Spring; it's a direct, logical
consequence of how proxy-based AOP works, and once you understand the proxy mechanism, the
behavior stops being mysterious and becomes predictable. The practical fix is to move the
self-invoked method into a separate bean, so the call genuinely goes through the proxy from
outside.

---

## Chapter 5: Data Access, Transactions, and the Rest of the Spring Ecosystem

### `@Transactional`

`@Transactional` is itself an AOP-backed annotation, working through exactly the proxy mechanism
described in Chapter 4. Placed on a method (idiomatically, at the **service layer** — the layer
that coordinates business logic across multiple lower-level operations, not the controller or
repository layer), it demarcates a transaction boundary: everything inside either commits
together or rolls back together on failure, preventing partial updates from leaving data in an
inconsistent state. The proxy intercepts the call, starts a transaction before your method body
runs, and commits or rolls back based on whether the method completes normally or throws.

### `CrudRepository` and `JpaRepository`

Spring Data builds a repository abstraction on top of this same DI-and-proxy machinery:
`CrudRepository` provides basic Create/Read/Update/Delete operations for an entity type without
you writing any implementation at all — Spring generates the implementation at runtime, again via
a dynamically-generated proxy, based on the method signatures you declare in an interface.
`JpaRepository` extends `CrudRepository` and adds JPA-specific capabilities: pagination, batch
operations, and explicit control over flushing the persistence context. Use plain
`CrudRepository` when only basic access is needed; reach for `JpaRepository` when you need those
JPA-specific extras.

### Design patterns inside Spring itself

It's worth naming explicitly that Spring's own internals are a working showcase of the design
patterns covered in the Core Java book's Chapter 15: the container's default singleton bean scope
is a direct application of the **Singleton pattern** (at framework scale, managing potentially
thousands of singleton instances per application, rather than one hand-written class); `@Bean`
factory methods and Spring Data's repository generation are applications of the **Factory
pattern** (delegating object creation rather than calling constructors directly); and Spring AOP,
as just covered, is a large-scale application of the **Proxy pattern**. Recognizing these patterns
inside the framework you're using is a genuinely useful way to reinforce why they matter in your
own code — Spring isn't just documentation for these patterns, it's proof they scale.

### The wider ecosystem, briefly

A few more pieces of core Spring worth knowing before moving to Spring Boot, since Boot builds on
all of them: **Spring MVC** is the traditional synchronous, blocking web framework (thread per
request). **Spring WebFlux**, introduced in Spring 5, is a non-blocking, reactive alternative
built on Project Reactor, suited to high-concurrency workloads that need to handle many
simultaneous connections with fewer threads (we cover this in depth in Chapter 15, alongside the
reactive-streams backpressure mechanism). **Spring Batch** provides infrastructure for
large-volume batch data processing — jobs composed of steps, each typically wiring a Reader
(pulls data), a Processor (applies business logic), and a Writer (outputs the result), all run
inside Spring's transactional and monitoring context.

With the core framework's mechanics covered — IoC, DI, AOP, and the data/transaction layer built
on top of them — we're ready for the pivot this book is really structured around: what actually
went wrong with *using* Spring at scale, and what Spring Boot was built specifically to fix.

---

# Part II — Why Spring Boot Exists

## Chapter 6: The Pain Points of Plain Spring

Everything in Part I is genuinely powerful — but using it in a real application meant confronting
a specific, recurring set of friction points that had nothing to do with IoC or AOP being wrong
ideas, and everything to do with the *ceremony* required to actually stand up and deploy an
application built on them:

- **Manual, extensive configuration.** Before sensible defaults existed, wiring a non-trivial
  Spring application meant explicitly declaring beans, data sources, transaction managers, and
  web-framework infrastructure yourself — via XML historically, or explicit `@Configuration`
  classes later. None of it was wrong, but almost all of it was *boilerplate*: the same kind of
  setup, repeated project after project, with only minor variation.
- **Dependency version management.** Spring itself is a large ecosystem of separately-versioned
  modules (Core, Data, Security, Web, and more), plus whatever third-party libraries a given
  project needs on top. Manually keeping every one of these versions mutually compatible, across
  an entire project, was genuinely tedious and a real, recurring source of runtime errors when
  versions drifted out of alignment.
- **Manual server setup and deployment.** A traditional Spring web application was packaged as a
  WAR file and deployed onto a separately-installed, separately-configured server (Tomcat, Jetty,
  WildFly). Getting a new environment stood up meant configuring that external server correctly
  *in addition to* configuring the application — two separate concerns that had to be kept in
  sync by hand.
- **Slower time-to-first-running-app.** The cumulative effect of the above: standing up even a
  simple new Spring project took real, non-trivial setup effort before you could write your first
  line of actual business logic.

None of these problems meant Spring's core ideas — IoC, DI, AOP — were wrong. They meant the
*experience of using* Spring accumulated exactly the kind of boilerplate and ceremony Spring
itself had originally set out to eliminate from EJB. **Spring Boot is Spring's answer to Spring's
own success**: once a framework becomes the default way to build a whole category of
applications, the friction in adopting and configuring it becomes the next problem worth solving.
Spring Boot doesn't replace anything from Part I — `@Autowired`, `@Transactional`, the
`ApplicationContext`, AOP proxies, all of it is still there, doing exactly what it always did.
Spring Boot's entire contribution is eliminating the *setup and configuration ceremony* around
that same core, through three specific mechanisms covered in the next three chapters:
auto-configuration, starters, and an embedded server.

---

## Chapter 7: Auto-Configuration — How Spring Boot Actually Decides What to Configure

### `@SpringBootApplication`: the entry point, unpacked

Nearly every Spring Boot application begins with a single annotation on its main class,
`@SpringBootApplication`, which is genuinely just a convenience bundle of three separate
annotations, each doing distinct work:

- **`@Configuration`** — marks the class itself as a source of bean definitions (Chapter 2).
- **`@ComponentScan`** — tells the container to scan the current package (and sub-packages) for
  `@Component`-stereotyped classes to auto-register (Chapter 3).
- **`@EnableAutoConfiguration`** — the genuinely new piece, and the mechanism this chapter is
  about.

### Auto-configuration is not magic — it's conditional configuration, evaluated

`@EnableAutoConfiguration` tells Boot to automatically configure the application based on the
libraries present on the classpath. Concretely: Boot ships with a large set of pre-written
`@Configuration` classes covering common infrastructure (a web MVC setup, a JPA/DataSource setup,
a security setup, and dozens more), each one guarded by **`@Conditional`-family annotations** —
`@ConditionalOnClass` is the most common, meaning "only activate this configuration if a specific
class is present on the classpath." At startup, Boot performs **condition evaluation**: it
examines the classpath, the beans already registered, and the active properties, and decides
which of its bundled auto-configuration classes actually apply to *your specific project*.

This is the entire mechanism, stated precisely, because it's worth being precise: auto-
configuration is not the framework guessing or doing something unknowable — it's a large,
pre-written library of conditional configuration classes, each answering "does this specific
piece of infrastructure make sense for what's actually on this classpath," evaluated
automatically so you don't have to write the equivalent wiring by hand. If you have a JDBC driver
and Spring Data JPA on your classpath, Boot's `DataSourceAutoConfiguration` and related classes
activate and wire up a `DataSource`, an `EntityManagerFactory`, and a `TransactionManager` for
you, using sensible defaults — because their `@ConditionalOnClass` guards matched. If you don't
have those dependencies, those same configuration classes simply don't activate, and nothing is
wired that you don't need.

### Overriding auto-configuration

Because auto-configuration is just conditional bean registration, overriding it is equally
mechanical: **`@ConditionalOnMissingBean`** is the annotation Boot's own auto-configuration
classes use internally to "back off" — if you've already defined your own bean of the relevant
type (in your own `@Configuration` class), the auto-configured default doesn't get created at
all, and your explicit bean wins. This is the actual mechanism behind "auto-configuration is
overridable" — it isn't a separate override system, it's the same conditional-evaluation
machinery, just checking for the presence of *your* bean as one of its conditions.

You can also disable specific auto-configuration classes explicitly, without needing to define a
competing bean: `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`, or
equivalently the `spring.autoconfigure.exclude` property. And you can customize an activated
auto-configuration's *behavior* (rather than replacing it outright) through ordinary
`application.properties`/`application.yml` settings, since most auto-configuration classes read
their defaults from exactly those properties.

If two different auto-configuration classes happen to define a bean with the same name, the
later one processed by the container generally takes precedence — but this ordering is fully
controllable via `@AutoConfigureOrder`, `@AutoConfigureBefore`, and `@AutoConfigureAfter`, which
let you explicitly declare load order rather than relying on incidental processing sequence.

---

## Chapter 8: Starters and Dependency Management

### Starter dependencies

A **Spring Boot starter** is a single dependency that transitively bundles every library
typically needed for one specific concern. `spring-boot-starter-web` pulls in Spring MVC, an
embedded Tomcat, Jackson (for JSON), and their compatible-version dependencies, all through one
declared dependency. `spring-boot-starter-data-jpa` bundles Spring Data JPA, Hibernate, and
related infrastructure. `spring-boot-starter-security` bundles Spring Security's core pieces.
This is the direct, concrete fix for the "dependency version management pain" identified in
Chapter 6: instead of individually tracking and version-aligning a dozen related libraries by
hand, you declare one starter, and Boot's dependency management ensures every library it pulls in
is a mutually compatible version.

### `spring-boot-starter-parent` and dependency version alignment

Most Boot projects inherit from `spring-boot-starter-parent` in their build file. This parent POM
supplies default Maven configuration: aligned versions for the entire ecosystem of dependencies
Boot commonly works with, a default Java version, and common build plugins — meaning individual
projects rarely need to pin specific library versions by hand at all. If a starter dependency
happens to pull in conflicting versions of some transitive library from two different paths,
Boot's dependency resolution mechanism picks a single, compatible version for the final build
automatically, preventing the classpath conflicts that plagued manual dependency management.

### Externalized, format-flexible configuration

Boot externalizes configuration entirely from code — `application.properties`,
`application.yml`, environment variables, and command-line arguments can all supply the same
logical settings, letting the identical build artifact run correctly across development, testing,
and production without any code changes. When sources overlap, there's a defined precedence
order, from highest to lowest priority: **command-line arguments > properties/YAML files
(including profile-specific variants) > environment variables/system properties > Boot's own
built-in defaults.** Getting this order wrong in your head is a genuinely common source of "why
isn't my environment variable taking effect" debugging sessions — it's worth memorizing.

Boot's **relaxed binding** adds format tolerance on top of this: a property like `server.port`
can be supplied as `server.port`, `server-port`, or `SERVER_PORT` and Boot resolves all of them
to the same logical setting — letting each configuration source (a properties file, a shell
environment variable) use its own natural casing convention without breaking configuration
binding.

**Spring Profiles** let you segregate configuration by environment: `application-dev.properties`
and `application-prod.properties` hold environment-specific settings, activated via the
`spring.profiles.active` property (settable via any of the config sources above), and `@Profile`
on a bean definition restricts that bean to being registered only when a matching profile is
active — the mechanism that lets one codebase behave correctly across genuinely different
deployment environments.

YAML versus properties files is a real, if secondary, format choice: YAML supports hierarchical,
nested configuration (more readable for complex structures) and comments, but is more
whitespace/indentation-sensitive and therefore more error-prone, and somewhat less universally
familiar than flat key-value properties files. Both can coexist in the same project; where keys
overlap, `application.properties` takes precedence over `application.yml`.

---

## Chapter 9: The Embedded Server and the New Deployment Model

### Collapsing two deployment steps into one

This is, concretely, the biggest single deployment-experience change Spring Boot made, and it's
worth stating plainly as the direct fix for the "manual server setup" pain point from Chapter 6.
Traditional Spring web deployment required a WAR file *and* a separately-installed, separately-
configured external servlet container to host it — two things, kept in sync by hand across every
environment. Spring Boot **embeds the server inside the application artifact itself**: Tomcat,
Jetty, or Undertow ships as a dependency (pulled in transitively by `spring-boot-starter-web`,
which defaults to Tomcat), and the entire application — your code plus the server that runs it —
packages into one executable JAR, runnable anywhere a JVM exists with a single `java -jar`
command. There is no longer a separate "configure the server" step at all in the common case;
the server *is* part of what you built.

Boot decides which embedded server to use based on classpath presence: if a specific server
dependency (Tomcat, Jetty, Undertow) is present, that one is auto-configured; if none is
specified explicitly, Tomcat is the default, since it's pulled in transitively by
`spring-boot-starter-web`. Switching servers means excluding the current one and including the
desired one as a dependency — Boot's auto-configuration (Chapter 7) handles the rest.

The default embedded Tomcat port is 8080, changeable via the `server.port` property. You can
disable the embedded web server entirely — `spring.main.web-application-type=none` — to build a
non-web application (a batch job, a messaging consumer) using the exact same Boot
infrastructure and dependency-management benefits, without paying for or starting a server at
all.

### WAR deployment is still available, when it's actually the right call

None of this forecloses traditional WAR deployment — it's genuinely still supported, for
organizations standardized on shared external application servers. Switching a Boot project from
JAR to WAR packaging requires changing the packaging type in the build file and having the main
application class extend `SpringBootServletInitializer`, which acts as the bridge letting a Boot
application bootstrap correctly inside an external servlet container. The tradeoff is exactly
the inverse of the embedded-server benefit: an embedded server is faster to set up and more
portable, at the cost of less centralized control over server configuration when many
applications need to share one externally-managed server's resources — which is a real,
legitimate reason some organizations still choose external deployment for specific applications.

### Containerization builds naturally on this model

Because a Boot application is already a single self-contained JAR with everything needed to run,
it containerizes especially cleanly — a Dockerfile just needs a Java base image, the JAR, and a
run command. Boot goes one step further with built-in tooling: the `spring-boot:build-image`
Maven/Gradle plugin goal packages an application into a Docker image *without requiring a
hand-written Dockerfile at all*, using Cloud Native Buildpacks under the hood — a genuinely
Boot-specific convenience worth knowing about, beyond generic Java-in-Docker knowledge. We'll
return to containerization and deployment strategy in more depth in Chapter 17.

With auto-configuration, starters, and the embedded-server model covered, the "why Boot exists
and how it actually works" story is complete. The rest of this book is a practical tour of
building, testing, and running real systems on top of everything covered so far.

---

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
