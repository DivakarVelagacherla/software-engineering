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

