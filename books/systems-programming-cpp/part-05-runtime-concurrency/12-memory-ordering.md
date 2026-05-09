# Chapter 57 — Memory Ordering

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain **why memory operations can be reordered** and distinguish between compiler reordering and CPU reordering, with concrete examples.
2. Distinguish between **sequential consistency** (`memory_order_seq_cst`) and weaker orderings (`acquire`, `release`, `relaxed`) and predict which reorderings each allows.
3. Use **release/acquire pairs** to build happens-before relationships between threads without full barriers.
4. Recognize **when relaxed atomics are correct** — counters and statistics where order does not matter — versus when they create race conditions.
5. Reason about **x86 vs ARM** memory models and explain why "fast" lock-free code on x86 may fail silently on ARM.
6. Implement a **single-producer/single-consumer (SPSC) ring buffer** with correct memory ordering and identify ordering bugs in naive implementations.
7. Spot the **double-checked locking antipattern** and understand why it fails without the right memory order, and how `std::call_once` avoids it.

This chapter is the bridge between "atomics exist" (Chapter 11) and "lock-free programming is possible" (Chapter 13). Memory ordering is the discipline that makes lock-free safe. It is also the part of concurrency that most strongly violates your intuition. Single-threaded code runs in source order. Multi-threaded code runs in source order *as observed by each thread*, but other threads may see operations in different orders — and the CPU and compiler are permitted by the C++ standard to let this happen. Memory ordering is how you constrain what's allowed.

---

## 57.1 Why Reordering Happens

Before we talk about fixing reordering, we need to understand why it exists. There are two sources.

### Compiler Reordering

The compiler is an optimizer. It sees two adjacent instructions:

```cpp
x = 1;
y = 2;
std::cout << "x=" << x << "\n";
```

The compiler observes: the assignment `x = 1` is never read. It can be eliminated. The output depends only on `y`, not `x`. So it might emit:

```cpp
y = 2;
std::cout << "x=" << x << "\n";  // x is uninitialized, but output is same
```

Or, considering instruction-level parallelism:

```cpp
a = huge_computation();  // takes 100 cycles
b = 2;
c = a + b;
```

The compiler may reorder to:

```cpp
b = 2;
a = huge_computation();
c = a + b;
```

So that `b = 2` executes while the CPU is waiting for `huge_computation()` to finish. The result is the same, and the code is faster.

But here's the problem: **in a multi-threaded program, the "result" is not just the value you see — it's the value every other thread sees.** When compiler reorderings happen on shared data without synchronization, they can break observable invariants.

Example. Thread A:

```cpp
data = 42;
ready = true;
```

Thread B:

```cpp
while (!ready) {}
std::cout << data << "\n";  // expect 42
```

If the compiler decides that `data` is not read in thread A's snippet (from its perspective as a single unit), it might reorder to:

```cpp
ready = true;
data = 42;
```

Now thread B wakes up (because `ready` is true) and reads `data` before thread A has written `42`. The program has a race condition.

The C++ standard permits this reordering **unless you use `std::atomic` or other synchronization.** When you write:

```cpp
std::atomic<bool> ready(false);
std::atomic<int> data(0);

// Thread A
data.store(42);
ready.store(true);

// Thread B
while (!ready.load()) {}
std::cout << data.load() << "\n";
```

The compiler understands that `data` and `ready` are atomic operations with potential ordering constraints. It will *not* reorder the stores in thread A past each other without an explicit memory order that allows it.

### CPU Reordering

Modern CPUs execute instructions out of order. From Chapter 2, recall the instruction pipeline: while one instruction is finishing, the next is decoding, and a third is being fetched. But even more aggressively, the CPU may execute instruction 5 *before* instruction 3 if instruction 5 doesn't depend on instruction 3's result.

The x86-64 CPU enforces one guarantee: **strong ordering for loads and stores to the same address**. If you write `x = 1`, then `y = x`, the CPU will not let you read the old value of `x`. But:

```cpp
x = 1;
y = 2;
```

The CPU may execute this as:

```
Store y=2 to memory
[many cycles of waiting for memory bus]
Store x=1 to memory
```

Because the stores are to different memory addresses, the CPU's store buffer can hold `y=2` in a queue while `x=1` is still pending. To thread B, which is polling for `x`, it may see `y=2` updated before `x=1`.

This is not allowed on x86 (x86 enforces total store order — stores to different addresses complete in order). But on ARM, this *is* allowed. ARM has a **weaker memory model**: stores to different locations may be reordered, loads may be reordered past stores, and so on.

The only thing that prevents reordering on any CPU is a **memory barrier** (or fence) — an instruction that says "nothing after this can start until everything before this is globally visible." On ARM, a barrier is expensive; it can cost 40-100 cycles. On x86, a barrier is cheaper but still costly. The C++ standard lets you specify which barriers you need, and which you don't, so you can optimize for your architecture.

### Store Buffers and Visibility Delays

Here's a concrete picture of why CPU reordering happens. Every CPU core has a **store buffer** — a small queue of pending writes. When the CPU executes `x = 1`, it doesn't write directly to memory; it adds the write to the store buffer and immediately returns. The store buffer flushes to the L3 cache (and then RAM) whenever the memory bus is free, which might be hundreds of cycles later.

Meanwhile, other cores are reading from their own caches. If core 1's store for `x` is still in the store buffer (not yet visible to core 2), core 2's load of `x` will see the old value.

A memory barrier forces the store buffer to flush: it says "don't execute anything after me until every write before me is globally visible." This is safe but slow.

---

## 57.2 Sequential Consistency: The Easiest Guarantee

If you use `memory_order_seq_cst` (the default for atomic operations), you get **sequential consistency**: every atomic operation appears to happen in some total order consistent with each thread's program order.

In other words:
- Your thread sees operations in source order.
- Every other thread sees operations in a single global order that respects all threads' source orders.
- The behavior is the same as if all operations were on a single thread, just interleaved somehow.

Example:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> x(0), y(0);
int r1, r2;

void thread1() {
    x.store(1);  // seq_cst by default
    r1 = y.load();
}

void thread2() {
    y.store(1);
    r2 = x.load();
}

int main() {
    std::thread t1(thread1);
    std::thread t2(thread2);
    t1.join();
    t2.join();
    
    // With seq_cst, at least one of r1 or r2 is 1.
    // (It is impossible for both to be 0.)
    std::cout << "r1=" << r1 << " r2=" << r2 << "\n";
    return 0;
}
```

With sequential consistency, you cannot have `r1 == 0 && r2 == 0`. Here's why:

- If `r1 == 0`, thread 1 read `y` *before* thread 2 wrote it. By the global ordering, thread 2's store to `y` must come before thread 1's load of `y`.
- But if thread 2's store to `y` comes first globally, then thread 1's store to `x` must come before it (by thread 1's order). So thread 2 sees `x == 1` when it loads, meaning `r2 == 1`.
- Therefore, `r1 == 0` implies `r2 == 1`.
- By symmetry, `r2 == 0` implies `r1 == 1`.
- So we cannot have both be 0.

**Cost:** Sequential consistency is implemented via full memory barriers. On x86, `store(seq_cst)` emits a `lock; mov` or similar (the lock prefix serializes memory traffic). On ARM, it emits a full `DMB ISH` barrier (approximately 40–100 cycles). This is the most expensive memory ordering, but the easiest to reason about.

---

## 57.3 Release/Acquire: Synchronization Without Barriers

Most programs don't need sequential consistency everywhere. The pattern that matters is: one thread *releases* (publishes) some data, and another thread *acquires* (sees) it.

```cpp
std::atomic<bool> ready(false);
std::atomic<int> data(0);

// Thread A
data.store(42, std::memory_order_relaxed);
ready.store(true, std::memory_order_release);

// Thread B
if (ready.load(std::memory_order_acquire)) {
    int x = data.load(std::memory_order_relaxed);
    std::cout << x << "\n";  // guaranteed to be 42
}
```

Here's what happens:

1. Thread A's `ready.store(..., release)` creates a **release barrier**: all stores *before* it (including the `data.store`) are guaranteed to be globally visible before the store to `ready` completes.
2. Thread B's `ready.load(..., acquire)` creates an **acquire barrier**: if the load returns true, all stores that happened before the store in thread A are now visible to thread B.

This is a **happens-before relationship**: the release synchronizes-with the acquire. Everything thread A did before the release is visible to thread B after the acquire.

**Cost:** On x86, a release/acquire pair is nearly free (one can be just a normal store, the other a normal load, because x86 has strong ordering anyway). On ARM, the acquire requires a `LDAR` (load-acquire) instruction and the release requires a `STLR` (store-release), which are cheaper than full barriers but still costly.

### Why Relaxed Is Not Enough

You might be tempted to write:

```cpp
// WRONG
std::atomic<bool> ready(false);
std::atomic<int> data(0);

// Thread A
data.store(42, std::memory_order_relaxed);
ready.store(true, std::memory_order_relaxed);  // WRONG: no ordering guarantee

// Thread B
while (!ready.load(std::memory_order_relaxed)) {}
int x = data.load(std::memory_order_relaxed);
std::cout << x << "\n";  // might be 0!
```

Because both stores are relaxed, the CPU is free to reorder them. Thread B might wake up and see `ready == true` before `data == 42`. The program is buggy.

---

## 57.4 Relaxed Atomics: When They Are Correct

Relaxed atomics (`memory_order_relaxed`) guarantee only **atomicity**: the operation is indivisible. A 64-bit `std::atomic<long>` store is never torn — other threads will see either the old value or the new value, never a mix.

Relaxed atomics are correct when there is **no ordering dependency**. Examples:

**Counters:**
```cpp
std::atomic<long> counter(0);

// Worker threads:
counter.fetch_add(1, std::memory_order_relaxed);  // Safe
```

The order in which increments happen does not matter. Each increment is atomic. The final count is correct regardless of CPU reordering.

**Statistics:**
```cpp
std::atomic<long> bytes_processed(0);
std::atomic<long> requests_handled(0);

// Multiple threads:
bytes_processed.fetch_add(len, std::memory_order_relaxed);
requests_handled.fetch_add(1, std::memory_order_relaxed);
```

Again, order does not matter. You only care about the final counts, not the history.

**Single-writer reads:**
```cpp
std::atomic<long> write_position(0);

// Thread A (writer):
write_position.store(new_pos, std::memory_order_relaxed);

// Thread B (reader, never modifies write_position):
long pos = write_position.load(std::memory_order_relaxed);
```

If only one thread writes, relaxed is safe. The reader is just observing; it doesn't need ordering guarantees.

**When NOT to use relaxed:** if there's a dependency — "I only care about `X` if `Y` is true" — you need ordering.

---

## 57.5 Memory Order Cheat Sheet

Here is the C++ memory order table:

| Order | Semantics | Load Barrier? | Store Barrier? | Use Case |
|-------|-----------|---------------|----------------|----------|
| `relaxed` | Atomicity only | No | No | Counters, independent updates |
| `acquire` (load) | Acquire barrier | Yes | No | Waiter on a lock/flag |
| `release` (store) | Release barrier | No | Yes | Notifier of a lock/flag |
| `acq_rel` | Both acquire and release | Yes | Yes | Read-modify-write that synchronizes |
| `seq_cst` | Full barrier (default) | Yes | Yes | When you want to not think; global ordering |
| `memory_order_consume` | Acquire, but weaker | Depends | No | (Rarely used; subtle) |

**Most practical rule:**
- Use `seq_cst` by default (it's the default anyway).
- For performance-critical lock-free code, use `acquire` on loads that wait for a flag, `release` on stores that set a flag, and `relaxed` for independent updates.
- Never mix relaxed with ordered operations on the same variable *unless* you are certain there's no dependency.

---

## 57.6 x86 vs ARM: Memory Models and Portability

This is where memory ordering becomes concrete and consequential.

### x86-64: Strong Memory Model

x86-64 enforces **total store order (TSO)**. In practice:
- A load cannot pass a prior store.
- Stores to different addresses complete in order (mostly).
- Only a load can "move past" a later load to a different address.

Result: `memory_order_seq_cst` on x86 is cheap. A `store(seq_cst)` is essentially a normal store (the CPU guarantees ordering). A `load(seq_cst)` is also essentially normal. There is no explicit fence instruction needed in most cases.

Code that works on x86 may appear fast because of this. Programmers sometimes write loose orderings, test on x86, and assume they work. Then the code breaks on ARM.

### ARM: Weak Memory Model

ARM allows:
- Stores to different addresses to be reordered.
- Loads to be reordered past stores.
- Speculative loads.

Result: `memory_order_seq_cst` on ARM requires explicit barriers. A `load(seq_cst)` is `LDAR` (load-acquire register). A `store(seq_cst)` is `STLR` (store-release register) or a full `DMB ISH` (data memory barrier, inner shareable domain).

A full barrier can cost 40-100 cycles — much more expensive than the equivalent on x86.

### The Portability Lesson

If you write lock-free code that works on x86, **always test on ARM or at least reason about ARM's model.** The code may have race conditions that x86's strong ordering hides.

Example of a bug that x86 hides:

```cpp
std::atomic<int> x(0), y(0);
// Thread A
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_relaxed);

// Thread B
int a = y.load(std::memory_order_relaxed);
int b = x.load(std::memory_order_relaxed);
```

On x86, if thread B reads `y == 1`, it's guaranteed to see `x == 1` (because x86 enforces store order).

On ARM, thread B could read `y == 1` and `x == 0` (stores were reordered). If thread B expects that "if `y` is set, `x` is set," the code is buggy on ARM.

The fix: use `release` on the store and `acquire` on the load:

```cpp
std::atomic<int> x(0), y(0);
// Thread A
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_release);

// Thread B
if (y.load(std::memory_order_acquire) == 1) {
    int b = x.load(std::memory_order_relaxed);  // guaranteed to be 1
}
```

Now the code is correct on all architectures.

---

## 57.7 Worked Example: Single-Producer/Single-Consumer Ring Buffer

Let's build a lock-free ring buffer with one producer and one consumer thread. This is a real pattern, used in high-performance logging, frame buffers, and audio processing.

### The Skeleton

```cpp
#include <atomic>
#include <array>
#include <cassert>

template <typename T, size_t Capacity>
class SPSCRingBuffer {
private:
    std::array<T, Capacity> buffer;
    std::atomic<size_t> write_pos{0};   // only written by producer
    std::atomic<size_t> read_pos{0};    // only written by consumer
    
public:
    bool try_push(const T& value) {
        size_t w = write_pos.load(std::memory_order_relaxed);
        size_t r = read_pos.load(std::memory_order_acquire);  // acquire to see consumer's progress
        size_t next_w = (w + 1) % Capacity;
        
        if (next_w == r) {
            // Buffer is full
            return false;
        }
        
        buffer[w] = value;
        write_pos.store(next_w, std::memory_order_release);  // release so consumer sees new data
        return true;
    }
    
    bool try_pop(T& value) {
        size_t r = read_pos.load(std::memory_order_relaxed);
        size_t w = write_pos.load(std::memory_order_acquire);  // acquire to see producer's progress
        
        if (r == w) {
            // Buffer is empty
            return false;
        }
        
        value = buffer[r];
        read_pos.store((r + 1) % Capacity, std::memory_order_release);  // release for producer
        return true;
    }
};
```

### Why This Ordering Works

**Producer's `try_push`:**
1. Read `write_pos` (relaxed: just my own state).
2. Read `read_pos` (acquire: I need to see consumer's latest progress, so I know if buffer is full).
3. Write to `buffer[w]`.
4. Update `write_pos` (release: consumer will acquire this, guaranteeing it sees the buffer write).

**Consumer's `try_pop`:**
1. Read `read_pos` (relaxed: just my own state).
2. Read `write_pos` (acquire: I need to see producer's latest progress, so I know if buffer has data).
3. Read from `buffer[r]`.
4. Update `read_pos` (release: producer will acquire this).

The acquire/release pairs create happens-before relationships:
- Producer's `write_pos.store(release)` synchronizes-with consumer's `write_pos.load(acquire)`.
- Consumer's `read_pos.store(release)` synchronizes-with producer's `read_pos.load(acquire)`.

This ensures that the buffer write is always visible before the index is updated, and vice versa.

### What Breaks With Relaxed

If you mistakenly use relaxed for everything:

```cpp
// WRONG: all relaxed
size_t w = write_pos.load(std::memory_order_relaxed);
size_t r = read_pos.load(std::memory_order_relaxed);  // might see stale value
buffer[w] = value;  // consumer might read garbage
write_pos.store(next_w, std::memory_order_relaxed);
```

The consumer might load `write_pos` before the producer's store to `buffer[w]` is visible. The consumer reads garbage.

### What Happens With seq_cst

Using `seq_cst` everywhere would work but be overkill:

```cpp
// Works, but expensive on ARM
size_t r = read_pos.load(std::memory_order_seq_cst);  // full barrier
```

Each `seq_cst` load/store might cost 40-100 cycles on ARM, for a total of several hundred cycles per operation. The acquire/release version costs nearly the same on x86 but a fraction on ARM, because the global ordering constraint is weaker.

---

## 57.8 Common Footguns

### Double-Checked Locking (The Classic Bug)

A classic pattern attempts to avoid locking on every access:

```cpp
// WRONG
std::mutex mut;
int* shared_ptr = nullptr;

int get_shared() {
    if (shared_ptr != nullptr) {  // check 1: no lock
        return *shared_ptr;
    }
    mut.lock();
    if (shared_ptr != nullptr) {  // check 2: under lock
        mut.unlock();
        return *shared_ptr;
    }
    shared_ptr = new int(42);
    mut.unlock();
    return *shared_ptr;
}
```

The bug: between check 1 and acquiring the lock, another thread might allocate and assign to `shared_ptr`. Thread A reads the uninitialized (or partially initialized) int. This is a classic data race.

The fix: make `shared_ptr` an atomic with acquire/release:

```cpp
// STILL WRONG: ordered wrong
std::atomic<int*> shared_ptr{nullptr};

int get_shared() {
    int* p = shared_ptr.load(std::memory_order_relaxed);  // WRONG: relaxed
    if (p != nullptr) {
        return *p;
    }
    // ... allocate and store ...
}
```

The relaxed load doesn't guarantee we see the initialization of the data that `p` points to. Another thread's store might be reordered.

The correct pattern uses a release/acquire pair or, better, `std::call_once`:

```cpp
// RIGHT: with acquire
int* p = shared_ptr.load(std::memory_order_acquire);
if (p != nullptr) {
    return *p;  // guaranteed to see all writes before store(release)
}
```

Or better, use `std::call_once`:

```cpp
std::once_flag flag;
int* shared_ptr = nullptr;

int get_shared() {
    std::call_once(flag, []() {
        shared_ptr = new int(42);
    });
    return *shared_ptr;
}
```

`std::call_once` handles the memory ordering internally and is both safer and clearer.

### Reading a Flag Without Acquire

```cpp
// WRONG
std::atomic<bool> ready{false};
int data = 0;  // NOT atomic

void thread1() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void thread2() {
    while (!ready.load(std::memory_order_relaxed)) {}  // WRONG: no acquire
    std::cout << data << "\n";  // might be 0
}
```

Thread 2's relaxed load doesn't synchronize with thread 1's release store. The CPU may not have flushed thread 1's store to `data` by the time thread 2 reads it. Use `acquire`:

```cpp
while (!ready.load(std::memory_order_acquire)) {}  // RIGHT
```

### Writing a Flag Without Release

```cpp
// WRONG
std::atomic<bool> ready{false};
int data = 0;

void thread1() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // WRONG: no release
}
```

Thread 2 might acquire and still see `data == 0` because thread 1's store to `data` wasn't flushed. Use `release`:

```cpp
ready.store(true, std::memory_order_release);  // RIGHT
```

---

## 57.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Atomic means thread-safe." | Atomic means indivisible. It does *not* imply ordering. `atomic<int>` with all relaxed loads/stores is a race condition if the values depend on each other. |
| "`memory_order_relaxed` is fine if it's in an atomic." | Relaxed atomicity is only safe if there is *no ordering dependency*. A relaxed load cannot be assumed to see a prior relaxed store on another variable. |
| "x86 is always correct, so I can test on x86 and ship everywhere." | x86's strong memory model hides bugs. Code may work on x86 and silently corrupt on ARM, as stores may reorder. Always reason about ARM's model or test on ARM. |
| "Memory barriers are free on modern CPUs." | Barriers (especially full seq_cst on ARM) cost 40-100 cycles, preventing out-of-order execution and flushing store buffers. In tight loops, this is significant. |
| "I need a lock for every shared variable." | Not always. A counter with relaxed atomics, or a flag with acquire/release, is faster and simpler than a lock. Locks have costs too: context switches, cache invalidation, priority inversion. |
| "If I use `std::atomic`, the compiler won't reorder anything." | The compiler respects the memory order you specify. Relaxed atomics can still be reordered (by the CPU). Only seq_cst prevents compiler reordering of that specific operation. |

---

## 57.10 Tradeoffs

| Ordering | Cost (ARM) | Cost (x86) | Use Case | Synchronization? |
|---|---|---|---|---|
| `relaxed` | 1–2 cycles | 1–2 cycles | Counters, independent updates | No (atomicity only) |
| `acquire` (load) | 5–10 cycles | 1–2 cycles | Waiting for a release | Yes (paired with release) |
| `release` (store) | 5–10 cycles | 1–2 cycles | Notifying about data | Yes (paired with acquire) |
| `acq_rel` | 15–20 cycles | 5–10 cycles | Compare-and-swap synchronization | Yes (full sync point) |
| `seq_cst` | 40–100 cycles | 5–10 cycles | Safe default, global ordering | Yes (full serialization) |

**Takeaway:** Choose the weakest ordering that is correct for your use case. On ARM, this is a significant optimization. On x86, it barely matters, but code written with weak ordering is portable.

---

## 57.11 Exercises

1. **Relaxed Counter Benchmark.** Write a program that increments a shared `std::atomic<long>` from 4 threads, 10 million times each. Time it three ways: (a) relaxed, (b) `acq_rel`, (c) `seq_cst`. Which is fastest? Why? (Hint: you might be surprised if you're on x86.)

2. **Ring Buffer from Scratch.** Implement a single-producer/single-consumer queue using two atomics (head and tail indices). Use the correct memory orderings. Write a test that spawns one producer thread and one consumer thread, pushes 1 million items, and verifies all are consumed in order. Then intentionally weaken one of the orderings to `relaxed` and see if the test fails.

3. **Double-Checked Locking Fix.** Take the buggy double-checked locking pattern and fix it three ways: (a) using atomic with acquire/release, (b) using a mutex, (c) using `std::call_once`. Compare the code clarity and performance if you benchmark 4 threads calling `get_shared()` 1 million times each.

4. **Detect Reordering.** Write a program (inspired by the test in §57.2) where both `r1` and `r2` should never be 0 if memory ordering is correct. Use `seq_cst` and verify they're never both 0. Then change to `relaxed` and run on an ARM device (or use a simulator). Do you see both as 0?

5. **Memory Model Comparison.** Read the C++ standard's section on memory ordering (usually section 32.4 or similar) or cppreference's "memory order" page. Write a one-paragraph summary of what "synchronizes-with" means, and give two examples: one where release/acquire creates a happens-before, and one where relaxed does not.

---

## 57.12 Summary

Memory ordering is the discipline that makes lock-free programming safe. It arises because compilers and CPUs reorder operations to improve performance. The C++ standard provides tools to constrain reordering:

- **Sequential consistency** (`seq_cst`) guarantees a global order; it is the default and the safest, but most expensive on weak-model CPUs like ARM.
- **Release/acquire pairs** create targeted synchronization: a release store synchronizes-with an acquire load, making everything before the release visible to code after the acquire.
- **Relaxed atomics** provide indivisibility without ordering; they are correct only for truly independent updates (counters) or in single-writer scenarios.
- **x86 hides bugs:** x86's strong memory model makes loose orderings appear correct, but they fail silently on ARM. Always reason about ARM's weak model or test on it.
- **Choose the weakest correct ordering.** The cost of acquire/release on ARM is much less than seq_cst; on x86, they cost nearly the same. Writing portable code with careful orderings is faster and more portable than defaulting to seq_cst everywhere.

The mental model to internalize: **each thread sees its own operations in order, but other threads may see a different order. Memory ordering is how you specify which orderings you require, and atomics are how you enforce them.**

---

> **[← Previous: Chapter 11 — Locks and Atomics](11-locks-and-atomics.md)** · **[↑ Part 5 — Runtime & Concurrency](README.md)** · **[Next: Chapter 13 — Lock-Free Data Structures →](13-lock-free-data-structures.md)** *(coming soon)*
