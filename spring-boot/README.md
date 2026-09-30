# Everything About Spring and Spring Boot

*A continuation of Internals of Core Java — how the framework that sits on top of the language actually works*

---

## How this book connects to the last one

*Internals of Core Java* covered the language and the JVM: objects, classes, reflection,
generics, collections, concurrency. This book is the layer built on top of that foundation.
Spring isn't a new language — it's a very large, very deliberate application of the concepts
from that book. The IoC container is built on reflection and classloading (Chapter 1 and
Chapter 14 of the last book). Spring AOP is built on dynamic proxies (also Chapter 14).
`@Transactional` and `@Async` only work because of how Java resolves method calls through those
proxies, which is really a question about polymorphism and dispatch (Chapter 5). Spring's own
internals lean on the Singleton, Factory, and Proxy design patterns (Chapter 15). None of that
is a coincidence — Spring is, in a real sense, a systematic demonstration of what those Core
Java ideas are *for*.

So this book assumes you've read the first one, or know its material, and it will point back to
specific chapters rather than re-explain them. What it adds is everything Spring and Spring Boot
build on top: dependency injection as an architectural discipline, a container that manages
object lifecycles for you, a mechanism for cross-cutting concerns, and — in the second half —
Spring Boot's answer to the operational pain that plain Spring accumulated over a decade of
real-world use.

The structure follows the arc you'd actually want to understand it in: first, what problem
Spring was solving and how it solves it under the hood; then, what new problems *Spring itself*
created at scale, and how Spring Boot was built specifically to solve those; then a deep,
practical tour of building and running real systems with Spring Boot, ending — like the last
book — with production debugging.

---

## Table of Contents

1. [Part I — Why Spring Exists](./01-why-spring-exists.md)
   The problem before Spring, IoC, dependency injection, AOP, data access & transactions
2. [Part II — Why Spring Boot Exists](./02-why-spring-boot-exists.md)
   The pain points of plain Spring, auto-configuration, starters, the embedded server
3. [Part III — Building With Spring Boot](./03-building-with-spring-boot.md)
   REST APIs, data access, testing, Actuator and observability
4. [Part IV — Spring Boot at Scale](./04-spring-boot-at-scale.md)
   Microservices, caching/async/reactive, security, scaling, production war stories
