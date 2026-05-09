# Chapter 13 — False Sharing

Two threads, two unrelated variables, one cache line. Thread A writes its variable, the CPU core flushes the entire line from all other caches. Thread B's core loads the line, writes its variable, flushes again. The hardware is doing nothing but shuttling the 64-byte cache line back and forth between cores. No data is shared; the threads never coordinate. But the *hardware* does not know that. The CPU's cache-coherence protocol sees writes to adjacent memory and assumes they are contention. This is **false sharing** — a cache line that bounces between cores because of accident, not intention. The result: code that should scale linearly across cores instead serializes and becomes slower than single-threaded code.

False sharing is not rare. It is common in per-thread counters, concurrent data structures, and anything that packs multiple fields into adjacent cache lines. And when it occurs, the performance cliff is catastrophic — a 10–100× slowdown with no change to the algorithm, no increase in contention, just the hardware being confused.

This chapter teaches you to recognize false sharing, measure it, eliminate it, and reason about when padding is worth the memory cost.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the cache-coherence protocol (MESI or MOESI) and why writes to different addresses on the same cache line cause invalidations and forced reloads.
2. Distinguish false sharing (contention on a cache line despite no data contention) from true sharing (threads coordinating access to the same variable).
3. Predict which code patterns trigger false sharing: per-thread counters in an array, adjacent reference counts, packed structs in concurrent contexts.
4. Use `perf c2c` (Linux) to visualize cache-line contention and identify the specific lines causing the problem.
5. Apply `alignas(64)` and `std::hardware_destructive_interference_size` to pad structures and avoid false sharing.
6. Understand the tradeoff between memory waste (padding) and performance (avoiding cache bounces).
7. Recognize that false sharing is different from true contention and requires different fixes.

---

## How False Sharing Happens

A cache line is 64 bytes on nearly all modern CPUs (Apple Silicon, Intel, AMD, 2025). When you access a single byte, the CPU fetches 64 bytes into cache. When thread A writes to one address on that line, and thread B writes to a different address on the same line, the cache-coherence protocol (MESI on x86, MOESI on some other systems) ensures that all copies of the line are consistent.

This is where the problem begins.

### The MESI Protocol

On x86-64 and most modern systems, cache coherence uses the **MESI protocol** (Modified, Exclusive, Shared, Invalid):

- **Modified (M)**: This core owns the line and has written to it. No other core has a copy.
- **Exclusive (E)**: This core has the line, but other cores might also (unmodified copies). No core has written to it.
- **Shared (S)**: Other cores might have this line in shared state. No core has written to it (or all cores have identical copies).
- **Invalid (I)**: This core does not have the line.

When thread A on core 1 writes to address X on the cache line, the line transitions to **Modified** on core 1. All other cores' copies are immediately marked **Invalid**. If thread B on core 2 subsequently reads from address Y on the same line, core 2 must fetch the entire line again. Meanwhile, core 1 might have finished and core 2 acquires the line. If thread A writes again, core 1 must fetch the line back.

The result: the line bounces between cores. The hardware is performing the cache-coherence protocol correctly, but the traffic is wasteful because the threads are writing to *different* data.

### A Concrete Example

Suppose two threads each maintain a counter in an array:

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

int main() {
    std::vector<std::atomic<long long>> counters(8, 0);
    
    auto thread_func = [&](int idx) {
        for (int i = 0; i < 10'000'000; i++) {
            counters[idx].fetch_add(1, std::memory_order_relaxed);
        }
    };
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t0(thread_func, 0);
    std::thread t1(thread_func, 1);
    
    t0.join();
    t1.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "2 threads, 10M increments each: " << elapsed.count() << " ms\n";
    return 0;
}
```

On a 2-core machine, this might take **3000–4000 ms**. The two `std::atomic<long long>` values (8 bytes each) fit comfortably on the same 64-byte cache line. Every time thread 0 increments `counters[0]`, the line is marked Modified on core 0. Thread 1 on core 1 then has a cache miss and must fetch the line. The line shuttles back and forth, and the hardware is spending nearly all its time managing coherence, not computing.

If you pad the array so that `counters[0]` and `counters[1]` land on different cache lines:

```cpp
std::vector<std::atomic<long long>> counters(8 * 8, 0);  // 64 * 8 bytes per counter

auto thread_func = [&](int idx) {
    for (int i = 0; i < 10'000'000; i++) {
        counters[idx * 8].fetch_add(1, std::memory_order_relaxed);  // index by stride
    }
};
```

The same program on the same 2-core machine runs in **200–300 ms**. Each core has exclusive ownership of its cache line and never invalidates the other. No bouncing. The speedup is 10–15×, and the *only* change was memory layout.

This is false sharing: the illusion of contention where none exists.

### Diagram: Cache Line Bouncing

```
Timeline:

Core 0, Thread A:  [counters[0]++]
                        ↓ writes to line, transitions to Modified
                   Line in Core 0: [Modified]
                   Line in Core 1: [Invalid]
                   
Core 1, Thread B:  [counters[1]++]
                        ↓ cache miss on same line; fetch from Core 0
                   Line in Core 0: [Shared]
                   Line in Core 1: [Exclusive]
                        ↓ write to line, transitions to Modified
                   Line in Core 0: [Invalid]
                   Line in Core 1: [Modified]

Core 0, Thread A:  [counters[0]++]
                        ↓ cache miss on same line; fetch from Core 1
                   Line in Core 0: [Modified]
                   Line in Core 1: [Invalid]

... repeats millions of times...
```

The bus between the caches is saturated with ownership transfers, not data transfers. Each transfer stalls the writing thread for ~50 nanoseconds (the latency of a cache miss). With millions of writes, this adds up to seconds.

---

## Detecting False Sharing

The symptoms are distinctive: your multi-threaded code scales poorly despite low algorithmic contention. Adding more threads *slows* the program instead of speeding it up. The application spends time in cache-coherence states, not computing.

### Using `perf c2c`

On Linux, the `perf c2c` (cache-to-cache) tool shows which cache lines are contested between cores.

```bash
perf record -e LLC-loads,LLC-load-misses -c 125 -d ./counter_program
perf c2c report --stdio
```

This invokes `perf` in "cache-to-cache" mode, recording L3 (last-level cache) loads and misses. The report shows:

```
           Overhead  RMT  LLC  Load Hitm  Store Hits  Store Miss  ... Address
  27.34%            68%  44%  8.21%      0.00%       100.00%     0xsomeaddr
  ....
```

The key columns:
- **RMT (Remote)**: percentage of loads that came from another core (not local L3).
- **LLC (Last-Level Cache)**: percentage of loads that hit in L3 (local).
- **Load Hitm**: percentage of loads that hit a modified line in another core's cache (requires invalidation and writeback).

High **Load Hitm** or high **RMT** on a specific address indicates the line is being contested between cores. If the address is your counter array, you have false sharing.

### Manual Detection with `perf stat`

If `perf c2c` is unavailable, measure with `perf stat`:

```bash
perf stat -e cache-references,cache-misses,LLC-loads,LLC-load-misses ./counter_program
```

If you see:
- High `LLC-load-misses` relative to `LLC-loads` (e.g., 80% of L3 loads are misses).
- High `cache-misses` overall despite a small working set.
- Poor scaling with thread count.

...then false sharing is likely. However, `perf stat` does not pinpoint the problematic address; use `perf c2c` for that.

### Symptoms Without Profiling

You can often spot false sharing without tools:

1. **Terrible scaling**: 4 threads run slower than 1 thread on the same data set.
2. **Linear structure with adjacent writes**: a vector or array where each thread writes to a different element.
3. **Per-thread counters**: common in thread pools or work-stealing data structures.
4. **Adjacent atomic fields**: two `std::atomic<int>` fields in a struct, accessed by different threads.

---

## Fixing It: Padding

The cure is simple: ensure each thread's data lands on a separate cache line. On most systems, that means 64 bytes of spacing.

### Using `alignas(64)`

In C++17 and later, use the `alignas` keyword to align a structure or variable to a cache-line boundary:

```cpp
struct PaddedCounter {
    alignas(64) std::atomic<long long> value;
};

int main() {
    std::vector<PaddedCounter> counters(8);
    
    auto thread_func = [&](int idx) {
        for (int i = 0; i < 10'000'000; i++) {
            counters[idx].value.fetch_add(1, std::memory_order_relaxed);
        }
    };
    
    std::thread t0(thread_func, 0);
    std::thread t1(thread_func, 1);
    
    t0.join();
    t1.join();
    
    return 0;
}
```

With `alignas(64)`, each `PaddedCounter` (which contains an 8-byte atomic) is placed at a 64-byte-aligned address. The remaining 56 bytes are wasted, but now `counters[0]` and `counters[1]` are on different cache lines, eliminating false sharing.

The performance jump is dramatic: same code, just alignment, 10–20× faster.

### `std::hardware_destructive_interference_size`

C++17 introduced `std::hardware_destructive_interference_size`, a constant that represents the cache line size on the current hardware:

```cpp
#include <new>

static_assert(std::hardware_destructive_interference_size == 64);

struct PaddedCounter {
    alignas(std::hardware_destructive_interference_size) std::atomic<long long> value;
};
```

This is cleaner than hardcoding 64, because the code adapts to the hardware. However, **be cautious**: not all platforms define this correctly. Some libstdc++ versions report it as 0 or have stale values. Test on your target hardware.

For maximum portability, use a conservative constant:

```cpp
constexpr std::size_t CACHE_LINE_SIZE = 64;

struct PaddedCounter {
    alignas(CACHE_LINE_SIZE) std::atomic<long long> value;
};
```

### Padding in Structs

When false sharing occurs in a struct with multiple fields, separate the hot (frequently accessed) fields:

```cpp
// Bad: hot and cold on the same line, causes false sharing
struct State {
    std::atomic<int> counter;  // frequently written
    int padding1;
    std::atomic<int> flag;     // also frequently written by another thread
};

// Good: pad counter to its own line
struct State {
    alignas(64) std::atomic<int> counter;
    char pad[56];  // explicit padding to fill the line
    
    alignas(64) std::atomic<int> flag;
    char pad2[56];
};
```

Or use a helper:

```cpp
template<typename T>
struct Padded {
    alignas(64) T value;
    char pad[64 - sizeof(T)];
};

struct State {
    Padded<std::atomic<int>> counter;
    Padded<std::atomic<int>> flag;
};
```

### Per-Thread Arrays

In lock-free or per-thread-state code, each thread often has its own copy of a variable to avoid contention:

```cpp
// Per-thread state: each thread writes to its own slot
std::vector<Padded<std::atomic<long long>>> per_thread_count(num_threads);

// Thread i increments only per_thread_count[i]
for (int i = 0; i < 10'000'000; i++) {
    per_thread_count[my_thread_id].value.fetch_add(1);
}

// At the end, sum all threads' counts (single-threaded aggregation)
long long total = 0;
for (int i = 0; i < num_threads; i++) {
    total += per_thread_count[i].value.load();
}
```

Each thread has exclusive access to its slot (no coherence traffic during the hot loop). Aggregation happens once, at the end, and is cheap.

---

## Common Cases

### Per-Thread Counters in Monitoring Systems

Thread pools, task schedulers, and monitoring systems often track per-thread statistics:

```cpp
// Thread pool with per-thread counters
struct ThreadPool {
    std::vector<std::atomic<long long>> tasks_executed;  // one per thread
    std::vector<std::atomic<long long>> tasks_queued;    // one per thread
};
```

If `tasks_executed` and `tasks_queued` are adjacent in memory and frequently updated by different threads, false sharing will destroy performance. Use padding:

```cpp
struct PerThreadStats {
    alignas(64) std::atomic<long long> tasks_executed;
    alignas(64) std::atomic<long long> tasks_queued;
};

struct ThreadPool {
    std::vector<PerThreadStats> stats;
};
```

Now each thread's counters are on separate cache lines.

### Reference Counting in Shared Pointers

`std::shared_ptr` uses a control block with reference counts. If two `shared_ptr` objects to different objects share a control block (rare) or if multiple shared_ptrs are in the same data structure, the control block can false-share:

```cpp
// This is rare but possible in hand-rolled reference counting:
struct RefCountedNode {
    std::atomic<int> ref_count;  // shared_ptr increments/decrements this
    std::atomic<int> delete_flag; // custom field, also atomic
    Data data;
};

// Two threads: one increments ref_count, another sets delete_flag
// Both operations hit the same cache line, causing false sharing
```

Solution: ensure the reference count occupies its own cache line, or avoid adjacent atomic fields in hot data structures.

### Lock-Free Queues

In a bounded lock-free queue, the write and read pointers might be adjacent:

```cpp
// Bad: head and tail on the same line
template<typename T>
class LockFreeQueue {
private:
    std::vector<T> buffer;
    std::atomic<int> head;  // written by consumer
    std::atomic<int> tail;  // written by producer
};

// Good: pad them to separate lines
template<typename T>
class LockFreeQueue {
private:
    std::vector<T> buffer;
    alignas(64) std::atomic<int> head;
    char pad1[56];
    alignas(64) std::atomic<int> tail;
    char pad2[56];
};
```

The producer thread writes `tail` without interfering with the consumer's reads of `head`.

### Sharded/Partitioned Data Structures

Hash maps and other structures sometimes shard their data across multiple mutexes or buckets to reduce contention:

```cpp
template<typename K, typename V>
class ShardedMap {
private:
    static constexpr int SHARDS = 16;
    struct Shard {
        std::mutex mu;
        std::unordered_map<K, V> map;
    };
    std::vector<Shard> shards;
};
```

If all `Shard` structures are packed tightly in memory and different threads access different shards, the mutexes might false-share. Pad them:

```cpp
struct Shard {
    alignas(64) std::mutex mu;
    std::unordered_map<K, V> map;
};
```

---

## True Sharing vs False Sharing

It is crucial to distinguish the two:

### False Sharing

- **What it is**: Threads write to *different* addresses on the same cache line.
- **The problem**: Cache coherence forces invalidations despite no data contention.
- **The fix**: Padding to separate the addresses onto different cache lines.
- **The cost**: Wasted memory (up to 56 bytes per element on 64-byte lines).

### True Sharing

- **What it is**: Threads need to coordinate access to the *same* variable.
- **The problem**: The variable is genuinely contested; you need synchronization.
- **The fix**: A lock, an atomic, or a change to the algorithm (e.g., reduce the update frequency, use sharding to spread contention).
- **Example**: Multiple threads incrementing the same shared counter. They *cannot* avoid invalidations because they *must* update the same word.

**False sharing is an accident; true sharing is intentional.** False sharing wastes memory without solving contention. True sharing requires synchronization because the contention is real.

If you have true sharing, padding will not help. You must reduce the synchronization frequency or change the algorithm.

Example of true sharing:

```cpp
std::atomic<int> shared_count;  // all threads increment the same counter

// Three threads all doing this:
shared_count.fetch_add(1);  // all write to the same atomic

// Even if you pad shared_count, it won't help,
// because they're writing to the same variable.
// The contention is real.
```

Solution to true sharing: use per-thread counters and aggregate at the end (converting true sharing to false sharing via padding).

---

## Worked Example: Per-Thread Counters vs Shared Counter

Let's benchmark both approaches.

### Naive Shared Counter (True Sharing)

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

std::atomic<long long> shared_count(0);

void worker(long long iterations) {
    for (long long i = 0; i < iterations; i++) {
        shared_count.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    int num_threads = 8;
    long long iterations = 10'000'000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; i++) {
        threads.emplace_back(worker, iterations);
    }
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Shared counter (8 threads, 80M ops): " << elapsed.count() << " ms\n";
    std::cout << "Final value: " << shared_count << "\n";
    return 0;
}
```

Expected output on an 8-core machine:
```
Shared counter (8 threads, 80M ops): 2800 ms
Final value: 80000000
```

All threads contend on the same atomic. The atomic is correct and prevents data races, but the contention serializes increments.

### Per-Thread Counters (False Sharing Danger, Then Fixed)

**Unpadded (false sharing):**

```cpp
std::vector<std::atomic<long long>> per_thread(8, 0);

void worker(int id, long long iterations) {
    for (long long i = 0; i < iterations; i++) {
        per_thread[id].fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    int num_threads = 8;
    long long iterations = 10'000'000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; i++) {
        threads.emplace_back(worker, i, iterations);
    }
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    long long total = 0;
    for (auto& v : per_thread) total += v;
    
    std::cout << "Per-thread counter, unpadded (false sharing): " << elapsed.count() << " ms\n";
    std::cout << "Final total: " << total << "\n";
    return 0;
}
```

Expected output:
```
Per-thread counter, unpadded (false sharing): 2400 ms
Final total: 80000000
```

Still slow! The 8 atomics fit on a few cache lines, so even though each thread writes to its own atomic, they are still false-sharing with neighbors.

**Padded (fixed):**

```cpp
struct PaddedAtomic {
    alignas(64) std::atomic<long long> value;
};

std::vector<PaddedAtomic> per_thread(8);

void worker(int id, long long iterations) {
    for (long long i = 0; i < iterations; i++) {
        per_thread[id].value.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    int num_threads = 8;
    long long iterations = 10'000'000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; i++) {
        threads.emplace_back(worker, i, iterations);
    }
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    long long total = 0;
    for (auto& v : per_thread) total += v;
    
    std::cout << "Per-thread counter, padded: " << elapsed.count() << " ms\n";
    std::cout << "Final total: " << total << "\n";
    return 0;
}
```

Expected output:
```
Per-thread counter, padded: 120 ms
Final total: 80000000
```

**Speedup: 2400 / 120 = 20×** from the same logic, just with padding.

This illustrates the difference:

| Approach | Time | Contention Type | Fix |
|---|---|---|---|
| Shared counter | 2800 ms | True sharing | Algorithm change (per-thread or reduce frequency) |
| Per-thread (unpadded) | 2400 ms | False sharing | Padding |
| Per-thread (padded) | 120 ms | None | N/A |

---

## Worked Example 2: Atomics in Structs

Suppose you have a concurrent data structure with multiple atomic fields:

```cpp
struct ConcurrentStats {
    std::atomic<int> reads;
    std::atomic<int> writes;
    std::atomic<int> errors;
};

std::vector<ConcurrentStats> nodes(1000);

// Different threads update different fields of the same node
void thread_read_worker(int node_id) {
    for (int i = 0; i < 1'000'000; i++) {
        nodes[node_id].reads.fetch_add(1, std::memory_order_relaxed);
    }
}

void thread_write_worker(int node_id) {
    for (int i = 0; i < 1'000'000; i++) {
        nodes[node_id].writes.fetch_add(1, std::memory_order_relaxed);
    }
}
```

If one thread increments `reads` and another increments `writes` on the same node, they false-share (both fields fit on one cache line). **The fix:**

```cpp
struct ConcurrentStats {
    alignas(64) std::atomic<int> reads;
    alignas(64) std::atomic<int> writes;
    alignas(64) std::atomic<int> errors;
};
```

Now each field is on its own cache line (wasteful, but necessary if they're accessed concurrently). Alternatively, if reads and writes are always accessed together, keep them packed and only separate fields that are truly contended.

---

## Pitfalls with Padding

### Memory Waste

Each padded element wastes up to 56 bytes (on 64-byte cache lines). For a million-element vector, that is 56 MB of wasted memory — significant on memory-constrained systems or in data centers.

Trade-off: speed vs memory. In performance-critical sections (thread pools, high-frequency trading), the tradeoff favors speed. In large data structures (millions of elements), you might accept some false sharing to save memory.

### Hardware Variations

Cache line size is not universal:

- **Intel/AMD x86-64**: 64 bytes (standard).
- **ARM**: 64 bytes (most recent cores); some older cores use 32 bytes.
- **Apple Silicon (M1/M2/M3/M4)**: some reports suggest 128 bytes for certain cores; official specs are unclear.
- **IBM POWER9/10**: 128 bytes.

If you hardcode 64 and run on Apple Silicon M4 with 128-byte lines, you can still have false sharing.

**Best practice**: use `std::hardware_destructive_interference_size` if available, or a conservative constant that you verify on your target hardware. Test with `perf c2c`.

### Read-Mostly Structures

If the data structure is *read* by many threads but *written* by few (or none during the hot loop), padding is wasteful. Read-only access does not cause invalidations. Padding trades memory for a benefit that may not exist.

Example: a configuration struct read by all threads but updated rarely. Do not pad it.

```cpp
// Bad: wastes memory if reads are frequent but writes are rare
struct alignas(64) Config {
    int setting1;
    int setting2;
    int setting3;
};

// Better: only pad if you know concurrent writes occur
struct Config {
    int setting1;
    int setting2;
    int setting3;
};
```

### Alignment Misalignment

If you declare a struct with `alignas(64)` and then allocate it in a way that doesn't respect alignment (e.g., new-ing it without an allocator), the alignment is lost:

```cpp
struct alignas(64) Padded {
    std::atomic<int> x;
};

// Bad: new may not respect alignment
Padded* p = new Padded();  // may not be 64-byte aligned

// Good: use std::make_unique with an aligned allocator
std::unique_ptr<Padded, AlignedDeleter> p(new Padded());  // assuming AlignedDeleter

// Or use a vector (vectors do respect alignment of elements in C++20 and later)
std::vector<Padded> v(100);
```

Always verify alignment with a static_assert or runtime check:

```cpp
static_assert(alignof(Padded) == 64);
assert((uintptr_t)p % 64 == 0);
```

---

## Recognizing False Sharing in the Wild

### Symptom Checklist

1. **Poor scaling with threads**: 4 threads are slower than 1; 8 threads are even slower.
2. **High cache-miss ratio**: `perf stat` shows LLC misses are high despite a small working set.
3. **Small fields accessed concurrently**: counters, flags, pointers in a struct or array.
4. **Contention on unrelated data**: threads are not conceptually competing, but performance suggests they are.

### Example Code Patterns

**Per-thread array without padding:**
```cpp
std::vector<std::atomic<int>> counter(num_threads);  // DANGER: false sharing
```

**Adjacent atomic fields in a struct:**
```cpp
struct Data {
    std::atomic<int> flag1;
    std::atomic<int> flag2;  // likely false-shares with flag1
};
```

**Reference count in a dense array:**
```cpp
std::vector<Node> nodes;  // if Node contains an atomic ref_count
for (int i = 0; i < threads; i++) {
    threads.emplace_back([&]() {
        nodes[my_shard].ref_count++;  // false-shares with neighbors
    });
}
```

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Shared counter (true sharing)** | Simple, cache-efficient memory | Scales poorly; all threads contend on one word | Rarely; only if updates are rare and aggregate value is the bottleneck |
| **Per-thread counter (unpadded)** | Eliminates true contention; simpler than sharding | False sharing kills performance if cache lines are contested | Never; always pad if you use this pattern |
| **Per-thread counter (padded)** | Near-linear scaling; false sharing eliminated; memory coherence traffic minimal | Wastes memory (56 bytes per element on 64-byte lines) | Lock-free monitoring; high-frequency counters; work-stealing queues |
| **Sharded map with padded mutexes** | Reduces lock contention; false-sharing-aware | More complex; memory overhead per shard | Concurrent maps; hash tables with many threads |
| **Read-write lock with separation** | Multiple readers; avoids false sharing on reads | Complex memory layout; overhead on write path | Read-heavy concurrent access; configuration structs |
| **Segregated fields by access pattern** | Reduced false sharing on hot fields | Requires careful analysis of access patterns | Mixed read-write workloads; complex data structures |

---

## Common Misconceptions

1. **"False sharing is rare."** It is not. It occurs in nearly every concurrent system with per-thread state, lock-free data structures, or packed atomic fields. The reason it seems rare is that people often attribute the slowdown to contention or incorrect algorithms and don't investigate further.

2. **"Padding a counter fixes contention."** Padding fixes *false* sharing (accidental cache-line contention despite no data contention). If threads are writing to the *same* counter, padding does not help; they still contend, and you need a different approach (per-thread aggregation, reduce update frequency, etc.).

3. **"Cache line size is always 64 bytes."** Not on all hardware. Apple Silicon, IBM POWER, and some ARM systems use 128 bytes. Always verify on your target hardware or use `std::hardware_destructive_interference_size` with caution (test it).

4. **"I should pad everything to be safe."** No. Padding wastes memory and cache capacity. Only pad fields that are frequently written by different threads. Read-mostly data benefits from tight packing.

5. **"False sharing is a low-level detail I should not worry about."** It is a low-level detail that has high-level consequences. A single line of padding can turn a program from unusable to scalable. Profiling and understanding false sharing are part of system design for concurrent workloads.

6. **"Atomics with `memory_order_relaxed` avoid false sharing."** No. `memory_order_relaxed` specifies memory ordering, not cache coherence. The MESI protocol still manages the cache line, and false sharing still occurs. Relaxed ordering is a separate concern.

---

## Exercises

1. **Reproduce false sharing.** Write a program with N threads, each incrementing a counter in an array (8 elements, unpadded). Measure time for 10M increments per thread. Then pad with `alignas(64)` and measure again. Record the speedup. Does it match the theory? Try N = 2, 4, 8, 16.

2. **perf c2c investigation.** On Linux, use `perf c2c record -e LLC-loads,LLC-load-misses ./counter_program` to capture false sharing. Run `perf c2c report --stdio` and identify the contended addresses. Do they match your counter array? Do they disappear after padding?

3. **Shared vs per-thread tradeoff.** Implement three versions: (a) one shared counter, (b) per-thread counters unpadded, (c) per-thread counters padded. Benchmark all three with 1, 2, 4, 8 threads. At what thread count does padding break even against the shared counter?

4. **Memory overhead vs performance.** Create a vector of 1M `PaddedAtomic<long long>` elements. Measure memory usage. Compare to unpadded: how much memory is wasted? Is it worth the performance gain for your use case? (Hint: 1M * 56 bytes ≈ 56 MB.)

5. **Struct field ordering.** Design a struct with multiple atomic fields accessed by different threads. First, put them adjacent (one cache line). Benchmark. Then separate them with `alignas(64)`. Does performance improve? Can you achieve a middle ground with selective padding?

6. **Hardware cache line detection.** Write a program that determines your CPU's cache line size by micro-benchmarking: access an array with varying strides and measure cache-miss rate. At what stride does the miss rate jump? Does it match your CPU's specification?

7. **Conceptual.** A coworker proposes using a single shared `std::atomic<int>` counter for a monitoring system serving 100 cores. Before rejecting it, what measurements would you suggest to validate or refute the concern? Under what conditions might a shared counter be acceptable?

---

## Summary

False sharing is cache-line contention without data contention — a hardware illusion that can serialize multi-threaded code to single-threaded speed. It arises when multiple threads write to different addresses on the same 64-byte cache line; the MESI protocol bounces the line between cores, wasting memory bandwidth and CPU time. The solution is padding: place frequently written fields on separate cache lines using `alignas(64)` or `std::hardware_destructive_interference_size`. The cost is memory (up to 56 bytes per element wasted), but the benefit is dramatic — 5–20× speedups are common. False sharing is distinct from true sharing (contention on the same variable); true sharing requires algorithmic fixes (per-thread aggregation, reduced update frequency), while false sharing requires layout fixes. Profiling with `perf c2c` reveals contended cache lines. The biggest misconception is that false sharing is rare; it is common in per-thread-state code and lock-free structures, and the reason it goes undiagnosed is that people attribute slowdown to algorithmic issues rather than hardware coherence. Understanding false sharing is essential for writing concurrent code that actually scales.

---

> **[← Previous: Memory Alignment and Layout](12-cache-misses.md)** · **[↑ Part 8](README.md)** · **[Next: Part 9 — Building A Mental Model For Any Language](../part-09-mental-model-for-any-language/README.md)** *(coming soon)*
