# Part 6 — Languages & Runtime Design

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Part 1 said every language is a bundle of design decisions. Part 6 takes the most consequential of those decisions apart: how source becomes execution (compiled, interpreted, JITed), how four major runtimes (CPython, Go, JVM, V8/.NET) actually execute code, what GC tradeoffs each picked, why Rust ownership exists, why C++ stays difficult — and a closing synthesis that gives you a transferable framework for evaluating any language.

By the end, you can read a new language's docs and place it on the design grid in your head: typing, memory model, execution mode, concurrency model, mutability default. That framework is the whole point of Part 6.

---

## Chapters

58. **[Compiled vs Interpreted](01-compiled-vs-interpreted.md)** — the spectrum, not a dichotomy.
59. **[JIT Compilation](02-jit-compilation.md)** — tiered compilation, profile-guided optimization, deoptimization.
60. **[How Python Executes Code](03-how-python-executes-code.md)** — bytecode, ceval, GIL, where slowness comes from.
61. **[Why Python Uses `__name__ == "__main__"`](04-why-python-uses-name-main.md)** — the import-side-effect problem and how other languages solve it.
62. **[How Go Builds Binaries](05-how-go-builds-binaries.md)** — static linking, cross-compilation, cgo's costs.
63. **[How the JVM Works](06-how-the-jvm-works.md)** — class loading, tiered JIT, escape analysis, startup cost.
64. **[Runtime GC Tradeoffs Across Languages](07-runtime-gc-tradeoffs.md)** — the choices JVM, Go, V8, .NET, CPython, BEAM made.
65. **[Why Rust Ownership Exists](08-why-rust-ownership-exists.md)** — memory safety + no GC + concurrency at compile time.
66. **[Why C++ Is Difficult](09-why-cpp-is-difficult.md)** — backwards compat, paradigms, UB, the build system tax.
67. **[Language Design Tradeoffs (Part 6 Synthesis)](10-language-design-tradeoffs.md)** — the framework for reading any language.

---

## How to Use Part 6

- **Apply the framework as you read.** Each chapter examines a real runtime; the closing chapter (67) gives you the framework. Try placing one language you know on each axis after each chapter.
- **Don't skip Ch 67.** It synthesizes everything; without it, the chapters feel like nine independent essays.

> **Next: Part 7 — Building Real Systems** *(coming soon)*
