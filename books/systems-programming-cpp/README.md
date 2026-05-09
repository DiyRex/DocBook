# Systems Programming & Software Architecture Foundations with C++

A book for intermediate developers who can write code but want to understand **what is actually happening** when their code runs — and why software is built the way it is.

This is not a C++ tutorial. C++ is the *teaching language* because it forces you to confront memory, lifetime, ownership, and execution directly. What you learn here transfers to every other language you will ever touch.

> **[← Back to DocBook home](../../README.md)**

---

## Start Here

- **[Preface — why this book exists](preface.md)**

Read in order. Each part builds on the last.

---

## Table of Contents

### [Part 1 — What Programming Actually Is](part-01-what-programming-is/)

The substrate: how programs become processes, what the CPU and memory really are, what the OS is doing on your behalf, and how source code becomes machine code.

1. [What Happens When A Program Runs](part-01-what-programming-is/01-what-happens-when-a-program-runs.md)
2. [CPU, RAM, Stack, Heap, Registers](part-01-what-programming-is/02-cpu-ram-stack-heap-registers.md)
3. [How Operating Systems Execute Programs](part-01-what-programming-is/03-how-os-executes-programs.md)
4. [Machine Code, Assembly, Compilers, Interpreters](part-01-what-programming-is/04-machine-code-assembly-compilers-interpreters.md)
5. [Memory Layout Of A Process](part-01-what-programming-is/05-memory-layout-of-a-process.md)
6. [Function Calls Internally](part-01-what-programming-is/06-function-calls-internally.md)
7. [Call Stack Deep Dive](part-01-what-programming-is/07-call-stack-deep-dive.md)
8. [Pointers As Memory Addresses](part-01-what-programming-is/08-pointers-as-memory-addresses.md)
9. [Why Abstractions Exist](part-01-what-programming-is/09-why-abstractions-exist.md)
10. [What Languages Actually Do](part-01-what-programming-is/10-what-languages-actually-do.md)

### [Part 2 — Memory & Execution Foundations](part-02-memory-execution/)

The dimension that dominates real-world performance and reliability. Stack and heap internals, manual memory management, ownership, lifetimes, RAII, fragmentation, the cache hierarchy, data-oriented design, smart pointers, reference counting, garbage collection internals, and how Python/Go/PHP runtimes actually manage memory.

11. [Stack vs Heap Internals](part-02-memory-execution/01-stack-vs-heap-internals.md)
12. [Manual Memory Management](part-02-memory-execution/02-manual-memory-management.md)
13. [Ownership Models](part-02-memory-execution/03-ownership-models.md)
14. [Lifetimes](part-02-memory-execution/04-lifetimes.md)
15. [RAII Explained Deeply](part-02-memory-execution/05-raii-explained-deeply.md)
16. [Memory Fragmentation](part-02-memory-execution/06-memory-fragmentation.md)
17. [Caches and CPU Locality](part-02-memory-execution/07-caches-and-cpu-locality.md)
18. [Data-Oriented Design](part-02-memory-execution/08-data-oriented-design.md)
19. [Smart Pointers Internals](part-02-memory-execution/09-smart-pointers-internals.md)
20. [Reference Counting](part-02-memory-execution/10-reference-counting.md)
21. [Garbage Collection Internals](part-02-memory-execution/11-garbage-collection-internals.md)
22. [How Python, Go, and PHP Actually Manage Memory](part-02-memory-execution/12-how-languages-manage-memory.md)

### [Part 3 — Understanding Abstractions](part-03-understanding-abstractions/)

Abstractions taken apart mechanically. What an abstraction is built out of, encapsulation from first principles, why classes exist, what objects are in memory, composition vs inheritance, interfaces and contracts, polymorphism internals, vtables, static vs dynamic dispatch, dependency injection from scratch.

23. [What Is An Abstraction (Mechanically)](part-03-understanding-abstractions/01-what-is-an-abstraction.md)
24. [Encapsulation From First Principles](part-03-understanding-abstractions/02-encapsulation.md)
25. [Why Classes Exist](part-03-understanding-abstractions/03-why-classes-exist.md)
26. [What Objects Are In Memory](part-03-understanding-abstractions/04-what-objects-are-in-memory.md)
27. [Composition vs Inheritance](part-03-understanding-abstractions/05-composition-vs-inheritance.md)
28. [Interfaces and Contracts](part-03-understanding-abstractions/06-interfaces-and-contracts.md)
29. [Polymorphism Internals](part-03-understanding-abstractions/07-polymorphism-internals.md)
30. [Virtual Tables](part-03-understanding-abstractions/08-virtual-tables.md)
31. [Static vs Dynamic Dispatch](part-03-understanding-abstractions/09-static-vs-dynamic-dispatch.md)
32. [Dependency Injection From Scratch](part-03-understanding-abstractions/10-dependency-injection-from-scratch.md)

### [Part 4 — Architecture Thinking](part-04-architecture-thinking/)

Organizing 50,000 lines of code so it stays understandable. What architecture is, why large programs become complex, coupling vs cohesion, separation of concerns, layered architecture, controllers/services/repositories, MVC, DDD, DI, boundaries, organizing large codebases, and ADRs.

33. [What Is Software Architecture](part-04-architecture-thinking/01-what-is-software-architecture.md)
34. [Why Large Programs Become Complex](part-04-architecture-thinking/02-why-large-programs-become-complex.md)
35. [Coupling vs Cohesion](part-04-architecture-thinking/03-coupling-vs-cohesion.md)
36. [Separation of Concerns](part-04-architecture-thinking/04-separation-of-concerns.md)
37. [Layered Architecture](part-04-architecture-thinking/05-layered-architecture.md)
38. [Controllers, Services, Repositories](part-04-architecture-thinking/06-controllers-services-repositories.md)
39. [Why Models Exist](part-04-architecture-thinking/07-why-models-exist.md)
40. [MVC Internals](part-04-architecture-thinking/08-mvc-internals.md)
41. [Domain-Driven Thinking](part-04-architecture-thinking/09-domain-driven-thinking.md)
42. [Why Frameworks Use DI](part-04-architecture-thinking/10-why-frameworks-use-di.md)
43. [Boundaries — When To Split Logic](part-04-architecture-thinking/11-boundaries-when-to-split.md)
44. [Organizing Large Codebases](part-04-architecture-thinking/12-organizing-large-codebases.md)
45. [Architectural Decision Records (and Part 4 Synthesis)](part-04-architecture-thinking/13-architectural-decision-records.md)

### Part 5 — Runtime & Concurrency *(coming soon)*

Processes vs threads, concurrency vs parallelism, schedulers, async runtime internals, why Go uses Context, cancellation propagation, event loops, coroutines, how Python AsyncIO works, thread safety, locks and atomics, memory ordering.

### Part 6 — Languages & Runtime Design *(coming soon)*

Compiled vs interpreted, JIT, how Python executes code, why Python uses `__name__ == "__main__"`, how Go builds binaries, how the JVM works, how garbage collectors work, why Rust ownership exists, why C++ is difficult, language design tradeoffs.

### Part 7 — Building Real Systems *(coming soon)*

Designing a CLI tool, a web server, a database layer, a mini framework, a DI container, an event bus, plugin architecture, hot reloading concepts, configuration systems, observability and logging.

### Part 8 — Advanced Systems Thinking *(coming soon)*

Memory mapped files, networking internals, syscalls, kernel vs user space, serialization internals, protocol design, performance engineering, profiling, CPU pipelines, branch prediction, SIMD, cache misses, false sharing.

### Part 9 — Building A Mental Model For Any Language *(coming soon)*

How to learn any programming language, recognizing common runtime patterns, mapping framework concepts across languages, recognizing architectural patterns, understanding any codebase quickly, reasoning about complexity, how senior engineers think, from developer to software architect.

---

## Conventions Used in This Book

- **Code blocks** are real, compilable code unless explicitly marked as pseudocode.
- **Memory diagrams** use ASCII art for portability. Read them carefully — they encode the mental model.
- **Experiments** are numbered and have explicit success criteria. If your output disagrees with the book, *the book is probably wrong about your platform*; investigate, don't move on.
- **Misconceptions** sections explicitly call out wrong mental models the author has seen developers carry for years.
- **Tradeoffs** sections list what each design choice *costs* — there is no free lunch in systems engineering.

---

## Prerequisites

- You can write a small program in some language (Python, JavaScript, Java, C, Go — any of these is fine).
- You have a Unix-like shell available (Linux, macOS, or WSL on Windows).
- You can install a C++20-capable compiler (`clang++` or `g++` ≥ 10) and basic tooling: `gdb` or `lldb`, `objdump`, `nm`, `strace` or `dtrace`.

You do **not** need prior C++ experience. C++ is introduced as we need it.

---

## Status

| Part | Chapters | Status |
|---|---|---|
| Part 1 | 10 | 10/10 ✅ |
| Part 2 | 12 | 12/12 ✅ |
| Part 3 | 10 | 10/10 ✅ |
| Part 4 | 13 | 13/13 ✅ |
| Part 5 | 12 | not started |
| Part 6 | 10 | not started |
| Part 7 | 10 | not started |
| Part 8 | 13 | not started |
| Part 9 | 8 | not started |
| **Total** | **~98** | **10 written** |
