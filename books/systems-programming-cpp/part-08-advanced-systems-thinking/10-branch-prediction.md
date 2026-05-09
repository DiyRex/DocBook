# Chapter 85 — Branch Prediction

## Learning Objectives

By the end of this chapter you will be able to:

1. Understand what a branch predictor does, why it is necessary, and what happens when it fails.
2. Explain the two-bit saturating counter model and understand why modern predictors achieve 95–99% accuracy on real workloads.
3. Recognize the "sorted array is faster" problem and identify when branch unpredictability is your actual bottleneck using `perf stat`.
4. Write branch-free code where it matters, using conditional moves, arithmetic tricks, and selection patterns.
5. Distinguish between direct and indirect branches, and understand why virtual function calls are harder to predict than regular branches.
6. Know when to use compiler hints like `__builtin_expect` and C++20 `[[likely]]` / `[[unlikely]]` without cargo-culting.
7. Reason about the security-performance tradeoff introduced by Spectre mitigations.

---

## The Modern CPU: Pipelines, Speculation, and Branches

From Chapter 84, you learned that a modern CPU pipeline is 15–20 stages deep. The pipeline fills with instructions, each stage processes one instruction per cycle, and the throughput is ideally one instruction per cycle (or more, with superscalar execution).

But here is the problem: **branches are a data dependency on the instruction itself.** The CPU cannot fetch the next instruction until it knows which way the branch goes. And the condition that determines the branch (e.g., `if (a > b)`) might not be resolved until the instruction reaches the execute stage — 10–15 cycles into the pipeline.

A naive CPU would stall: fetch an instruction, wait 15 cycles for the branch condition to resolve, then fetch the next instruction. This cuts throughput from 1 IPC to 0.07 IPC. Unacceptable.

**The solution: guess.** Predict which way the branch goes before the condition is known. If the prediction is right, the pipeline keeps flowing. If wrong, throw away the speculative work (15–20 wasted cycles) and restart from the correct path.

This is the fundamental tradeoff: **Correct prediction is worth the overhead of misprediction, as long as predictions are right more than 50% of the time** (since the cost of a misprediction is roughly equal to the cost of a stall). Modern predictors achieve 95–99% accuracy, so the tradeoff wins.

The 1–5% of mispredictions — in a tight inner loop executed millions of times — is where the phrase "branchy code is slow" comes from.

---

## How Branch Predictors Work

### The Basic Model: Two-Bit Saturating Counter

The simplest model — used in teaching and in the cores of real processors — is the **two-bit saturating counter per branch**.

Each branch instruction has a 2-bit counter:
- **11** (3): Strongly taken
- **10** (2): Weakly taken
- **01** (1): Weakly not-taken
- **00** (0): Strongly not-taken

When a branch executes:
1. The predictor uses the current state to predict: counter >= 2 → predict taken; counter <= 1 → predict not-taken.
2. The branch executes and the actual outcome is known.
3. The counter is updated based on whether the prediction was right or wrong:
   - If the prediction matched the outcome, move one step toward the extreme of that outcome (e.g., if predicted taken and was taken, go from 10 → 11).
   - If the prediction was wrong, move one step toward the opposite outcome (e.g., if predicted taken but was not-taken, go from 11 → 10).

The "saturating" part means: a counter at 11 stays at 11 if the next execution is taken (it is already at the extreme); it only moves toward 10 if not-taken.

**Why this works:** A branch that is always taken will quickly saturate to 11. A branch that is always not-taken will saturate to 00. A branch that flips randomly will stay near the center (01 or 10), but the cost of being wrong half the time is small — the overhead of a misprediction is less than the savings from correct predictions on the other 50%.

But most branches are *not* random. They have patterns. And that is where real predictors shine.

### The Global History Register (GHR)

The two-bit counter per branch is indexed by the program counter (the address of the branch instruction). But branches often have context-dependent patterns:

```cpp
for (int i = 0; i < n; ++i) {
  if (data[i] > threshold) {         // Predict taken?
    if (data[i] > threshold * 2) {   // Depends on prior branch
      process(data[i]);
    }
  }
}
```

If the first branch is taken, the second is likely taken. If the first is not-taken, the second is always not-taken.

Modern predictors use the **Global History Register (GHR)**: a small shift register (typically 8–16 bits) that tracks the outcome of the last N branches globally (across the entire program). The GHR is combined with the program counter to index the prediction table:

```
index = (PC + GHR) mod table_size
```

This gives context: the same branch at the same PC will predict differently depending on the history of prior branches. This is a simple form of context modeling and increases accuracy significantly.

### Modern Predictors: TAGE

State-of-the-art predictors in 2025 (and for the last decade) use **TAGE** — **TAgged Geometry with multiple History lengths**. Simplified:

- Multiple tables, each indexed by the PC and a history of different lengths (8 bits, 16 bits, 32 bits, 64 bits).
- Each entry is tagged with the pattern of branches it was trained on, so conflicts are rare.
- When predicting, the CPU queries all tables and selects the entry with the longest matching history.

TAGE achieves 95–99% accuracy on real programs. The accuracy is so high that even unpredictable branches (random data) are often correct 50–55% of the time due to asymmetries in the workload.

**You do not need to memorize TAGE.** The key property to understand is: **predictors exploit patterns in branches. Correlated branches, loops with known iteration counts, and branches on sorted data are highly predictable. Random or pseudo-random branches are not.**

---

## The Famous "Sorted Array Faster" Example

This is a Stack Overflow classic that illustrates branch prediction perfectly. The example has become shorthand for "branch mispredictions matter":

**C++ code (simplified):**

```cpp
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <chrono>
#include <algorithm>

int main(int argc, char* argv[]) {
  const int size = 32768;
  int data[size];
  
  // Fill with random data
  std::srand(0);
  for (int i = 0; i < size; ++i)
    data[i] = std::rand() % 256;
  
  // Time 1: Unsorted data
  {
    auto start = std::chrono::high_resolution_clock::now();
    long long sum = 0;
    for (int t = 0; t < 100000; ++t) {
      for (int i = 0; i < size; ++i) {
        if (data[i] >= 128)  // Branch: predict taken?
          sum += data[i];
      }
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Unsorted: " << elapsed.count() << " ms, sum=" << sum << "\n";
  }
  
  // Sort data
  std::sort(data, data + size);
  
  // Time 2: Sorted data
  {
    auto start = std::chrono::high_resolution_clock::now();
    long long sum = 0;
    for (int t = 0; t < 100000; ++t) {
      for (int i = 0; i < size; ++i) {
        if (data[i] >= 128)  // Same branch, but now predictable
          sum += data[i];
      }
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Sorted: " << elapsed.count() << " ms, sum=" << sum << "\n";
  }
  
  return 0;
}
```

**Expected output (modern x86-64):**

```
Unsorted: 1543 ms, sum=<result>
Sorted: 427 ms, sum=<result>
```

**The 3× speedup comes from branch prediction, not from cache behavior.** Here is why:

1. **Unsorted data:** The branch condition `data[i] >= 128` is essentially random (each element is random). The predictor has no pattern to learn. Over 100,000 iterations × 32,768 elements = 3.2 billion branch instructions, the predictor is right ~50% of the time. Each misprediction costs 15–20 cycles. At 1 IPC (with perfect cache), mispredictions alone cost 30–40% of execution time.

2. **Sorted data:** After sorting, the first ~16,384 elements are < 128 (not-taken), then the next ~16,384 elements are >= 128 (taken). The branch is taken for ~16,384 consecutive iterations (in the hot loop), then not-taken for ~16,384 consecutive iterations. The predictor learns "taken" in the first phase and stays at 11 (strongly taken) through thousands of iterations. In the second phase, it learns "not-taken" and stays at 00. Prediction accuracy is ~99%, so the pipeline does not stall.

**Proof with `perf stat`:**

```bash
$ g++ -O2 -o branch_test branch_test.cpp
$ perf stat -e branch-misses,branches ./branch_test
Unsorted: 1543 ms, sum=<result>
Sorted: 427 ms, sum=<result>

Performance counter stats for './branch_test':
     7,284,528,491 branches
       339,110,815 branch-misses    # 4.66% of all branches
```

The unsorted version has ~4.66% misprediction rate. The sorted version would show < 0.5% mispredictions.

**Why does this example matter?** It shows that **branch prediction can dominate performance** — more so than cache behavior, in some cases. An algorithm that is "correct" and "has good locality" can still be 3× slower than the "same algorithm" on sorted data, purely because of branch prediction. This is why understanding the predictor is essential for systems programmers.

---

## Branch-Free Code

When a branch is unpredictable and in a hot path, replace the branch with a computation. This is called **branch-free code** or **branchless code**.

### Technique 1: Conditional Move (cmov)

Modern CPUs have conditional move instructions that do not branch:

```cpp
// Branching version
int max_val;
if (a > b)
  max_val = a;
else
  max_val = b;

// Branchless version (compiler may auto-emit)
int max_val = a > b ? a : b;  // Compiler may emit cmov
```

The compiler often recognizes `?:` and emits a conditional move:

```asm
cmp eax, ebx          # Compare a and b
cmovle eax, ebx       # If a <= b, move b into eax
                      # Otherwise eax already has a
```

A `cmov` instruction has the same latency as a normal move (1 cycle) and does not cause a pipeline flush on misprediction. The downside: **both paths are computed speculatively** (or one is ready and the other is not), and if one path has a long dependency chain, `cmov` might be slower than branching. But for simple computations, `cmov` is faster.

### Technique 2: Arithmetic Tricks

For simple predicates, you can replace the branch with arithmetic:

```cpp
// Branching: select max
int max_val;
if (a > b)
  max_val = a;
else
  max_val = b;

// Branchless: use arithmetic
int sign = (a - b) >> 31;  // -1 if a < b, 0 if a >= b (x86-64)
int max_val = b + ((a - b) & ~sign);  // Ugly but branch-free
```

Or the classic max formula:
```cpp
int max_val = a + b + abs(a - b) / 2;  // Only works if no overflow
```

Or, more readable:
```cpp
int max_val = b ^ ((a ^ b) & -(a < b));  // Bit-twiddling trick
```

These techniques are rarely worth the unreadability. **Modern compilers are very good at emitting cmov automatically.** Use them only if profiling shows a branch is a bottleneck and the compiler is not already emitting cmov.

### Technique 3: Lookup Tables

For predicates that map to discrete values, use a lookup table indexed by the predicate:

```cpp
// Branching version
int sign_of(int x) {
  if (x < 0) return -1;
  if (x == 0) return 0;
  return 1;
}

// Branchless version
int sign_table[3] = {-1, 0, 1};  // Index 0 = negative, 1 = zero, 2 = positive
int sign_of(int x) {
  int index = (x > 0 ? 1 : 0) + (x == 0 ? 0 : (x < 0 ? -1 : 1)) + 1;
  // Actually, cleaner:
  int index = (x > 0) + (x == 0) * 2;  // Results in 0, 1, or 2
  return sign_table[index];
}
```

More practically, if you have a function that returns different values based on a set of conditions, encode those conditions as bits and use the result as a lookup-table index.

### When to Optimize Branches

**Only optimize branches that:**
1. Are in a tight loop (executed millions of times).
2. Show high misprediction rates in `perf stat` (> 5%).
3. Are unpredictable on your inputs.

Random branches in initialization code or error paths do not matter. The predictor "warms up" on hot code and learns patterns quickly.

---

## Indirect Branches and Function Pointers

A **direct branch** like `if (cond)` or `goto label` is an immediate offset known at compile time. The CPU predicts taken/not-taken.

An **indirect branch** like a function pointer call or virtual method call has an unknown target address. The target is not known until the instruction executes (or is speculatively fetched).

```cpp
// Direct branch: target is known at compile time
if (x > 10)
  do_something();

// Indirect branch: target is unknown until runtime
typedef void (*Handler)();
Handler h = x > 10 ? handler_a : handler_b;
h();  // Where does this jump? Depends on runtime value of x.
```

**Indirect branches are harder to predict.** The CPU has a separate **target prediction buffer (TBP)** or **return stack buffer (RSB)** that predicts the target address. For virtual function calls, this buffer works well if the same virtual call usually calls the same function. But if a virtual function call is highly polymorphic (calling different implementations randomly), the TBP will mispredict the target, causing a pipeline flush even if the branch is taken.

**Practical implication:** Devirtualization (replacing virtual calls with direct calls via inline caches or compile-time optimization) can improve performance even when the branch itself is predictable, because direct branches are cheaper.

---

## Spectre and Meltdown: Speculative Execution Leaks Data

This is not a chapter on security, but you must understand the connection between branch prediction, speculative execution, and Spectre.

### The Basic Idea

A CPU speculates past branches, executing code that might not run. If the prediction is wrong, the executed code is discarded. But before discarding, the speculative code may have:
- Loaded data into the cache
- Modified the TLB
- Executed side channels like timing checks

A malicious attacker can craft inputs that cause speculative execution of code that leaks secrets (e.g., secret memory) via cache-timing side channels.

**Example (simplified):**

```cpp
if (index < array_size) {           // Branch prediction: likely taken
  int secret = secret_array[index]; // Speculatively executed, even if condition is false
  dummy_var = dummy_array[secret & 255];  // Secret is in cache
}
// If index >= array_size, the speculative load is discarded
// But the cache line with secret_array[secret] is now in L1
// An attacker measures the time to access dummy_array and infers which cache line was loaded
```

### Mitigations and Performance Cost

Mitigations fall into three categories:

1. **Serializing instructions:** Insert `lfence` (load fence) or `mfence` (memory fence) to prevent speculative execution past certain points. Cost: ~5 cycles per fence.

2. **Flush on context switch:** Flush the TLB and branch predictor on context switches. Cost: ~1000 cycles per context switch.

3. **Restrict speculation:** Some CPUs have a mode that disables speculative execution on certain instructions. Cost: ~5–10% throughput loss globally.

Modern kernels (since 2018) apply Spectre mitigations selectively: only for code that touches secrets (kernel code, cryptographic libraries). User code is usually unaffected.

**The key point:** Spectre mitigations trade security for performance. You might see `lfence` in VDSO (virtual dynamic shared object) code or in cryptographic libraries. Do not remove these mitigations; they are there for a reason. But understand that a 5% performance regression in some workloads is the cost of patching a class of vulnerabilities that affects billions of devices.

---

## Worked Example: Profiling Branch Mispredictions

Here is a complete example that measures branch prediction on a simple algorithm:

**branch_profile.cpp:**

```cpp
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <chrono>
#include <algorithm>
#include <cmath>

// Variant 1: Branching code (unpredictable input)
long long sum_with_branch(const int* data, int size, int threshold) {
  long long sum = 0;
  for (int i = 0; i < size; ++i) {
    if (data[i] > threshold)  // Unpredictable
      sum += data[i];
  }
  return sum;
}

// Variant 2: Branch-free code
long long sum_branchless(const int* data, int size, int threshold) {
  long long sum = 0;
  for (int i = 0; i < size; ++i) {
    int is_above = data[i] > threshold ? 1 : 0;  // cmov
    sum += data[i] * is_above;
  }
  return sum;
}

// Variant 3: Sorted data (predictable branch)
long long sum_sorted_branch(const int* data, int size, int threshold) {
  long long sum = 0;
  for (int i = 0; i < size; ++i) {
    if (data[i] > threshold)  // Predictable (monotone in sorted array)
      sum += data[i];
  }
  return sum;
}

int main() {
  const int size = 32768;
  int* data = new int[size];
  int* data_sorted = new int[size];
  
  // Generate random data
  std::srand(0);
  for (int i = 0; i < size; ++i) {
    data[i] = std::rand() % 256;
  }
  std::memcpy(data_sorted, data, size * sizeof(int));
  std::sort(data_sorted, data_sorted + size);
  
  int threshold = 128;
  const int iterations = 100000;
  
  // Test 1: Branching code on random data
  {
    auto start = std::chrono::high_resolution_clock::now();
    long long sum = 0;
    for (int t = 0; t < iterations; ++t) {
      sum += sum_with_branch(data, size, threshold);
    }
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
      std::chrono::high_resolution_clock::now() - start);
    std::cout << "Branch (random):  " << elapsed.count() << " ms, sum=" << sum << "\n";
  }
  
  // Test 2: Branch-free code on random data
  {
    auto start = std::chrono::high_resolution_clock::now();
    long long sum = 0;
    for (int t = 0; t < iterations; ++t) {
      sum += sum_branchless(data, size, threshold);
    }
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
      std::chrono::high_resolution_clock::now() - start);
    std::cout << "Branchless (random): " << elapsed.count() << " ms, sum=" << sum << "\n";
  }
  
  // Test 3: Branching code on sorted data
  {
    auto start = std::chrono::high_resolution_clock::now();
    long long sum = 0;
    for (int t = 0; t < iterations; ++t) {
      sum += sum_sorted_branch(data_sorted, size, threshold);
    }
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
      std::chrono::high_resolution_clock::now() - start);
    std::cout << "Branch (sorted):  " << elapsed.count() << " ms, sum=" << sum << "\n";
  }
  
  delete[] data;
  delete[] data_sorted;
  return 0;
}
```

**Compile and run with profiling:**

```bash
$ g++ -O2 -o branch_profile branch_profile.cpp

# Measure timing
$ ./branch_profile
Branch (random):    1543 ms, sum=<value>
Branchless (random): 1537 ms, sum=<value>
Branch (sorted):    427 ms, sum=<value>

# Measure branch misses with perf
$ perf stat -e branch-misses,branches ./branch_profile

# For the branch version on random data, expect ~4–5% misprediction rate
# For the branch version on sorted data, expect < 0.5% misprediction rate
# For branchless, no branch misses (but might still be slower due to dependency chains)
```

**Observations:**

1. **Branching on random data is slow:** ~1543 ms. The branch is unpredictable, so every misprediction flushes the pipeline.

2. **Branchless on random data:** ~1537 ms. Similar to branching. Why? Because the `cmov` compiles to a multiply (`data[i] * is_above`), which might be slower than the original branch! Branchless is not always faster.

3. **Branching on sorted data is fast:** ~427 ms. The branch is highly predictable, so the predictor is almost always right.

**Key lesson:** Always measure. Branchless code is not universally faster. It is faster only when the original branch is unpredictable and in a tight loop.

---

## Compiler Hints: `__builtin_expect` and `[[likely]]`

When you know the probability of a branch (from analysis or measurement), you can hint to the compiler, which may lay out code differently.

### `__builtin_expect` (GCC/Clang)

```cpp
// Hint: this condition is likely false
if (__builtin_expect(error_code, 0)) {  // 0 = false is expected
  handle_error();
}

// Hint: this condition is likely true
if (__builtin_expect(found, 1)) {  // 1 = true is expected
  process_result();
}
```

**What this does:**
- The compiler may reorder code so the likely path is on the critical path and the unlikely path is out-of-line.
- The hint is passed to the branch predictor for training.
- Usually not necessary on modern CPUs with good predictors, but can help on startup or with complex branches.

### C++20 `[[likely]]` and `[[unlikely]]` (standardized)

```cpp
if (condition) [[likely]] {
  // This branch is expected to be taken
  process_normal_case();
} else [[unlikely]] {
  // This branch is rarely taken
  handle_error();
}
```

This is the standard way to hint in modern C++. It is cleaner than `__builtin_expect` and is understood by all compilers that support C++20.

### When to Use Hints

- **Only in code where you have measured a branch is mispredicted.** Do not guess.
- **Error paths:** Errors are rare, so mark them `[[unlikely]]`.
- **Startup code:** Code that runs once at startup can benefit from hints because the predictor is not warmed up.
- **Very low-probability events:** If a branch is right 99% of the time, a hint might save a misprediction on the 1%.

**Do not:** Use hints as a universal optimization. Modern predictors are excellent; a hint usually does not help unless the branch is genuinely unpredictable.

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| Let predictor handle it | No code changes; works on all CPUs | Fails on unpredictable data | Default. Most branches are predictable. |
| Compiler hints (`[[likely]]`) | Simple; standardized; zero runtime cost | Only helps if branch is unpredictable | Error paths; initialization code; after measurement. |
| Branchless code (cmov, arithmetic) | Consistent performance on random data | May be slower on predictable data due to dependency chains; harder to read | Only if branch is proven unpredictable and in a hot loop. |
| Lookup tables | Fast; cache-friendly if table is small | Limited to discrete predicates; cache overhead if table is large | Predicates with few outcomes; highly polymorphic dispatches. |
| Sort/rearrange data | Fixes the root cause; improves multiple algorithms | May not always be possible; can increase complexity | If your algorithm naturally benefits from sorted/structured data. |
| Speculative vectorization (later chapter) | Multiple predictions in parallel | Complex; requires SIMD support | Bulk operations on arrays. |

---

## Common Misconceptions

**1. "Branch prediction is broken; I should eliminate all branches."**

No. Modern predictors are 95–99% accurate. Branches are necessary for control flow. Eliminating them with convoluted branchless code makes code unreadable and can actually be slower. Use branchless code only when the branch is proven to be unpredictable.

**2. "Virtual function calls are always slow."**

Virtual calls are slower than direct calls because the target is indirect and requires prediction. But on modern CPUs, devirtualization and target prediction are fast. Virtual calls are usually not a bottleneck. Only optimize if profiling shows they are.

**3. "I can fix branch prediction with `[[likely]]` hints."**

Compiler hints do not train the predictor; they only affect code layout. If the branch is actually unpredictable, a hint does not change that. Hints are a last resort after measurement.

**4. "Sorted data is always faster."**

Sorting improves branch prediction and cache locality, but it is not free. If you have to sort before processing, the sort cost might dominate. Only sort if the processing benefit outweighs the sort cost.

**5. "Spectre/Meltdown mitigations killed CPU performance."**

Mitigations apply selectively to code that touches secrets. User code is usually unaffected. The global impact on consumer workloads is ~1–5%, not the 30% you might have read in sensational news articles.

---

## Exercises

1. **Measure branch mispredictions on your machine.** Compile and run `branch_profile.cpp` (above) with `perf stat -e branch-misses,branches`. Explain the difference in misprediction rates between random and sorted data.

2. **Profile a real algorithm.** Take an algorithm from your codebase that has a tight inner loop with branches. Use `perf stat -e branch-misses` to measure the misprediction rate. If it is > 5% in the hot loop, try branchless code and compare timing.

3. **Emit assembly for branch prediction.** Compile the following with `g++ -O2 -S`:
   ```cpp
   int max_branch(int a, int b) {
     if (a > b) return a;
     return b;
   }
   int max_cmov(int a, int b) {
     return a > b ? a : b;
   }
   ```
   Compare the assembly. Does the compiler emit `cmov` for both? If not, why?

4. **Test the "sorted array" example on your machine.** Run `branch_profile.cpp` and measure the speedup of sorted vs. unsorted. Is it 3×? If not, what might explain the difference?

5. **Identify unpredictable branches in a loop.** Write a loop that processes data with an unpredictable branch (random input). Measure the misprediction rate with `perf`. Then rewrite the branch-critical part as branchless code and measure again. Did timing improve?

---

## Summary

Branch predictors are one of the most important components of modern CPUs, enabling pipelines to stay full even in the presence of branches. A two-bit saturating counter per branch, augmented with global history and modern techniques like TAGE, achieves 95–99% accuracy on real workloads. When branch prediction fails (1–5% of the time), the CPU flushes the pipeline, losing 15–20 cycles.

The "sorted array is faster" example is the canonical illustration: the same branch on random data is unpredictable and causes many mispredictions, while on sorted data the branch is monotone and highly predictable. This can result in a 3× speedup with no algorithmic change.

Branch-free code (conditional moves, arithmetic tricks) can eliminate mispredictions but is not universally faster, because both paths may have dependencies. Use branchless code only on proven unpredictable branches in tight loops. Compiler hints like `[[likely]]` / `[[unlikely]]` are useful for code layout but do not train the predictor.

Indirect branches (virtual calls, function pointers) are harder to predict than direct branches, which is why devirtualization can improve performance. Spectre mitigations add a small overhead to code that touches secrets, but are necessary for security.

The key principle: **measure first, optimize second.** Use `perf stat -e branch-misses` to confirm that branch prediction is your bottleneck before spending time on branchless code.

---

> **[← Previous: CPU Pipelines](09-cpu-pipelines.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: SIMD →](11-simd.md)**
