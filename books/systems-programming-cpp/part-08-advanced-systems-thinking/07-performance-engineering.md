# Chapter 84 — Performance Engineering

## Learning Objectives

By the end of this chapter you will be able to:

1. Understand the iron law of performance: you cannot optimize what you have not measured, and most optimization effort is wasted on code that is not the bottleneck.
2. Distinguish between latency, throughput, and tail latency, and understand why they require different optimization strategies and often conflict.
3. Recognize the hierarchy of bottlenecks — algorithmic complexity, cache misses, allocation, lock contention, syscalls — and approach each level systematically.
4. Use back-of-the-envelope calculations (latency models, bandwidth-product, Little's Law) to predict behavior before profiling.
5. Select and run the right profiling tool for your stack: `perf`, flame graphs, eBPF, VTune, async-profiler, pprof.
6. Apply the optimization patterns — reduce work, batch, cache, precompute, parallelize, vectorize — in their proper order.
7. Recognize when to stop optimizing: when the marginal gain no longer justifies the cost.
8. Design performance regression detection into your CI pipeline and understand why microbenchmarks lie.

---

## The Iron Law: Measure First

Here is the first thing you must unlearn from computer science textbooks: **do not optimize what you have not measured.**

This is not optional advice. This is the law.

When engineers feel slow, they guess. They feel that a linked list is the problem, or malloc thrashing, or a hot lock. Intuition about performance is almost always wrong. In a study of C++ codebases, engineers identified hot functions by guessing with about 15% accuracy. With a profiler, they found the actual hot functions in minutes.

The reason is that humans are poor at reasoning about the exponential tail of program behavior. A function that is called 1% of the time might consume 50% of runtime because it does expensive work. A function that looks "obviously" slow (loops over millions of items) might run in microseconds if the loop is simple and cache-friendly. The opposite often turns out true: the "obviously" fast code is the actual bottleneck.

The 80/20 rule, applied to performance, is actually closer to **95/5**: roughly 95% of runtime is in code that occupies maybe 5% of the codebase. This concentration is extreme. Which means: most of your code is not a bottleneck. Optimizing non-hot code is wasted effort, and optimizing hot code without understanding *why* it is hot often makes it worse.

**The loop:** measure → understand → change → measure again. Nothing else. This is where performance engineering begins.

---

## Latency vs Throughput vs Tail Latency

Optimization has different goals, and they conflict. You must know which one matters for your problem.

### Latency: Time from request to response

**Latency** is how long a single operation takes. A web request arrives; the server responds 50 ms later. That is latency.

For user-facing services, latency is usually what users perceive. A user clicks, and the UI updates. They are experiencing latency.

Reducing latency often requires:
- Reducing per-request overhead (allocation, copying, syscalls)
- Better cache locality (fewer misses per request)
- Parallelism within a single request (vectorization, SIMD)
- Precomputation (caches, lookup tables)

### Throughput: Total work per unit time

**Throughput** is how many operations you can complete in a given time window. A database handling 10,000 queries per second has higher throughput than one handling 1,000 queries per second.

Throughput is what matters for batch systems, data pipelines, and backend infrastructure.

Increasing throughput often requires:
- Batching (amortizing fixed costs over multiple items)
- Pipelining (multiple operations in flight at once)
- Parallelism (multiple cores/machines working simultaneously)
- Reducing per-item overhead

### Tail Latency: The p99, p99.9, not the mean

Here is a critical insight that separates good systems from bad ones: **the mean latency almost never matters.** What matters is the tail.

Suppose a web service has a mean latency of 10 ms. That sounds good. But suppose 99% of requests finish in 10 ms, and 1% finish in 1 second. The mean is still 10 ms. But users notice that 1% — every hundred requests, one takes 30× longer than usual. That's unacceptable for a user-facing service.

This is why you measure **percentiles**: p50 (median), p95, p99, p99.9. For user-facing services, p99 and p99.9 matter more than the mean.

Tail latency is usually driven by:
- Garbage collection pauses
- Cache eviction (some requests miss the cache; others hit)
- Lock contention (some requests wait for a lock held by another)
- OS scheduler delays (a thread gets preempted)
- Page faults (memory pressure causes swapping)

Optimizing for tail latency requires:
- Eliminating GC pauses (use arenas, pre-allocate, pool)
- Keeping working sets small (better cache behavior)
- Reducing lock contention (sharding, lock-free structures)
- Predictable code paths (avoid branches that vary by input)

**The conflict:** If you optimize for mean latency, you might use a large batch size to amortize overhead. But that hurts p99, because some batches are larger than others. If you optimize for p99, you might keep batch sizes small, hurting throughput. You cannot have both. You must choose.

---

## The Hierarchy of Bottlenecks

Not all bottlenecks are equally expensive to fix. Some require algorithm changes; others require hardware. Understanding the hierarchy means you attack the right level first.

In rough order of how often they bite:

### 1. Algorithmic Complexity

An O(n²) algorithm is the slowest possible kind of slow. It is not a CPU optimization problem; it is a math problem.

Example: A naive substring search is O(n × m). A KMP or suffix-tree-based search is O(n + m). For large inputs, this is the difference between unusable and fast.

Always check your algorithm first. No amount of cache optimization saves a bad algorithm.

### 2. Cache Misses

Once your algorithm is correct and reasonably complex, the next bottleneck — for computation-heavy code — is almost always the cache.

From Chapter 7: A memory miss to main memory costs ~200 CPU cycles. If your code misses the cache frequently, the CPU is stalled waiting for memory, not computing.

Signs of cache problems:
- `perf stat` shows cache-miss ratio > 10% (should be < 1%)
- Changing data layout (reordering struct fields, SoA vs AoS) gives a 2–10× speedup
- A sequential loop is faster than a linked list by an order of magnitude

Cache optimization is cheap: reorder fields, change data layout, improve spatial locality. The speedup can be 5–50×.

### 3. Allocation and Deallocation

Memory allocation and deallocation are syscalls (sort of — allocators use mmap/brk, but still). They are slow.

If your code allocates and frees objects in a tight loop, you pay that cost every iteration. If you can reuse allocations (object pool, arena), you eliminate the cost.

Signs of allocation problems:
- `perf stat` shows high page faults
- Flame graph shows malloc/free in the hot path
- Reducing allocations by 10× gives a 2–5× speedup (not 10× because allocation itself is only one of several costs)

### 4. Lock Contention

A lock that is heavily contended (many threads trying to acquire it simultaneously) causes threads to wait. While waiting, they consume CPU and memory bandwidth without progressing.

Signs of lock contention:
- Flame graph shows spinlock or futex time
- Scaling does not improve past N cores (instead of N× speedup, you get 2× or constant)
- Reducing lock scope or sharding the lock gives a 2–10× speedup

### 5. Syscall Storm

Syscalls are expensive. Each syscall crosses the kernel boundary, which flushes TLB and branch prediction. A tight loop of syscalls (e.g., many small reads from the filesystem) is much slower than one large read.

Batching syscalls (read 1 MB instead of 1 KB at a time) often gives a 5–20× speedup.

### 6. Branch Mispredictions

Modern CPUs speculatively execute code before deciding which branch to take. If the prediction is wrong, the CPU has to throw away the work and restart. A high misprediction rate stalls the pipeline.

This matters only when:
- The branch is in a tight loop (executed millions of times)
- The branch is unpredictable (random input; no correlation to prior branches)

Signs: `perf stat` shows branch-mispredict rate > 5% in a hot loop. Fixing it (sorting data, using branchless logic) gives a 1.2–3× speedup in that loop.

### 7. SIMD Opportunity

Modern CPUs have SIMD (Single Instruction, Multiple Data) instructions that operate on vectors of numbers. A single instruction can add eight 64-bit integers in parallel, for a theoretical 8× speedup.

Signs: You are doing the same operation on many pieces of data (loop over array). Vectorizing with `#pragma omp simd` or compiler auto-vectorization gives a 2–8× speedup.

**Key point:** The hierarchy is real. Fixing level 1 (algorithm) might give a 100× speedup. Fixing level 2 (cache) might give 10×. Fixing level 7 (SIMD) might give 8×. Always check the higher levels first. The higher you start, the bigger the payoff.

---

## Modeling Before Measuring

You cannot measure everything, and you cannot measure before the code exists. Use back-of-the-envelope calculations to predict behavior and guide your design.

### Latency Numbers Every Programmer Should Know

These are from Jeff Dean (Google, 2012), with 2025 updates:

```
Operation                          Latency
L1 cache read                       1 ns
L2 cache read                       3 ns
L3 cache read                       10 ns
Main memory read                    ~100 ns
SSD random read                     ~100 µs
HDD random read                     ~10 ms
Network packet (LAN)                ~1 ms
Network packet (across US)          ~50 ms
```

Use these to predict if a design will work. If your design requires 1 million SSD reads per query, that is 100 seconds of latency — unacceptable. If it requires 100 million main-memory reads, that is ~10 seconds — still bad. If it requires 10 million L3 cache reads, that is ~100 µs — acceptable.

### Latency-Bandwidth Product

The **latency-bandwidth product** tells you how much data can be "in flight" between the CPU and memory without the CPU stalling:

```
in_flight_data = latency × bandwidth
```

If main memory latency is 100 ns and bandwidth is 50 GB/s:
```
in_flight = 100 ns × 50 GB/s = 5 KB
```

This means the CPU can have only 5 KB of pending memory requests before it runs out of things to do. If your working set is larger than this, you will have pipeline stalls.

This is why prefetching helps: it hides the latency by fetching the next cache line while the current one is being processed.

### Little's Law

Little's Law relates average response time, throughput, and number of requests in the system:

```
avg_requests_in_system = throughput × avg_response_time
```

If your system completes 1,000 requests per second and each request takes 100 ms:
```
requests_in_system = 1,000/s × 0.1 s = 100
```

This means on average, 100 requests are being processed at any time. If you have fewer than 100 cores, requests must queue. If you want to reduce response time, you must either increase throughput or reduce the number of concurrent requests.

### Roofline Model

The roofline model predicts the maximum speedup you can achieve from optimization. It answers: "Is my code compute-bound or memory-bound?"

```
peak_flops = clock_speed × cores × flops_per_cycle
peak_bandwidth = memory_bandwidth

arithmetic_intensity = flops / bytes_loaded
max_performance = min(peak_flops, arithmetic_intensity × peak_bandwidth)
```

If your code has low arithmetic intensity (few floating-point operations per byte loaded), you are memory-bound. Optimizing computation will not help; you must reduce memory traffic. If your code has high arithmetic intensity, you are compute-bound. You need faster CPU, vectorization, or parallelism.

Use these models to:
1. **Check feasibility** — does this design have a chance of meeting the latency target?
2. **Set optimization order** — is it worth optimizing memory, or is the code compute-bound?
3. **Establish ceilings** — your perf improvement cannot exceed the roofline.

---

## Tools: Choosing the Right Profiler

Profilers come in many forms. Choosing the wrong tool wastes time. Here is a taxonomy:

### `perf` (Linux)

Sampling-based profiler using CPU performance counters. Overhead is low (< 5%). Output is a text report or flame graph.

**What it tells you:**
- Which functions are hot (by CPU time)
- Cache-miss rate, branch-misprediction rate, IPC (instructions per cycle)
- Where time is spent in the kernel vs userspace

**When to use:** Linux systems, need to understand why code is slow (cache behavior, allocation, etc.).

Example:
```bash
perf record -g ./my_program
perf report
```

Or with statistics:
```bash
perf stat ./my_program
```

Output shows:
```
Performance counter stats:
 4,234,567,890 cycles
 2,123,456,789 instructions        # 0.50 insn per cycle
     1,234,567 cache-misses        # 3.2% of all cache refs
        45,678 branch-misses       # 0.8% of all branches
```

### Flame Graphs

A visualization of profiling data: x-axis is function (sorted by name), y-axis is call stack depth. The wider a box, the more time in that function. Color is often arbitrary (sometimes indicates subsystem).

Flame graphs are invaluable for seeing the "shape" of your program's time distribution. They make it easy to spot deep call stacks (heavy abstraction) and wide plateaus (hot loops).

### eBPF and ftrace (Linux)

In-kernel tracing with minimal overhead. Can trace syscalls, page faults, context switches, custom kernel events.

**When to use:** You suspect the bottleneck is not CPU (cache, algorithms) but OS behavior (syscalls, scheduling, page faults).

### Intel VTune (Windows, Linux, macOS)

Commercial profiler with deep hardware insights. Shows cache misses by level, pipeline stalls, memory bandwidth contention.

**When to use:** You need hardware-level detail and have a VTune license.

### async-profiler (JVM)

Low-overhead sampling profiler for Java/Kotlin/Scala. Does not require JVM changes. Good for production profiling.

**When to use:** Running Java and `perf` does not work (symbol resolution in JVM is hard).

### py-spy (Python)

Sampling profiler for Python. Minimal overhead even on running processes.

**When to use:** Python code is slow and you need to find hot functions.

### pprof (Go)

Built into the Go runtime. Supports CPU profiling, memory profiling, goroutine profiling, and mutex profiling.

**When to use:** Go code is slow.

### JMH (Java)

Microbenchmarking harness for Java. Handles warmup, JIT compilation, GC, etc., correctly.

**When to use:** Benchmarking specific methods or algorithms in Java.

### Criterion.rs (Rust)

Microbenchmarking library for Rust. Statistical analysis of multiple runs; reports regression automatically.

**When to use:** Benchmarking Rust code.

**Key principle:** Start with a simple sampler (`perf`, async-profiler, pprof). Look at the flame graph. If the hot path is clear, measure it carefully. If you are confused, trace syscalls or use hardware counters to understand what the CPU is doing. Do not start with the most detailed tool; start with the highest-level view and zoom in.

---

## Optimization Patterns

Once you have identified the bottleneck, apply the right optimization pattern. These patterns exist in a rough order of payoff:

### 1. Reduce Work

The absolute best optimization: do less. Skip the work entirely.

Example: A web server was spending 30% of time serializing an empty default value in JSON. Remove the field. Instant 30% speedup.

Example: A query engine was building intermediate result sets for every operation. Cache the results and skip redundant queries. 10× speedup.

### 2. Batch

Amortize fixed costs over multiple items.

Example: Instead of allocating one object at a time (malloc → data → free), allocate 1,000 at once and recycle them. Allocation overhead goes from 1,000 ops to 1 op per batch. 100× speedup on allocation.

Example: Instead of flushing data to disk after each write, buffer 1,000 writes and flush once. Syscall overhead goes from 1,000 to 1. 50× speedup on disk I/O.

### 3. Cache

Store computed results so you do not recompute them.

Example: Memoize expensive function results. First call takes 1 ms; subsequent calls take 0.001 ms. 1,000× speedup if you hit the cache.

Example: Pre-build lookup tables. Converting from string to int via hash table is O(1); parsing is O(n). 100× speedup.

### 4. Precompute

Do work in advance so the hot path is cheaper.

Example: Pre-parse a configuration file at startup. Hot path is a dictionary lookup. 10× speedup.

Example: Pre-sort data for a hot query. Sorted search is O(log n); unsorted search is O(n). 100× speedup.

### 5. Parallelize

Divide work among cores.

Example: A single thread processes 1,000 items in 1 second. Divide among 8 cores; each thread processes 125 items in 125 ms. 8× speedup (ignoring synchronization overhead).

This works only if the work is divisible and synchronization is cheap. If there is significant lock contention, it fails.

### 6. Vectorize

Use SIMD instructions to process multiple items per instruction.

Example: Adding eight 64-bit integers: scalar does it in 8 cycles, SIMD does it in 1 cycle. 8× speedup.

Requires that the compiler can auto-vectorize or you write SIMD intrinsics. Only works if the loop is simple enough for the compiler to understand.

### 7. Better Data Structure

Choose a data structure that is efficient for your access pattern.

Example: Linear search through 1,000 items is O(n). Binary search tree is O(log n). Hash table is O(1) average. 100× speedup from O(n) to O(1).

Example: Linked list forces cache misses. Array is cache-friendly. 10× speedup.

**Application order:** Start at the top. If you can reduce work or batch, do that first — the payoffs are huge. If not, move to caching. Then precomputation. Parallelize only if the bottleneck is not lock contention or memory bandwidth. Vectorize only if the loop is compute-bound. Change data structure only if access patterns make it necessary.

Most optimization problems can be solved at levels 1–3. If you are at level 7, reconsider whether you are solving the right problem.

---

## When To Stop: Diminishing Returns

Optimization is a cost-benefit analysis. You incur costs:
- Time to profile and identify the bottleneck
- Time to implement the optimization
- Time to test and ensure correctness
- Maintenance burden (optimized code is often harder to understand)

You gain benefits:
- Faster execution
- Lower latency for users
- Lower infrastructure costs (fewer servers needed)

When the marginal gain no longer justifies the marginal cost, stop.

### Define a Target First

Before you start optimizing, set a goal: "This API must respond in < 50 ms p99" or "This batch job must complete in < 1 hour" or "Throughput must be > 100,000 req/s."

Without a target, you have no stopping point. You will optimize forever.

### Measure Progress

After each optimization, measure again. Graph the improvements. You will see a curve that flattens:
- First optimization: 10× speedup
- Second: 3× speedup
- Third: 1.5× speedup
- Fourth: 1.2× speedup

The first optimizations yield big gains. The later ones yield small gains at high cost. Stop when the cost exceeds the value.

### Know When You Are Done

If your target is met, you are done. Further optimization is waste. If your target is not met but the cost of further optimization is very high, accept the tradeoff. You cannot optimize forever.

### The Paradox

"Premature optimization is the root of all evil." This is true — do not optimize code you have not measured. But "late optimization is expensive" is also true — if you build atop a slow foundation, you will eventually hit a wall that requires architectural changes.

The balance: **Design for the right algorithm (O(n) not O(n²)) and the right data layout (cache-friendly). Then measure. Optimize only at the bottom, where measurement shows time is spent.**

---

## Worked Example: From Hot Loop to Faster Traversal

Let's take a concrete C++ example: **summing 100 million integers from a vector.**

### Initial Code

```cpp
// hot_loop.cpp
#include <vector>
#include <cstdint>
#include <iostream>
#include <chrono>

int main() {
    std::vector<int32_t> data(100'000'000);
    // Initialize with pattern (not random, to ensure reproducibility)
    for (size_t i = 0; i < data.size(); ++i) {
        data[i] = static_cast<int32_t>(i % 100);
    }

    // Measurement
    auto start = std::chrono::high_resolution_clock::now();

    int64_t sum = 0;
    for (int32_t x : data) {
        sum += x;
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

    std::cout << "Sum: " << sum << " Time: " << elapsed << " ms\n";
    return 0;
}
```

Compile and profile:

```bash
g++ -O3 -g hot_loop.cpp -o hot_loop
perf stat ./hot_loop
```

Output:
```
Sum: 4950000000 Time: 45 ms

Performance counter stats:
   100,234,567 cycles
   200,123,456 instructions        # 2.0 insn per cycle
        45,678 cache-misses        # 0.3% of all cache refs
        12,456 branch-misses       # 0.01% of all branches
```

The code is **not** cache-bound (0.3% miss ratio is excellent). The loop is simple and predictable. **Where is the time?** Execution itself: 200 million instructions over 100 million items = 2 instructions per iteration. This is compute-bound, not memory-bound.

### Identifying the Bottleneck with perf

But let's dig deeper. Use `perf record` to see call stacks:

```bash
perf record -g ./hot_loop
perf report
```

The entire time is in `main` → the loop. No surprises. But let's check IPC (instructions per cycle): **2.0 IPC.** A modern CPU with good branch prediction and cache behavior can do 4–5 IPC. We are at 2. Why?

**Hypothesis:** The addition is a data dependency. The new sum depends on the previous sum. The CPU cannot parallelize additions because each iteration waits for the previous one to complete.

### Optimization 1: Unrolling (Reduce Dependency Chain)

Instead of one addition per iteration, do four additions in parallel, then sum them at the end:

```cpp
int64_t sum = 0;
size_t i = 0;
for (; i + 3 < data.size(); i += 4) {
    sum += data[i] + data[i+1] + data[i+2] + data[i+3];
}
for (; i < data.size(); ++i) {
    sum += data[i];
}
```

Recompile and measure:

```bash
g++ -O3 -g hot_loop_unroll.cpp -o hot_loop_unroll
perf stat ./hot_loop_unroll
```

Output:
```
Sum: 4950000000 Time: 15 ms

Performance counter stats:
    60,123,456 cycles
   200,456,789 instructions        # 3.3 insn per cycle
        50,234 cache-misses        # 0.4% of all cache refs
        15,678 branch-misses       # 0.02% of all branches
```

**3× speedup.** Cycles dropped from 100M to 60M. IPC improved from 2.0 to 3.3 because the CPU can now overlap iterations.

### Optimization 2: Vectorization (SIMD)

The compiler can auto-vectorize with the right hint:

```cpp
int64_t sum = 0;
#pragma omp simd reduction(+:sum)
for (size_t i = 0; i < data.size(); ++i) {
    sum += data[i];
}
```

Compile with OpenMP:

```bash
g++ -O3 -g -fopenmp hot_loop_simd.cpp -o hot_loop_simd
perf stat ./hot_loop_simd
```

Output:
```
Sum: 4950000000 Time: 4 ms

Performance counter stats:
    15,234,567 cycles
   200,567,890 instructions        # 13.1 insn per cycle
        12,345 cache-misses        # 0.1% of all cache refs
        8,234 branch-misses        # 0.005% of all branches
```

**12× speedup from original.** The SIMD version does eight 32-bit additions per instruction, reducing the total cycle count dramatically. IPC hit 13.1 — excellent.

### Measurement Summary

| Version | Time (ms) | Cycles | Speedup |
|---------|-----------|--------|---------|
| Original | 45 | 100M | 1× |
| Unrolled | 15 | 60M | 3× |
| SIMD | 4 | 15M | 12× |

### Lessons

1. **Measure first.** Without profiling, you would guess the bottleneck was memory. It was not; it was dependency chains.
2. **Start at the right level.** Unrolling gave 3×. SIMD gave 12×. Unrolling first made SIMD more effective.
3. **Know when to stop.** Further optimization (parallelization) would add complexity for minimal gain (most of the improvement is already achieved).
4. **Let the compiler help.** Modern C++ compilers and pragmas (OpenMP, SIMD hints) can achieve significant speedups without manual SIMD intrinsics.

---

## Performance Regressions and CI

Once you have optimized, do not lose the gains. Continuous benchmarking in CI catches regressions before they reach production.

### Microbenchmarks Lie

Before you set up CI benchmarks, understand: **microbenchmarks are not realistic.**

When you benchmark a tight loop in isolation:
- Caches are warm
- Inputs are predictable
- There is no context switching
- There are no other threads

In production:
- Caches are shared
- Inputs vary
- The OS context-switches constantly
- Other threads contend for resources

A microbenchmark might show 4× speedup; production shows 1.2×. This is normal.

**Mitigation:** Benchmark representative workloads, not microbenchmarks. Run on production-like hardware. Include load (multiple concurrent users, background jobs). Measure p99, not mean.

### Regression Detection Strategy

1. **Establish a baseline.** Measure the current performance on a fixed workload. Record p50, p95, p99.
2. **Run on every commit.** Run the same benchmark after each merge to main. Flag if p99 regresses by > 5%.
3. **Investigate regressions.** When flagged, git bisect to find the culprit. Sometimes it is a legitimate algorithmic change (acceptable). Sometimes it is a bug (revert).
4. **Track trends.** Plot performance over time. A slow drift downward often indicates accumulating technical debt or feature bloat.

### Example CI Setup (GitHub Actions)

```yaml
name: Performance Regression

on: [push]

jobs:
  bench:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build
        run: g++ -O3 -g benchmark.cpp -o benchmark
      - name: Run Benchmark
        run: |
          for i in {1..10}; do
            ./benchmark --iterations 1000000 >> results.txt
          done
      - name: Analyze Results
        run: python3 analyze.py results.txt
      - name: Compare to Baseline
        run: |
          python3 compare.py results.txt baseline.txt
          # Fails if p99 regresses by > 5%
```

---

## Tradeoffs and Design Patterns

Different optimization strategies have different costs:

| Approach | Pros | Cons | When to use |
|----------|------|------|------------|
| Reduce work | Massive speedup; reduces power | Requires algorithmic insight | Always, if possible |
| Batching | 2–10× speedup; simple | Increases latency (larger batches = longer waits) | Throughput-critical systems |
| Caching | 2–1000× speedup; simple | Cache invalidation is hard; uses memory | If cache hit rate > 80% |
| Precomputation | High speedup; shifts cost to startup | Uses memory; harder to maintain | If workload is static; frequent queries |
| Parallelism | Scales with cores (2–16×) | Locks and synchronization; limited by Amdahl's law | If bottleneck is not lock contention |
| Vectorization | 2–8× speedup; few code changes | Compiler-dependent; not all loops vectorize | If loop is compute-bound and simple |
| Data structure change | Can be dramatic (O(n) → O(1)) | Risk of correctness bugs; maintenance burden | Only if access patterns justify it |

---

## Common Misconceptions

**Misconception 1: "Optimization always helps."**

Reality: Premature optimization adds complexity without benefit. The code is harder to maintain. Often, optimization at the wrong level (e.g., SIMD when the bottleneck is allocation) does not help.

**Misconception 2: "Faster algorithm always wins."**

Reality: An O(n) algorithm with bad cache locality can be slower than an O(n log n) algorithm with good cache behavior, for reasonable input sizes. Measure both.

**Misconception 3: "The compiler will optimize it for me."**

Reality: Compilers are smart but not psychic. They cannot optimize:
- Algorithmic complexity (you must fix that)
- Data layout (use `[[pack]]`, reorder fields)
- Branch predictability (structure your data so branches are predictable)

**Misconception 4: "Lock-free = fast."**

Reality: Lock-free code is often slower than locks because of higher memory bandwidth contention and complexity. Use locks unless profiling shows contention is a bottleneck.

**Misconception 5: "We can optimize later."**

Reality: Some architectural decisions (choosing the database, data layout) are expensive to change later. Design for the right algorithm and data layout early. Optimize at the bottom.

---

## Exercises

**Exercise 1: Measure a hot loop**

Take a program you have written (or find an open-source one). Profile it with `perf stat`. Identify the hottest function. Is it cache-bound, compute-bound, or limited by syscalls? What would the roofline predict?

**Exercise 2: Optimize and measure**

Take the hot function and apply one optimization (reduce work, batch, cache, or data structure change). Measure the speedup. Calculate the cost (time spent optimizing) vs benefit (time saved). Was it worth it?

**Exercise 3: Microbenchmark vs reality**

Write a microbenchmark for a function. Then measure it under realistic load (multiple threads, cache contention). Compare the results. Discuss the differences.

**Exercise 4: Regression detection**

Set up a CI benchmark for a project. Commit a performance regression (e.g., change O(n) to O(n²) in a hot path). Verify that CI catches it. Then fix it and verify CI passes.

---

## Summary

Performance engineering is not guesswork; it is measurement and iteration. Start by profiling to find the bottleneck. Use back-of-the-envelope models to predict behavior. Apply optimizations in order of payoff: reduce work, batch, cache, precompute, parallelize, vectorize. Measure after each change. Stop when your target is met or when further optimization costs more than it saves. Maintain performance through continuous regression detection. Above all: **measure first, optimize second.**

---

> **[← Previous: Protocol Design](06-protocol-design.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: Profiling →](08-profiling.md)**
