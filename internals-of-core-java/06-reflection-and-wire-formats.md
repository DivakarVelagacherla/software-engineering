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

