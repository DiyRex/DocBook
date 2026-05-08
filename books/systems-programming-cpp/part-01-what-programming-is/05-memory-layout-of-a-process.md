# Chapter 5 — Memory Layout Of A Process

## Learning Objectives

By the end of this chapter you will be able to:

1. Identify every major region of a running C++ process — `.text`, `.rodata`, `.data`, `.bss`, the heap, the stack, shared library mappings, thread-local storage — and explain *why each one exists separately*.
2. Predict where a given variable lives based on how it was declared.
3. Read `/proc/<pid>/maps` (Linux) or `vmmap` (macOS) and identify each region.
4. Explain **alignment**, **padding**, and how a struct's fields are laid out in memory — and why that matters for performance and correctness.
5. Distinguish a **virtual address** from a **physical address** from an **offset within a region**, and never confuse them again.
6. Recognize the typical *failure modes* that come from each region: stack overflow, heap corruption, NULL deref, segfault from writing `.rodata`, executable-stack attacks, etc.

This chapter is the consolidation of Chapters 1–4 from the angle of memory. Once you can mentally locate any variable in this layout, debugging memory bugs and reasoning about performance both become dramatically easier.

---

## 5.1 The Mental Picture We're Building

Take any C++ program. Pause it in the middle of execution. The kernel knows, for that process, **every byte of memory the process can legally touch**, organized as a list of contiguous regions called **mappings** or **VMAs** (virtual memory areas). Each mapping has:

- A virtual address range (`start` to `end`, exclusive).
- A permission set (`r`, `w`, `x` in any combination).
- A backing source (a file, swap, or "anonymous" zero-fill).
- Flags (private vs shared, copy-on-write, locked, etc.).

Everything else in this chapter is detail about *which* mappings exist for a typical process and *why*.

A simplified picture (Linux, x86-64, with ASLR enabled):

```
HIGH ADDRESS  0x7fff_ffff_ffff   +-----------------------------+
                                 | Kernel space (inaccessible) |
                                 +-----------------------------+
                                 |   [vsyscall/vdso]            |   (kernel-provided fast path)
                                 +-----------------------------+
                                 |    [stack] (rw-)             |   grows down
                                 |       v                      |
                                 +-----------------------------+
                                 |   ... unused ...             |
                                 +-----------------------------+
                                 |   /lib/libc.so.6  (r-x)      |
                                 |   /lib/libc.so.6  (r--)      |
                                 |   /lib/libc.so.6  (rw-)      |
                                 |   ...other shared libs...    |
                                 +-----------------------------+
                                 |   ... unused ...             |
                                 +-----------------------------+
                                 |   [heap] (rw-)               |   grows up
                                 |       ^                      |
                                 +-----------------------------+
                                 |   .bss   (rw-)   uninit globals
                                 |   .data  (rw-)   init globals
                                 |   .rodata (r--)  string lits, const tables
                                 |   .text  (r-x)   your code
                                 +-----------------------------+
                                 |   (low memory, mostly       |
                                 |    unmapped, including 0x0) |
LOW ADDRESS    0x0000_0000_0000  +-----------------------------+
```

Read it slowly. We will now go region by region.

---

## 5.2 The Code Region: `.text`

Your machine code lives in `.text`, mapped **read + execute**, **not write**. This is critical:

- **Not writable** so a buggy or malicious memory write can't modify the running program.
- **Executable** so the CPU is allowed to fetch instructions from these pages. Pages without the X bit fault if the CPU's program counter ever lands on them — this is the **NX bit / DEP** (Data Execution Prevention), a defense against code-injection exploits.

The `.text` mapping is typically file-backed by your executable. The kernel doesn't copy your code into RAM up front; it sets up the mapping, and the first instruction fetch faults the relevant page in. Other processes running the same binary share the same physical pages — code is read-only and identical, so why duplicate it?

Shared libraries (`libc.so`, `libstdc++.so`, etc.) get their own `.text` mappings, also read-execute. On a busy server with many copies of `nginx` running, all of them share *one* physical copy of `nginx`'s `.text` and *one* of `libc`'s. This is a major reason dynamic linking exists.

### Experiment 5.1 — find your code

```cpp
#include <cstdio>
int demo() { return 42; }
int main() {
    std::printf("address of demo()  = %p\n", (void*)&demo);
    std::printf("address of main()  = %p\n", (void*)&main);
    return 0;
}
```

Run it twice. You'll see different addresses (ASLR), but in each run, `demo` and `main` will be close together — both inside `.text`. Now:

```bash
# Linux
./a.out & cat /proc/$!/maps | grep -E 'r-xp.*a.out'; wait
```

The address you printed will fall inside one of those `r-xp` ranges. You have just located your function in your address space.

---

## 5.3 The Read-Only Data Region: `.rodata`

`.rodata` (read-only data) holds **constants** — values the compiler knows will never be written:

- String literals: `"hello, world"`.
- Constant tables: lookup tables for switch statements, virtual function tables (vtables), `const` global arrays.
- C++ `constexpr` values that need addresses.

Permissions: **read only, no execute, no write.**

What this means concretely:

```cpp
char* p = "hello";   // legal in C, deprecated/illegal in modern C++
p[0] = 'H';          // segfault — writing to .rodata
```

The string `"hello"` lives in `.rodata`. Trying to modify it triggers a fault from the MMU before any kernel code runs — the hardware itself enforces it.

Same effect with `const`:

```cpp
const int table[] = {1, 2, 3, 4};
*(int*)&table[0] = 99;   // undefined behavior; usually segfaults
```

`.rodata` exists for two reasons:

1. **Safety.** Constants shouldn't be writeable; making them physically read-only catches bugs and makes some attacks impossible.
2. **Sharing.** Like `.text`, multiple processes running the same binary share the same physical pages of `.rodata`.

### Experiment 5.2 — addresses tell you the region

```cpp
#include <cstdio>
const char* lit = "hi";
int main() {
    int local = 1;
    static int global_init = 2;
    static int global_zero;
    int* heap = new int(3);

    std::printf("string literal: %p\n", (void*)lit);
    std::printf("global_init   : %p\n", (void*)&global_init);
    std::printf("global_zero   : %p\n", (void*)&global_zero);
    std::printf("local (stack) : %p\n", (void*)&local);
    std::printf("heap          : %p\n", (void*)heap);
    std::printf("function code : %p\n", (void*)&main);
    return 0;
}
```

Run this. You will see something like:

```
string literal: 0x55a0c3c2200a   <- .rodata
global_init   : 0x55a0c3c25018   <- .data
global_zero   : 0x55a0c3c2502c   <- .bss
local (stack) : 0x7ffd35c19a44   <- stack (very high address)
heap          : 0x55a0c5e8b2b0   <- heap
function code : 0x55a0c3c21155   <- .text
```

Notice the patterns:

- `.text`, `.rodata`, `.data`, `.bss`, `heap` all live at *similar* (executable-relative) addresses — they were laid out by the loader near the binary's load address.
- The **stack** is at a totally different address — way up high, often around `0x7fff_xxxx_xxxx`.
- This is one of the most reliable fingerprints. A pointer in the `0x7fff…` range? Stack. A pointer near the executable's load address? Static data or heap.

This experiment is the single most useful diagnostic for "where does my variable live?" — and you can do it in five minutes for any program.

---

## 5.4 The Initialized-Data Region: `.data`

`.data` holds globals and statics that have an explicit non-zero initializer:

```cpp
int counter = 42;          // -> .data
static const int K = 7;    // may go to .data or .rodata depending on usage
```

Permissions: **read + write.** No execute. (You don't want your data executable.)

The bytes are stored on disk in your binary — open the binary in a hex editor and you'd find `42` literally there. The loader maps that file region into memory; modifications stay in the process's private copy (copy-on-write the moment you write).

`.data` is small in well-designed programs. Most large structures are allocated dynamically, not as globals; most globals are zero-initialized (`.bss`).

---

## 5.5 The Zero-Initialized Region: `.bss`

`.bss` ("Block Started by Symbol" — historical name from the IBM 704 era) holds globals and statics that are *zero* (or default-) initialized:

```cpp
int big[1000000];          // -> .bss
static char buffer[4096];  // -> .bss
```

Permissions: **read + write.**

The defining property: **`.bss` does not occupy space in the binary.** The ELF/Mach-O header records "this region is N bytes long; map it as zero-fill." The loader allocates virtual address space, points it at the kernel's shared zero page, and zero-fills only when you write.

This is why a program with `int huge[1'000'000];` as a global has the same on-disk size as one without it — but uses 4 MB of RAM (lazily) when run.

### Misconception: ".bss is uninitialized memory"

No — in C++ (and C with file-scope statics), `.bss` is *zero-initialized*. The standard guarantees globals start as zero. The kernel's mechanism (zero pages, demand zero-filling) is *how* it makes that guarantee cheap. The "uninitialized" name is historical.

Stack-local variables, on the other hand, *are* genuinely uninitialized in C/C++ unless you give them an initializer. Reading them is undefined behavior.

```cpp
int local;       // garbage in C++; whatever was on the stack last
int local = 0;   // zero
static int s;    // zero (in .bss)
```

This asymmetry — globals zero by default, locals not — is the source of countless bugs and one of Rust's design responses (no uninitialized values, ever).

---

## 5.6 The Heap

The **heap** is a region of memory the program asks the OS for, dynamically grows, and manages with an allocator. From the kernel's perspective, it's just one or more `mmap`-backed regions plus possibly a "data segment" extended via `brk()`.

From your code's perspective, when you call `malloc(n)` (or `new`), you get back a pointer into the heap. The C library's allocator (glibc's `ptmalloc`, jemalloc, mimalloc, tcmalloc — many implementations) chose where; you don't.

Heap allocations are **not contiguous** with each other. Each allocation can be anywhere in the heap region; the allocator tries (with varying success) to keep them packed but has to handle free/realloc/free patterns that fragment the space.

### Anatomy of a malloc

When you call `malloc(64)`, the allocator does roughly:

1. Look up the size class (e.g., "64 bytes" → bin #4).
2. Check that bin's free list. If a chunk is free, pop it.
3. If not, possibly carve a chunk from a larger free chunk; or get a new big region from the OS via `mmap` (large) or `brk` (small, in glibc).
4. Update bookkeeping: each chunk has metadata (size, in-use bit) immediately before its pointer.
5. Return the pointer.

When you call `free(p)`, it does roughly:

1. Look at the metadata before `p` to find the size.
2. Possibly merge with adjacent free chunks (coalescing).
3. Push onto the appropriate bin's free list.

Heap fragmentation, allocator selection, and arena design are entire subfields. We will dedicate part of Part 2 to allocators (we'll even build one in Part 7). For now: **the heap is a region the OS gives you; an allocator sub-divides it; metadata around your blocks tracks what's allocated.** That's the core mental model.

### Multiple heaps in one process

It is normal for one process to have several allocators in play:

- The C library's `malloc`/`free` (used by `new`/`delete` by default, and by C code).
- A separate allocator for a runtime — Python's allocator, V8's GC heap, Go's runtime heap.
- An arena for one subsystem (e.g., a request-scoped pool in a web server).
- A custom pool for a hot data structure (a node pool for a tree).

All of them sit on top of the same `mmap`/`brk` syscalls. Following the chain down is always possible, and that's the discipline we want.

---

## 5.7 The Stack

We covered the stack discipline in Chapter 2; here we look at its layout in the address space.

A typical Linux process starts with a single 8 MB stack (configurable via `ulimit -s`). It is mapped at a high address (just below the kernel space), and grows *downward*. Each thread spawned later gets its own stack, mapped somewhere else (often via `pthread_create`-allocated `mmap`'d region).

Below the stack's "low end" is a **guard page** — a single page mapped with no permissions. If the stack pointer ever crosses into it, the CPU faults; the kernel sees the access, recognizes it as stack overflow, and kills the process with `SIGSEGV`. That fault is the *only* mechanism that catches stack overflow. There is no software check; the hardware itself does it.

For deep recursion or large stack-locals, you can blow the stack — and the failure manifests as a segfault, not as a friendly "stack full" message.

### Experiment 5.3 — stack growth direction

```cpp
#include <cstdio>
void recurse(int depth) {
    int local;
    std::printf("depth=%d local=%p\n", depth, (void*)&local);
    if (depth < 5) recurse(depth + 1);
}
int main() { recurse(0); }
```

Run it. You will see the addresses *decrease* — each recursive call has its frame at a lower address than the caller. This empirically confirms "the stack grows down."

(Subtraction is in increments of one stack frame, perhaps 16-32 bytes for this trivial function.)

---

## 5.8 Shared Library Mappings

Each shared library your program uses is mapped in as several regions:

```
7f1c2a800000-7f1c2a900000 r--p libc.so.6     (read-only headers)
7f1c2a900000-7f1c2aa20000 r-xp libc.so.6     (executable code)
7f1c2aa20000-7f1c2ab30000 r--p libc.so.6     (rodata)
7f1c2ab30000-7f1c2ab36000 rw-p libc.so.6     (.data + .bss for libc)
```

Why multiple mappings per library? Because each section needs different permissions. The ELF loader splits the library file into segments and maps each with the right protection.

Of these, only the **rw-p** mapping is private per process — that's where libc keeps its globals (e.g., `errno`, the heap arenas, `stdin`/`stdout`/`stderr` structures). The `r-xp` and `r--p` parts are physically shared across all processes that use libc.

This is also what `LD_PRELOAD` (Linux) and `DYLD_INSERT_LIBRARIES` (macOS) hook into: by getting your library mapped first, you can have your function definitions take precedence over libc's. It's how `valgrind`, `LD_PRELOAD`-based hot patches, and security monitors work.

---

## 5.9 Thread-Local Storage

Each thread needs *its own* copy of certain variables — `errno`, the per-thread heap arena, anything you mark `thread_local` in C++ or `__thread` in C.

These live in a per-thread region called **TLS**. The runtime (libc + the dynamic linker's TLS support) sets it up when each thread starts and tears it down when each thread ends. A special CPU register or per-thread base address (`fs` on x86-64 Linux, `gs` on x86-64 Windows, `tpidr_el0` on ARM64) points at the current thread's TLS region. Compilers emit `mov %fs:offset, %eax` to read a thread-local; that single offset is computed and patched at load time.

If you've ever wondered "how does `errno` magically be different per thread without explicit thread parameters?" — that's TLS. `errno` is a macro that expands to something like `*__errno_location()`, where `__errno_location` is a libc function that returns `&fs:errno_offset`.

---

## 5.10 Alignment and Padding

A side note on layout *within* objects:

A 4-byte `int` cannot live at any byte address; it must live at an address that is a multiple of 4. (More precisely, the ABI specifies an alignment for each type; on x86-64 it's typically equal to the size for primitives.) An unaligned access *might* work on x86 (slower) and *might* fault on ARM. Compilers obey alignment rules automatically.

This affects struct layout:

```cpp
struct A {
    char  c;   // 1 byte
    int   i;   // 4 bytes, must be 4-byte aligned
    char  d;   // 1 byte
};
// sizeof(A) is NOT 6. It's 12.
```

Layout:

```
offset 0:  c (1 byte)
offset 1: padding (3 bytes)   <- to align i
offset 4:  i (4 bytes)
offset 8:  d (1 byte)
offset 9: padding (3 bytes)   <- so the struct's size is a multiple of its alignment (4)
total: 12 bytes
```

Reorder for compactness:

```cpp
struct B {
    int   i;   // 4 bytes
    char  c;   // 1 byte
    char  d;   // 1 byte
};
// sizeof(B) is 8: i(4) + c(1) + d(1) + 2 padding for alignment.
```

In high-traffic data structures (game engine entities, network protocol buffers, database row layouts), this matters. A struct that is 32 bytes vs 48 bytes is the difference between fitting half a cache line and a full cache line per item — a 1.5× memory bandwidth difference for free.

### Experiment 5.4 — measure padding

```cpp
#include <cstdio>
struct A { char c; int i; char d; };
struct B { int i; char c; char d; };

int main() {
    std::printf("sizeof(A) = %zu, sizeof(B) = %zu\n", sizeof(A), sizeof(B));
    A a;
    std::printf("&a.c=%p &a.i=%p &a.d=%p\n", &a.c, &a.i, &a.d);
}
```

You'll see a 12 vs 8. Reading the field offsets confirms where the padding went.

---

## 5.11 Permissions Tell You What's Possible

Looking at any process's map and you can answer questions about *what is even possible* in this process:

- Can the process modify its own code? Look at `.text` permissions. If they're `r-x`, no. (JITs map their own pages `rwx` or use `mprotect` to flip between writable and executable. We will revisit this in Part 7.)
- Can the process execute data? Look at the heap and stack permissions. Modern systems mark them `rw-` (no `x`); attempts to execute fault. This defeats classic stack-smashing exploits that wrote shellcode to the stack and jumped to it.
- Where can the process write? Anywhere with the `w` bit. `.text` and `.rodata` are not writable; the heap, stack, and `.data`/`.bss` are.

This is the **W^X** (write-XOR-execute) policy: a page should be either writable or executable, never both. Most modern OSes enforce this by default, with explicit opt-out for JITs.

---

## 5.12 Cross-Region Bugs

Most "memory bugs" are really *boundary violation* bugs. Recognizing them by which region:

| Bug | Region | What happens |
|---|---|---|
| Stack buffer overflow | stack | Overwrite return address, saved registers, or guard page → segfault or RCE exploit |
| Heap buffer overflow | heap | Overwrite next chunk's metadata → corrupted free list → eventual crash or RCE |
| Use after free | heap | Access freed chunk; allocator may have reused it → silent corruption |
| Double free | heap | Free a chunk twice → corrupted free list |
| NULL dereference | low memory (unmapped) | Page 0 is unmapped → segfault |
| Wild pointer write to .text | .text (read-execute) | Hardware traps → segfault |
| Wild pointer write to .rodata | .rodata (read-only) | Hardware traps → segfault |
| Stack overflow | guard page | Hardware traps → segfault |
| Reading uninit local | stack | Garbage; UB; nondeterministic results |
| Reading uninit heap | heap | Garbage; UB; whatever the allocator left there |

Knowing the region tells you the diagnostic strategy: stack issues respond to `valgrind` and AddressSanitizer; heap issues likewise; `.text`/`.rodata` issues are often "your code was hit by a wild write from somewhere"; null derefs are usually trivial to find.

---

## 5.13 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| Many mappings (each section, each library) | More page-table entries; slightly slower fork; complex map | Fine-grained permissions; sharing of read-only data; debugger introspection |
| Per-thread stack | RAM and address-space cost (8 MB × N threads, even if mostly unused) | Thread isolation of locals; simple call discipline |
| Larger pages (2 MB / 1 GB) | Internal fragmentation; not always available | Fewer TLB entries; less TLB miss overhead |
| Multiple heaps / arenas | More memory waste; complexity | Bypasses lock contention; predictable lifetime; fast bulk free |
| TLS | Per-thread memory cost; setup at thread start | Fast per-thread state without explicit threading |
| W^X | JITs and hot patchers must `mprotect` or use double mappings | Defeats most code-injection exploits |

---

## 5.14 Common Misconceptions

| Misconception | Reality |
|---|---|
| "The stack and heap are in fixed locations." | Both are randomized by ASLR. Their *relative* position is conventional but absolute addresses change per run. |
| "Globals are slow because they're 'somewhere far away.'" | Globals are at known addresses, often hot in cache, and accessed by direct addressing. They're typically as fast as locals. |
| "Heap allocation is just a stack push." | Heap allocation involves looking up a free list, possibly splitting/coalescing, possibly a syscall, and updating metadata. It's much slower than stack allocation. |
| "Padding is wasted memory." | Padding makes the *common case* (aligned access) fast; without it, most CPUs would fault or run slower on misaligned reads. |
| "Globals are bad in modern code." | Globals are bad for *testing and concurrency*, not because of where they live in memory. The performance is fine; the design coupling is the issue. |
| "Each thread has its own heap." | By default, all threads in a process share one heap. Some allocators give each thread an arena to reduce contention, but the address space is shared. |

---

## 5.15 Exercises

1. **Map a real process.** Pick a moderately complex program (`firefox`, `python3 -c 'import time; time.sleep(60)'`, your editor) and inspect its `/proc/<pid>/maps` (Linux) or `vmmap <pid>` (macOS). Count: how many distinct mappings? How many distinct shared libraries? How big is the largest mapping? The smallest?

2. **Locate every kind of variable.** Extend Experiment 5.2 to include: a `const int` global, a `thread_local int`, a function-local `static`, a function pointer (`&printf`), a string from `std::string("hi")` after `s.c_str()`. Predict each address's region before running. How many surprises?

3. **Padding optimization.** Take this struct and reorder its fields for minimum size:

   ```cpp
   struct Bad {
       char a;
       double b;
       char c;
       int d;
       char e;
       short f;
   };
   ```

   Print `sizeof` and the offset of each field before and after. By how many bytes did you reduce it?

4. **Force a write to `.rodata`.** Write a program that intentionally tries to modify a string literal:

   ```cpp
   char* p = (char*)"abc";
   p[0] = 'X';
   ```

   What happens at runtime? Now try with `-O0` vs `-O2` — does optimization change the behavior? Look at `objdump -h` on the binary to find `.rodata` and confirm where the literal lives.

5. **Stack vs heap performance.** Allocate 10000 small objects on the stack vs the heap, time both:

   ```cpp
   // Stack version
   for (int i = 0; i < 10000; ++i) { int local[64]; local[0] = i; doSomething(local); }
   // Heap version
   for (int i = 0; i < 10000; ++i) { int* p = new int[64]; p[0] = i; doSomething(p); delete[] p; }
   ```

   Which is faster? By how much? Why?

6. **TLS in action.** Spawn 4 threads, each incrementing a counter 1,000,000 times. Use:
   - Version A: a single shared `int counter`. (You will see it under-count due to data races.)
   - Version B: a `thread_local int counter` per thread, summed at the end.

   Time both. The thread-local version is faster — why? What does this tell you about the cost of cache line bouncing between cores?

7. **Conceptual.** Explain to a colleague: a coworker found a bug where a buffer overflow on the stack lets an attacker control where the program returns. Describe (a) why this works (what does the overflow overwrite?), (b) why writing executable code to the stack used to work but does not now (W^X / NX), (c) what defenses today's systems layer on top to make even *return-to-libc*-style attacks hard (ASLR, stack canaries, control-flow integrity).

---

## 5.16 What's Next

You can now look at any address in any C++ program and locate it in the process's layout. You can predict where any variable lives. You can read process maps. You can reason about why memory bugs manifest the way they do.

In Chapter 6 we go from layout to *mechanics*: what does a function call actually do? What is pushed and popped from the stack? How do registers get saved? How is the return address tracked? After Chapter 6 you will be able to read a stack frame manually in a debugger and know exactly what each byte is for.

After Chapter 7 (Call Stack Deep Dive) and Chapter 8 (Pointers as Memory Addresses), Part 1 wraps with two conceptual chapters (9 and 10) that connect the metal we have been studying to the abstractions and languages we will spend the rest of the book deconstructing.

---


**[← Previous: Chapter 4 — Machine Code, Assembly, Compilers, Interpreters](04-machine-code-assembly-compilers-interpreters.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 6 — Function Calls Internally →](06-function-calls-internally.md)**
