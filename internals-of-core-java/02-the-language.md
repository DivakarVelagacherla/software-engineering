# Part II — The Language

## Chapter 4: The Object Model — Classes, Objects, and Their Building Blocks

### Classes and objects

A **class** is a blueprint: it defines what state (fields) and behavior (methods) its
instances will have. An **object** is a concrete instance of a class — a specific banking
`Customer` with a specific name and account number, created from the `Customer` blueprint,
capable of calling `deposit()`, `withdraw()`, and `checkBalance()` because the class defined
those methods. A class can technically be declared with no fields or methods at all and still
be instantiated — the resulting objects just have no particular state or behavior beyond what
`Object` itself provides.

Objects are created several ways: the `new` keyword (`Customer c = new Customer();`), a
**factory method** on another class (`Calendar.getInstance()`), `clone()`ing an existing
object, deserialization (Chapter 13), or reflectively (Chapter 14). The `new` keyword is by far
the most common, but factory methods are worth noticing as a pattern in their own right — they
let a class control *how* instances get created (returning a cached instance, a subtype chosen
based on input, etc.) in a way a raw constructor call can't.

### Constructors

A **constructor** is a special method that initializes a new object; it shares the class's
name and has no return type. Constructors can be **overloaded** — multiple constructors with
different parameter lists, giving callers different ways to construct an object depending on
what information they have available at the time. What constructors *cannot* be is
**overridden**, and they cannot be **polymorphic** in the way instance methods are: there's no
dynamic dispatch mechanism for constructors, because by definition a constructor call always
resolves to a specific class's constructor at compile time, not based on a runtime type that
doesn't exist yet (the object is still being built).

A **private constructor** is legal and useful — it's the standard tool for preventing outside
instantiation, used heavily by classes that expose only static factory methods, utility classes
with only static members, and the Singleton pattern (Chapter 15).

### `this` and `super`

`this` is a read-only reference to the current instance — you cannot reassign it. It resolves
ambiguity when a parameter shadows a field (`this.name = name;`), lets a constructor invoke a
sibling constructor (`this(...)`), and can be passed or returned to hand out a reference to the
current object. `super` accesses the parent class's members and constructor — `super.method()`
calls the parent's version of an overridden method, and `super(...)` as the first line of a
constructor invokes the parent's constructor. Attempting to use `super` in a class with no
explicit superclass (i.e., beyond the implicit `Object`) is a compile error, since there's
nothing meaningful for it to refer to.

### The methods every object inherits

Every class in Java implicitly extends `Object`, which means every object automatically has:
`equals()`, `hashCode()`, `toString()`, `clone()`, `finalize()` (legacy, see Chapter 3),
`wait()`, `notify()`, and `notifyAll()` (concurrency primitives, covered in Chapter 11). We'll
spend real time on `equals()` and `hashCode()` together in Chapter 8, because getting them right
— and *together* — is one of the most consequential correctness decisions you make in ordinary
Java code, with direct consequences for how `HashMap` and `HashSet` behave.

### Packages and access control

A **package** is a namespace grouping related classes and interfaces — organizationally similar
to folders, but with real compiler-enforced meaning: packages prevent naming collisions, allow
package-private visibility, and structure large codebases into modular, locatable units. If two
different packages happen to define classes with the same simple name, both can be used in the
same program as long as you disambiguate with the fully-qualified name (`package1.ClassName`
vs. `package2.ClassName`).

Java's four access levels form a strictly nested hierarchy of visibility:

- **`public`** — accessible from anywhere.
- **`protected`** — accessible within the same package, plus subclasses anywhere.
- **default (package-private, no modifier)** — accessible only within the same package. This is
  what you get if you *omit* a modifier entirely, and it's a genuinely useful middle ground, not
  just an accident of leaving something off.
- **`private`** — accessible only within the declaring class itself.

A **top-level class** (as opposed to a nested class) cannot be declared `private` or
`protected` — only `public` or package-private — because a top-level class that no other class
could ever reference would be useless by construction.

The reason to prefer `private` fields with public getters/setters over plain public fields
isn't ceremony for its own sake: it's that it lets you add validation, change the internal
representation later without breaking callers, and control exactly what mutation is allowed —
all without touching any code outside the class. This is your first concrete look at
**encapsulation**, the first of the four pillars we cover properly in the next chapter.

### Nested and anonymous classes

Java lets you define a class inside another class, and the flavor you choose determines its
relationship to the enclosing instance:

- A **non-static (inner) class** holds an implicit reference to its enclosing instance and can
  access even the enclosing class's `private` members. Because of that implicit link to an
  instance, it **cannot declare static members** — static belongs to the class itself, but a
  non-static inner class only ever exists tied to a specific outer instance.
- A **static nested class** behaves like an ordinary top-level class that just happens to be
  namespaced inside another — no implicit outer-instance reference, and it *can* have static
  members.
- A **local class** is defined inside a method body, scoped to that method.
- An **anonymous class** is a class with no name at all, defined and instantiated in the same
  expression — historically the standard way to implement a one-off interface or subclass
  inline (an event handler, a `Runnable`), before lambdas (Chapter 10) took over most of that
  role for functional interfaces specifically. Anonymous classes remain useful when you need to
  implement multiple methods, hold your own instance fields, or extend a concrete class rather
  than an interface — things a lambda structurally cannot do.

With the object model's vocabulary in place — classes, constructors, access control, nesting —
we're ready to talk about what these building blocks are actually *for*: the four pillars of
object-oriented design that Java's syntax exists to express.

---

## Chapter 5: The Four Pillars of OOP

Object-oriented programming rests on four ideas — encapsulation, inheritance, polymorphism, and
abstraction — that Java bakes directly into its syntax rather than leaving as conventions. This
chapter treats each one not as a definition to memorize, but as a design tool with real
tradeoffs, because that's how they actually get used.

### Encapsulation: hiding state behind behavior

Encapsulation means bundling data and the methods that operate on it into a single unit, and
restricting direct access to that data from outside — "putting important information into a
safe," and controlling exactly which doors into it exist. In Java this is enforced through
access modifiers (Chapter 4): fields go `private`, and any interaction happens through
deliberately exposed methods.

The payoff isn't abstract. It's that you can change how data is *stored* internally — switch a
`List` to a `Set`, add caching, add validation — without breaking any code outside the class,
because outside code was never allowed to touch the internals directly in the first place. It
also directly improves correctness and security: if the only way to modify a field is through a
method you control, that method is the one place you need to enforce invariants, and unwanted
external changes simply aren't possible.

### Inheritance: reuse through IS-A relationships

Inheritance lets a class acquire the fields and methods of another via `extends`, so a `Car`
class doesn't have to redeclare everything a general `Vehicle` already defines. A class cannot
extend itself (a compile error), and — this is a deliberate design decision, not an oversight —
**Java does not support multiple inheritance of classes**. If `ElectricCar` could extend both
`Vehicle` and `RechargeableDevice`, and both defined a conflicting `powerLevel()` method, there
would be no unambiguous way to resolve the collision; this is the classic **diamond problem**,
and Java sidesteps it entirely by disallowing multiple class inheritance outright.

Interfaces provide a controlled workaround: a class can implement *many* interfaces at once,
because interfaces (traditionally) contribute only method signatures, not state or competing
implementations. Since Java 8 introduced default methods on interfaces (covered fully in
Chapter 10), the diamond problem can technically resurface — if two implemented interfaces
declare *default* methods with the same signature, the implementing class **must** explicitly
override the method to resolve the ambiguity, optionally delegating to one interface's version
via `InterfaceName.super.methodName()`. The compiler simply refuses to let the ambiguity stand
unresolved.

### Composition: reuse through HAS-A relationships

Where inheritance models "is a kind of," **composition** models "is built from" — a class holds
a reference to another class as a field, rather than inheriting from it. `Car` doesn't inherit
from `Engine`; it *has* an `Engine`. This distinction sits on top of a small, precise vocabulary
worth having exactly right:

- **Association** — the most general relationship between two classes; they know about and use
  each other, with no stronger claim than that.
- **Aggregation** — a "weak" HAS-A relationship: the contained object can exist independently of
  the container. A `Department` has `Employee`s, but an `Employee` continues to exist if the
  `Department` is dissolved.
- **Composition (in the strict sense)** — a "strong" HAS-A relationship: the contained object's
  lifecycle is bound to the container's. A `House` has `Room`s that don't meaningfully exist
  independently of the house.

The design principle **"favor composition over inheritance"** follows directly from this: using
objects within other objects, instead of inheriting from a parent class, avoids tight coupling
to a parent's implementation details and sidesteps the "fragile base class" problem, where a
change to a superclass ripples unpredictably through every subclass. A concrete worked example:
refactoring a `Vehicle` class that has both `fly()` and `sail()` methods — clearly violating
single responsibility — into separate `Airplane` and `Boat` classes that each `extends Vehicle`
(a genuine IS-A relationship, since both really are vehicles), while pulling shared capabilities
like propulsion into a composed `Engine` field (a HAS-A relationship, since "has an engine"
isn't part of what makes something a vehicle at all). Inheritance isn't wrong here — it's
composition *instead of* inheritance for the parts that were never really an IS-A relationship
to begin with.

### Polymorphism: one interface, many behaviors

Polymorphism means the same piece of code behaves differently depending on the actual type of
object it's operating on — call `draw()` on a `Circle` reference and get a circle; call it on a
`Square` reference and get a square, even if both are being handled through a shared `Shape`
type. Java gives you two distinct flavors of this, resolved at two different times:

- **Compile-time polymorphism (overloading)** — multiple methods in the same class sharing a
  name but differing in parameter count, type, or order. The compiler picks which overload to
  call by matching the arguments at the call site, purely from static information; overload
  resolution can never be influenced by anything known only at runtime. Overloading *cannot* be
  distinguished by return type alone — two methods with identical parameter lists but different
  return types are not valid overloads.
- **Runtime polymorphism (overriding, via dynamic method dispatch)** — a subclass provides its
  own implementation of a method already defined by a superclass, with the same name, same
  parameters, and a compatible return type; the overriding method also cannot be *more*
  restrictive in access than the one it overrides. Which implementation actually runs is decided
  at runtime, based on the object's actual class — not the type of the reference variable
  pointing to it. This is dynamic method dispatch, and it's the mechanism that lets code written
  against a `Shape` reference correctly call `Circle`'s `draw()` when the object underneath
  happens to be a `Circle`.

The `@Override` annotation doesn't change behavior — it's a compiler check, telling the
compiler "I intend this to override something from a superclass," so that a typo in the method
signature (which would otherwise silently create an unrelated overload instead of an override)
becomes a compile error instead of a silent bug. Using it is close to free and catches a real
category of mistake.

### Abstraction: exposing what, hiding how

Abstraction means presenting only the essential contract of something and hiding the
implementation details behind it — you know a `List` supports `add()`, `get()`, and iteration,
without needing to know whether it's backed by an array or a linked structure underneath. The
practical payoff is **loose coupling**: because callers depend only on the abstraction, the
concrete implementation can change (or be swapped entirely) without breaking anything that
depends on the contract.

Java gives you two tools for expressing abstraction, and choosing between them is one of the
most common real design decisions in the language:

- An **abstract class** achieves *partial-to-full* abstraction: it can mix abstract method
  declarations (no implementation) with fully concrete methods, and it can hold instance state.
  A class containing even one abstract method must itself be declared `abstract`, and abstract
  classes can never be instantiated directly — they exist purely to be extended. Because Java
  only allows single class inheritance, a class can extend at most one abstract class.
- An **interface** traditionally achieved *full* abstraction — pure contract, no implementation,
  no state. Since Java 8, interfaces can also carry `default` methods (with a body, overridable
  by implementers) and `static` methods (belonging to the interface itself, never overridable);
  since Java 9, `private` interface methods are allowed too, for sharing code between default
  methods without exposing it. A class can implement *any number* of interfaces at once — this
  is Java's substitute for multiple inheritance of *behavior contracts*, even though multiple
  inheritance of *state* remains disallowed.

The rule of thumb that falls out of this: reach for an **interface** when you want unrelated
classes to share a contract without necessarily sharing implementation or a common ancestor —
`Comparable` doesn't care whether you're a `String` or a custom `Employee` class. Reach for an
**abstract class** when you have closely related classes that should share both some concrete
behavior *and* some state, in addition to a contract. `Comparable` (natural, single ordering,
defined by the class itself via `compareTo()`) versus `Comparator` (external, and you can define
as many different orderings as you need) is a good small case study in exactly this
kind of interface-driven design: both express "how do I sort this," but one is baked into the
type and one is supplied by the caller for a specific use.

These four pillars aren't independent trivia — they're the vocabulary the rest of this book
uses without re-explaining. The next chapter drops down a level, into the primitive and string
types that sit underneath the object model we've just built.

---

## Chapter 6: Primitives, Wrappers, and Strings

### Java is not 100% object-oriented, on purpose

Java has eight **primitive types** (`int`, `char`, `boolean`, `double`, and so on) that are
*not* objects: they have a fixed size, live directly on the stack (or inline within an object's
layout on the heap when they're fields), and are never `null` — they always carry a concrete
default value (`0`, `false`, `0.0`, and so on). This is why Java is sometimes described as not
being a "100% object-oriented" language: in a fully object-oriented language, everything is an
object. Java's designers accepted this deliberately, because primitives are dramatically cheaper
— less memory, no allocation overhead, no indirection through a reference — and this mix of
primitive and object types also makes Java easier to interoperate with non-object-oriented
systems and APIs. Everything about **wrapper classes**, described next, exists to bridge the
gap this design choice creates.

### Wrapper classes and autoboxing

Every primitive type has a corresponding **wrapper class** — `Integer` for `int`, `Double` for
`double`, and so on — that packages the primitive value inside a real object. Wrapper classes
are `final` and immutable, and provide useful static utility methods (`Integer.valueOf()`,
`Integer.parseInt()`). Critically, they exist because **Java's collections can only hold
objects, not primitives** — you cannot create a `List<int>`, only a `List<Integer>`, because
generics (Chapter 9) are themselves built entirely around reference types.

**Autoboxing** is the compiler automatically converting a primitive to its wrapper where an
object is expected; **unboxing** is the reverse. This convenience hides two genuinely sharp
edges worth knowing before they bite you in production:

1. **Integer caching and `==`.** The JVM caches `Integer` instances for values in the range
   -128 to 127 as a performance optimization. Comparing two boxed `Integer`s with `==` inside
   that range happens to work, because both references point at the same cached instance — but
   outside that range, `==` compares object *references*, not values, and two separately-boxed
   `Integer`s holding the same number will compare `false`, even though `.equals()` on the same
   pair would correctly return `true`. This is a real, recurring source of bugs precisely
   because the code "works" during testing with small numbers and breaks in production with
   larger ones. The rule is simple and absolute: **never use `==` to compare boxed types; always
   use `.equals()`.**
2. **`NullPointerException` from unboxing `null`.** If a wrapper reference is `null` and the
   compiler needs to unbox it into a primitive context — assigning it to an `int`, or using it
   in arithmetic — you get a `NullPointerException` at the unboxing point, which can be
   genuinely confusing to trace back to its source if the boxing/unboxing is implicit and not
   visible at the crash site.

### `==` versus `.equals()`, generalized

The Integer-caching trap above is really a specific instance of a much more general rule that
applies to *every* object type, not just wrappers: **`==` compares references** (are these two
variables pointing at the exact same object in memory, or for primitives, are the raw values
identical), while **`.equals()` compares logical content** as defined by the class's own
`equals()` implementation. Two separate `Employee` objects representing the same employee should
be `.equals()` but will never be `==`, because they're distinct objects occupying distinct
memory even if every field matches. Chapter 8 covers what happens when a class's `equals()` is
overridden without also overriding `hashCode()` — spoiler: hash-based collections silently
break — because the two methods form a contract, not two independent choices.

### Strings: immutability and the string pool

`String` is one of the most-used types in Java, and almost everything unusual about it flows
from one design decision: **`String` is immutable**. Once created, a `String`'s contents can
never change — every operation that looks like it modifies a string (`.toUpperCase()`,
`.substring()`, `.concat()`, and so on) actually returns a brand-new `String` object, leaving
the original untouched. Internally, a `String` object wraps a character array (or, since Java 9's
"compact strings" optimization, a byte array plus a coder flag distinguishing Latin-1 from
UTF-16 content) holding the string's contents.

Immutability was chosen deliberately, for several compounding reasons:

- **Security** — strings are used pervasively for things like file paths, network hosts, and
  class names; if a string could be mutated after being validated, that would open a class of
  time-of-check-to-time-of-use vulnerabilities.
- **Safe caching / the string pool** — because a `String` can never change, the JVM can safely
  let multiple variables share the exact same underlying object. String **literals** are stored
  in a special heap region called the **string pool**: whenever the JVM encounters a new string
  literal, it checks the pool first, and reuses the existing object if an identical literal is
  already there, rather than allocating a new one. This is a real memory optimization, and it's
  the reason `String a = "hi"; String b = "hi";` gives you `a == b`, `true` — both variables
  point at the same pooled object — while `String c = new String("hi");` explicitly bypasses the
  pool and allocates a fresh heap object, so `a == c` is `false` even though `a.equals(c)` is
  `true`. This is one of the sharpest, most classic `==` vs `.equals()` gotchas in the entire
  language, and it exists purely because of the pool's optimization strategy.
- **Thread safety by default** — an object that can never change state needs no synchronization
  to be safely shared across threads, since there's no mutation for concurrent access to race
  on. (This is a specific instance of a much more general point about immutability and
  concurrency that we'll return to properly in Chapter 11.)
- **Safe use as a hash key** — because a `String`'s content (and therefore its `hashCode()`)
  can never change after construction, it's an ideal `HashMap` key: its hash-bucket location
  will never become stale the way a mutable object's could (see Chapter 8 and Chapter 3's leak
  discussion). Java even caches a `String`'s computed hash code after first use, since it can
  never need recomputing.

The pool isn't free, though — it trades a lookup cost for the memory savings, and that lookup
becomes relatively wasteful when a program generates huge numbers of genuinely unique strings
that will never be reused, since the pool then does real work checking for a match that's
essentially guaranteed not to exist.

### `StringBuilder` and `StringBuffer`

Because `String` is immutable, code that builds up a string through many small modifications —
appending inside a loop, for instance — would otherwise allocate a new `String` object at every
step, which is wasteful. `StringBuilder` and `StringBuffer` exist as genuinely *mutable*
alternatives with an internal resizable character buffer, so repeated appends modify the same
underlying storage instead of allocating anew each time.

The only difference between the two is thread-safety: `StringBuffer`'s methods are
`synchronized`, making it safe to share across threads at the cost of that synchronization
overhead; `StringBuilder` is not synchronized, making it faster in the overwhelmingly common
case of single-threaded use. Use `StringBuilder` by default; reach for `StringBuffer`
specifically when multiple threads genuinely need to build up the same string concurrently — a
shared log-entry buffer being written to by several worker threads is the textbook case.

With primitives, boxing, and strings covered, we've filled in the last of the "everyday value
types" gap. The next chapter turns to what happens when things go wrong — Java's exception
model — before we move into the collections framework that ties objects, generics, and
`equals()`/`hashCode()` together.

---

## Chapter 7: Exceptions — When Things Go Wrong

### The `Throwable` hierarchy

Every throwable in Java descends from `Throwable`, which splits into two conceptually different
branches:

- **`Error`** — represents serious, generally *unrecoverable* problems, usually at the level of
  the JVM or the environment itself: `OutOfMemoryError`, `StackOverflowError`. Application code
  is not expected to catch and "handle" these in any meaningful sense, because the underlying
  condition (memory exhaustion, a runaway recursive call) usually means the process is no
  longer in a trustworthy state to continue.
- **`Exception`** — represents conditions a well-written program can reasonably anticipate and
  recover from. This branch further splits into **checked** and **unchecked** exceptions, and
  that split is where most of the design texture lives.

**Checked exceptions** (subclasses of `Exception` other than `RuntimeException`) must be either
caught or declared in a method's `throws` clause — the compiler enforces this. This is a
deliberate design choice to *force* callers to acknowledge a failure mode at compile time,
reserved for situations the API author considers genuinely expected and worth guaranteeing
isn't silently ignored — reading a file that might not exist (`IOException`) or querying a
database that might be unreachable (`SQLException`) are the textbook cases.

**Unchecked exceptions** (`RuntimeException` and its subclasses, like
`NullPointerException` and `IllegalArgumentException`) require no such declaration or handling
— they can be thrown and propagate freely without the compiler forcing any acknowledgment.
These typically represent programming errors rather than expected environmental failures: a
`NullPointerException` usually means a bug, not a condition the caller was supposed to plan
around.

Custom exceptions are worth reaching for when built-in types don't communicate business
context clearly enough — throwing `InsufficientFundsException` in a banking application makes
the failure self-documenting and easy to catch specifically, versus a generic
`IllegalArgumentException` that could mean almost anything at the call site.

### `try`, `catch`, `finally`, and what actually happens

The three blocks have distinct jobs: `try` wraps code that might fail; `catch` handles a
specific failure; `finally` runs regardless of whether an exception occurred, and is the
classic place for cleanup — closing streams, releasing connections. A few of `finally`'s
behaviors surprise people who haven't seen them written down:

- **`finally` still runs even if `try` or `catch` contains a `return` statement** — the return
  value is computed, but `finally` executes before control actually leaves the method, ensuring
  cleanup always happens even on the "successful" exit path.
- **`try` with `finally` and no `catch` at all is legal** — useful when you want to guarantee
  cleanup while still letting the exception propagate up to a caller better positioned to handle
  it.
- **`finally` will *not* run if the JVM exits via `System.exit()`** during `try` or `catch` —
  this is the one real escape hatch, since `System.exit()` terminates the JVM immediately rather
  than unwinding normally.
- **A genuinely sharp gotcha:** if the `finally` block itself throws a *new* exception, that new
  exception silently replaces whatever exception was originally thrown in `try` — the original
  is lost unless it's deliberately chained. This is exactly the problem
  **try-with-resources** (Java 7+) was introduced to solve: when a resource implementing
  `AutoCloseable` is closed automatically at the end of a try-with-resources block and that
  `close()` call itself throws, the *original* exception is preserved as the primary one, with
  the close-time exception attached via `Throwable.addSuppressed()` rather than replacing it
  outright. Prefer try-with-resources over manual `finally`-based cleanup whenever the resource
  supports `AutoCloseable`.
- Each `try` can have **only one** `finally` block — multiple `finally` blocks on a single
  `try`/`catch` structure aren't legal syntax at all.
- **Multi-catch** lets one `catch` clause handle several exception types with shared logic:
  `catch (IOException | SQLException e) { ... }`.

### Performance and design notes

`try`/`catch`/`finally` carries some overhead from exception management, but it's generally
minimal unless exceptions are actually being *thrown* frequently — throwing captures a stack
trace, which is comparatively expensive, so exceptions should model genuinely exceptional
conditions, not routine control flow. **Catching `Throwable`** is specifically bad practice,
worth calling out on its own: because `Throwable` is the superclass of *both* `Exception` and
`Error`, catching it also catches things like `OutOfMemoryError` and `StackOverflowError` —
serious, usually unrecoverable JVM-level conditions that a normal application has no business
trying to "handle" and continue past. Doing so can mask a program that's already in a corrupted
or unstable state, letting it limp along in a way that's worse than simply terminating.

Exception handling closes out the foundational language chapters. From here, we move into the
data structures Java gives you to actually organize and process information at scale — starting
with the Collections Framework, which is where several of the concepts from this and the
previous chapters (`equals()`/`hashCode()`, generics, immutability) all converge at once.

---

