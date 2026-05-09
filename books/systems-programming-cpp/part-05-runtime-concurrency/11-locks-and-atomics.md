# Chapter 56 — Locks and Atomics

By now you understand that threads share memory and file descriptors. You know that `counter++` from two threads can race and lose increments. You've seen that critical sections need synchronization. This chapter teaches the two fundamental tools: **locks** (mutual exclusion with blocking) and **atomics** (hardware-supported read-modify-write without explicit blocking). A lock is a mutex with automatic cleanup via RAII. An atomic is a hardware operation. They sit at different levels of abstraction and solve overlapping problems. Knowing which to reach for is half the battle.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain how a mutex works at the kernel level: userland fast path with atomic compare-and-swap (CAS), slow path with futex syscalls to suspend threads.
2. Properly use `std::lock_guard`, `std::unique_lock`, and `std::scoped_lock` to synchronize access with automatic unlock via RAII.
3. Understand read-write locks (`std::shared_mutex`) and when they outperform plain mutexes (many readers, few writers).
4. Use `std::atomic<T>` for lock-free counters, flags, and state machines: load, store, compare_exchange, fetch_add.
5. Implement compare-exchange loops and understand the CAS-retry pattern at the heart of lock-free algorithms.
6. Reason about memory ordering: `memory_order_relaxed` for atomicity only, `memory_order_seq_cst` for full barriers (and why it's the safe default).
7. Recognize Treiber stacks and Michael-Scott queues as examples of lock-free structures and understand why they are subtle (ABA problem, hazard pointers).
8. Measure and decide between locking and lock-free approaches by understanding contention, cache effects, and fairness.

---

## How a Mutex Actually Works

A mutex is not magic. Under the hood, modern mutexes (like glibc's pthread mutex) combine two strategies: a fast path for the common case (uncontended lock acquisition) and a slow path (futex syscall) when contention is detected.

### The Fast Path: Atomic Compare-and-Swap

When thread A tries to acquire a mutex for the first time, it doesn't immediately call into the kernel. Instead, it performs an atomic compare-and-swap (CAS) operation on a word of memory inside the mutex. On x86-64, this is a single instruction: `cmpxchg` with a `LOCK` prefix.

```cpp
// Simplified pseudocode of the mutex fast path
std::mutex mu;

void acquire() {
    int expected = 0;  // unlocked
    int desired = 1;   // locked by this thread
    
    // Atomic CAS: if mu.value == 0, set it to 1 and succeed
    // Otherwise, fail and jump to slow path
    if (!mu.compare_exchange_strong(expected, desired)) {
        slow_path();  // contention; ask kernel to sleep
    }
}
```

If no other thread holds the lock (the mutex word is 0), the CAS succeeds in one instruction. Thread A's cache line containing the mutex word is now marked as owned (exclusive) in the L3 cache. Cost: ~10-40 cycles on modern CPUs, mostly hidden by instruction pipelining.

If another thread holds the lock, the CAS fails because the mutex word is already 1. Now thread A must call the slow path.

### The Slow Path: futex System Call

When the CAS fails, thread A calls a syscall: `futex(FUTEX_WAIT)` on Linux, or the equivalent on macOS/Windows.

```cpp
// Simplified slow path (Linux, using futex)
void slow_path() {
    // Tell the kernel: "Put me to sleep on this futex address.
    // Wake me when another thread calls FUTEX_WAKE on the same address."
    syscall(SYS_futex, &mu.value, FUTEX_WAIT, 1, nullptr, nullptr, 0);
    // Control returns here only when another thread wakes us or timeout occurs
}
```

The kernel marks thread A as blocked and deschedules it. Thread A is not spinning; it consumes no CPU. The scheduler runs other threads.

When thread C unlocks the mutex, it doesn't just CAS the word back to 0. It also calls `futex(FUTEX_WAKE)`:

```cpp
void release() {
    mu.store(0);  // atomic store; CAS back to unlocked
    // Wake up to 1 thread waiting on this futex
    syscall(SYS_futex, &mu.value, FUTEX_WAKE, 1, nullptr, nullptr, 0);
}
```

The kernel finds thread A in the futex wait queue, marks it runnable, and adds it to the scheduler's runqueue. On the next scheduling tick, thread A resumes execution.

### Diagram: Fast Path vs Slow Path

```
Thread A: mu.lock()

                    ┌─ Uncontended (fast path)
                    │
                    └─ attempt atomic CAS
                         success? (uncontended)
                         │        ├─ YES: acquire in ~40 cycles, return
                         │        │
                         │        └─ NO: contention detected, jump to slow path
                         │
                         └─ futex(FUTEX_WAIT) syscall
                              [Thread A blocks, kernel deschedules]
                              [~1-2 microseconds to context switch]
                              [Thread A consumes 0 CPU while sleeping]
                              
                         ... time passes, another thread holds lock ...
                         
                         [Unlock calls futex(FUTEX_WAKE)]
                         [Kernel marks Thread A runnable]
                         [Scheduler eventually runs Thread A]
                         └─ return from syscall, continue
```

This is why **uncontended locks are nearly free**: a CAS instruction is cheap, and if only one thread ever acquires the lock, no syscall occurs. Contended locks are expensive because syscalls involve context switches, TLB flushes, and scheduler overhead.

### Memory Ordering Inside the Lock

The CAS instruction on x86-64 with the `LOCK` prefix includes a full memory barrier. This means:
- All loads and stores before the lock are visible to threads that acquire the same lock later.
- The acquire acts as a barrier against code motion.

This is crucial: **a lock is not just about mutual exclusion; it's also a synchronization point that ensures memory visibility.** More on this in Chapter 12 (Memory Ordering), but for now: if thread A writes to a variable inside a critical section protected by a lock, and thread B locks the same lock later, thread B will see thread A's writes. The lock acts as a checkpoint.

---

## std::lock_guard, std::unique_lock, std::scoped_lock

To avoid manually calling `mu.lock()` and `mu.unlock()`, and to handle early returns and exceptions correctly, C++ provides RAII wrappers around mutexes. These are the lock classes.

### std::lock_guard: Simple and Strict

`std::lock_guard<T>` is the simplest. Its constructor acquires the lock, its destructor releases it.

```cpp
#include <mutex>
#include <thread>
#include <iostream>

int counter = 0;
std::mutex mu;

void worker() {
    {
        std::lock_guard<std::mutex> lock(mu);  // acquires mu
        // critical section
        counter++;
        std::cout << "counter=" << counter << "\n";
    }  // destructor releases mu
    
    // non-critical section here; lock is released
}

int main() {
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
    std::cout << "final: " << counter << "\n";
    return 0;
}
```

`std::lock_guard`:
- Constructor calls `mu.lock()` (blocks until acquired).
- Destructor calls `mu.unlock()` (atomically releases).
- Cannot be moved or copied; it's tied to one scope.
- No way to check if the lock is held; no way to release early; no way to reacquire.

If you need to release the lock early or check its status, use `std::unique_lock`.

### std::unique_lock: Flexible Ownership

`std::unique_lock<T>` is more powerful. It models *ownership* of the lock: the lock can be acquired, released, and reacquired.

```cpp
void example() {
    std::unique_lock<std::mutex> lock(mu);  // acquires
    // critical section
    counter++;
    
    lock.unlock();  // release early
    std::cout << "lock released\n";
    // now non-critical code
    
    lock.lock();    // reacquire
    counter++;      // critical again
}  // destructor releases if still held
```

`std::unique_lock` also supports **deferred locking**:

```cpp
void example() {
    std::unique_lock<std::mutex> lock(mu, std::defer_lock);
    // mu is not locked yet
    
    // ... do something non-critical ...
    
    lock.lock();   // now lock
    counter++;
    // critical section
}  // destructor releases
```

And it supports **move semantics**: ownership can be transferred:

```cpp
std::unique_lock<std::mutex> acquire_lock() {
    std::unique_lock<std::mutex> lock(mu);
    return lock;  // move; lock is still held by the caller
}

void example() {
    auto lock = acquire_lock();  // receive ownership
    counter++;
    // lock is held
}  // destructor releases
```

You can also check if the lock is held:

```cpp
void example() {
    std::unique_lock<std::mutex> lock(mu, std::defer_lock);
    if (lock.try_lock()) {
        // lock acquired
        counter++;
    } else {
        // could not acquire (another thread holds it)
        std::cout << "lock busy\n";
    }
    // destructor releases if held
}
```

### std::scoped_lock: Multiple Locks Without Deadlock

When you need to acquire multiple locks, deadlock is a risk. If thread A locks mu1 then mu2, and thread B locks mu2 then mu1, and both threads execute simultaneously, they can deadlock: A waits for mu2 (held by B), B waits for mu1 (held by A).

`std::scoped_lock` solves this by acquiring multiple locks in a fixed order, regardless of the order you pass them:

```cpp
std::mutex mu1, mu2;
int a = 0, b = 0;

void transfer() {
    // UNSAFE: deadlock risk
    // std::lock_guard<std::mutex> lock1(mu1);
    // std::lock_guard<std::mutex> lock2(mu2);
    
    // SAFE: scoped_lock acquires in a canonical order
    std::scoped_lock locks(mu1, mu2);
    
    // both mutexes are now held in a consistent order
    a++;
    b--;
}
```

Internally, `std::scoped_lock` calls `std::lock()` which uses a deadlock-avoidance algorithm (e.g., acquiring locks in address order). The effect: no matter how many threads call `transfer()`, they all acquire the locks in the same order, so deadlock is impossible.

### Summary Table: Lock Types

| Class | Use When | Pros | Cons |
|---|---|---|---|
| `lock_guard<T>` | Lock for an entire scope | Simple, minimal overhead, exception-safe | Cannot unlock early, cannot check status |
| `unique_lock<T>` | Need unlock/relock, deferred lock, or move ownership | Flexible, movable, can defer lock | Slightly more overhead (state tracking) |
| `scoped_lock` | Acquiring multiple locks | Deadlock-free ordering, simple syntax | Only for acquiring all locks at once |

---

## Read-Write Locks

In many real-world scenarios, the critical section has many **readers** (threads that only read a shared variable) and few **writers** (threads that modify it). A plain mutex serializes all access, but readers don't conflict with each other — two threads can safely read the same variable simultaneously.

`std::shared_mutex` allows multiple threads to hold a **shared lock** (read lock) simultaneously, or one thread to hold an **exclusive lock** (write lock) alone.

```cpp
#include <shared_mutex>

struct UserCache {
    mutable std::shared_mutex mu;
    std::unordered_map<int, User> cache;
    
    // Reader: acquires shared lock
    User get(int id) const {
        std::shared_lock<std::shared_mutex> lock(mu);  // shared lock
        auto it = cache.find(id);
        if (it != cache.end()) return it->second;
        throw std::runtime_error("not found");
    }
    
    // Writer: acquires exclusive lock
    void set(int id, const User& u) {
        std::unique_lock<std::shared_mutex> lock(mu);  // exclusive lock
        cache[id] = u;
    }
};

int main() {
    UserCache cache;
    
    std::thread reader1([&]() {
        for (int i = 0; i < 100; i++) {
            cache.get(1);  // shared lock
        }
    });
    
    std::thread reader2([&]() {
        for (int i = 0; i < 100; i++) {
            cache.get(1);  // shared lock; runs simultaneously with reader1
        }
    });
    
    std::thread writer([&]() {
        cache.set(1, User{1, "Alice"});  // exclusive lock; waits for readers
    });
    
    reader1.join();
    reader2.join();
    writer.join();
    return 0;
}
```

Reader 1 and Reader 2 both call `get()`, which acquires a shared lock. The locks grant simultaneously; both readers hold the lock at the same time. The writer calls `set()`, which acquires an exclusive lock. If readers are holding shared locks, the writer *waits*. If the writer holds an exclusive lock, new readers wait. This is the **reader-writer** pattern.

### Performance: When It Wins and When It Doesn't

`shared_mutex` shines when:
- **Read-heavy workload**: 100 readers to 1 writer. Readers get fast parallel access.
- **Long critical section**: readers hold locks for a long time, so parallelism matters.

`shared_mutex` loses when:
- **Contended writes**: writers frequently modify the data. Exclusive locks still serialize access.
- **Short critical section**: the overhead of distinguishing shared vs exclusive locks is more than the time spent in the critical section itself.
- **Equal read-write ratio**: no parallelism advantage if reads and writes are balanced.

Benchmark your workload. A plain `std::mutex` is often faster for short critical sections because it has less bookkeeping.

---

## Atomic Operations

Sometimes you don't want to pay the cost of a lock (even an uncontended one). For simple operations like incrementing a counter, `std::atomic<T>` provides lock-free synchronization.

### std::atomic<T>: Atomic Types

`std::atomic<T>` is a template wrapper around a type T that makes operations on it atomic (indivisible). On most architectures, this is implemented using hardware atomics, not locks.

```cpp
#include <atomic>
#include <thread>

std::atomic<int> counter(0);  // atomic counter initialized to 0

void worker() {
    for (int i = 0; i < 1000; i++) {
        counter++;  // atomic increment
    }
}

int main() {
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
    std::cout << "counter=" << counter << "\n";  // prints 2000, always
    return 0;
}
```

Unlike `counter++` on a plain `int`, which is a load-modify-store sequence (non-atomic), `counter++` on an `std::atomic<int>` is a single **atomic operation**. Under the hood, it becomes an `inc` or `add` instruction with a `LOCK` prefix (x86-64), which ensures exclusivity.

### Core Operations

```cpp
std::atomic<int> x(5);

// Load: read the value
int v = x.load();                  // read atomically

// Store: write the value
x.store(10);                       // write atomically

// Exchange: write new value, return old value
int old = x.exchange(20);          // old == 10

// Compare-exchange (strong): CAS loop kernel
int expected = 20;
if (x.compare_exchange_strong(expected, 30)) {
    // success: x was 20, now 30
} else {
    // failure: x was not 20; expected is updated to actual value
}

// Compare-exchange (weak): may fail spuriously (useful on ARM/PowerPC)
std::atomic<bool> done(false);
while (!done.compare_exchange_weak(expected, desired)) {
    // retry on spurious failure
}

// Arithmetic operations (for integral types)
x.fetch_add(5);      // x += 5, atomically; return old value
x.fetch_sub(3);      // x -= 3, atomically
x.fetch_and(0xFF);   // x &= 0xFF, atomically
x.fetch_or(0x01);    // x |= 0x01, atomically

// Prefix/postfix operators
++x;                 // increment, return new value
x++;                 // increment, return old value
--x;
x--;
```

### Hardware Implementation: x86-64 vs ARM

On x86-64, atomics use the `LOCK` prefix. For example, `x++` becomes:

```asm
lock incq %rax      # atomic increment; prefixed with LOCK
```

The `LOCK` prefix causes the CPU to:
1. Acquire exclusive ownership of the cache line containing `x`.
2. Execute the increment.
3. Release the cache line.
4. Ensure all other CPUs see the change before the instruction completes.

On ARM (ARMv8), atomics use load-linked / store-conditional (LL/SC):

```asm
ldaxr %x0, [%x1]    # load-acquire exclusive: load & acquire barrier
add %x0, %x0, 1     # increment
stlxr %w2, %x0, [%x1]  # store-release exclusive: store & release barrier
```

The LL/SC pair works differently: if another CPU modifies the memory location between the load and store, the store fails (returns nonzero in %w2), and the operation retries.

Both approaches achieve the same goal: atomic operations that don't require explicit locking.

### Example: Lock-Free Counter

```cpp
#include <atomic>
#include <thread>
#include <vector>

std::atomic<long long> total(0);

void accumulate(int n) {
    for (int i = 0; i < n; i++) {
        total.fetch_add(1);  // lock-free increment
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; i++) {
        threads.emplace_back(accumulate, 1000);
    }
    for (auto& t : threads) t.join();
    
    std::cout << "total=" << total << "\n";  // always 8000
    return 0;
}
```

This is much faster than using a mutex because no thread blocks or context-switches. Atomics are ideal for simple counters, flags, and state machine transitions.

---

## Compare-Exchange Loops

The `compare_exchange` operation is the building block of lock-free algorithms. It's not a full solution by itself; you wrap it in a loop.

### The CAS Loop Pattern

```cpp
std::atomic<int> x(5);

// Goal: atomically set x to 20 if it's currently between 10 and 19
void try_update() {
    int expected = x.load();  // read current value
    
    while (true) {
        if (expected < 10 || expected > 19) {
            // condition not met; don't update
            return;
        }
        
        int desired = 20;
        if (x.compare_exchange_strong(expected, desired)) {
            // success: x was expected, is now desired
            return;
        }
        // failure: x changed; expected is now the new value
        // loop and try again
    }
}
```

The pattern:
1. Load current value into `expected`.
2. Check a condition and compute new value.
3. CAS: try to swap expected for new value.
4. If CAS succeeds, we're done.
5. If CAS fails (another thread changed x), loop back to step 1 with the new value in `expected`.

This is the kernel of algorithms like Treiber stacks and Michael-Scott queues.

### Example: CAS Loop for Min-Update

```cpp
std::atomic<int> min_value(INT_MAX);

// Atomically update min_value to min(min_value, new_val)
void update_min(int new_val) {
    int expected = min_value.load();
    
    while (new_val < expected) {
        if (min_value.compare_exchange_strong(expected, new_val)) {
            return;  // success
        }
        // expected is now the actual current value
        // loop if new_val < expected
    }
}
```

If two threads call `update_min()` simultaneously with values 5 and 7:
- Thread A reads `min_value = INT_MAX`, computes desired = 5, CAS succeeds, returns.
- Thread B reads `min_value = INT_MAX`, but by the time it CAS, min_value is 5. CAS fails; expected becomes 5.
- Thread B's condition `7 < 5` is false, so it returns without updating.

The final value is correctly 5.

---

## Memory Ordering Briefly

`std::atomic` operations accept a `memory_order` parameter that controls how strictly the operation orders memory. This is a subtle topic; Chapter 12 covers it deeply. For now, understand the key cases:

### memory_order_relaxed

Atomicity only. No memory barrier. Use when you don't care about synchronization with other threads, only about indivisibility.

```cpp
std::atomic<bool> flag(false);

void thread_a() {
    flag.store(true, std::memory_order_relaxed);  // just atomic store
}

void thread_b() {
    if (flag.load(std::memory_order_relaxed)) {  // just atomic load
        // other data may not be visible here!
    }
}
```

If thread_a writes to shared memory before calling `store`, thread_b might not see those writes even if the flag is true. Use only when you're certain no synchronization is needed.

### memory_order_seq_cst (Sequential Consistency)

Full barriers on all sides. Strongest guarantee: all threads see operations in a single, consistent order. This is the **default** and the safest choice.

```cpp
std::atomic<int> x(0);
std::atomic<bool> done(false);

void thread_a() {
    x.store(42, std::memory_order_seq_cst);  // full barrier
    done.store(true, std::memory_order_seq_cst);
}

void thread_b() {
    while (!done.load(std::memory_order_seq_cst)) {  // full barrier
        // spin
    }
    int v = x.load(std::memory_order_seq_cst);  // guaranteed to see 42
}
```

If thread_b sees `done = true`, it will see `x = 42`. The seq_cst barriers ensure this ordering is visible.

Most of the time, use `memory_order_seq_cst` (the default). Only optimize to weaker orderings (`acquire`, `release`, `relaxed`) after profiling shows a bottleneck and after careful reasoning about what synchronization you actually need. More in Chapter 12.

---

## Lock-Free Data Structures (Brief)

Atomics enable lock-free data structures where multiple threads manipulate a structure without explicit locks. Two famous examples:

### Treiber Stack

A lock-free stack using atomics:

```cpp
template<typename T>
class TreiberStack {
private:
    struct Node {
        T value;
        Node* next;
    };
    
    std::atomic<Node*> head;
    
public:
    TreiberStack() : head(nullptr) {}
    
    void push(const T& val) {
        Node* new_node = new Node{val, nullptr};
        
        // CAS loop: keep trying until we successfully link our node
        while (true) {
            Node* old_head = head.load();
            new_node->next = old_head;
            
            if (head.compare_exchange_strong(old_head, new_node)) {
                return;  // success; our node is now at the head
            }
            // another thread pushed; old_head is updated to actual head, retry
        }
    }
    
    bool pop(T& result) {
        while (true) {
            Node* old_head = head.load();
            if (!old_head) return false;  // empty
            
            Node* new_head = old_head->next;
            
            if (head.compare_exchange_strong(old_head, new_head)) {
                result = old_head->value;
                delete old_head;
                return true;
            }
            // another thread popped or pushed; retry
        }
    }
};
```

The push and pop both use CAS loops. Multiple threads can push simultaneously; the CAS ensures that each node is atomically linked.

### The ABA Problem

Lock-free structures have a subtle bug: the **ABA problem**.

Suppose:
1. Thread A reads `head = Node_X`.
2. Thread B pops Node_X, then pushes a different node with the same pointer value (reused memory).
3. Thread A's CAS sees the pointer value is still the same and succeeds, even though the node is different.

This is rare but possible. Solutions:
- **Hazard pointers**: threads announce which pointers they're reading; memory reclamation waits until no thread is reading.
- **Epoch-based reclamation**: delay freeing memory until all threads have advanced past the current epoch.
- **Use a library**: production lock-free data structures (Intel TBB, Folly, etc.) handle ABA for you.

### Recommendation: Use Library Implementations

Writing lock-free code is error-prone. The Treiber stack is elegant but doesn't handle ABA. Instead:
- Use `std::queue` with `std::mutex` for correctness.
- For high-performance needs, use a battle-tested library: **Intel TBB**, **Folly**, **Boost.Lockfree**.
- Profile before optimizing to lock-free; many "lock-free" implementations lose to well-optimized mutexes under real contention.

---

## Worked Example: Blocking vs Lock-Free Counter

Let's compare two designs for a shared counter: blocking (with mutex) and lock-free (with atomics). We'll benchmark both.

### Blocking Counter

```cpp
#include <mutex>

class BlockingCounter {
private:
    mutable std::mutex mu;
    long long count;
    
public:
    BlockingCounter() : count(0) {}
    
    void increment() {
        std::lock_guard<std::mutex> lock(mu);
        count++;
    }
    
    long long get() const {
        std::lock_guard<std::mutex> lock(mu);
        return count;
    }
};
```

### Lock-Free Counter

```cpp
#include <atomic>

class LockFreeCounter {
private:
    std::atomic<long long> count;
    
public:
    LockFreeCounter() : count(0) {}
    
    void increment() {
        count.fetch_add(1);
    }
    
    long long get() const {
        return count.load();
    }
};
```

### Benchmark Code

```cpp
#include <chrono>
#include <thread>
#include <vector>
#include <iostream>

template<typename Counter>
void benchmark(const char* name, int threads, long long iterations_per_thread) {
    Counter counter;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> workers;
    for (int i = 0; i < threads; i++) {
        workers.emplace_back([&]() {
            for (long long i = 0; i < iterations_per_thread; i++) {
                counter.increment();
            }
        });
    }
    
    for (auto& t : workers) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << " (" << threads << " threads, "
              << threads * iterations_per_thread << " ops): "
              << elapsed.count() << " ms\n";
}

int main() {
    benchmark<BlockingCounter>("Mutex counter", 4, 10000000);
    benchmark<LockFreeCounter>("Atomic counter", 4, 10000000);
    return 0;
}
```

### Expected Results

On a 4-core machine with high contention:

```
Mutex counter (4 threads, 40000000 ops): 340 ms
Atomic counter (4 threads, 40000000 ops): 85 ms
```

The lock-free counter is 4x faster because:
- No syscalls (futex).
- No context switches.
- No lock contention; each CPU core updates its cache line atomically.
- The `fetch_add` instruction is nearly cache-hit cheap if the line is already owned by the core.

**However**, under *low* contention (few threads, long critical section):

```
Mutex counter (1 thread, 40000000 ops): 45 ms
Atomic counter (1 thread, 40000000 ops): 43 ms
```

The difference is minimal because the mutex fast path (uncontended CAS) is almost as cheap as an atomic fetch_add.

### When Each Wins

| Scenario | Winner | Why |
|---|---|---|
| High contention (many threads on same counter) | Lock-free atomic | No syscalls, no context switches, cache efficiency |
| Low contention (few threads, rarely acquire lock) | Either (similar) | Mutex fast path is nearly free; atomic fetch_add is similar cost |
| Complex critical section | Mutex | Easier to reason about; locks serialize properly. Lock-free requires careful CAS loop design |
| Reader-heavy workload | `shared_mutex` | Multiple readers; single writer |
| Simple flag or counter | Atomic | Minimal overhead |

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| `std::mutex` | Simple, easy to reason about, handles complex critical sections | Syscalls under contention, context switches, slower than lock-free | Most cases; default choice |
| `std::shared_mutex` | Parallelizes readers | Bookkeeping overhead, slower if contention is high | Read-heavy workloads (many readers, few writers) |
| `std::atomic<T>` | Lock-free, very fast, no syscalls | Subtle (CAS loops, ABA problem, memory ordering), limited to simple types | Counters, flags, simple state machines |
| Lock-free structures (Treiber, MPMC) | Maximum throughput under high contention | Complex to implement correctly, ABA problem, harder to debug | Performance-critical infrastructure (thread pools, work queues) |
| Condition variables + mutex | Allows waiting for a condition | Requires mutex and explicit signaling | Producer-consumer patterns, waiting for events |

---

## Common Misconceptions

1. **"Atomics are always faster than locks."** False. Under low contention, a mutex's fast path (uncontended CAS) is comparable to an atomic fetch_add. Under high contention, atomics have the advantage because they avoid syscalls. The real win is no blocking — threads don't sleep. But if contention is light, measure first.

2. **"Lock-free code is always better."** False. Lock-free algorithms are harder to write, test, and debug. They don't compose well (you can't combine two lock-free structures into a larger lock-free structure easily). Use lock-free only after profiling shows it's a bottleneck and you have expertise.

3. **"I should never use a mutex because atomics are faster."** Backwards. Mutexes are the right tool for protecting complex critical sections. Atomics are for simple operations. Use mutexes by default, optimize to atomics if profiling indicates contention on a simple counter or flag.

4. **"If I use atomic<int>, my data is automatically thread-safe."** Incomplete. `atomic<int>` makes individual operations on the int atomic, but if you access multiple atomics in sequence, they can still race. For example:

   ```cpp
   std::atomic<int> x(0), y(0);
   
   void thread_a() {
       x.store(1);
       y.store(1);
   }
   
   void thread_b() {
       int v1 = x.load();
       int v2 = y.load();
       // may see x=1, y=0 if thread_a is mid-way
   }
   ```

   Each load/store is atomic, but the sequence is not. You need a lock to protect compound operations.

5. **"Memory barriers in atomic operations are expensive."** Somewhat true, but understand the cost. A `seq_cst` operation is more expensive than `relaxed`, but both are still cheaper than a mutex under contention. The real cost is on weaker architectures (ARM, PowerPC) where barriers involve memory fences. On x86-64, the cost is relatively low (a few cycles), often hidden by the CPU's out-of-order execution.

6. **"Compare-and-swap is only for expert lock-free programming."** False. CAS loops appear in many standard library algorithms and are a core building block. Understanding the pattern (read, compute, CAS-loop) is foundational knowledge for concurrent systems.

---

## Exercises

1. **Mutex fast path benchmark.** Write a program that increments a counter protected by a `std::mutex` with varying thread counts (1, 2, 4, 8, 16). Measure wall-clock time for 10 million increments total. Do you see a knee in the graph where context switches dominate? At how many threads does the slow path (futex syscall) become the bottleneck?

2. **Lock-free counter under contention.** Compare `std::atomic<long long>` vs `std::mutex`-protected counter. Increment 10 million times across N threads. Measure at N=1, 2, 4, 8. At what thread count does lock-free win by >10%? Why?

3. **Scoped_lock vs manual ordering.** Write a function that acquires two mutexes (mu1, mu2) in order. Implement it three ways: (a) manual `lock_guard` for each (correct only if you remember the right order), (b) `unique_lock` with try_lock and retry on failure, (c) `scoped_lock`. Benchmark all three with many threads trying to acquire both locks simultaneously. Which is fastest and easiest to reason about?

4. **Reader-writer performance.** Implement a read-heavy cache with `std::shared_mutex`. Simulate 100 threads reading and 1 thread writing. Measure total time. Then implement the same with `std::mutex`. Which is faster? At what reader-to-writer ratio does `shared_mutex` break even?

5. **CAS loop for a linked list insert.** Implement a simple lock-free singly linked list (not a stack) with CAS-based insert. Handle the case where two threads try to insert at the same position simultaneously. Do NOT worry about memory reclamation (assume nodes are never freed). Test with 4 threads inserting 1000 elements each. Verify the final list has 4000 elements in valid order (may be interleaved, but no duplicates or missing elements).

6. **Memory ordering experiment.** Write a program with two threads: Thread A writes to a regular `int`, then sets an `std::atomic<bool>` to true. Thread B waits for the atomic to be true, then reads the regular `int`. Compile with `-O2` and run many times. Does it always read the correct value? Now change the atomic operations to `memory_order_relaxed`. Can the program now break? (Hint: on weakly ordered systems like ARM, it might; on x86-64, it might not because x86 is strongly ordered.)

---

## Summary

Locks and atomics are the two fundamental primitives for concurrent programming. A mutex is a kernel synchronization primitive wrapped in RAII classes (`lock_guard`, `unique_lock`, `scoped_lock`) that handle the common patterns of locking one or multiple resources without deadlock. Atomics (`std::atomic<T>`) provide lock-free operations on simple types using hardware instructions, eliminating syscalls and context switches. The cost-benefit tradeoff depends on contention: light contention favors simple mutexes; heavy contention on simple operations favors atomics. Complex critical sections always belong behind a mutex; complex lock-free data structures are rarely worth building from scratch because they're subtle and libraries handle ABA correctly. Understanding the fast path (uncontended CAS) and slow path (futex syscall) of a mutex explains why uncontended locks are nearly free and contended locks are expensive. Memory ordering (the topic of Chapter 12) determines what other threads see when they access atomics; the default `memory_order_seq_cst` is safe but slightly slower; only use weaker orderings after profiling and careful reasoning.

---

**[← Previous: Chapter 55 — Thread Safety](10-thread-safety.md)** · **[↑ Part 5](README.md)** · **[Next: Chapter 57 — Memory Ordering →](12-memory-ordering.md)**
