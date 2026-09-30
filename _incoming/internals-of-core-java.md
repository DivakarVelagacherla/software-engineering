# Internals of Core Java

*How the language actually works, from bytecode to garbage collection*

---

## How to read this book

This is not a list of interview questions. It's an attempt to explain Java as one connected
system rather than a pile of independent facts. Every topic in Java touches several others:
you cannot really understand `HashMap` without understanding `hashCode()`/`equals()`, and you
cannot understand those without understanding object identity, which in turn depends on how
the JVM lays objects out in memory. So instead of organizing this book around interview
questions ("What is polymorphism?"), it's organized the way the concepts actually depend on
each other — starting from the machine that runs your code, moving up through the language's
object model, then into the data structures, concurrency, and design tools built on top of it.

Read it in order the first time. After that, use it as a reference — each chapter stands
on its own, but points backward when it's relying on something explained earlier.

---

## Table of Contents

**Part I — The Machine Underneath**
1. The JVM: What Actually Runs Your Code
2. Memory: The Heap, the Stack, and Where Objects Live
3. Garbage Collection: Automatic Memory Management

**Part II — The Language**

4. The Object Model: Classes, Objects, and Their Building Blocks
5. The Four Pillars of OOP
6. Primitives, Wrappers, and Strings
7. Exceptions: When Things Go Wrong

**Part III — Data**

8. The Collections Framework
9. Generics and Type Safety

**Part IV — Functional Java**

10. Lambdas, Streams, and the Java 8 Turn

**Part V — Concurrency**

11. Threads, Locks, and the Java Memory Model
12. The Executor Framework and Modern Concurrency

**Part VI — Reflection and Wire Formats**

13. Serialization
14. Reflection and Dynamic Behavior

**Part VII — Design**

15. Design Patterns and SOLID Principles

**Part VIII — Where the Language Is Going**

16. Modern Java: Records, Sealed Classes, and the Module System

**Part IX — Keeping It Running**

17. Debugging and Performance in Production

---

# Part I — The Machine Underneath

## Chapter 1: The JVM — What Actually Runs Your Code

Every other chapter in this book is downstream of one fact: Java code does not run directly
on your processor. It runs on the **Java Virtual Machine**, a program that itself runs on your
processor and interprets (or compiles) Java's intermediate representation — **bytecode** — into
whatever instructions your actual hardware understands. This one design decision is the root
cause of almost everything distinctive about Java: its portability, its automatic memory
management, its runtime type information, and even many of its performance characteristics.

### JDK, JRE, and JVM

These three acronyms describe three nested layers of the same system, and confusing them is
one of the most common early mistakes:

- The **JVM** (Java Virtual Machine) is the engine. It loads class files, verifies them, and
  executes their bytecode — either by interpreting it instruction-by-instruction or by
  compiling hot paths to native machine code (more on this below). The JVM is what makes Java
  "write once, run anywhere": the same `.class` file runs unmodified on any platform that has
  a JVM implementation.
- The **JRE** (Java Runtime Environment) is the JVM plus the standard library — the classes
  in `java.lang`, `java.util`, `java.io`, and so on that every Java program depends on. If you
  only want to *run* Java programs, this is all you need.
- The **JDK** (Java Development Kit) is the JRE plus the tools needed to *write* Java
  programs: the compiler (`javac`), the debugger, `jar`, `javadoc`, and since Java 11, the JRE
  is no longer distributed separately at all — the JDK is the only thing Oracle and OpenJDK
  ship.

You cannot have a JDK without a JRE — the JRE's classes are exactly what the compiler and
tools are built on top of. A useful modern wrinkle: since Java 9, the `jlink` tool lets you
build a **custom minimal runtime image** containing only the modules your application actually
needs, rather than shipping the full JRE. This matters for containerized deployments where
image size and startup time count — we'll come back to this when we discuss the module system
in Chapter 16.

### The three components of the JVM

Structurally, the JVM breaks into three cooperating pieces:

**The ClassLoader** is responsible for finding `.class` files and loading their bytecode into
the JVM. Loading is *lazy* — a class is not loaded until it's first referenced at runtime, not
at program startup. This is why you can have hundreds of classes in a large application without
paying the cost of loading all of them if a given run only exercises a subset of code paths.

Class loading itself happens through a **hierarchy of loaders**, each delegating upward before
trying to load a class itself:

- **Bootstrap ClassLoader** — loads the core `java.lang.*` and other foundational JDK classes.
  Written in native code, has no parent.
- **Platform/Extension ClassLoader** — loads platform-specific extension libraries.
- **Application/System ClassLoader** — loads your application's own classes from the classpath.

This delegation model exists for a good reason: it prevents a malicious or accidental
`java.lang.String` class on your classpath from shadowing the real one, because a request to
load `java.lang.String` always gets delegated up to the Bootstrap loader first, which finds its
own trusted copy before the request ever reaches your Application loader.

Frameworks that need to load plugins or generate classes dynamically (Spring, application
servers, IDE plugin systems) use a **custom `URLClassLoader`** pointed at a JAR or directory,
calling `loadClass()` to bring new code into a running JVM without a restart. This is also why
class *unloading* is tricky: a class can only be garbage collected once its ClassLoader itself
becomes unreachable, which means every instance of every class it loaded — and every reference
to the loader — must also be gone. In practice this mostly happens during hot-redeploys in
application servers, where each redeploy gets a fresh ClassLoader.

**The Runtime Data Areas** are where the JVM keeps everything it needs while your program runs
— we'll spend all of Chapter 2 on this, because it's the foundation for understanding memory
in Java.

**The Execution Engine** actually runs the bytecode instructions. This is where interpretation
and JIT compilation happen.

### Bytecode, interpretation, and JIT compilation

When `javac` compiles your `.java` file, it doesn't produce machine code — it produces
bytecode, a compact, platform-neutral instruction set. The JVM's execution engine can run this
bytecode two ways:

1. **Interpretation** — read each bytecode instruction and execute it directly, one at a time.
   Simple, fast to start, but slower per-instruction than native code.
2. **Just-In-Time (JIT) compilation** — the JVM profiles the running program, identifies "hot"
   methods and loops (code executed very frequently), and compiles *those specific paths* to
   native machine code, plus applies optimizations like method inlining. Once compiled, that
   code runs at native speed for the rest of the program's life.

This is why long-running Java processes tend to "warm up" — early execution is interpreted and
comparatively slow, but as the JIT compiler identifies and compiles hot paths, throughput
climbs. It also explains a genuinely practical tradeoff: for short-lived processes (a quick CLI
tool, some serverless invocations), the JIT's compilation overhead may never be paid back by
the runtime speedup it provides, which is one of the few legitimate reasons to consider running
with the interpreter only (`-Xint`) or tuning down the JIT's aggressiveness. For any normal
long-running server application, though, JIT compilation is almost always a net win, and this
tradeoff should be treated as a niche exception, not a rule of thumb.

`Class.forName()` vs `ClassLoader.loadClass()` is a related, smaller distinction worth
knowing: `Class.forName()` loads *and initializes* a class immediately — running its static
initializers — while `loadClass()` only loads the class definition, deferring initialization
until the class is actually used. Historically, JDBC driver registration relied on this
distinction, since drivers register themselves in a static block that needs to run eagerly.

With this picture of the engine in place, we can now talk about where it actually keeps your
data — which is the subject of the next chapter, and the foundation for understanding garbage
collection, object identity, and a large fraction of Java's performance characteristics.

---

## Chapter 2: Memory — The Heap, the Stack, and Where Objects Live

The JVM divides its runtime memory into distinct regions, each with a different lifecycle,
different performance characteristics, and a different relationship to garbage collection.
Understanding this layout is the prerequisite for understanding almost every subtle Java bug:
why `==` sometimes surprises you, why a `static` field can leak memory for the life of your
process, and why the stack overflows on deep recursion but the heap "just" runs out of memory
gradually.

### Heap, Stack, Method Area, and Native Stacks

- **Heap** — where all objects and their instance data live, shared across the entire
  application (and across all threads). This is the region garbage collection manages, and by
  far the largest and most performance-sensitive.
- **Stack** — each thread has its own stack, used to track method calls: local variables,
  method parameters, and the call chain itself, organized as a Last-In-First-Out structure.
  Because it's LIFO and per-thread (no sharing, no synchronization needed), the stack is much
  faster to allocate on and read from than the heap, which has to deal with a much more complex
  allocation and reclamation story. This speed difference is also why deep, unbounded recursion
  throws `StackOverflowError` — each nested call consumes another stack frame, and the stack's
  size is fixed and comparatively small.
- **Method Area / Metaspace** — stores class-level metadata: the bytecode for methods, the
  constant pool, and `static` fields. Before Java 8 this lived in a fixed-size region called
  **PermGen** (Permanent Generation), which was actually part of the heap and had a hard size
  ceiling — a classic source of `OutOfMemoryError: PermGen space` under heavy dynamic
  classloading (application servers doing hot redeploys, frameworks generating lots of proxy
  classes, were the usual victims). Java 8 replaced PermGen with **Metaspace**, which lives in
  *native* memory outside the JVM heap and grows dynamically by default — removing that
  specific fixed-ceiling failure mode, though a genuine classloader leak can still exhaust
  native memory over time.
- **Native Method Stacks** — support calls into native (non-Java) code via JNI; not something
  application code usually needs to think about directly.

### `static` and the Method Area

The `static` keyword is really a memory-location decision as much as an access-control one:
a `static` field or method belongs to the *class*, not to any instance, and is physically
stored once in the Method Area, created when the class is loaded and persisting as long as the
class stays loaded — shared by every instance of that class. This is precisely why static
members can be accessed without creating an object, and why every instance sees the same value
for a static field.

It also explains a family of related facts that otherwise look like arbitrary rules:

- **Static methods cannot be overridden** — overriding depends on dynamic dispatch at runtime
  based on the *object's* actual type, but static methods are resolved at compile time and
  belong to the class, not an object, so there's no instance to dispatch on. (A subclass *can*
  declare a static method with the same signature, but this is "hiding," not overriding — the
  method called depends on the *reference type* at compile time, not the object's runtime
  type.)
- **Static methods cannot access non-static members** — there's no implicit instance (`this`)
  available in a static context to look those members up on.
- **`this` and `super` cannot be used in a static context** — same reason: both refer to an
  instance, and static code doesn't have one.
- **A static initializer block runs exactly once**, when the class is first loaded — this is
  the mechanism for one-time setup of static state. If it throws an exception, the class fails
  to initialize, wrapped in `ExceptionInInitializerError`; any later attempt to use that class
  throws `NoClassDefFoundError`, because the JVM refuses to use a class whose initialization
  previously failed. This is a subtle but important cause-and-effect chain: a `NoClassDefFoundError`
  in production sometimes traces back to a static initializer that threw on the *first* attempt
  to use the class, long before the error you're actually looking at.

### `final` and where it fits

`final` is a promise about *reassignment*, not necessarily about deep immutability, and this
distinction trips people up constantly. A `final` variable can't be reassigned once initialized
— for primitives, that means the value is fixed. For object references, it means the
*reference* can't be pointed at a different object, but the object it points to can still be
freely mutated if it's otherwise mutable. `final List<String> names = new ArrayList<>();` lets
you keep calling `names.add(...)` forever; what you can't do is write `names = otherList;`.

`final` on a method prevents overriding; `final` on a class prevents subclassing entirely.
Combined, `final class` + `final` methods is the standard recipe for a class whose behavior you
want to *guarantee* stays fixed regardless of what anyone does with it later — utility classes,
security-sensitive types, and (as we'll see in Chapter 4) immutable value classes all lean on
this. `final` is also required for effectively-capturing local variables inside lambdas and
anonymous inner classes, which we'll return to in Chapter 10.

One caveat worth flagging early because it resurfaces in Chapter 14: `final` is a *compiler and
JVM contract*, not an unbreakable law. Reflection can bypass it (`setAccessible(true)` +
`Field.set()`), and the change may even appear to "work" — but because the JIT compiler is
allowed to optimize under the assumption that a `final` field never changes, the results can be
inconsistent depending on when and where the field is read. Treat this as "technically
possible, not actually safe," not as a real escape hatch.

With memory geography established, we're ready for the part of the JVM that makes manual
memory management unnecessary in Java: the garbage collector.

---

## Chapter 3: Garbage Collection — Automatic Memory Management

Garbage collection is the JVM's answer to a problem every systems language has to solve: when
is it safe to reclaim the memory an object occupies? Java's answer is automatic and
reachability-based, and understanding *how* it decides reachability — and how it's evolved
across JVM versions — explains both why Java programs rarely segfault and why they can still,
very much, leak memory.

### Reachability, not reference counting

The garbage collector's core algorithm is to trace outward from a set of **GC roots** —
local variables on live thread stacks, static fields, JNI references — following every object
reference reachable from those roots. Anything not reached by this trace is garbage, regardless
of how many references objects hold *to each other*. This is the crucial difference from a
naive reference-counting collector: two objects that reference each other in a cycle, but that
nothing else in the live program can reach, are still correctly identified as garbage and
collected together. Reference-counting garbage collectors need a separate cycle detector to
handle this case; Java's tracing collector handles it for free, as a direct consequence of how
reachability is defined.

### The generational hypothesis

Most JVM garbage collectors are **generational**, built on an empirical observation called the
*weak generational hypothesis*: most objects die young. A huge fraction of objects allocated in
a typical program are short-lived — temporary variables, intermediate stream results, per-request
scratch objects — while a much smaller fraction survive for a long time (caches, connection
pools, application state).

The heap is split accordingly:

- The **Young Generation** holds newly created objects. It's collected frequently, and because
  most objects there die almost immediately, these collections are fast and cheap.
- The **Old (Tenured) Generation** holds objects that have survived several young-generation
  collection cycles — meaning they've been "promoted" because they proved they're not
  short-lived. Old-generation collections are less frequent but more expensive, since there's
  more live data to trace and the collector often needs to compact memory to avoid
  fragmentation.

This two-tier structure is *why* generational collection is efficient: it concentrates effort
on the region where most garbage actually accumulates, and does much less frequent, more
expensive work on the region where objects tend to survive.

### The collectors themselves

The JVM has shipped several garbage collector implementations over the years, each tuned for
a different point on the throughput/latency tradeoff curve:

| Collector | Character | Era |
|---|---|---|
| Serial GC | Single-threaded, simple, stop-the-world | small/simple apps |
| Parallel GC | Multi-threaded stop-the-world, throughput-optimized | default in Java 8 |
| CMS (Concurrent Mark-Sweep) | Mostly concurrent, lower pause times | largely superseded by G1 |
| G1 GC | Region-based, balances pause time and throughput | default from Java 9+ |
| ZGC | Region-based, sub-millisecond pauses even on huge heaps | opt-in, low-latency workloads |
| Shenandoah GC | Similar low-latency goals to ZGC, different algorithm | opt-in, low-latency workloads |

The throughline across this list is the industry's steady push toward reducing **stop-the-world
pause times** — the periods where the JVM must freeze all application threads to safely trace
and reclaim memory. Serial and Parallel GC accept longer pauses in exchange for simplicity and
raw throughput; G1 tries to bound pause times while still delivering good throughput; ZGC and
Shenandoah push pause times down to the sub-millisecond range even on multi-gigabyte heaps, at
the cost of more background bookkeeping work.

### `finalize()`, and why you shouldn't reach for it

Historically, objects could define a `finalize()` method that the garbage collector calls
before reclaiming them, intended as a last chance to release resources. In practice this
mechanism turned out to be unreliable: `finalize()` may never run at all if the garbage
collector doesn't get around to that object before the JVM exits, its timing is unpredictable,
and it adds overhead to collection. Modern Java code should not depend on it — use
**try-with-resources** and `AutoCloseable` for deterministic cleanup instead (see Chapter 7).
`finalize()` is worth knowing about because it still appears in `Object`'s method list and in
older codebases, but treat it as legacy.

### The reference-strength spectrum

Beyond ordinary ("strong") references, Java exposes three progressively weaker reference types
in `java.lang.ref`, each changing how eagerly the collector is allowed to reclaim the object:

- **Strong reference** — the default. As long as any strong reference to an object exists, it
  is *not* eligible for collection, period.
- **Soft reference** (`SoftReference`) — eligible for collection, but the collector will only
  actually reclaim it under *memory pressure*, typically right before it would otherwise throw
  `OutOfMemoryError`. Ideal for memory-sensitive caches: hold data as long as there's room to
  spare, but don't let it cause an OOM.
- **Weak reference** (`WeakReference`) — eligible for collection as soon as no strong
  references exist, regardless of memory pressure. Used by `WeakHashMap`, and in general
  anywhere you want a reference that doesn't artificially keep an object alive — the classic
  case is caches keyed by objects whose lifecycle you don't control.
- **Phantom reference** (`PhantomReference`) — enqueued only *after* the object has already
  been finalized and its memory is about to be reclaimed; `get()` on a phantom reference always
  returns `null`. Used for precise post-finalization cleanup scheduling — a low-level tool, most
  visible in things like tracking the release of off-heap `ByteBuffer` native memory.

This spectrum — Strong > Soft > Weak > Phantom — is really a spectrum of "how badly does the
collector need to reclaim memory before it's willing to break this reference," and it's the
right mental model for reasoning about cache design in Java: a plain `HashMap` cache holds
strong references and will never shrink on its own; a cache built on `SoftReference` values
will shrink automatically as memory gets tight.

### Memory leaks in a garbage-collected language

Java programs absolutely can leak memory, despite automatic garbage collection — because a
"leak" here just means an object is unintentionally still *reachable* from a GC root, even
though the program logically no longer needs it. Reachability is a fact about references, not
about intent, and the collector has no way to know your intent. The usual culprits:

- **Static collections that grow forever** — a `static` `List` or `Map` used as an ad-hoc cache
  with nothing ever removed keeps every entry reachable for the life of the class (which, per
  the earlier discussion of the Method Area, is usually the life of the whole application).
- **Listeners/callbacks that are registered but never deregistered** — the subject holds a
  reference to the observer, keeping it alive long after the code that registered it cares.
- **Unclosed resources** — streams, connections, and similar objects can hold native memory or
  external handles that aren't released just because the Java object itself becomes
  unreachable; this is a different kind of leak (resource, not heap), but shows up the same way
  in symptoms.
- **`ThreadLocal` values in pooled-thread environments** — a thread pool reuses its threads
  indefinitely, so a `ThreadLocal` value set on a pooled thread and never cleared stays
  reachable for as long as that thread lives, which in a pool is effectively forever.

The mismatch between a mutable object's `hashCode()` and its position in a `HashMap` bucket
(covered in full in Chapter 8) is worth a preview here too: if a key object's hash-relevant
state changes after it's inserted, the entry becomes permanently unreachable *at its original
bucket* — the object is technically still referenced by the map internally, so it's not garbage
collected, but you can never look it up again. That's a leak hiding inside a data structure
that's technically doing nothing wrong.

### Diagnosing `OutOfMemoryError` — a practical workflow

When a real application throws `OutOfMemoryError`, the investigation follows a fairly standard
path, worth internalizing as a checklist:

1. **Check heap sizing** — `-Xms` (initial heap size) and `-Xmx` (maximum heap size) JVM flags.
   Sometimes the "leak" is just an undersized heap for legitimate load.
2. **Capture a heap dump** — either on suspicion (`jmap`) or automatically at the moment of
   failure (`-XX:+HeapDumpOnOutOfMemoryError`).
3. **Analyze the dump** with a tool like Eclipse Memory Analyzer (MAT) or VisualVM, looking for
   *dominator* objects — objects retaining unexpectedly large amounts of memory — and counts of
   instances that are far higher than they should logically be.
4. **Trace the GC-root reference chain** keeping the suspect objects alive, to find the actual
   retaining reference in your code.
5. **Fix the retaining reference** — remove it, scope it more narrowly, switch to a
   `WeakReference`, or add proper cleanup (deregistering listeners, closing resources, evicting
   caches).
6. **Verify** with before/after profiling.

We'll return to this workflow with more tooling detail in Chapter 17. For now, the important
takeaway is that garbage collection removes the need to manually free memory, but it does not
remove the need to think about object lifetime and reachability — it just changes *how* you
reason about it, from "did I remember to call `free()`" to "is anything still holding a
reference to this that it shouldn't be."

With the machine layer covered — how code runs, where data lives, and how it's reclaimed — we
can move to the language itself: the object model Java gives you to build programs on top of
that machine.

---

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

# Part III — Data

## Chapter 8: The Collections Framework

### The shape of the framework

The Collections Framework is a unified set of interfaces, implementations, and algorithms for
storing and manipulating groups of objects. Its core interfaces — `Collection`, `List`, `Set`,
`Queue`, and `Map` (technically outside the `Collection` hierarchy, but universally considered
part of the framework) — each express a different contract about ordering and duplicates:
`List` is ordered and permits duplicates; `Set` guarantees uniqueness; `Queue` models
FIFO/priority processing order; `Map` stores key-value pairs. Choosing among them is really
choosing which of those guarantees you need. Every `Collection` shares a baseline of common
operations — `add`, `remove`, `clear`, `size`, `isEmpty` — plus support for **iteration**: an
`Iterator` walks a collection's elements one at a time, and `ListIterator` extends that with
bidirectional traversal and in-place modification, available only for `List`s specifically.

### `ArrayList` vs. `LinkedList` vs. `HashSet`: choosing by access pattern

These three types are the workhorses of everyday Java, and the right choice is entirely a
question of what operation you do most:

- **`ArrayList`** is backed by a resizable array. This gives O(1) indexed access, but insertion
  or removal in the middle requires shifting every subsequent element, and growing past the
  current capacity means allocating a new (typically ~1.5x larger) array and copying everything
  over. Choose it when you need frequent random access via index and don't often insert/remove
  from the middle. If you know a list will be repeatedly cleared and reused at some predictable
  size, pre-sizing its initial capacity avoids the repeated resize-and-copy cost.
- **`LinkedList`** is a doubly-linked list. Insertion and removal at either end (or at a node
  you already hold) is O(1), but indexed access is O(n) because it must walk the list from an
  end. Choose it when you're frequently adding/removing from the beginning or middle — building
  a queue or a stack, for instance.
- **`HashSet`** guarantees no duplicates and gives average O(1) lookup, insertion, and deletion,
  at the cost of no ordering guarantee at all. It's ideal for membership-testing scenarios: "is
  this item already in the set."

If you need `HashSet`'s uniqueness *and* insertion-order iteration, `LinkedHashSet` gives you
both, at the same O(1) average performance, by threading a doubly-linked list through the hash
buckets to remember insertion order. If you need elements kept continuously **sorted**, reach
for `TreeSet` (or `TreeMap` for key-value pairs) instead, backed by a self-balancing
**Red-Black tree**, giving O(log n) insert/delete/lookup in exchange for always-sorted
iteration. A concrete decision point: a high-frequency trading application that needs prices
continuously sorted for fast access to the best price is a good `TreeMap`/`TreeSet` use case,
since re-sorting an `ArrayList` on every update would be far more expensive than maintaining
sorted order incrementally. `TreeSet`/`TreeMap` require their elements to either implement
`Comparable` or be given an explicit `Comparator` at construction time — without one, insertion
throws `ClassCastException` at *runtime*, since generics don't (and can't, per Chapter 9) verify
this at compile time.

### `hashCode()` and `equals()`: the contract that makes hashing work

This is the single most consequential pairing in the whole collections story, so it's worth
walking through carefully. A `HashMap` (and `HashSet`, which is literally implemented as a
`HashMap` under the hood, storing elements as keys against a dummy constant value) stores
entries in an array of **buckets**. `hashCode()` determines *which bucket* a key belongs in;
`equals()` determines, *within that bucket*, whether a given key actually matches an existing
entry (necessary because two different keys can legitimately land in the same bucket — a
**hash collision**).

This means `hashCode()` and `equals()` are not two independent choices — they form a single
contract: **objects that are `.equals()` to each other must produce the same `hashCode()`.**
If you override `equals()` without also overriding `hashCode()` (the default `Object.hashCode()`
is based on identity, unrelated to your custom equality logic), you break this contract:
two objects your code considers logically equal can end up with different hash codes, land in
different buckets, and the `HashMap` will simply fail to find an entry that's actually there —
silently, with no error, just wrong-looking behavior that's maddening to debug. **Always
override both together, or neither.**

There's a second, related trap that compounds this: **never use a mutable object as a
`HashMap` key** (or `HashSet` element) if any field that participates in `hashCode()` can
change after insertion. If it does change, the object's hash code changes, but the map has
already placed it in a bucket based on the *old* hash code — so a lookup with the mutated key
computes a *different* bucket and the entry is never found again, even though it's still
sitting in the map, unreachable and un-removable through normal means. This is precisely the
kind of memory leak described in Chapter 3: the object is technically referenced, so it's not
garbage, but it's permanently orphaned from any code path that could find or clean it up.
Prefer immutable keys — which is exactly why `String` (immutable, Chapter 6) is such a common
and safe choice for map keys.

### `HashMap` internals, across Java versions

Concretely: a `HashMap` is an array of buckets; a hash function maps each key to a bucket index.
When two keys collide into the same bucket, prior to Java 8 the JVM handled that by chaining
them in a simple linked list within the bucket, giving O(n) worst-case lookup within a
pathologically collision-heavy bucket. **Since Java 8**, if a single bucket's chain grows past a
threshold (8 entries, with the overall table sized at least 64 buckets), that bucket's linked
list is converted into a balanced **red-black tree**, dropping worst-case lookup within that
bucket to O(log n). This treeification is purely an internal optimization against pathological
collision patterns (including adversarial ones) — it doesn't change the *average* case, which
remains O(1) for insertion, deletion, and lookup under normal, well-distributed hashing;
`TreeMap`/`TreeSet`, by contrast, are *always* O(log n), because they're always a tree,
unconditionally sorted.

`ConcurrentHashMap` is the thread-safe sibling: safe for concurrent access without external
locking, and much better under contention than simply wrapping a `HashMap` with
`Collections.synchronizedMap()` (which serializes *every* access behind one lock, effectively
making it single-threaded under contention) or the ancient `Hashtable` (which does the same).
Historically (pre-Java 8), `ConcurrentHashMap` achieved this through **segment-based lock
striping** — dividing the map into a fixed number of independently-lockable segments. Since
Java 8, it uses finer-grained **per-bin locking** (CAS operations for the common case, with
`synchronized` only on the specific bin actually being modified), which scales substantially
better under heavy concurrent access. We'll return to concurrent collections more broadly in
Chapter 11.

### Sorting: `Comparable`, `Comparator`, and the algorithms underneath

`Comparable` defines a type's single **natural ordering**, implemented by the class itself via
`compareTo()` — appropriate when there's genuinely one obvious way to order instances of a
type. `Comparator` defines an **external, custom ordering**, implemented separately from the
class — appropriate when you need multiple different orderings of the same type, or when you
can't modify the class to implement `Comparable` at all. `Collections.sort()` (which sorts a
`List` in place, mutating the original) and `Arrays.sort()` (for arrays) both work without an
explicit `Comparator` *only if* the elements implement `Comparable` — otherwise, sorting throws
`ClassCastException` at runtime. Sorting a list containing `null` elements always throws
`NullPointerException`, since `null` has no defined comparison order.

Under the hood, `Collections.sort()` and `Arrays.sort()` for object arrays both use **TimSort**,
a hybrid, *stable* (preserves the relative order of equal elements) sorting algorithm combining
merge sort and insertion sort, specifically tuned to perform very well on data that's already
partially sorted — a common real-world case. `Arrays.sort()` for *primitive* arrays instead uses
a Dual-Pivot Quicksort, which doesn't need to preserve stability (primitives have no distinct
"identity" beyond their value, so stability is meaningless for them) and is faster for that
reason. `Stream.sorted()` (Chapter 10) is functionally similar but returns a *new* sorted
stream rather than mutating the original collection in place — a meaningful difference when
you care about not touching the source data.

`ConcurrentModificationException` is the last piece of practical collections knowledge worth
covering here: it's thrown when a collection's structure is modified (elements added or removed)
while it's being iterated by anything other than the iterator's own `remove()`/`add()` methods —
Java's *fail-fast* iterators detect this via an internal modification-count check. The fix
during single-threaded manual iteration is to use `Iterator.remove()` instead of the
collection's own `remove()`; in genuinely concurrent, multi-threaded scenarios, switch to a
concurrency-safe collection instead — `CopyOnWriteArrayList` (each iterator works off an
immutable snapshot array taken at the moment of iterator creation, so any concurrent
modification during iteration is completely invisible to it) or `ConcurrentHashMap`.

Collections tie together nearly everything from earlier chapters — object identity, equality,
immutability, and now hashing — into the data structures real programs are built from. The next
chapter looks at the mechanism that lets those structures be written once and used safely with
any type: generics.

---

## Chapter 9: Generics and Type Safety

Generics let you parameterize classes and methods over a type — `List<String>` instead of a
raw `List` that could silently hold anything — moving type-mismatch errors from a runtime
`ClassCastException` discovered deep in production to a compile-time error caught the moment
you write the wrong thing. This is the entire value proposition: **type safety** (the compiler
catches invalid usages before the program ever runs) and **reduced duplication** (one generic
`Box<T>` class replaces what would otherwise be a separate near-identical class per type you
needed to box).

### Type erasure

The Java compiler enforces generic type constraints *only at compile time*. After compilation,
generic type information is stripped from the bytecode entirely — a process called **type
erasure**. At runtime, `List<Integer>` and `List<String>` are indistinguishable; both are simply
`List`. This was a deliberate backward-compatibility decision: it let generics be retrofitted
into the JVM and bytecode format in Java 5 without breaking every piece of existing compiled
code that predated generics.

Type erasure has a genuinely useful, concrete consequence worth internalizing: **you cannot
create an array of a generic type** (`new T[]` is illegal). Arrays in Java are covariant and
enforce element-type safety *at runtime* on every store into them — but by the time an array
would need to check "is this a valid `T`," erasure has already thrown that type information
away. Allowing `new T[]` would silently defeat the array's own runtime store-checking, so the
language disallows it outright rather than allow a hole in type safety to slip through.

### Type inference and the diamond operator

Generic **type inference** lets the compiler deduce type arguments from context rather than
requiring you to spell them out everywhere. The clearest everyday example is the **diamond
operator**, introduced in Java 7: instead of writing `List<String> list = new
ArrayList<String>();`, redundantly repeating the type argument, you can write
`List<String> list = new ArrayList<>();` — the compiler infers `ArrayList<String>` from the
declared variable type on the left-hand side. This is a small piece of syntax, but it removes a
huge amount of visual noise from generics-heavy code, and it's now the idiomatic way to write
any collection instantiation.

Generics complete the picture of how Java achieves compile-time safety around the collections
covered in the last chapter. From here, we shift from *how data is stored* to *how it's
processed* — the functional-programming features Java 8 added on top of everything we've
covered so far.

---

# Part IV — Functional Java

## Chapter 10: Lambdas, Streams, and the Java 8 Turn

Java 8 was the largest single shift in the language's history, and nearly everything it added —
lambdas, the Stream API, `Optional`, default/static interface methods — is really one coherent
idea: letting you pass *behavior* around as a value, the way you'd pass an `int` or a `String`,
without the ceremony that behavior required before.

### Functional interfaces

A **functional interface** is an interface with exactly one abstract method (often abbreviated
SAM, for Single Abstract Method) — `Runnable`, `Comparator`, `Callable`. This single-method
constraint is what makes an interface a valid *target type* for a lambda expression: the
lambda's body becomes that one method's implementation. Critically, `default` and `static`
methods on the interface **don't count** toward this limit — an interface with one abstract
method and five default methods is still a valid functional interface, because only the
abstract method needs an implementation supplied by the lambda. A functional interface *can*
extend another interface, but only if that parent interface contributes no additional abstract
methods (only `default`/`static` ones) — otherwise the single-abstract-method guarantee breaks.
The `@FunctionalInterface` annotation isn't required for any of this to work, but it's a useful
compiler-enforced guardrail against accidentally adding a second abstract method later and
silently breaking every lambda written against that interface.

Java ships a standard library of common functional interfaces so you rarely need to declare
your own: `Function<T,R>` (takes a `T`, returns an `R`), `Predicate<T>` (takes a `T`, returns
`boolean` — the backbone of `filter()`), `Consumer<T>` (takes a `T`, returns nothing — the
backbone of `forEach()`), `Supplier<T>` (takes nothing, returns a `T`), and `BiFunction<T,U,R>`
(takes two arguments, returns a result).

### Lambdas vs. anonymous classes

Both let you implement an interface's method inline, but they differ in real ways beyond
brevity:

- A **lambda** has **no scope or identity of its own** — `this` and `super` inside a lambda body
  refer to the *enclosing* instance and class, exactly as if the lambda's code were written
  directly in the surrounding method. Lambdas are structurally lighter weight and, in most JVM
  implementations, don't generate a full named class the way anonymous classes do (they're
  desugared via `invokedynamic` at the bytecode level).
- An **anonymous class** creates a genuine, separate class with its own `this`, capable of
  holding its own instance fields and implementing multiple methods — something a lambda
  structurally cannot do, since it's tied to exactly one method by definition.

Lambdas can only capture local variables from their enclosing scope that are **final or
effectively final** — never reassigned after initialization. Attempting to mutate a captured
local inside a lambda body is a compile-time error, not a runtime surprise. This restriction
exists to guarantee the lambda is state-consistent and safely re-invocable without hidden
side effects on the surrounding method's local state — a direct echo of the immutability-and-
concurrency theme from Chapters 3 and 6.

A few smaller, sharp-edged facts round out the picture: lambdas *can* throw exceptions, but if
the functional interface's method doesn't declare a checked exception, any checked exception
thrown inside the lambda body must be caught and handled (or wrapped as unchecked) right there,
since there's nowhere else for it to be declared. And `synchronized` cannot be used as a bare
block directly inside a lambda body, because a lambda has no intrinsic monitor object of its own
to lock on the way a method does implicitly — if synchronization is genuinely needed inside a
lambda, it has to lock on some explicit external object.

**Method references** (`ClassName::methodName`) are pure syntactic sugar over a lambda that
does nothing but call an existing method — `System.out::println` instead of
`(x) -> System.out.println(x)`. Same functional-interface target typing underneath, just less
visual noise when the lambda body would be a single method call anyway.

### The Stream API

A **Stream** provides a declarative, functional-style pipeline for processing sequences of
elements — filtering, transforming, and reducing — without mutating the underlying source
collection at all. Streams come in two flavors of operation:

- **Intermediate operations** (`filter()`, `map()`, `sorted()`, and others) return *another*
  stream and are **lazy** — nothing actually executes when you call them; they just build up a
  pipeline description.
- **Terminal operations** (`forEach()`, `collect()`, `reduce()`, `count()`, and others) trigger
  the actual traversal and processing of the whole pipeline, producing a concrete result or side
  effect.

This laziness matters practically: it's the reason `Stream.iterate()` and `Stream.generate()`
can produce genuinely **infinite** streams — an infinite sequence generated by repeatedly
applying a function to a seed (`iterate`) or by an independent supplier producing each value with
no relation to the last (`generate`) — since nothing is actually computed until a terminal
operation (typically combined with `limit()`) forces evaluation of a bounded prefix.

`map()` versus `flatMap()` is one of the most commonly confused pairs in the Stream API: `map()`
performs a strict **1:1** transformation — each input element produces exactly one output
element. `flatMap()` handles the case where each input element produces its *own* stream (a
nested `List<List<X>>`, for instance), and flattens all of those individual streams into one
combined stream — genuinely `1:many`, then flattened. Reach for `flatMap()` specifically when
you're dealing with nested collections you want unwrapped into a single flat sequence.

`peek()` is worth a specific caution: it's designed for debugging — inspecting elements as they
pass through the pipeline without transforming them — and should not be relied on to drive real
application logic, because JVM stream implementations are permitted to skip or reorder `peek()`
calls under certain optimizations (particularly around short-circuiting terminal operations).
Treat any side effect inside `peek()` beyond logging as a code smell.

`findFirst()` (returns the first element by encounter order — meaningful and deterministic for
sequential streams) versus `findAny()` (returns *some* element with no order guarantee, but can
short-circuit faster, particularly on parallel streams where "the first one any thread happens
to finish with" is cheaper to compute than "the genuinely first one in sequence").

**`parallelStream()`** splits the underlying data across worker threads in the JVM's default
**common `ForkJoinPool`**, whose size defaults to one less than the number of available
processor cores. This is genuinely useful for CPU-bound bulk processing of large datasets, but
carries two real caveats worth internalizing rather than reaching for reflexively: first, the
splitting-and-merging overhead can make parallel streams *slower* than a plain sequential stream
for small datasets, where the coordination cost dwarfs any parallelism gain; second, because the
common pool is shared across the *entire JVM process*, a long-running or blocking task submitted
to a parallel stream in one part of an application can starve unrelated parallel streams
elsewhere in the same process — a subtle form of resource contention that's easy to miss until
it shows up as mysterious latency somewhere seemingly unrelated.

`Collectors` is the Stream API's utility class for the common "gather results back into
something concrete" step of a pipeline — `Collectors.toList()`, `Collectors.toMap()` (which
throws `IllegalStateException` on encountering a duplicate key unless you supply an explicit
merge function), `Collectors.groupingBy()`, `Collectors.joining()`, and more, all designed to
plug into the `collect()` terminal operation.

### `Optional`: making absence explicit

`Optional<T>` is a container type representing "a value that might not be present," introduced
specifically to address what its designers have called the "billion-dollar mistake" of null
references — the pervasive risk of `NullPointerException` from a method that silently returns
`null` to signal absence, with no way for the caller to know that without reading documentation
or the source. `Optional.of(value)` requires a genuinely non-null value and throws immediately
if given `null` — appropriate when you're certain the value can't be absent, and want to fail
fast if that assumption is ever wrong. `Optional.ofNullable(value)` safely wraps a value that
might legitimately be `null`, producing an empty `Optional` in that case rather than throwing.
Callers then handle absence explicitly and locally via `isPresent()`, `ifPresent()`, or
`orElse()`, instead of implicitly, far away from where the `null` originated.

One design guideline worth stating plainly: `Optional` is meant for **return types**, signaling
to a caller "this might not produce a value" — using it as a *method parameter* is generally
discouraged, since it complicates call sites and method signatures without actually preventing
someone from passing an `Optional` that itself wraps `null`, defeating the purpose while adding
ceremony.

Functional Java is where the collections framework (Chapter 8) and the object model (Chapters
4–5) meet a genuinely different programming style layered on top of the same underlying
language. The next part shifts axis entirely — from *what* your program computes to *how many
things happen at once* while it computes it.

---

# Part V — Concurrency

## Chapter 11: Threads, Locks, and the Java Memory Model

### What a thread is, and how to create one

A **thread** is an independent path of execution within a program — the smallest unit of CPU
scheduling. Multiple threads within one process share the same memory space (notably, the heap
from Chapter 2), which is exactly what makes inter-thread communication cheap and direct, and
exactly what makes uncoordinated concurrent access to shared state dangerous.

You create a thread by either extending `Thread` directly, or — generally the preferred
approach — implementing the `Runnable` interface and passing an instance to a `Thread`
constructor. `Runnable` is just the task definition (a single `run()` method, and it's a valid
functional interface, so lambdas work directly with it); `Thread` is the actual mechanism of
execution. Preferring `Runnable` keeps your task class free to extend some *other* class if it
needs to, since Java only allows single class inheritance — extending `Thread` directly uses up
that one inheritance slot for something that's arguably not really what your class *is*, but
merely how it happens to run.

A `Thread` instance follows a strict lifecycle — **New → Runnable → (Blocked / Waiting / Timed
Waiting) → Terminated** — and can only be `start()`ed **once**. Calling `start()` again on a
`Thread` that's already run throws `IllegalThreadStateException`; to run the same task again,
you construct a fresh `Thread` instance around it.

### Synchronization: `synchronized`, `volatile`, and the two problems they solve

Concurrent access to shared mutable state creates two genuinely distinct problems, and it's
worth being precise about which tool solves which:

- **Atomicity** — whether an operation executes as a single, indivisible unit with no
  interleaving from other threads. Even a simple-looking `counter++` is *not* atomic — it's
  actually read, increment, write, three separate steps, and two threads interleaving those
  steps can lose an update.
- **Visibility** — whether a write to a variable by one thread is guaranteed to be observably
  up-to-date when another thread reads it. Without any synchronization, each thread may work off
  a locally cached copy of a variable and never observe another thread's more recent write.

**`synchronized`** (on a method or an explicit block) solves *both* problems together: it
provides mutual exclusion (only one thread can hold a given monitor lock at a time, giving
atomicity to the guarded section) and it establishes the memory-barrier effects that guarantee
visibility of everything written before the lock was released to whatever thread next acquires
it. Locking an entire *method* is simple but coarse — it locks the whole method for the life of
every call; locking a *block* gives finer control, letting you shrink the critical section to
just the lines that actually touch shared state, reducing contention between threads that don't
actually need to wait on each other. If an exception is thrown inside a `synchronized` block,
the JVM still releases the lock as part of normal exception unwinding — there's no risk of a
permanent deadlock just because an exception happened to occur inside a critical section.

**`volatile`** solves *only* visibility, not atomicity. A `volatile` variable's reads and writes
go directly to main memory, bypassing thread-local caching, so every thread always sees the
latest write — but `volatile` provides **no mutual exclusion**. `volatile int counter; counter++`
is still a race condition, because the read-increment-write sequence isn't atomic just because
the variable itself is always visible. This is the single most commonly conflated distinction in
Java concurrency, worth internalizing precisely: **`volatile` guarantees you'll see the latest
value; it does not guarantee that a compound operation on that value happens as one indivisible
step.** For simple flags and single-write-many-read state, `volatile` alone is often sufficient
and cheaper than full synchronization; for anything involving a read-modify-write sequence, it's
not enough on its own.

For atomic operations without locking at all, the `java.util.concurrent.atomic` package
(`AtomicInteger`, `AtomicLong`, `AtomicReference`) provides lock-free thread safety built on
hardware **compare-and-swap (CAS)** instructions — genuinely faster than `synchronized` under
contention for simple counters, flags, and references, though CAS-based atomics don't help with
invariants spanning multiple variables at once, where a real lock is still the right tool.

### The Java Memory Model

The **Java Memory Model (JMM)** is the formal specification underlying everything in this
section: it defines exactly which behaviors are and aren't guaranteed when multiple threads
read and write shared memory, via the concept of a **happens-before** relationship — a
partial ordering that determines when one thread's write is guaranteed to be visible to
another thread's subsequent read. `synchronized`, `volatile`, and `final` fields each
establish specific happens-before edges; without one of these establishing a connection between
a write and a read, the JVM (and the underlying hardware) is free to reorder, cache, or delay
that write in ways that produce genuinely surprising, non-deterministic behavior — the
**visibility problem** in its most general form. Nearly every "works on my machine, hangs or
misbehaves only in production" concurrency bug in Java traces back to code that assumed
visibility without ever establishing a happens-before relationship to guarantee it.

### `wait()`, `notify()`, and the producer/consumer pattern

`wait()`, `notify()`, and `notifyAll()` (inherited from `Object`, as noted back in Chapter 4)
are the low-level primitives for threads to signal each other about state changes, and they
must be called from within a `synchronized` block on the same monitor object they're
coordinating around. `wait()` releases the held lock and suspends the calling thread until
another thread calls `notify()`/`notifyAll()` on the same object; the classic use case is a
bounded producer/consumer buffer, where a producer `wait()`s while the buffer is full and a
consumer `wait()`s while it's empty, each `notify()`-ing the other after changing the buffer's
state. The wait condition must always be checked in a `while` loop, not a plain `if` — a thread
can wake from `wait()` spuriously, without a genuine corresponding `notify()`, so the condition
needs to be re-verified after waking, not just assumed true because a wakeup happened.

In practice, hand-writing this pattern with raw `wait()`/`notify()` is mostly of historical and
educational value today — `BlockingQueue` implementations in `java.util.concurrent` (covered
next chapter) encapsulate exactly this bounded-buffer coordination correctly and far more safely
than a hand-rolled version, and should be preferred for real production code.

### Deadlocks, starvation, and diagnosis

A **deadlock** occurs when two or more threads each hold a resource the other needs and neither
will release what it holds — a circular wait, with all involved threads blocked forever. The
standard prevention strategy is enforcing a **consistent, global lock-acquisition order** across
every code path that needs multiple locks — in a banking transfer example, always locking the
lower account number before the higher one, *regardless* of the direction money is moving,
eliminates the possibility of two transfers deadlocking against each other by acquiring the same
two locks in opposite order. **Starvation**, a related but distinct problem, happens when a
thread is perpetually denied access to a resource because other threads keep winning the
race for it — addressed with fair locking policies (`ReentrantLock`'s fairness option queues
waiting threads FIFO instead of allowing "barging") or balanced thread priorities.

Diagnosing either in a running system starts with a **thread dump** — a snapshot of every
thread's current stack trace and state, obtained via `jstack <pid>` or by sending a `SIGQUIT`
signal to the JVM process (`Ctrl+\` on Unix/Linux, `Ctrl+Break` on Windows). We'll return to
this as part of the broader debugging workflow in Chapter 17.

Threads, locks, and the memory model are the theoretical foundation; the next chapter covers
the higher-level tools — the Executor framework, concurrent collections, and modern structured
concurrency — that most real Java code actually uses to avoid hand-rolling any of this directly.

---

## Chapter 12: The Executor Framework and Modern Concurrency

### Why not just create `Thread`s directly

Creating and managing raw `Thread` objects for every unit of concurrent work doesn't scale:
threads are relatively expensive to create, there's no natural place to bound how many run
concurrently, and there's no built-in way to get a *result* back from a `Thread` cleanly. The
**Executor framework** (`java.util.concurrent`) exists to solve exactly this: it decouples
*submitting* work from the mechanics of *how* that work actually gets executed, via a managed
pool of reusable worker threads.

### `ExecutorService`, `Runnable`, and `Callable`

`ExecutorService` is the central abstraction — a higher-level replacement for manually
juggling `Thread` objects, offering lifecycle-management methods:

- **`execute(Runnable)`** — runs a task, returns nothing; any exception thrown inside goes to
  the thread's uncaught-exception handler, easy to lose track of if you're not watching for it.
- **`submit(...)`** — accepts either a `Runnable` or a `Callable<V>`, and returns a `Future<V>`
  you can use to retrieve a result, check completion, or — importantly — surface an exception
  that occurred inside the task, wrapped in `ExecutionException` when you call `Future.get()`.
- **`shutdown()`** — stops accepting new tasks but lets already-queued work finish gracefully.

`Runnable` versus `Callable<V>` mirrors the `execute()`/`submit()` split: `Runnable.run()`
returns nothing and can't throw a checked exception; `Callable<V>.call()` returns a result and
*can* throw checked exceptions — genuinely more versatile whenever a task produces a value or
needs to signal failure through a typed exception rather than silently.

### Inside `ThreadPoolExecutor`

`ExecutorService` implementations are typically backed by a `ThreadPoolExecutor`, whose
internal decision process on every submitted task is worth having as an explicit mental model:

1. If an idle **core** thread is available, use it.
2. Otherwise, if the pool hasn't yet reached its **core pool size**, create a new thread.
3. Otherwise, **queue** the task.
4. If the queue is full, grow the pool toward its **maximum pool size**.
5. If the pool is already at maximum and the queue is still full, hand the task to the
   **`RejectedExecutionHandler`** — a pluggable policy for what to do with work the pool
   genuinely cannot accept (the default throws; custom handlers can log, retry with backoff, run
   on the calling thread, or discard).

The executor itself moves through a small lifecycle of its own — **RUNNING → SHUTDOWN** (no new
tasks accepted, queued tasks still finish) **→ TERMINATED** (everything's done) — with
`shutdownNow()` providing a more aggressive `STOP` path that attempts to interrupt in-flight
tasks rather than letting them finish naturally.

Thread **interruption** throughout all of this is fundamentally **cooperative**: calling
`Thread.interrupt()` (or `Future.cancel(true)`) merely *sets a flag* on the target thread — it
does not forcibly stop anything. A task has to voluntarily and periodically check
`Thread.interrupted()` or `isInterrupted()` (or correctly handle `InterruptedException` from a
blocking call it's inside) and choose to clean up and exit. Nothing in the language forces a
thread to actually halt just because it's been asked to.

### Concurrent collections, revisited

Chapter 8 covered `ConcurrentHashMap`'s internals in detail; the broader family worth knowing:
`CopyOnWriteArrayList` (safe iteration via an immutable snapshot, at the cost of copying the
whole backing array on every write — good for read-heavy, write-rare lists like listener
registries) and the general principle that `java.util.concurrent`'s purpose-built concurrent
collections almost always outperform manually wrapping a plain collection with
`Collections.synchronizedList()`/`synchronizedMap()`, which serializes every single access
behind one coarse lock.

`ThreadLocal<T>` gives each thread its own genuinely independent copy of a variable, requiring
no synchronization at all since there's no sharing to protect against — the standard use cases
are per-request context in a web server (each HTTP request typically runs on its own thread),
and historically, non-thread-safe objects like `SimpleDateFormat` that needed a
per-thread instance. As flagged back in Chapter 3, the flip side is a genuine leak risk in
pooled-thread environments: a `ThreadLocal` value set on a pooled thread stays reachable for as
long as that thread is reused by the pool — which, in a long-lived pool, can effectively be
forever if it's never explicitly cleared.

**`CountDownLatch`** versus **`CyclicBarrier`** are both coordination primitives for a group of
threads working toward a shared point, but with a key structural difference: `CountDownLatch`
is a **one-time** gate — threads `countDown()` an initial count, and waiting threads proceed
once it hits zero, with no way to reset it for reuse. `CyclicBarrier` is a barrier point that a
*fixed number* of participating threads must all reach before any of them proceed, and it
**automatically resets** once tripped, making it suitable for repeated, multi-stage,
round-based computation where every participant needs to resynchronize at each stage. Use
`CountDownLatch` for "wait for N one-time events to complete before proceeding" (e.g., don't
start serving traffic until every subsystem has finished initializing); use `CyclicBarrier` for
"repeatedly wait for everyone to catch up before the next round begins."

`synchronized` versus `ReentrantLock` is the last comparison worth having explicit, since it
comes up constantly: `synchronized` is simpler and automatically handles lock acquisition and
release, including across exceptions, but offers no further control. `ReentrantLock` requires
manual `lock()`/`unlock()` (always paired with `try`/`finally` to guarantee release), but adds
real capabilities `synchronized` doesn't have: `tryLock()` for a non-blocking or timed
acquisition attempt, `lockInterruptibly()` to abort waiting on the lock if the thread is
interrupted, a configurable fairness policy, and support for multiple `Condition` objects per
lock (versus `synchronized`'s single implicit wait-set) — genuinely useful when you need more
nuanced coordination than one lock and one wait condition can express.

This closes out the concurrency story built up across two chapters: threads and the memory
model as the foundation, the Executor framework and concurrent collections as the practical
tools built on top. The next part moves to a different axis entirely — how Java objects cross
process boundaries (serialization) and how code can inspect and manipulate itself at runtime
(reflection).

---

# Part VI — Reflection and Wire Formats

## Chapter 13: Serialization

**Serialization** converts an object into a byte stream, for storage or transmission across a
network; **deserialization** reverses the process, reconstructing the object from that byte
stream. A class opts in by implementing the marker interface `Serializable` — a marker
interface, discussed further in the next chapter, being one with no methods at all, that exists
purely to signal a capability to the runtime.

`serialVersionUID` is a version identifier for a `Serializable` class, letting the runtime
verify that the class definition used to *serialize* an object still matches the class
definition being used to *deserialize* it. If they don't match, deserialization fails with
`InvalidClassException`, specifically to prevent a stale or incompatible class definition from
silently producing a corrupted object.

Two field-level details recur constantly in practice:

- The **`transient`** keyword excludes a specific field from serialization entirely — its value
  is simply not written out, and after deserialization it reverts to its type's default (`null`
  for objects, `0`/`false` for primitives). This is both a privacy tool (don't serialize a
  password field) and a necessity tool: if a `Serializable` class contains a field whose *type*
  isn't itself `Serializable`, attempting to serialize it throws `NotSerializableException`
  unless that specific field is marked `transient` (or the field's class is made `Serializable`
  too, or you customize the process entirely — see below).
- **`static` fields are never serialized**, for a reason that follows directly from Chapter 2's
  discussion of the Method Area: `static` fields belong to the *class*, not to any individual
  object's state, and serialization is specifically about capturing *instance* state. On
  deserialization, static fields simply retain whatever value the currently-running class
  definition already has.

For full manual control over the byte format, a class can override `writeObject()`/
`readObject()` to customize exactly how it's (de)serialized — useful for handling transient
fields that need special reconstruction logic, or for managing compatibility across class
versions by hand. Taking this further, implementing `Externalizable` instead of `Serializable`
hands you *complete* control via mandatory `writeExternal()`/`readExternal()` methods, in
exchange for taking on full responsibility for versioning correctness that `Serializable`'s
default mechanism otherwise partially handles for you via `serialVersionUID`.

Circular references — object A referencing B, which references back to A — are handled
correctly without infinite recursion: the serialization mechanism tracks every object reference
it's already written, and on encountering the same reference again, writes a back-reference to
the already-serialized object instead of serializing it a second time, preserving the original
object graph's structure on deserialization.

Serialization is also one of the ways a **Singleton** (Chapter 15) can be accidentally broken —
naive deserialization constructs a brand-new instance, bypassing the private constructor
entirely, unless the class specifically implements `readResolve()` to redirect deserialization
back to the existing singleton instance.

---

## Chapter 14: Reflection and Dynamic Behavior

### What reflection is, and its cost

**Reflection** (`java.lang.reflect`) lets a running program inspect and manipulate classes,
methods, and fields it didn't necessarily know about at compile time — dynamically
instantiating objects, invoking methods, and reading or writing fields by name at runtime. This
is the mechanism underneath testing frameworks, dependency injection containers, and
serialization libraries that need to work generically across arbitrary user-defined classes
without those classes needing to implement some shared interface. The tradeoff is real: bypassing
normal compile-time type checking has a genuine performance cost (reflective calls are
substantially slower than direct calls) and a genuine safety cost (it can bypass access
modifiers entirely via `setAccessible(true)`), so it should be reached for deliberately, not
casually.

A sharp, recurring gotcha worth stating precisely, because it connects directly back to
Chapter 2's discussion of `final`: reflection **can** modify a `final` field
(`setAccessible(true)` followed by `Field.set()`), and it will appear to "work" in the sense that
the call doesn't throw — but this breaks the language's own immutability contract, and because
the JIT compiler is permitted to optimize code under the assumption that a `final` field's value
never changes, the *observed* effect of the reflective write can be inconsistent depending on
when and where the field happens to be read elsewhere in the running program. Treat this as
"technically possible, not actually safe" — a fact worth knowing for debugging someone else's
surprising behavior, not a technique to reach for deliberately.

### Marker interfaces

A **marker interface** is an interface with no methods or fields at all — `Serializable` and
`Cloneable` are the canonical JDK examples. Its only purpose is to "mark" a class as having some
capability or eligibility, checkable at runtime via `instanceof`, without requiring any actual
method implementation. This pattern has become less central since annotations (which can carry
metadata beyond a simple yes/no marker) became widespread, but it remains foundational to how
core JDK mechanisms like serialization and `Object.clone()` decide whether an operation is
permitted at all.

### Dynamic proxies

A **dynamic proxy** (`java.lang.reflect.Proxy` plus an `InvocationHandler`) generates an
implementation of one or more interfaces *at runtime*, routing every method call on the proxy
through a single `invoke()` callback you define. This is the mechanism behind
**aspect-oriented** style cross-cutting concerns — logging, transaction management, security
checks — applied uniformly across method calls without hand-writing a wrapper class per
interface. It's literally the underlying mechanism for things like Spring's declarative
`@Transactional` support: the framework generates a proxy around your bean at runtime, and every
method call gets intercepted to wrap the real call in a transaction before and after it runs.

A minimal reflective **dependency injection** sketch illustrates the same pattern from a
different angle: scan a class's fields via reflection for a custom annotation like `@Inject`,
then use reflection to instantiate and assign the required dependency into each annotated field
(bypassing private access via `setAccessible(true)`). This is the essential mechanism that much
larger DI frameworks (Spring, Guice) build considerably more machinery around, but the core idea
— inspect, then construct and wire based on what you find — is exactly this.

### Cloning: shallow versus deep

`Cloneable` is another marker interface: implementing it signals that calling `clone()` on the
class is permitted (without it, `Object.clone()` throws `CloneNotSupportedException`), but it
does *not* by itself give you any particular cloning behavior — the default `Object.clone()`
performs a **shallow copy**: it duplicates the object's own fields, but any field that's itself
a reference to another object still points at that *same* shared nested object in both the
original and the clone, so mutating the nested object through one is visible through the other.
A **deep copy** additionally, recursively copies every nested object too, producing a clone
that shares nothing at all with the original. Achieving a deep copy requires either manually
cloning each nested object yourself (typically inside an overridden `clone()` or a dedicated copy
constructor), or — for complex object graphs where hand-written cloning would be tedious and
error-prone — serializing the whole object to a byte stream and immediately deserializing it
back, which produces a fully independent copy of the entire graph as a side effect of how
serialization works, at the cost of requiring everything involved to be `Serializable` and being
considerably slower than a hand-written deep clone.

Reflection and serialization round out the "dynamic behavior" side of Java. The next part turns
to design at a higher level of abstraction — the patterns and principles that shape how classes
and objects are put together into larger systems, building directly on the OOP pillars from
Chapter 5.

---

# Part VII — Design

## Chapter 15: Design Patterns and SOLID Principles

Design patterns are proven, named solutions to recurring design problems — a shared vocabulary
that lets developers communicate intent quickly ("just use a Builder here") rather than
re-deriving a solution from scratch every time. They add a small amount of structural
indirection in exchange for maintainability, and that tradeoff is usually worth it for anything
beyond genuinely trivial code.

### Singleton

**Singleton** guarantees a class has exactly one instance, reachable through a single global
access point — appropriate for genuinely shared, unique resources like a configuration manager
or a database connection pool. The classic implementation: a `private` constructor (preventing
outside instantiation, as covered in Chapter 4), a `private static` instance field, and a
`public static` accessor that lazily creates the instance on first call.

That naive lazy version is **not thread-safe** by default — two threads racing to call the
accessor for the first time simultaneously can both see a `null` instance and each create their
own, producing two "singleton" instances. There are several standard fixes, worth knowing in
order of increasing elegance:

- Make the accessor method `synchronized` — correct, but pays a synchronization cost on
  *every* call, forever, even though the race only matters on the very first call.
- **The Bill Pugh / initialization-on-demand holder idiom**: put the instance in a `private
  static` field of a separate **static nested class**. Because the JVM's own class-loading
  mechanism guarantees a class is loaded lazily and initialized exactly once, thread-safely, by
  the classloader itself (a direct consequence of the classloading model from Chapter 1), this
  gives you a lazily-created, genuinely thread-safe singleton with **no explicit
  synchronization at all** — the nested holder class simply isn't loaded (and its static field
  isn't initialized) until something first references it.
- **Enum-based singleton**: declare a single-element `enum` (`enum Config { INSTANCE; ... }`).
  This is widely considered the most robust option, because it solves not just the basic race
  condition but also every one of the ways a "normal" Singleton can be *broken after the fact*:
  reflection can normally call a private constructor directly, bypassing the singleton
  guarantee entirely — but the JVM specifically forbids reflective instantiation of enum
  constants. Naive deserialization normally constructs a fresh instance — but enum
  deserialization is handled specially by the JVM to always resolve back to the existing
  constant. And `clone()` would normally be able to produce a second instance — but `Enum`
  doesn't support cloning at all. Where a hand-rolled Singleton needs `readResolve()`,
  constructor guards, and an overridden `clone()` to defend against these three attack surfaces
  individually, enum-based singleton closes all three by construction.

### Builder

**Builder** constructs a complex object step by step, letting different parts of construction
happen independently before final assembly, and is the standard alternative once a class's
constructor would otherwise need many parameters (especially many *optional* ones) — avoiding
both constructor-overload explosion and the classic bug-magnet of a long constructor call where
it's easy to pass arguments in the wrong order. It's distinct from **Factory** in what problem
each solves: Factory decides *which concrete type* to instantiate, in one step, hiding that
decision from the caller; Builder controls *how* a (possibly single, already-known) complex
object gets assembled, piece by piece, giving the caller fine control over the construction
process itself rather than over which class gets chosen.

### Strategy vs. State

Both patterns delegate behavior to an interface with multiple interchangeable implementations,
and are easy to mix up structurally — the distinction is about **who decides and why**.
**Strategy** lets a *client* pick one algorithm out of an interchangeable family, chosen
externally, and that choice generally doesn't change based on anything intrinsic to the object
using it — sorting with one `Comparator` versus another is a Strategy-shaped decision. **State**
is about an object changing its *own* behavior as its *own* internal condition changes, with the
transition typically driven from inside the object itself — from the outside, the object
appears to dynamically change what class it effectively belongs to as it moves between states
(a `TrafficLight` behaving differently depending on whether it's currently Red, Yellow, or
Green, with the light itself managing when to transition).

### Observer

**Observer** lets objects (observers, or listeners) register with a subject (an event source)
to be notified when the subject's state changes, decoupling the event source entirely from
whatever logic reacts to the event — observers can be added or removed dynamically without the
subject needing to know anything about who's currently listening or what they do with the
notification. This is the foundational pattern underneath essentially all event-driven and
reactive system design, GUI toolkits very much included.

### SOLID

The five SOLID principles are a compact set of heuristics for keeping object-oriented designs
maintainable as they grow, and they read cleanly as direct extensions of the four pillars from
Chapter 5:

- **S — Single Responsibility.** A class should have only one reason to change. A
  `VehicleRegistration` class that also handles vehicle insurance now has two unrelated reasons
  to be modified, which is exactly the kind of tangled responsibility this principle flags.
- **O — Open/Closed.** Classes should be open for *extension* but closed for *modification* —
  new behavior should be addable without editing code that already works. A `VehicleService`
  class that needs to support a new electric-vehicle service type should be extended via a new
  `ElectricVehicleService` subclass, not modified in place.
- **L — Liskov Substitution.** Anywhere a superclass reference is expected, any subclass
  instance should be substitutable without breaking correctness. If a `Vehicle` superclass
  defines `startEngine()`, and `ElectricCar` genuinely can't implement that meaningfully because
  it has no traditional engine, that's a signal the abstraction itself is wrong — not that
  `ElectricCar` should throw or silently no-op, either of which would violate the substitution
  guarantee callers of `Vehicle` are entitled to rely on.
- **I — Interface Segregation.** Don't force a client to depend on methods it doesn't use;
  prefer several small, focused interfaces over one large one. A fat `VehicleOperations`
  interface with `drive`, `refuel`, `charge`, and `navigate` forces an `ElectricCar` to either
  implement a meaningless `refuel()` or throw from it; splitting into `Drivable`, `Refuelable`,
  `Chargeable`, and `Navigable` lets each vehicle type implement exactly what applies to it.
- **D — Dependency Inversion.** High-level modules shouldn't depend directly on low-level
  implementation details; both should depend on abstractions. A `VehicleTracker` that logs
  positions shouldn't be coupled to one specific GPS device model — it should depend on a
  `GPSDevice` interface, letting any conforming implementation be swapped in without touching
  `VehicleTracker` at all. This is, not coincidentally, the same idea Chapter 5's abstraction
  discussion called "loose coupling" — Dependency Inversion is that principle applied
  specifically to how classes acquire their collaborators.

Design patterns and SOLID close out the "how to structure code" half of this book. The final
two parts look forward and outward: where the language itself has been heading in recent
releases, and how to actually keep a Java system healthy once it's running in production.

---

# Part VIII — Where the Language Is Going

## Chapter 16: Modern Java — Records, Sealed Classes, and the Module System

Java's release cadence since Java 9 (a new version every six months, with periodic long-term
support releases) has been steadily aimed at two goals that show up across almost every feature
added: reducing boilerplate, and letting the compiler enforce more correctness than it
previously could — while, true to Java's original design philosophy from Chapter 1, preserving
backward compatibility throughout.

### The module system (Java 9, "Project Jigsaw")

Before Java 9, the largest unit of code organization was the package, and package-private
visibility was only ever a *soft* boundary — nothing stopped another JAR on the classpath from
adding a class to the same package name and reaching in. The **module system** introduces a
genuinely stronger boundary: a module is declared in a `module-info.java` file, which specifies
its dependencies (`requires`) and exactly which packages it exposes to the outside (`exports`).
Anything not explicitly exported is now **truly** inaccessible from outside the module — real
encapsulation enforced by the JVM itself, not just a naming convention.

The practical benefits compound: genuinely stronger encapsulation and security (internal
implementation details are no longer just "discouraged" from external access but actually
unreachable), more explicit and manageable dependency graphs for large applications, and —
tying back to Chapter 1's mention of `jlink` — the ability to build a custom, minimal runtime
image containing only the modules an application actually needs, meaningfully reducing both
memory footprint and startup time versus shipping the full JRE. For large enterprise
applications built from many internal components, this modularity simplifies updates, supports
better internal versioning discipline, and generally makes a large system more manageable as a
collection of well-defined, independently reasoned-about parts rather than one undifferentiated
classpath.

### Records (Java 14+)

A **record** is a concise syntax for declaring an immutable data-carrying class. Given just the
component names and types, the compiler automatically generates a canonical constructor,
accessor methods, and correct `equals()`, `hashCode()`, and `toString()` implementations — all
the boilerplate that a hand-written immutable value class (Chapter 6's immutability principles,
applied directly) would otherwise require you to write and keep in sync by hand. Record
components are implicitly `final`, reinforcing immutability by construction. Records are ideal
for simple data models and data-transfer objects, where a class's entire purpose is to hold a
fixed set of values, not to carry independent behavior.

### Sealed classes and interfaces (Java 15, finalized in 17)

A **sealed** class or interface restricts, via an explicit `permits` clause, exactly which
classes are allowed to directly extend or implement it — turning what used to be an open-ended,
unknowable set of possible subclasses into a closed, compiler-known set. This directly enables
**exhaustive pattern matching**: a `switch` over a sealed type's permitted subclasses can be
verified complete by the compiler, with no `default` case needed and a compile error if a new
permitted subclass is ever added without updating every switch that pattern-matches over the
hierarchy. This is genuinely useful for modeling closed domain hierarchies where you *want* the
compiler catching missing cases — a `Shape` that's definitively only ever `Circle`, `Square`, or
`Triangle`, and nothing else, ever.

### The throughline (Java 17–21 and beyond)

Sealed classes, pattern matching for `switch`, and the Foreign Function & Memory API (Java 17);
virtual threads, structured concurrency, scoped values, sequenced collections, and record
patterns (Java 21) — each addresses a genuinely different corner of the language, but the
consistent theme across all of them is the same one this chapter opened with: shrinking the gap
between what a correct program has to say explicitly and what the compiler can verify on its
behalf, without breaking anything written against an earlier version of the language. Virtual
threads in particular are worth a forward pointer: they're lightweight, JVM-scheduled threads
(as opposed to the OS-scheduled "platform threads" this book's concurrency chapters have been
describing throughout) designed to make the thread-per-task programming model from Chapter 11
viable at a scale — hundreds of thousands of concurrent threads — that would be impossible with
traditional OS threads, without requiring a rewrite into a fundamentally different asynchronous
programming style.

With the language's trajectory in view, the last chapter turns from *how Java is designed* to
*how to keep a Java system healthy once it's actually running* — pulling together the debugging
and diagnostic threads that have been seeded throughout this book into one practical reference.

---

# Part IX — Keeping It Running

## Chapter 17: Debugging and Performance in Production

This closing chapter is deliberately different from the rest of the book: less "how the
language works," more "what to actually do" when it stops working the way you expect. Every
technique here has already been introduced somewhere earlier in this book — this chapter just
gathers the practical workflows into one place.

### `OutOfMemoryError`

Revisiting and consolidating the workflow from Chapter 3: check heap sizing (`-Xms`/`-Xmx`)
first, since sometimes the problem is genuinely just an undersized heap for real load, not a
leak at all. If a leak is suspected, capture a heap dump — either on-demand with `jmap`, or
automatically at the moment of failure with `-XX:+HeapDumpOnOutOfMemoryError`, which is worth
enabling by default on any production JVM, since a dump captured after the fact is far less
useful than one captured at the exact moment of failure. Analyze the dump with a tool like
Eclipse Memory Analyzer (MAT) or VisualVM, specifically looking for **dominator** objects —
objects retaining unexpectedly large amounts of memory through the reference chains they hold —
and instance counts far higher than the application logic should ever produce. From there, trace
the GC-root reference chain keeping the suspect objects alive (Chapter 3's reachability model is
exactly what you're manually reconstructing here) back to the actual retaining reference in your
code, and fix it: remove the reference, scope it more narrowly, switch to a `WeakReference`
(Chapter 3), or add proper cleanup for listeners and resources.

### Thread problems: deadlocks, starvation, and hangs

For anything that looks like threads are stuck, blocked, or the application has simply stopped
making progress, the tool is a **thread dump** (`jstack <pid>`, or `SIGQUIT` via `Ctrl+\`/`Ctrl+Break`
as covered in Chapter 11) — a snapshot of every thread's current stack trace and state at a
single point in time. Reading a thread dump for a suspected deadlock means looking for two or
more threads each shown as `BLOCKED`, waiting on a lock that's held by one of the *other*
blocked threads — a circular wait, visible directly in the dump's lock-ownership information.
Tools like Java VisualVM can often detect and highlight this automatically rather than requiring
manual inspection of every thread's stack.

### Memory-leak-shaped bugs that aren't `OutOfMemoryError` (yet)

Not every leak announces itself with a crash — sometimes it shows up first as gradually
degrading performance, or (per Chapter 8) as a `HashMap` that mysteriously "loses" entries it
should still contain. The Chapter 8 mutable-key scenario is worth restating here as a debugging
pattern specifically: if a cache or map seems to silently drop entries that were definitely
inserted, check whether the key type is mutable and whether any field contributing to its
`hashCode()` could have changed after insertion — that specific failure mode produces no
exception at all, just entries that are permanently unreachable through normal lookup.

### JIT and startup-time tradeoffs

As covered in Chapter 1, the JIT compiler trades upfront compilation cost for long-run
execution speed, which is nearly always the right tradeoff for a long-running server process —
but worth remembering as a real, occasionally relevant lever for short-lived processes (quick
CLI tools, certain serverless-style invocations) where startup time dominates and the runtime
performance benefit never gets the chance to pay itself back.

### `equals()`/`hashCode()` correctness as a production performance issue, not just a correctness one

It's worth closing on a point that ties this whole book together: the `equals()`/`hashCode()`
contract from Chapter 8 isn't purely an academic collections-correctness rule — it's directly a
**production performance and correctness issue** the moment those objects are used as cache
keys. A cache keyed by objects with a broken or missing `hashCode()`/`equals()` pairing either
silently "loses" entries it should be finding (degrading cache-hit performance in a way that
looks like an unrelated slowdown, not an obvious bug) or, worse, returns *wrong* data if two
logically-different objects happen to be treated as equal by accident. Tracing a production
performance regression back to a class whose `equals()` was overridden six months ago without a
matching `hashCode()` override is a genuinely common story — and it's a direct, concrete
illustration of why this book insisted, all the way back in Chapter 8, on treating the two
methods as one inseparable contract rather than two independent choices.

---

## Closing note

Everything in this book connects back to a small number of foundational facts: bytecode running
on a virtual machine rather than bare metal, a heap managed by generational, reachability-based
garbage collection, an object model built around encapsulation and dynamic dispatch, and a
standard library — collections, streams, concurrency utilities — built consistently on top of
all of that. Interview questions ask you to recite these facts in isolation. Understanding Java
well means seeing how few independent ideas they actually reduce to, and how directly one
explains the next.
