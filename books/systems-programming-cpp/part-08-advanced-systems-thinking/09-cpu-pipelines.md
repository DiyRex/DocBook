# Chapter 9 — CPU Pipelines

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the **pipeline** — the stages from fetch to commit — and recognize that a modern CPU executes 4+ instructions per cycle in parallel, not sequentially.
2. Understand **out-of-order execution**: how a CPU may execute instruction N+5 before instruction N if N+5 does not depend on N.
3. Identify the three classes of **hazards** that prevent pipelining — data, control, and structural — and recognize how modern CPUs mitigate them.
4. Define **instruction-level parallelism (ILP)** and predict which code exposes it (independent instructions) and which does not (long dependency chains).
5. Recognize **speculative execution**: how branch predictions allow the CPU to guess your code's path and execute ahead of confirmation.
6. Measure performance stalls with `perf stat`, connect them to pipeline effects (branch mispredicts, cache misses, register pressure), and know when each matters.
7. Explain why **loop unrolling**, **software pipelining**, and **branchless** code help the scheduler find parallelism.
8. Predict, without running code, whether a loop is likely to be pipeline-friendly or pipeline-hostile.

---

## A Modern CPU Executes Multiple Instructions Per Cycle

Here is a fact that surprises most programmers: modern CPUs do *not* execute one instruction per cycle.

A contemporary Intel or AMD processor executes 4–8 instructions *in parallel*, every cycle. Not "per N cycles" — per *one* cycle. On an Apple M-series chip, it is even more: up to 12 instructions may be in flight across different execution units. The CPU pretends to be sequential (one instruction at a time), but it is internally massively parallel.

Why does this matter? Because it means:

- Your code's **apparent** complexity — the number of instructions you write — is not what determines speed.
- The **dependency structure** of your instructions determines speed. Code with long chains of dependencies cannot be parallelized, no matter how many parallel units the CPU has.
- **Branch misses**, **cache misses**, and **structural conflicts** stall the entire pipeline, killing the parallelism.

In Part 1, we said: "performance is about which simple things you do, in what order, and with what data nearby." This chapter adds a fourth dimension: *which independent operations can be done in parallel*.

---

## The Pipeline: A Five-Stage Machine (Simplified)

Conceptually, a CPU pipeline has these stages (simplified; modern CPUs have 14+ stages):

```
Stage 1: Fetch        — Read the instruction from L1 instruction cache
Stage 2: Decode       — Figure out what the instruction does
Stage 3: Execute      — Do the operation (add, multiply, memory load)
Stage 4: Memory       — Service cache hits/misses (load/store)
Stage 5: Writeback    — Store the result in a register
```

On a five-stage pipeline, at any given moment:

- Instruction 1 is at Stage 5 (writeback), finishing.
- Instruction 2 is at Stage 4 (memory), waiting for cache.
- Instruction 3 is at Stage 3 (execute), running an ALU operation.
- Instruction 4 is at Stage 2 (decode), figuring out what to do.
- Instruction 5 is at Stage 1 (fetch), being read from cache.

They are all executing *simultaneously* on different resources. If there were no stalls (cache misses, dependencies, branch mispredictions), the CPU would retire one instruction per cycle. With a 5-stage pipeline, you could have up to 5 instructions in flight.

Modern CPUs are much wider. An Intel Xeon or Core i7 has multiple execution pipelines:

- 4 ALU (add/subtract/logic) units
- 2 load/store units
- 2 multiply/divide units (or more)
- Vector units (SSE, AVX)

So at any cycle, the CPU can issue up to 4 integer operations + 2 memory operations + vector operations *in the same cycle*, provided there are no dependencies between them. That is **instruction-level parallelism (ILP)**.

### A diagram

```
Cycle:     0    1    2    3    4    5    6
          ----+----+----+----+----+----+
Instr A:  |F--D--E--M--W|
Instr B:     |F--D--E--M--W|
Instr C:        |F--D--E--M--W|
Instr D:           |F--D--E--M--W|
Instr E:              |F--D--E--M--W|
          ----+----+----+----+----+----+

(F=Fetch, D=Decode, E=Execute, M=Memory, W=Writeback)
```

At cycle 4, the CPU is making *progress* on all five instructions. If they are independent, it retires one per cycle. If Instruction D depends on the result of Instruction A, then D must *stall* (wait) until A completes. That stall kills throughput.

---

## Out-Of-Order Execution

Simplistic in-order execution says: "execute instruction N, then N+1, then N+2, in order."

Modern CPUs do not do that. They do **out-of-order execution**: if instruction N depends on earlier instructions but instruction N+5 does not, the CPU will execute N+5 *first*.

### Why this matters

Consider:

```cpp
int a = x + y;           // Instruction 1
int b = a * 2;           // Instruction 2 (depends on 1)
int c = p + q;           // Instruction 3 (independent)
int d = c * 3;           // Instruction 4 (depends on 3)
int result = b + d;      // Instruction 5 (depends on 2 and 4)
```

In-order execution:
```
Cycle 0: Fetch instr 1 (add x, y)
Cycle 1: Execute instr 1
Cycle 2: Fetch instr 2 (mul a, 2)
Cycle 3: Execute instr 2 (but a not ready; stall)
Cycle 4: Execute instr 2
Cycle 5: Fetch instr 3 ... (wasted!)
```

Out-of-order execution:
```
Cycle 0: Fetch instr 1, 2, 3, 4, 5
Cycle 1: Execute instr 1 and instr 3 (both have data)
Cycle 2: Execute instr 2 (a is ready) and instr 4 (c is ready)
Cycle 3: Execute instr 5 (both b and d are ready)
```

Out-of-order is much faster because the CPU does not stall on instr 2; it moves on to independent work.

Modern CPUs track the **data dependencies** between instructions (not the order you wrote them in) and schedule instructions based on when their inputs are ready, not on the order.

---

## The Reorder Buffer and Register Renaming

How does the CPU implement out-of-order execution while maintaining the *illusion* of sequential execution?

### The reorder buffer (ROB)

The reorder buffer is a large queue (typically 200+ entries on x86) that holds instructions *after* they have been executed but *before* they have been retired (committed to the architectural state).

```
Instructions enter the ROB in order.
They execute out-of-order (when inputs are ready).
Results are held in the ROB until all earlier instructions have retired.
Then they retire in order.
```

This way, if your code depends on the side effects of previous instructions (memory writes, exceptions), those effects happen in the correct order.

### Register renaming

Here is a problem: suppose you do this:

```cpp
rax = a;      // rax has value a
rax = b;      // rax now has value b
rc = rax + 1; // rax should have value b, not a
```

If the CPU had only 16 physical registers (x86), both writes to `rax` would block each other: the first write must commit before the second can start. But the second read of `rax` happens after the first write, so there is a false dependency.

**Register renaming** solves this: the CPU internally keeps 100+ physical registers, not 16. When you write `rax`, it actually writes to one of `preg_47`, `preg_92`, etc. When you read `rax`, the CPU looks up which physical register currently holds the value of `rax`.

```
Architectural register:  rax         rax         rax
Physical registers:      preg_47  -> preg_92 -> preg_155

Writes:                  rax=a       rax=b       rax=c
Reads:                                 rax        rax
```

This breaks the false dependency. `rax=b` no longer has to wait for `rax=a` to retire; they just use different physical registers. The CPU can execute them in parallel, or out of order, and still maintain the illusion that you are writing and reading the same register.

---

## Hazards: Three Ways the Pipeline Stalls

### Data hazards

A **data hazard** occurs when an instruction needs the result of an earlier instruction that has not yet computed it.

Example:

```cpp
int a = x + y;
int b = a * 2;  // Cannot execute until 'a' is ready
```

The second instruction depends on the first. The CPU must either:
- **Stall** the pipeline: wait for `a` to be ready, then execute `b`. Cost: 1+ cycles of lost throughput.
- **Forward** the result: as soon as the first instruction's result is computed (before writeback), forward it directly to the second instruction. Cost: ~1 cycle, much better.

Modern CPUs always forward when possible. But some data hazards cannot be forwarded:

- **Load-to-use delays**: A load from memory takes many cycles to complete. If the next instruction uses that value, it must wait.
  ```cpp
  int x = arr[i];      // load from memory (~50-300 cycles)
  int y = x + 1;       // must wait for x
  ```

- **Multiply-to-use delays**: Multiplication is slow (~3-4 cycles on modern CPUs). If the next instruction uses the result immediately, it must wait.
  ```cpp
  long a = (long)x * y;    // ~3 cycles
  long b = a + z;          // wait ~3 cycles
  ```

### Control hazards

A **control hazard** occurs when the CPU encounters a branch and does not know which way it will go.

```cpp
if (x > 0) {
    // path A
} else {
    // path B
}
```

The fetch stage has already read 10 instructions ahead (prefetching), but it does not know which 10 to fetch until the branch condition is resolved. Options:

- **Stall** the fetch until the branch is resolved. Cost: 10+ cycles of lost throughput.
- **Predict** which way the branch will go, fetch that path, and execute speculatively. If correct, no penalty. If wrong, **flush** the incorrectly fetched instructions and refetch the correct path. Cost: 0 cycles if correct, 10-20 cycles if mispredicted.

Modern CPUs always predict. This is covered in detail in Chapter 10 (Branch Prediction).

### Structural hazards

A **structural hazard** occurs when two instructions want to use the same hardware resource at the same time.

Example: suppose your CPU has only one multiplier. If you issue two multiply instructions in the same cycle, one must stall waiting for the other to finish.

Modern CPUs have multiple execution units, so structural hazards are rare. But they can still happen:

- Multiple stores to the same address in the same cycle (memory bus contention).
- Multiple branches in the same cycle (branch predictor saturation, rare).
- Vector instructions that require more resources than available.

---

## Instruction-Level Parallelism (ILP)

The CPU's ability to execute instructions in parallel is called **instruction-level parallelism**. It is the ratio of instructions issued per cycle to the maximum possible (in our example, 4–8 per cycle for modern x86).

The CPU finds ILP by analyzing the **dependency graph** of your code.

### Independent instructions: ILP = high

```cpp
int a = x + y;
int b = p + q;
int c = m + n;
int d = s + t;
```

No instruction depends on another. The CPU can execute all four in parallel (assuming 4 ALU units). ILP ≈ 4.0.

### Dependent chain: ILP = low

```cpp
int a = x + y;
int b = a + 1;
int c = b + 1;
int d = c + 1;
```

Each instruction depends on the previous. The CPU can execute only one per cycle (plus some pipelining overlap, so maybe 1.2–1.5 per cycle). ILP ≈ 1.0.

### Why ILP matters

A modern CPU can issue 4–8 instructions per cycle. But if your code has long dependency chains, the CPU achieves only ILP ≈ 1.0 and can issue only 1 instruction per cycle. You are wasting 75–87% of the CPU's execution capacity.

This is the fundamental reason loop unrolling helps: it exposes independent iterations to the scheduler.

---

## Speculative Execution

When the CPU encounters a branch, it must know the condition before deciding which path to take. But the condition may not be computed for several cycles (it depends on a memory load, or a complex computation). The CPU cannot wait; it must guess.

**Speculative execution** means the CPU guesses which way the branch will go (using a branch predictor, Chapter 10), fetches and executes that path, and *commits* the results only if the guess was correct.

### If the prediction is correct

```cpp
if (x > 0) {           // Predictor: "likely taken"
    // Fetch and execute speculatively
    result = x * 2;    // Real work happens
}
// If correct, the result is committed. No penalty.
```

### If the prediction is wrong

```cpp
if (x > 0) {           // Predictor: "likely taken"
    result = x * 2;    // Executed speculatively
} else {
    result = -x;       // Never executed
}
// Prediction was wrong. Flush all speculative work.
// Refetch the correct path.
// Cost: ~15-20 cycles on x86 (the misprediction penalty).
```

The penalty for a misprediction is typically 15–20 cycles on modern x86. That is catastrophic for throughput. A single misprediction can waste more cycles than 50 correct predictions gain.

This is why:
- Branch-predictable code is fast (Chapter 10).
- Branch-heavy code with unpredictable patterns is very slow.
- Branchless code (using `cmov` or bitwise tricks) is fast even when the predictor fails.

---

## Stalls You Should Care About

### Branch misses

A branch misprediction flushes the pipeline and costs 15–20 cycles on x86. If you have unpredictable branches in a hot loop, misses will dominate your performance.

**Measurement:**
```bash
perf stat -e branch-misses,branches ./your_program
```

If `branch-misses / branches` > 1%, you have a problem.

### Cache misses

A cache miss (L1 or L2) stalls the pipeline because the missing value is needed soon. If the miss reaches main memory, the stall is ~200 cycles.

**Measurement:**
```bash
perf stat -e cache-misses,cache-references ./your_program
```

If `cache-misses / cache-references` > 1%, investigate.

### Store-to-load forwarding failures

The CPU has a clever optimization: when you do a store (write) followed by a load from the same address, the CPU forwards the stored value directly to the load, without waiting for memory.

```cpp
arr[i] = x;       // store
int y = arr[i];   // load from same address
// CPU forwards x to y, no memory wait
```

But if the store and load address do not match precisely, the forwarding fails, and the load must wait for memory.

```cpp
arr[i] = x;
int y = arr[i+1];  // different address, no forward
// Load stalls, waiting for memory
```

**Measurement:**
```bash
perf stat -e mem_inst_retired.stlb_miss,mem_inst_retired.stlb_hit ./your_program
```

(Varies by CPU; see `perf list` for exact event names.)

### Partial register dependencies

Writing to an 8-bit or 16-bit register (e.g., `al`, `ax`) in x86 when the full 64-bit register (`rax`) was used recently can cause a stall, because the CPU must merge the partial write with the previous value.

```cpp
rax = big_value;
al = x;           // Stall: must merge with rax's high bits
```

Avoid if possible.

### Register pressure

If you have more live values (values in use) than physical registers, the CPU must spill to memory and reload. Each spill/reload adds cycles.

**Measurement:**
```bash
perf stat -e page-faults ./your_program
```

(Not direct, but spills often correlate with cache misses.)

---

## How Compilers Help: The Scheduler

The C++ compiler's **optimizer** includes a **scheduler**: a module that reorders instructions to maximize ILP and hide latencies.

### Loop unrolling

The compiler can unroll loops to expose more independent iterations:

**Original:**
```cpp
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}
```

**After unrolling by 4:**
```cpp
for (int i = 0; i < N; i += 4) {
    c[i]     = a[i]     + b[i];     // iteration i
    c[i+1]   = a[i+1]   + b[i+1];   // iteration i+1
    c[i+2]   = a[i+2]   + b[i+2];   // iteration i+2
    c[i+3]   = a[i+3]   + b[i+3];   // iteration i+3
}
```

All four iterations can execute in parallel if there are enough ALUs. ILP jumps from ~1 to ~4.

With `-O3`, compilers often unroll automatically. You can also use pragmas:

```cpp
#pragma GCC unroll 4
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}
```

### Software pipelining

For loops with long dependency chains, the compiler can **pipeline** the loop: overlap iterations so that while iteration N is executing, iteration N+1 is being fetched and iteration N-1 is finishing.

This is complex to describe without assembly, but the idea is: if a single iteration takes 10 cycles (due to a long dependency chain and a cache miss), the compiler can arrange the code so that a new iteration *starts* every 2 cycles, even though each completes in 10. The throughput is (10 / N) per iteration, but the latency is hidden.

### Instruction scheduling

The compiler reorders independent instructions to minimize stalls:

**Original (bad):**
```cpp
int a = x + y;
int b = a * 2;  // must wait for a
int c = p + q;
int d = c * 3;
```

**After scheduling (good):**
```cpp
int a = x + y;
int c = p + q;  // start while waiting for a
int b = a * 2;
int d = c * 3;
```

The compiler moves the independent `int c = p + q;` between the two dependent operations, so the CPU can do useful work while waiting for `a`.

### Vectorization

The compiler can use SIMD instructions (SSE, AVX) to process multiple elements in parallel.

**Original:**
```cpp
for (int i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}
```

**After vectorization with AVX:**
```cpp
for (int i = 0; i < N; i += 8) {
    // Load 8 floats from a and b
    __m256 va = _mm256_loadu_ps(&a[i]);
    __m256 vb = _mm256_loadu_ps(&b[i]);
    // Add 8 floats in parallel
    __m256 vc = _mm256_add_ps(va, vb);
    // Store 8 floats to c
    _mm256_storeu_ps(&c[i], vc);
}
```

One SIMD instruction does the work of 8 scalar instructions. ILP and throughput both improve dramatically.

---

## Where You Must Help

Compilers are good at finding ILP in straightforward code. But they cannot always see what you see:

### Hot loops with long dependency chains

```cpp
long sum = 0;
for (int i = 0; i < N; ++i) {
    sum += arr[i];  // Each iteration depends on previous
}
```

The compiler cannot unroll this naively because each iteration depends on `sum` from the previous. But you can:

```cpp
long sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
for (int i = 0; i < N; i += 4) {
    sum0 += arr[i];
    sum1 += arr[i+1];
    sum2 += arr[i+2];
    sum3 += arr[i+3];
}
long sum = sum0 + sum1 + sum2 + sum3;
```

Now the four sums are independent, and the CPU can execute all four additions in parallel. Throughput increases 3–4×.

### Branchy code with unpredictable branches

```cpp
for (int i = 0; i < N; ++i) {
    if (cond[i]) {    // Branch predictor fails
        result += heavy_work(arr[i]);
    }
}
```

The branch predictor cannot predict this (it depends on random data). Each miss costs 15+ cycles. You could rewrite branchless:

```cpp
for (int i = 0; i < N; ++i) {
    int work = cond[i] ? heavy_work(arr[i]) : 0;
    result += work;
}
```

Now there is no branch; the CPU uses `cmov` (conditional move) to select the result. Slower per iteration, but no misses.

### Pointer chasing

```cpp
for (Node* n = head; n; n = n->next) {
    result += process(n);
}
```

Each iteration depends on the `next` pointer from the previous node. If nodes are scattered in memory, each iteration includes a cache miss (~200 cycles). The loop is bottlenecked on memory latency, not compute.

Solution: prefetch ahead.

```cpp
Node* n = head;
Node* prefetch_ptr = head;
while (n) {
    // Prefetch a few nodes ahead
    if (prefetch_ptr) {
        __builtin_prefetch(prefetch_ptr->next);
        prefetch_ptr = prefetch_ptr->next;
    }
    result += process(n);
    n = n->next;
}
```

This tells the CPU to fetch ahead, hiding the latency.

---

## Worked Example: Linear Search vs Branchless Minimum

Consider finding the minimum value in an array:

**Branching version:**

```cpp
int find_min_branch(const int* arr, int n) {
    int min = arr[0];
    for (int i = 1; i < n; ++i) {
        if (arr[i] < min) {
            min = arr[i];
        }
    }
    return min;
}
```

**Branchless version:**

```cpp
int find_min_branchless(const int* arr, int n) {
    int min = arr[0];
    for (int i = 1; i < n; ++i) {
        int mask = (arr[i] < min) ? -1 : 0;  // all 1s or all 0s
        min = (min & ~mask) | (arr[i] & mask);
    }
    return min;
}
```

Or, using `cmov`:

```cpp
int find_min_cmov(const int* arr, int n) {
    int min = arr[0];
    for (int i = 1; i < n; ++i) {
        if (arr[i] < min) min = arr[i];
        // Compiled to: cmp + cmov (no branch)
    }
    return min;
}
```

### Benchmark results (on real data)

With *random* data (branch predictor fails):

```
Branching:    450 ms   (many mispredictions)
Branchless:   120 ms   (no branch stalls)
```

With *sorted* data (branch predictor succeeds):

```
Branching:    80 ms    (no mispredictions)
Branchless:   120 ms   (unnecessary bitwise work)
```

The branching version is faster on predictable data but catastrophically slow on random data. The branchless version is stable.

Which do you use?

- If the data is predictable (already sorted, or clustered), use the branching version.
- If the data is unpredictable, use branchless.
- If you do not know, use branchless and profile.

Measure with `perf stat`:

```bash
perf stat -e branch-misses,branches ./min_branch < random_data.txt
perf stat -e branch-misses,branches ./min_branch < sorted_data.txt
```

You will see that the random version has many more misses.

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Dependent chain code** | Simple, clear intent | Serializes operations, low ILP (~1) | When you have no choice (reductions). Then optimize with techniques below. |
| **Loop unrolling** | Exposes parallel iterations, increases ILP | Larger code, more register pressure | Hot loops with independent iterations; let compiler do it at -O3. |
| **Software pipelining** | Hides long latencies | Complex, compiler-dependent | Compute-heavy loops with long chains; let compiler do it. |
| **Branchless code** | No mispredictions on random data, vectorizable | Unnecessary work on predictable data, harder to read | When branches are unpredictable; profile first. |
| **Prefetching** | Hides load latencies in pointer chasing | Complicates code, tuning required | Pointer-heavy loops; use `__builtin_prefetch`. |
| **Reducing register pressure** | Fewer spills, better throughput | May force temporary memory use | Hot loops near register limit; profile. |

---

## Common Misconceptions

1. **"More instructions = slower."** No. Modern CPUs execute 4+ instructions per cycle. What matters is the dependency depth, not the count.

2. **"Branching is always bad."** Branches are bad *only if they mispredict*. Predictable branches are free.

3. **"Out-of-order execution is magic."** It is not. The CPU can reorder instructions only if they are independent. Long dependency chains still serialize.

4. **"The reorder buffer is unlimited."** No. At ~200 entries, it can be exhausted. If you have more than 200 instructions in flight (rare), the fetch engine stalls.

5. **"Register renaming eliminates all false dependencies."** No, it eliminates false *write-after-write* and *read-after-write* hazards. True data dependencies (reads that need earlier writes) cannot be eliminated.

6. **"My loop is slow because I have too many instructions."** Maybe, but more likely it is slow because the instructions are dependent, or there are branch misses, or you miss the cache. Measure with `perf stat` first.

---

## Exercises

1. **Dependency graph.** Take a small loop (5–10 lines). Draw the data dependencies (which values depend on which earlier values). Identify the **critical path** — the longest chain of dependencies. That chain determines the minimum latency of the loop, no matter how many ALUs the CPU has.

2. **Measure ILP.** Compile a loop with `-O2` and with `-O3`. Run both with `perf stat -e instructions,cycles`. The ratio `instructions / cycles` approximates ILP. Which optimization level wins? Why?

3. **Unroll and measure.** Write a simple accumulation loop:
   ```cpp
   long sum = 0;
   for (int i = 0; i < N; ++i) sum += arr[i];
   ```
   Manually unroll by 4 (four independent sums). Time both. The unrolled version should be 2–4× faster if the CPU has multiple ALUs.

4. **Branch predictor test.** Write two loops over the same data: one with a predictable branch (e.g., `if (i % 2 == 0)`) and one with an unpredictable branch (e.g., `if (arr[i] % 2 == 0)` with random `arr`). Measure `perf stat -e branch-misses`. Record the ratio of misses. How much slower is the unpredictable branch?

5. **Branchless minimum.** Implement both the branching and branchless versions of `find_min` above. Benchmark both on sorted and random data. At what point does the branchless version win?

6. **Register pressure.** Write a loop that computes several independent accumulations:
   ```cpp
   long sum0 = 0, sum1 = 0, ..., sum7 = 0;
   for (int i = 0; i < N; i += 8) {
       sum0 += arr[i];
       ...
       sum7 += arr[i+7];
   }
   ```
   Compile with `-O2` and examine the assembly (`-S -masm=intel`). How many registers does the loop use? Are there spills?

7. **Prefetch.** Write a pointer-chasing loop (linked list traversal) and measure it with `perf`. Then add `__builtin_prefetch` a few nodes ahead. Does it improve throughput?

---

## Summary

Modern CPUs are not sequential machines. They execute multiple instructions per cycle in parallel, provided those instructions are independent. Dependency chains, branch misses, and cache misses stall the pipeline and kill parallelism. The scheduler (in the CPU and the compiler) works to expose independent instructions and hide latencies. When your code is slow, it is usually because the dependency chains are long, branches mispredict, or memory stalls. Loop unrolling, branchless code, and prefetching are the tools to break those bottlenecks. Measure with `perf stat`; profile where you think the stalls are; then optimize.

The next chapter, Branch Prediction, zooms in on one of the most important stalls: the cost of guessing wrong on a conditional branch.

---

> **[← Previous: Profiling](08-profiling.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: Branch Prediction →](10-branch-prediction.md)**
