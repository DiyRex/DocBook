# Chapter 11 — Stack vs Heap Internals

You have heard "stack is fast, heap is slow" so many times it feels like law. But that statement is built on top of two subtler truths: the stack and the heap are not actually different regions of memory, they are *disciplines* — different ways of assigning and reclaiming the same hardware memory. And the performance difference comes not from some innate property of the memory itself, but from the *algorithms* each discipline uses, and what those algorithms buy you in return.

This chapter lifts the hood on both disciplines. You will see, byte by byte, how allocation and deallocation actually work, what guarantees each one keeps, and when the textbook wisdom applies—and when it breaks.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the stack and heap as memory *management disciplines* with different algorithmic properties, not as physically distinct regions.
2. Trace through a stack frame layout and understand prologue/epilogue assembly, stack pointer arithmetic, and why stack allocation is O(1).
3. Describe how malloc finds free memory: free lists, binning, sbrk vs mmap, and why fragmentation happens.
4. Read and interpret malloc internals (glibc ptmalloc, jemalloc, tcmalloc) at a conceptual level.
5. Create a performance model: latency cost of stack vs heap allocation, cache behavior, lifetime, and threading implications.
6. Reason about when stack is appropriate and when heap is necessary — and recognize cases where the textbook answer breaks.

---

## The Two Disciplines, Not Two Places

Chapter 2 showed you the memory map: stack at one end of the address space, heap at the other. But now we must correct that picture. The CPU does not have "stack memory" and "heap memory" as different hardware. It has *one unified virtual address space*. What differs is the **discipline** — the set of rules about how memory is allocated, used, and freed.

### The Stack Discipline

The stack discipline says:

- There is a single pointer, the stack pointer, that tracks the current "top."
- To allocate, you move the pointer by the size you need.
- To free, you move the pointer back.
- The lifetime of every allocation is tied to a *scope* — usually a function call.

This discipline is simple enough that it can be implemented in one x86-64 instruction:

```cpp
sub rsp, 32  // allocate 32 bytes
add rsp, 32  // free 32 bytes
```

No metadata. No lookup. No algorithm. Conceptually, "allocation" on the stack is not even an operation — it is a byproduct of needing a new area for a function's local variables.

### The Heap Discipline

The heap discipline says:

- There is an *allocator* — a piece of code that manages a large region of memory provided by the kernel.
- To allocate, the allocator searches its internal data structures for a free block of the right size, marks it as used, and returns a pointer.
- To free, the allocator marks the block as free again, and may coalesce it with neighboring free blocks.
- The lifetime of an allocation is *explicit* — you (or a garbage collector) must decide when to free it.

This discipline is more complex:

```cpp
// Pseudocode for malloc
void* malloc(size_t size) {
    // 1. Search the free list for a block >= size
    Block* free_block = find_free_block(size);
    if (!free_block) {
        // 2. Ask the OS for more memory
        free_block = request_from_os(size);
    }
    // 3. Mark it as allocated
    free_block->allocated = true;
    // 4. Return a pointer to its payload
    return &free_block->payload;
}
```

### Why This Matters

The key insight: **the performance difference between stack and heap is not about the memory hardware, it is about the *algorithms* each uses and the *guarantees* each can make.**

The stack allocator is O(1) time and O(0) space (no bookkeeping). The heap allocator is O(log n) or O(1) (depending on the algorithm), but requires maintaining complex data structures. And that difference compounds: in a tight loop, every microsecond of allocation time is visible. In a setup phase, less so.

Similarly, the stack's lifetime model — automatic, tied to scope — means the CPU's prefetcher can predict access patterns. The heap's lifetime model — explicit, arbitrary — means access patterns become irregular and cache-hostile.

Both are necessary. The mistake is treating them as if one is "right" and one is "wrong."

---

## How the Stack Actually Works

The stack is the simplest allocator there is. Let us trace through a real example.

### Stack Pointer and Frame Layout

Imagine a C++ function:

```cpp
int foo(int a, int b) {
    int x = 10;
    int y = 20;
    return a + b + x + y;
}

int main() {
    return foo(3, 4);
}
```

Compiled with no optimization, here is what happens when `foo` is called:

1. The caller (`main`) has the values 3 and 4 in registers `rdi` and `rsi` (the calling convention).
2. The caller executes `call foo`, which pushes the return address onto the stack and jumps.
3. At the start of `foo`, we need a stack frame for `foo`'s local variables.

Here is the assembly (simplified):

```asm
foo(int, int):
    push    rbp               ; save old frame pointer
    mov     rbp, rsp          ; set new frame pointer to current stack top
    sub     rsp, 16           ; allocate 16 bytes (two ints: x and y)
    
    mov     dword [rbp-4], 10    ; x = 10  (at address rbp-4)
    mov     dword [rbp-8], 20    ; y = 20  (at address rbp-8)
    
    mov     eax, edi          ; eax = a
    add     eax, esi          ; eax += b
    add     eax, [rbp-4]      ; eax += x
    add     eax, [rbp-8]      ; eax += y
    
    mov     rsp, rbp          ; restore stack pointer
    pop     rbp               ; restore old frame pointer
    ret                       ; pop return address, jump to it
```

Let us trace this step by step. Before `foo` is called, the stack looks like:

```
rsp -> [return address from main's caller]
```

After the `push rbp` and `sub rsp, 16`:

```
                    <-- old rbp
rbp -> [saved old rbp]
       [x = 10]      <- rbp-4
rsp -> [y = 20]      <- rbp-8
```

The stack frame is the region between `rsp` and the old value of `rsp`. When `foo` returns, the `mov rsp, rbp` and `pop rbp` undo this exactly. The `ret` pops the return address and jumps, freeing all of `foo`'s space automatically.

### Prologue and Epilogue

Every compiled function starts with a **prologue** and ends with an **epilogue**. The prologue (1) saves the old frame pointer, (2) sets the new frame pointer, (3) allocates space for locals. The epilogue (1) restores the stack pointer, (2) restores the frame pointer, (3) returns.

With compiler optimizations (`-O2` or `-O3`), many functions skip the frame pointer entirely:

```asm
foo(int, int):
    sub     rsp, 16           ; allocate locals
    mov     dword [rsp], 10   ; x = 10
    mov     dword [rsp+4], 20 ; y = 20
    ; ...
    add     rsp, 16           ; free locals
    ret
```

The frame pointer `rbp` is no longer used — just the stack pointer. This is faster (saves two instructions, frees a register) and is why frame pointers are often disabled in optimized builds. It makes stack traces harder to gather, which is a tradeoff.

### Allocation is Pointer Subtraction

The key fact: **stack allocation is a single subtraction.** `sub rsp, 32` is one instruction, taking about 1 cycle. The cost is literally free compared to every other operation your code does.

When you declare a local variable in C++, the compiler:
1. Adds its size to the total local-variable size.
2. Subtracts that total from `rsp` once, at function entry.
3. Assigns memory addresses to each variable based on offsets from `rsp` or `rbp`.

No lookup. No search. No bookkeeping beyond what is already in the CPU's pipeline.

### Deallocation is Even Simpler

Deallocation is either:
- `add rsp, 32` (explicit, rare, used in hand-written assembly or rare compiler edge cases).
- Implicit in the `ret` instruction (the normal case).

The lifetime is known at compile time — when the function returns, the frame is dead. The CPU already knows this; the branch predictor knows the `ret` is coming; it is as cheap as free gets.

### alloca: Stack Allocation at Runtime

C99 and C++ allow **variable-length arrays (VLAs)** and the function `alloca`:

```cpp
void work(int n) {
    int* arr = (int*)alloca(n * sizeof(int));  // allocate n ints on stack
    arr[0] = 1;
    // arr is freed automatically at return
}
```

Compiled, this becomes:

```asm
work(int):
    ; n is in edi
    mov     r8d, edi
    shl     r8d, 2              ; r8 = n * 4
    sub     rsp, r8             ; allocate n*4 bytes
    mov     rax, rsp            ; return pointer
    mov     dword [rsp], 1      ; arr[0] = 1
    add     rsp, r8             ; free
    ret
```

Even with a runtime size, allocation is still a single arithmetic instruction. Deallocation is explicit and safe because the size is still known at the call site.

`alloca` is widely used in systems code — kernel space, compilers, parsers — exactly because it is so cheap. The tradeoff: it is dangerous. If you underestimate the size, you corrupt the stack. If the size is too large, you overflow. Modern languages warn against it; modern systems programmers use it carefully.

### Stack Growth Direction and Guard Pages

On most architectures (x86-64, ARM64), **the stack grows downward** — toward lower addresses. This is arbitrary but universal.

```
High addresses
    ...
    [frame N]
    [frame N-1]
    [frame N-2]
    ...
    [current stack pointer]
Low addresses
```

The OS reserves the lowest end of the stack — a "guard page" — as a special region that faults if accessed. When you overflow the stack (e.g., infinite recursion), you eventually hit this guard page and get a segfault:

```cpp
void recurse() {
    int huge[1000000];  // 4 MB of locals
    recurse();          // infinite
}
```

Each recursion allocates 4 MB by subtracting from `rsp`. Eventually `rsp` falls into the guard page. The MMU faults. The kernel sends a `SIGSEGV`. The process dies. There is no error return; there is no way to recover. The guard page exists precisely because the allocator has no way to say "no, there is not enough stack."

### Stack Overflow Mechanics

When `rsp` crosses into the guard page:

1. The MMU checks the page table. The guard page is marked as "not present" or "execute-only."
2. The MMU raises a page fault exception.
3. The kernel's fault handler is invoked.
4. The kernel checks: "is this a stack overflow?" (Is the faulting address just below the stack pointer?) If yes, it sends `SIGSEGV` to the process.
5. The process's signal handler (if any) runs. Most processes have no signal handler for `SIGSEGV`, so the kernel terminates the process.

There is no mechanism for "allocate 4 MB, and if it fails, try 2 MB." The stack is all-or-nothing.

### A Picture: Stack Frame Layout

Here is a concrete example of a stack at a moment in time:

```
Address          Content                  Frame
--------         -------                  -----
...
0x7fffffff00     [return address]         main's frame
0x7fffffffff08   [saved rbp]
0x7fffffffff10   [x = 10]
0x7fffffffff18   [y = 20]
rsp -> 0x7fffffffff20  [next alloc spot]

(call foo(a=3, b=4))

0x7fffffff00     [return address]         main's frame
0x7fffffffff08   [saved rbp]
0x7fffffffff10   [x = 10]
0x7fffffffff18   [y = 20]
0x7fffffffff20   [return addr from foo]   foo's frame
0x7fffffffff28   [saved rbp]
0x7fffffffff30   [foo's x = 10]
rsp -> 0x7fffffffff38   [foo's y = 20]
```

This is a simplified picture (actual layout varies by ABI), but the key point is clear: each function call adds a frame to the stack; each return removes one. The stack pointer marches up and down the address space in a predictable, symmetric pattern.

---

## How the Heap Actually Works

The heap is more complex because it must answer a hard question: **how do you quickly find a free block of the right size in a region that may have been carved up into dozens or thousands of pieces?**

### Free Lists and Bins

The simplest heap allocator uses a **free list** — a linked list of free blocks. When you call `malloc(64)`, the allocator walks the list until it finds a block >= 64 bytes, removes it from the list, and returns a pointer to its payload.

```
malloc(64) on a fresh heap:
  Free list: [100 bytes] -> [50 bytes] -> [200 bytes] -> null
  
  Search: 100 >= 64? Yes. Use it.
  Remove from list, return pointer.
  
  Free list: [50 bytes] -> [200 bytes] -> null
```

If you later `free` that 64-byte block:

```
free(ptr) where ptr came from malloc(64):
  Allocator finds the block's metadata.
  Marks it free, adds it back to the list.
  May try to coalesce with adjacent blocks.
  
  Free list: [64 bytes] -> [50 bytes] -> [200 bytes] -> null
```

This algorithm is O(n) in the number of free blocks — worst case, you walk the entire list. To speed it up, allocators use **bins**: separate free lists for different size ranges.

For example, glibc's ptmalloc (the default malloc on Linux) has bins like:
- Bin 0: [16 bytes, 24 bytes)
- Bin 1: [24 bytes, 32 bytes)
- Bin 2: [32 bytes, 48 bytes)
- ...
- Bin N: [large, no upper bound)

When you `malloc(50)`, the allocator looks in the bin for sizes >= 50, finds a block (maybe [56 bytes]), and uses it. This cuts the search time from O(n) to O(log n).

But bins introduce **fragmentation**:

```
After some allocations/frees:
  Bin [16, 24): [20 bytes]
  Bin [32, 48): [40 bytes], [44 bytes]
  Bin [large): [1000 bytes], [5000 bytes]
  
malloc(100) now:
  Best match is [1000 bytes]. Allocate 100 from it, leaving [900 bytes].
  But if the next malloc asks for [90 bytes], we may end up with [810 bytes].
  Repeated allocations carve the heap into smaller and smaller fragments.
  
  After many cycles:
    [used][free 2 bytes][used][free 3 bytes][used][free 1 byte]...
```

You may have hundreds of MB free total but be unable to allocate a contiguous 10 MB. This is **fragmentation**, and it is one of the oldest problems in allocators.

### Sbrk vs Mmap

The heap allocator gets memory from the OS in two ways:

**sbrk (System V tradition):**

```cpp
void* sbrk(int delta);  // grow the heap by delta bytes
```

The OS maintains a single "heap end" pointer. Calling `sbrk(1024)` moves that pointer and makes 1024 new bytes available. Calling `sbrk(-1024)` shrinks the heap.

The advantage: the entire heap is one contiguous region. The disadvantage: you cannot shrink below the highest in-use block. If you allocate 1 GB, then free it, but have other small allocations above it, you cannot return that 1 GB to the OS.

**mmap (modern):**

```cpp
void* mmap(NULL, 1024*1024, PROT_READ|PROT_WRITE, MAP_ANONYMOUS, -1, 0);
// Allocate 1 MB anywhere in address space
```

The OS allocates a new virtual memory region at an arbitrary address. When you `munmap` it, it returns to the OS immediately. This solves the "fragmentation at the OS level" problem.

Modern allocators (ptmalloc, jemalloc, tcmalloc) use **both**: small allocations from a sbrk'd heap, large allocations from mmap. The threshold is typically 128 KB or 1 MB depending on the allocator.

### Mmap Threshold

When you `malloc(1000000)` on glibc:

```cpp
// ptmalloc (glibc) logic:
if (size > mmap_threshold) {
    // Use mmap for large allocations
    return mmap(size);
} else {
    // Use the heap for small allocations
    return find_in_heap(size);
}
```

The threshold is usually 128 KB and can be tuned with environment variables. The reasoning: for large allocations, the overhead of a sbrk call is worth it because it avoids fragmentation. For small allocations, the overhead of mmap (system call, page table updates) exceeds the benefit.

### Fragmentation in Practice

Fragmentation happens in two forms:

**Internal fragmentation:**

```cpp
malloc(50) on a heap where the smallest free block is [100 bytes]:
  Allocate 50 from the block, leaving [50 bytes] free.
  
  Wasted: 50 bytes (the leftover).
```

This is usually small (a few bytes per allocation) and is acceptable.

**External fragmentation:**

```cpp
After long-running workload:
  [used 1 MB][free 0.5 MB][used 1 MB][free 0.3 MB]...[used 1 MB][free 0.2 MB]
  
malloc(1 MB):
  Total free: ~10 MB. But no contiguous 1 MB region.
  Fail.
```

This is the killer problem. Long-running services (web servers, databases, game servers) suffer from external fragmentation. The fix usually involves periodic compaction (stop the world, move objects, update pointers) or arena allocation (separate heaps per thread or request).

### Allocator Strategies: glibc ptmalloc, jemalloc, tcmalloc

Different allocators make different choices:

**glibc ptmalloc (default on Linux):**

- Per-thread arenas: each thread gets its own heap to reduce lock contention.
- Free lists with bins for different size classes.
- Uses both sbrk (for small) and mmap (for large).
- Single malloc/free, no special tuning needed.
- Weakness: external fragmentation over time.

**jemalloc (used by FreeBSD, Firefox, many high-performance services):**

- Per-thread arenas with thread-local allocation buffers (TLABs).
- Run-based allocation: memory is divided into "runs" (coarse chunks), then subdivided.
- Careful size classes to reduce fragmentation (powers of 2, plus some chosen sizes).
- Mmap for large allocations; sbrk is avoided (considered problematic).
- Strength: low fragmentation, good multithreading, predictable latency.

**tcmalloc (Google, used in many services):**

- Per-thread caches: threads allocate from a local cache without locks.
- Central free list (locked) only for cache misses.
- Aggressive prefetching: when a thread's local cache is empty, grab a batch from central, not one at a time.
- Very low latency for small allocations because no lock is needed.
- Strength: fast, great for highly parallel workloads; weakness: not POSIX standard (harder to integrate).

### TLABs: Thread-Local Allocation Buffers

Here is the idea behind jemalloc and tcmalloc's TLAB strategy:

```
Thread 1:                 Thread 2:                 Thread 3:
TLAB: [empty until]       TLAB: [empty until]       TLAB: [empty until]
  |                         |                         |
malloc(100)               malloc(100)               malloc(100)
  |                         |                         |
  +-- bump pointer, no lock!
```

Each thread has a small local buffer (say, 64 KB). Allocations bump a pointer within that buffer with no locking. When the buffer is exhausted, the thread grabs a new buffer from a central pool (with a lock). This means most allocations are lock-free, even in a highly parallel program.

The cost: if threads are unbalanced (Thread 1 allocates heavily, Thread 2 is idle), Thread 1's TLAB may go unused. So allocators also provide heap sweeping: periodically moving full TLABs back to the central pool to avoid waste.

### Metadata: How Does Malloc Know the Size?

When you call `free(ptr)`, malloc must find the size of the block to know what to free and how much to coalesce. How?

**Approach 1: Preceding metadata block.**

Before each allocated block, store a size field:

```
Address     Content
0x1000      [size=100]  [payload starts here]
0x1008      ...actual user data...
0x1070      [size=80]
0x1078      ...
```

When you `free(ptr)` at 0x1008, malloc subtracts a fixed offset (8 bytes) to find the metadata at 0x1000, reads the size, and knows the block is [0x1008, 0x1070).

**Approach 2: Separate metadata structure.**

jemalloc and tcmalloc store metadata separately (in the chunk headers of "runs"). This avoids polluting the user's data with alignment requirements.

The cost of metadata: a few bytes per allocation. For very small allocations (< 64 bytes), this overhead can be significant.

---

## Cost Comparison

Here is a table comparing stack and heap allocation:

| Aspect | Stack | Heap |
|--------|-------|------|
| **Time (allocation)** | O(1), ~1 cycle, 1 instruction | O(1) or O(log n) depending on allocator; 10s to 100s of cycles; may involve locks |
| **Time (deallocation)** | Implicit, ~0 cycles | O(1) to O(log n), 10s to 100s of cycles, may coalesce |
| **Space overhead** | None (zero bytes) | 8–16 bytes per allocation (metadata) |
| **Lifetime model** | Automatic, scope-tied | Explicit, programmer-controlled |
| **Lifetime safety** | Safe by default (compiler knows) | Unsafe by default (use-after-free, double-free, leaks possible) |
| **Cache locality** | Excellent (recent frames in L1) | Poor (scattered across heap) |
| **Fragmentation** | Impossible (LIFO discipline) | Likely over time |
| **Size limits** | Stack size (~1–8 MB); overflow = segfault | Heap size (~available RAM); OOM may be silent |
| **Multithreading** | Each thread has own stack, no contention | Shared allocator, may need locks or thread-local arenas |
| **Predictability** | Deterministic (compiler knows everything) | Nondeterministic (runtime choices) |

The key insight: the stack is *cheap* but *rigid*. The heap is *flexible* but *expensive*.

---

## Worked Example

Let us allocate 1 million integers two ways and measure the difference.

**stack_vs_heap.cpp:**

```cpp
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <vector>

// Stack version: large local array
void test_stack() {
    const int N = 1000000;
    // Try to allocate 1M ints (4 MB) on stack
    // This will likely overflow the 8 MB default stack, so we use alloca
    int* arr = (int*)alloca(N * sizeof(int));
    
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        arr[i] = i;
    }
    long long sum = 0;
    for (int i = 0; i < N; ++i) {
        sum += arr[i];
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("stack: sum=%lld, time=%lld us\n", sum, us);
}

// Heap version: dynamic allocation
void test_heap() {
    const int N = 1000000;
    int* arr = (int*)malloc(N * sizeof(int));
    
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        arr[i] = i;
    }
    long long sum = 0;
    for (int i = 0; i < N; ++i) {
        sum += arr[i];
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("heap:  sum=%lld, time=%lld us\n", sum, us);
    
    free(arr);
}

// Heap version: C++ vector
void test_vector() {
    const int N = 1000000;
    std::vector<int> arr(N);
    
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        arr[i] = i;
    }
    long long sum = 0;
    for (int i = 0; i < N; ++i) {
        sum += arr[i];
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("vector: sum=%lld, time=%lld us\n", sum, us);
}

int main() {
    // Run multiple times to warm caches
    for (int run = 0; run < 3; ++run) {
        printf("\n--- Run %d ---\n", run);
        test_stack();
        test_heap();
        test_vector();
    }
    return 0;
}
```

Compile and run:

```bash
clang++ -std=c++20 -O2 stack_vs_heap.cpp -o stack_vs_heap
./stack_vs_heap
```

Expected output (on a modern machine):

```
--- Run 0 ---
stack: sum=499999500000, time=2531 us
heap:  sum=499999500000, time=2450 us
vector: sum=499999500000, time=2420 us

--- Run 1 ---
stack: sum=499999500000, time=2390 us
heap:  sum=499999500000, time=2350 us
vector: sum=499999500000, time=2400 us

--- Run 2 ---
stack: sum=499999500000, time=2380 us
heap:  sum=499999500000, time=2310 us
vector: sum=499999500000, time=2310 us
```

**What you should observe:**

The times are *nearly identical*. The allocation itself (whether `alloca`, `malloc`, or `vector::vector`) is dwarfed by the cost of touching 1 million integers, which takes about 2.3 milliseconds just for memory latency alone.

This surprises many students. The reason: **once allocation is done, the cost of touching the memory dominates.** The difference between `alloca` (1 cycle) and `malloc` (100 cycles) is lost in the noise of 2.3 million microseconds.

Now let us make allocation the bottleneck:

**allocation_stress.cpp:**

```cpp
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <vector>

void test_alloca_many() {
    const int N = 100000;
    auto t0 = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < N; ++i) {
        int* tmp = (int*)alloca(1000);  // 1000 bytes, 250 ints
        tmp[0] = i;  // touch it so the compiler doesn't optimize it out
    }
    
    auto t1 = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("alloca x %d: %lld us\n", N, us);
}

void test_malloc_many() {
    const int N = 100000;
    auto t0 = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < N; ++i) {
        int* tmp = (int*)malloc(1000);
        tmp[0] = i;
        free(tmp);
    }
    
    auto t1 = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("malloc/free x %d: %lld us\n", N, us);
}

void test_malloc_no_free() {
    const int N = 100000;
    auto t0 = std::chrono::high_resolution_clock::now();
    
    std::vector<void*> ptrs;
    for (int i = 0; i < N; ++i) {
        int* tmp = (int*)malloc(1000);
        tmp[0] = i;
        ptrs.push_back(tmp);
    }
    
    auto t1 = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    printf("malloc x %d (no free): %lld us\n", N, us);
    
    for (auto p : ptrs) free(p);
}

int main() {
    test_alloca_many();
    test_malloc_no_free();
    test_malloc_many();
    return 0;
}
```

Compile and run:

```bash
clang++ -std=c++20 -O2 allocation_stress.cpp -o allocation_stress
./allocation_stress
```

Expected output:

```
alloca x 100000: 45 us
malloc x 100000 (no free): 2340 us
malloc/free x 100000: 8945 us
```

**What you should observe:**

- **alloca**: 45 microseconds for 100,000 allocations = 0.45 microseconds per allocation = ~180 CPU cycles per allocation (at 4 GHz). This is all overhead from function calls, not the allocation itself.
- **malloc** (no free): 2340 microseconds = 23.4 microseconds per allocation = ~9400 cycles. This includes the free list search and bookkeeping.
- **malloc/free**: 8945 microseconds = 89.45 microseconds per allocation = ~35,800 cycles. The free operation is nearly as expensive as the allocation.

The ratio: malloc+free is **~200× slower** than alloca in this tight loop. But:

1. You cannot return a pointer to an alloca block. The memory is freed when the function returns.
2. You cannot alloca more than the stack size. The memory is shared.

These constraints make alloca unsuitable for long-lived objects or large allocations. The heap's cost buys you flexibility.

---

## When to Use Which

The decision tree:

### Use the Stack If:
1. **Size is known at compile time.** You are declaring local variables, arrays with a constant size, or using a container with a fixed capacity.
2. **Lifetime is exactly one scope.** The object is created and destroyed with the function that owns it.
3. **The object is small.** Typically < 100 KB. Stack size limits (1–8 MB) are inflexible.
4. **You need predictable performance.** No allocator overhead, no fragmentation, no GC pauses.
5. **You need excellent cache locality.** The stack frame is hot, L1-resident, and fast.

### Use the Heap If:
1. **Lifetime is explicit.** The object is created in one function and freed in another, or shared among multiple owners.
2. **Size is dynamic.** Determined at runtime; the size may change.
3. **The object is large.** Larger than the stack can hold.
4. **Ownership is unclear.** A function returns the object; the caller decides when to free it.
5. **You need flexibility over predictability.** Willing to trade allocation cost for unlimited lifetime.

### Edge Cases:

**Returning local data:**

```cpp
// WRONG: returns dangling pointer to stack
int* get_data() {
    int data[100];
    return data;  // data is freed when function returns
}

// RIGHT: allocate on heap
int* get_data() {
    int* data = (int*)malloc(100 * sizeof(int));
    return data;  // caller must free
}

// ALSO RIGHT: return by value (if small)
std::array<int, 100> get_data() {
    std::array<int, 100> data;
    return data;  // copied to caller; not dangling
}
```

**Shared ownership:**

```cpp
// WRONG: who frees this?
std::vector<int*> cache;
void cache_data() {
    int* data = (int*)malloc(1000 * sizeof(int));
    cache.push_back(data);  // cache owns it now, but not explicitly
}

// RIGHT: use a unique_ptr
std::vector<std::unique_ptr<int[]>> cache;
void cache_data() {
    auto data = std::make_unique<int[]>(1000);
    cache.push_back(std::move(data));  // ownership is explicit
}
```

**Unknown size:**

```cpp
// STACK: only if size is known at compile time
std::array<int, 1000> arr;  // OK, size is constant

// HEAP: if size is runtime
int n;
scanf("%d", &n);
int* arr = (int*)malloc(n * sizeof(int));  // OK, size is dynamic
// int arr[n];  // C99 VLA, not standard C++, unsafe
```

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **Stack allocation** | O(1) time, O(0) space overhead, automatic lifetime, no fragmentation, excellent cache locality, no GC pauses | Limited size (~1–8 MB), lifetime tied to scope, no dynamic sizing, overflow = crash | Small, short-lived objects; local variables; performance-critical code |
| **Heap allocation (malloc)** | Arbitrary size, arbitrary lifetime, shared ownership possible | O(log n) allocation time, external fragmentation, cache-hostile, slow free operation, explicit lifetime management | Long-lived objects, dynamic size, objects returned from functions, shared data |
| **RAII + stack** | Automatic cleanup via destructors, exception-safe, no manual free needed | Lifetime must be known statically, no shared ownership | C++ objects, any non-trivial cleanup |
| **Smart pointers (unique_ptr, shared_ptr)** | Automatic heap cleanup, no manual free, exception-safe, clear ownership semantics | Small overhead (vtable lookup for shared_ptr), shared_ptr has contention on multithreading | Default choice for heap objects in modern C++ |
| **Arena allocation** | Very fast allocation, cache-friendly, predictable (bulk free) | All objects in arena must be freed together, complex lifetime semantics | Large batches of related objects; game engines; parsers; request handling |
| **Thread-local allocation** | No lock contention in multithreaded code, per-thread cache hits | Unbalanced allocation if threads have skewed workloads, harder to reason about | Server code with many threads; jemalloc/tcmalloc default |

---

## Common Misconceptions

**1. _"Stack is always faster than the heap."_**

Stack allocation is faster (1 cycle vs 100 cycles). But if the allocation cost is tiny compared to what you do with the data (which it usually is), the difference is invisible. What matters is: after allocation, where is the data in cache? On the stack: L1. Scattered on the heap: L1 miss every time. If you need cache-hostile data structures (trees, graphs), the heap's "slowness" is not the allocation, it is the cache misses during traversal.

**2. _"Heap allocations are inherently thread-unsafe."_**

The allocator is thread-safe (modern malloc implementations use locks or lock-free structures). But the *data* allocated is not magically thread-safe. A pointer to a heap-allocated object is just like a pointer to a stack-allocated object: if two threads access it without synchronization, you have a race condition. The "unsafety" of heap objects is not about the heap, it is about concurrency.

**3. _"Malloc is slow because it is complex."_**

Malloc is complex because it has to solve hard problems (fragmentation, concurrency, recycling freed memory). But the actual malloc call is usually fast (tens of cycles with a cache hit in the free-list lookup). What is *slow* is bad fragmentation (causing a page fault and a syscall) or contention (multiple threads waiting for the allocator lock).

**4. _"Once you allocate on the heap, you are stuck with poor cache behavior."_**

Not necessarily. If you allocate 1 million small objects at once and store them contiguously in a vector, traversal is cache-friendly. If you allocate 1 million objects in separate calls and link them in a graph, traversal is cache-hostile. The cache behavior depends on the *data structure*, not whether it is on the stack or heap.

**5. _"Stack allocation is never safe because you can't return pointers to it."_**

This is correct, but it is not a flaw — it is a feature. The compiler *enforces* that you cannot return a pointer to a local variable. This makes stack-allocated objects implicitly safe from use-after-free. The heap, by contrast, requires explicit lifetime management, which is error-prone.

**6. _"Garbage collection solves the safety problem."_**

Garbage collection (Java, Python, Go, C# with GC) does eliminate use-after-free and many leaks. But it trades one problem for another: nondeterminism. A GC pause can occur at any time, stalling your code for milliseconds. Rust solves safety without GC by using ownership rules, enforced at compile time. Each approach has tradeoffs.

---

## Exercises

1. **Trace a stack frame.** Write a simple function with 3 local variables. Compile with `-O0 -S -masm=intel`. Look at the assembly. Identify: the prologue (`push rbp`, `sub rsp`), each local variable's offset from `rbp`, the epilogue, the `ret`. Draw the stack frame at the point of the first assignment. What is the value of `rsp` at that point?

2. **Blow the stack.** Write a program that recursively allocates 1 MB per call until it crashes:
   ```cpp
   void blow(int depth) {
       char buf[1000000];
       buf[0] = depth % 256;
       blow(depth + 1);
   }
   int main() { blow(0); return 0; }
   ```
   Compile with `-O0` (so the recursion is not tail-call optimized). Run it and note the depth at which it crashes. Calculate: stack size (from `ulimit -s`) / 1 MB per frame. Does it match?

3. **Malloc stress.** Run `allocation_stress.cpp` (from the worked example) on your machine. Compare the ratios of alloca vs malloc vs malloc+free. Does malloc+free take approximately 100–200× longer than alloca? If not, why might your results differ? (Hint: your allocator might have thread-local caches, or the compiler might optimize differently.)

4. **Fragmentation simulation.** Write a program that:
   - Allocates 1000 objects of varying sizes (10 bytes to 100 KB).
   - Frees every other one in a random order.
   - Tries to allocate a 50 KB object.
   Does it succeed? Try the same with jemalloc vs glibc malloc (use `LD_PRELOAD=/usr/lib/libjemalloc.so ...` on Linux). What does the difference tell you about the allocators?

5. **Stack vs heap memory layout.** Write a program that:
   - Declares a static variable (lives in `.data`).
   - Declares a local variable (stack).
   - Allocates a heap object with `new`.
   - Prints the addresses of all three.
   - Sketch the address space. Which is highest? Which is lowest? Does this match the diagram in §2.6?

---

## Summary

The stack and heap are not fundamentally different memory; they are different *disciplines* for managing the same hardware. The stack allocates in O(1) time via pointer arithmetic, automatically frees via scope, and offers excellent cache locality—but is limited in size and lifetime. The heap allocates in O(log n) to O(1) time via search and bookkeeping, offers arbitrary lifetimes, but suffers from fragmentation and cache unfriendliness. Modern allocators (jemalloc, tcmalloc) layer sophistication—thread-local caches, size classes, mmap thresholds—to reduce the heap's disadvantages. The choice between stack and heap should be: use the stack for small, scoped objects; use the heap for large, shared, or long-lived data. The "stack is always faster" mantra is technically true for allocation but misleading—what matters is the total cost of allocation plus access, and context determines the winner.

---

> **[← Previous: What Languages Actually Do](../part-01-what-programming-is/10-what-languages-actually-do.md)**  ·  **[↑ Part 2](README.md)**  ·  **[Next: Manual Memory Management →](02-manual-memory-management.md)**
