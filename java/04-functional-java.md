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

