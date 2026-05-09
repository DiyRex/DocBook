# Chapter 11 — Garbage Collection Internals

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain garbage collection as a reachability problem: the runtime finds what is still accessible from known roots, and everything else is garbage.
2. Describe the fundamental phases that every collector must accomplish — discovering live objects, freeing dead ones, and optionally compacting — and the tradeoffs each phase creates.
3. Compare the major collector strategies — mark-sweep, copying/semi-space, generational, tri-color marking, concurrent and incremental approaches — and predict when each is appropriate.
4. Understand write barriers and load barriers as the mechanisms that let collectors run concurrently with mutating code.
5. Recognize real-world collectors (JVM G1, ZGC, Go's collector, V8) as applications of these principles to different tradeoff points.
6. Debug memory issues that cross the boundary between application code and GC — leaks, pauses, promotion failures — by understanding what the collector is actually doing.

---

Garbage collection is not magic. The magic is in the engineering of one very boring problem: **given a set of pointers, find all the reachable objects, and free the rest.** The interesting parts are the engineering shortcuts that do this work fast, with minimal pause time, and without stopping the entire world for too long.

This chapter deconstructs that engineering. By the end, the GC in Python, Java, Go, and JavaScript will no longer be a black box. You will see the fundamental mechanisms, the tradeoffs they embody, and the specific choices each language made about which tradeoffs to accept.

---

## 11.1 Roots and Reachability

Garbage collection is fundamentally a **reachability problem**. An object is *live* if there exists a chain of pointers from a known starting place to that object. Everything else is garbage.

The "known starting places" are **roots**. Every GC must identify them correctly; if you miss a root, the collector may free objects that are actually still reachable, leading to use-after-free. If you count something as a root that isn't, objects stay alive forever and the heap grows unnecessarily (a leak).

Roots include:

- **Globals and static data**: any pointer-containing variable with static lifetime.
- **Stack frames**: local variables, parameters, temporary values on the stack during function execution.
- **Registers**: at GC time, values still in CPU registers are considered reachable.
- **Pinned or registered objects**: some GC systems allow the mutator (running code) to explicitly register objects as roots — useful for passing between native code and the runtime.

Once the roots are known, reachability is straightforward: if a root points to object A, A is live. If A points to B, B is live. If B points to C, C is live. Transitively, A, B, and C are all live. Everything else on the heap is garbage.

The problem is not the concept. The problem is **speed and pause time**. Traversing all reachable objects takes time proportional to the number of live objects and the complexity of the reference graph. During that traversal, if the mutator is stopped, the application pauses. Some collectors hide this pause by working concurrently; others accept the pause as a tradeoff.

---

## 11.2 Mark-Sweep Collectors

The simplest GC strategy is **mark-sweep**: traverse from the roots, mark everything reachable, then sweep the heap and free the unmarked objects.

### Phase 1: Mark

Starting from the roots, the collector performs a depth-first or breadth-first search (usually depth-first) over the reference graph:

```pseudocode
mark(object):
    if object is already marked:
        return
    mark object as live
    for each pointer field in object:
        child = get(field)
        if child is not null:
            mark(child)

gc_mark():
    for each root r in roots:
        mark(r)
```

If there are cycles in the reference graph (A points to B, B points to A), the cycle detection in the `if already marked` check prevents infinite recursion. This is a major advantage of mark-sweep over reference counting, which struggles with cycles.

### Phase 2: Sweep

After marking, walk the heap linearly and free everything not marked:

```pseudocode
gc_sweep():
    for each object obj in heap:
        if obj is not marked:
            free(obj)
        else:
            unmark obj  // clear the mark for next GC
```

### ASCII Diagram of Mark-Sweep

```
Before GC:
┌─────────────────────────────────────────┐
│ Heap: [A]→[B]→[C]  [D]→[E]   [F]  [G] │
│ Root points to A                        │
│ A→B, B→C (chain)                       │
│ D→E (separate)                         │
│ F, G unreachable                       │
└─────────────────────────────────────────┘

After Mark:
┌─────────────────────────────────────────┐
│ [A✓]→[B✓]→[C✓]  [D]→[E]   [F]  [G]     │
│ Root→A reachable, so A marked           │
│ A→B→C transitively reachable, marked    │
│ D, E not reachable from root, unmarked  │
│ F, G unmarked                           │
└─────────────────────────────────────────┘

After Sweep (free unmarked):
┌─────────────────────────────────────────┐
│ [A]→[B]→[C]                             │
│ Free list: [D], [E], [F], [G]           │
└─────────────────────────────────────────┘
```

### Pros and Cons

**Pros:**
- Handles cycles naturally (no reference-counting cycle collectors needed).
- Simple conceptually and relatively easy to implement.
- Can work with any heap layout.

**Cons:**
- **Stop-the-world**: the entire application pauses while mark and sweep run. The pause time is proportional to the number of live objects and the size of the reference graph.
- **No compaction**: free blocks are scattered across the heap. Repeated mark-sweep can lead to **fragmentation** — small gaps between allocated objects that waste space and reduce cache locality.
- **Pause time unpredictable**: if the program allocates heavily in one phase, the heap becomes fragmented, and the next GC may take much longer.

**When to use:**
- Teaching or research (simplicity).
- Applications where occasional pauses are acceptable (batch jobs, non-interactive systems).
- Embedded systems with tight memory constraints and predictable workloads.

---

## 11.3 Copying and Semi-Space Collectors

To avoid fragmentation, some collectors use **copying**: divide the heap into two equal halves, allocate in one half ("from-space"), and when GC runs, copy live objects into the other half ("to-space"), then flip the pointers.

### Algorithm

```pseudocode
gc_copy():
    to_space = heap[size/2 : size]   // empty half
    from_space = heap[0 : size/2]    // full half
    
    for each root r in roots:
        r = copy_object(r, to_space)
    
    // Evacuate the rest by traversing the copy frontier
    scan_pointer = start of to_space
    while scan_pointer < end of to_space:
        obj = object at scan_pointer
        for each field in obj:
            child = obj.field
            if child is in from_space and not yet copied:
                child = copy_object(child, to_space)
                obj.field = new location
        scan_pointer += size of obj

    // Flip: from now on, to_space is from_space
    swap(from_space, to_space)
    free from_space entirely

copy_object(obj, to_space):
    if obj is not in from_space:
        return obj
    if obj already copied (forwarding pointer exists):
        return obj.forwarding_pointer
    new_location = allocate space in to_space
    copy obj to new_location
    obj.forwarding_pointer = new_location
    return new_location
```

### Diagram

```
Before GC:
┌────────────────────────┬────────────────────┐
│ From-space (full)      │ To-space (empty)   │
│ [A]→[B]→[C]  [D] [F]   │ (free)             │
│ Root→A                 │                    │
└────────────────────────┴────────────────────┘

Copying in progress:
┌────────────────────────┬────────────────────┐
│ From-space             │ To-space (filling) │
│ [A]→[B]→[C]  [D] [F]   │ [A']→[B']→[C']     │
│ (marked to copy)       │ Root now→A'        │
└────────────────────────┴────────────────────┘

After flip:
┌────────────────────────┬────────────────────┐
│ To-space (now active)  │ From-space (free)  │
│ [A]→[B]→[C]           │ (free)             │
│ Root→A                 │                    │
└────────────────────────┴────────────────────┘
```

### Pros and Cons

**Pros:**
- **Automatic compaction**: live objects are copied contiguously into to-space, eliminating fragmentation and improving cache locality.
- **Allocation is trivial**: bump-pointer allocation (advance a pointer, allocate) is the fastest possible allocation scheme.
- **Fast pause time for small nurseries**: pause time is proportional to the number of live objects, not the heap size. For a small young generation (nursery), this is fast.

**Cons:**
- **Doubles memory overhead**: you need two heap halves, so you only use half the physical memory for allocation at any time.
- **Survivor bias**: if most objects die young, copying wastes time moving the few survivors. Generational collectors use semi-space only for the young generation to mitigate this.
- **Complex pointer updates**: every pointer from from-space to to-space must be updated. This is either done during the copy traversal (if you scan carefully) or requires a separate pass.

**When to use:**
- Young generation in a generational collector.
- Real-time systems where pause time must be bounded (because pause is proportional to live data, not heap size).
- JVM G1, Hotspot ParNew, and many production collectors use semi-space or variants for the young gen.

---

## 11.4 Generational Garbage Collection

The **weak generational hypothesis** states that **most objects die young**. Empirically, this is true in most applications: a request comes in, allocates temporary objects to process it, returns a response, and the temporary objects are freed. The few objects that survive — user sessions, cached state, application state — tend to live a long time.

Generational GC exploits this hypothesis by dividing the heap into **generations**:

- **Young/Nursery generation**: where new objects are allocated. GCs here are frequent and cheap.
- **Old/Tenured generation**: where long-lived objects are promoted. GCs here are rare and expensive.

When the nursery fills, the GC runs, surviving objects are promoted to the old generation, and the nursery is reclaimed. The old generation is collected infrequently (when it fills, or on a background schedule).

### Write Barriers and Card Marking

The problem: when collecting the young generation, you must find all roots, including pointers *from the old generation into the young generation*. Scanning the entire old generation defeats the purpose (young gen GC would be slow).

Solution: **write barriers**. Every time a reference is written (an assignment that could create a pointer from old to young), the runtime records the source address:

```pseudocode
// Pseudocode for write barrier
set_field(obj, field_offset, new_value):
    obj[field_offset] = new_value
    if is_old_generation(obj) and is_young_generation(new_value):
        record_write(obj)  // obj is a root for young gen GC
```

This recording can be done at various granularities:

- **Card marking**: divide the old generation into fixed-size "cards" (typically 512 bytes). When a write barrier fires, mark the card as "dirty." During young gen GC, scan only dirty cards.
- **Write log**: maintain a log of all write locations. Scan the log during GC.

Card marking is efficient because a single byte (or bit) represents a large block, and checking dirtiness is very cheap. Modern GCs (HotSpot, V8, Go) use card marking.

### Example: Generational Collection Cycle

```
Initial state:
Young Gen: [A] [B] [C] [D] [E]  (5 objects, young)
Old Gen:   [X] [Y] [Z]           (3 objects, old)

After first young gen GC (assume A, C survive; B, D, E die):
Young Gen: [A] [C]               (promoted, now old; nursery empty)
Old Gen:   [X] [Y] [Z] [A] [C]   (5 objects)
Dirty cards: none (objects in old gen are stable)

Application allocates [F] [G] [H]:
Young Gen: [F] [G] [H]
Old Gen:   [X] [Y] [Z] [A] [C]

Code executes: Y.field = G  (old-to-young pointer):
Dirty cards: mark card containing Y

Young gen fills again. GC scans dirty cards, finds Y→G:
Young Gen after GC: [G] [H]  (F dies, G and H promoted)
Old Gen:   [X] [Y] [Z] [A] [C] [G] [H]
```

### Pros and Cons

**Pros:**
- **Young gen collections are cheap**: they collect only the small nursery and nearby survivors, not the entire heap.
- **Scales with allocation rate**: long-lived objects are collected infrequently; the frequency of collection depends on how fast you fill the nursery.
- **Real-world performance**: most applications with generational GC achieve very good pause times (milliseconds for young, seconds for old).

**Cons:**
- **Complexity**: write barriers add overhead to every assignment, card marking requires bookkeeping, and the promotion mechanism must be designed carefully.
- **Old gen collections are still expensive**: when the old generation fills, a full collection is expensive and pauses the application.
- **Fragmentation in old gen**: old gen is often mark-sweep, which can fragment. Some collectors add periodic full compactions, which are even more expensive.

**When to use:**
- Production JVM applications (default in HotSpot).
- JavaScript engines (V8, JavaScriptCore).
- Python (though CPython also uses reference counting).

---

## 11.5 Tri-Color Marking and Concurrent Collection

Generational GC reduces pause time for young collections, but old gen collections still pause the application. To reduce old gen pauses, many collectors allow GC to run **concurrently** with the mutator (the application code).

**Tri-color marking** enables this. Every object is in one of three states:

- **White**: not yet visited by the collector.
- **Gray**: visited, but its children have not been scanned.
- **Black**: visited, and all its children have been scanned.

The **invariant**: *there is no pointer from black to white*. A black object's children are guaranteed to be at least gray; it never points directly to unvisited white objects.

### The Algorithm

```pseudocode
// Mark phase, can run concurrently with mutator
concurrent_mark():
    worklist = all roots
    for each obj in worklist:
        if obj is white:
            obj.color = gray
        else if obj is gray:
            obj.color = black
            for each child of obj:
                if child is white:
                    worklist.add(child)
                    child.color = gray

// If mutator executes while marking is running:
//   write_barrier(obj, child) ensures the invariant holds
```

### Write Barrier for Concurrent Collection

When the mutator runs concurrently with marking and executes `obj.field = child`:

```pseudocode
write_barrier_for_concurrent_gc(obj, child):
    if obj is black and child is white:
        // This would violate the invariant!
        // Push child back to gray to be rescanned
        child.color = gray
        worklist.add(child)
```

The barrier is fast (a few branches and memory writes) and preserves the invariant. If a black object gets a new white child, the barrier catches it and re-grays the child.

### Diagram

```
Initial (no marking started):
All white: [A] [B] [C] [D]

Mark begins, roots marked gray:
Gray (work to do): [A] [B]
White (not visited): [C] [D]

Scan A (children C, D):
Black (done): [A]
Gray (work to do): [B] [C] [D]
White: (none)

Scan B (no children):
Black: [A] [B]
Gray: [C] [D]

Scan C (child D already gray):
Black: [A] [B] [C]
Gray: [D]

Scan D:
Black: [A] [B] [C] [D]
Gray: (none)

Mark phase done. Sweep all white objects (there are none; all are black).
Next cycle, mark all black as white again to prepare for next GC.
```

### Pros and Cons

**Pros:**
- Marking can run concurrently with the mutator, dramatically reducing pause time.
- Incrementally marking (processing a few objects, yielding to mutator, resuming) is possible.
- Scales to large heaps because pause is not proportional to heap size.

**Cons:**
- Write barriers add overhead to *every* pointer assignment. In a heavily mutating workload, this overhead is noticeable.
- Complex implementation: coordinating concurrent marking with the mutator requires careful synchronization.
- Floating garbage: objects that became unreachable during marking are not collected until the next cycle. This wastes memory but is acceptable in practice.

**When to use:**
- Large heap applications where pause time matters (JVM G1, ZGC, Shenandoah).
- Real-time systems (Go GC, Shenandoah).
- Modern Python (experimental concurrent GC in 3.13+).

---

## 11.6 Real-World Collectors: A Survey

Production garbage collectors apply these principles in different combinations, each making specific tradeoffs:

### JVM G1 (Garbage First)

**Strategy**: region-based, generational, concurrent marking.

**How it works**: divide the heap into fixed-size regions (typically 1–32 MB). Young and old generation objects can be in any region. Young gen GC copies objects from young regions to survivor regions. Concurrent marking runs in the background, and when regions accumulate garbage, G1 collects them in garbage-first order (highest garbage ratio first).

**Pause time target**: explicitly configurable (e.g., "pause for at most 200ms").

**When to use**: large heap JVM applications where pause time is predictable and bounded.

### JVM ZGC and Shenandoah

**Strategy**: ultra-concurrent, using **colored pointers** or **load barriers**.

**How it works**: 
- ZGC colors pointers themselves: the unused bits of a pointer encode whether the object is marked, remapped, etc.
- At GC time, ZGC runs marking concurrently; the mutator's load barrier checks the pointer color and redirects if needed.
- Shenandoah is similar but uses load barriers on object references instead of pointer colors.

**Pause time**: sub-millisecond (often under 1ms even for large heaps).

**Tradeoff**: load barriers on *every* reference access (read) add latency, but pause time is dramatically reduced.

**When to use**: ultra-low-pause JVM applications (trading throughput for pause predictability).

### Go GC

**Strategy**: concurrent tri-color mark-sweep, non-copying, no compaction.

**How it works**: 
- Allocation is in a single heap (no generations).
- Concurrent marking via write barriers; mutator and marker run concurrently.
- Sweep phase is concurrent: freed blocks are added to free lists while the mutator runs.

**Pause time**: typically 100–300 microseconds (sub-millisecond).

**Tradeoff**: no compaction means fragmentation is possible, but Go's allocator handles it reasonably well. No generational young gen means large workloads full of temporary objects may have less-than-ideal pause time.

**When to use**: service-oriented Go programs where pause time under 1ms is important.

### V8 (JavaScript)

**Strategy**: generational with incremental marking and concurrent sweeping.

**How it works**:
- Young gen (scavenger): copy-based, frequent.
- Old gen: mark-sweep, incremental marking (pause for a few milliseconds, mark some objects, resume mutator, repeat), concurrent sweeping.

**Pause time**: typically 10–50ms for full GC, but incremental marking distributes the pause.

**Tradeoff**: JavaScript applications often have large heaps and allocate heavily; incremental marking spreads the GC work to hide pauses.

**When to use**: JavaScript engines in browsers and Node.js.

---

## 11.7 Worked Example: A Simple Mark-Sweep Collector

To cement understanding, here is pseudocode for a minimal mark-sweep collector:

```cpp
// Pseudo-code. Real collectors are much more complex.

struct Object {
    uint32_t type_id;
    uint32_t mark_bit : 1;
    uint32_t generation : 2;
    uint32_t size;
    // ... payload ...
};

struct Heap {
    Object* objects;
    size_t object_count;
    size_t max_objects;
};

// Roots: globals, stack, registers (simplified)
void collect_roots(Heap* heap, std::vector<Object*>& roots) {
    // Globals: scan .data, .bss
    // Stack: scan stack frames
    // Registers: saved CPU register state
    // For this example, assume roots are registered explicitly
}

void mark(Object* obj, Heap* heap) {
    if (!obj || obj->mark_bit) return;
    obj->mark_bit = 1;
    
    // Scan this object's children
    uint32_t* payload = (uint32_t*)(obj + 1);
    for (size_t i = 0; i < (obj->size / sizeof(uint32_t)); i++) {
        Object* child = (Object*)payload[i];
        if (child >= heap->objects && 
            child < heap->objects + heap->object_count) {
            mark(child, heap);
        }
    }
}

void gc_mark(Heap* heap) {
    std::vector<Object*> roots;
    collect_roots(heap, roots);
    for (Object* root : roots) {
        mark(root, heap);
    }
}

void gc_sweep(Heap* heap) {
    size_t write_idx = 0;
    for (size_t i = 0; i < heap->object_count; i++) {
        Object* obj = &heap->objects[i];
        if (obj->mark_bit) {
            obj->mark_bit = 0;  // Clear for next cycle
            if (write_idx != i) {
                heap->objects[write_idx] = *obj;
            }
            write_idx++;
        }
        // else: this object is freed (overwritten on next allocation)
    }
    heap->object_count = write_idx;
}

void gc_collect(Heap* heap) {
    gc_mark(heap);
    gc_sweep(heap);
}
```

**Key observations:**
- Cycles (A→B, B→A) are handled naturally by the `mark_bit` check.
- This collector is stop-the-world; the entire application pauses during `gc_collect`.
- Real collectors add incremental marking, write barriers, generational divisions, compaction, and concurrent marking — all refinements of this basic algorithm.

---

## 11.8 Tradeoffs in GC Design

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| **Manual (malloc/free)** | No pause; maximum control; fastest allocation and deallocation | Use-after-free; double-free; memory leaks; complex code | Systems code (C, C++); performance-critical paths |
| **Reference counting** | Immediate cleanup; no pause; cache-friendly | Cycles need secondary collection; slower assignment (rc++/rc--); memory overhead | Swift, Objective-C; systems with predictable lifetimes |
| **Mark-sweep** | Handles cycles; simple; no memory doubling | Stop-the-world pause; fragmentation; pause proportional to live data | Teaching, batch jobs, embedded systems |
| **Copying/semi-space** | Automatic compaction; fast allocation; pause proportional to live data | Doubles heap memory; copying overhead | Young generation in generational collectors |
| **Generational** | Young collections very fast; reduces old-gen frequency; scales well | Complex; write barrier overhead; fragmentation in old gen | Production JVM, JavaScript, Python |
| **Concurrent marking** | Sub-millisecond pauses even for large heaps; hidden GC work | Write barrier overhead; floating garbage; implementation complexity | Large-heap services; low-latency applications |
| **Incremental GC** | Pause distributed across time; predictable latency | Floating garbage; repeated traversal overhead | Real-time, soft-realtime applications |

---

## 11.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Garbage collection means I don't have to think about memory." | GC handles deallocation, but memory leaks (holding references too long), fragmentation, and pause time are still your concern. |
| "GC is always slower than manual memory management." | In allocation rate and throughput, GC is often *faster* than malloc due to bump-pointer allocation and amortized costs. Peak latency is higher (pauses) but throughput is competitive. |
| "Generational GC means you have two heaps." | Generational GC is a *collection strategy*, not a memory division. The heap is unified; GC frequency varies by age. |
| "Write barriers have negligible cost." | Write barriers run on *every* pointer assignment. In mutation-heavy workloads, this overhead is 5–15%. Compiler optimizations help but don't eliminate it. |
| "GC pause time is unpredictable." | Modern collectors (G1, ZGC, Shenandoah) target pause time explicitly and achieve predictable sub-millisecond pauses. Traditional mark-sweep is unpredictable; modern collectors are not. |
| "Concurrent GC means the mutator and collector run at the same time without synchronization." | Concurrent collectors use write barriers, careful synchronization, and invariants (like tri-color marking) to ensure correctness. There is synchronization; it is hidden. |
| "Copying collectors waste half the heap." | Copying is used for the *young generation* only (which is small). The old generation is not copied, so total memory overhead is modest (20–30%, not 50%). |
| "If GC runs in the background, my app never pauses." | All collectors have a stop-the-world final marking or remark phase. Concurrent collectors reduce the length of this pause (from seconds to milliseconds or microseconds) but don't eliminate it. |

---

## 11.10 Debugging GC Issues

When GC behavior is a problem, these debugging techniques help:

### Long Pause Times

**Symptom**: application freezes for multiple seconds.

**Diagnosis**:
- Check GC log (`-XX:+PrintGC` in JVM, `GODEBUG=gctrace=1` in Go).
- Pause time is often proportional to live data; large heaps = large pauses (in mark-sweep).
- In generational collectors, full GC (triggered by old gen overflow) is much more expensive than young gen GC.

**Remedy**:
- Reduce live data (fewer long-lived objects, release references sooner).
- Switch to a low-pause collector (G1, ZGC, Shenandoah in JVM; change GC tuning in Go).
- Increase heap size to reduce collection frequency (tradeoff: more memory usage).

### Memory Leaks

**Symptom**: heap grows indefinitely.

**Diagnosis**:
- Use a heap profiler (JVM: JProfiler, YourKit; Go: pprof; Node: --inspect).
- Identify which objects are accumulating.
- Trace references: why is that object still reachable? Check listener registrations, caches, global collections.

**Remedy**:
- Remove the reference (unregister listener, evict cache entry, clear global).
- Use weak references (Java: WeakReference; JavaScript: WeakMap; Go: runtime.SetFinalizer).

### High Allocation Rate

**Symptom**: GC runs frequently, CPU goes to GC instead of application.

**Diagnosis**:
- Profile allocation (`-XX:+PrintAllocationProfile` in JVM).
- Identify hot paths that allocate excessively.

**Remedy**:
- Reduce allocation (reuse objects, use object pools, stack allocation where possible).
- Use allocation-aware data structures (arrays instead of linked lists, bulk operations instead of per-item allocation).

---

## 11.11 Exercises

1. **Mark-sweep walkthrough**: Draw a heap with 5 objects (A, B, C, D, E) where A is a root, A→B→C is a chain, D→E→D is a cycle, and F is unreachable. Walk through mark and sweep, showing which objects get freed.

2. **Generational hypothesis**: Profile a web request in a language you know (use a heap profiler). What fraction of allocated objects survive to the end of the request? If most die (generational hypothesis holds), estimate how much faster a young-gen GC would be vs. a full GC.

3. **Write barrier cost**: In a tight loop that mutates a large data structure (e.g., updating an array of objects), estimate how much overhead the write barrier adds. (Hint: measure allocation rate with and without frequent writes.)

4. **Pause time analysis**: Run a program in a GC'd language with GC logging enabled. Collect 10–20 GC pause times. Plot them: are they normally distributed? Bimodal (young vs. old)? What's the 99th percentile pause time?

5. **Concurrent GC reasoning**: If a concurrent collector detects a white object that has no incoming pointers (is unreachable), but that object was white when marking started, will it be freed in this cycle or the next? Explain.

6. **Reference counting vs. mark-sweep**: Design a small program (pseudocode, no implementation needed) that would be faster under reference counting than mark-sweep, and another that would be slower. Explain why.

---

## 11.12 Summary

Garbage collection is a reachability problem: find all objects reachable from roots, free the rest. Every production collector applies the same fundamental techniques — marking, sweeping, copying, generational division, write barriers, concurrent marking — in different combinations to hit different latency, throughput, and memory tradeoffs.

Mark-sweep is the conceptual baseline: simple, handles cycles, but stop-the-world pauses. Copying reduces pause time and fragments but wastes memory. Generational collectors exploit the weak generational hypothesis to make young-gen collections fast and cheap. Concurrent marking, enabled by write barriers and tri-color invariants, hides most GC pause time by running collection in parallel with the mutator.

Real-world collectors — JVM G1, ZGC, Go, V8 — layer these techniques intelligently. Understanding them is the foundation for predicting pause time, diagnosing memory issues, and choosing the right GC configuration for your application.

When GC pauses hurt your application, you now have the conceptual vocabulary to diagnose why and the design space of solutions to explore. That is the point of this chapter: making GC concrete, not magical.

---

**[← Previous: Chapter 10 — Reference Counting](10-reference-counting.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 12 — How Python/Go/Laravel Manage Memory →](12-how-languages-manage-memory.md)**
