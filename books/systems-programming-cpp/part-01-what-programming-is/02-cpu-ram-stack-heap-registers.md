# Chapter 2 — CPU, RAM, Stack, Heap, Registers

## Learning Objectives

By the end of this chapter you will be able to:

1. Describe the **fetch–decode–execute** cycle and explain why it is the only thing a CPU truly does.
2. Identify the major hardware components a programmer must reason about — registers, the cache hierarchy, RAM — and explain why "memory" is not one thing but a layered system with vastly different speeds.
3. Distinguish the **stack** and the **heap** as two memory disciplines, not two physical regions, and explain when each is appropriate.
4. Read simple x86-64 (or ARM64) assembly and recognize register names, memory references, and the role of the stack pointer and instruction pointer.
5. Explain why "fast code" is mostly "code that the cache and the branch predictor like" — *not* "code that runs fewer instructions."

This chapter is physical. We are temporarily stepping below the abstraction of "the program" and looking at the *machine* that runs it. Once you can see the machine, all the abstractions above it will start to make sense as ways of *managing it*.

---

## 2.1 The Only Thing a CPU Does

A CPU is, despite all its complexity, a machine that does exactly one thing in a loop:

```
loop forever:
    instruction = read bytes at address PC
    PC = PC + size of that instruction
    decode and execute the instruction
```

`PC` is the **program counter** (called `RIP` on x86-64, `PC` on ARM64). It is a single register that holds the memory address of the next instruction to execute. That's it. That's the whole show.

Every fancy thing you have ever seen a computer do — render a video frame, train a neural network, run a kernel, host a database — is *some sequence* of those tiny steps, executed billions of times per second.

This sounds like a humbling oversimplification. It is also literally true. And it is the most important fact in this book, because it tells you something deep about systems:

> **Performance is not about doing complicated things. It is about which simple things you do, in what order, and with what data nearby.**

A modern CPU can execute 5–10 billion instructions per second. The reason your software is slow is almost never that it executes too many instructions. It is that the CPU spent most of its time *waiting* — waiting for memory, waiting for a branch outcome, waiting for a cache miss to resolve. Understanding the machine is understanding what causes the CPU to wait.

---

## 2.2 The Memory Hierarchy: Why "RAM" Is a Lie

When a beginner draws a computer, they draw two boxes: "CPU" and "RAM," with an arrow between them. This picture is wrong. Not slightly wrong — *catastrophically* wrong, because it gives you a model in which all memory accesses are equal. They are not. They differ in speed by **two orders of magnitude**.

The real picture, on a modern Intel/AMD/Apple-Silicon machine, is roughly:

```
                        Approx. latency      Approx. size
CPU registers           ~0 cycles            ~1 KB total
L1 cache                ~4 cycles            32-64 KB per core
L2 cache                ~12 cycles           256 KB - 1 MB per core
L3 cache                ~40 cycles           4-64 MB shared
Main memory (RAM)       ~200-300 cycles      8 GB - 1 TB
SSD                     ~50,000 cycles       100 GB - 10 TB
Network                 millions of cycles   "infinite"
```

A CPU running at 4 GHz executes a cycle in 0.25 nanoseconds. So:

- An L1 hit costs ~1 ns. The CPU barely notices.
- A main-memory miss costs ~75 ns. In that time, the CPU could have executed **300 other instructions** if they didn't depend on the missing data.

This is the central performance fact of modern computing: **the CPU is not the bottleneck. Memory is.** The CPU spends most of its time, in most real workloads, waiting for memory.

This fact reshapes everything. Languages that allocate aggressively (Python, Java) are not slow because of "interpreter overhead" — they are slow because every object is a separate heap allocation, and every traversal of a data structure is therefore a tour of unrelated cache lines, each one a potential ~75 ns stall. Languages and idioms that lay data out contiguously (C, C++, Rust, Zig, Go's structs, NumPy arrays) are fast because the cache *already has* the next item by the time you need it.

This is why "data-oriented design" exists. We will dedicate a chapter to it in Part 2. For now, just internalize: **memory is a hierarchy, not a single thing, and most programs win or lose based on cache behavior, not algorithmic complexity.**

### Cache lines

Memory does not move into the cache one byte at a time. It moves in **cache lines** — typically 64 bytes. When you access `array[0]`, the CPU does not fetch one byte; it fetches the surrounding 64 bytes into L1, on the assumption that you will probably want them too. This is **spatial locality**.

This is why iterating an array of `int` in order is dramatically faster than walking a linked list of `int`. The array is contiguous: 16 ints fit in one cache line, and once you bring in line N, the next 15 iterations are L1 hits. The linked list scatters its nodes across the heap; each node is, with high probability, a fresh cache line, and possibly a fresh page, and possibly a TLB miss. Same algorithmic complexity, 10–100× speed difference.

You did not need to know how the cache works to write the linked list. But you cannot reason about *why* it's slow — and you cannot fix it — without knowing.

---

## 2.3 Registers: The Hands of the CPU

A CPU has a small fixed set of named storage slots called **registers**. On x86-64 there are 16 general-purpose 64-bit registers (`rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`, `r8`–`r15`), plus floating-point/vector registers (`xmm0`–`xmm15` or more), plus special-purpose ones (`rip`, flags). On ARM64 there are 31 general-purpose registers (`x0`–`x30`), plus 32 SIMD registers, plus `sp`/`pc`.

Registers are *not* memory. They are not addressable by pointer. They live inside the CPU itself. They are the *only* place the CPU can directly perform arithmetic. To add two numbers in RAM, the CPU must:

1. Load value A from RAM into a register.
2. Load value B from RAM into a register.
3. Add register-to-register.
4. Store the result register back to RAM.

Every operation you write in C++, Python, or any language ultimately compiles down to this dance: shuffle data between memory and registers, do a tiny operation, shuffle back.

By **calling convention**, certain registers have agreed-upon roles:

- On x86-64 SysV (Linux/macOS), the first six integer arguments to a function go in `rdi, rsi, rdx, rcx, r8, r9`. The return value goes in `rax`. `rsp` is the stack pointer; `rbp` is the (optional) frame pointer.
- On ARM64, the first eight arguments go in `x0`–`x7`; return value in `x0`; `sp` is the stack pointer; `lr` (`x30`) holds the return address.

You do not memorize this. You do, however, want to know *that it exists*, because it explains an enormous amount of low-level behavior: why C function calls are so cheap, why stack traces work, why FFI between languages is hard, why varargs (`printf`) needs special handling.

### Experiment 2.1 — See the registers in action

Save as `add.cpp`:

```cpp
int add(int a, int b) {
    return a + b;
}

int main() {
    return add(2, 3);
}
```

Compile *without* optimization (so the compiler doesn't inline everything away):

```bash
clang++ -O0 -S -masm=intel add.cpp -o add.s
cat add.s
```

You will see something close to (annotated):

```asm
add(int, int):
    push    rbp                  ; save old frame pointer
    mov     rbp, rsp             ; new frame pointer = current stack pointer
    mov     dword ptr [rbp-4], edi   ; spill 'a' (in edi) to stack
    mov     dword ptr [rbp-8], esi   ; spill 'b' (in esi) to stack
    mov     eax, dword ptr [rbp-4]   ; load 'a' into eax
    add     eax, dword ptr [rbp-8]   ; eax += 'b'
    pop     rbp                  ; restore frame pointer
    ret                          ; jump to return address
```

Read it slowly. Then read it again with `-O2`:

```bash
clang++ -O2 -S -masm=intel add.cpp -o add-O2.s
```

And you'll see the entire `add` collapse to:

```asm
add(int, int):
    lea     eax, [rdi + rsi]
    ret
```

Two instructions. The compiler recognized: arguments are already in `rdi`/`rsi`; `lea` (load effective address) will compute `rdi + rsi` into `eax` in one cycle; we're done.

This is the optimizer's job — and the closer you get to the metal, the more visible its effects become. *The C++ you write is not the code that runs.* The compiler is a translator, and a good optimizer is a translator that aggressively rewrites your prose into clearer instructions for the machine.

---

## 2.4 The Stack: A Discipline, Not a Place

Programmers say "the stack" as if it were a specific region of memory. It *is* a region — but the more useful way to think about it is as a **discipline**: a particular pattern of allocating and freeing memory that maps perfectly onto how function calls nest.

The discipline is simple:

- There is a single pointer, the **stack pointer** (`rsp` on x86-64, `sp` on ARM64), that always points to the current top of the stack.
- To "allocate" N bytes on the stack, you subtract N from the stack pointer. (The stack on most architectures grows *downward* — toward lower addresses.)
- To "free" them, you add N back.
- Everything between the original `rsp` and the new `rsp` is yours to use, and is automatically reclaimed when the function returns.

That's the entire stack. It is a region of memory plus a single register plus the convention that pushes and pops nest perfectly with function calls.

### Why the stack is fast

- **Allocation is one instruction** — `sub rsp, 32` allocates 32 bytes. There is no allocator, no free list, no metadata. The cost is essentially zero.
- **Freeing is one instruction** (or implicit on return) — `add rsp, 32` or just `leave; ret`.
- **Locality** — recent stack frames are always in L1 cache. The current call's local variables are essentially free to access.
- **Predictable lifetime** — local variables live exactly until the function returns. The compiler knows this; the CPU's prefetcher knows this.

### Why the stack has limits

- **Size is fixed up front.** A typical stack is 1 MB (Windows, default thread) to 8 MB (Linux main thread). If you `int big[10'000'000];` as a local, you blow the stack and your program crashes with a segfault — *not* an allocation error, because there's no allocator to report one. The stack guard page just faults.
- **Lifetime is tied to the call.** You cannot return a pointer to a local variable. The memory is no longer yours after `return`. C++ will compile it; the consequences will not be pleasant.
- **No dynamic sizing.** The size of every stack object must be known at compile time (with rare exceptions like C99 VLAs).

### What goes on the stack

In a modern compiled language, the stack typically holds:

- The **return address** (where to jump after this function ends).
- **Saved registers** the function needs to restore for its caller.
- **Local variables** (sometimes — many live in registers and never touch memory).
- **Spill slots** — temporaries the compiler couldn't keep in registers.
- **Outgoing function arguments** that don't fit in argument registers.

We will draw stack frames in detail in Chapter 7. For now, internalize this:

> The stack is not "where local variables live." The stack is "the bookkeeping memory for function calls." Local variables live there as a side effect.

---

## 2.5 The Heap: A Discipline of Indeterminacy

The **heap** is the other major memory region. Where the stack is bound to the lifetimes of function calls, the heap is bound to nothing. You ask for memory; you get a block; the block is yours until you give it back.

Concretely:

- The kernel hands the program a few large regions of memory (via `mmap` or `brk`).
- An **allocator** — `malloc`/`free` in C, `new`/`delete` in C++, the GC in Java/Python/Go — manages those regions, splitting them into smaller chunks on demand and tracking which chunks are free.
- When you call `malloc(64)`, the allocator finds a 64-byte free chunk, marks it allocated, and returns its address.
- When you call `free(p)`, the allocator marks that chunk free again and possibly coalesces it with neighbors.

The heap is necessary because not every lifetime fits the stack discipline. Consider:

- An object created in one function and returned to its caller.
- A list whose size is determined at runtime.
- An object shared between multiple owners with unrelated lifetimes.

For these, you need memory that *outlives* the function that created it — i.e., you need the heap.

### What the heap costs

- **Allocation is not free.** Every `malloc` is a small algorithm — find a free chunk of the right size, possibly carve it out of a larger one, update bookkeeping data structures. It can be tens to hundreds of cycles. Compared to a stack allocation (~1 cycle), this is enormous.
- **Fragmentation.** Over time, the free regions become scattered and small. You may have 100 MB free total but no contiguous 1 MB region. Long-running services suffer from this; we'll look at compacting allocators (used by JVMs and Go) in Part 2.
- **Cache unfriendliness.** Heap allocations are scattered across the address space. Walking a tree of heap-allocated objects is a tour of distant cache lines.
- **Lifetime tracking.** Who owns this memory? Who frees it? When? Get it wrong, and you have either a leak (never freed) or a use-after-free (freed too early). This is the single largest source of security bugs in software. The languages that "feel safe" — Java, Python, Go, Rust — all use a different mechanism (GC, ownership) precisely to take this out of the programmer's hands.

### The heap is not one thing

When we say "the heap" we usually mean the C/C++ allocator's region. But:

- A garbage-collected runtime (Java, Go, Python) has its own heap, with its own allocator and its own collector — implemented on top of the OS's `mmap` calls, just like `malloc` is.
- A program can have multiple heaps — for example, a per-thread arena, a per-request arena, a small-object pool, a large-object cache. Game engines and high-performance servers often do this for control.

The heap is a discipline as much as the stack is. The discipline is "managed dynamic lifetimes." Different runtimes implement it differently; the *job* is the same.

---

## 2.6 Visualizing It Together

Here is a process's address space in approximate order, low addresses at top:

```
0x0000000000000000   +------------------------+
                     |   (unmapped, NULL)     |   <-- dereferencing null traps here
                     +------------------------+
                     |   .text  (your code)    |   read+execute
                     +------------------------+
                     |   .rodata (constants)  |   read-only
                     +------------------------+
                     |   .data  (init globals)|   read+write
                     +------------------------+
                     |   .bss   (zero globals)|   read+write
                     +------------------------+
                     |                        |
                     |   HEAP grows down -->  |   malloc/new live here
                     |   v                    |
                     |                        |
                     |   ... lots of unused.. |
                     |                        |
                     |                  ^     |
                     |   STACK grows up | <-- |   function frames live here
                     |                        |
                     +------------------------+
                     |   shared library maps  |   libc, others
                     +------------------------+
                     |   kernel space         |   inaccessible
0xFFFFFFFFFFFFFFFF   +------------------------+
```

Some details vary by OS and ABI (on Linux the heap typically grows up and lives just above `.bss`; the stack grows down from a high address; shared libs live somewhere in between). The exact map is on your machine in `/proc/<pid>/maps` (Linux) or `vmmap <pid>` (macOS).

The picture to keep in mind:

- **Stack and heap grow toward each other.** Historically that meant they could collide; today, they're far apart with guard pages, but the *idea* of the stack at one end and the heap at the other end of the addressable region remains.
- **All of this is per process.** Two running programs do not share any of this. Each one believes it has the whole address space to itself. The OS makes that illusion real via virtual memory (Chapter 3).

---

## 2.7 A First Look at Locality and Why Code Is Slow

Now we put it all together with a small experiment. Consider two programs that compute the sum of an array of `int`s.

**Version A — array (contiguous):**

```cpp
#include <chrono>
#include <cstdio>
#include <vector>

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> v(N, 1);

    auto t0 = std::chrono::steady_clock::now();
    long long sum = 0;
    for (int x : v) sum += x;
    auto t1 = std::chrono::steady_clock::now();

    std::printf("array: %lld in %lld us\n", sum,
        (long long)std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count());
}
```

**Version B — linked list:**

```cpp
#include <chrono>
#include <cstdio>

struct Node { int v; Node* next; };

int main() {
    constexpr int N = 10'000'000;
    Node* head = nullptr;
    for (int i = 0; i < N; ++i) head = new Node{1, head};

    auto t0 = std::chrono::steady_clock::now();
    long long sum = 0;
    for (Node* n = head; n; n = n->next) sum += n->v;
    auto t1 = std::chrono::steady_clock::now();

    std::printf("list:  %lld in %lld us\n", sum,
        (long long)std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count());
}
```

Compile both with `-O2`:

```bash
clang++ -std=c++20 -O2 array.cpp -o array
clang++ -std=c++20 -O2 list.cpp  -o list
./array
./list
```

You will typically see the linked-list version take **5–20× longer** than the array version, despite executing the same number of additions and the same number of loop iterations. The difference is entirely cache behavior: the array fits in cache lines back-to-back; the list scatters its nodes across the heap.

This is the most important performance lesson in the book. Algorithmic complexity (Big-O) tells you how an algorithm scales asymptotically. **Cache behavior tells you what its constant factor is, and the constant factor is often what determines whether your software is usable.** A linked list and an array both have O(N) traversal cost. Only one of them is fast.

---

## 2.8 The CPU Is Not Sequential, Even Though It Looks Sequential

A modern CPU is, internally, *aggressively* parallel. While your program logically does one instruction at a time, the CPU actually:

- **Pipelines** instructions: while instruction 1 is executing, instruction 2 is decoding, instruction 3 is being fetched.
- **Executes out of order**: if instruction 5 doesn't depend on instructions 2–4, the CPU may execute it before them.
- **Speculates on branches**: when it hits an `if`, it *guesses* which way you'll go and starts executing that path; if it guessed wrong, it discards the work.
- **Renames registers**: the 16 architectural registers are mapped onto 100+ physical registers internally to avoid false dependencies.

You do not need to understand any of this to write correct code. You *do* need to understand it to write fast code, because:

- A **branch misprediction** costs ~15-20 cycles. A loop with unpredictable branches runs much slower than one with predictable branches.
- A **data dependency** chain (each instruction depending on the previous) prevents the CPU from using its parallelism. Code that is "wider" — multiple independent streams of computation — runs faster.
- A **cache miss** stalls the entire pipeline if the missing value is needed soon.

We will look at this concretely in Part 8 (CPU Pipelines, Branch Prediction, SIMD). For now, the takeaway: **the CPU is a deeply parallel, speculative machine pretending to be sequential.** The "pretending" is the abstraction your code sees. The "deeply parallel" is what determines how fast your code runs.

---

## 2.9 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| Stack allocation | Limited size, fixed lifetime | Free allocation, perfect cache locality |
| Heap allocation | Slow allocator, fragmentation, lifetime hazards | Arbitrary lifetimes, dynamic sizing |
| Contiguous data (array, struct-of-arrays) | Inflexible, expensive insert/delete in middle | Cache-friendly, vectorizable, predictable |
| Pointer-linked data (linked list, tree of nodes, polymorphic objects) | Cache-hostile, hard to vectorize, allocator pressure | Flexible structure, easy dynamic edits |
| Tons of registers (more parallelism) | More state to save on context switch, larger encoded instructions | More work in flight, fewer memory round-trips |
| Branch-heavy code | Branch mispredictions, hard to vectorize | Expressive control flow |
| Branchless code | Sometimes does "wasted" work | Pipeline-friendly, vectorizable |

Notice the recurring shape: **convenient and flexible** versus **predictable and fast**. You will see this pattern at every level of the stack, in every language, in every framework. Convenience and predictability are in tension. There is no universal winner.

---

## 2.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "RAM access is fast." | RAM access is ~75 ns. The CPU runs at <0.3 ns/cycle. A miss to RAM is *250 lost instructions of work*. |
| "Modern CPUs are so fast my code will be fast too." | Modern CPUs are bottlenecked on memory and branches. You can write code 10–100× slower than necessary by ignoring this. |
| "The stack and heap are different memory." | They're the same hardware memory, with different *disciplines* of management. |
| "`new`/`malloc` is just like a stack push." | `new` runs an algorithm to find space, updates a free list, may take a lock, may call into the kernel. It is hundreds to thousands of times slower than a stack push. |
| "Algorithmic complexity is what matters." | Big-O is necessary but not sufficient. Constant factors driven by cache and branch behavior often dominate in practice. |
| "Programs run sequentially." | Logically yes, hardware-wise absolutely not. The CPU may be running 8 instructions ahead of you. |

---

## 2.11 Exercises

1. **Locality benchmark.** Run the array vs linked-list benchmark from §2.7 on your own machine. Record both numbers. Try N = 10⁴, 10⁵, 10⁶, 10⁷. Plot or list the ratio. At what N does the gap become large? What does that tell you about which level of cache the array still fits in but the linked list doesn't?

2. **Read assembly.** Write three small functions:
   - `int sum_loop(int* p, int n)` — sum n ints
   - `int sum_unrolled(int* p, int n)` — sum n ints, hand-unrolled by 4
   - `int sum_branchless_max(int* p, int n)` — return max, but written without `if`

   Compile each with `-O2 -S -masm=intel`. Look for: how many adds in the loop body? How does the unrolled version compare? Does the branchless `max` use `cmov`?

3. **Touch the stack until it breaks.** Write:

   ```cpp
   void blow(int depth) {
       int filler[1024];   // 4 KB per frame
       filler[0] = depth;
       blow(depth + 1);
   }
   int main() { blow(0); return 0; }
   ```

   Compile with `-O0` and run. It will crash with a segfault. Now, *before* it crashes, what was the approximate stack depth? (Add a `printf` of `depth`, but be aware printf itself uses stack — the number you see may be off by a few.) On your system, what is the default stack size? Try `ulimit -s` (Linux/macOS).

4. **Heap allocator pressure.** Write a program that does 10 million `new int` followed by `delete`. Time it. Now do 10 million `new int` and free them all at the end (or even leak them). Time that too. What does the difference tell you about the cost of `new`/`delete` versus the cost of just `new`?

5. **Stack vs heap object.** Write a function that takes a `std::vector<int>(1000)` by value vs by reference. Compile both with `-O2` and look at the assembly for the call site. What is the difference? Why does `-O2` not magically eliminate it? (Hint: the compiler does not always know the function will not modify the vector.)

6. **Conceptual.** A coworker says "we should use a linked list because we're inserting in the middle a lot." Write three follow-up questions you would ask before agreeing, drawing on what this chapter taught you about cache behavior. (One acceptable answer is "how big is the structure and how often do we *traverse* it versus *insert*?" — but write more.)

7. **Read your own process map.**

   ```bash
   # Linux
   cat /proc/self/maps

   # macOS
   vmmap $$ | head -40
   ```

   Identify: where is the stack? Where is the heap? Where is libc? Where is the executable code segment? Sketch the memory map of your shell. (`$$` is the shell's PID.)

---

## 2.12 What's Next

You now have a picture of the machine: a CPU executing instructions in a tight loop, with a layered memory system that punishes scattered access, with two memory disciplines (stack and heap) that offer different tradeoffs, and a small set of registers that are the only place arithmetic actually happens.

In Chapter 3 we look at the layer that makes all of this *safe and shared* — the operating system. We will see how the kernel creates the illusion that every program has the whole machine to itself, how it switches between programs, how it protects them from each other, and how every "feature" of a modern programming environment (file I/O, threads, networking, even allocating memory) eventually becomes a system call into the kernel.

After Chapter 3, you will have a complete picture of the *substrate* on which all software runs. Then we begin building back up.

---


**[← Previous: Chapter 1 — What Happens When A Program Runs](01-what-happens-when-a-program-runs.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 3 — How Operating Systems Execute Programs →](03-how-os-executes-programs.md)**
