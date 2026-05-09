# Chapter 10 — Reference Counting

## Learning Objectives

By the end of this chapter you will be able to:

1. **Describe** the reference counting algorithm: on every pointer copy, increment; on every pointer destruction, decrement; when it hits zero, run the destructor.
2. **Trace** the lifetime of a reference-counted object, from allocation through the final decrement, and predict when its destructor runs.
3. **Explain** why reference counting is deterministic (destructors run immediately when the last reference dies) but has a fatal weakness: cycles.
4. **Compare** atomic vs non-atomic refcount increments, and calculate the performance cost of making counts thread-safe.
5. **Identify** the cycle problem in concrete scenarios (parent–child object graphs, event listeners, caches) and recognize why it causes leaks.
6. **Implement** the standard fixes: weak references, cycle detection, and backup garbage collection.
7. **Reason about** reference counting vs tracing GC: when to use each, and the performance/pauses tradeoff.

We are going to build one of the simplest automatic memory management schemes that actually works. It is not free — there are per-operation costs and a fatal flaw that hits real code — but it is honest: when an object is no longer reachable, it dies *immediately*, not "eventually after a GC pause."

This chapter assumes you understand pointers, ownership, and the semantic difference between stack and heap (Chapter 8).

---

## 10.1 The Core Algorithm

Reference counting is built on a single idea: **every dynamically allocated object has a counter that tracks how many pointers currently point to it.** When you copy a pointer, you increment. When you destroy a pointer, you decrement. When the counter hits zero, the object is dead and you free its memory. That's the entire algorithm.

Let's make it concrete. Imagine a simple `Node` type (think of it as part of a tree or linked list):

```cpp
struct Node {
    int value;
    Node* next;
};
```

In a reference-counted world, every `Node` secretly carries a refcount:

```cpp
struct Node {
    int refcount;      // Hidden from the programmer; the RC system maintains it
    int value;
    Node* next;
};
```

Now consider a sequence of operations:

```cpp
Node* a = allocate_node(1, nullptr);   // a->refcount = 1
Node* b = a;                           // b->refcount = 2
Node* c = a;                           // c->refcount = 3
b = nullptr;                           // b->refcount = 2
a = nullptr;                           // a->refcount = 1
c = nullptr;                           // c->refcount = 0, FREE THE OBJECT
```

ASCII diagram of an object's refcount over time:

```
     Allocate       Copy to b    Copy to c    Drop b      Drop a      Drop c
        |              |            |           |           |           |
        1              2            3           2           1           0 (FREE)
        +              +            +           -           -           -
        |______________|____________|___________|___________|___________|
        
        a lives
         |_________b lives___________|
                  |_________c lives___________|
```

The **key invariant**: the refcount equals the number of live pointers to the object. When the last pointer is destroyed, the count becomes zero, and the memory is immediately returned to the allocator. No pause. No tracing. No ambiguity.

---

## 10.2 Implementation Details: The Reference-Counting Wrapper

In practice, you don't sprinkle refcount fields into every struct. Instead, you use a **wrapper type** that holds the refcount and a pointer to the actual object. C++ `std::shared_ptr` is the canonical example; Swift and Objective-C do it in the runtime.

Here's a minimal C++ refcounted pointer:

```cpp
template <typename T>
class RefPtr {
private:
    struct ControlBlock {
        std::atomic<int> refcount;  // Atomic for thread safety
        T* object;
        
        ControlBlock(T* obj) : refcount(1), object(obj) {}
    };
    
    ControlBlock* block;
    
public:
    // Constructor: allocate object and control block
    RefPtr(T* obj) : block(new ControlBlock(obj)) {}
    
    // Copy constructor: increment
    RefPtr(const RefPtr& other) : block(other.block) {
        if (block) block->refcount.fetch_add(1, std::memory_order_release);
    }
    
    // Destructor: decrement, free if zero
    ~RefPtr() {
        if (block && block->refcount.fetch_sub(1, std::memory_order_acquire) == 1) {
            delete block->object;
            delete block;
        }
    }
    
    // Access the object
    T* operator->() const { return block->object; }
    T& operator*() const { return *block->object; }
};
```

Every copy increments the refcount. Every destruction decrements. When the refcount reaches zero, the object is deleted. The control block (which holds the refcount) is freed alongside the object.

The timing of this is **deterministic and immediate**: the moment the last `RefPtr` is destroyed, the object's destructor runs. No delay, no pause. This is a stark contrast to garbage collection.

---

## 10.3 Atomic vs Non-Atomic Refcounts

The refcount increment/decrement is a load-modify-store sequence. In a single-threaded program, a plain `int` is fine:

```cpp
object->refcount++;  // Not atomic, but no one else is touching it
```

In a multi-threaded program, two threads can race:

```cpp
// Thread A                              // Thread B
load (refcount = 5)                      load (refcount = 5)
increment (6)                            increment (6)
store (6)                                store (6)
                                         // Both threads think refcount is 6
                                         // But two increments should give 7
```

The object is never freed (refcount is stuck at 6), or it is freed too early (both threads fetch_sub and both see it reach zero). This is a bug.

The fix: use `std::atomic<int>`:

```cpp
block->refcount.fetch_add(1, std::memory_order_release);
```

This is a single CPU instruction (on most platforms) that is atomic. It is also *expensive*:

- **Atomic increment cost**: 10–30 cycles on a typical modern CPU, depending on cache locality.
- **Non-atomic increment cost**: 1 cycle.
- **The difference**: 20–30 extra cycles per copy/destruction.

For reference-counted objects that are copied frequently, this adds up. A tight loop that copies and destroys a shared_ptr millions of times will be dominated by the atomic cost.

**Why does CPython get away with non-atomic refcounts?** Because Python has the Global Interpreter Lock (GIL). Only one thread runs Python bytecode at a time. The refcount field is protected by the GIL, so no atomics are needed. This is one of the reasons why CPython threads are less useful than you'd expect — the GIL serializes refcount operations.

Swift and Objective-C use atomic refcounts because they support true multi-threading.

---

## 10.4 The Cycle Problem

Reference counting has a fatal flaw: it cannot handle cycles.

### The Problem

Imagine two objects that point to each other:

```cpp
struct Node {
    RefPtr<Node> next;
};

RefPtr<Node> a = new Node{};
RefPtr<Node> b = new Node{};

a->next = b;   // a.refcount = 1, b.refcount = 2
b->next = a;   // a.refcount = 2, b.refcount = 2
```

Now both objects have a refcount of 2:
- `a` is referenced by: the local variable `a` (1) and `b->next` (1).
- `b` is referenced by: the local variable `b` (1) and `a->next` (1).

Now destroy the local variables:

```cpp
a = nullptr;   // a's refcount goes from 2 to 1. Not freed.
b = nullptr;   // b's refcount goes from 2 to 1. Not freed.
```

Both objects still have a refcount of 1, but there are no external pointers to them. They are *unreachable*, but alive. The memory is leaked forever.

ASCII diagram of the cycle:

```
        a <--+
        |    |
        +--> b
        
Before destruction:
  a.refcount = 2 (local var + b->next)
  b.refcount = 2 (local var + a->next)
  
After dropping locals:
  a.refcount = 1 (b->next still holds it)
  b.refcount = 1 (a->next still holds it)
  
  Both are UNREACHABLE from outside, but ALIVE.
  MEMORY LEAK.
```

This is **The Cycle Problem**, and it is the reason reference counting alone is not a complete solution to memory management. Any real system that uses refcounts must handle cycles somehow.

### Why Cycles Matter

Cycles are not rare:

- **Parent–child data structures**: a tree node holds a `RefPtr` to its children, and each child holds a `RefPtr` back to its parent for "traversal up the tree."
- **Event listeners**: an object registers a listener with an event emitter. The listener holds a reference to the emitter (so it can unregister). The emitter holds a reference to the listener (so it can call it).
- **Caches**: a cache holds a reference to cached values. A cached value holds a reference to the cache (to invalidate itself).
- **Closures**: a lambda captures `this`, and `this` holds a reference to a shared_ptr of itself.

In practice, cycles appear in most non-trivial object graphs.

---

## 10.5 Fixing Cycles: Weak References

The standard solution is **weak references**. A weak reference does not increment the refcount.

```cpp
template <typename T>
class WeakPtr {
private:
    ControlBlock* block;
    T* object;
    
public:
    // Construct from a RefPtr: does NOT increment refcount
    WeakPtr(const RefPtr<T>& strong) : block(strong.block), object(strong.block->object) {}
    
    // Convert back to strong: increment refcount, or return null if freed
    RefPtr<T> lock() {
        if (block && block->refcount > 0) {
            block->refcount.fetch_add(1);
            return RefPtr<T>(object);
        }
        return RefPtr<T>(nullptr);
    }
};
```

A weak reference observes the object without keeping it alive. When the last *strong* reference is destroyed, the refcount hits zero and the object is freed. Weak references become dangling pointers — but you can ask "is it still alive?" via `lock()`.

Now fix the cycle:

```cpp
struct Node {
    RefPtr<Node> next;         // Strong reference to the next node
    WeakPtr<Node> prev;        // Weak reference back to the previous node
};

RefPtr<Node> a = new Node{};
RefPtr<Node> b = new Node{};

a->next = b;              // a.refcount = 1, b.refcount = 2
b->prev = a;              // a.refcount = 1 (weak doesn't increment), b.refcount = 2

a = nullptr;              // a.refcount goes from 1 to 0, a is freed
b = nullptr;              // b.refcount goes from 2 to 1... wait, a is already freed
```

With weak references, when `a` is dropped, its refcount becomes zero and it is deleted. `b->prev` becomes a dangling weak pointer (harmless). When `b` is dropped, its refcount goes from 2 to 1... no, it goes from 1 to 0, because `b->prev` does not hold a strong reference anymore. `b` is freed. No leak.

Swift uses this pattern extensively. `@strong` (the default) and `@weak` capture types in closures. Objective-C uses `strong` and `weak` properties.

**The rule**: in a reference-counted system, any cycle must have at least one weak reference to break it. The ownership must form a *DAG* (directed acyclic graph), not a general graph.

---

## 10.6 The Backup Plan: Cycle Detection

Weak references require programmer discipline. Forget to make one reference weak, and you have a leak.

CPython's approach is more conservative: **reference counting handles the common case, but a separate garbage collector handles cycles.**

Every so often (after every 700 object allocations, by default), CPython runs `gc.collect()`, which:

1. Traces all reachable objects (like a full mark-phase of a tracing GC).
2. Identifies objects that are unreachable but have nonzero refcounts (i.e., caught in cycles).
3. Breaks the cycles arbitrarily (by clearing references).
4. Allows the refcount mechanism to free them.

This is a hybrid approach:

- **Fast path**: most objects are freed immediately by the refcount decrement.
- **Slow path**: cycles are caught by a garbage collector that runs occasionally.

The advantage: you get deterministic destruction for acyclic objects *and* automatic cycle handling. The disadvantage: the GC pauses can be noticeable if you have many cycles, and the GC itself has overhead.

You can tune this in CPython:

```python
import gc
gc.disable()           # Turn off automatic collection
gc.collect()           # Manually run it
gc.set_threshold(10)   # Run every 10 allocations (more frequent)
```

---

## 10.7 Real Implementations

Let's see how the major reference-counting systems handle cycles.

### CPython (Reference Counting + Cycle Detector)

Every Python object has a refcount field:

```c
typedef struct {
    int refcount;
    PyTypeObject *type;
    // ... rest of object data
} PyObject;
```

When a `Py_DECREF(obj)` happens, the refcount is decremented. If it hits zero, the object is freed immediately. Cycles are caught by the garbage collector, which is a separate subsystem that runs every 700 allocations (tunable).

**Cost**: every Python operation that changes a pointer (assign, function parameter, return) increments/decrements refcounts. This is a fundamental source of CPython's slowness.

**Benefit**: objects are freed immediately when no longer referenced, leading to predictable behavior and small memory footprints.

### Swift (Reference Counting with Compiler Insertion)

Swift's refcount is not in the object itself (usually). Instead, the compiler inserts refcount operations at compile time:

```swift
var a: SomeClass? = SomeClass()   // Compiler emits: rc_increment(&a)
a = nil                            // Compiler emits: rc_decrement(&a)

func foo(x: SomeClass) {           // Compiler emits: rc_increment(x)
    ...
} // Compiler emits: rc_decrement(x)
```

The programmer writes high-level code; the compiler inserts the plumbing. This is automatic reference counting (ARC). Swift ARC also supports `weak` and `unowned` to break cycles:

```swift
class Node {
    var next: Node?                // Strong reference
    weak var prev: Node?           // Weak reference, does not increment
    unowned var parent: Node?      // Unowned (crashed if dangling, used for guaranteed ownership)
}
```

**Cost**: compile time increases (inserting all those operations). Runtime cost is the same as any refcount system — atomic increments/decrements.

**Benefit**: memory is freed immediately, and the language forces you to think about ownership.

### Objective-C (Reference Counting with Manual Annotations, Automatic with ARC)

Objective-C originally required manual refcounting:

```objc
// Manual
NSObject* obj = [[NSObject alloc] init];  // refcount = 1
[obj retain];                              // refcount = 2
[obj release];                             // refcount = 1
[obj release];                             // refcount = 0, freed
```

This was error-prone, so Apple added Automatic Reference Counting (ARC), which works like Swift — the compiler inserts the operations.

### C++ shared_ptr (Library-Level Reference Counting)

C++ has no built-in refcounting in the language (though many propose it). Instead, you use `std::shared_ptr`, a library template:

```cpp
std::shared_ptr<Node> a = std::make_shared<Node>();
std::shared_ptr<Node> b = a;  // refcount = 2

auto c = a;                   // refcount = 3
c = nullptr;                  // refcount = 2
```

`shared_ptr` is opt-in — you have to use it explicitly. This gives you control but is verbose. It is also slower than a language-level implementation because the refcount operations cannot be optimized as aggressively by the compiler.

To break cycles:

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;    // Weak reference breaks cycles
};
```

---

## 10.8 Atomic Costs: A Calculation

Let's quantify the cost of atomic refcounting in a real scenario.

Assume:
- Atomic increment/decrement: 20 cycles.
- Non-atomic increment/decrement: 1 cycle.
- L1 cache hit: 4 cycles.
- L3 cache hit: 40 cycles.
- Memory access: 200 cycles.

Scenario: tight loop that copies and destroys a shared_ptr 1 million times:

```cpp
for (int i = 0; i < 1000000; i++) {
    std::shared_ptr<Node> temp = node;  // Atomic increment (20 cycles)
    // ... use temp ...
}                                        // Atomic decrement (20 cycles)
```

Each iteration: 40 cycles. Total: 40 million cycles. On a 3 GHz CPU, that's 13 milliseconds.

Same loop with manual memory management:

```cpp
for (int i = 0; i < 1000000; i++) {
    Node* temp = node;  // No increment (copy is free)
    // ... use temp ...
}                        // No decrement (temp goes away)
```

Each iteration: ~1 cycle (pointer copy). Total: 1 million cycles. ~0.3 milliseconds.

**The refcount version is 40x slower.** This is why hot loops often use raw pointers or non-atomic refcounts (protected by locks elsewhere).

---

## 10.9 Reference Counting vs Tracing GC

Let's compare refcounting and tracing garbage collection (see Chapter 11 for a deep dive on GC).

### Refcounting Pros

- **Deterministic destruction**: destructors run immediately when the last pointer dies. Useful for RAII patterns (file handles closing, locks being released).
- **Low latency**: no full pauses. The system never has to stop the world and trace.
- **Memory locality**: objects are freed as soon as they're unreachable, keeping the heap compact.
- **Predictable peak memory**: you don't accumulate dead objects waiting for a collection.

### Refcounting Cons

- **Cycles**: requires weak references or a backup cycle collector.
- **Per-operation cost**: every pointer copy/destruction touches the refcount field. In multi-threaded code, atomic operations are expensive.
- **Cache line bouncing**: if two threads hammer on the same refcount, they fight for the cache line. Performance can be bad.
- **No optimization across refcount ops**: `a = b; b = c; a = nullptr;` does multiple increments/decrements where smarter code might do one.

### Tracing GC Pros

- **No cycles**: tracing GC handles cycles naturally.
- **Batch work**: all refcount operations happen in one pause, and the pause can be optimized heavily.
- **No per-operation cost**: pointers are just pointers. Copying a pointer is free.
- **Generational optimization**: young objects are GC'd often, old objects rarely.

### Tracing GC Cons

- **Pause latency**: the GC pauses the entire program while it traces.
- **Unpredictable destruction order**: a destructor might not run for milliseconds after the object becomes unreachable.
- **Memory overhead**: GC needs headroom to be efficient. Refcount systems can be leaner.
- **Harder to reason about**: "when is my resource freed?" is a harder question in GC.

### When to Use Each

**Use reference counting when:**
- You need deterministic destruction (file handles, locks, transaction cleanup).
- Objects are long-lived and mostly acyclic (e.g., scene graphs, ASTs).
- You are in a single-threaded or GIL-protected environment.
- You want predictable latency.

**Use tracing GC when:**
- Objects are highly interconnected and cycles are common.
- You don't care about the exact timing of destruction.
- Latency jitter from GC pauses is acceptable.
- You want the lowest per-operation cost.

Many real systems use a hybrid: reference counting for fast paths, GC for cycles. This is what CPython, Swift, and modern Java do.

---

## 10.10 Worked Example: The Cycle Leak and How to Fix It

Let's build a concrete example that demonstrates the cycle problem and the solutions.

### Version 1: The Leak

```cpp
#include <iostream>
#include <memory>

struct Node {
    int id;
    std::shared_ptr<Node> next;
    
    Node(int id) : id(id) {
        std::cout << "Node " << id << " created\n";
    }
    
    ~Node() {
        std::cout << "Node " << id << " destroyed\n";
    }
};

int main() {
    {
        std::shared_ptr<Node> a = std::make_shared<Node>(1);
        std::shared_ptr<Node> b = std::make_shared<Node>(2);
        
        a->next = b;  // a refcount = 1, b refcount = 2
        b->next = a;  // a refcount = 2, b refcount = 2
        
        // a and b go out of scope
        // a refcount: 2 -> 1 (local var dies), b refcount: 2 -> 1 (local var dies)
        // Both refcounts are 1. Neither is freed. LEAK.
    }
    std::cout << "End of scope. If no destructors printed, we have a leak.\n";
    return 0;
}
```

Output:
```
Node 1 created
Node 2 created
End of scope. If no destructors printed, we have a leak.
```

Nodes 1 and 2 were never destroyed. They are stuck in a cycle, unreachable but alive.

### Version 2: Weak Reference Fix

```cpp
struct Node {
    int id;
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;    // Weak reference breaks the cycle
    
    Node(int id) : id(id) {
        std::cout << "Node " << id << " created\n";
    }
    
    ~Node() {
        std::cout << "Node " << id << " destroyed\n";
    }
};

int main() {
    {
        std::shared_ptr<Node> a = std::make_shared<Node>(1);
        std::shared_ptr<Node> b = std::make_shared<Node>(2);
        
        a->next = b;  // a refcount = 1, b refcount = 2
        b->prev = a;  // a refcount = 1 (weak doesn't increment)
        
        // a goes out of scope: a refcount 1 -> 0, a is freed
        // b goes out of scope: b refcount 2 -> 1 (a->next is gone, so only the local var holds it)
        //                      then b refcount 1 -> 0, b is freed
    }
    std::cout << "End of scope.\n";
    return 0;
}
```

Output:
```
Node 1 created
Node 2 created
Node 1 destroyed
Node 2 destroyed
End of scope.
```

Both nodes are freed. No leak.

### Version 3: Automatic Cycle Detection (Simulating CPython's gc)

In C++, you'd typically just use weak_ptr. But here's what automatic cycle detection would look like in pseudocode:

```
Periodically (every N allocations):
  1. Scan all objects with refcount > 0.
  2. For each object, mark it as "potentially dead."
  3. For each object, traverse its strong pointers and unmark the objects they point to.
  4. Objects still marked are unreachable but have nonzero refcounts (cycles).
  5. Free them.
```

This is O(n) in the number of objects, which is expensive, but it catches cycles automatically without weak pointers. CPython does this.

---

## 10.11 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Manual refcounting (C-style)** | Explicit; cheap (no atomics); you see every increment/decrement | Error-prone (leak on missed decrement); tedious; leak-prone for cycles | Legacy code; embedded systems; when you need to understand every operation |
| **Automatic refcounting (Swift ARC, Obj-C ARC)** | Refcount operations inserted by compiler; deterministic destruction; fast | Cycles still need weak refs; atomic costs in multithreaded code; slower than raw pointers | Objects with clear ownership; RAII patterns; most modern code |
| **Reference counting + weak refs** | Deterministic, no cycles if you use weak for back-pointers | Requires programmer discipline (remembering to use weak); two pointer types to reason about | DAG-like data structures; parent–child relationships; event listeners |
| **Reference counting + cycle collector** | Deterministic for acyclic objects; cycles handled automatically | Periodic GC pauses (smaller than full GC, but still pauses); two memory management styles | CPython; systems where some jitter is acceptable |
| **Tracing GC only** | No cycles; no per-operation cost; simple conceptually | Unpredictable pauses; can't rely on timely destruction; higher memory overhead | Many objects; cycles common; latency jitter acceptable (servers, most user code) |
| **Borrow checker (Rust)** | Memory safety at compile time; no runtime cost; no cycles possible; no GC | Steep learning curve; some programs hard to express; slow compile times | Systems code; performance-critical code; code where safety is paramount |

---

## 10.12 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Reference counting is free." | Not free. Atomic increments are expensive (20+ cycles each). CPython pays this cost on every object creation/destruction. Tracing GC is often faster in aggregate. |
| "Reference counting prevents all leaks." | It prevents use-after-free (mostly), but not reference leaks. Cycles are reference leaks. CPython's cycle collector catches them, but they require special handling. |
| "weak_ptr is just for breaking cycles." | Weak pointers are useful whenever you want to observe an object without keeping it alive — caches, listeners, back-pointers. Cycles are just the most critical case. |
| "If an object has a refcount > 0, it's not being accessed." | Refcount > 0 means "someone holds a pointer to it." It doesn't mean "it's being used right now" or "it's reachable." Cycles can be refcount > 0 and unreachable. |
| "Garbage collection is slower than reference counting." | On modern hardware, tracing GC (esp. generational) is often *faster* in aggregate. The pause-time is higher, but the total CPU cost is lower because there's no per-operation overhead. |
| "Swift's ARC is faster than Python's refcount because it's 'automatic.'" | Swift's ARC is faster because the compiler can optimize away redundant increments/decrements. CPython's refcount system is fundamentally more expensive because it's runtime-based. |
| "All reference-counted systems require weak pointers." | No. CPython uses refcount + cycle collector. C++ shared_ptr can be used without weak_ptr if you ensure your object graphs are acyclic (e.g., trees). |

---

## 10.13 Exercises

1. **Trace a refcount scenario.** Write out the refcount of each object at each line of this code:

   ```cpp
   std::shared_ptr<Node> a = std::make_shared<Node>();  // a refcount = ?
   std::shared_ptr<Node> b = a;                          // a refcount = ?
   {
       std::shared_ptr<Node> c = b;                      // a refcount = ?
   }                                                      // a refcount = ?
   a = nullptr;                                          // a refcount = ?
   // What gets freed and when?
   ```

2. **Identify cycles in real code.** Find three places in a codebase you know where reference cycles might occur (parent–child structs, event listeners, closures, caches). For each, describe: (a) why the cycle exists, (b) which references should be weak to break it.

3. **Cost of atomics.** Measure the cost of atomic increments in a tight loop:
   
   ```cpp
   // Version 1: atomic increment
   std::atomic<int> counter(0);
   for (int i = 0; i < 100000000; i++) {
       counter.fetch_add(1, std::memory_order_release);
   }
   
   // Version 2: non-atomic increment
   int counter_na = 0;
   for (int i = 0; i < 100000000; i++) {
       counter_na++;
   }
   ```
   Time both versions. Calculate the ratio. How many cycles per atomic operation?

4. **Design a cycle detector.** Write pseudocode for a simple cycle detection algorithm that runs periodically: given a set of objects with refcounts > 0, identify which ones are unreachable and have nonzero refcounts. (Hint: mark-and-sweep, but adapted for refcount rather than reachability.)

5. **Weak pointer usage.** Refactor the cycle example from §10.10 using weak_ptr instead of manual cycle detection. Ensure destructors run when you drop the shared_ptrs.

6. **CPython vs C++ refcount cost.** Read the CPython source (Objects/object.c, the `Py_INCREF` / `Py_DECREF` macros). Compare the cost with C++ `std::shared_ptr`. What optimizations does C++ have that CPython lacks?

---

## 10.14 Summary

Reference counting is the simplest automatic memory management scheme: every object has a counter; increment on pointer copy, decrement on pointer destruction; free when it hits zero. It gives you **deterministic destruction** — the moment an object is no longer reachable, it dies — which is valuable for RAII patterns and resource cleanup.

The cost is paid on every operation. Atomic refcounts in multithreaded code are expensive (20+ cycles each). And refcounting has a fatal flaw: **cycles**. Two objects that point to each other leak. This is fixed by weak references (programmer discipline) or a backup cycle collector (CPython's approach). Many modern languages (Swift, Objective-C) use refcounting with compiler insertion and weak references. Others (Java, Go, Python-with-PyPy) use tracing GC instead.

The mental model: refcounting is pay-as-you-go; GC is batch-and-pause. The choice is not which is "correct" — both work — but which latency/throughput tradeoff fits your problem.

---

**[← Previous: Chapter 9 — Smart Pointers Internals](09-smart-pointers-internals.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 11 — Garbage Collection Internals →](11-garbage-collection-internals.md)**
