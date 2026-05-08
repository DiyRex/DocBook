# Chapter 7 — Call Stack Deep Dive

## Learning Objectives

By the end of this chapter you will be able to:

1. Read a stack trace fluently — understand what each line means, what's missing, and how to fill in the gaps.
2. Explain how debuggers (`gdb`, `lldb`) and profilers (`perf`, `Instruments`) walk the stack: frame pointer chains vs DWARF unwinding.
3. Explain stack overflow precisely — what triggers it, what diagnostic clues you have, and how to design code that tolerates deep recursion.
4. Describe how **fibers**, **coroutines**, **green threads**, and **goroutines** decouple "logical call stacks" from "OS call stacks" — and why this is the single most important trick in modern concurrency.
5. Connect the call-stack concept to **exception unwinding**, **`setjmp`/`longjmp`**, and **garbage collector stack walking**.

If Chapter 6 taught you what one call does, this chapter teaches you what *all the calls together* look like — and why that single structure, the call stack, is the spine of half the things in modern systems.

---

## 7.1 The Call Stack as a Data Structure

Forget for a moment that the stack lives in memory. Think of it as a logical data structure: a **stack of frames**, one per active function call.

```
top (currently executing)
+-------------------+
| frame for f3      |   <- f3 was called by f2
+-------------------+
| frame for f2      |   <- f2 was called by f1
+-------------------+
| frame for f1      |   <- f1 was called by main
+-------------------+
| frame for main    |
+-------------------+
| frame for         |
| __libc_start_main |   <- the C runtime (Chapter 1)
+-------------------+
| frame for _start  |
+-------------------+
bottom (initial)
```

Each frame contains:
- The **return address** — where to jump back to.
- **Saved callee-saved registers** that the function needs to restore.
- **Local variables** (those that didn't fit in registers).
- **Outgoing arguments** to deeper calls (those that didn't fit in registers).

Every active function call has exactly one frame. The frame is born when the function is entered (the prologue allocates it) and dies when the function returns (the epilogue deallocates it). At any moment, the chain of frames from "currently executing function" down to `_start` is the program's **call stack** at that instant.

When you see a stack trace, you are seeing exactly this chain, walked from top to bottom.

---

## 7.2 Reading a Stack Trace

A real `gdb` stack trace looks like:

```
#0  0x00007f8a91e3c4f1 in __libc_read at ../sysdeps/unix/syscall-template.S:120
#1  0x00007f8a91dca7d3 in _IO_new_file_underflow at fileops.c:519
#2  0x00007f8a91dcb902 in __GI__IO_default_uflow at genops.c:380
#3  0x00007f8a91dbf03f in __GI__IO_getline_info at iogetline.c:67
#4  0x000055d6e8d214a3 in MyParser::readLine() at parser.cpp:42
#5  0x000055d6e8d22b1a in MyParser::parse(std::__cxx11::basic_string<...>) at parser.cpp:88
#6  0x000055d6e8d23012 in main at main.cpp:14
```

Decoding it:

- `#0` is the *innermost* frame — what the program is currently executing. In this case, deep inside libc, blocked on a `read` syscall.
- `#6` is the outermost — `main` itself.
- Each line shows: **frame number, address of the saved return PC, demangled function name, source location** (if debug info is available).
- The progression `#0 → #6` is **inside-out**: who is currently running, who called them, who called *that*, all the way to main.

When debugging, the most common question is: "How did we get here?" The trace tells you in one direction (who called whom). The other direction — "what will happen next?" — requires reasoning about what each frame will do when control returns to it.

### What's missing

Stack traces have several blind spots:

- **Inlined functions** may not appear at all. The compiler folded them into their caller; from the CPU's perspective, there's no frame for them. Modern DWARF can record inlining info, and good debuggers/profilers reconstruct *virtual* frames for inlined code, but the support isn't universal.
- **Tail-called functions** can also vanish: `f` tail-calls `g`, `f`'s frame is replaced by `g`'s, and a stack trace inside `g` shows `g`'s caller as `f`'s caller, not `f`.
- **Optimized-out variables.** Even when a frame is visible, the variables might not be. The compiler may have placed them in registers that have since been overwritten, or eliminated them entirely.
- **Asynchronous stacks.** If your program is in a coroutine or has just resumed from `await`, the OS call stack tells you about *this* resumption, not the logical chain that led to the original async call. (Modern runtimes maintain "async stack" metadata to reconstruct it; not all do.)

This is why "release builds are hard to debug": optimization erases information. The standard trick is to ship `-O2 -g` — full optimization with debug info — and accept that some traces will be partial.

---

## 7.3 How Debuggers Actually Walk the Stack

There are two strategies.

### Strategy 1: Frame pointer chain

If the binary preserves frame pointers (`-fno-omit-frame-pointer`), each frame's `rbp` value is saved at a known location in the next-deeper frame. The walker:

1. Read current `rbp` from the CPU.
2. The 8 bytes at `[rbp + 8]` are the return address. Symbolicate it (look it up in the binary's symbol table to get a function name).
3. The 8 bytes at `[rbp + 0]` are the *previous* `rbp`. Set `rbp` to that.
4. Repeat until `rbp` is null or invalid.

This is fast and simple. It's how `perf` profiling normally works (it samples `rbp` and `rip` thousands of times per second).

### Strategy 2: DWARF unwinding tables

If the binary doesn't preserve frame pointers (the default at `-O1+` on most compilers), the debugger consults **`.eh_frame`** or **`.debug_frame`** sections — *unwind tables* generated by the compiler that say:

> "At PC X, the caller's `rip` is at `[rsp + 8]`, the caller's `rbp` is in register R, the caller's `rsp` is `rsp + 32`."

The unwinder reads the current PC, looks up that PC in the unwind table, applies the recipe, gets the previous frame, repeats.

This is more flexible (works without frame pointers) and produces accurate traces even with aggressive optimization. It is also more expensive — table lookups, possibly multiple memory accesses per frame.

Production profilers usually use frame pointers when available. Debuggers usually use DWARF — they aren't as latency-sensitive.

---

## 7.4 What Stack Overflow Actually Looks Like

The stack has a fixed maximum size (8 MB on Linux by default; check with `ulimit -s`). The kernel allocates the stack region with a **guard page** below it. The guard page has *no* permissions: any access to it traps.

The mechanism:

1. Your function's prologue does `sub rsp, 8000` (allocating 8000 bytes of locals).
2. If this is the call that exceeds 8 MB, the new `rsp` is now in the guard page.
3. When the function tries to write to a local — e.g., `mov [rsp+0], rax` — the write fault.
4. The CPU raises a page fault. The kernel sees the access is to a guard page. It delivers `SIGSEGV` to the process.

That's the full story. There is no "stack manager" that sees the overflow coming. The hardware is the watchdog.

The diagnostic clues:

- **A SIGSEGV at a stack address.** Look at the faulting address: if it's just below the bottom of the stack region, you've overflowed.
- **A very deep stack trace, often with the same function repeated.** Recursion is the usual culprit.
- **Allocating a huge local.** `int big[10'000'000];` can blow the stack on entry, before any recursion.

### Mitigations

- **Don't recurse deeply on stacks.** Convert recursion to iteration with an explicit data structure (e.g., a vector-based DFS instead of recursive DFS).
- **Use `ulimit -s unlimited`** or the `pthread_attr_setstacksize` API to allocate a bigger stack for known-deep workloads.
- **Use a "stack-checking" function attribute** to insert probes that check before huge allocations.
- **Use coroutines/fibers/segmented stacks** for huge concurrency where you don't want to commit 8 MB per stack.

The last point is the bridge to our next big topic: decoupling the *logical* stack from the OS stack.

---

## 7.5 Coroutines, Fibers, and Green Threads: Many Stacks Per Thread

A traditional OS thread has one stack — fixed, big (8 MB), allocated by the kernel. If you want 1000 concurrent things and use one thread each, you commit at least 8 GB of address space (and modest amount of RAM, lazily) just to stacks.

This is fine for hundreds of threads. For tens of thousands or millions of concurrent things, it doesn't scale. The solution is to decouple the *concept* of a call stack from a kernel thread.

### Coroutines

A **coroutine** is a function that can pause itself, return control to its caller, and later resume from where it paused — with its locals and call stack preserved.

```cpp
// C++20 coroutine sketch
task<int> get_user_id(std::string name) {
    auto db = co_await connect();         // suspend until DB connects
    auto row = co_await db.query(name);   // suspend until query result
    co_return row.id;
}
```

Each `co_await` is a suspension point. When suspended, the coroutine's state — its locals, its current "instruction" — must be preserved somewhere. C++20 coroutines store this state on the **heap** (in a *coroutine frame*), not on the call stack. Resuming a coroutine means jumping back into the saved state.

This means: thousands of coroutines can share *one* OS thread. Each has its tiny coroutine frame on the heap; the OS thread's single 8 MB stack hosts whatever coroutine is currently running. When one suspends, the runtime picks another and resumes it.

### Stackful coroutines / fibers

An older, simpler model: a **fiber** has its own stack — but allocated by user code, much smaller than the OS thread default (often 64 KB). Switching between fibers means saving the registers (including `rsp`) of one and restoring those of another.

Boost.Context, Microsoft's `CreateFiber`, and many game engines use this model. Cost: ~50–100 ns per switch (compared to 1–10 µs for an OS thread context switch — a 10–100× difference).

### Green threads / goroutines

Go's **goroutines** are stackful coroutines with growable stacks. Each goroutine starts with a small stack (typically 2 KB) and grows on demand by allocating a new, larger stack and copying. The Go runtime schedules goroutines onto a smaller pool of OS threads (M:N scheduling) — typically one OS thread per CPU core, multiplexing thousands of goroutines.

The result: spawning a goroutine costs microseconds, not milliseconds; using 100,000 goroutines is normal; the system stays responsive.

### Why this matters for your mental model

Understanding "stack as a logical concept, separable from OS stack" unlocks understanding of:

- **Async/await in any language.** Each pending future has a saved state somewhere; the runtime manages a queue of ready futures and runs them on a worker pool.
- **Generators (Python, JS).** Each `yield` is a coroutine suspension. The generator object holds the saved state.
- **Erlang/BEAM processes.** Millions of "processes" per node, each with its own tiny stack, scheduled by the BEAM VM. Same trick.
- **Game engine task systems.** Tasks suspend on dependencies, resume when satisfied. Same trick.

The pattern is universal: **when you need many concurrent units of work, you stop pretending each one needs an OS-scale stack and OS-scale context switch.** Then you build a userspace scheduler. Every modern concurrency abstraction is a variation on this.

We will dedicate Part 5 to concurrency in detail. For now, internalize: **the call stack is a logical structure. Where it lives — on the OS-allocated stack, in heap-allocated coroutine frames, in fiber stacks — is an implementation choice.**

---

## 7.6 Exception Unwinding Revisited

Throwing an exception in C++ unwinds the call stack, frame by frame, running destructors at each level. Concretely:

1. `throw e` constructs the exception object on a special heap-managed area.
2. The runtime calls `__cxa_throw`, which looks at the unwind tables.
3. For each frame from innermost outward:
   - The unwind table tells the runtime: "in this frame, this PC range, run cleanup procedure C." The procedure runs all destructors for objects whose lifetimes are ending.
   - If the table says "this frame catches exceptions of this type," control transfers to the catch handler. Done.
   - Otherwise, the frame is destroyed (locals destructed, frame popped), and we continue to the next frame.
4. If we reach the bottom of the stack with no handler: `std::terminate()` is called. The default behavior is to abort.

Notice how this *uses* the same call-stack structure as normal returns. Exception handling isn't a parallel mechanism; it's another way of walking the stack, with a different rule about who gets control.

This is also why exceptions can cross many frames "for free" semantically but expensively performance-wise: the runtime must walk the unwind tables for every frame in between, running destructors as it goes.

Languages without zero-cost exceptions (Python, Java in some implementations) use simpler mechanisms — typically, every function call records "I'm in a try" state, and an exception just unwinds via normal returns with a flag. Cheaper to throw, more expensive in the *non-throwing* path.

---

## 7.7 `setjmp` / `longjmp`: Manual Stack Unwinding

Before C++ exceptions, C had `setjmp` and `longjmp`:

```c
#include <setjmp.h>
jmp_buf env;

void deep() {
    longjmp(env, 1);    // jumps back to setjmp, returning 1
}

int main() {
    if (setjmp(env) == 0) {
        printf("first call\n");
        deep();
        printf("never reached\n");
    } else {
        printf("longjmp returned us here\n");
    }
}
```

`setjmp` saves the current registers (including stack pointer, frame pointer, `rip`) into `env`. `longjmp` restores them — instantly winding back the stack to the moment `setjmp` was called.

This is dangerous because, unlike C++ exceptions, `longjmp` does *not* run destructors. Anything that needed cleanup is leaked. For this reason, `setjmp`/`longjmp` is generally avoided in C++ — exceptions or explicit error returns are used instead. But the *mechanism* — saving and restoring register state to "teleport" the program to a saved call-stack position — is the same trick that powers fibers, coroutines, and exception unwinding. They're variations on a theme.

---

## 7.8 Garbage Collectors and Stack Walking

A garbage collector needs to know which objects are in use. The set of "in-use" objects is the **transitive closure** from a set of **roots**. The roots include:

- Global pointers (in `.data`/`.bss`).
- Pointers in CPU registers.
- **Pointers on the call stack.**

So a GC has to walk the call stack and find every pointer in every active frame. This is non-trivial:

- The collector needs to know, for each frame, *which slots contain pointers* and which contain raw integers (you don't want to mistake a random integer for a pointer and follow it). This is **precise GC**.
- Or it can treat anything that *looks* like a pointer as a pointer — **conservative GC**. (Boehm's GC is conservative; it occasionally over-retains because of false positives but doesn't need any cooperation from the compiler.)
- It needs to walk the stack at "safe points" — places where the program has agreed to be in a consistent state (most languages: at function entry, at allocation, at backward branches in loops). At a safe point, the runtime suspends all mutator threads, walks their stacks, finds roots, marks reachable objects, possibly compacts, then resumes.

This is one of the biggest "hidden costs" of GC'd languages: every thread must cooperate to be stopped, every frame must be walkable, every pointer must be findable. Compiled languages with GC (Go, Java, C#) generate **stack maps** — per-PC tables saying "at this instruction, slots 0-3 of the frame contain pointers; slot 4 is an int" — analogous to the unwind tables for exceptions.

When you read about "GC pauses," you're reading about: stop all threads → walk all stacks → mark all reachable → unstop. The "stop the world" phase is, fundamentally, a stack-walking operation.

---

## 7.9 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| OS-thread stack (8 MB) | Fixed, large; doesn't scale to millions | Simple, fully isolated; no runtime support needed |
| Goroutine / green-thread stack (small + grow) | Stack grow may copy; runtime complexity | Massive concurrency, cheap creation |
| Stackful fiber (small fixed) | Risk of overflow if you allocate too small | Cheap context switch, simple semantics |
| Coroutines (heap frame) | Per-coroutine heap allocation; harder profiling | Tightest packing of concurrent state; works without runtime stacks |
| Frame pointers | One register, two instructions/call | Cheap stack walking |
| DWARF unwinding | Larger binary, slower unwind | No runtime register cost; works with optimization |
| Exception-based control flow | Slow throw path; binary size from tables | Decoupled error handling; automatic cleanup via destructors |
| `setjmp`/`longjmp` | No destructor support; UB across C++ objects | Tiny, fast (~20 ns to longjmp); useful for C-style cleanup |

---

## 7.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "A stack trace shows all the functions that were called." | Inlined, tail-called, and optimized-out frames may not appear. Async stacks are particularly lossy. |
| "Stack overflow is detected by a check." | It's detected by hardware: a fault on the guard page. No software check is involved. |
| "Coroutines are just syntactic sugar for callbacks." | Coroutines preserve the call stack's structure: locals, scoping, RAII, exception flow all work normally. Callbacks fragment all of that. |
| "Goroutines are OS threads." | Goroutines are M:N — scheduled by the Go runtime onto a small pool of OS threads. Most goroutines are not bound to any particular OS thread. |
| "Exception throwing has no cost in non-throwing code." | The non-throwing path has no cost. The throwing path is expensive. The binary still contains the unwind tables, which add to size. |
| "GC pauses are a Java thing." | Every GC'd language pauses (or does concurrent collection). Go has them too — typically very short. The mechanism (stack walking) is universal. |

---

## 7.11 Exercises

1. **Anatomy of a real trace.** In any program you have, intentionally trigger a crash (e.g., `*(int*)0 = 1;` or recurse infinitely). Inspect the resulting core dump (`gdb ./prog core`) or use `bt` in a live debugger. Identify: (a) the innermost frame, (b) the outermost frame, (c) any frames you didn't expect (libc, libstdc++, the runtime). Are there any inlined frames marked? Are any visible at all?

2. **Force inlining off and compare traces.** Take a small program, mark some functions `[[gnu::noinline]]`. Compile with and without. Trigger a stack trace from inside one of those functions. Compare: how does the trace change?

3. **Watch a stack overflow.**
   ```cpp
   void rec() { int big[1024]; big[0] = 1; rec(); }
   int main() { rec(); }
   ```
   Compile `-O0 -g`. Run under `gdb`. When it crashes, `bt` until you can't bt any more. How deep did it go? Compute: stack size / per-frame size. Does it match?

4. **Tail-call recursion.**
   ```cpp
   int rec(int n, int acc) { return n == 0 ? acc : rec(n - 1, acc + n); }
   int main() { return rec(1'000'000, 0); }
   ```
   Compile `-O0`. What happens? Now `-O2`. What changes? Why? Inspect the assembly to confirm.

5. **Build a coroutine state machine by hand.** Write a function that, manually, simulates a coroutine: it takes a "state" parameter, switches on it, performs partial work, returns a "yielded value plus next state." Now write the matching driver. Compare to a `co_await` version (or to a Python `yield`-based generator). Notice: coroutine syntax is hiding *exactly this hand-written state machine* — the compiler is generating it for you.

6. **DWARF inspection.**
   ```bash
   objdump --dwarf=frames ./your-program | head
   ```
   Read a few entries. Each is an unwind recipe for a PC range. You don't need to decode every detail; just verify they exist and are surprisingly compact.

7. **Conceptual.** Describe to a colleague: "Goroutines, async/await, and exception unwinding are all variations on the same idea." What is that idea? Connect it to what this chapter taught you about the call stack as a logical structure.

---

## 7.12 What's Next

You have Chapter 1's loader, Chapter 2's CPU and memory, Chapter 3's OS, Chapter 4's compilation pipeline, Chapter 5's process layout, Chapter 6's function call mechanics, and now Chapter 7's full call-stack picture. There's one piece of plumbing left: **pointers** — the typed view of "addresses in memory" that ties everything together and is the source of more confusion than any other feature in C++.

Chapter 8 takes pointers seriously: not as a syntax curiosity, but as the bridge between Chapter 5's regions and Chapter 6's frames. After it, you will see pointers as references to specific bytes in specific regions, with specific lifetime constraints — and you'll never write `int*` casually again.

Then Chapters 9 and 10 close out Part 1 conceptually: why abstractions exist, and what programming languages actually do underneath the surface.

---


**[← Previous: Chapter 6 — Function Calls Internally](06-function-calls-internally.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 8 — Pointers As Memory Addresses →](08-pointers-as-memory-addresses.md)**
