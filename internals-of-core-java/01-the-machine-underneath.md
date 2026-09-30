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

