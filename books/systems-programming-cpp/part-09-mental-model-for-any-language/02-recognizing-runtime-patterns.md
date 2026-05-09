# Chapter 92 — Recognizing Common Runtime Patterns

Every modern runtime ships some combination of the same pieces: an event loop, a thread pool, a scheduler, a garbage collector, a JIT compiler. The names change. The languages change. The underlying patterns do not. Once you can name the pieces, every "look at this fancy new tech" feels familiar.

This chapter teaches you to recognize five core runtime patterns and the tradeoffs each makes. By the end, you will be able to look at any unfamiliar runtime—Bun, Deno, Pony, Mojo, WebAssembly—and place it on a map of known patterns. You will understand what it is optimizing for and what it sacrifices.

---

## 92.1 Event Loop + Worker Pool

### The Pattern

One thread runs an **event loop**. CPU-heavy work is offloaded to a **thread pool**. The loop waits for I/O, wakes sleeping threads, and routes work.

```
┌─────────────────────────────────────────┐
│ Main Thread: Event Loop                 │
├─────────────────────────────────────────┤
│ while (!done) {                         │
│   1. Run ready I/O callbacks            │
│   2. Wait on epoll/kqueue for I/O       │
│   3. Re-queue tasks when I/O ready      │
│ }                                       │
└──────────────┬──────────────────────────┘
               │
      ┌────────┴────────┐
      │                 │
   ┌──▼────┐  ┌───────┐ │
   │Thread1│  │Thread2│ │... (worker pool)
   │ CPU   │  │ CPU   │
   │ work  │  │ work  │
   └───────┘  └───────┘
```

### Where You See It

- **Node.js**: One event loop (single-threaded for JavaScript); `libuv` thread pool for file I/O and DNS.
- **Tokio (Rust)**: Multi-threaded variant has one event loop per core + a separate blocking thread pool.
- **Boost.Asio (C++)**: Proactor pattern; you can spawn multiple threads on the same I/O multiplexer.
- **JavaScript in the browser**: Event loop on the main thread, Web Workers as the worker pool.

### The Tradeoff

**Pros:**
- Simple mental model: loop + threads.
- I/O-bound tasks scale (one thread → millions of concurrent connections).
- Pool size is predictable (you control the number of worker threads).

**Cons:**
- Requires explicit separation: I/O work goes in the loop, CPU work goes in the pool.
- Synchronization overhead when moving work between loop and pool.
- If the loop itself does CPU work, I/O stalls.

### Recognition Checklist

- Is there a single thread running an event loop?
- Is there a separate pool of threads for blocking operations?
- Does the language distinguish between "I/O-bound" and "CPU-bound" code?

If yes to all three, you are looking at event loop + worker pool.

---

## 92.2 M:N Scheduler (User-Space Tasks on OS Threads)

### The Pattern

**N** user-space tasks (goroutines, green threads, tasks) are multiplexed onto **M** OS threads. The runtime scheduler decides which task runs on which thread. Tasks are lighter than threads; you can spawn millions.

```
┌────────────────────────────────────┐
│ Runtime Scheduler                  │
├────────────────────────────────────┤
│ runq: [task1, task3, task5, ...]   │
│ (ready queue)                      │
└──────────────┬─────────────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
  ┌──▼──┐  ┌──▼──┐  ┌──▼──┐
  │ OS  │  │ OS  │  │ OS  │
  │Thr1 │  │Thr2 │  │Thr3 │
  └─────┘  └─────┘  └─────┘
  (M << N)
```

### Where You See It

- **Go (goroutines)**: ~2,000 goroutines per thread is typical. Runtime balances load.
- **Erlang BEAM**: Erlang processes are user-space tasks. The BEAM scheduler assigns them to scheduler threads (one per CPU core by default).
- **Older Java green threads** (pre-1.3): Thousands of Java threads on a few OS threads. Deprecated because native threads won.
- **Kotlin (Coroutines)**: Many suspendable tasks on a pool of threads.

### The Tradeoff

**Pros:**
- Lightweight concurrency: task overhead is bytes, not megabytes.
- Language controls scheduling: predictable behavior, no OS preemption surprises.
- Massive scale: millions of tasks on one machine.

**Cons:**
- Tasks cannot use true parallelism: if you have M threads, at most M tasks run in parallel, even on a 64-core machine.
- Language must implement a full scheduler: complex, potential for bugs.
- Load balancing across cores is non-trivial: if one core's scheduler thread is busy, others may be idle.

### Recognition Checklist

- Are lightweight tasks (coroutines, goroutines) the primary concurrency unit?
- Are there fewer of these tasks than OS threads you'd expect?
- Does the language runtime manage thread-to-task mapping?

If yes to all, you are looking at M:N scheduling.

---

## 92.3 Per-Process Heap + Isolated Collection

### The Pattern

Each process (or each major component) has its own heap and garbage collector. No shared mutable state between processes. Collection runs per-process, not globally. Coordination happens via message passing.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Process A    │  │ Process B    │  │ Process C    │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ Heap A       │  │ Heap B       │  │ Heap C       │
│ GC A         │  │ GC B         │  │ GC C         │
│ (isolated)   │  │ (isolated)   │  │ (isolated)   │
└──────────────┘  └──────────────┘  └──────────────┘
                   │         │
                   └────┬────┘
                    Message
                    Passing
```

### Where You See It

- **Erlang BEAM**: Each Erlang process has its own heap. No global GC pause. Collection is per-process.
- **Some microkernel systems**: Each server is a separate process with isolated memory.
- **Shared-nothing database architectures**: Each shard has its own heap; coordination via RPCs.

### The Tradeoff

**Pros:**
- No global GC pause: if process A is collecting garbage, process B is unaffected.
- Fault isolation: a crash in one process does not crash all.
- Scalability: each process can be GC'd independently.

**Cons:**
- Data copying on message passing: to send an object from A to B, it must be copied (or serialized).
- Higher memory overhead: each process needs its own heap and GC bookkeeping.
- Harder to share: no pointers across process boundaries.

### Recognition Checklist

- Are processes/components isolated, with separate heaps?
- Is the only communication mechanism message passing?
- Does garbage collection happen per-process, not globally?

If yes to all, you are looking at per-process isolation.

---

## 92.4 Generational + Concurrent Garbage Collection

### The Pattern

Objects are split into **generations** by age. Young objects are collected frequently and quickly. Old objects are collected rarely and carefully. The collector runs **concurrently** with the main program, not stopping it completely.

```
┌─────────────────────────────────┐
│ Heap                            │
├──────────────┬──────────────────┤
│ Young Gen    │ Old Gen          │
│ (frequent    │ (rare collection)│
│  collection) │                  │
│              │                  │
│ ████████     │ ██████████████   │
│              │                  │
└──────────────┴──────────────────┘
       │ collection = fast
       │ (most objects die young)
       │
   Promotion
   (rare)
       │
       └─→ Old Gen
```

### Where You See It

- **Java G1GC, ZGC, Shenandoah**: Generational collectors; concurrent marking and compaction.
- **V8 (JavaScript)**: Young generation (fast, frequent), old generation (slow, rare).
- **.NET**: Generational GC with concurrent collection option.
- **Go**: Concurrent mark-and-sweep (not generational, but concurrent).

### The Tradeoff

**Pros:**
- Young collection is very fast: most allocations are temporary.
- Concurrent marking: does not stop the world (much).
- Scales to large heaps: old generation is huge but rarely collected.

**Cons:**
- Write barriers: need to track pointers from old objects to young objects, adding overhead.
- Floating garbage: objects created during concurrent marking are not collected in that cycle (wasted).
- Heap fragmentation if not careful.

### Recognition Checklist

- Are objects split by age (young and old generations)?
- Are young objects collected frequently, old objects rarely?
- Does collection happen concurrently, without stopping the program entirely?

If yes to at least two, you are looking at generational + concurrent GC.

---

## 92.5 Reference Counting + Cycle Collector

### The Pattern

Objects track how many references point to them. When the count drops to zero, the object is freed immediately. A separate **cycle collector** periodically hunts for reference cycles (A → B → A) and breaks them.

```
┌─────────────┐
│ Object A    │
│ refcount=2  │  (held by B and by a local variable)
└──────┬──────┘
       │
       ├─→ B's field
       └─→ local var

(when local var goes away, refcount=1)
(when B is freed, refcount=0)
(Object A is freed immediately)

But:
┌────────────┐
│ Object A   │ ─────┐
└────────────┘      │
     ▲              │
     │              │
     └──────────────┘
(A points to itself, cycle, refcount never 0)
(Cycle collector must break it)
```

### Where You See It

- **CPython**: Reference counting for most objects; cycle collector runs periodically.
- **PHP**: Reference counting + cycle collector.
- **Swift**: Reference counting by default (though with some optimizations).

### The Tradeoff

**Pros:**
- Predictable destruction: objects are freed immediately when the count hits zero.
- Cache-friendly: refcount updates are local to the object.
- No full-world GC pause: cycles are handled asynchronously.

**Cons:**
- Refcount overhead: every assignment that changes refcount is a write.
- Cycle collector is complex and must pause the world (briefly) to break cycles.
- Synchronization nightmare in multithreaded code: refcount updates must be atomic or locked.

### Recognition Checklist

- Are objects freed immediately when no references exist?
- Is there a separate cycle collector that runs periodically?
- Is refcount update overhead visible in profiling?

If yes to all, you are looking at reference counting + cycle collection.

---

## 92.6 Tiered JIT Compilation

### The Pattern

Code starts as **interpreted bytecode** (fast startup). Hot code is compiled to **baseline JIT** (quick compilation, moderate optimization). Very hot code is compiled by an **optimizing JIT** (slow compilation, aggressive optimization, with deoptimization safety nets).

```
Entry Point: Bytecode Interpreter
  │
  ├─ Task runs                 (fast startup)
  │
  ├─ Some code runs hot (5k+ invocations)
  │
  └─→ Baseline JIT kicks in    (compile, < 100ms)
      │
      ├─ Optimized code runs   (10-20x faster)
      │
      ├─ Some code runs *very* hot (100k+ invocations)
      │
      └─→ Optimizing JIT kicks in (aggressive compile, 1s+)
          │
          ├─ Speculated code    (50-100x faster)
          │
          └─→ Assumption broken (e.g., type changes)
              │
              └─→ Deoptimization: fall back to interpreter
```

### Where You See It

- **Java HotSpot**: C1 (baseline) → C2 (optimizing JIT) → deoptimization.
- **V8 (JavaScript)**: Sparkplug (baseline) → TurboFan (optimizing JIT).
- **.NET**: Tier0 (jitted) → Tier1 (optimized jitted).

### The Tradeoff

**Pros:**
- Instant startup: interpreted bytecode runs immediately, no compilation delay.
- Peak performance: aggressive optimization after warmup matches C++.
- Adaptive: JIT specializes based on observed types and values.
- Safe fallback: deoptimization lets the JIT take risks.

**Cons:**
- Warmup time: peak performance takes minutes for some workloads.
- Non-deterministic: what is fast now might be slow after a GC or code change.
- Complexity: JIT compiler is 100k+ lines of code.

### Recognition Checklist

- Does code start as interpreted bytecode?
- Is there a two-stage (or three-stage) compilation pipeline?
- Are there compilation thresholds (e.g., "compile after 10k invocations")?

If yes to all, you are looking at tiered JIT.

---

## 92.7 Stackless Coroutines + Async Runtime

### The Pattern

Functions that can suspend (`async fn`) do not use the call stack. Instead, they are **state machines** compiled to heap-allocated objects. An **async runtime** (event loop + scheduler) drives these state machines forward.

```
async fn read_file() {
  data = await read_from_disk()  // suspend point 1
  result = await process(data)    // suspend point 2
  return result
}

Compiled to:

struct read_file_state {
  state: int         // which await point?
  data: Vec<u8>      // local variables across awaits
  
  fn poll() {        // called by runtime
    match state {
      0 => { issue_read(); state = 1 }
      1 => { data = result; issue_process(); state = 2 }
      2 => { return Pending(result) }
    }
  }
}
```

### Where You See It

- **Rust (Tokio, async-std)**: Coroutines as state machines.
- **Python (asyncio)**: Async functions compile to state machines (simplified).
- **C++20**: Coroutines are stackless (heap-allocated state).
- **JavaScript Promises/async-await**: Internally state machines.

### The Tradeoff

**Pros:**
- Memory-efficient: state machine is bytes, not a megabyte stack.
- Scales to millions of concurrent tasks.
- Can suspend at arbitrary points (await).

**Cons:**
- State machine overhead: local variables live on the heap.
- Requires an async runtime: you cannot use blocking I/O directly.
- "Colored" functions: if you call an async function, your entire call chain is async.
- Debugging is harder: no stack trace, just a state machine.

### Recognition Checklist

- Are coroutines stackless (no OS stack per coroutine)?
- Are they compiled to state machines?
- Is there a dedicated async runtime driving them?

If yes to all, you are looking at stackless coroutines + async runtime.

---

## 92.8 Mapping Terminology Across Languages

Every language names these concepts differently. Here is a translation table:

| Concept | Go | Rust | Erlang | Python | Java | C++ |
|---------|-----|------|--------|--------|------|-----|
| **Lightweight task** | Goroutine | Task / Future | Process | Coroutine | Virtual Thread | Coroutine |
| **Scheduler** | Runtime scheduler | Tokio / async-std | BEAM scheduler | asyncio loop | Virtual Thread scheduler | Event loop (external) |
| **Task suspension** | Channel send/recv | `.await` | `receive` | `await` | Virtual Thread `.park()` | `co_await` |
| **Task communication** | Channel | Message (via async) | Message passing | Queue / Channel | Shared memory | Channel (external) |
| **M:N threading** | Native (M:N) | Library-level | Native (M:N) | No (1:1 + GIL) | Library-level (virtual) | Library-level |

**The key insight:** A goroutine, a Rust Future, an Erlang process, and a Python asyncio coroutine are *fundamentally the same thing*: a unit of execution that can be suspended and resumed. They differ in how they are scheduled and what API they expose, but the internals are similar.

---

## 92.9 Worked Example: Classifying Three Runtimes

### Example 1: Bun (JavaScript)

Bun is a JavaScript runtime (like Node.js, but faster).

**Patterns:**
- **Execution**: Tiered JIT (V8 engine).
- **Concurrency**: Event loop + worker pool.
- **GC**: Generational concurrent GC (borrowed from V8).

**Implication**: Bun starts fast (bytecode interpretation), scales to thousands of concurrent I/O tasks (event loop), and has good peak performance (V8 JIT). For CPU-bound work, you spawn workers.

### Example 2: Deno (JavaScript/TypeScript)

Deno is also a JavaScript runtime, but with a different philosophy (better standard library, URL imports).

**Patterns:**
- **Execution**: Tiered JIT (also V8).
- **Concurrency**: Event loop + Tokio (Rust async runtime underneath).
- **GC**: Generational concurrent GC (V8).

**Implication**: Deno is almost identical to Bun under the hood. The difference is architectural: Deno's event loop is tied to Tokio (a Rust library), while Node.js uses libuv. Both scale to thousands of connections.

### Example 3: Pony

Pony is a systems language designed for actor-based concurrency.

**Patterns:**
- **Concurrency**: M:N scheduler (actors are lightweight tasks).
- **GC**: Per-actor generation + concurrent collection (no global pause).
- **Memory**: Reference counting + cycle collector (for some versions).
- **Execution**: AOT compiled to native code.

**Implication**: Pony trades peak performance (no JIT) for predictability (no GC pauses) and massive concurrency (millions of actors). Ideal for distributed systems; poor for tight numerical loops.

---

## 92.10 When You See A New Runtime, Ask

When you encounter an unfamiliar runtime, ask these questions in order:

1. **How is the code executed?**
   - Interpreted? Bytecode? JIT? AOT?
   - Timeline: instant startup or warmup delay?

2. **How is concurrency expressed?**
   - Event loop? Goroutines? Threads? Actors?
   - Can you scale to thousands of concurrent tasks, or dozens?

3. **How is memory managed?**
   - GC? Reference counting? Manual?
   - Are there pause times?

4. **What is it optimized for?**
   - High throughput? Low latency? Startup time? Simplicity?
   - What workload makes it shine?

5. **What is it bad at?**
   - CPU-bound compute? Tight memory budgets? Hard real-time?
   - Where would you reach for something else?

**Example:** "WebAssembly (WASM) + WASI"
1. AOT compiled to bytecode, then JIT'd in the browser or VM.
2. Concurrency: currently minimal; async/await is new.
3. Memory: linear memory (one flat 32/64-bit address space), with manual allocation or a managed runtime inside the WASM module.
4. Optimized for: portable, sandboxed code that runs in browsers and edge servers.
5. Bad at: high-frequency trading (latency), massive parallel compute (no threads yet).

---

## 92.11 Common Tradeoffs

| Pattern | Pros | Cons | Latency | Throughput | Memory |
|---------|------|------|---------|-----------|--------|
| **Event loop + pool** | Scales I/O; simple. | Separates I/O and CPU work. | Good (no blocking) | Good (async) | Good (no thread overhead) |
| **M:N scheduler** | Lightweight; massive scale. | Language-level complexity. | Varies (fair scheduling) | Fair (load balancing) | Excellent |
| **Per-process heap** | No global pause; isolation. | Data copying; overhead. | Excellent (isolated GC) | Good (parallel GC) | Poor (per-process overhead) |
| **Generational GC** | Young collection is fast. | Write barriers; complexity. | Good (short pauses) | Good (fast young gen) | Good (on-heap) |
| **Refcount + cycle** | Predictable destruction. | Atomic overhead; cycles. | Good (immediate) | Fair (atomic cost) | Fair (cycle overhead) |
| **Tiered JIT** | Instant startup + peak perf. | Warmup time; complexity. | Fair (post-warmup) | Excellent (optimized) | Fair (JIT overhead) |
| **Stackless coroutines** | Memory-efficient; scales. | State machine overhead. | Good | Excellent | Excellent |

---

## 92.12 Common Misconceptions

**"Go is fast because goroutines."**

False. Go is fast because AOT compilation and a simple concurrency model. Goroutines are lightweight, but they do not make code faster per se. They allow more concurrency on fewer resources.

**"JIT is always faster than AOT."**

False. JIT can specialize on runtime data, making peak performance excellent. But AOT has zero warmup, better predictability, and lower memory overhead. For short-lived programs, AOT is faster.

**"Garbage collection pauses are always bad."**

False. Modern concurrent GC has sub-millisecond pauses. For most services (web servers, APIs), sub-millisecond pauses are unnoticeable. For hard real-time (robotics, high-frequency trading), they are unacceptable.

**"More threads = faster."**

False. Threads are heavy. On a 4-core machine, 4 threads saturate the CPU. 100 threads cause context-switch overhead. For I/O, async is better than threads.

**"Async is always better than threads."**

False. Async scales to thousands of I/O tasks. But each I/O library call must be async (deep in the call stack). Threads work with blocking I/O. Choose based on your workload.

**"Reference counting is slower than GC."**

Depends. In single-threaded code, refcount is very fast (no global pause, immediate freeing). In multithreaded code, refcount atomic operations are a bottleneck; GC is faster. CPython uses refcount because the GIL serializes access.

---

## 92.13 Exercises

**1. Pattern identification.** Pick three programming languages or runtimes you have not used deeply (e.g., Nim, Scala, Kotlin, Clojure). For each, determine:
   - Execution model (interpreted / bytecode / JIT / AOT)?
   - Concurrency pattern (event loop / M:N / threads / actors)?
   - GC strategy (generational / reference count / concurrent)?
   
   Write one paragraph explaining what workload it is optimized for.

**2. Benchmark tradeoff.** Choose two runtimes from exercise 1. Write a simple program (e.g., read 10k files, sum numbers) and measure:
   - Startup time.
   - Peak throughput (tasks per second).
   - Memory usage.
   - Variance (how predictable are the numbers?).
   
   Explain the tradeoffs you observe.

**3. Design a runtime.** You are designing a new runtime for IoT devices with tight memory budgets (<50 MB) and hard real-time constraints (p99 latency < 10 ms). Choose points on each axis:
   - Execution: interpreted / bytecode / JIT / AOT?
   - Concurrency: threads / async / goroutines?
   - GC: manual / reference count / incremental / concurrent?
   - Abstraction: high or low?
   
   Justify each choice.

**4. Predict performance.** Without running code, predict the behavior of three runtimes (e.g., CPython, PyPy, Go) on:
   - A loop that counts primes to 1 million.
   - A server handling 1,000 concurrent I/O requests.
   - Memory overhead for spawning 100,000 tasks.
   
   Run the code and compare.

**5. Migration problem.** You have a Python service (asyncio, 10 core machines, handling 5k concurrent connections). It is CPU-bound and taking 500 ms per request. You must move to another language. Using the pattern framework, which patterns would you choose, and why? What would you expect to measure?

**6. Pattern convergence.** Find two modern languages (released in the last 5 years) that borrowed concurrency patterns from each other. Example: Java virtual threads borrowed from Go goroutines. Explain what each language learned.

---

## 92.14 Summary

Every modern runtime is built from a small set of patterns: event loop + worker pool for I/O concurrency, M:N schedulers for lightweight tasks, generational GC for fast young collection, reference counting for predictable destruction, tiered JIT for adaptive performance, and stackless coroutines for memory efficiency.

These patterns are not rigid. Most runtimes combine them: Go uses M:N scheduling + concurrent GC. Java uses tiered JIT + generational GC. JavaScript uses event loop + tiered JIT.

To understand any runtime, ask: Which patterns does it use? What tradeoffs does it make? The answers tell you what it is optimized for and where it will disappoint you.

You cannot build a language that is fast at everything and simple at everything. Every design choice buys one thing and costs another. Understanding the patterns lets you read between the lines, see what a language is prioritizing, and choose the right tool for your job.

---

> **[← Previous: How To Learn Any Programming Language](01-how-to-learn-any-language.md)** · **[↑ Part 9](README.md)** · **[Next: Mapping Framework Concepts Across Languages →](03-mapping-framework-concepts.md)**
