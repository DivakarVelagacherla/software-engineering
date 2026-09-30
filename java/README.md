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

1. [Part I — The Machine Underneath](./01-the-machine-underneath.md)
   The JVM, memory (heap/stack), garbage collection
2. [Part II — The Language](./02-the-language.md)
   The object model, the four pillars of OOP, primitives/wrappers/strings, exceptions
3. [Part III — Data](./03-data.md)
   The Collections Framework, generics and type safety
4. [Part IV — Functional Java](./04-functional-java.md)
   Lambdas, streams, and the Java 8 turn
5. [Part V — Concurrency](./05-concurrency.md)
   Threads, locks, the Java Memory Model, the Executor Framework
6. [Part VI — Reflection and Wire Formats](./06-reflection-and-wire-formats.md)
   Serialization, reflection and dynamic behavior
7. [Part VII — Design](./07-design.md)
   Design patterns and SOLID principles
8. [Part VIII — Where the Language Is Going](./08-where-the-language-is-going.md)
   Modern Java: records, sealed classes, the module system
9. [Part IX — Keeping It Running](./09-keeping-it-running.md)
   Debugging and performance in production
