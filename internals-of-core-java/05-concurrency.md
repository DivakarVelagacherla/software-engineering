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

