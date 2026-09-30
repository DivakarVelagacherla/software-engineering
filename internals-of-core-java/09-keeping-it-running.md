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
