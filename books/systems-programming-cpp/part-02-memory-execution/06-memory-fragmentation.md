# Chapter 6 — Memory Fragmentation

A process owns 200 MB of free memory and cannot allocate 1 MB. This is not a contradiction. It is fragmentation, and it kills long-running services with a slowness that looks like a leak.

You allocate. You free. You allocate again. Over hours or days of running, the heap becomes a patchwork of used and free blocks scattered across the address space. The total free memory is large, but the largest contiguous free block is small. When you need a megabyte in one piece, the allocator looks at its free lists, finds nothing big enough, and either fails (returning null) or calls into the kernel for fresh memory from the OS. Either way, your service degrades.

This chapter is about how fragmentation happens, why it matters more in some systems than others, and the practical techniques that guard against it.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish **external fragmentation** (free memory, but scattered into pieces too small for the request) from **internal fragmentation** (wasted space inside an allocation block due to size-class rounding).
2. Visualize fragmentation in a heap and predict when it accumulates given a pattern of allocations and frees.
3. Understand why modern allocators use **size classes** and **segregated free lists** to reduce fragmentation and improve allocation speed.
4. Recognize the **arena allocator pattern** — allocate many objects from a single pool, then free all at once — and implement a simple bump allocator.
5. Explain why C++ cannot compact memory like garbage-collected languages can (due to pointer semantics), and what that costs in terms of long-term heap degradation.
6. Choose between standard `malloc`, specialized allocators like `jemalloc` or `tcmalloc`, custom pools, and arenas based on your workload's allocation and freeing pattern.

---

## 6.1 External vs Internal Fragmentation

There are two kinds of fragmentation. They look similar to the user (the allocator can't give you the block you asked for) but they happen for opposite reasons.

### External Fragmentation

You have free memory, but it is scattered into pieces, and the largest piece is smaller than what you need.

Picture a heap after a while:

```
    0x1000  +--------+--------+--------+--------+--------+
            | USED   | FREE   | USED   | FREE   | USED   |
            | 50 KB  | 30 KB  | 40 KB  | 20 KB  | 60 KB  |
            +--------+--------+--------+--------+--------+
            ^        ^        ^        ^        ^
     layout: Allocate A (50K), allocate B (40K), allocate C (60K).
             Free B. Now request 40 KB: OK. Request 50 KB: no contiguous block.
             Total free: 50 KB (30 + 20). Largest block: 30 KB.
```

This is **external fragmentation**. The memory exists but is split into pieces too small for the request.

It happens because:

- **Random-sized allocations.** You allocate 17 bytes, then 42 bytes, then 103 bytes. Each gets its own block.
- **Random frees.** You free in a different order than you allocated. Blocks in the middle of the heap are freed, leaving holes. New allocations don't fill those holes exactly.
- **Time.** Fragmentation is cumulative. A long-running service (web server, database, game engine) may run for weeks. Even a tiny size mismatch, multiplied by millions of allocations, creates a slowly degrading heap.

A real consequence: the heap grows even though you're just recycling memory. The allocator calls `mmap` or `sbrk` to ask the OS for fresh pages, because the existing free memory is not usable.

### Internal Fragmentation

You allocated a block, and part of it is wasted because the allocator rounded up to a size class.

Picture a single allocation:

```
    You request: 17 bytes
    Allocator hands you: 32-byte block (because 32 is the next size class)
    You use: 17 bytes
    Wasted: 15 bytes
```

This is **internal fragmentation**. The memory is yours, but you're not using all of it.

It happens because allocators use **size classes** — fixed-size buckets (8, 16, 32, 64, 128, 256 bytes, etc.) — to speed up allocation. Instead of asking "what size is your request?" and carving a block of exactly that size, the allocator rounds to the nearest class. This wastes space but makes the free lists simpler.

A real allocator's size classes might be:
```
8, 16, 32, 64, 128, 256, 512, 1K, 2K, 4K, 8K, 16K, ...
```

If you allocate 17 bytes, you get 32. If you allocate 100 bytes, you get 128. The internal fragmentation per allocation is at most 50% of the block size (worst case: you ask for size-class-size + 1, and get the next class, which doubles the size). In practice it's much less.

### When Each Matters

**Internal fragmentation** is a constant overhead. If you allocate a million small objects and each wastes 10 bytes internally, that's 10 MB lost. It's annoying but predictable.

**External fragmentation** is insidious. It is cumulative and unpredictable. A service that runs for 1 day might fragment slightly; the same service running for 30 days becomes unusable. And the process's memory footprint (RSS) climbs even though the logical data size is constant.

---

## 6.2 Why It Matters for Long-Running Services

Consider a web server. Each request allocates memory: HTTP parser buffers, request object, route handler locals, database result sets. Each response frees that memory. Over a week of requests, the heap looks like a war zone.

The server's heap might look like this after 1 million requests:

```
    +----------+
    | 4 GB RSS |
    +----------+
    | logical data: 2 GB (still in use)
    | total free: 1.8 GB
    | largest contiguous block: 600 MB
    | working memory per request: 50 MB (should succeed)
    +----------+
```

But then a spike request comes that needs 1 GB for a batch operation:

```
    Allocate 1 GB...
    largest contiguous block: 600 MB
    -> FAIL or kernel allocates new pages
    -> RSS jumps to 5 GB
    -> OOM killer activates or process crashes
```

This is not a memory leak. This is fragmentation. The allocator has the memory, but not in the right shape.

Long-running services suffer most because:

1. **Time multiplies fragmentation.** 1 million small decisions, each one slightly suboptimal, add up.
2. **Unpredictable spikes.** A normal request needs 50 MB; a batch request needs 1 GB. The fragmented heap cannot satisfy the 1 GB request even though logically it has it.
3. **The symptom is mysterious.** Your monitoring shows "RSS growing" but "heap usage stable." Users report slowness or failures with no clear cause.

Databases, game engines, real-time systems, and any 24/7 service face this. The fix is not to allocate less; it is to allocate *smarter*.

---

## 6.3 How Modern Allocators Fight It

### Size Classes and Segregated Free Lists

Instead of one big free list, a modern allocator keeps multiple free lists, one per size class.

A simplified picture:

```cpp
struct Allocator {
    FreeList freelist_8;    // blocks of 8 bytes
    FreeList freelist_16;   // blocks of 16 bytes
    FreeList freelist_32;   // blocks of 32 bytes
    // ... up to, say, 128 MB
};
```

When you `malloc(17)`, the allocator:
1. Rounds 17 up to the next size class: 32.
2. Looks at `freelist_32`.
3. Pops a 32-byte block if available; if not, carves one from a larger region.

This has two benefits:

- **Speed.** Finding a 32-byte block in a dedicated list is O(1) (the list is a linked list of blocks of the same size).
- **Reduced fragmentation.** All blocks of size 32 go on the 32-byte list. They tend to be freed and re-allocated together, reducing the mix of fragment sizes.

A real allocator might have size classes like:

```
8, 16, 32, 64, 128, 256, 512, 1K, 2K, 4K, 8K, 16K, 32K, 64K, 128K, 256K, 512K, 1M, ...
```

The spacing between classes varies. Small sizes (8–64 bytes) increment by 8 or 16. Medium sizes (64–4K) double every few steps. Large sizes (>4K) grow exponentially. This is chosen so that the wasted space (internal fragmentation) on small allocations stays reasonable — usually under 25 percent.

Imagine you allocate 100 bytes with this scheme. It rounds to 128. You waste 28 bytes (22%). If you allocate 1000 bytes, it rounds to 1K (1024). You waste 24 bytes (2.4%). The waste percentage decreases as sizes grow, and the allocator is designed so that common sizes experience minimal waste.

Modern allocators (*libc malloc*, *jemalloc*, *tcmalloc*) use this approach.

### Per-Thread Arenas

The global heap is a bottleneck in multithreaded programs. Every allocation and free requires locking or atomic operations. *jemalloc* (used by FreeBSD by default) and *tcmalloc* (used by Google's services) split the heap into **arenas** — one per thread.

```
Thread 1   +----------+    Thread 2   +----------+
           | Arena 1  |              | Arena 2  |
           +----------+              +----------+
           (malloc on   (malloc on
            Thread 1)    Thread 2)
                       (minimal lock contention)
```

Each thread allocates and frees from its own arena with almost no locking. If Thread 1 allocates and Thread 2 frees, one of them might have to communicate, but usually they don't. This is a huge win in multithreaded workloads.

The downside: if one thread allocates a lot and then dies, its arena's memory is trapped; it can't be used by other threads. This is rare in practice (threads usually live for the entire process) but possible.

### SLUB Allocator (Kernel)

The Linux kernel's **SLUB** allocator (Slab Allocator) applies the same idea at the OS level.

Kernel code allocates memory for network buffers, VFS inodes, task structs, etc. These are often fixed-size (network buffers are always 2 KB, inodes are always ~500 bytes), so the kernel pre-sizes everything. It has slab caches:

```
slab_cache[inode_size]
slab_cache[dentry_size]
slab_cache[buffer_2k]
```

Each cache is a pool of fixed-size objects. Allocation is a pop from the pool; freeing is a push. The fragmentation within a slab is zero (every object is the same size). Fragmentation between slabs is minimal because you're not mixing sizes.

---

## 6.4 Pool and Arena Allocators

Sometimes you know your allocation pattern in advance. You're going to allocate a bunch of objects, use them, and free them all together. For that, an **arena allocator** (also called a **pool allocator**) is the right tool.

### The Pattern

```cpp
{
    ArenaAllocator arena;
    
    // Many allocations
    for (int i = 0; i < 1'000'000; ++i) {
        auto obj = arena.allocate(sizeof(MyObject));
        construct(obj);
    }
    
    // Use objects
    // ...
    
    // One operation: free them all
    arena.reset();  // or arena goes out of scope
}
```

This is common in:

- **Parsing.** Parse a file, allocate AST nodes, use the tree, then discard the entire tree.
- **Per-request allocation.** A web request allocates memory for parsing, processing, and rendering. At the end of the request, free everything.
- **Batch processing.** Read 1000 records, allocate a struct per record, process, write, throw away.

### Bump Allocator Implementation

The simplest arena allocator is a **bump allocator**. It is a single pointer that marches forward through a pre-allocated buffer. Allocation is a pointer bump; freeing is a no-op; at the end, reset the pointer.

```cpp
class BumpAllocator {
private:
    std::vector<std::byte> buffer;
    size_t offset = 0;

public:
    explicit BumpAllocator(size_t capacity) : buffer(capacity) {}

    void* allocate(size_t size) {
        if (offset + size > buffer.size()) {
            // Could grow the buffer here, or fail
            throw std::bad_alloc();
        }
        void* ptr = buffer.data() + offset;
        offset += size;
        return ptr;
    }

    // No individual free(); that's the whole point
    void reset() {
        offset = 0;  // Back to the start
    }

    size_t bytes_used() const { return offset; }
};
```

Usage:

```cpp
int main() {
    BumpAllocator arena(10 * 1024 * 1024);  // 10 MB buffer

    // Allocate a lot
    std::vector<int*> ptrs;
    for (int i = 0; i < 1'000'000; ++i) {
        ptrs.push_back((int*)arena.allocate(sizeof(int)));
    }

    // Free them all with one operation
    arena.reset();

    // Or let the allocator go out of scope
    return 0;
}
```

This is *dramatically* faster than `malloc`/`free`. No free list. No fragmentation. One `new` (the buffer). One `delete` (on reset). Allocation is a single pointer bump — typically one or two CPU instructions.

### When To Use It

Use a bump allocator when:

- You allocate many small objects.
- You free them all at roughly the same time.
- You don't need to free individual objects.

Do *not* use when:

- You allocate and free objects individually over a long time.
- You need to free some objects but keep others.

For the "right" use case, a bump allocator is 10–100× faster than `malloc`.

---

## 6.5 Compaction: Why C++ Can't, GCs Can

In a garbage-collected language (Java, Go, Python), the runtime can move objects around in memory. A **compacting GC** pauses the program, looks at which objects are still live, slides them together, and updates every pointer to point to the new location.

Before:

```
    +----------+----------+                  +----------+
    | USED obj | FREE     |                  | USED obj |
    +----------+----------+                  +----------+
     (after fragmentation)              (after compaction)
```

After:

```
    +----------+----------+----------+
    | USED obj | USED obj | USED obj |  ... contiguous, no holes
    +----------+----------+----------+
```

Then:

```
    for each object:
        update all references to it
```

This eliminates external fragmentation entirely. The heap is always densely packed.

### Why C++ Cannot Compact

C++ pointers are just addresses. They have no meaning outside the value itself. If I move an object from address `0x12345` to address `0x54321`, every pointer to it becomes invalid. To fix them, the runtime would need to:

1. Find every pointer to the object. (In a language with pointer arithmetic, this is a hard problem — is that `0x12340` a pointer to the object, or just a random value that happens to be nearby?)
2. Update it. (Okay.)

But the problem is *deeper*. In C++:

```cpp
int* p = &my_object;
void* raw = (void*)p;     // I just converted a pointer to a void*
send_over_network(raw);   // Sent to another process
// ooooops
```

A copy of the pointer now exists in a data structure or over the network. The runtime cannot track it. Compaction is impossible.

Even within a single program, consider:

```cpp
std::vector<int*> pointers;
MyObject obj;
pointers.push_back(&obj);  // Store its address

// If the GC moves 'obj', that pointer in the vector is now stale.
// The GC can't know to update it without scanning every bit of the heap.
```

The heap contains pointers, data, metadata, everything. Distinguishing a pointer from a random value is impossible. The GC would have to conservatively assume every word-aligned value *might* be a pointer, find it, and update it. This is incredibly expensive and error-prone.

This is a fundamental constraint of C++'s **pointer semantics**. Other languages avoid it:

- **Java references are not addresses.** A Java reference is an opaque handle looked up in a table. The GC can move the object and update the table entry.
- **Go pointers are the same as C++, but Go forbids storing pointers in data that outlives the object.** The runtime knows all pointers are "in-flight" in the stack or registers during the current function, so it can find and update them during GC.
- **Rust pointers obey ownership rules,** and the borrow checker ensures you cannot have a stale pointer. But Rust still cannot compact arbitrary owned data because the invariant is weaker than Java's.

C++ chose **not** to restrict pointers, which means C++ cannot safely compact. This is a foundational design decision, not an oversight.

### The Tradeoff

**Garbage-collected languages:**

- Pros: Automatic memory management, no fragmentation (GC compacts), no use-after-free.
- Cons: Pause times when GC runs (some GCs pause for 10–100+ ms). Prediction is hard.

**C++:**

- Pros: No GC pauses, exact control over when memory is freed, fine-grained performance predictability.
- Cons: Fragmentation over time, manual lifetime management, use-after-free risk, memory leaks.

Long-running services in C++ must manage fragmentation explicitly. Databases, game engines, high-frequency trading systems all have custom allocators. Garbage-collected languages have it easier at the cost of pauses.

---

## 6.6 Worked Example: Reproducing Fragmentation

Let's build a program that allocates random-size blocks, frees every other one, and watches the RSS grow. The key insight: after freeing 50% of blocks, the heap has 50% free space, but the largest contiguous block is much smaller than the heap size. This is external fragmentation in action.

The allocator's perspective is useful to visualize. After 100 allocations of random sizes (1 KB to 100 KB), the heap looks like a scattered patchwork. When we free every other block, we create holes. But those holes are scattered:

```
Block 1 (10 KB, USED)  Block 2 (30 KB, FREE)  Block 3 (5 KB, USED)
Block 4 (40 KB, FREE)  Block 5 (7 KB, USED)   Block 6 (35 KB, FREE)
...
```

Total free: ~50% of the heap. But the free blocks are small, and a request for 50 MB (which the heap technically has space for) fails because no 50 MB contiguous block exists.

Now let's code it:

```cpp
#include <cstdlib>
#include <vector>
#include <random>
#include <cstdio>
#include <chrono>
#include <unistd.h>

int main() {
    std::mt19937 rng(12345);
    std::uniform_int_distribution<size_t> sizes(1000, 100000);  // 1 KB to 100 KB

    std::vector<void*> ptrs;
    ptrs.reserve(100000);

    // Phase 1: allocate 100,000 blocks of random size
    fprintf(stderr, "Phase 1: Allocating 100,000 random-size blocks...\n");
    for (int i = 0; i < 100000; ++i) {
        size_t sz = sizes(rng);
        void* p = malloc(sz);
        if (!p) {
            fprintf(stderr, "malloc failed at iteration %d (sz %zu)\n", i, sz);
            return 1;
        }
        ptrs.push_back(p);
    }
    fprintf(stderr, "Phase 1 done. RSS is now elevated.\n");
    sleep(1);

    // Phase 2: free every other block
    fprintf(stderr, "Phase 2: Freeing every other block...\n");
    for (size_t i = 0; i < ptrs.size(); i += 2) {
        free(ptrs[i]);
    }
    fprintf(stderr, "Phase 2 done. Freed ~50%% of blocks, but RSS stays high (fragmentation).\n");
    sleep(1);

    // Phase 3: try to allocate a large block
    fprintf(stderr, "Phase 3: Trying to allocate a 50 MB block...\n");
    void* big = malloc(50 * 1024 * 1024);
    if (!big) {
        fprintf(stderr, "Failed to allocate 50 MB even though ~50%% of heap is free!\n");
        fprintf(stderr, "This is external fragmentation.\n");
    } else {
        fprintf(stderr, "Allocated 50 MB (lucky, or the allocator is good).\n");
        free(big);
    }

    // Cleanup
    for (size_t i = 1; i < ptrs.size(); i += 2) {
        free(ptrs[i]);
    }

    return 0;
}
```

Build and run:

```bash
clang++ -std=c++20 -O2 frag.cpp -o frag
./frag
```

Watch RSS in another terminal:

```bash
watch -n 0.1 'ps aux | grep frag | grep -v grep'
```

Or directly:

```bash
/usr/bin/time -v ./frag
```

You will see:

```
Phase 1: Allocating 100,000 random-size blocks...
Phase 1 done. RSS is now elevated.
Phase 2: Freeing every other block...
Phase 2 done. Freed ~50% of blocks, but RSS stays high (fragmentation).
Phase 3: Trying to allocate a 50 MB block...
Failed to allocate 50 MB even though ~50% of heap is free!
This is external fragmentation.
```

Now let's rewrite it with a bump allocator:

```cpp
#include <vector>
#include <random>
#include <cstdio>
#include <cstring>
#include <unistd.h>

class BumpAllocator {
    std::vector<std::byte> buffer;
    size_t offset = 0;
public:
    explicit BumpAllocator(size_t capacity) : buffer(capacity) {}
    
    void* allocate(size_t size) {
        if (offset + size > buffer.size()) return nullptr;
        void* ptr = buffer.data() + offset;
        offset += size;
        return ptr;
    }
    
    void reset() { offset = 0; }
    
    size_t capacity() const { return buffer.size(); }
};

int main() {
    std::mt19937 rng(12345);
    std::uniform_int_distribution<size_t> sizes(1000, 100000);

    BumpAllocator arena(6 * 1024 * 1024 * 1024);  // 6 GB arena
    std::vector<void*> ptrs;
    ptrs.reserve(100000);

    // Phase 1: allocate
    fprintf(stderr, "Phase 1: Allocating 100,000 random-size blocks (with bump allocator)...\n");
    for (int i = 0; i < 100000; ++i) {
        size_t sz = sizes(rng);
        void* p = arena.allocate(sz);
        if (!p) {
            fprintf(stderr, "allocate failed at iteration %d\n", i);
            return 1;
        }
        ptrs.push_back(p);
    }
    fprintf(stderr, "Phase 1 done. RSS is elevated but clean (no fragmentation).\n");
    sleep(1);

    // Phase 2: nothing to do! (no individual frees)
    fprintf(stderr, "Phase 2: Reset (free all at once)...\n");
    arena.reset();
    fprintf(stderr, "Phase 2 done. RSS drops immediately.\n");
    sleep(1);

    return 0;
}
```

Results: RSS stays low, allocation is fast, no fragmentation.

---

## 6.7 Tradeoffs

| Approach | Fragmentation | Allocation Speed | Best For | Worst For |
|---|---|---|---|---|
| glibc `malloc` | High over time (external); moderate (internal) | Medium (~100s of cycles) | General-purpose, diverse workloads | Long-running services, fragmentation-sensitive |
| jemalloc / tcmalloc | Lower (size classes, per-thread arenas) | Fast (~10s of cycles with arena hits) | Multithreaded services, moderate allocation pressure | Real-time systems (no guarantees) |
| Arena (bump) allocator | Zero (allocate-many-free-all pattern) | Very fast (~1 cycle per alloc) | Per-request workloads, parsing, batch processing | Long-lived objects with independent lifetimes |
| Pool allocator (fixed-size) | Zero (all objects same size) | Instant (~1 cycle) | Networking, fixed-size objects (buffers, packets) | Variably-sized allocations |
| Garbage collector (compacting) | Zero (compaction) | Medium (GC pauses 10-100+ ms) | Languages with pointer indirection (Java, Go) | Real-time systems, latency-sensitive code |

---

## 6.8 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Fragmentation means I have a memory leak." | No. A leak means allocated blocks are never freed. Fragmentation means free blocks are scattered and unusable. Different problems, different fixes. |
| "I should use `malloc` because it's always the best." | `malloc` is general-purpose but not best. Specialized allocators (jemalloc, tcmalloc, pools) outperform it for specific patterns. Profile, measure, choose. |
| "An arena allocator means I lose flexibility." | Yes, but that's a feature. If your workload *fits* the arena pattern (allocate-many, free-all), you get huge gains. If it doesn't fit, don't use it. |
| "Garbage collection solves fragmentation for free." | GC solves fragmentation by compacting, but you pay in pause time. A 100 ms GC pause is free memory you'll never get back. Tradeoff, not free. |
| "C++ can't have compacting GCs because pointers are addresses." | Right. This is a fundamental constraint. Other languages relax it. C++ chose flexibility; that choice has costs. |
| "I should minimize allocation by reusing objects." | Sometimes. But object pooling and reuse add complexity. Modern allocators are so fast that a simple allocation-per-use pattern often beats pooling. Measure. |
| "More total memory means less fragmentation." | False. If your working set grows, fragmentation can grow too. 10 GB with 9 GB working set can fragment as much as 4 GB with 3.5 GB working set. The ratio matters. |

---

## 6.9 Exercises

1. **Measure real fragmentation.** Write a program that allocates blocks, frees some, and prints (a) total RSS, (b) total allocated, (c) total free. On macOS use `vm_stat`; on Linux use `/proc/self/smaps`. Over time, track how RSS grows while allocated stays flat.

2. **Implement a pool allocator for fixed-size objects.** Design a simple memory pool:
   ```cpp
   template<typename T>
   class ObjectPool {
       std::vector<T> storage;
       std::vector<bool> free_list;
   public:
       T* allocate() { /* find a free slot */ }
       void free(T* p) { /* mark as free */ }
   };
   ```
   Benchmark it against `new`/`delete` for 1 million allocations. How much faster?

3. **Compare allocators.** Benchmark the same workload (10 million allocations of random sizes, then frees every other one) with (a) glibc `malloc`, (b) `jemalloc` (if available), (c) a bump allocator. Record RSS and time.

4. **Reproduce the fragmentation example.** Run the `frag.cpp` example above. Then modify it to use `jemalloc` instead of glibc `malloc`. Does the 50 MB allocation succeed? Why or why not?

5. **Observe size classes.** Write a tiny allocator that wraps `malloc` and prints every allocation request and the size class assigned:
   ```
   Request: 17 bytes -> size class 32
   Request: 100 bytes -> size class 128
   ...
   ```
   Allocate 10,000 random-size blocks (same distribution as §6.6). How much total internal fragmentation?

6. **Per-thread vs global arena.** Write a multithreaded program where each thread does many allocations and frees. Measure contention with (a) a single global heap (default `malloc`), (b) per-thread arenas (if you have jemalloc). Which is faster? By how much?

7. **Conceptual.** A coworker says "let's use an arena allocator for everything to avoid fragmentation." Write three follow-up questions to understand if that's the right choice for your workload.

---

## 6.10 Summary

Fragmentation is the silent killer of long-running services. External fragmentation — free memory scattered into pieces — accumulates over time as you allocate and free blocks of different sizes. Internal fragmentation — wasted space inside allocation blocks — is a constant but manageable overhead.

Modern allocators (jemalloc, tcmalloc) use size classes and per-thread arenas to reduce fragmentation. For workloads that fit the pattern (allocate-many, free-all), arena and bump allocators are dramatically faster and completely eliminate fragmentation. C++ cannot compact like GCs can, because pointers are addresses; this is a fundamental tradeoff.

Measure your allocation pattern. If you fit an arena allocator, use it. If you need general-purpose, consider jemalloc or tcmalloc. And remember: RSS growing while peak allocation stays flat is fragmentation, not a leak.

---

> **[← Previous: RAII Explained Deeply](05-raii-explained-deeply.md)** · **[↑ Part 2](README.md)** · **[Next: Caches and CPU Locality →](07-caches-and-cpu-locality.md)**
