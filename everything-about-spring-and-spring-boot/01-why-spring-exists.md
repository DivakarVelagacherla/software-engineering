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

