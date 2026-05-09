# Chapter 12 — How Python, Go, and PHP Actually Manage Memory

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain how CPython uses **reference counting** combined with a **cycle collector** to manage memory, and predict when objects are freed.
2. Describe Go's **concurrent tri-color mark-sweep garbage collector**, understand **escape analysis**, and predict whether an allocation goes on the stack or heap.
3. Understand how PHP's **request-lifetime memory model** and **reference counting** create a different memory profile from other languages.
4. Predict performance characteristics and memory leaks in each language, based on its underlying strategy.
5. Recognize the **implementation-specific gotchas** that leak through each language's abstraction (when the GC pauses, when cycles hold memory, when goroutines retain references).
6. Make informed choices about memory-sensitive operations in each language, avoiding patterns that exploit the runtime in pathological ways.

This chapter is the inverse of Chapter 11. There, we studied the theory of garbage collection and reference counting in the abstract. Here, we take three real runtimes apart and show what each one actually does when you run code. The goal is not to memorize facts about CPython's reference count field, but to build a model of each runtime that lets you **predict** its behavior: where the leak is, why the pause happened, why memory usage doubled.

---

## 12.1 The Setup: Why Runtime Choice Matters

From Chapter 10, you know that every language is a bundle of tradeoffs. But a language's tradeoff is not monolithic — it is carved into specific mechanisms that have specific costs.

Two Python programs, running on CPython vs PyPy, have the same syntax and semantics, but different memory characteristics:
- CPython: reference counting (predictable finalization, cost on every assignment)
- PyPy: generational GC (longer pause times, fewer atomic operations)

Two Go programs doing the same work will have different latency profiles if one is carefully tuned for escape analysis and the other allocates on the heap recklessly.

Two PHP applications will hit memory limits differently depending on whether they were designed for the request-lifetime model or whether they were ported from a long-lived service.

**The skill is not memorizing these mechanisms. The skill is building a mental model of each mechanism, then using that model to predict performance characteristics without having to profile everything.**

This chapter teaches those models.

---

## 12.2 CPython: Reference Counting + Cycle Collection

CPython is the reference implementation of Python (the one you probably use; the others are PyPy, Jython, IronPython, MicroPython). It uses a deceptively simple strategy: **every object has a reference count field (`ob_refcnt` in the C API), and when the count drops to zero, the object is freed immediately.**

Let's trace what this means.

### The Reference Count Field

Every Python object in CPython is a `PyObject*`, and the C struct looks roughly like this:

```c
// Simplified; actual CPython is more complex
typedef struct {
    Py_ssize_t ob_refcnt;      // The refcount
    PyTypeObject *ob_type;      // The type
    // ... rest of the object data
} PyObject;
```

When you do:

```python
x = [1, 2, 3]        # ob_refcnt = 1
y = x                # ob_refcnt = 2
z = x                # ob_refcnt = 3
del y                # ob_refcnt = 2
del z                # ob_refcnt = 1
del x                # ob_refcnt = 0 -> freed immediately
```

Each assignment bumps the count up; each deletion or scope exit bumps it down. When the count hits zero, the object's destructor runs immediately, and the memory is returned to Python's allocator.

**This means:** unlike in Java or Go, you know *exactly* when an object dies. It dies when the last reference to it disappears. This is sometimes called *deterministic finalization*, and it has real consequences.

### Consequence 1: Destructors Run Eagerly (Predictable Resource Cleanup)

If an object holds a file handle, a database connection, or a lock, the `__del__` destructor can release it immediately:

```python
def process_file(path):
    f = open(path)          # ob_refcnt = 1
    data = f.read()
    return data             # f's refcount -> 0, __del__ runs, file closes immediately
```

In Java or Go, the file might stay open for milliseconds until the GC runs. In CPython, it's closed right now. This is why CPython's context manager (`with`) is less critical than in other languages — even without it, the file is closed when the variable goes out of scope.

(This is also why some engineers prefer CPython for systems programming: you can reason about resource lifetimes in a way you cannot in full-GC languages.)

### Consequence 2: Refcount Operations Are Atomic Costs

Every assignment is *not free*. It is O(1), but it has real cost: an `Py_INCREF` (increment refcount) and possibly a `Py_DECREF` (decrement and maybe free) at scope exit. In a tight inner loop, this cost adds up.

In contrast, a GC language amortizes this cost across the entire GC pause: you pay it once per garbage collection, not once per assignment.

This is one reason PyPy and Jython are sometimes faster on long-running code: they replace refcounting with generational GC, which has lower per-operation cost.

### The Cycle Collector: The Refcount's Blind Spot

Refcounting has one fatal flaw: **it cannot detect cycles**.

```python
a = []
b = []
a.append(b)
b.append(a)       # a -> b -> a, a cycle

del a
del b             # Refcounts are 1 each (a and b point to each other)
                  # But no external reference exists!
                  # The objects are unreachable.
                  # Yet refcounting cannot see this.
```

When `a` and `b` are deleted, their refcounts drop from 2 to 1. They are unreachable from the outside, but neither can be freed because each is referenced by the other.

CPython solves this with a **cycle collector**, a separate GC that runs periodically (usually when the number of allocated objects crosses a threshold). The cycle collector finds these cycles and breaks them.

```python
import gc
gc.collect()      # Run the cycle collector manually (blocks)
                  # Returns the number of objects freed
```

**This means:** CPython is not "just" reference counting. It is reference counting + a periodic mark-sweep-style GC for cycles. You get deterministic finalization most of the time, but occasionally the program pauses while the cycle collector runs.

### What This Costs in Practice

**Advantages of CPython's approach:**

1. **Predictable finalization**: resource cleanup happens immediately. Files, sockets, locks close when no longer referenced.
2. **Small per-object overhead**: just one integer per object.
3. **Fast typical case**: most objects die with no GC pause.
4. **Debuggable**: a reference cycle is a concrete, findable bug in your code, not a hidden GC artifact.

**Disadvantages:**

1. **Per-operation overhead**: every assignment has atomic cost.
2. **Stop-the-world pause**: the cycle collector pauses the entire program when it runs.
3. **Memory overhead**: you need a bit of slack in your refcount (since it's an integer, not unbounded).
4. **Pathological behavior**: if you create cycles faster than the cycle collector runs, memory grows unbounded until collection catches up.
5. **C extensions can break it**: native code that forgets `Py_INCREF` or `Py_DECREF` can corrupt the refcount, leading to crashes or memory leaks.

### The Global Interpreter Lock (GIL)

CPython uses a **Global Interpreter Lock** (GIL) to make refcounting safe in the presence of threads. Every Python bytecode operation that touches an object must acquire the GIL. This means:

- Only one OS thread can run Python bytecode at a time.
- Threads on separate cores *cannot* parallelize CPU-bound work.
- But I/O-bound operations (system calls that release the GIL) can run in parallel.

The GIL exists because refcounting is not atomic. If thread A increments a refcount while thread B decrements it, and both happen simultaneously on different cores, the result is undefined. The GIL prevents this by ensuring only one thread touches refcounts at a time.

(This is a common point of confusion: PyPy and Jython do not have a GIL because they use generational GC instead of refcounting. Go and Java do not have a GIL because their memory model is built on atomic operations and barriers from the start.)

---

## 12.3 Go: Concurrent Tricolor Mark-Sweep, Escape Analysis, and Segmented Stacks

Go's memory model is almost entirely opposite to CPython's. There is no refcounting, no destructor guarantee, and no explicit memory management. Instead, Go uses **concurrent tricolor mark-sweep garbage collection** with an explicit design goal of keeping pause times below 1 millisecond.

### The Escape Analysis Compiler Pass

Before the runtime even starts, Go's compiler runs **escape analysis**: for each allocation, it decides whether the object's lifetime is bounded by a single function's scope, or whether it might escape to the heap.

```go
func localBuffer() {
    buf := [256]byte{}      // Stack allocation: escapes=false
    _ = buf
}

func heapBuffer() {
    buf := make([]byte, 256) // Heap allocation: escapes=true
    return buf               // Returned, must be on heap
}

func hiddenEscape() {
    buf := make([]byte, 256)
    f := func() { _ = buf }  // buf captured by closure
    go f()                   // Run in goroutine: must be on heap
    return                   // buf escapes implicitly
}
```

The compiler builds a **flow-sensitive call graph** and marks which allocations escape. Stack allocations are free: they are deallocated when the function returns, no GC interaction needed.

Heap allocations are tracked by the GC.

This is Go's answer to C++'s RAII: you don't manage memory explicitly, but the compiler is smart enough to put short-lived objects on the stack anyway, so the GC only touches long-lived data.

### The Mark-Sweep Algorithm

Go's GC runs concurrently with user code (unlike CPython's cycle collector, which stops everything). It uses **tricolor marking**:

1. **White**: objects not yet scanned.
2. **Gray**: objects that have been marked reachable, but their children have not been scanned yet.
3. **Black**: objects that have been scanned, and all their children are reachable.

The collector scans from roots (stack, static data), marking objects gray, then scans gray objects to mark their children. Objects still white at the end are unreachable and are swept.

The key insight: **the GC runs mostly in parallel with user threads**, using **write barriers** (checks on pointer assignments) to track mutations during concurrent marking.

### The Pause Time Target

Go's GC is tuned to keep pause times under 1 millisecond. This is dramatically lower than a typical GC in Java (10-100ms) or Python (10-50ms on larger heaps).

How? By doing most work concurrently and by aggressively tuning the heap-size-to-object-count ratio. Go deliberately uses about 2× the live memory (the "GC overhead") to keep scans fast.

```go
// Control the GC trigger
debug.SetGCPercent(100)  // Default: trigger GC when heap is 100% larger than live set
debug.SetGCPercent(50)   // Trigger more often: lower pause time, higher CPU cost
```

### Goroutine Stacks: Segmented, Growable

Go's goroutines do not use OS threads directly. Instead, they run on a pool of OS threads, and their **stacks are managed by the runtime**, growing and shrinking as needed.

When a goroutine's stack overflows:

1. The runtime allocates a larger chunk of memory.
2. The runtime copies the goroutine's stack to the new chunk.
3. All local pointers (frame pointers, instruction pointers) are updated.
4. The old chunk is deallocated.

This is cheap because stacks are small (start at ~2KB) and stack-to-heap copying is rare. It also means goroutines can be extremely lightweight: you can spawn millions of them.

**One consequence:** goroutines can hold references to long-lived objects (globals, channels, other goroutines), and those objects stay on the heap until the goroutine exits. If you spawn unbounded goroutines without cleanup, you can leak memory.

```go
// Leak: goroutine never exits, holds reference to large buffer
func leakyServer(listener net.Listener) {
    for {
        conn, _ := listener.Accept()
        go func(c net.Conn) {
            // Bug: this goroutine runs forever because the loop never breaks
            for {
                buf := make([]byte, 1024*1024)  // 1MB, allocated per iteration
                c.Read(buf)
            }
        }(conn)
    }
}
```

A malformed goroutine that never exits can accumulate memory indefinitely because the runtime cannot garbage-collect its objects until the goroutine is done.

### What This Costs in Practice

**Advantages of Go's approach:**

1. **No manual memory management**: no `new`, `delete`, or lifetime bugs.
2. **Predictable CPU cost**: the compiler makes escape analysis decisions statically, so memory behavior is repeatable.
3. **Minimal pause times**: sub-millisecond GC pauses are acceptable even for latency-sensitive applications.
4. **Escape analysis catches optimizations**: short-lived allocations are fast; long-lived ones are GC-tracked.
5. **Massive concurrency is cheap**: millions of goroutines are feasible because stacks are small and allocation is efficient.

**Disadvantages:**

1. **No deterministic finalization**: you cannot rely on objects being freed at a specific time. If a resource (file, connection, lock) must be cleaned up, use `defer` or context cancellation explicitly.
2. **Higher baseline memory use**: escape analysis is conservative, so some objects that could be stack-allocated are heaped anyway. Plus the GC target of 2× memory overhead.
3. **Goroutine leaks are real**: if a goroutine never exits, its referenced objects stay on the heap forever.
4. **Compile-time optimization**: if the compiler's escape analysis thinks an allocation escapes (conservatively), it will be heaped even if it doesn't. You can't always argue with the compiler.

---

## 12.4 PHP / Laravel: Request Lifetimes and Copy-on-Write

PHP's memory model is radically different from both CPython and Go. PHP is designed for **short-lived request processing**: each web request runs in an isolated process (or a long-lived worker that processes requests sequentially), and at the end of the request, *the entire memory arena is freed*.

### The Request Lifetime Model

In a traditional PHP application:

1. A web request arrives.
2. The PHP interpreter starts, allocates memory.
3. The application processes the request, allocating objects, arrays, strings, database connections.
4. The response is sent.
5. All memory is freed. The process is recycled or reused for the next request.

This model has a profound consequence: **memory leaks are bounded by request duration**. If your application leaks 100KB per request, but processes 100 requests per second, it will use 10MB per second — until the process is recycled (usually after 10,000 requests or 1 hour), at which point all memory is freed.

This is why a PHP application can run indefinitely with a slow memory leak that would crash a long-lived Python service.

### Reference Counting (Again)

Like CPython, PHP uses **reference counting** for object lifetime. But the cycle collector in PHP is less aggressive: it typically runs once per request, after request processing.

```php
<?php
$a = new stdClass();
$b = new stdClass();
$a->ref = $b;
$b->ref = $a;      // Cycle

unset($a);
unset($b);         // Refcounts are 1 each, not freed yet
                   // But at end of request, all memory freed anyway
?>
```

This means cycles in PHP are often *not* a problem: the request ends, and all memory is freed regardless.

### Copy-on-Write and Shared Memory Segments

Modern PHP (via shared hosting or opcode caching) uses **copy-on-write** semantics to allow multiple PHP processes to share compiled bytecode:

1. Compiled PHP opcodes are stored in a shared memory segment (APC, OPcache).
2. Multiple PHP processes can read from this shared memory.
3. When a process modifies a variable, the OS copies the page to the process's private memory.

This allows a web hosting provider to run 100 PHP processes on a single machine, where each process consumes only the delta of modified memory, not the full bytecode.

The implication: **large arrays passed by value are not as expensive as they seem**. In some PHP versions, the array is not copied until it is modified. Before that, it is shared with copy-on-write semantics.

```php
<?php
function processArray($arr) {   // $arr is an alias or copy-on-write reference
    // If we read only, $arr is not copied.
    // If we modify $arr, the copy happens.
    $arr[0] = 'modified';       // Now we have our own copy
}

$large = array_fill(0, 1000000, 'value');
processArray($large);           // Passes $large by reference or CoW; cheap
?>
```

### Opcache: Persistent Compiled Code

The **Opcache** extension (enabled by default in modern PHP) compiles PHP source to opcodes once, then caches the opcodes in memory. On subsequent requests, the cached opcodes are reused, skipping the parse/compile phase.

This has implications:

1. **Static data persists**: global variables, class static properties, and function-level statics can retain values across requests (if configured).
2. **Memory usage is stable**: the most expensive part of PHP — parsing and compilation — happens once, then the same process handles many requests.

```php
<?php
static $cache = null;           // Static variable
if ($cache === null) {
    $cache = expensiveCompute(); // Computed once per process, reused
}
?>
```

### What This Costs in Practice

**Advantages of PHP's approach:**

1. **Bounded memory leaks**: slow leaks do not accumulate indefinitely; they are reset per request.
2. **Shared opcodes**: multiple processes share compiled code via copy-on-write.
3. **Minimal GC overhead**: the reference collector runs infrequently; most memory is freed at request end.
4. **Simple programming model**: developers rarely think about lifetime; objects exist for the duration of the request, then vanish.

**Disadvantages:**

1. **No long-lived state without explicit management**: you cannot easily cache data across requests without falling back to external state (Redis, memcached, databases).
2. **Request startup cost**: each process/request startup includes some initialization and memory allocation.
3. **Scaling: processes, not threads**: PHP typically uses process-per-request or worker pools, not shared threads. This has higher memory cost and limits connection pooling.
4. **Static variable semantics**: static data persists across requests, which is useful but also a footgun — thread-like issues can appear if two requests interact via shared static memory.

---

## 12.5 A Cross-Language Memory Comparison

Now let's put them side by side:

| Characteristic | CPython | Go | PHP/Laravel |
|---|---|---|---|
| **Core strategy** | Reference counting + cycle collector | Concurrent tri-color mark-sweep | Reference counting + request lifetime |
| **Deterministic finalization** | Yes: `__del__` on refcount zero | No: object freed whenever GC runs | No: freed at request end |
| **Per-operation cost** | High: every assignment bumps refcount | Low: most objects on stack | Very low: request-scoped |
| **GC pause time** | 50–500ms (cycle collector) | <1ms (concurrent) | Minimal (per-request reset) |
| **Memory overhead** | 1× live set (refcount field) | ~2× live set (GC target) | ~1× live set per request |
| **Cycles** | Must be detected and broken | No cycles (heap objects are DAGs) | Yes, but freed at request end |
| **Leaks** | Can happen from cycles or misuse | Can happen from goroutines holding references | Bounded per request, reset per worker restart |
| **Scalability** | 1–10K concurrent objects typical | 10–100M goroutines feasible | 100–1K requests per process typical |
| **Best for** | Scripts, notebooks, systems tools | Services, networking, high concurrency | Web requests, CRUD, high throughput |

---

## 12.6 The Leak Signatures: How Each Language Fails

Each memory model has a characteristic failure mode. Learning to recognize it is half the battle.

### Python: The Cycle Leak

Symptoms:
- Memory grows slowly but persistently.
- `gc.collect()` shrinks it briefly, then it grows again.
- Profilers show no single object consuming all memory.

Root cause:
- A cycle of objects (A → B → A) that is created and never broken.
- The cycle collector's run interval is too long, or the threshold is too high.

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

def create_cycle():
    a = Node(1)
    b = Node(2)
    a.next = b
    b.next = a       # Cycle
    return a         # a leaks until GC runs

# If called frequently, cycles accumulate:
for i in range(1000000):
    create_cycle()   # 2M objects in cycles, unreachable but not freed
    if i % 100000 == 0:
        print(f"Iteration {i}: {len(gc.get_objects())} objects")
```

Fix:
```python
# Option 1: Break the cycle explicitly
def create_cycle():
    a = Node(1)
    b = Node(2)
    a.next = b
    b.next = a
    a.next = None    # Break before return
    return a

# Option 2: Use weakref to avoid the cycle
import weakref

class Node:
    def __init__(self, data):
        self.data = data
        self.next = None
        self.prev = weakref.ref(None)

# Option 3: Run GC more aggressively
gc.set_debug(gc.DEBUG_LEAK)
gc.set_threshold(1000, 10, 10)  # More frequent collection
```

### Go: The Goroutine Leak

Symptoms:
- Memory grows over time, tracking the number of requests handled.
- `runtime.NumGoroutine()` increases indefinitely.
- The GC pause time is stable, but the heap grows.

Root cause:
- A goroutine that never exits, holding references to request-scoped objects.
- Common in HTTP handlers, message processors, or concurrency primitives.

```go
// Leak: goroutine never exits
func handleRequest(w http.ResponseWriter, r *http.Request) {
    data := make([]byte, 1024*1024)  // 1MB allocation
    go func() {
        // This goroutine runs forever, never exits
        for {
            _ = data   // Reference keeps data on heap
            time.Sleep(100 * time.Millisecond)
        }
    }()
    w.WriteHeader(http.StatusOK)
}

// Leak: channel never read, sender blocked
func leakyChannel() {
    results := make(chan int, 10)
    go func() {
        for i := 0; i < 100; i++ {
            results <- i  // Goroutine blocks after filling the buffer
        }
    }()
    _ = results
    return          // Goroutine blocked forever, holding memory
}
```

Fix:
```go
// Use context to signal goroutine to exit
func handleRequest(w http.ResponseWriter, r *http.Request, ctx context.Context) {
    data := make([]byte, 1024*1024)
    go func() {
        for {
            select {
            case <-ctx.Done():
                return              // Exit when request is done
            default:
                _ = data
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()
    w.WriteHeader(http.StatusOK)
}

// Ensure channels are drained
func properChannel() {
    results := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            results <- i
        }
        close(results)              // Signal goroutine is done
    }()
    for v := range results {        // Drain the channel
        _ = v
    }
}
```

### PHP: The Shared State Leak

Symptoms:
- One request's state affects another request.
- Static variables contain data from previous requests.
- Memory spikes on specific request patterns.

Root cause:
- Static variables (class statics, function statics) persisting across requests.
- Shared memory segments not cleaned up between requests.

```php
<?php
// Shared state leak
class RequestTracker {
    static $allRequests = [];  // Never cleared

    static function trackRequest($id) {
        self::$allRequests[] = $id;  // Accumulates forever
    }
}

RequestTracker::trackRequest($_REQUEST['id']);
echo "Tracked " . count(RequestTracker::$allRequests) . " requests";
// Output after 1M requests: "Tracked 1000000 requests" (unbounded)
?>
```

Fix:
```php
<?php
// Option 1: Clear static state at request end
class RequestTracker {
    static $allRequests = [];

    static function trackRequest($id) {
        self::$allRequests[] = $id;
    }

    static function reset() {
        self::$allRequests = [];   // Clear at end of request
    }
}

// In app middleware or bootstrap:
register_shutdown_function([RequestTracker::class, 'reset']);

// Option 2: Use external storage (cache) instead of statics
$redis = new Redis();
$redis->lpush('requests', $_REQUEST['id']);  // Unbounded, but explicit
?>
```

---

## 12.7 Worked Example: Processing 1 Million Records

Let's process the same task in all three languages and observe memory behavior.

**Task**: Read 1 million JSON records from disk, extract a field, and compute the sum.

### Python (CPython)

```python
import json
import gc
import sys

def process_python():
    total = 0
    with open('records.json') as f:
        for line in f:
            record = json.loads(line)
            total += record['value']
    return total

if __name__ == '__main__':
    gc.collect()
    total = process_python()
    gc.collect()
    print(f"Total: {total}")
    print(f"Objects: {len(gc.get_objects())}")
```

**Memory profile:**
- Startup: ~10 MB (interpreter, libraries)
- Per record: ~1 KB (dict, int, float)
- Peak: ~1 GB (1M objects × 1KB)
- Cleanup: ~10 MB (GC collection frees everything)
- GC pauses: ~50–200ms when cycle collector runs

**Why:**
- Each `record` dict is a refcounted object.
- The `total` int is also refcounted.
- No cycles, so cycle collector does minimal work.
- Memory is freed as records are processed and released.

### Go

```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "os"
    "runtime"
)

func processGo() (int64, error) {
    var total int64
    f, _ := os.Open("records.json")
    defer f.Close()

    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
        var record struct {
            Value int64 `json:"value"`
        }
        json.Unmarshal(scanner.Bytes(), &record)
        total += record.Value
    }
    return total, scanner.Err()
}

func main() {
    runtime.GC()  // Force GC before
    total, _ := processGo()
    runtime.GC()  // Force GC after
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Total: %d\n", total)
    fmt.Printf("Alloc: %v MB, TotalAlloc: %v MB\n", m.Alloc/1024/1024, m.TotalAlloc/1024/1024)
}
```

**Memory profile:**
- Startup: ~5 MB (Go runtime)
- Per record: ~100 bytes (unmarshaled struct, buffer)
- Peak: ~50 MB (much lower than Python)
- Cleanup: ~5 MB
- GC pauses: <1ms, possibly zero if all records fit in stack buffers

**Why:**
- Escape analysis determines that `record` fits in the function frame (stack).
- The `scanner.Bytes()` buffer is reused across iterations.
- No object allocations per record; only stack frames.
- GC barely runs because there are few heap objects.

### PHP

```php
<?php
$total = 0;
$file = fopen('records.json', 'r');
while ($line = fgets($file)) {
    $record = json_decode(trim($line), true);
    $total += $record['value'];
}
fclose($file);
echo "Total: $total\n";
?>
```

**Memory profile:**
- Startup: ~2 MB (PHP runtime)
- Per record: ~500 bytes (array, strings)
- Peak: ~500 MB (depends on PHP process memory limit)
- Cleanup: All freed at end of request
- GC pauses: Negligible (inline cycle detection)

**Why:**
- PHP parses one line per iteration, decodes the JSON, extracts the value, then releases the array.
- The `$total` int is reused.
- No persistent objects; memory is freed inline.
- Request ends, entire process memory is freed.

### Comparison

| Language | Peak Memory | GC Pauses | Per-Record Cost | Scalability |
|---|---|---|---|---|
| Python | ~1 GB | 50–200ms | High | Single-threaded, millions of objects |
| Go | ~50 MB | <1ms | Very low | Multi-goroutine, billions of objects |
| PHP | ~500 MB | None | Medium | Request-scoped, many processes |

**Key insight**: Go's escape analysis and stack allocation make the difference. The same algorithmic work costs Go 1/20th the memory of Python because most objects never touch the heap.

---

## 12.8 Common Performance Mistakes Per Language

### Python

**Mistake 1: Holding references in module-level lists**

```python
# Wrong: cache accumulates forever
_cache = []

def process(item):
    _cache.append(item)  # Grows unbounded
    ...
```

Fix: Explicitly clear or use bounded caches.

```python
from functools import lru_cache

@lru_cache(maxsize=1000)  # Bounded by size
def process(item):
    ...
```

**Mistake 2: Creating cycles with `__del__`**

```python
# Wrong: __del__ creates a cycle
class Circular:
    def __init__(self):
        self.ref = self  # Self-reference
    def __del__(self):
        print("deleted")

a = Circular()
del a              # __del__ is not called until GC
```

Fix: Avoid cycles; use context managers instead.

```python
class Proper:
    def __enter__(self):
        return self
    def __exit__(self, *args):
        print("cleaned up")

with Proper() as p:
    ...            # Cleanup guaranteed at scope end
```

**Mistake 3: Passing large objects to threads without copying**

```python
# Wrong: GIL contention
data = [0] * 10000000
def worker():
    for x in data:  # Thread acquires GIL to read data
        ...

threads = [threading.Thread(target=worker) for _ in range(4)]
[t.start() for t in threads]  # All threads serialize on GIL
```

Fix: Use multiprocessing for CPU-bound work; use asyncio for I/O-bound.

```python
import multiprocessing

def worker(data):
    return sum(data)

if __name__ == '__main__':
    data = [0] * 10000000
    with multiprocessing.Pool(4) as pool:
        results = pool.map(worker, [data] * 4)  # Separate processes
```

### Go

**Mistake 1: Leaking goroutines by not canceling context**

```go
// Wrong: goroutine never exits
func serve(listener net.Listener) {
    for {
        conn, _ := listener.Accept()
        go func(c net.Conn) {
            io.Copy(io.Discard, c)  // Blocks forever if client doesn't close
        }(conn)
    }
}
```

Fix: Use context and timeouts.

```go
func serve(listener net.Listener, ctx context.Context) {
    for {
        conn, _ := listener.Accept()
        go func(c net.Conn) {
            // Timeout or context cancellation ensures goroutine exits
            c.SetReadDeadline(time.Now().Add(5 * time.Second))
            io.Copy(io.Discard, c)
        }(conn)
    }
}
```

**Mistake 2: Unbounded allocation due to escape analysis being conservative**

```go
// Wrong: escape analysis pessimistically heaps large arrays
func process() {
    large := [1000000]int{}  // Compiler thinks "this might escape"
    // ... use large
}
```

Fix: Profile and use escape analysis diagnostics.

```go
// go build -gcflags="-m" shows escape analysis decisions
// For large arrays, be explicit about stack vs heap:

func processSmall() {
    large := [1000000]int{}  // On stack
    _ = large
}

func processLarge() {
    large := make([]int, 1000000)  // Explicitly heap-allocated
    _ = large
}
```

**Mistake 3: Creating millions of goroutines without limits**

```go
// Wrong: unbounded goroutine spawning
func handleRequests(listener net.Listener) {
    for {
        conn, _ := listener.Accept()
        go handleConnection(conn)  // 1 goroutine per connection
                                    // On a busy server: millions of goroutines
    }
}
```

Fix: Use a pool or semaphore to limit concurrency.

```go
// Option 1: Worker pool
func handleRequests(listener net.Listener) {
    sem := make(chan struct{}, 1000)  // Max 1000 concurrent
    for {
        conn, _ := listener.Accept()
        go func(c net.Conn) {
            sem <- struct{}{}
            defer func() { <-sem }()
            handleConnection(c)
        }(conn)
    }
}

// Option 2: Bounded queue
func handleRequests(listener net.Listener) {
    queue := make(chan net.Conn, 100)
    for i := 0; i < 10; i++ {
        go worker(queue)
    }
    for {
        conn, _ := listener.Accept()
        queue <- conn
    }
}
```

### PHP/Laravel

**Mistake 1: Large arrays passed by value without realizing the copy**

```php
// Expensive, but only if modified
function processArray($arr) {
    $arr[0] = 'modified';  // Now array is copied
    // ... expensive work
}

$large = array_fill(0, 1000000, 'value');
processArray($large);  // Array is copied on write
```

Fix: Pass by reference if you need to modify.

```php
function processArray(&$arr) {  // Reference, not copy
    $arr[0] = 'modified';
}
```

**Mistake 2: Accumulating data in static variables**

```php
// Wrong: static accumulates across requests
static $log = [];

function logEvent($event) {
    global $log;  // Or static $log in a class
    $log[] = $event;  // Grows every request
}
```

Fix: Use external storage or reset explicitly.

```php
// Store in Redis or cache
$cache = app('cache');
$cache->push('log', $event);  // Unbounded, but explicit and resettable

// Or reset at end of request
register_shutdown_function(function() {
    $GLOBALS['log'] = [];  // Clear
});
```

**Mistake 3: Not cleaning up resources in opcache-enabled environments**

```php
// Wrong: PDO connection persists across requests (if assigned to static)
static $pdo = null;

if ($pdo === null) {
    $pdo = new PDO('mysql:...');
    // This connection persists for the lifetime of the process!
    // If the database goes away, future requests get stale connections.
}
```

Fix: Reconnect on each request, or use connection pooling.

```php
// Reconnect each request
$pdo = new PDO('mysql:...');  // New connection per request
// ... use ...
unset($pdo);  // Explicit close

// Or use connection pooling library (implicit reconnect)
```

---

## 12.9 Tradeoffs: Memory Strategy vs Performance, Predictability, and Simplicity

| Strategy | Control | Pause Time | Per-Op Cost | Leak Risk | Simplicity |
|---|---|---|---|---|---|
| Manual (C, C++) | Maximum | None | Very low | Highest | Low |
| Reference counting (CPython, Swift) | Moderate | Moderate | High | Cycles | Moderate |
| Mark-sweep (Go, Java) | Low | Moderate | Low | Goroutines/threads | High |
| Request-lifetime (PHP) | Low | None | Very low | Bounded | Very high |
| Generational (PyPy, V8, JVM) | Low | Low | Very low | Rare | High |

---

## 12.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Garbage collection eliminates memory leaks." | GC eliminates use-after-free and double-free. It does not eliminate leaks (cycles in Python, goroutine leaks in Go, static data in PHP). |
| "Python is slow because it's interpreted." | Python is slow because of dispatch cost, allocation cost, and atomic refcount operations. The interpreter is a small part of it. PyPy (JIT'd) is 5–10× faster on CPU-bound code. |
| "Go is fast because it's compiled to native code." | Go is fast because escape analysis puts most allocations on the stack, reducing GC pressure. The compilation is a secondary factor. |
| "PHP is slow because it starts from zero every request." | Modern PHP with opcache is fast because bytecode is cached. The request-lifetime model is actually an advantage for throughput (no long-lived state to manage). |
| "Garbage collectors make programs unpredictable." | A well-tuned GC (like Go's) has predictable pause times. The unpredictability comes from poor tuning, not the existence of GC. |
| "Reference counting is simpler than mark-sweep." | Reference counting has the complexity of cycle detection, atomic operations, and determining when an object truly becomes unreachable. Mark-sweep is simpler in principle. |

---

## 12.11 Exercises

1. **Memory leak diagnosis.** You have a long-running Python service that grows memory linearly over 24 hours. Describe: (a) how you would detect whether the leak is due to a reference cycle or a forgotten reference, (b) what diagnostic tools you would use (`gc.get_objects()`, `objgraph`, profilers), (c) the fix for each case.

2. **Escape analysis in Go.** Write a small Go program that creates a large array in a function. Use `go build -gcflags="-m"` to see the escape analysis output. Then intentionally make it escape by returning it or capturing it in a closure. Observe how the compiler's decision changes.

3. **Goroutine leak detection.** Write a Go program that spawns 100 goroutines that each spawn 100 more goroutines, without a way to stop them. Use `runtime.NumGoroutine()` to verify the leak. Then fix it using context cancellation.

4. **Request-lifetime model in PHP.** Write a PHP script that accumulates data in a static variable across mock "requests" (loop iterations). Measure memory growth. Then add explicit cleanup and verify the difference.

5. **Cross-language memory profile.** Write the same algorithm in Python, Go, and PHP (or just two of them). Process a large dataset and record peak memory, runtime, and GC pauses. Explain the differences based on the memory strategies.

6. **Conceptual.** A colleague says "let's rewrite our Python service in Go to save memory." Based on this chapter, what questions would you ask about the current bottleneck before agreeing?

---

## 12.12 Summary

Every "memory-managed" language is actually a specific engineering choice about **when** and **how** memory is freed, and what the programmer has to think about. CPython's refcounting is predictable but costly; Go's concurrent GC is fast and low-latency but requires thinking about goroutine lifetimes; PHP's request-lifetime model is simple but only viable for request-scoped work.

Understanding these models — refcounting vs mark-sweep, escape analysis vs whole-heap GC, process-per-request vs long-lived services — is the difference between debugging memory issues and staring at a profile without understanding it. The mechanisms are different, but the principle is the same: **know the cost model, predict the failure mode, and avoid the patterns that trigger pathological behavior**.

---

**[← Previous: Chapter 11 — Garbage Collection Internals](11-garbage-collection-internals.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 13 — Smart Pointers & RAII](13-smart-pointers-raii.md)** *(coming soon)*
