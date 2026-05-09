# Chapter 85 — Profiling

A profiler is a microscope. Different microscopes show different cells, at different magnifications, under different lighting. A light microscope reveals structure invisible to the naked eye but cannot reach the atomic scale. An electron microscope goes further but requires a vacuum chamber and kills the sample. A confocal microscope illuminates only one plane of focus. Knowing which microscope to reach for, and how to read what you see, is half of effective performance work.

Most engineers approach profiling as an afterthought: the code is slow, so they run a profiler, see a list of hot functions, and optimize the first one. This is like using an electron microscope to look for a stain on a slide. You are using the tool correctly but asking it the wrong question. Profiling is not an endpoint; it is a systematic practice. You profile to answer a specific question. You read the output to find not the hottest function, but the bottleneck — the constraint that, when removed, yields measurable improvement.

This chapter teaches you to read profiling data like an engineer: to distinguish signal from noise, to know when a profiler is lying, and to build profiling into your workflow as the foundation of iterative optimization.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish between sampling-based and instrumentation-based profiling, and understand the tradeoffs in accuracy, overhead, and recompilation cost.
2. Use `perf record` and `perf report` (or equivalent tools on your platform) to capture and analyze CPU profiles, including call-stack attribution and statistical validity.
3. Interpret flame graphs: identify hot spots, read stack depth and width, spot unexpected function widths, and diagnose missing inlining or heavy abstraction.
4. Profile memory allocation: use `heaptrack`, `massif`, or jemalloc profilers to track allocation sites, detect unexpected churn, and quantify memory overhead.
5. Use tracing tools (`strace`, `dtrace`, eBPF) to observe exact event timings when statistical sampling is insufficient.
6. Recognize common profiler artifacts: skid (sample attribution error), JIT-induced blindness (inlined functions not visible), and wall-clock vs CPU-time confusion.
7. Apply a profiler-first workflow: set a latency/throughput target, profile the baseline, identify the dominant cost, fix it, re-profile, and iterate.
8. Understand language-specific profiling (Java's async-profiler and JFR, Python's py-spy, Go's pprof) and when to use each.

---

## Sampling vs Instrumentation: The Central Tradeoff

Every profiler answers the question "where does time go?" in one of two ways: by sampling the program state periodically, or by instrumenting (adding hooks to) every function entry and exit.

### Sampling Profilers

A sampling profiler sets a hardware timer to fire N times per second (typically 100–1000 Hz). When the timer fires, the profiler captures the call stack — which function is running, and which functions called it. After running for a few seconds, the profiler has captured thousands of samples, each representing a snapshot of the program state.

**Accuracy:** Sampling is statistical. If a function consumes 50% of CPU time, it will appear in roughly 50% of samples. With 1000 samples, the margin of error is ±3% (standard error = sqrt(p × (1-p) / n)). More samples → tighter error bars.

**Overhead:** A sampling profiler has minimal overhead. Capturing a stack trace is a few microseconds. If you sample 1000 times per second, that is 1 ms of profiling overhead per second of wall-clock time — negligible.

**Recompilation:** None needed. You can profile a release binary, with optimizations enabled, as-is.

**What you lose:** Sampling cannot count exact occurrences. You cannot say "malloc was called 10,423 times" — you can say "malloc was in the call stack in ~15% of samples, so it probably consumed ~15% of time." This is usually fine for performance work, where you care about hotspots, not exact counts.

Example (Linux):
```bash
perf record -g -F 99 ./my_program  # Sample 99 times per second, capture stacks
perf report                        # Interactive stack browser
```

### Instrumentation Profilers

An instrumentation profiler inserts code (either compile-time or runtime) to emit an event on every function entry and exit. After running, you have an exact timeline: function A started at 1000 µs, B started at 1020 µs, B returned at 1100 µs, A returned at 1150 µs.

**Accuracy:** Perfect, by definition. You know exactly which functions were called and when.

**Overhead:** Extremely high. Every function call incurs the cost of a profiling call. For short functions that are called millions of times, this overhead can slow the program by 10–100×. And that slowdown changes timing, so you are profiling a distorted version of the real program.

**Recompilation:** Usually required. You compile with instrumentation flags (e.g., `-finstrument-functions`), which inserts the profiling hooks.

**When to use:** When you need exact call counts or when you have a few hot functions that you want to analyze in detail. Not for whole-program profiling.

Example (compile-time instrumentation):
```bash
g++ -finstrument-functions mycode.cpp -o mycode
# Generates trace of every function entry/exit with timing
```

**The practical choice:** Use sampling for initial exploration and call-graph construction. Use instrumentation only when you have a specific hypothesis and need exact timing.

---

## CPU Profiling with `perf` and Flame Graphs

`perf` (Linux performance counters) is the standard sampling profiler for C and C++. It has minimal overhead and works on release binaries without modification.

### Basic Workflow

```bash
# Record a profile: sample every ~10 ms, capture full call stacks (-g)
perf record -g -F 100 ./my_program input.dat

# This creates perf.data, a binary trace file (~1–100 MB)

# View the report: interactive stack-tree browser
perf report

# Or dump to text (easier for scripts)
perf script > trace.txt
```

In `perf report`, navigate with arrow keys:
- Press Enter on a function to expand its callers/callees
- '+' to expand subtrees
- 'a' to annotate (show source code with instruction-level costs if debug symbols are available)

### Reading the Output

A `perf report` output looks like:

```
   41.23%  my_program  libc.so.6        [.] __libc_malloc
   |--98.5%-- complex_algorithm
   |          main
   |--1.2%-- alloc_temp_buffer
                debug_print
```

This says: malloc consumed 41.23% of samples. Of those, 98.5% were called from `complex_algorithm` (called from `main`). The remaining 1.2% were called from `alloc_temp_buffer` (in a debug print path).

The key insight: the wide `malloc` bar does not mean malloc is the bottleneck. It means your algorithm calls malloc frequently. The bottleneck might be:
- Algorithmic (you are calling malloc too often — fix the algorithm)
- Allocator contention (malloc itself is slow — switch to a faster allocator like jemalloc)
- Work inside malloc (malloc does expensive initialization — use a custom allocator or object pool)

### CPU vs Off-CPU: The Hidden Bottleneck

`perf record` shows time when the program is **running on the CPU**. It does not show time when the program is **waiting** — blocked on I/O, waiting for a lock, sleeping, or preempted by the OS scheduler.

This is a critical blind spot. Often, the slowest programs are not consuming much CPU at all; they are blocked waiting for something. A web service that is I/O-bound (waits for database queries) will look like it has no hot functions — the CPU is barely used. But the latency is high.

**Off-CPU profiling** captures the opposite: when and why your program is not running. On Linux, this uses eBPF tracing:

```bash
# Requires kernel 4.17+, eBPF must be enabled
# See https://www.brendangregg.com/offcpuanalysis.html
offcputime -f 99 my_program > off_cpu.txt
```

Off-CPU profiling reveals:
- Lock contention (threads blocked on futex or pthread_mutex)
- I/O waits (read/write syscalls blocking)
- Page faults (memory pressure causing swaps)
- Scheduler delays (thread was runnable but not scheduled)

For a slow web service, off-CPU profiling often reveals that the actual bottleneck is not CPU-bound code at all — it is a slow database query, or lock contention in the connection pool, or waiting for a network response.

### Flame Graphs: Visual Profiling

A **flame graph** is a visual representation of profiling data. The x-axis represents sample count (wider = more samples, i.e., more CPU time). The y-axis represents call-stack depth. Each box is a function. The colors are usually arbitrary (sometimes indicate module: kernel, libc, your code).

To generate a flame graph:

```bash
perf record -g ./my_program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
# Open flame.svg in a browser
```

**How to read a flame graph:**

1. **Wide boxes = hot code.** If you see a function that spans the entire width, it consumed most CPU time.

2. **Stack depth.** A deep tower (many nested boxes) indicates deep call stacks — heavy abstraction layers. A shallow tower indicates few layers of indirection.

3. **Unexpected width.** If a function is wide but does not look expensive (e.g., `_memmove` or `memcpy`), that is a signal that **you are copying too much data.** The function itself is fine; the problem is the call pattern.

4. **Suspiciously narrow functions.** A function that should be doing a lot of work but is barely visible might have been inlined by the compiler. Inlining is usually good, but if it hides the real work, you are not getting a clear picture.

5. **Level differences.** Functions at different stack depths can reveal call patterns. If `malloc` appears directly under `main` and also deep in the stack, it means malloc is called from both high-level and low-level code — a sign of fragmented allocation.

**Case example:**

A flame graph showed a wide spike in `std::sort` at 35% of CPU time. Initially, the team thought the sort algorithm was slow. But zooming in revealed that the real bottleneck was the comparator function — which was doing string comparisons on keys that were not themselves strings, but structs containing strings. The comparator was constructing temporary strings on every comparison, and those constructions (allocation, copying) consumed 90% of the sort's time. The fix: use a custom comparator that compared the struct fields directly, avoiding string construction. Speedup: 8×.

---

## Memory Profiling

CPU profiling tells you where time goes. Memory profiling tells you where memory goes — both peak usage and allocation rate.

### Heap Profiling with `heaptrack`

`heaptrack` is a heap profiler for Linux that works with C and C++ programs without instrumentation. It wraps malloc/free and captures every allocation.

```bash
heaptrack ./my_program input.dat
# Generates heaptrack.my_program.<pid>.gz

heaptrack_gui heaptrack.my_program.<pid>.gz
# Opens an interactive GUI
```

The output shows:

- **Total allocations:** How many times did malloc get called?
- **Peak heap size:** What is the maximum bytes allocated at once?
- **Allocation rate:** Bytes per second. High rates indicate memory churn.
- **Top allocation sites:** Which lines of code allocated the most memory?

Each allocation is attributed to its call stack. If your program allocates 1 GB total, heaptrack shows you which function called malloc, and what called that function, recursively up to main.

**Detecting churn:** A program that allocates 100 MB/second (on a system with 16 GB RAM) might still have peak usage of only 100 MB — but the allocator is thrashing. Every allocation triggers a syscall or page fault. The CPU overhead of allocation can dominate. Use heaptrack to measure allocation rate:

```
Allocations per second: 500,000
Bytes allocated per second: 250 MB/s
```

If this is high and your program's working set is much smaller, you have a churn problem. Solutions:
- Use an object pool (pre-allocate a large chunk, reuse objects)
- Use an arena allocator (allocate many objects together, free them all at once)
- Use a custom allocator (jemalloc, mimalloc) that batches allocations
- Switch to stack allocation or static allocation where possible

### Valgrind Massif

Valgrind's `massif` tool is more intrusive but gives detailed memory graphs over time:

```bash
valgrind --tool=massif ./my_program input.dat
# Generates massif.out.<pid>

ms_print massif.out.<pid>
```

Output shows a timeline of heap usage, with a bar chart and allocation sites at each peak. Use this when you suspect memory leaks or runaway memory growth.

### jemalloc's Built-In Profiler

If your program uses jemalloc (a faster, more scalable malloc), you can enable profiling at compile-time:

```bash
g++ mycode.cpp -o mycode -ljemalloc -DJEMALLOC_ENABLE_CXX -UJEMALLOC_STATS
# At runtime:
MALLOC_CONF=prof_leak=true ./mycode
```

jemalloc writes a detailed profile to `/tmp/jeprof.*`. This is useful for detecting memory leaks in production code.

### Memory vs Allocation Rate

Do not confuse peak memory (important for system capacity planning) with allocation rate (important for CPU overhead). A program with peak usage of 1 GB is fine if that is 1 GB total. But a program that allocates 10 GB per second is thrashing, even if peak usage is only 100 MB. Measure both.

---

## Tracing: Exact Event Times

Sampling profilers are great for identifying hot spots, but sometimes you need exact timing. When did system call X happen? How long did it block? What was the call stack?

### `strace`: System Call Tracing

`strace` intercepts all system calls and logs them with timing:

```bash
strace -c -e trace=read,write,open,close ./my_program
# Shows a summary of syscall counts and time spent in each
```

Output:
```
% time     seconds  usecs/call     calls    errors name
  50.23       1.234      1234         1            read
  30.12       0.741        45        16            write
  ...
```

Use `strace` when you suspect your program is doing too many syscalls, or syscalls are taking a long time (sign of I/O contention).

Example: A program that opens a file in a tight loop:
```bash
strace -c ./my_program
% time     seconds  usecs/call     calls    errors name
  80.05       8.234      100        82         open    # 100 µs per open!
```

The fix: batch the file operations, or open the file once and keep it open.

### eBPF Tracing: In-Kernel Profiling

eBPF (Extended Berkeley Packet Filter) is an in-kernel virtual machine that can run profiling programs with minimal overhead. Tools like `bcc` and `bpftrace` let you write custom profilers:

```bash
# Trace malloc calls and their latencies
bpftrace -e 'u:libc:malloc { @latency = hist(nsecs - @start[tid]); @start[tid] = nsecs; }'
```

This is powerful for investigating lock contention, cache effects, and I/O patterns, but requires kernel support and Linux expertise.

### Distributed Tracing: Service-Level Profiling

For systems with multiple services (microservices), use distributed tracing tools like Jaeger or Tempo. These propagate a trace ID across service boundaries and collect timing information from each service.

A trace might show:
- Request entered service A at 1000 µs
- Service A called service B at 1020 µs (overhead: 20 µs)
- Service B responded at 2500 µs (latency: 1480 µs)
- Service A finished at 2600 µs

This reveals bottlenecks across the system, not just within a single service.

---

## Language-Specific Profiling

Different languages and runtimes have different profiling tools, optimized for their execution model.

### Java: async-profiler and JFR

Java programs are compiled to bytecode, which is JIT-compiled to native code at runtime. This makes profiling tricky — the JIT might inline functions, or optimize away code, making it invisible to the profiler.

**async-profiler** is a sampling profiler that understands the JVM:

```bash
jps  # Find the Java process ID
async-profiler record -d 30 -e cpu <pid>
async-profiler dump <pid> > profile.html
```

**Java Flight Recorder (JFR)** is a built-in profiler that captures events (allocations, lock waits, GC pauses) with low overhead:

```bash
jcmd <pid> JFR.start
jcmd <pid> JFR.dump filename=profile.jfr
jfr view profile.jfr
```

### Python: py-spy and Scalene

Python is slow, and GIL contention is often the bottleneck. **py-spy** is a sampling profiler that works on running processes without instrumentation:

```bash
sudo py-spy record -o profile.svg ./my_script.py
```

**Scalene** is a newer profiler that measures CPU, memory, and GPU time separately:

```bash
pip install scalene
scalene my_script.py
```

### Go: pprof

Go includes a built-in profiler (`runtime/pprof`). To use it:

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // your code
}
```

Then:
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### Rust: pprof and dhat

**pprof** is a Rust binding to the perf profiler:

```toml
[dependencies]
pprof = { version = "0.13", features = ["flamegraph"] }
```

```rust
let guard = pprof::ProfilerGuard::new(100).unwrap();
// your code
guard.report().build().unwrap().flamegraph("profile.svg").unwrap();
```

---

## When Profilers Lie

Profilers are tools, and tools have limitations. Understanding when they mislead is crucial.

### Skid: Sample Attribution Error

When a sampling profiler captures a stack, the CPU may have executed a few instructions past the interrupt point. Those instructions are attributed to the function that was running at sample time, but the real cost was earlier. For instance, a malloc might have returned, and the next function executed a few instructions, when the sample fired. The cost of malloc is underestimated.

Skid is usually small (< 10% error) but can be significant in tight loops with tiny functions.

### JIT Inlining: The Invisible Function

JIT-compiled languages (Java, JavaScript, C#) inline functions aggressively. When the profiler samples code, an inlined function has no separate stack frame — it is invisible. A profiler might show that your hot loop calls function A, which is 80% of time. But if A is inlined into the caller, the real cost is in the loop body, and you are looking at the wrong function.

Mitigation: Use language-specific profilers (async-profiler for Java) that understand inlining.

### Wall-Clock vs CPU Time

A process can be running but not consuming CPU (blocked on I/O, waiting for a lock, sleeping). `perf` measures CPU time — the time the process actually spent on the CPU core. But wall-clock time (real time) includes time blocked.

If a program runs for 10 seconds of wall-clock time but only 2 seconds of CPU time, it is spending 80% blocked. A CPU profiler will not show why. You need a blocking-time profiler or a trace tool.

### Profiler Overhead Distorts Timing

Instrumentation profilers are accurate about what they measure, but their overhead is high. The overhead itself consumes CPU, and it can change timing significantly. A function that normally runs in 100 ns might run in 1 µs under instrumentation — a 10× slowdown. If you are profiling a multi-threaded program, the overhead might change lock contention patterns, masking the real bottleneck.

Sampling profilers have minimal overhead (< 5%), so they are more truthful about timing.

### Missing Debug Symbols

If your binary has no debug symbols, the profiler cannot map instruction addresses to function names. You will see hex addresses instead of function names. Always compile with `-g` for profiling:

```bash
g++ -O2 -g mycode.cpp -o mycode  # -g adds debug symbols, -O2 optimizes
perf record -g ./mycode
```

---

## A Profiler-First Workflow

Effective performance engineering is not heroic optimization of guessed-at hotspots. It is disciplined, iterative, measurement-driven work.

### The Loop

1. **Set a target.** What is the latency or throughput goal? p99 < 100 ms? 10,000 requests/sec?

2. **Measure the baseline.** Run your program with a representative input. Measure wall-clock time, CPU time, memory peak. Record the profiling data.

3. **Identify the dominant cost.** Run a profiler. Look at the flame graph or top-10 functions. Which single change would have the biggest impact?

4. **Fix the dominant cost only.** Do not optimize every function. Fix the one that appears to be the bottleneck. Make one change.

5. **Re-measure.** Run the same profiler again with the same input. Did latency improve? By how much?

6. **Iterate.** If you have not hit the target, go to step 3.

### Example: Optimizing a String Search

Baseline profile shows:
```
35% memcpy (called by string::operator=)
25% string::find (comparison loop)
15% malloc (constructing temporary strings)
20% other
```

Naive guess: "memcpy is the bottleneck, optimize it." Wrong. `memcpy` is being called because you construct temp strings. The bottleneck is the algorithm.

**Iteration 1:** Rewrite the algorithm to avoid temporary string construction. Use string_view instead.

Result:
```
18% memcpy
45% custom_find_loop (new comparison logic)
8% malloc
25% other
```

`memcpy` is now only 18%, so you fixed the allocation cost. But `custom_find_loop` is now dominant. This is progress — you found the real bottleneck.

**Iteration 2:** Optimize `custom_find_loop` with SIMD (SSE string comparison).

Result:
```
8% memcpy
12% custom_find_loop (now vectorized)
4% malloc
70% other
```

You have now eliminated the low-hanging fruit. The "other" category suggests the bottleneck is no longer in string search; it is elsewhere (I/O? lock contention?). You have hit diminishing returns.

### When to Stop

Stop optimizing when:
- You have hit the target (p99 < 100 ms)
- The profiler shows no clear next bottleneck (work is evenly distributed)
- The marginal gain per hour of work is too small to justify
- You are optimizing code that is not on the critical path (runs 1% of the time)

A common mistake: spending a week to optimize a function that is 2% of runtime. That can be a 50% speedup of that function, but only a 1% improvement overall. Not worth it.

---

## Reading a Flame Graph: Case Study

You have a web service that handles image resizing. Baseline latency is 500 ms per image. The target is 100 ms.

You profile the service with a typical 1 MB JPEG input:

```
perf record -g -F 99 ./image_service test.jpg
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

The flame graph shows:
- 40% in `libjpeg` (JPEG decoding)
- 30% in `libtiff` (TIFF library, for output)
- 20% in `memcpy` (from the resize scaling loop)
- 10% in `malloc` and overhead

Your first instinct: optimize libjpeg. But libjpeg is a mature C library; you will not beat it. Your second instinct: switch to a faster JPEG library (libjpeg-turbo). That might give a 20% speedup total.

But look closer at the flame graph. The `memcpy` stack shows:
```
memcpy
  <-- resize_scale_loop
      <-- bilinear_interpolation
```

`bilinear_interpolation` is doing the heavy lifting, and it is copying data on every pixel. The flame graph shows the copies are to temporary buffers, not output. The algorithm is creating a temporary scaled image in every call.

Real fix: **Change the algorithm.** Instead of creating a temporary at each level, use an in-place or streaming approach. This eliminates the allocations and copies.

Result: Total latency drops to 250 ms. You halved the time by fixing the algorithm, not by optimizing math libraries.

Then profile again. New dominant cost: `libjpeg` at 60% (because other costs are gone). Now it is worth optimizing JPEG decoding. Switch to `libjpeg-turbo` (10% speedup). Now latency is ~220 ms.

Continue profiling and fixing until you hit the 100 ms target.

---

## Tradeoffs: Profiler Comparison

| Tool | Pros | Cons | When to Use |
|------|------|------|-------------|
| `perf` (Linux) | Low overhead, works on release binaries, no recompile | Linux-only, sampling only | First-pass profiling on Linux |
| Flame graphs | Visual, easy to spot patterns | Requires post-processing, no exact timing | Understanding call patterns |
| `heaptrack` | Per-allocation tracking, low overhead | Linux-only, traces malloc not your code | Memory churn diagnosis |
| Valgrind Massif | Works everywhere, detailed timeline | Very high overhead (10–100×), slow | Memory leaks and growth over time |
| Intel VTune | Hardware-level insights (cache, pipelines) | Expensive, steep learning curve | Deep hardware analysis (when you have budget) |
| async-profiler (Java) | Understands JIT inlining, low overhead | JVM-only | Java application profiling |
| py-spy (Python) | Works on running processes, low overhead | Sampling only (accurate but statistical) | Python application profiling |
| `strace` | Exact syscall timing, works everywhere | High overhead (5–10×), noisy output | Syscall-level debugging |
| eBPF tracing | Custom, in-kernel, low overhead | Requires Linux 4.17+, steep learning curve | Lock contention, page faults, custom events |

---

## Common Misconceptions

**"The profiler shows function X is hottest; I must optimize X."** False. X might be hot because your algorithm calls it too many times, or because X is called from a loop. Optimizing X without fixing the algorithm wastes time. Understand *why* X is hot.

**"I measured with instrumentation profiling, so my data is exact."** True, but misleading. Your program is 20× slower under instrumentation, so the timing is wrong. And lock contention patterns change. Use sampling for speed; use instrumentation only for targeted investigation.

**"The profiler shows no clear bottleneck; performance is evenly distributed."** This is actually good news. It means you have already fixed the low-hanging fruit. Either set a less aggressive target, or accept that further optimization requires architectural changes, not code tweaks.

**"Profiling slows the program; I should not do it in production."** True for instrumentation. But sampling profilers (perf, py-spy) have < 5% overhead. You can and should profile in production if you have a performance issue.

**"Bigger functions are always slower than smaller functions."** False. A large function might have excellent cache locality and few branches, while a small function might call expensive operations. Function size is not a signal. Profile to find out.

---

## Exercises

1. **Baseline profiling:** Take a simple C++ program (e.g., a sorting routine, or a file processor) and profile it with `perf record`. Generate a flame graph. Identify the top 3 cost sources. Are they algorithmic, or implementation-level (allocation, cache misses)?

2. **Memory churn detection:** Write a C++ program that allocates and deallocates objects in a loop (e.g., a producer-consumer queue). Use `heaptrack` to measure allocation rate. Then modify the program to use an object pool, and re-profile. Measure the allocation rate reduction.

3. **Algorithm vs hardware optimization:** Take a naive string-search algorithm (e.g., brute-force substring search). Profile it and identify the cost (comparisons? copying?). Then apply a hardware-level optimization (e.g., use memcmp instead of char-by-char comparison). Then apply an algorithmic optimization (e.g., KMP or Boyer-Moore). Which optimization had the biggest impact?

4. **Profiling with different tools:** Profile the same program with two tools (e.g., `perf` and `heaptrack`). Do they show the same bottleneck? Why or why not?

5. **Interpreting a flame graph:** Study a publicly available flame graph (e.g., from a blog post or GitHub repo). What does the width tell you? What does the depth tell you? What would you optimize first, and why?

---

## Summary

Profiling is not guessing; it is measurement. A profiler is a tool to see where your program spends time, and to validate that your fixes work. The two main profiling approaches — sampling (cheap, low overhead, statistical) and instrumentation (accurate, high overhead, slow) — answer different questions. Sampling profilers like `perf` are your primary tool. Flame graphs let you see patterns. Memory profilers reveal allocation churn. Tracing tools show exact event timings. But profilers can lie: JIT inlining, skid, and overhead distortion all introduce errors. The profiler-first workflow — measure, identify the bottleneck, fix it, re-measure, iterate — is the only reliable path to performance. When profilers disagree, or you get surprising results, understand the tool's limitations before you reject the data.

---

> **[← Previous: Performance Engineering](07-performance-engineering.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: CPU Pipelines →](09-cpu-pipelines.md)**
