# Chapter 55 — Thread Safety

## Learning Objectives

By the end of this chapter you will be able to:

1. Define a **data race** precisely: two or more threads, at least one performing a write, with no synchronization. Know that this results in undefined behavior in C++.
2. Distinguish a **race condition** (timing-dependent outcome) from a **data race** (undefined behavior), and understand that synchronizing away the data race does not eliminate all race conditions.
3. Explain what **visibility** and **reordering** are — both at the compiler level and the CPU level — and why one thread's writes may arrive out of order to another thread.
4. Describe what `std::atomic<T>` provides (atomic operations plus synchronization) and what `volatile` does *not* provide (no thread-safety guarantee).
5. Evaluate the tradeoffs among mutexes, atomics, lock-free data structures, thread-local storage, and immutable objects for a given synchronization problem.
6. Read and reason about subtle C++ concurrency code, identifying where reordering could occur and predicting thread interleavings.

---

## 55.1 The Central Paradox of Threading

You want concurrency — multiple threads doing work simultaneously — because:

- **Hardware**: Modern CPUs have multiple cores. Your program, running on one core, is leaving the others idle.
- **Responsiveness**: A web server handling 10,000 concurrent connections needs to service each one, but blocking on I/O from one client should not block others.
- **Throughput**: A batch processor can achieve linear speedup (within limits) by splitting work across cores.

But concurrency introduces a new class of bugs that do not exist in sequential code. These bugs are:

- **Non-deterministic**: they depend on scheduling decisions by the OS, the CPU, and the compiler.
- **Non-reproducible**: they occur rarely, in production, under load, but vanish in the lab when you add instrumentation.
- **Non-local**: the bug in Thread A may be triggered by code in Thread B that looks completely unrelated.

This chapter is about understanding what those bugs are, why they happen, and what mechanisms prevent them.

---

## 55.2 The Three Hazards

### Data Race: Undefined Behavior

A **data race** occurs when:

1. Two or more threads access the same memory location.
2. At least one access is a write.
3. There is no synchronization between the threads (no lock, no atomic, no memory barrier).

In C++, a data race is **undefined behavior**. The standard says nothing about what happens. The program may:

- Crash.
- Produce silently wrong results.
- Appear to work in testing but fail at 3 AM in production.
- Do something different each time you run it.

The reason is that both the compiler and the CPU are allowed to reorder and cache operations. Without synchronization, there is no contract about what one thread sees from another.

**Minimal example:**

```cpp
#include <thread>
#include <vector>

int x = 0;

void increment() {
    for (int i = 0; i < 1'000'000; ++i) {
        x++;  // Data race: two threads, one write, no sync
    }
}

int main() {
    std::vector<std::thread> threads;
    threads.emplace_back(increment);
    threads.emplace_back(increment);
    
    for (auto& t : threads) t.join();
    
    std::printf("x = %d (expected 2'000'000)\n", x);
    return 0;
}
```

Run this a few times. You will not get 2,000,000. You might get 1,000,457. Then 1,234,891. Then 1,001,234. Each run is different. That is undefined behavior.

### Race Condition: Timing-Dependent Outcome

A **race condition** is not the same as a data race. A race condition occurs when the **outcome depends on the relative timing of events**, even if all accesses are properly synchronized.

You have correctly eliminated all data races, but the behavior is still not what you want.

**Example:**

```cpp
#include <thread>
#include <mutex>

struct Account {
    int balance = 100;
    std::mutex lock;
    
    void transfer_to(Account& dest, int amount) {
        std::lock_guard<std::mutex> l1(lock);
        std::lock_guard<std::mutex> l2(dest.lock);
        balance -= amount;
        dest.balance += amount;
    }
};

int main() {
    Account a, b;
    std::thread t1([&]{ a.transfer_to(b, 50); });
    std::thread t2([&]{ b.transfer_to(a, 30); });
    t1.join();
    t2.join();
    
    std::printf("a=%d, b=%d (total=%d)\n", a.balance, b.balance, 
                a.balance + b.balance);
    return 0;
}
```

This is *not* a data race — every access is locked. But depending on the order of execution, you might print:

- `a=80, b=120` (total=200) if the transfers execute sequentially.
- `a=80, b=120` (total=200) always, because the total is correct.

Actually, the total is *always* correct — no money is created or destroyed. But a human reading the code might expect `a=50, b=150` if they assumed `a.transfer_to(b)` runs first. The outcome is race-condition-dependent on the timing, but the race is not a bug because the contract (total is conserved) holds.

The point: **synchronization eliminates data races, but it does not eliminate race conditions.** Race conditions are a higher-level problem, about *what* you are synchronizing, not *whether* you are synchronizing.

### Deadlock: Threads Waiting Forever

A **deadlock** occurs when two or more threads are each waiting for a resource that another thread holds, and no thread can proceed.

**Classic example:**

```cpp
#include <thread>
#include <mutex>

std::mutex m1, m2;

void thread_a() {
    std::lock_guard<std::mutex> l1(m1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1)); // Let B acquire m2
    std::lock_guard<std::mutex> l2(m2); // Waiting for m2, held by B
    std::printf("A done\n");
}

void thread_b() {
    std::lock_guard<std::mutex> l2(m2);
    std::this_thread::sleep_for(std::chrono::milliseconds(1)); // Let A acquire m1
    std::lock_guard<std::mutex> l1(m1); // Waiting for m1, held by A
    std::printf("B done\n");
}

int main() {
    std::thread t_a(thread_a);
    std::thread t_b(thread_b);
    
    t_a.join(); // This may never return
    t_b.join();
    return 0;
}
```

Run this. It will hang. Neither thread can proceed because:

- Thread A holds `m1` and is waiting for `m2`.
- Thread B holds `m2` and is waiting for `m1`.

Deadlock is one of the hardest bugs to debug because it gives no error message — the program simply freezes.

Prevention strategies:

- **Lock ordering**: always acquire locks in the same global order. (If you always acquire `m1` before `m2`, you cannot deadlock.)
- **Try-lock with timeout**: acquire locks with a timeout; if you cannot acquire all locks within the timeout, release what you have and retry.
- **Lock-free algorithms**: avoid locks entirely.

---

## 55.3 Why Reordering Happens: Compiler and CPU

This section is about the deep reason data races produce undefined behavior. It requires understanding that **neither the compiler nor the CPU respects the order of operations in your code** if it thinks the reordering is invisible to a sequential reader.

### Compiler Reordering

The compiler optimizes your code by reordering reads and writes, as long as the reordering would not change the result in a single-threaded program.

**Example:**

```cpp
int x = 0, y = 0;

void thread_a() {
    x = 1;
    y = 1;
}

void thread_b() {
    if (y == 1) std::printf("saw y\n");
    if (x == 0) std::printf("but not x\n");
}
```

A sequential reader running thread A then thread B would expect: if `y == 1` at the check, then `x == 1` because `x` was set first. But the compiler can reorder thread A's writes:

```cpp
void thread_a() {
    y = 1;  // Reordered!
    x = 1;
}
```

Now if thread B executes between the reordered writes, it might print both "saw y" and "but not x" — even though no data race occurred (both accesses are to separate variables), the compiler's reordering violated your intent.

### CPU Reordering

Modern CPUs have *out-of-order execution*. While your code logically does:

```
store x = 1
store y = 1
```

The CPU might internally do:

```
initiate store y = 1 to the write buffer
initiate store x = 1 to the write buffer
store y = 1 completes and leaves the write buffer
store x = 1 is still pending
  (another thread reads y and sees 1, but x's write hasn't left the write buffer yet)
store x = 1 completes
```

The CPU reorders because it has multiple load/store units and a write buffer. It issues the stores in parallel, and they may complete in a different order than they were issued.

### Visibility vs. Reordering

Two distinct problems:

1. **Visibility**: Thread B doesn't see Thread A's write to `x` because the write is still in A's write buffer, not yet visible to other cores.
2. **Reordering**: Even if both writes are visible globally, they may be visible in a different order than they were issued.

Both are allowed without synchronization in C++.

### A Timeline

Here's what can happen without synchronization:

```
Time  Thread A              Thread B              RAM
----  --------              --------              ---
 1    x = 1 (in buffer)                          x = 0
 2    y = 1 (in buffer)                          y = 0
 3                          reads y → 0           (not visible yet)
 4                          reads x → 0           (not visible yet)
 5    y = 1 leaves buffer                        x = 0, y = 1
 6                          reads y → 1          x = 0, y = 1
 7    x = 1 leaves buffer                        x = 1, y = 1
 8                          reads x → 1          x = 1, y = 1
```

But reordering can also happen:

```
Time  Thread A              Thread B              RAM
----  --------              --------              ---
 1    y = 1 (in buffer)                          x = 0, y = 0
 2                          reads y → 0
 3    x = 1 (in buffer)                          x = 0, y = 0
 4    y = 1 leaves buffer                        x = 0, y = 1
 5                          reads y → 1
 6    x = 1 leaves buffer                        x = 1, y = 1
 7                          reads x → 1
```

Or:

```
Time  Thread A              Thread B              RAM
----  --------              --------              ---
 1    y = 1 (in buffer)
 2    x = 1 (in buffer)
 3                          reads y → 0
 4    y = 1 leaves buffer                        x = 0, y = 1
 5                          reads y → 1
 6                          reads x → 0          (x's write still pending!)
 7    x = 1 leaves buffer                        x = 1, y = 1
```

Without explicit synchronization, all three timelines are possible.

---

## 55.4 The Synchronization Primitives

### std::atomic<T>: Atomic Operations with Synchronization

An `std::atomic<T>` is a template that wraps a type `T` and provides:

1. **Atomic operations**: read, write, and compare-and-swap are indivisible at the hardware level.
2. **Happens-before synchronization**: an atomic write is visible to atomic reads on other threads, with guarantees about what other writes become visible too.

**Minimal example:**

```cpp
#include <atomic>
#include <thread>

std::atomic<int> x = 0;
std::atomic<int> y = 0;

void thread_a() {
    x.store(1, std::memory_order_release);
    y.store(1, std::memory_order_release);
}

void thread_b() {
    int ry = y.load(std::memory_order_acquire);
    int rx = x.load(std::memory_order_acquire);
    if (ry == 1 && rx == 0) {
        std::printf("impossible!\n"); // This will never print
    }
}

int main() {
    std::thread t_a(thread_a);
    std::thread t_b(thread_b);
    t_a.join();
    t_b.join();
    return 0;
}
```

Here, `release` and `acquire` semantics ensure: if thread B sees `y == 1`, then thread B also sees `x == 1`. The write to `y` **synchronizes with** the read of `y`.

### Memory Ordering

`std::atomic` operations take a `std::memory_order` argument that controls synchronization strength:

- **`memory_order_relaxed`**: no synchronization, just atomicity. Other threads see the write *eventually*, but there's no guarantee about when or relative to what.
- **`memory_order_acquire` / `memory_order_release`**: one-way synchronization. Release (on a write) ensures this write and all earlier writes are visible before a later acquire (on a read) returns.
- **`memory_order_acq_rel`**: two-way synchronization (for operations like compare-and-swap).
- **`memory_order_seq_cst`** (sequential consistency): all threads see all atomics in the same order. Strongest guarantee, slowest in practice.

**The intuition**:

- `relaxed`: "just make this operation atomic, don't sync."
- `release`: "I'm done with my work; make it visible to anyone who's waiting."
- `acquire`: "wait for someone's release; when I see the signal, I see everything they did before it."

Most hand-written concurrent code uses `acquire`/`release`. Sequential consistency (`seq_cst`) is the default and is safe but slower.

### volatile: What It Does NOT Do

A common misconception: **`volatile` does not provide thread-safety.**

`volatile` is a compiler directive that says: "this variable might change outside the compiler's control (e.g., memory-mapped hardware); don't optimize away reads/writes." It disables certain optimizations but provides *no synchronization*.

**What volatile does:**

```cpp
volatile int x = 0;
int v1 = x;  // Must actually load from memory, not assume cached value
int v2 = x;  // Must load again, not reuse v1
```

**What volatile does NOT do:**

```cpp
volatile int x = 0;
std::thread t1([](){ x = 1; });
std::thread t2([](){ 
    std::printf("%d\n", x);  // Might see 0 or 1, no ordering guarantee
});
```

Never use `volatile` for threading. Use `std::atomic<T>` instead.

### Mutexes: Coarse-Grained Synchronization

A **mutex** (mutual exclusion lock) ensures that only one thread can hold the lock at a time. While a thread holds the lock, all other threads trying to acquire it must wait.

```cpp
#include <mutex>
#include <thread>

int x = 0;
std::mutex m;

void increment() {
    for (int i = 0; i < 1'000'000; ++i) {
        std::lock_guard<std::mutex> lock(m);
        x++;  // Protected: only one thread at a time
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::printf("x = %d (expected 2'000'000)\n", x);
    return 0;
}
```

This will always print 2,000,000. The lock ensures that `x++` (which is a read-modify-write operation) appears atomic.

**Cost of mutexes:**

- **Uncontended**: acquiring an uncontended lock is ~20–50 ns on modern hardware. The compiler may optimize it away entirely if it's certain the mutex is uncontended.
- **Contended**: if two or more threads are waiting, the waiting thread is **descheduled** by the OS. The context switch cost is ~1–10 microseconds. In a highly contended scenario, you spend more time switching than working.
- **Unfairness**: a newly arriving thread might acquire the lock before a thread that has been waiting (spurious wakeups, lock implementation details).

**Recursive mutexes** (`std::recursive_mutex`) allow the same thread to acquire the lock multiple times. Less common; usually a sign that you need to rethink your locking strategy.

**Reader-writer locks** (`std::shared_mutex`) allow multiple readers but only one writer. Useful for read-heavy workloads:

```cpp
#include <shared_mutex>

std::shared_mutex m;
int x = 0;

void reader() {
    std::shared_lock<std::shared_mutex> lock(m);  // Multiple readers OK
    std::printf("x = %d\n", x);
}

void writer() {
    std::unique_lock<std::shared_mutex> lock(m);  // Exclusive
    x++;
}
```

The downside: reader-writer lock overhead is higher than a simple mutex (more complex bookkeeping). Use them only if you have many more readers than writers.

---

## 55.5 Lock-Free and Wait-Free Algorithms

Locks have a fundamental problem: **lock contention**. If many threads want the lock simultaneously, most are descheduled, and you lose parallelism.

**Lock-free** algorithms avoid locks by using atomic operations (especially compare-and-swap) to coordinate without blocking. The guarantee: *the system as a whole always makes progress*. Some thread will always be able to make forward progress, even if others are delayed.

**Wait-free** algorithms go further: *every thread always makes progress*, even under contention. These are even harder to write and rarely necessary.

### Example: Lock-Free Counter

```cpp
#include <atomic>

class LockFreeCounter {
    std::atomic<int> value = 0;
public:
    void increment() {
        int old = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_strong(
            old, old + 1, 
            std::memory_order_release, 
            std::memory_order_relaxed)) {
            // If someone else incremented, retry with the new value
        }
    }
    int read() const {
        return value.load(std::memory_order_acquire);
    }
};
```

The `compare_exchange_strong` operation atomically:

1. Reads the current value.
2. Compares it to `old`.
3. If equal, writes `old + 1` and returns true.
4. If not equal, updates `old` to the current value and returns false.

If another thread incremented between our read and our compare-exchange, we retry.

**Tradeoffs:**

- **Pros**: no thread is ever descheduled. Under high contention, a lock-free algorithm often outperforms a mutex.
- **Cons**: very hard to reason about. The retry loop can spin, wasting CPU if contention is high. Not always faster than a mutex (uncontended locks are already nearly free).

**Rule of thumb**: use `std::atomic` with `acquire`/`release` ordering for simple synchronization (flags, counters). Use lock-free algorithms (or real lock-free data structures like Intel's TBB) only after profiling shows lock contention is a problem.

Do not write custom lock-free data structures unless you are an expert. Use the algorithms library or third-party battle-tested code.

---

## 55.6 Thread-Local Storage and Immutability

Two other approaches to avoid contention:

### Thread-Local Storage

`thread_local` variables are per-thread: each thread gets its own copy.

```cpp
#include <thread>

thread_local int counter = 0;

void work() {
    for (int i = 0; i < 1'000'000; ++i) {
        counter++;  // No contention; each thread increments its own copy
    }
}

int main() {
    std::thread t1(work);
    std::thread t2(work);
    t1.join();
    t2.join();
    // counter is undefined here (which copy?)
    return 0;
}
```

No synchronization needed for reads/writes to thread-local variables — they're never accessed by other threads.

**Cost:**

- Allocation: thread creation allocates space for all thread-local variables. Hundreds to thousands of variables is fine; millions is wasteful.
- Access: accessing thread-local storage is slightly slower than a stack variable (a small address offset).

**When to use:**

- Per-thread buffers, caches, or accumulators (sum locally, then lock only to update global total).
- Per-request state in a web framework (request context, session data).

### Immutable Objects

If an object is immutable — no one can modify it after construction — then multiple threads can read it without any synchronization.

```cpp
class ImmutableVector {
    const std::vector<int> data;
public:
    ImmutableVector(std::vector<int> v) : data(std::move(v)) {}
    
    int at(int idx) const { return data.at(idx); }
    int size() const { return data.size(); }
    // No mutators
};

int main() {
    auto vec = std::make_shared<const ImmutableVector>(...);
    std::thread t1([vec]{ std::printf("%d\n", vec->at(0)); });
    std::thread t2([vec]{ std::printf("%d\n", vec->size()); });
    // No locks needed; both threads read safely
}
```

**Advantages:**

- Zero synchronization cost for readers.
- No race conditions possible (immutable = no mutations to race on).
- Easy to reason about (no hidden state changes).

**Disadvantage:**

- Updating requires allocating a new object and replacing the reference atomically. In a high-write scenario, this is expensive.

---

## 55.7 Worked Example: A Concurrent Counter

Let's implement a counter class five ways and see the tradeoffs.

### Version 1: Unsafe

```cpp
class UnsafeCounter {
    int value = 0;
public:
    void increment() { value++; }
    int read() const { return value; }
};
```

Fast, but data race under contention. Result is unpredictable.

### Version 2: Mutex-Protected

```cpp
#include <mutex>

class MutexCounter {
    int value = 0;
    mutable std::mutex m;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(m);
        value++;
    }
    int read() const {
        std::lock_guard<std::mutex> lock(m);
        return value;
    }
};
```

Safe. Under light contention, fast. Under heavy contention, threads are descheduled waiting for the lock.

### Version 3: Atomic

```cpp
#include <atomic>

class AtomicCounter {
    std::atomic<int> value = 0;
public:
    void increment() {
        value.fetch_add(1, std::memory_order_relaxed);
    }
    int read() const {
        return value.load(std::memory_order_relaxed);
    }
};
```

Safe. No descheduling. May be faster or slower than mutex depending on hardware and contention; on x86-64 with light contention, often comparable.

### Version 4: Lock-Free with Retries

```cpp
#include <atomic>

class LockFreeCounter {
    std::atomic<int> value = 0;
public:
    void increment() {
        int old = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_weak(
            old, old + 1, 
            std::memory_order_relaxed)) {}
    }
    int read() const {
        return value.load(std::memory_order_relaxed);
    }
};
```

No descheduling, but the retry loop can spin under contention. On real hardware with many cores, this can outperform a mutex because no one is descheduled.

### Version 5: Sharded (Thread-Local Accumulation)

```cpp
#include <atomic>
#include <vector>
#include <thread>

class ShardedCounter {
    struct Shard {
        std::atomic<long long> value = 0;
        char padding[128 - sizeof(std::atomic<long long>)];  // False sharing
    };
    std::vector<Shard> shards;
    
public:
    ShardedCounter() : shards(std::thread::hardware_concurrency()) {}
    
    void increment() {
        int shard_id = std::hash<std::thread::id>{}(std::this_thread::get_id()) 
                     % shards.size();
        shards[shard_id].value.fetch_add(1, std::memory_order_relaxed);
    }
    
    long long read() const {
        long long total = 0;
        for (const auto& s : shards) {
            total += s.value.load(std::memory_order_relaxed);
        }
        return total;
    }
};
```

Each thread increments its own shard (no contention on increments). Reading is O(number of threads). With padding to prevent false sharing, this can be orders of magnitude faster under high contention.

### Benchmark Results (Conceptual)

Under 8 cores, 10 million increments each:

```
Unsafe           ~10 ms    (wrong answer; uncontended write buffer tricks it)
Atomic           ~150 ms   (safe, but global contention)
Mutex            ~180 ms   (descheduling overhead)
Lock-free retry  ~140 ms   (spinning costs cache cycles)
Sharded          ~50 ms    (per-shard, no contention)
```

The lesson: **the best synchronization is no synchronization**. Sharding eliminates the need for shared mutable state entirely.

---

## 55.8 Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| No sync (data race) | Fastest | Undefined behavior, wrong results | Never (bugs) |
| `std::atomic` | Fast, no descheduling, clear semantics | Contention still exists, slower than unsharded under heavy load | Simple flags, counters, synchronization primitives |
| Mutex | Simple, fair, well-understood | Descheduling cost, contention bottleneck | General-purpose protection when contention is light |
| Reader-writer lock | Multiple readers simultaneously | Higher overhead than mutex, complex fairness issues | Read-heavy workloads (10:1 reader:writer ratio or more) |
| Lock-free algorithms | No descheduling, potentially fast | Hard to reason about, spinning under contention, rare bugs | High-contention scenarios after profiling |
| Sharded/thread-local | Eliminate contention entirely | More complex, read cost, per-thread memory overhead | High-contention counters, buffers, accumulators |
| Immutable + COW | Zero sync cost for readers, clear semantics | Expensive writes (allocation + swap), not suitable for mutable state | Mostly-read data shared across threads |

---

## 55.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "volatile makes my code thread-safe." | `volatile` disables compiler optimizations but provides no synchronization. Use `std::atomic<T>` instead. |
| "A lock protects an object; I don't need to think about it." | A lock protects a specific critical section. If you forget to lock somewhere, the lock elsewhere is useless. |
| "Atomic operations are always faster than mutexes." | On uncontended paths, mutexes are nearly free. Under heavy contention, atomics avoid descheduling but still fight for cache. Sharding wins. |
| "Lock-free algorithms are always faster." | Lock-free avoids descheduling but not contention. A spinning CAS loop can be slower than a descheduled thread that lets others run. Profile before choosing. |
| "My code is thread-safe if I synchronize all accesses." | Synchronization prevents data races, not race conditions. A correctly locked object can still behave non-deterministically based on timing. |
| "I should use reader-writer locks for everything." | Reader-writer locks have higher overhead than mutexes. Only worth it if reads vastly outnumber writes (10:1 or more). |
| "Immutable objects have no synchronization cost." | They don't, but creating an immutable copy is expensive. Immutability is cheap when objects are read-heavy and writes are rare. |

---

## 55.10 Exercises

1. **Find the data race.** In the following code, identify the data race and rewrite the critical section with a lock:

   ```cpp
   struct Node { int val; Node* next; };
   Node* head = nullptr;
   
   void push(int x) {
       auto n = new Node{x, nullptr};
       n->next = head;
       head = n;
   }
   
   void print() {
       for (Node* p = head; p; p = p->next) {
           std::printf("%d\n", p->val);
       }
   }
   ```

   If two threads call `push` and `print` simultaneously, what could go wrong? Write a mutex-protected version.

2. **Memory order experiment.** Write a program that demonstrates the difference between `memory_order_relaxed` and `memory_order_acquire`/`memory_order_release`. Create a flag and a payload:

   ```cpp
   std::atomic<bool> ready{false};
   int payload = 0;
   ```

   One thread sets `payload = 42`, then sets `ready = true`. Another thread spins until `ready` is true, then reads `payload`. With `relaxed` ordering, what could happen? With `acquire`/`release`, what is guaranteed?

3. **Sharding exercise.** Write a sharded counter class for 4 shards. Implement `increment()`, `read()`, and a `reset()`. Measure (conceptually or with code) how the read cost grows as you add more shards. At what shard count is the read cost higher than the benefit?

4. **Deadlock scenario.** Given two accounts, write a function that transfers money between them. What is the minimal set of rules needed to guarantee no deadlock? (Hint: think about lock ordering.)

5. **Immutability design.** Redesign the `Counter` from §55.7 as an immutable object. Instead of `increment()`, have a method that returns a new `Counter` with an incremented value. What does a multi-threaded program look like with this design? What are the tradeoffs?

6. **Race condition in synchronized code.** Write a class with a mutex-protected method that reads a value, does some work, then writes it back. Now write a sequence of calls (from multiple threads) where the method is correctly locked but the overall behavior is non-deterministic. Explain why synchronization did not prevent the non-determinism.

---

## 55.11 Summary

Thread safety is not about locking. It is about **visibility and ordering** — making sure that writes from one thread become visible to reads on another in a predictable order.

Without synchronization, the compiler and the CPU reorder operations, and you have undefined behavior. With synchronization, you establish **happens-before** relationships: "this operation on Thread A is visible before that operation on Thread B."

The tools are:

- **`std::atomic<T>`** for simple flags and counters.
- **Mutexes** for protecting critical sections (and light to moderate contention).
- **Lock-free algorithms** when contention is measured and proven to be a bottleneck.
- **Thread-local storage and sharding** to eliminate shared mutable state entirely.
- **Immutable objects** when writes are rare and reads are frequent.

The key insight: the best synchronization is **no shared mutable state**. Before reaching for locks, ask: can I make this object immutable? Can I use thread-local storage? Can I shard the work? Shared mutable state is the root of most threading bugs.

The next chapter looks at specific lock and atomic APIs, with deep dives into implementation details and performance characteristics.

---

**[← Previous: How Python AsyncIO Works](09-how-python-asyncio-works.md)** · **[↑ Part 5](README.md)** · **[Next: Locks and Atomics →](11-locks-and-atomics.md)**
