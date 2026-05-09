# Part 2 — Memory & Execution Foundations

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Part 1 built the substrate: how programs become processes and how source becomes machine code. Part 2 zooms in on the single dimension that dominates real-world performance and reliability — **memory**. By the end you'll have a concrete mental model for the stack and heap, manual memory management and its bug families, ownership and lifetimes, RAII, fragmentation, the cache hierarchy, data-oriented design, and how four major language runtimes (C++ with smart pointers, Python, Go, PHP/Laravel) actually handle memory.

After Part 2, you can read most production C++ code with confidence and reason about why the same algorithm performs differently in two different languages.

---

## Chapters

11. **[Stack vs Heap Internals](01-stack-vs-heap-internals.md)**
    Two disciplines, not two places. Stack frames, heap allocators (free lists, sbrk vs mmap, jemalloc/tcmalloc), thread-local arenas, allocation cost reality.

12. **[Manual Memory Management](02-manual-memory-management.md)**
    `malloc`/`free`, `new`/`delete`, placement new, alignment. The four bug families and how AddressSanitizer catches them.

13. **[Ownership Models](03-ownership-models.md)**
    Five patterns across C, C++, Rust, and GC'd languages. Move semantics in C++ concretely. Linear and affine types briefly.

14. **[Lifetimes](04-lifetimes.md)**
    Storage duration vs object lifetime. The four storage classes. Temporary objects, lifetime extension, dangling references.

15. **[RAII Explained Deeply](05-raii-explained-deeply.md)**
    Resource lifetime tied to object lifetime. Stack unwinding under exceptions. The Rule of Zero/Three/Five.

16. **[Memory Fragmentation](06-memory-fragmentation.md)**
    External vs internal fragmentation. Why long-running services drift. Pool and arena allocators. Why C++ can't compact and GCs can.

17. **[Caches and CPU Locality](07-caches-and-cpu-locality.md)**
    The memory hierarchy. Cache lines as the unit of transfer. Spatial and temporal locality. Why `std::list` is almost always wrong.

18. **[Data-Oriented Design](08-data-oriented-design.md)**
    Designing data layout first. AoS vs SoA. ECS in game engines. SIMD friendliness. When DoD wins and when it hurts.

19. **[Smart Pointers Internals](09-smart-pointers-internals.md)**
    `unique_ptr`, `shared_ptr` control blocks, `weak_ptr`, `enable_shared_from_this`, `make_shared` vs raw `new`, the cost of atomic refcounts.

20. **[Reference Counting](10-reference-counting.md)**
    The simplest "automatic" scheme. Atomic vs non-atomic counts. The cycle problem. CPython, Swift ARC, Objective-C ARC.

21. **[Garbage Collection Internals](11-garbage-collection-internals.md)**
    Roots, mark-sweep, copying, generational, tri-color marking, concurrent and incremental collectors. JVM G1/ZGC, Go GC, V8.

22. **[How Python, Go, and PHP Actually Manage Memory](12-how-languages-manage-memory.md)**
    Four runtimes side by side. Implications for application code. Common per-language performance mistakes.

---

## How to Use Part 2

- **Read in order.** Each chapter assumes the vocabulary of the previous ones (especially Ch 11–15).
- **Run the experiments.** Many chapters include benchmarks comparing two layouts or two allocation strategies. Running them on your own hardware is where the model becomes intuition.
- **Cross-reference Part 1.** The CPU/cache discussion in Ch 17 builds directly on Ch 2; the layout discussion in Ch 14 builds on Ch 5.

> **Next: Part 3 — Understanding Abstractions** *(coming soon)*
