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

