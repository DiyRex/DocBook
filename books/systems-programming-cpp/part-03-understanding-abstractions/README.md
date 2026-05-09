# Part 3 — Understanding Abstractions

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Part 1 introduced abstractions as contracts that separate *what* from *how*. Part 3 takes the mechanisms apart. By the end you will know what an abstraction is built out of, what encapsulation actually buys you, what classes and objects look like in memory, when inheritance is the right answer and when composition is, what an interface really promises, how polymorphism is implemented across four mechanisms, what a vtable is byte-for-byte, when static dispatch beats dynamic, and why "Dependency Injection" is just two words: pass dependencies in.

After Part 3, the architecture chapters in Part 4 have all the vocabulary they need.

---

## Chapters

23. **[What Is An Abstraction (Mechanically)](01-what-is-an-abstraction.md)**
    The three parts of every abstraction: interface, contract, implementation. Mechanism vs policy. The boundary.

24. **[Encapsulation From First Principles](02-encapsulation.md)**
    Invariants, local reasoning, access specifiers as documentation, tell-don't-ask, opaque pointers across modules.

25. **[Why Classes Exist](03-why-classes-exist.md)**
    Bundling data with the operations that maintain its invariants. Aggregate vs encapsulated. When classes are the wrong tool.

26. **[What Objects Are In Memory](04-what-objects-are-in-memory.md)**
    Padding and alignment; vptr layout when virtuals appear; multiple inheritance; identity vs value.

27. **[Composition vs Inheritance](05-composition-vs-inheritance.md)**
    What inheritance actually does mechanically. The fragile base class problem. When inheritance is the right tool.

28. **[Interfaces and Contracts](06-interfaces-and-contracts.md)**
    Pure-virtual interfaces, concepts (compile-time interfaces), structural vs nominal typing, contracts beyond types.

29. **[Polymorphism Internals](07-polymorphism-internals.md)**
    The four polymorphisms: ad-hoc, parametric, subtype, structural. How each is implemented and what it costs.

30. **[Virtual Tables](08-virtual-tables.md)**
    Vtable + vptr step by step. Inheritance, overriding, virtual destructors, multiple inheritance, RTTI, devirtualization.

31. **[Static vs Dynamic Dispatch](09-static-vs-dynamic-dispatch.md)**
    Where the decision happens. CRTP, std::variant + std::visit, type erasure, std::function. Costs and when to choose.

32. **[Dependency Injection From Scratch](10-dependency-injection-from-scratch.md)**
    Pass dependencies in. Three forms of injection. When you want a container. Service locator anti-pattern.

---

## How to Use Part 3

- **The chapters build on each other.** Read in order — Ch 30 (vtables) presumes Ch 29 (polymorphism), which presumes Ch 28 (interfaces).
- **Use godbolt.org for the mechanism chapters.** Inspect the actual codegen for the patterns in Ch 26, 29, 30, 31. The mental model becomes intuition only when you see the assembly.
- **Resist over-applying patterns.** Each chapter ends with "When this is the wrong tool" or equivalent — those are the most important sections.

> **Next: Part 4 — Architecture Thinking** *(coming soon)*
