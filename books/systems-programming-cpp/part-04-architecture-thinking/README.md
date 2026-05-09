# Part 4 — Architecture Thinking

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Parts 1–3 built the substrate, the memory model, and the abstraction mechanisms. Part 4 zooms out: how do you organize 50,000 lines of code so it stays understandable? What is "architecture" actually, beyond box diagrams? When do you split a module into a service? Why do every modern framework's controller, service, and repository names converge?

By the end, you have working vocabulary for every architectural conversation in industry — coupling, cohesion, separation of concerns, layered/hexagonal/clean architecture, MVC variants, DDD, DI containers, boundaries, ADRs.

---

## Chapters

33. **[What Is Software Architecture](01-what-is-software-architecture.md)** — Decisions expensive to change. Quality attributes. ADRs.
34. **[Why Large Programs Become Complex](02-why-large-programs-become-complex.md)** — Essential vs accidental. Conway's law. Big Ball of Mud.
35. **[Coupling vs Cohesion](03-coupling-vs-cohesion.md)** — Constantine's six and seven. Diagnosing in C++ specifically.
36. **[Separation of Concerns](04-separation-of-concerns.md)** — One reason to change. SRP reconsidered. Cross-cutting concerns.
37. **[Layered Architecture](05-layered-architecture.md)** — Three layers, hexagonal, clean. Why strict layering fails. Anti-patterns.
38. **[Controllers, Services, Repositories](06-controllers-services-repositories.md)** — What each is for and what each must NOT do.
39. **[Why Models Exist](07-why-models-exist.md)** — Domain vs data model. Anemic vs rich. The ORM trap. DTOs.
40. **[MVC Internals](08-mvc-internals.md)** — Smalltalk MVC vs Web MVC. MVVM, MVP. Why MVC without DI is untestable.
41. **[Domain-Driven Thinking](09-domain-driven-thinking.md)** — Ubiquitous language, bounded contexts, aggregates, strategic vs tactical.
42. **[Why Frameworks Use DI](10-why-frameworks-use-di.md)** — The wiring problem. Auto-wiring. Lifetimes/scopes. When you don't need a container.
43. **[Boundaries — When To Split Logic](11-boundaries-when-to-split.md)** — Seven boundary types. Signals to split. The microservice trap.
44. **[Organizing Large Codebases](12-organizing-large-codebases.md)** — Layer-first vs feature-first. Monorepo vs polyrepo. Build times as a signal.
45. **[Architectural Decision Records (and Part 4 Synthesis)](13-architectural-decision-records.md)** — The ADR habit. A worked walk-through tying every chapter together.

---

## How to Use Part 4

- **Apply each chapter to your own codebase as you read.** The point of architecture is judgment, not vocabulary; running each chapter against a real system you know is where the chapters become useful.
- **Don't adopt patterns wholesale.** Each chapter has a "when this is the wrong tool" or "anti-pattern" section. Those are usually the most important sections.
- **Read Ch 45 last.** It synthesizes the rest with a single worked example.

> **Next: Part 5 — Runtime & Concurrency** *(coming soon)*
