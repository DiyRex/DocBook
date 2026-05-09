# Chapter 7 — Caches and CPU Locality

## Learning Objectives

By the end of this chapter you will be able to:

1. Describe the cache hierarchy — L1, L2, L3, main memory — and explain why the gap between CPU speed and memory speed has only grown wider over 30 years.
2. Recognize that memory does not move to the CPU one byte at a time; it moves in **cache lines** (typically 64 bytes), and understand how this shapes data structure design.
3. Define **spatial** and **temporal** locality, predict which access patterns exhibit each, and explain why prefetchers exploit them.
4. Trace the path of a memory access through the cache hierarchy: hit, miss, prefetch trigger, line eviction.
5. Predict, without profiling, whether a piece of code is cache-friendly or cache-hostile.
6. Run `perf stat` to measure cache misses, understand the output, and connect it to your code.
7. Recognize that for 90% of performance problems in computation-heavy code, the bottleneck is *not* algorithmic complexity, but *cache behavior*.

## The Memory Hierarchy: A 30-Year Gap

In Chapter 2 we first saw the memory hierarchy and its latencies. Let's make that concrete and understand why it matters.

A modern CPU core (Intel, AMD, Apple Silicon, 2025) runs at roughly 3–4 GHz. That is 0.25–0.33 nanoseconds per cycle. Main memory, by contrast, responds in ~50–100 nanoseconds. That is **a 150–400× difference in latency**.

Here is the full hierarchy:

```
Component              Latency              Size per core    Total per chip
L1 cache               ~4 cycles (1 ns)     32–64 KB         same per core
L2 cache               ~12 cycles (3 ns)    256 KB–1 MB      same per core
L3 cache               ~40 cycles (10 ns)   4–64 MB          shared
Main memory (RAM)      ~200 cycles (50 ns)  ∞ (slow)         8 GB–1 TB
SSD                    ~50,000 cycles       ∞                100 GB–10 TB
Network                millions             ∞                ∞
```

This is not a small difference. A cache miss to main memory costs you the same amount of CPU time as **200 simple arithmetic instructions**. If your code misses the cache frequently, the CPU spends almost all its time *waiting*, not computing.

To visualize: if an L1 hit is one second of wall-clock time, a main-memory miss is 200 seconds — more than 3 minutes. That is the performance gap you are managing.

### Why the gap has grown, not shrunk

You might reasonably ask: "Why don't CPU makers just make memory as fast as the CPU?" The answer is physics.

Memory speed is limited by:
- **Propagation delay** — signals travel at light speed through silicon, and distances add up.
- **Density** — fast memory is power-hungry and generates heat. You cannot make the entire system run at L1 speed without melting it.
- **Capacity** — you could have 64 MB of L1 cache per core, but that would use enormous die area and power.

The CPU makers chose **throughput** (making chips fast) over memory latency (making RAM fast). So the latency gap has actually *widened* over 30 years, not narrowed. In 1995, a memory access was maybe 10–20× slower than a CPU cycle. Today it is 200–400×. The gap is *worse* than it was.

The only defense against this gap is the **cache hierarchy**: small, fast memories that sit between the CPU and main memory, and a layer of logic that moves data intelligently. The cache is not an optimization. It is the foundation of computing. Without it, modern CPUs would be useless.

## Cache Lines, Not Bytes

This is crucial: memory does not move into the CPU one byte at a time. It moves in fixed-size chunks called **cache lines**. On nearly every modern CPU, a cache line is **64 bytes**.

When you access a single `int` (4 bytes) that is not yet in cache, the CPU does not fetch 4 bytes from RAM. It fetches 64 bytes — the entire cache line that contains that `int` — and places it in L1 cache. The remaining 60 bytes come along for free.

This is called **spatial locality exploitation**: the assumption that if you need byte X, you will probably need bytes X+1, X+2, etc. soon.

### Why this matters for data structure design

Consider these two access patterns:

**Scenario A: Contiguous array traversal**

```cpp
std::vector<int> data(10'000);
for (int x : data) {
    total += x;
}
```

On the first access to `data[0]`, the CPU fetches a 64-byte cache line. That line contains `data[0]` through `data[15]` (64 bytes / 4 bytes per int). The next 15 iterations are L1 cache hits — essentially free. Only after iteration 15 does the CPU fetch the next cache line.

For 10,000 ints, you have approximately 625 cache misses (10,000 / 16). Cost: ~625 × 50 ns = **31 microseconds** for memory fetches. The arithmetic itself takes a few microseconds. Memory is the bottleneck, but at least the cache lines are being exploited.

**Scenario B: Linked list traversal**

```cpp
struct Node { int value; Node* next; };
Node* head = ...;
long long total = 0;
for (Node* n = head; n; n = n->next) {
    total += n->value;
}
```

Each `Node` is allocated independently on the heap. The `next` pointers point to arbitrary memory addresses. When you fetch `n->next`, that pointer almost certainly points to a cache line you do not have. You miss the cache, wait 50 ns, load the line, extract the pointer, and immediately miss the cache again. Ten thousand iterations mean roughly ten thousand cache misses.

Cost: ~10,000 × 50 ns = **500 microseconds** for memory fetches alone. The arithmetic is irrelevant.

Same operation count. **Same algorithmic complexity.** The linked list is **16× slower** because it does not respect spatial locality.

This is why the benchmark in Chapter 2 showed such a dramatic gap. This is why linked lists are almost always the wrong choice. This is why "data-oriented design" is not a vague principle but a mathematical fact.

### Cache line awareness in practice

This shapes every data structure decision:

- **Struct layout matters.** If you have a hot field and a cold field in the same struct, and they share a cache line, the CPU loads both every time it needs the hot field. Consider splitting them.
- **Padding sometimes helps.** If two threads write to adjacent fields on the same cache line, you get false sharing (we'll return to this). A strategic `char pad[56];` can push thread A's field onto its own line.
- **Array-of-structs vs struct-of-arrays** depends on access patterns. If you access `.x` and `.y` of all points, struct-of-arrays (SoA) keeps related data together in a cache line. If you access one point at a time, array-of-structs (AoS) keeps point data together.

For now, the lesson: **64-byte cache lines are the unit of currency in performance.** When you design a data structure, you are not designing it for the CPU; you are designing it for the cache.

## Spatial and Temporal Locality

Memory access patterns fall into two categories:

### Spatial Locality

You access memory locations that are *close together* in the address space.

**Example:**
```cpp
for (int i = 0; i < N; ++i) {
    arr[i] = arr[i] * 2;
}
```

Each iteration accesses `arr[i]`, then `arr[i+1]`. If the array is contiguous, `arr[i+1]` is likely on the same cache line as `arr[i]`. High spatial locality.

**Counter-example:**
```cpp
int* arr = malloc(large_size);
for (int i = 0; i < N; ++i) {
    arr[i * 1024] = arr[i * 1024] * 2;  // stride of 1024
}
```

Each iteration jumps 1024 elements forward. If 1024 elements span multiple cache lines, you are missing on almost every access. Low spatial locality.

### Temporal Locality

You access the *same* memory location multiple times within a short span of code.

**Example:**
```cpp
int x = arr[0];  // fetch arr[0]
int y = x + 1;
int z = x * 2;   // x is in a register, no fetch
int w = x / 3;   // still in a register
```

You fetch `arr[0]` once and use it three times. That is high temporal locality.

**Counter-example:**
```cpp
for (int i = 0; i < 10'000'000; ++i) {
    arr[i] = expensive_function(arr[i]);
}
```

You fetch each array element once, use it once, and never touch it again. Low temporal locality (for the array, though possibly high for instruction cache).

### Why prefetchers care

Modern CPUs have **hardware prefetchers** — circuits that predict future memory accesses and fetch data before it is needed. They exploit both patterns:

- **Spatial prefetchers** detect linear, contiguous access and speculatively fetch the *next* cache line before you ask for it.
- **Stride prefetchers** detect patterns like "every 4th element" and fetch accordingly.

This is why linear traversal is so much faster than random access, even for the exact same number of iterations. The prefetcher sees `arr[i]`, then `arr[i+1]`, and predicts `arr[i+2]` and `arr[i+3]` are coming. By the time you need them, they are already in cache.

With random access — `arr[rand()]` — the prefetcher cannot predict anything. Every access is a cache miss.

## Hardware Prefetchers and Prefetching Behavior

On modern CPUs, prefetchers are not optional; they are a core part of the cache system. Understanding them changes how you think about performance.

### Next-line prefetcher

The simplest prefetcher fetches the next sequential cache line. If you miss the cache accessing line N, the prefetcher issues a request for line N+1 before you ask for it.

This is extremely effective for the linear array case. By the time you iterate to the next element, it is already in cache.

### Stride prefetcher

More sophisticated CPUs track *strides* — repeated memory access intervals. If you access addresses 0x1000, 0x1040, 0x1080 (stride of 64 bytes — exactly one cache line), the prefetcher detects the pattern and fetches 0x10C0 before you ask for it.

Strides do not have to be regular through the whole access. The prefetcher detects the last two or three accesses, infers a pattern, and acts on it.

### Prefetch thrashing and misallocation

Prefetchers can also work against you. If you access memory in a chaotic, unpredictable pattern, the prefetcher may:

- **Fetch the wrong line** — a false prediction wastes memory bandwidth and cache capacity.
- **Evict useful data** — the prefetched but unneeded line evicts a line you will need soon.
- **Waste power** — bus accesses are expensive; fetching useless data burns energy.

This is why "random access is slow" is not entirely true. The slowness comes from both the miss latency *and* the prefetcher's inability to help you.

## Cache-Friendly Data Structures

Let's compare common data structures through the lens of cache behavior.

### Linked list vs vector

**`std::list<int>` (doubly-linked list):**

```cpp
struct ListNode {
    int data;
    ListNode* prev;
    ListNode* next;
};
```

Each node is individually allocated on the heap. Nodes are scattered across the address space. Walking the list is a succession of pointer dereferences, each of which is likely a cache miss. Even with 10 million elements, you might achieve only ~200 MB/s of memory throughput, when the CPU is capable of 50+ GB/s.

**`std::vector<int>` (contiguous array):**

Stores all ints back-to-back in memory. Walking it is a stream of sequential addresses. The prefetcher anticipates every fetch. Memory throughput: 40+ GB/s.

**The gap is not 2×. It is 100–200×.**

Yet many codebases use `std::list` because the interface is convenient — it supports efficient insertion and deletion anywhere in the list. The *algorithm* is O(n) for list, O(n) for vector (if inserts are in the middle). But the constant factors are so different that they might as well be different complexities.

Rule of thumb: **use `std::vector` or `std::deque` unless you have a specific reason (and a benchmark) to use `std::list`.** The reason is almost never what you think it is.

### Hash map with chaining vs. open addressing

**Hash map with chaining:**

```cpp
struct Bucket { std::list<Pair> chains; };
std::vector<Bucket> table;
```

Multiple keys hash to the same bucket. Instead of storing them in the bucket, you store a `std::list` of entries. A hash miss means walking a linked list. Cache-hostile.

**Open addressing (linear probing, quadratic probing, etc.):**

```cpp
std::vector<Pair> table;
```

All entries are in a single contiguous array. On a collision, you probe the next slot in the array. Even with probe sequences that wrap around, you stay relatively close in the address space. Much better cache behavior.

Also consider that lookup cost, in the chaining case, is O(1) *expected* but O(n) *worst case*, while open addressing with a good probing strategy is more consistent.

**Recommendation:** use `std::unordered_map` (which uses chaining) only if you know you need the stability guarantees. Otherwise, consider a flat hash map or open addressing. Or use `std::map` (a tree), which has better cache locality than you'd expect because tree nodes can be packed densely on the heap if allocated with a pool.

### Array-of-structs vs struct-of-arrays

Suppose you have 10,000 points, each with x, y, z coordinates.

**Array-of-structs (AoS):**

```cpp
struct Point { float x, y, z; };
std::vector<Point> points(10'000);

for (auto& p : points) {
    p.x *= 2;
    p.y *= 2;
    p.z *= 2;
}
```

Each `Point` is 12 bytes. A cache line holds roughly 5 points. Good cache locality if you access all three coordinates of a point. Poor locality if you only access `.x` of all points, because you load `.y` and `.z` but do not use them.

**Struct-of-arrays (SoA):**

```cpp
struct Points {
    std::vector<float> x, y, z;
};
Points points;
points.x.resize(10'000);
points.y.resize(10'000);
points.z.resize(10'000);

for (int i = 0; i < 10'000; ++i) {
    points.x[i] *= 2;
    points.y[i] *= 2;
    points.z[i] *= 2;
}
```

Each coordinate is stored separately, contiguously. A cache line holds 16 floats (64 bytes / 4 bytes). If you transform all three coordinates per iteration, you still load three separate cache lines, but you get perfect spatial locality for each coordinate.

If you only transform `.x`, you load only one array, and get 16× more x-values per cache miss compared to AoS.

**The choice depends on your access pattern:**

- **One point at a time**: AoS
- **All of one coordinate**: SoA
- **Mixed**: probably AoS with padding to align hot fields

This is not "one is always better." It is "match the data structure to the access pattern."

## Worked Example: Row-Major vs Column-Major Traversal

Let's benchmark the classic example that shows cache behavior.

**The code:**

```cpp
#include <chrono>
#include <cstdio>
#include <cstring>

constexpr int N = 1024;
int matrix[N][N];

int main() {
    // Initialize (row-major is cache-friendly)
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            matrix[i][j] = i * N + j;
        }
    }

    // Benchmark 1: Row-major (fast)
    auto t0 = std::chrono::steady_clock::now();
    long long sum_rows = 0;
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            sum_rows += matrix[i][j];
        }
    }
    auto t1 = std::chrono::steady_clock::now();
    auto row_time = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    std::printf("Row-major:    %lld us (sum=%lld)\n", row_time, sum_rows);

    // Benchmark 2: Column-major (slow)
    t0 = std::chrono::steady_clock::now();
    long long sum_cols = 0;
    for (int j = 0; j < N; ++j) {
        for (int i = 0; i < N; ++i) {
            sum_cols += matrix[i][j];
        }
    }
    t1 = std::chrono::steady_clock::now();
    auto col_time = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    std::printf("Column-major: %lld us (sum=%lld)\n", col_time, sum_cols);
    std::printf("Ratio: %.1f×\n", (double)col_time / row_time);

    return 0;
}
```

**Expected output:**

```
Row-major:    700 us (sum=536854528)
Column-major: 7500 us (sum=536854528)
Ratio: 10.7×
```

(Exact numbers vary by CPU, but the ratio is consistently 5–15×.)

**Why the gap?**

In C++, arrays are stored in **row-major order**: `matrix[i][j]` occupies memory at address `base + i * N * 4 + j * 4`. Successive elements in row i are 4 bytes apart. Successive elements in column j are N * 4 bytes apart (4096 bytes on a 1024×1024 matrix).

When you traverse row-major:
- Access `matrix[0][0]` → cache miss, fetch line containing `matrix[0][0]..matrix[0][15]`.
- Access `matrix[0][1]..matrix[0][15]` → cache hits (same line).
- Access `matrix[0][16]` → miss, fetch next line. But the prefetcher has likely already fetched it.

When you traverse column-major:
- Access `matrix[0][0]` → cache miss, fetch that line.
- Access `matrix[1][0]` → 4096 bytes away, almost certainly a miss.
- Access `matrix[2][0]` → another miss.
- Every single access is a cache miss (or a hit in L3, which is slow).

Same number of operations. Same number of accesses. The difference is pure cache behavior.

### Observing with perf

Run the benchmark and measure cache behavior:

```bash
clang++ -std=c++20 -O2 matrix.cpp -o matrix
perf stat -e cache-references,cache-misses,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./matrix
```

**Output (example):**

```
Row-major:    710 us (sum=536854528)
Column-major: 8200 us (sum=536854528)
Ratio: 11.5×

Performance counter stats for './matrix':
    1,048,576 cache-references
       10,320 cache-misses       #    0.98% of all cache refs
      125,436 L1-dcache-load-misses
      131,072 LLC-loads
      131,070 LLC-load-misses    # <-- Here: almost every access misses L3
```

The key line: **L3 load misses is nearly equal to L3 loads.** That means almost every access is missing the cache hierarchy and going to main memory. Column-major traversal has defeated the prefetcher and the entire cache system.

Row-major, by contrast:

```
    1,048,576 cache-references
      600,000 cache-misses       #    0.06% of all cache refs
       50,000 L1-dcache-load-misses
      131,072 LLC-loads
        1,000 LLC-load-misses    # <-- Only 1,000 misses, not 131,072
```

The LLC (last-level cache, L3) is doing its job: 131,072 loads, 1,000 misses. The prefetcher is working. Row-major stays in cache.

## False Sharing (Forward Reference)

Before we move on, a brief mention: on systems with multiple cores, a related problem arises. Suppose thread A writes to `shared_data[0]` and thread B writes to `shared_data[1]`, and both integers are on the same 64-byte cache line. Even though they are writing to *different* addresses, the cache line bounces between the cores: A's core loads it, writes its word, and the line is invalidated (cache coherence); B's core loads it, writes its word, and it is invalidated again.

Neither thread is accessing data from the other thread, but they are causing cache misses due to *false sharing*. The cache line is shared even though the *data* is not.

We will return to this in Part 8 (Concurrency), but it is worth knowing that not all cache misses come from capacity or compulsory misses. Some come from contention on cache lines between cores.

## Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Linked list** | Efficient insertion/deletion anywhere | Cache-hostile; hard to parallelize; prefetchers useless | Almost never. Pools or other structures usually beat it. |
| **Vector** | Cache-friendly linear access; prefetcher friendly | Expensive insert/delete in the middle; reallocations | Default choice for most ordered collections. |
| **Hash map (chaining)** | Simple to implement; handles collisions predictably | Cache-hostile walk of collision chains | When you know collisions will be rare and chain length small. Prefer open addressing. |
| **Hash map (open addressing)** | Better cache locality; denser memory usage | Poor cache behavior on collision chains (still better than chaining); harder to remove | Modern default for hash maps. |
| **Tree (B-tree, AVL)** | Good cache locality if packed; O(log N) guaranteed; ordered | Pointer chasing; more memory overhead per node | When you need ordering and dynamic size. |
| **Flat hash map or Swiss table** | Excellent cache locality; modern CPU optimizations | Requires careful rehashing; larger working set if sparse | High-performance code; profiling shows hash is a bottleneck. |
| **Array-of-structs** | Access one point/entity at a time efficiently | Wastes cache on unused fields; poor if you access one coordinate across all entities | When entities are accessed as units. |
| **Struct-of-arrays** | Excellent if you transform/operate on one attribute across all entities; vectorization friendly | Complex memory layout; poor if you access one entity at a time | Data-parallel processing; SIMD workloads; columnar databases. |

The theme: **layout and access pattern are king.** No abstract data structure is universally best. Measure.

## Common Misconceptions

1. **"My code is slow because I'm not using the right algorithm."** Maybe, but unlikely. Most slow code is slow because the working set doesn't fit in cache, or access is random. Big-O complexity explains *scaling*, not *constant factors*. Constant factors dominate in real performance.

2. **"Caches are transparent; I don't need to think about them."** Caches are transparent to *correctness* but not to *performance*. You can ignore them until performance matters, then you must understand them.

3. **"More cache is always better."** More cache helps you fit more data, but the latency of L3 is 10× that of L1. A smaller L1-resident working set often beats a larger L3-resident one.

4. **"Random access is slow only because RAM is slow."** True, but RAM slowness is a constant you cannot change. What you *can* change is *how much* RAM you hit. Random access means no prefetcher help, which means every miss goes to the slowest level.

5. **"Optimizing for cache is premature optimization if I don't have a benchmark."** False. Cache-friendly patterns (contiguous arrays, sequential access) are also the *simplest* to code. They are not a complexity-performance tradeoff; they are superior in both directions.

6. **"Cache line size is only relevant to CPU designers."** No. Knowing your CPU's cache line size (usually 64 bytes) is as important as knowing your stack size. It shapes struct packing, alignment, and false sharing.

## Exercises

1. **Reproduce the matrix benchmark.** Save the code above, compile with `-O2`, and run with `perf stat`. Record the row-major and column-major times. Try a different matrix size (e.g., 512, 2048). At what size does the gap grow larger or smaller? Why?

2. **Cache line awareness in structs.** Write a struct with fields you know are accessed at different rates:
   ```cpp
   struct HotCold {
       int hot;         // accessed in the inner loop
       char cold[1000]; // rarely accessed
   };
   ```
   Write a benchmark that sums `hot` over an array of `HotCold`, first with the fields in one struct, then with them separated (or with padding). Measure with `perf`. Does separating improve the cache-miss ratio? By how much?

3. **List vs vector in practice.** Write a function that takes 10,000 values, inserts them into either a `std::list` or `std::vector` at random positions (using insertion, not just appending), then traverses the result. Measure both construction time and traversal time. Which is faster overall? (You may be surprised.)

4. **Stride detection.** Write code that accesses an array with a stride larger than a cache line (e.g., every 64th element, or every 256th). Run `perf stat` to measure cache misses. Do you see a pattern? Can you modify the stride so that the prefetcher can help?

5. **Conceptual.** A coworker says "we're using a linked list because we need to insert and remove items frequently." Before agreeing, what three questions would you ask, drawing on this chapter? (One example: "How often do we traverse the list compared to inserting?")

6. **SoA vs AoS benchmark.** Implement both versions of the Points example above and benchmark a transformation that modifies all three coordinates vs. one that modifies only `.x`. Record the time for each version on each operation. What do you learn?

7. **perf stat output interpretation.** Run a simple loop over an array with `perf stat -e cache-references,cache-misses,LLC-loads,LLC-load-misses` and interpret each value. What fraction of your accesses hit L1? Miss to main memory? What does that tell you?

## Summary

Caches are not a performance optimization; they are an architectural necessity. The CPU is hundreds of times faster than main memory, and the gap widens every year. The cache hierarchy is the workaround, and it works by assuming your code exhibits spatial and temporal locality. Linear access patterns, contiguous arrays, and cache-line-aware layout are not fussy or low-level; they are the foundation of every high-performance system.

If your code is slow and not bottlenecked on I/O, it is almost certainly cache-bound. Measure with `perf stat`. Understand the miss patterns. Choose data structures that respect your access patterns. The difference between cache-friendly and cache-hostile is often 5–20×, and no algorithmic optimization can overcome a 5–20× disadvantage.

The next chapter, Data-Oriented Design, formalizes this principle: build systems around the data and access patterns, not around conceptual abstractions.

---

> **[← Previous: Memory Fragmentation](06-memory-fragmentation.md)**  ·  **[↑ Part 2](README.md)**  ·  **[Next: Data-Oriented Design →](08-data-oriented-design.md)**
