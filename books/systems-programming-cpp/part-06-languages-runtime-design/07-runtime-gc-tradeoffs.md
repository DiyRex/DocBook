# Chapter 64 — Runtime GC Tradeoffs Across Languages

Every garbage-collected runtime makes the same fundamental decision: which point in the tradeoff space between latency, throughput, and complexity to occupy. This decision, made once at runtime design time, shapes the behavior of every program that runs on that runtime. You cannot escape it. You can only understand it well enough to write code that cooperates with your GC rather than fighting it.

This chapter is not about garbage collection algorithms. Chapter 11 covered those—mark-sweep, copying, generational, tri-color marking, write barriers. This chapter asks a different question: given that you have those tools, *which did you choose, and why?* Six production runtimes made six different choices, each reflecting a bet about what matters to the programs running on them.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Identify the three dominant axes of GC design—pause time, throughput, and programmer effort—and understand that no runtime optimizes all three.
2. Compare the GC strategies of six major runtimes (JVM, Go, V8, .NET CoreCLR, CPython, Erlang BEAM) and map each to a position in the tradeoff space.
3. Explain why allocation rate matters more than heap size for real-world performance.
4. Predict the memory overhead and pause patterns of code running on each runtime.
5. Recognize when your code is fighting the GC and apply language-specific techniques to cooperate with it.

---

## The Three Axes

Every GC runtime optimizes for some combination of three goals, each in tension with the others:

**Axis 1: Pause Time vs. Throughput**

A stop-the-world GC pauses the application to mark and sweep. The longer it runs, the longer the pause. But if you make pauses shorter, you have to run GC more frequently—which wastes CPU on bookkeeping. A 100 millisecond pause every 10 seconds (10% GC overhead) uses the same CPU as ten 10 millisecond pauses (10% overhead), but the application feels different: the 100ms pause is a "stutter," while frequent small pauses are distributed.

**Axis 2: Programmer Effort vs. Runtime Cost**

Explicit memory management (C++) imposes burden on the programmer: you must allocate and deallocate, and get it right. Automatic GC eliminates that burden but costs CPU, memory, and complexity. Some runtimes (CPython) add reference counting, shifting some work to the runtime at every assignment. Others (Erlang) make GC per-process, shifting complexity to the scheduler.

**Axis 3: Predictability vs. Flexibility**

A GC optimized for predictable pause times (Go, Erlang) must sacrifice flexibility: it cannot compact the heap (too expensive), cannot fully exploit multi-core (too much coordination), cannot grow the heap to peak demand without cost. A GC optimized for flexibility (JVM G1) accepts unpredictable pause times and requires careful tuning.

### Diagram: The Tradeoff Space

```
                      Low Pause
                         ▲
                         │
    Erlang BEAM          │  Go
         •               │  •     ZGC
         │               │  │      •
         │               │  │      │
         └───────────────┼──┼──────┼─────────► High Throughput
                         │  │      │
      CPython ───────────┤  │      └─ .NET
         •               │  │         • CoreCLR
                         │  │
         ────────────────┼──┼─────────
                   V8 •  │  │
                         │  │ JVM G1
                    (High GC effort)
```

The diagram is a simplification. Every runtime is actually a multi-dimensional point. But the pattern holds: no runtime dominates all three axes.

---

## The JVM: Choosing Throughput with Knobs for Latency

The JVM (HotSpot) prioritizes **throughput by default**, but gives operators knobs to tune pause time. This reflects the JVM's origin: a platform for long-running server applications that process millions of requests, not latency-critical systems.

### The Default: Generational, Region-Based G1GC

Since Java 9, G1GC (Garbage First) is the default collector. From Chapter 11, G1 divides the heap into regions and uses concurrent marking. But the key design choice is **generational with intra-heap regions**:

- Young generation objects are allocated in small regions (Eden).
- Collections of the young generation are frequent (~few seconds) and pause for 5–50 milliseconds.
- Old generation collections are concurrent (the pause is brief; most marking runs while the application runs).
- Fragmentation is controlled by region-based collection, but not eliminated.

For a typical web service:

```
Heap size:           2–8 GB (common on modern servers)
Young gen size:      ~10–20% of heap
Young gen pause:     5–50 ms every few seconds
Old gen pause:       <1 ms (mostly concurrent)
Memory overhead:     ~10% for collection bookkeeping
```

The pause happens because even concurrent marking needs a brief "remark" phase to catch objects that became unreachable during concurrent marking. That pause is usually <10 milliseconds.

### Tuning for Latency: ZGC and Shenandoah

If pause time is critical (low-latency trading, interactive services), you can switch collectors:

```bash
java -XX:+UseZGC application.jar      # <1 millisecond pauses
java -XX:+UseShenandoahGC app.jar     # <10 millisecond pauses
```

**ZGC** (Z Garbage Collector) uses **colored pointers**: the high-order bits of a pointer encode GC metadata. A load barrier on every reference read checks the color and redirects if needed. The cost is ~5–10% throughput loss, but pauses drop to sub-millisecond, even for 100 GB heaps.

**Shenandoah** is similar but uses load barriers instead of colored pointers. Both achieve pause times that would be impossible with G1: the collector runs almost entirely concurrently, only stopping the world for root scanning (microseconds) and final remark (microseconds).

The tradeoff is explicit: you give up throughput (95–97% vs. 98% with G1) to get predictable pauses.

### Why JVM Allocation Patterns Matter

The JVM's generational GC is tuned for short-lived allocations. A request that allocates 10 MB of temporary objects, processes them, and discards them is ideal: those objects die in the young generation without touching the old generation.

But high-allocation-rate workloads (real-time data processing, scientific computing, high-frequency trading) fight this design. If a program allocates 1 GB/second, young gen collections happen every ~100 milliseconds (for a 100 MB young gen), and each collection pauses the application. The GC overhead becomes visible.

**The key metric is allocation rate, not heap size.** A 10 GB heap with 1 MB/second allocation rate (pauses every ~1 minute) is simpler to tune than a 1 GB heap with 100 MB/second allocation rate (pauses every few seconds).

---

## Go: Choosing Sub-Millisecond Pauses at the Cost of Memory

Go's garbage collector prioritizes **pause time predictability**. The target is sub-millisecond pauses—typically 100–300 microseconds, even for very large heaps. This reflects Go's use case: network services that must respond quickly to incoming requests.

### Non-Generational, Concurrent Mark-Sweep

Go's GC is fundamentally simpler than the JVM's:

- **No generations**: all objects are in one heap.
- **Concurrent mark-sweep**: marking runs concurrently with the application; sweep is concurrent.
- **No compaction**: the heap may fragment, but Go's allocator is designed to handle scattered free blocks.
- **Non-moving**: objects are never moved, so pointers remain valid.

The algorithm is tri-color marking (Chapter 11), but Go's design choice is to run it concurrently with minimal coordination:

```go
// Every allocation checks if GC is running
func allocate(size uint64) *Object {
    if gcState.concurrent && gcState.shouldYield() {
        yield()  // let the GC run
    }
    return _alloc(size)
}

// GC marks concurrently
func gcConcurrentMark() {
    for object in allObjects {
        if reachable(object) {
            mark(object)
        }
    }
}
```

The pause comes from:

1. **STW mark start**: scan roots, start concurrent marking (~microseconds).
2. **Concurrent marking**: application runs; GC marks in parallel.
3. **Mark termination**: final re-scan to catch objects that became reachable during concurrent marking (~microseconds).
4. **Concurrent sweep**: free unmarked objects while application runs.

Total pause: typically 50–300 microseconds.

### The Memory Cost

To achieve this, Go accepts **2× memory overhead**. The GC tuning works like this:

```go
// Pseudocode: Go's GC trigger
trigger_threshold = heap_marked_last_cycle * 2.0

if heap_allocated > trigger_threshold {
    start_gc()
}
```

If the last GC cycle marked 100 MB as live, the next GC is triggered when the heap reaches 200 MB. This overshoots ensures that marking can run concurrently without the heap filling up before marking completes.

The cost: a Go program uses roughly 2× as much memory as an equivalent JVM program. A web service in Go might use 500 MB where the same service on the JVM uses 250 MB.

### Why Go Chose This Point

Go is designed for network services and cloud deployments. In this context:

1. **Pause predictability matters**: network services must respond to requests within milliseconds. Sub-millisecond GC pauses are invisible.
2. **Memory is cheap**: a 500 MB service costs $0.01/month on cloud infrastructure. The programmer effort saved by avoiding GC tuning is more valuable.
3. **Simplicity is valuable**: Go's GC is easier to reason about than the JVM's. No generations, no write barriers on every assignment, no card marking bookkeeping.

The non-moving design (objects never relocate) also avoids issues with C interop: a pointer to a Go object can be safely passed to C code without fear of invalidation.

---

## V8: Generational with Hidden Allocation Patterns

V8 (Google's JavaScript engine) uses **generational GC with incremental marking**, reflecting JavaScript's workload: mostly temporary objects created during event processing, with a small set of long-lived objects (closures, cached results, DOM nodes in browsers).

### Young Generation: Scavenger

New objects are allocated in a small "scavenger" space (typically 1–8 MB). When it fills, the scavenger runs:

1. **Evacuating semi-space GC**: copy live objects to a survivor space, flip.
2. **Promotes survivors**: objects that survive two scavenges are promoted to old generation.

The scavenger is fast (pauses <10 ms) because the space is small, but it runs frequently (every few megabytes of allocation).

### Old Generation: Incremental Mark-Sweep

For long-lived objects, V8 uses **incremental marking**: instead of marking everything at once, the marker marks a slice of the heap (a few milliseconds of work), yields to the application, resumes. This distributes the pause:

```
Application timeline:
├─ [run 100ms] ─ [mark 5ms] ─ [run 100ms] ─ [mark 5ms] ─ [run 100ms] ─
│   pause      │  total      │ pause      │  total      │ pause      │
│   invisible  │  ~1% GC     │ invisible  │  ~1% GC     │ invisible  │
```

The pause is distributed so that no single pause exceeds a few milliseconds, even though the total GC time is 1–3% of execution.

### The Hidden Cost: NaN Boxing and Hidden Classes

V8's design choice has hidden costs that affect allocation patterns:

**Hidden Classes**: Every JavaScript object has a hidden class that encodes its shape (which properties it has, in what order). When you add a property, the hidden class changes:

```javascript
let obj = {};        // hidden class A (empty)
obj.x = 1;          // hidden class B (has property x)
obj.y = 2;          // hidden class C (has properties x, y)
```

Each hidden class assignment creates bookkeeping overhead. V8 optimizes for "monomorphic" objects that don't change shape, so code that avoids dynamic property addition cooperates well with the GC.

**NaN Boxing**: V8 encodes JavaScript values in 64 bits. Numbers are tagged differently from objects to distinguish them from pointers. This tagging causes objects to be allocated differently than in other runtimes: a number doesn't allocate; an object does, even if it's a boxed number.

These design choices are invisible to JavaScript code but shape performance. Writing "GC-friendly" JavaScript means:

1. Avoid adding properties to objects dynamically (keep hidden classes stable).
2. Avoid creating wrapper objects for primitives.
3. Pre-allocate arrays with enough capacity (grow-and-copy during incremental marking is expensive).

---

## .NET CoreCLR: Server GC and Workstation GC

.NET offers a choice: **workstation GC** (low latency, single-threaded) or **server GC** (throughput, multi-threaded, per-CPU heaps).

### Generational with Three Generations

Like the JVM and V8, .NET uses three generations:

- **Gen 0**: objects allocated since last GC (nursery, ~few MB).
- **Gen 1**: survivors of Gen 0 collections (buffer, ~few tens of MB).
- **Gen 2**: long-lived objects (main heap, ~few GB).

When Gen 0 fills, a Gen 0 GC runs (fast, ~10 ms). Objects that survive are promoted to Gen 1. When Gen 1 fills, both Gen 0 and Gen 1 are collected (slower, ~50 ms). When Gen 2 fills, a full GC (slow, 100–500 ms).

### Server GC: Per-CPU Heaps

In ASP.NET Core (the most common .NET workload), server GC is the default. Instead of one global heap, there's a heap per CPU core:

```
Heap for CPU 0: [Gen 0] [Gen 1] [Gen 2]
Heap for CPU 1: [Gen 0] [Gen 1] [Gen 2]
Heap for CPU 2: [Gen 0] [Gen 1] [Gen 2]
...
```

Each CPU's heap is GC'd independently. A thread running on CPU 0 allocates from heap 0, and when heap 0's Gen 0 fills, a GC of heap 0 runs (without pausing threads on other CPUs).

This design scales: on an 8-core server, heap operations are mostly lock-free and local to each core. The drawback is higher memory overhead (8 separate Gen 2 spaces instead of 1), but this is acceptable for servers with multiple gigabytes of RAM.

### Tuning for Latency: Concurrent GC

.NET also supports **concurrent GC** (`<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>` in the project file), which runs GC marking concurrently with the application. This reduces pause time from 100–500 ms to 10–50 ms, at the cost of 5–10% throughput loss.

---

## CPython: Reference Counting Plus Cycle Collection

CPython (the standard Python implementation) uses **reference counting**, not tracing GC. Every object carries a count of pointers to it:

```python
x = MyObject()      # ref count = 1
y = x               # ref count = 2
del x               # ref count = 1
del y               # ref count = 0 -> free immediately
```

### Why Reference Counting?

1. **Predictable finalization**: when an object's ref count reaches zero, its destructor runs immediately. In a tracing GC, destructors are delayed until the next collection cycle.
2. **Interoperability with C**: it's easy to increment and decrement ref counts at C extension boundaries.
3. **Simple mental model**: programmers can reason about lifetimes locally.

### The Problem: Cycles

If two objects reference each other, their ref counts never reach zero:

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

a = Node(1)
b = Node(2)
a.next = b
b.next = a  # cycle! a -> b -> a

del a       # a's ref count goes to 0 (b still holds it? No, it goes to 1, not 0)
del b       # b's ref count goes to 1
            # both objects are unreachable, but neither has ref count 0
```

CPython solves this with a **cycle detector**, a separate GC that periodically finds and frees cyclic garbage. This detector runs infrequently (when allocation threshold is hit) and does a mark-sweep on suspects.

### Pauses and Memory Overhead

- **Pause times**: very short (microseconds) and frequent (every MB or so of allocation), rather than long and rare. Programmers rarely notice the pauses.
- **Memory overhead**: each object carries a ref count (typically 8 bytes on 64-bit systems). Python objects are already large (~56 bytes for a basic instance), so this isn't the dominant cost.
- **CPU overhead**: every assignment (`x.field = y`) increments the ref count of `y` and decrements the ref count of the old value of `x.field`. This is fast (two atomic operations) but pervasive.

CPython cooperates with CPython's GC by avoiding cycles (the ref counting model incentivizes this) and using built-in data structures (lists, dicts) that are implemented in C and optimized for ref counting.

Forward reference: See Chapter 60 (reference counting design) for deeper analysis.

---

## Erlang BEAM: Per-Process Heaps and Global Pauses Avoided

Erlang (and Elixir, which runs on the same BEAM virtual machine) takes a radically different approach: **per-process garbage collection**. Each Erlang process has its own isolated heap, and GCs are per-process, not global.

### Process Isolation

In Erlang, lightweight "processes" are not operating system threads; they're scheduled by the BEAM runtime. Each process has:

- **Private heap** (typically 1–10 MB): objects allocated by that process.
- **Message mailbox** (shared with sender): incoming messages.
- **Stack and registers** (private).

When a process's heap fills, only that process is paused for GC. Other processes continue running. This eliminates the "stop the world" problem entirely:

```
Process A: ─────────── [GC pause 10ms] ──────────
Process B: ───── [GC pause 15ms] ─────────
Process C: ──────────────── [GC pause 8ms] ────

Global pause: 0 ms (processes GC independently)
```

### The Cost: Peak Memory Usage

The cost is memory: if you have 10,000 processes, each with a 1 MB heap, you use 10 GB of memory. An equivalent monolithic application might use 100 MB. This is the tradeoff: predictable latency for higher memory overhead.

### Generational Within Each Process

Each BEAM process uses a small generational heap:

- **Young generation** ("nursery"): ~2 KB, collected very frequently.
- **Old generation** ("mature"): ~1 MB, collected less frequently.

Collections are fast because the process's own heap is small. Even in the worst case (a Gen 1 collection of the old generation), the pause is single-digit milliseconds.

### Why Erlang Chose This Point

Erlang is designed for highly concurrent systems: telecom switch software, messaging, chat servers. In these workloads:

1. **Pause predictability is critical**: a 100 ms pause in one process affects the whole system. Erlang makes the pause affect only one lightweight process, which is acceptable.
2. **Concurrency is the primary abstraction**: you think in terms of thousands of processes, not threads. Per-process GC aligns with this mental model.
3. **Memory is cheap relative to latency**: a telecom system that costs 10× more to build than a monolithic web service is acceptable if it can handle 10,000× more connections.

Erlang cooperates with its GC by:

- Avoiding large allocations (break into smaller messages).
- Using tail recursion (doesn't accumulate stack frames).
- Letting processes die and restart (natural heap reclamation).

---

## Worked Example: Processing 1 Million JSON Records

To make the tradeoffs concrete, consider a workload: read 1 million JSON records from disk, decode, extract fields, and write results. Measured on:

- **JVM (G1GC)**: 2 GB heap, 8 cores.
- **Go**: default settings.
- **.NET (Server GC)**: 2 GB heap, 8 cores.
- **Node.js (V8)**: default settings.

### Hypothetical Results

```
Runtime    | Wall Time | GC Pause | Max Pause | GC CPU | RSS   |
-----------|-----------|----------|-----------|--------|-------|
JVM G1     | 45 sec    | 250 ms   | 180 ms    | 2.8%   | 1.2GB |
Go         | 48 sec    | 180 ms   | 95 ms     | 3.2%   | 2.4GB |
.NET       | 42 sec    | 200 ms   | 140 ms    | 2.5%   | 1.8GB |
Node.js    | 60 sec    | 380 ms   | 220 ms    | 4.1%   | 1.5GB |
```

**Why the differences?**

- **JVM G1** is balanced: good throughput (45 sec), reasonable pauses (180 ms total).
- **Go** is slower due to memory overhead (2.4 GB RSS) and higher allocation rate (more GC runs), but pauses are more frequent and lower (95 ms max).
- **.NET Server GC** wins on throughput (42 sec) because per-CPU heaps reduce contention and allow lock-free allocation.
- **Node.js** is slowest because JavaScript's boxing and hidden class overhead (slower JSON parsing) and V8's scavenger pauses add up.

The allocation rate varies:

- **JVM/Go/.NET**: ~22 MB/sec (typical for JSON parsing).
- **Node.js**: ~31 MB/sec (hidden class churn, temporary strings).

This is why **allocation rate matters more than heap size**: a 2 GB heap with 22 MB/sec allocation rate is easy to tune. A 2 GB heap with 200 MB/sec allocation rate is hard to tune (GC runs every few seconds).

---

## Implications For Your Code

### 1. Allocation Rate Dominates

The single most important metric for GC-friendly code is **allocation rate**, not heap size or object count. Minimize allocations per operation:

**In Java:**
```java
// Bad: allocates per iteration
for (int i = 0; i < items.size(); i++) {
    String s = "Item " + items.get(i);  // allocates a String
    process(s);
}

// Good: reuses StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < items.size(); i++) {
    sb.setLength(0);
    sb.append("Item ").append(items.get(i));
    process(sb.toString());  // one temporary object, reused
}
```

**In Go:**
```go
// Bad: allocates per iteration
for i := 0; i < len(items); i++ {
    s := fmt.Sprintf("Item %v", items[i])  // allocates a string
    process(s)
}

// Good: allocate once, reuse
var buf bytes.Buffer
for i := 0; i < len(items); i++ {
    buf.Reset()
    fmt.Fprintf(&buf, "Item %v", items[i])
    process(buf.String())  // one allocation amortized over loop
}
```

### 2. Pre-Size Collections

Pre-allocating collections to final size avoids grow-and-copy:

**In Java:**
```java
List<Item> items = new ArrayList<>(1_000_000);  // allocate once
for (Item item : source) {
    items.add(item);  // no reallocation
}
```

**In Go:**
```go
items := make([]Item, 0, 1_000_000)  // cap = 1M, no reallocation
for item := range source {
    items = append(item, item)
}
```

### 3. Object Pooling for Hot Paths

In latency-sensitive code, recycle objects instead of allocating:

**In Java:**
```java
// Allocate once at startup
ThreadLocal<byte[]> buffer = ThreadLocal.withInitial(() -> new byte[4096]);

// In a hot method
byte[] buf = buffer.get();
// Use buf, then reset (don't deallocate)
Arrays.fill(buf, (byte) 0);
```

**In Go:**
```go
var pool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

// In a hot method
buf := pool.Get().([]byte)
// Use buf
pool.Put(buf)
```

### 4. Know Your GC's Barriers

Write barriers (Chapter 11) add overhead to every pointer assignment. Code that mutates objects heavily (linked lists, graphs) pays this cost on every mutation. Code that allocates once and reads repeatedly pays no barrier cost.

**Barrier-heavy (problematic for modern GCs):**
```java
LinkedList<Item> list = new LinkedList<>();
for (Item item : items) {
    list.add(item);  // write barrier on every add
}
```

**Barrier-light (better for modern GCs):**
```java
List<Item> list = new ArrayList<>();
for (Item item : items) {
    list.add(item);  // no barrier (young gen allocation)
}
```

---

## Common Misconceptions

### 1. "Modern GCs Make Allocation Free"

Allocation is cheap, but not free. A tracing GC (JVM, Go, V8) must periodically scan all live objects to find garbage. If allocation rate is high, GC runs frequently, and the overhead becomes visible. Reference counting (CPython) amortizes the cost per assignment. No GC eliminates the fundamental cost of memory management.

### 2. "Bigger Heaps Are Always Better"

A larger heap means GC runs less frequently, but each run takes longer and pauses the application more. A 10 GB heap with 100 MB/sec allocation has a GC pause every 100 seconds. A 1 GB heap with 10 MB/sec allocation has a GC pause every 100 seconds, but the pause is 10× shorter. Choose the smallest heap that avoids excessive GC, not the largest possible heap.

### 3. "Choose Low-Pause GC and Problem Solved"

Low-pause GCs (ZGC, Shenandoah, Go) trade throughput for pause predictability. If your workload doesn't care about pause time (batch processing, overnight jobs), low-pause GC wastes 5–10% throughput. Choose based on your requirements, not because "low pause" sounds good.

### 4. "GC Tuning Is a Black Art"

GC tuning is not black art; it's engineering. The JVM's `-XX:+PrintGCDetails` and `-XX:+PrintGCDateStamps` flags produce logs that show pause frequency, pause duration, and GC cause. Analysis of these logs reveals whether you're allocation-bound (increase young gen size), object-retention-bound (profile to find leaks), or pause-bound (switch collectors or reduce allocation rate). Go's `runtime.MemStats` and .NET's `GC.GetTotalMemory()` provide similar insights.

### 5. "Erlang's Per-Process GC Is Always Better for Latency"

Erlang's per-process GC is excellent for systems with many independent processes (telecom, messaging). It's poor for systems with shared mutable state (databases, large in-memory caches, scientific computing). You can't separate pause time from your system's architecture.

---

## Exercises

1. **JVM Heap Tuning**: Write a Java program that allocates and discards 10 million temporary objects. Measure pause times with `-XX:+PrintGCDetails` using G1GC (default), ZGC, and Shenandoah. Plot the pause profile for each. Which would you choose for a web service? For a batch job?

2. **Go Memory Overhead**: Write a Go program and a Java program that process the same workload (e.g., 100 MB of JSON). Measure RSS (resident set size) for each. Compute the memory overhead ratio. Then reduce allocation rate in both and re-measure. How does overhead change?

3. **V8 Hidden Classes**: Write a Node.js program that dynamically adds properties to objects:
   ```javascript
   let obj = {};
   for (let i = 0; i < 100_000; i++) {
       obj['prop' + i] = i;  // creates new hidden class each time
   }
   ```
   Measure GC pause time. Then rewrite to use a Map. Measure again. What changed? Why?

4. **CPython Cycle Collection**: Write a Python program with cyclic references:
   ```python
   class Node:
       def __init__(self, value):
           self.value = value
           self.next = None
   
   a = Node(1)
   b = Node(2)
   a.next = b
   b.next = a
   del a, b  # Cycle is now garbage
   ```
   Disable the cycle collector (`gc.disable()`) and measure how much memory the cycle occupies before `gc.collect()` is called. Re-enable and measure. What's the cost of reference counting without cycle collection?

5. **Erlang Process Isolation**: Write an Elixir program that spawns 1,000 lightweight processes, each processing a chunk of data. Measure the total pause time during processing (it should be near zero for any individual observer). Compare to a single-process implementation. What's the memory overhead?

---

## Summary

Every garbage collector makes explicit tradeoffs between pause time, throughput, and programmer complexity. The JVM optimizes for throughput (G1GC) with options for latency (ZGC). Go optimizes for pause predictability at the cost of memory. V8 distributes pause time over many small increments. .NET offers both paths (workstation vs. server GC). CPython uses reference counting for immediate finalization. Erlang sacrifices memory and throughput for per-process GC that eliminates global pauses.

These choices are made at runtime design time. You cannot escape them. You can only write code that cooperates with your chosen runtime by minimizing allocation rate, pre-sizing collections, and understanding the specific barriers and collection patterns your runtime uses. The "best" GC is not the lowest-pause or highest-throughput; it's the one that aligns with your system's requirements—and sometimes that alignment requires changing your code, not changing your runtime.

---

> **[← Previous: How the JVM Works](06-how-the-jvm-works.md)**  ·  **[↑ Part 6](README.md)**  ·  **[Next: Why Rust Ownership Exists →](08-why-rust-ownership-exists.md)**
