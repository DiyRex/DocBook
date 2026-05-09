# Chapter 89 — Cache Misses (Deeper)

## Learning Objectives

By the end of this chapter you will be able to:

1. Classify cache misses into three types (compulsory, capacity, conflict) and determine which type is limiting your code using `perf stat`.
2. Measure cache performance on real hardware using Linux perf, Intel VTune, and AMD uProf, and interpret miss rates in context of your workload.
3. Apply data layout transformations — array-of-structs to struct-of-arrays, hot/cold splitting, field reordering — to reduce misses.
4. Implement loop tiling (cache blocking) to keep working sets in cache, with measured speedups of 5–15×.
5. Apply software prefetching (`__builtin_prefetch`) effectively, understanding when it helps and when it hurts.
6. Recognize TLB (Translation Lookaside Buffer) misses as a distinct performance problem and apply hugepages to reduce them.
7. Understand the tradeoffs between miss-reduction techniques and know which to apply first.
8. Run a matrix multiply microbenchmark from naive (cache-hostile) to tiled (cache-friendly) and see the progression of optimizations.

---

## Recap: What We Know About Misses

Chapter 7 introduced the cache hierarchy and basic locality. Quick recap:

- **Cache lines are 64 bytes.** Memory moves in 64-byte chunks, not individual bytes.
- **Spatial locality:** if you access one address, you likely need nearby addresses (exploit by loading cache lines).
- **Temporal locality:** if you access one address, you likely need it again soon (keep it in cache).
- **Prefetchers:** modern CPUs predict future accesses and fetch proactively.

This chapter goes deeper: **what happens when you miss?** And more importantly, **how do you measure which kind of miss you have, and what do you do about it?**

---

## The Three Kinds of Misses (The 3 C's)

All cache misses fall into three categories. Knowing which type is limiting you determines your optimization strategy.

### Compulsory Misses

A **compulsory miss** is the first access to a cache line. The line is not in cache, so it must be fetched from a lower level of the hierarchy (or from main memory).

Every program has some compulsory misses — they are unavoidable. If your program needs to load 1 GB of data from disk, that is 1 GB / 64 bytes ≈ 16 million compulsory misses just to bring the data into memory.

**You cannot eliminate compulsory misses.** You can only minimize them by:
- Reducing the working set (use smaller data types, compress)
- Reducing the number of unique accesses (reuse data)

### Capacity Misses

A **capacity miss** occurs when your working set exceeds the size of the cache. The CPU evicts useful data to make room for new data, then later needs the evicted data again.

Example: An L1 cache is 32 KB. If your code accesses 64 KB of data in a tight loop, the CPU will load 32 KB, then as you access the next 32 KB, it evicts the first 32 KB. When you loop back to the first 32 KB, you miss again.

**How to detect:** Compare the number of cache misses to the cache size and your working set size:
```
miss_rate = misses / accesses
effective_working_set = accesses * miss_rate * cache_line_size

if effective_working_set > cache_size:
    capacity miss problem
```

More pragmatically: if you can fit your hot data in a smaller cache but your code still misses, you have a capacity problem.

**Solutions:**
- **Reduce working set:** Use smaller data types, compress, filter early.
- **Fit in cache:** Redesign the algorithm to work on smaller chunks (loop tiling).

### Conflict Misses

A **conflict miss** occurs in set-associative caches when multiple data items map to the same cache set, even though there is space elsewhere in the cache.

A modern CPU cache is typically 8-way associative, meaning each cache set can hold 8 different lines. If you access 9 items that map to the same set, the 9th will evict the 1st, causing a miss even if other sets in the cache are empty.

**Example:** An L1 cache with 32 KB and 64-byte lines has 512 lines (32 KB / 64 B). With 8-way associativity, there are 512 / 8 = 64 sets. Items whose addresses hash to the same set will conflict.

On modern CPUs (Intel, AMD), conflicts are rare because of high associativity. **Conflict misses are usually not your bottleneck.** If you suspect conflicts, you can measure with `perf` (see below).

---

## Measuring Cache Misses on Real Hardware

### Using `perf stat`

The simplest measurement is Linux `perf stat`. It uses hardware performance counters to count misses.

```bash
perf stat -e cache-references,cache-misses,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./your_program
```

**Output (example):**

```
Performance counter stats for './sum_array':
    536,870,912 cache-references
         98,765 cache-misses                # 0.02% of all cache refs
      1,234,567 L1-dcache-load-misses
      2,345,678 LLC-loads
         12,345 LLC-load-misses             # 0.53% of all LLC loads
```

**How to interpret:**

1. **cache-misses / cache-references:** This is the overall miss rate. 0.02% is excellent. > 5% indicates a problem.

2. **L1-dcache-load-misses:** Misses in the L1 data cache (most performance-sensitive). This is where the majority of your misses likely are.

3. **LLC-load-misses:** Misses in the last-level cache (L3). These go all the way to main memory (100 ns latency). Even a small percentage here can be expensive.

4. **Ratio of LLC-load-misses / LLC-loads:** If this ratio is > 1%, you are hitting main memory often. Dangerous.

**Comparing miss rates:** What is "good"?

- **Sequential array access:** 0.1–0.5% (prefetcher handles it)
- **Strided access (predictable):** 1–5% (prefetcher detects the stride)
- **Cache-resident working set:** 0.1–2% (good spatial locality)
- **Random access:** 50–100% (every access is a miss)
- **Capacity-limited:** 10–30% (working set is larger than cache)

### Using Intel VTune (Windows, Linux, macOS)

VTune provides deeper hardware insights than `perf stat`:

```bash
vtune -collect memory-access -app-working-dir . -- ./your_program
vtune -report summary -r ./r000ma
```

Output shows:
- **L1 hit rate, L2 hit rate, L3 hit rate** (separately)
- **Memory bandwidth saturation** (% of peak bandwidth used)
- **Cache line utilization** (how many bytes per fetched line are actually used)
- **False sharing** (cache lines bouncing between cores)

VTune is invaluable for detailed tuning but requires a license (free community edition exists).

### Using AMD uProf (AMD Ryzen / EPYC)

Similar to VTune, but optimized for AMD hardware:

```bash
uprof record -O ./your_program
uprof translate -O ./output.csv
```

### Interpreting the Bottleneck Type

Once you have miss counts, categorize:

**If LLC-load-misses are low (< 1% of LLC loads):**
- Your cache hierarchy is working. Misses are mostly L1 hits in L2/L3 (relatively fast, ~10 ns).
- Problem: Maybe not misses, but **branch mispredictions** or **dependency chains** (compute-bound, not memory-bound).

**If LLC-load-misses are high (> 5% of LLC loads):**
- You are hitting main memory often. This is **capacity** or **compulsory** misses.
- **Capacity miss?** Does your working set fit in L3 (typically 4–32 MB)? If not, you have a capacity problem.
- **Compulsory miss?** Is this your first pass through the data? Compulsory misses are unavoidable on the first pass, but if you are looping, subsequent passes should have lower miss rates.

**If L1-dcache-load-misses are high but LLC-load-misses are low:**
- You are missing L1, but hitting L2/L3. Your working set fits in cache, but the access pattern is not spatially local.
- **Solution:** Improve locality (reorder loops, reorder data).

---

## Common Patterns and Their Miss Rates

### Sequential Array Access (Best Case)

```cpp
std::vector<int> data(10'000'000);
for (int x : data) {
    total += x;
}
```

**Expected miss rate:** 0.1–0.5%

**Why:** The prefetcher detects linear access and fetches ahead. Each miss brings in 16 ints (64 bytes / 4 bytes). Amortizing: ~10,000,000 / 16 ≈ 625,000 misses.

**Measurement:**
```bash
perf stat ./sequential_sum
L1-dcache-load-misses: ~600,000
LLC-load-misses: ~100 (prefetcher handles everything, L3 hits most)
```

### Strided Access (Prefetcher Helps if Stride is Small)

```cpp
std::vector<int> data(10'000'000);
for (int i = 0; i < data.size(); i += 4) {  // stride of 4 ints = 16 bytes
    total += data[i];
}
```

**Expected miss rate:** 1–3%

**Why:** The prefetcher can detect the stride (every 4 accesses = 16 bytes apart). It fetches ahead, but the misses are higher than sequential because the stride is explicit.

### Strided Access Beyond Prefetcher (Prefetcher Fails)

```cpp
std::vector<int> data(10'000'000);
for (int i = 0; i < data.size(); i += 256) {  // stride of 256 ints = 1024 bytes
    total += data[i];
}
```

**Expected miss rate:** 30–50%

**Why:** The stride is large (1024 bytes = 16 cache lines). Each access is to a new cache line, and the prefetcher cannot keep up. Most accesses miss.

### Random Access (Worst Case)

```cpp
std::vector<int> data(10'000'000);
std::vector<int> indices(10'000'000);
// Populate indices with random values
for (int idx : indices) {
    total += data[idx];
}
```

**Expected miss rate:** 80–100%

**Why:** Each access is to an unpredictable address. The prefetcher cannot predict anything. Almost every access is a cache miss.

---

## Remediation Patterns

Once you have identified a miss problem, apply these patterns in order of cost/benefit.

### Pattern 1: Reorder Data (Array-of-Structs → Struct-of-Arrays)

If you are missing cache but only accessing one field of a struct, switch to struct-of-arrays.

**Before (Array-of-Structs):**

```cpp
struct Particle {
    float x, y, z, vx, vy, vz;  // 24 bytes
};
std::vector<Particle> particles(1'000'000);

// Update only positions
for (auto& p : particles) {
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.z += p.vz * dt;
}
```

**Problem:** You load 24 bytes (the whole struct) but only use 6 bytes (x, y, z). The prefetcher brings in the velocity fields too. Wasted cache traffic.

**After (Struct-of-Arrays):**

```cpp
struct ParticleArrays {
    std::vector<float> x, y, z, vx, vy, vz;
};

// Update only positions
for (int i = 0; i < particles.x.size(); ++i) {
    particles.x[i] += particles.vx[i] * dt;
    particles.y[i] += particles.vy[i] * dt;
    particles.z[i] += particles.vz[i] * dt;
}
```

**Benefit:** Each array is loaded independently. If you access position, you load only the position cache lines (no wasted velocity data).

**Measurement:**

```bash
# Before
perf stat ./before
    L1-dcache-load-misses: 2,000,000
    LLC-load-misses: 50,000

# After
perf stat ./after
    L1-dcache-load-misses: 500,000
    LLC-load-misses: 5,000
```

**Speedup:** 4–10× depending on workload.

### Pattern 2: Hot/Cold Splitting

If a struct has fields with different access frequencies, split them.

**Before:**

```cpp
struct Record {
    int hot_field;      // accessed every iteration
    char cold_data[1000]; // accessed rarely
};
std::vector<Record> records(100'000);

for (auto& r : records) {
    total += r.hot_field;
}
```

**Problem:** Each access to `hot_field` brings in the 1000-byte `cold_data`. Wasted cache.

**After:**

```cpp
struct RecordHot {
    int hot_field;
};
struct RecordCold {
    char cold_data[1000];
};

std::vector<RecordHot> hot(100'000);
std::vector<RecordCold> cold(100'000);

for (auto& r : hot) {
    total += r.hot_field;
}
```

**Benefit:** Hot loop only loads the hot data.

### Pattern 3: Field Reordering in Structs

If you access fields in a specific order, reorder them in memory to improve spatial locality.

**Example:**

```cpp
// Before: fields in logical order, but poor access locality
struct Config {
    int id;
    std::string name;
    float value;
    std::vector<int> options;
    bool enabled;
};

// Typical access: iterate and sum value
for (const auto& c : configs) {
    total += c.value;
}
```

**Problem:** To access `value`, you load the struct, which includes `id`, `name`, and other fields. If you only care about `value` and `enabled`, the rest is wasted.

**After: Reorder for access pattern**

```cpp
struct Config {
    float value;
    bool enabled;
    int id;
    std::string name;
    std::vector<int> options;
};
```

**Benefit:** Hot fields (`value`, `enabled`) share a cache line. Cold fields (`options`) are at the end and not loaded unnecessarily.

### Pattern 4: Loop Tiling (Cache Blocking)

For multi-level loops over large data, tile the loops to keep working sets in cache.

**Classic example: Matrix multiply**

**Naive (cache-hostile):**

```cpp
// Matrix multiply: C = A * B
// A is M×K, B is K×N, C is M×N
void matmul_naive(float A[M][K], float B[K][N], float C[M][N]) {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            float sum = 0;
            for (int k = 0; k < K; ++k) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
}
```

**Why it is slow:**

- For each output element C[i][j], we iterate through all K elements of row A[i] and column B[*][j].
- The access to B[k][j] is column-major (stride of N elements). In a 1000×1000 matrix, each miss is 1000 * 4 bytes = 4000 bytes away from the previous access.
- The prefetcher cannot help. Result: 10–30 cache misses per output element (catastrophic).

**Example numbers (1000×1000 matrices):**

- Time: ~50 seconds
- Cache misses: ~500 billion
- Memory bandwidth usage: ~100% (completely memory-bound, no computation)

**Tiled version (cache-friendly):**

```cpp
// Tile size: choose to fit working set in L3
constexpr int TILE = 64;

void matmul_tiled(float A[M][K], float B[K][N], float C[M][N]) {
    // Iterate over tiles
    for (int ii = 0; ii < M; ii += TILE) {
        for (int jj = 0; jj < N; jj += TILE) {
            for (int kk = 0; kk < K; kk += TILE) {
                // Compute one tile of C (TILE×TILE)
                for (int i = ii; i < std::min(ii + TILE, M); ++i) {
                    for (int j = jj; j < std::min(jj + TILE, N); ++j) {
                        float sum = 0;
                        for (int k = kk; k < std::min(kk + TILE, K); ++k) {
                            sum += A[i][k] * B[k][j];
                        }
                        C[i][j] += sum;
                    }
                }
            }
        }
    }
}
```

**Why it is fast:**

- Each tile operation uses TILE² elements from A, TILE² from B, TILE² from C: all fit in L3 cache (8 MB).
- Once a tile is loaded, the computation is dense (TILE³ operations on TILE² cache lines).
- Arithmetic intensity: TILE³ operations / (3 × TILE² cache lines) = TILE/3 operations per cache line.
- For TILE=64: 64/3 ≈ 21 operations per cache line. This is compute-bound, not memory-bound.

**Example numbers (1000×1000 matrices, tiled with TILE=64):**

- Time: ~2 seconds (25× faster)
- Cache misses: ~10 million (50× fewer)
- Memory bandwidth usage: ~20% (mostly compute)

**Measurement comparison:**

```bash
# Naive
perf stat ./matmul_naive
    L1-dcache-load-misses: 100,000,000
    LLC-load-misses: 500,000,000
    cycles: 150,000,000,000

# Tiled
perf stat ./matmul_tiled
    L1-dcache-load-misses: 10,000,000
    LLC-load-misses: 10,000,000
    cycles: 5,000,000,000
```

**Speedup:** 25–50× depending on matrix size and CPU.

---

## Software Prefetching

Modern CPUs have hardware prefetchers, but they cannot predict all access patterns. For known-pattern access (e.g., a stride known at compile time), software prefetching can help.

### Using `__builtin_prefetch`

```cpp
void prefetch_example(int* data, int size) {
    for (int i = 0; i < size; ++i) {
        // Prefetch 4 iterations ahead
        __builtin_prefetch(&data[i + 16], 0, 3);
        // Process current element
        process(data[i]);
    }
}
```

**Arguments to `__builtin_prefetch(address, rw, locality)`:**

1. **address:** The address to prefetch.
2. **rw:** 0 = read prefetch (default), 1 = write prefetch.
3. **locality:** 0 = no temporal locality (use once), 1 = low, 2 = medium, 3 = high (reuse soon).

### When Prefetching Helps

1. **Strided access beyond hardware prefetcher scope:**

```cpp
std::vector<int> data(10'000'000);
for (int i = 0; i < data.size(); i += 512) {  // 2 KB stride, beyond prefetcher
    __builtin_prefetch(&data[i + 512 * 4]);  // Prefetch 4 strides ahead
    total += data[i];
}
```

**Expected speedup:** 1.5–3× (reduces stalls waiting for prefetch).

2. **Linked-list traversal:**

```cpp
struct Node { int value; Node* next; };
for (Node* n = head; n; n = n->next) {
    __builtin_prefetch(n->next->next);  // Prefetch the next node's next
    process(n->value);
}
```

**Expected speedup:** 1.5–2× (hides some memory latency).

### When Prefetching Hurts

1. **Excessive prefetches (cache pollution):**

```cpp
for (int i = 0; i < size; ++i) {
    __builtin_prefetch(&data[i + 1]);  // Too aggressive
    __builtin_prefetch(&data[i + 2]);
    __builtin_prefetch(&data[i + 3]);
    process(data[i]);
}
```

**Problem:** Each prefetch uses a memory bus slot. If you prefetch too aggressively, you waste bandwidth. Hardware prefetchers already handle this case.

**Result:** No speedup, or slowdown (5–10%).

2. **Prefetching data you never use:**

```cpp
for (int i = 0; i < size; ++i) {
    if (i % 1000 == 0) {
        __builtin_prefetch(&data[i + 10'000]);  // Only used 1 in 1000 times
    }
    process(data[i]);
}
```

**Problem:** You are prefetching data that might be evicted before use.

**Result:** Cache pollution, slowdown.

### Measuring Prefetch Effectiveness

```bash
# Without prefetch
perf stat ./loop_no_prefetch
    LLC-load-misses: 5,000,000
    cycles: 100,000,000

# With prefetch
perf stat ./loop_with_prefetch
    LLC-load-misses: 2,000,000
    cycles: 50,000,000

# Speedup: 2× (you halved LLC misses and cycles)
```

**Rule of thumb:** Prefetch only if you measure a real speedup. Software prefetching is easy to get wrong.

---

## TLB Misses (Page Table Misses)

We have focused on CPU caches, but there is another level: the **Translation Lookaside Buffer (TLB)**.

The CPU must translate virtual addresses (what your code sees) to physical addresses (what the hardware uses). A TLB is a small cache of these translations. If the TLB misses, the CPU must walk the page table, which requires multiple main-memory accesses.

### TLB Hierarchy

```
Component               Size         Latency
L1 TLB (instruction)    128 entries  1 cycle
L1 TLB (data)           64 entries   1 cycle
L2 TLB (unified)        512 entries  10 cycles
Page table walk         N/A          50+ cycles (memory accesses)
```

**A TLB miss is expensive**: it requires walking a page table (usually 4–5 memory accesses) to translate a single address.

### When TLB Becomes a Bottleneck

- **Large working set:** If your code accesses > 64 distinct pages, you will exceed the L1 TLB. TLB misses increase.
- **Random access pattern:** Each access is to a different page (worst case for TLB).
- **Many threads:** Each thread has its own TLB. On context switches, the TLB is flushed, forcing reloads.

### Detecting TLB Misses

```bash
perf stat -e dTLB-load-misses,dTLB-loads ./program
```

**Output (example):**

```
    10,000,000 dTLB-loads
        50,000 dTLB-load-misses     # 0.5% TLB miss rate (low, OK)
```

```
    10,000,000 dTLB-loads
     1,000,000 dTLB-load-misses     # 10% TLB miss rate (high, problem)
```

### Solution: Hugepages

A standard page is 4 KB. A huge page is 2 MB (or 1 GB on some systems). Using hugepages reduces the number of page table entries needed.

**Example:** A 1 GB working set needs:
- **4 KB pages:** 1 GB / 4 KB = 262,144 pages.
- **2 MB pages:** 1 GB / 2 MB = 512 pages.

Hugepages fit better in the TLB, reducing misses.

**Linux setup:**

```bash
# Reserve hugepages
echo 1000 > /proc/sys/vm/nr_hugepages

# Use in your program
void* huge_data = mmap(nullptr, 1'000'000, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
```

**Or transparently with madvise:**

```cpp
std::vector<int> data(10'000'000);
madvise(data.data(), data.size() * sizeof(int), MADV_HUGEPAGE);
```

**Speedup:** 1.5–3× if TLB was a bottleneck. If TLB was not a bottleneck, no change.

---

## Worked Example: Matrix Multiply Progression

Let's trace a realistic optimization from naive to highly optimized.

### Setup

Multiply two 1000×1000 float matrices.

```cpp
#include <chrono>
#include <cstdio>
#include <cmath>
#include <cstring>

constexpr int M = 1000, K = 1000, N = 1000;
float A[M][K], B[K][N], C[M][N];

void initialize() {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < K; ++j) {
            A[i][j] = (i + j) / 1000.0f;
        }
    }
    for (int i = 0; i < K; ++i) {
        for (int j = 0; j < N; ++j) {
            B[i][j] = (i - j) / 1000.0f;
        }
    }
    std::memset(C, 0, sizeof(C));
}

auto measure(const char* name, void(*func)()) {
    auto t0 = std::chrono::high_resolution_clock::now();
    func();
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
    std::printf("%s: %lld ms\n", name, ms);
    return ms;
}
```

### Version 1: Naive (Cache-Hostile)

```cpp
void matmul_naive() {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            float sum = 0;
            for (int k = 0; k < K; ++k) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
}
```

**Run:**

```bash
g++ -O2 -g matmul.cpp -o matmul
perf stat ./matmul
```

**Output:**

```
Naive: 45000 ms (45 seconds!)

Performance counter stats:
       L1-dcache-load-misses: 100,000,000
       LLC-load-misses: 500,000,000
```

**Analysis:** Extremely slow. Every access to B[k][j] is a cache miss (column-major, strided). Almost all memory accesses miss to main memory.

### Version 2: Transpose B First (Improve Locality)

```cpp
float B_T[N][K];  // B transposed

void transpose_b() {
    for (int i = 0; i < K; ++i) {
        for (int j = 0; j < N; ++j) {
            B_T[j][i] = B[i][j];
        }
    }
}

void matmul_transposed() {
    transpose_b();
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            float sum = 0;
            for (int k = 0; k < K; ++k) {
                sum += A[i][k] * B_T[j][k];  // Now row-major in both!
            }
            C[i][j] = sum;
        }
    }
}
```

**Output:**

```
Transposed: 8000 ms (8 seconds)

Performance counter stats:
       L1-dcache-load-misses: 50,000,000
       LLC-load-misses: 80,000,000
```

**Analysis:** 5.6× speedup. The transpose overhead is negligible; the benefit is huge (both A and B are row-major now).

### Version 3: Tiled (Cache-Blocking)

```cpp
constexpr int TILE = 64;

void matmul_tiled() {
    for (int ii = 0; ii < M; ii += TILE) {
        for (int kk = 0; kk < K; kk += TILE) {
            for (int jj = 0; jj < N; jj += TILE) {
                for (int i = ii; i < std::min(ii + TILE, M); ++i) {
                    for (int k = kk; k < std::min(kk + TILE, K); ++k) {
                        float a = A[i][k];
                        for (int j = jj; j < std::min(jj + TILE, N); ++j) {
                            C[i][j] += a * B[k][j];
                        }
                    }
                }
            }
        }
    }
}
```

**Output:**

```
Tiled: 1200 ms (1.2 seconds)

Performance counter stats:
       L1-dcache-load-misses: 10,000,000
       LLC-load-misses: 5,000,000
```

**Analysis:** 37.5× speedup over naive. Miss rates dropped dramatically. Each tile fits in L3 cache, so most operations are cache hits.

### Comparison Table

| Version | Time (ms) | L1 Misses | LLC Misses | Speedup |
|---------|-----------|-----------|------------|---------|
| Naive | 45000 | 100M | 500M | 1× |
| Transposed | 8000 | 50M | 80M | 5.6× |
| Tiled | 1200 | 10M | 5M | 37.5× |

**Lessons:**

1. **Locality matters enormously.** The naive version is memory-bound; the tiled version is compute-bound.
2. **Simple fixes help.** Transpose is a 5× win with one pass over the data.
3. **Algorithmic reordering beats micro-optimizations.** Tiling beats any instruction-level optimization.

---

## Tradeoffs

| Technique | Pros | Cons | When to use |
|-----------|------|------|------------|
| **AoS → SoA** | 5–20× speedup if accessing one field; simple | Complex layout; poor if accessing whole object | Accessing one attribute across many objects |
| **Hot/cold splitting** | 2–5× speedup; simple refactor | Requires knowing access patterns | Known hot fields accessed repeatedly |
| **Loop tiling** | 5–50× speedup for nested loops; enables compiler optimizations | Code becomes more complex; needs tuning per cache size | Large nested loops; compute-heavy kernels |
| **Software prefetch** | 1.5–3× for strided access; explicit control | Easy to get wrong; bandwidth waste possible | Known-pattern access beyond HW prefetcher |
| **Hugepages** | 1.5–3× if TLB is bottleneck; transparent | Requires OS support; memory overhead | Large working sets; random access |
| **Transpose data** | 2–10× speedup; simple | Extra memory, one-time cost | Column-major access to large matrices |
| **Reorder fields** | 1.5–2× for hot loop; easy | Requires understanding access patterns | Struct with mixed hot/cold fields |

---

## Common Misconceptions

**Misconception 1: "Bigger arrays are always slower."**

Reality: A 1 GB sequential array can be faster than a 1 MB random-access array. Sequential access exploits the prefetcher; random access does not. Throughput for sequential 1 GB can be 40+ GB/s. For random-access 1 MB, it can be 1 GB/s or less.

**Misconception 2: "Cache optimization is micro-optimization."**

Reality: Cache behavior can be the difference between a 1-second and 100-second runtime. It is not a 10% improvement; it is a 100× improvement. This is macro-optimization.

**Misconception 3: "Software prefetching always helps."**

Reality: Prefetching is easy to get wrong. The hardware prefetcher is usually smarter than you are. Benchmark before and after. Most software prefetch attempts hurt performance.

**Misconception 4: "I should optimize for L1 cache."**

Reality: Optimizing for L1 (smallest, fastest) often creates cache line conflicts or false sharing. Optimize for the cache level that is the bottleneck. Measure to know which one it is.

**Misconception 5: "TLB misses do not matter; they are rare."**

Reality: For large working sets (> 256 MB) or random access patterns, TLB misses can be 10% of runtime. They are not rare in those workloads.

**Misconception 6: "The compiler will vectorize my loop for me."**

Reality: The compiler can auto-vectorize simple loops, but it often cannot. If you need vectorization (SIMD), you may need to rewrite the loop or use `#pragma omp simd`. Cache optimization often enables vectorization (e.g., tiling makes loops simple enough for the compiler to vectorize).

---

## Exercises

**Exercise 1: Measure cache miss types**

Run the matrix multiply naive version. Use `perf stat` to measure L1 and L3 miss rates. Predict the bottleneck (capacity, compulsory, or conflict). Then implement the transposed version and measure again. How much did each miss type change?

**Exercise 2: Tiling parameter tuning**

Implement the tiled matrix multiply. Try different tile sizes (32, 64, 128, 256). Measure time and cache misses for each. Which tile size is best? Can you predict the best size from your CPU's cache hierarchy?

**Exercise 3: Struct layout optimization**

Write a struct with 10 fields (mix of hot and cold). Write a loop that accesses only the hot fields. Measure cache misses. Then reorder the struct to put hot fields first. Measure again. Did it improve? By how much?

**Exercise 4: Software prefetching experiment**

Write a loop with strided access (stride > cache line). Benchmark with and without `__builtin_prefetch`. Try different prefetch distances (4, 8, 16 strides ahead). Which is best? At what distance does prefetching start to hurt?

**Exercise 5: TLB behavior**

Write a program that allocates a large array and accesses it randomly. Measure dTLB-load-misses. Then run it again after enabling hugepages. Does performance improve? Can you quantify the TLB miss cost?

**Exercise 6: Hot/cold analysis**

Find a real C++ data structure you use frequently (e.g., a user profile struct). Measure which fields are accessed in your hot loop. Rewrite the code to split hot and cold. Measure the speedup.

---

## Summary

Cache misses come in three types — compulsory, capacity, conflict — and each requires different remediation. Measure with `perf stat` to identify which type is limiting you. Apply data layout transformations (AoS → SoA, hot/cold splitting) for cheap wins. Implement loop tiling to keep working sets in cache (5–50× speedup). Use software prefetching sparingly and only when measured. For large working sets, use hugepages to reduce TLB misses.

The key insight: **cache behavior is not a detail; it is the primary determinant of runtime** for computation-heavy code. Optimization without understanding miss patterns is wasted effort. Measure, identify the bottleneck, and apply the right technique. The progression from naive (45 seconds) to tiled (1.2 seconds) on a simple matrix multiply shows the stakes: a 37× difference comes from understanding and eliminating cache misses, not from clever algorithms or micro-optimizations.

---

> **[← Previous: SIMD](11-simd.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: False Sharing →](13-false-sharing.md)**
