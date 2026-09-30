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

