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

### [Part 5 — Runtime & Concurrency](part-05-runtime-concurrency/)

What your code looks like while it's running. Processes vs threads, concurrency vs parallelism, schedulers, async runtime internals, why Go uses Context, cancellation propagation, event loops, coroutines, how Python AsyncIO works, thread safety, locks and atomics, memory ordering.

46. [Processes vs Threads](part-05-runtime-concurrency/01-processes-vs-threads.md)
47. [Concurrency vs Parallelism](part-05-runtime-concurrency/02-concurrency-vs-parallelism.md)
48. [Schedulers](part-05-runtime-concurrency/03-schedulers.md)
49. [Async Runtime Internals](part-05-runtime-concurrency/04-async-runtime-internals.md)
50. [Why Go Uses Context](part-05-runtime-concurrency/05-why-go-uses-context.md)
51. [Cancellation Propagation](part-05-runtime-concurrency/06-cancellation-propagation.md)
52. [Event Loops](part-05-runtime-concurrency/07-event-loops.md)
53. [Coroutines](part-05-runtime-concurrency/08-coroutines.md)
54. [How Python AsyncIO Works](part-05-runtime-concurrency/09-how-python-asyncio-works.md)
55. [Thread Safety](part-05-runtime-concurrency/10-thread-safety.md)
56. [Locks and Atomics](part-05-runtime-concurrency/11-locks-and-atomics.md)
57. [Memory Ordering](part-05-runtime-concurrency/12-memory-ordering.md)

### [Part 6 — Languages & Runtime Design](part-06-languages-runtime-design/)

How source becomes execution, how major runtimes actually run code, the GC tradeoffs each picked, why Rust ownership exists, why C++ stays difficult, and a transferable framework for evaluating any language.

58. [Compiled vs Interpreted](part-06-languages-runtime-design/01-compiled-vs-interpreted.md)
59. [JIT Compilation](part-06-languages-runtime-design/02-jit-compilation.md)
60. [How Python Executes Code](part-06-languages-runtime-design/03-how-python-executes-code.md)
61. [Why Python Uses `__name__ == "__main__"`](part-06-languages-runtime-design/04-why-python-uses-name-main.md)
62. [How Go Builds Binaries](part-06-languages-runtime-design/05-how-go-builds-binaries.md)
63. [How the JVM Works](part-06-languages-runtime-design/06-how-the-jvm-works.md)
64. [Runtime GC Tradeoffs Across Languages](part-06-languages-runtime-design/07-runtime-gc-tradeoffs.md)
65. [Why Rust Ownership Exists](part-06-languages-runtime-design/08-why-rust-ownership-exists.md)
66. [Why C++ Is Difficult](part-06-languages-runtime-design/09-why-cpp-is-difficult.md)
67. [Language Design Tradeoffs (Part 6 Synthesis)](part-06-languages-runtime-design/10-language-design-tradeoffs.md)

### [Part 7 — Building Real Systems](part-07-building-real-systems/)

Putting the substrate to work. Each chapter walks through a real subsystem you'll build (or replace) at some point.

68. [Designing a CLI Tool](part-07-building-real-systems/01-designing-a-cli-tool.md)
69. [Designing a Web Server](part-07-building-real-systems/02-designing-a-web-server.md)
70. [A Database Layer](part-07-building-real-systems/03-a-database-layer.md)
71. [A Mini Framework](part-07-building-real-systems/04-a-mini-framework.md)
72. [A DI Container](part-07-building-real-systems/05-a-di-container.md)
73. [An Event Bus](part-07-building-real-systems/06-an-event-bus.md)
74. [Plugin Architecture](part-07-building-real-systems/07-plugin-architecture.md)
75. [Hot Reloading](part-07-building-real-systems/08-hot-reloading.md)
76. [Configuration Systems](part-07-building-real-systems/09-configuration-systems.md)
77. [Observability and Logging](part-07-building-real-systems/10-observability-and-logging.md)

### [Part 8 — Advanced Systems Thinking](part-08-advanced-systems-thinking/)

OS-adjacent (memory-mapped files, networking, syscalls, kernel/userspace, serialization, protocol design) and CPU-adjacent (performance engineering, profiling, pipelines, branch prediction, SIMD, cache misses, false sharing).

78. [Memory Mapped Files](part-08-advanced-systems-thinking/01-memory-mapped-files.md)
79. [Networking Internals](part-08-advanced-systems-thinking/02-networking-internals.md)
80. [Syscalls](part-08-advanced-systems-thinking/03-syscalls.md)
81. [Kernel vs User Space](part-08-advanced-systems-thinking/04-kernel-vs-user-space.md)
82. [Serialization Internals](part-08-advanced-systems-thinking/05-serialization-internals.md)
83. [Protocol Design](part-08-advanced-systems-thinking/06-protocol-design.md)
84. [Performance Engineering](part-08-advanced-systems-thinking/07-performance-engineering.md)
85. [Profiling](part-08-advanced-systems-thinking/08-profiling.md)
86. [CPU Pipelines](part-08-advanced-systems-thinking/09-cpu-pipelines.md)
87. [Branch Prediction](part-08-advanced-systems-thinking/10-branch-prediction.md)
88. [SIMD](part-08-advanced-systems-thinking/11-simd.md)
89. [Cache Misses (Deeper)](part-08-advanced-systems-thinking/12-cache-misses.md)
90. [False Sharing](part-08-advanced-systems-thinking/13-false-sharing.md)

### [Part 9 — Building A Mental Model For Any Language](part-09-mental-model-for-any-language/)

The closing part. How to walk into any new codebase, language, or framework and orient yourself in hours instead of weeks.

91. [How To Learn Any Programming Language](part-09-mental-model-for-any-language/01-how-to-learn-any-language.md)
92. [Recognizing Common Runtime Patterns](part-09-mental-model-for-any-language/02-recognizing-runtime-patterns.md)
93. [Mapping Framework Concepts Across Languages](part-09-mental-model-for-any-language/03-mapping-framework-concepts.md)
94. [Recognizing Architectural Patterns](part-09-mental-model-for-any-language/04-recognizing-architectural-patterns.md)
95. [Understanding Any Codebase Quickly](part-09-mental-model-for-any-language/05-understanding-any-codebase.md)
96. [Reasoning About Complexity](part-09-mental-model-for-any-language/06-reasoning-about-complexity.md)
97. [How Senior Engineers Think](part-09-mental-model-for-any-language/07-how-senior-engineers-think.md)
98. [From Developer to Software Architect](part-09-mental-model-for-any-language/08-from-developer-to-architect.md)

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
| Part 5 | 12 | 12/12 ✅ |
| Part 6 | 10 | 10/10 ✅ |
| Part 7 | 10 | 10/10 ✅ |
| Part 8 | 13 | 13/13 ✅ |
| Part 9 | 8 | 8/8 ✅ |
| **Total** | **98** | **98/98 ✅** |
