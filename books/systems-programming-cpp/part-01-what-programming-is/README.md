# Part 1 — What Programming Actually Is

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

The substrate of all software. By the end of Part 1 you will have an unbroken mental model from the program counter all the way up to source code: how programs become processes, what the CPU is actually doing, how the OS arranges multiple programs to share one machine, how source code becomes machine code, where everything lives in memory, and how function calls compose into the call stack you see in every debugger.

After Part 1, every part that follows is a different lens on the same substrate.

---

## Chapters

1. **[What Happens When A Program Runs](01-what-happens-when-a-program-runs.md)**
   The kernel, loader, dynamic linker, and C runtime startup — the four actors that all run before your `main()`. Tracing `./hello` from the shell to first instruction.

2. **[CPU, RAM, Stack, Heap, Registers](02-cpu-ram-stack-heap-registers.md)**
   The fetch–decode–execute loop. Why "memory" is a hierarchy, not a single thing. The stack and heap as disciplines, not places. Why cache behavior dominates performance.

3. **[How Operating Systems Execute Programs](03-how-os-executes-programs.md)**
   Processes, virtual memory, kernel vs user mode, system calls, scheduling, and the four "everything is a..." abstractions that organize all of Unix.

4. **[Machine Code, Assembly, Compilers, Interpreters](04-machine-code-assembly-compilers-interpreters.md)**
   The full pipeline from source to machine code, what an assembler does, what optimization actually looks like, what an interpreter and a JIT add — and why every language has a runtime if you look hard enough.

5. **[Memory Layout Of A Process](05-memory-layout-of-a-process.md)**
   `.text`, `.rodata`, `.data`, `.bss`, the heap, the stack, shared library mappings, thread-local storage. Where every variable lives based on how it was declared.

6. **[Function Calls Internally](06-function-calls-internally.md)**
   The x86-64 SysV calling convention, ARM64 conventions, stack frames, register saving rules, prologue and epilogue, inlining, tail calls, leaf functions, variadic functions, exception unwinding briefly.

7. **[Call Stack Deep Dive](07-call-stack-deep-dive.md)**
   Reading stack traces, how debuggers walk the stack, stack overflow mechanics, and the unifying idea that links coroutines, fibers, goroutines, async/await, exception unwinding, and GC stack walking.

8. **[Pointers As Memory Addresses](08-pointers-as-memory-addresses.md)**
   A pointer as "typed view of an address." Pointers vs references vs iterators vs smart pointers. The four classic hazards. Why "no-pointer" languages have hidden pointers. Identity vs value.

9. **Why Abstractions Exist** *(coming soon)*

10. **What Languages Actually Do** *(coming soon)*

---

## How to Use Part 1

- **Do the experiments.** Each chapter has 4–7 experiments using real tools (`strace`, `objdump`, `gdb`, `cat /proc/.../maps`, etc.). Reading without doing produces shallow understanding; experiments build the mental model.
- **Do at least one exercise per chapter.** They are designed to expose the gap between "I read this" and "I can explain this."
- **Don't skip ahead.** Later parts assume the substrate is solid. If a Part 2 chapter feels confusing, the chapter you skipped in Part 1 is usually the cure.

> **Next: Part 2 — Memory & Execution Foundations** *(coming soon)*
