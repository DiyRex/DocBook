# Chapter 11 — SIMD: Single Instruction, Multiple Data

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what SIMD is, why it exists, and how it differs from scalar execution.
2. Identify SIMD architectures on your platform (SSE, AVX, AVX-512, NEON, SVE) and their vector widths.
3. Recognize when the compiler auto-vectorizes a loop and when it refuses, with concrete reasons.
4. Write loops that compiler can vectorize without manual intrinsics.
5. Use compiler flags (`-fopt-info-vec`, `-march=native`) to see what was vectorized.
6. Write and benchmark manual SIMD code using intrinsics (`_mm256_add_ps`, etc.).
7. Understand the constraints: alignment, data layout, data dependencies, and control flow.
8. Know when SIMD is the right tool and when other optimizations matter more.

This chapter builds on the data-oriented design foundation from Chapter 8. SIMD is not a replacement for good memory layout; it is an accelerator for code that already has good data layout. A misaligned, strided access pattern will be slow even with SIMD intrinsics. But a tight, contiguous loop can become 4–32× faster with one CPU instruction doing the work of 4, 8, 16, or 32 scalar operations in parallel.

---

## 11.1 What SIMD Actually Is

**Single Instruction, Multiple Data (SIMD)** means: one CPU instruction operates on multiple independent values at once.

Consider a scalar addition:

```cpp
float a = 1.0f, b = 2.0f;
float c = a + b;  // One ADD instruction, one result
```

The CPU executes an `add` instruction, reads two registers, writes one result. Compare this to a vector addition:

```cpp
float a[4] = {1.0f, 2.0f, 3.0f, 4.0f};
float b[4] = {5.0f, 6.0f, 7.0f, 8.0f};
float c[4];

// Hypothetical SIMD addition: one instruction, four results
// c[0] = a[0] + b[0]
// c[1] = a[1] + b[1]
// c[2] = a[2] + b[2]
// c[3] = a[3] + b[3]
```

One `vadd` instruction (vector add) reads two vector registers, each holding 4 floats, and writes 4 results to another register.

**On real hardware**, the registers are:

- **x86-64 (Intel/AMD):**
  - SSE: 128-bit registers (`xmm0`–`xmm15`). Holds 4 floats or 2 doubles.
  - AVX: 256-bit registers (`ymm0`–`ymm15`). Holds 8 floats or 4 doubles.
  - AVX-512: 512-bit registers (`zmm0`–`zmm31`). Holds 16 floats or 8 doubles.
- **ARM (NEON on ARMv8):**
  - 128-bit registers (`v0`–`v31`). Holds 4 floats or 8 int16s.
- **ARM (SVE on Armv8.2+):**
  - Variable-width registers (128–2048 bits, depending on CPU). Holds 2–32 floats.

A 256-bit AVX register packing 8 floats means one `vaddps` (vector add packed single-precision) instruction does the work of 8 scalar `add` instructions.

### Why the CPU Has SIMD

The CPU has multiple execution units. While one integer ALU is computing, another can work in parallel. SIMD exploits this parallelism without requiring multiple threads or synchronization.

The cost is low: one SIMD instruction decodes once, fetches operands once, executes once. The payoff is 4–32× work per instruction.

---

## 11.2 When the Compiler Auto-Vectorizes

Modern C++ compilers (GCC 5+, Clang 5+, MSVC 2015+) have loop vectorizers that automatically transform scalar loops into SIMD code. You do not write intrinsics; the compiler does it for you.

### A Vectorizable Loop

```cpp
#include <vector>

void add_arrays(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

Compile with `-O3`:

```bash
clang++ -O3 -S -masm=intel simd_test.cpp
```

The compiler sees:

1. **Loop trip count is large and predictable** (based on `n`).
2. **No data dependencies between iterations.** The `i`-th iteration does not depend on iteration `i-1`.
3. **Contiguous memory access.** Each array is accessed sequentially with unit stride.
4. **Simple operation** (addition). The compiler can map it to one instruction.

Result: the compiler rewrites the loop to process 4, 8, or 16 elements per iteration (depending on register width and data type).

### What the Compiler Actually Generated

If you inspect the assembly with `-S -masm=intel`, you might see:

```asm
.L_loop:
    movups  ymm0, [rdi + rax]      ; Load 8 floats from a[i..i+7]
    movups  ymm1, [rsi + rax]      ; Load 8 floats from b[i..i+7]
    vaddps  ymm0, ymm0, ymm1       ; Add them (8 floats in parallel)
    movups  [rdx + rax], ymm0      ; Store 8 floats to c[i..i+7]
    add     rax, 32                ; Next 8 floats (8 * 4 bytes)
    cmp     rax, rcx               ; Loop condition
    jl      .L_loop
```

This loop processes 8 floats per iteration instead of 1. The trip count is halved. The effective throughput is 8×.

### Verifying What Was Vectorized

Use `-fopt-info-vec` (GCC/Clang) or `-Rpass=loop-vectorize` (Clang) to see what the compiler vectorized:

```bash
clang++ -O3 -fopt-info-vec simd_test.cpp 2>&1 | grep -i "vectorized"
```

Output:

```
simd_test.cpp:5:5: remark: vectorized loop (vectorization factor: 8) [-Rpass=loop-vectorize]
```

This means the loop was vectorized with a factor of 8 (processing 8 elements per iteration).

---

## 11.3 Why the Compiler Refuses to Vectorize

Even with `-O3`, many loops remain scalar. Here are the real reasons compilers bail out.

### 1. Pointer Aliasing

```cpp
void process(float* a, float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

The compiler cannot prove that `a`, `b`, and `c` do not overlap. If they do, vectorizing this loop could produce wrong results:

```cpp
process(a, a, a, n);  // a[i] = a[i] + a[i]
```

If the compiler uses 8-wide SIMD, it loads 8 values from `a`, adds them (doubling each), and writes back. But the stores might overwrite values that the loop still needs to read. The compiler is conservative: it does not vectorize unless aliasing is impossible.

**Fix:** Use `restrict` or `__restrict__` to tell the compiler the pointers do not alias:

```cpp
void process(float* __restrict a, float* __restrict b, float* __restrict c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

Now the compiler can vectorize safely.

### 2. Data Dependencies Between Iterations

```cpp
float accumulate(const float* a, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) {
        sum += a[i];  // sum depends on sum from i-1
    }
    return sum;
}
```

The loop body reads `sum` (from the previous iteration) and writes `sum` (for the next iteration). This is a **data dependency chain**:

```
Iteration 0: sum = 0 + a[0]
Iteration 1: sum = (0 + a[0]) + a[1]  <- depends on iteration 0
Iteration 2: sum = ((0 + a[0]) + a[1]) + a[2]  <- depends on iteration 1
...
```

You cannot execute iterations in parallel if each one waits for the previous. The compiler refuses to vectorize.

**Fix:** Break the dependency chain by unrolling with multiple accumulators:

```cpp
float accumulate(const float* a, size_t n) {
    float sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
    size_t i = 0;
    for (; i + 4 <= n; i += 4) {
        sum0 += a[i];
        sum1 += a[i+1];
        sum2 += a[i+2];
        sum3 += a[i+3];
    }
    // Handle remainder
    float sum = sum0 + sum1 + sum2 + sum3;
    for (; i < n; ++i) {
        sum += a[i];
    }
    return sum;
}
```

Now the four accumulators are independent; the CPU can execute all four `add` instructions in parallel.

### 3. Complex Control Flow

```cpp
void filter(const float* a, float* b, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        if (a[i] > 0.5f) {
            b[i] = a[i] * 2.0f;
        } else {
            b[i] = 0.0f;
        }
    }
}
```

The branch creates unpredictability. With SIMD, you cannot selectively execute some lanes and not others (well, you can with masking, but the compiler may decline if the condition is unpredictable). Modern compilers *can* vectorize this with masked operations, but only if the branch pattern is predictable.

**Fix:** Make the operation branchless:

```cpp
void filter(const float* a, float* b, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        b[i] = (a[i] > 0.5f) ? (a[i] * 2.0f) : 0.0f;
    }
}
```

Or use a conditional move or max:

```cpp
for (size_t i = 0; i < n; ++i) {
    b[i] = std::max(0.0f, a[i] * 2.0f);  // Clamp to zero
}
```

### 4. Non-Contiguous Access (Stride > 1)

```cpp
void sum_every_nth(const float* a, float* result, size_t n) {
    *result = 0.0f;
    for (size_t i = 0; i < n; i += 2) {  // stride of 2
        *result += a[i];
    }
}
```

The stride of 2 means you skip every other element. The compiler can sometimes vectorize strided access, but it is less efficient. Better to restructure the data (SoA, as discussed in Chapter 8) so access is contiguous.

### 5. Pointer Arithmetic and Unclear Loop Bounds

```cpp
void process(float* p, float* end) {
    while (p < end) {
        *p += 1.0f;
        p++;
    }
}
```

The compiler cannot determine the trip count (number of iterations) without analyzing the pointers, which is hard. Use an index-based loop:

```cpp
void process(float* p, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        p[i] += 1.0f;
    }
}
```

Now the trip count is explicit.

---

## 11.4 Manual SIMD: Intrinsics and Alternatives

When auto-vectorization fails or is insufficient, you can write SIMD code by hand.

### 11.4.1 Intrinsics: Direct Instruction Mapping

Intrinsics are inline functions that map 1:1 to CPU instructions. The compiler does not try to be clever; it just emits the instruction you ask for.

#### SSE Example

```cpp
#include <xmmintrin.h>  // SSE

void add_sse(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; i += 4) {
        __m128 av = _mm_loadu_ps(&a[i]);    // Load 4 floats (unaligned)
        __m128 bv = _mm_loadu_ps(&b[i]);    // Load 4 floats (unaligned)
        __m128 cv = _mm_add_ps(av, bv);     // Add 4 floats in parallel
        _mm_storeu_ps(&c[i], cv);           // Store 4 floats (unaligned)
    }
}
```

Intrinsics are portable across compilers (GCC, Clang, MSVC all support the same SSE intrinsics), but **not** portable across architectures. This code runs on x86-64; it will not compile on ARM.

#### AVX Example

AVX operates on 256-bit registers (8 floats):

```cpp
#include <immintrin.h>  // AVX

void add_avx(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; i += 8) {
        __m256 av = _mm256_loadu_ps(&a[i]);
        __m256 bv = _mm256_loadu_ps(&b[i]);
        __m256 cv = _mm256_add_ps(av, bv);
        _mm256_storeu_ps(&c[i], cv);
    }
}
```

The instruction names are similar but with a `256` prefix.

#### FMA (Fused Multiply-Add)

A common pattern is `d = a * b + c`. On hardware with FMA, this is one instruction:

```cpp
#include <immintrin.h>

void fmadd_avx(const float* a, const float* b, const float* c, float* d, size_t n) {
    for (size_t i = 0; i < n; i += 8) {
        __m256 av = _mm256_loadu_ps(&a[i]);
        __m256 bv = _mm256_loadu_ps(&b[i]);
        __m256 cv = _mm256_loadu_ps(&c[i]);
        
        // FMA: d = a * b + c (executed as one instruction)
        __m256 dv = _mm256_fmadd_ps(av, bv, cv);
        
        _mm256_storeu_ps(&d[i], dv);
    }
}
```

The `fmadd` instruction is typically lower latency and higher throughput than separate `mul` and `add`.

### 11.4.2 Aligned vs Unaligned Loads

Loading from unaligned addresses (`_mm_loadu_ps`) is safe but slower than aligned loads (`_mm_load_ps`):

```cpp
// Unaligned load (slower, safe)
__m256 av = _mm256_loadu_ps(ptr);

// Aligned load (faster, requires ptr to be 32-byte aligned)
__m256 av = _mm256_load_ps(ptr);
```

Enforce alignment with:

```cpp
alignas(32) float a[1000];  // Guarantee 32-byte alignment
```

Or query alignment with `std::alignment_of`:

```cpp
static_assert(alignof(float[]) >= 32);
```

### 11.4.3 OpenMP `#pragma simd`

You can hint to the compiler to vectorize a loop without writing intrinsics:

```cpp
#pragma omp simd collapse(1)
for (size_t i = 0; i < n; ++i) {
    c[i] = a[i] + b[i];
}
```

This is a compiler hint, not a guarantee. If the compiler cannot vectorize (due to aliasing, dependencies, etc.), the loop runs scalar. But you can strengthen the hint with clauses:

```cpp
#pragma omp simd collapse(1) \
    aligned(a, b, c : 32) \
    reduction(+:sum)
for (size_t i = 0; i < n; ++i) {
    sum += a[i];
}
```

The `aligned` clause tells the compiler pointers are 32-byte aligned. The `reduction` clause tells the compiler how to fold the partial sums.

### 11.4.4 `std::simd` (C++26 / std)

C++26 introduces `std::simd`, a portable wrapper over intrinsics:

```cpp
#include <experimental/simd>  // Currently experimental

using simd = std::simd<float>;

void add_stdsimd(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; i += simd::size()) {
        auto av = simd(a + i, std::vector_aligned);
        auto bv = simd(b + i, std::vector_aligned);
        auto cv = av + bv;
        cv.copy_to(c + i, std::vector_aligned);
    }
}
```

`std::simd` abstracts the vector width, alignment, and register type. The same code works on SSE, AVX, NEON, and SVE.

**Status:** `std::simd` is experimental. It is available in GCC 10+ and Clang 14+, but not yet in the C++ standard library. Use for forward-compatible code.

### 11.4.5 Third-Party Libraries

- **Google Highway** (https://github.com/google/highway): Portable SIMD abstraction, supports SSE, AVX, NEON, SVE.
- **xsimd** (https://github.com/xtensor-stack/xsimd): Portable SIMD with NumPy-style syntax.
- **ISPC** (Intel SPMD Program Compiler): A separate compiler that generates SIMD code from scalar-like code.

For most applications, auto-vectorization or `std::simd` is sufficient.

---

## 11.5 Data Layout Constraints for SIMD

SIMD requires the same data layout discipline as auto-vectorization. More so, in fact.

### 11.5.1 SoA Wins Over AoS

From Chapter 8, you know that SoA is preferable for SIMD. Here is why concretely:

**AoS (interleaved data):**

```cpp
struct Particle {
    float x, y, z;
};
Particle particles[1'000'000];

// Stride is 12 bytes (size of struct)
for (int i = 0; i < 1'000'000; ++i) {
    particles[i].x += 1.0f;
}
```

To load the `x` values, the CPU must stride through memory, skipping `y` and `z`. A SIMD load from a strided pattern requires scatter/gather instructions, which are slow.

**SoA (contiguous arrays):**

```cpp
struct Particles {
    float x[1'000'000], y[1'000'000], z[1'000'000];
};

for (int i = 0; i < 1'000'000; ++i) {
    particles.x[i] += 1.0f;
}
```

The `x` array is contiguous. SIMD loads are efficient; you load 8 or 16 floats in one instruction.

### 11.5.2 Alignment: Power of Two

SIMD loads are most efficient when the address is aligned to the register width. For AVX (256 bits = 32 bytes), load from an address divisible by 32:

```cpp
alignas(32) float a[1'000'000];
```

If you load from a misaligned address, the CPU takes a penalty (or uses the slow `_loadu_ps` variant).

### 11.5.3 Trip Count is a Multiple of Vector Width

If you process `n` elements and `n % 8 != 0`, you must handle the remainder:

```cpp
void add_avx(const float* a, const float* b, float* c, size_t n) {
    // Main loop: process 8 elements per iteration
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 av = _mm256_loadu_ps(&a[i]);
        __m256 bv = _mm256_loadu_ps(&b[i]);
        __m256 cv = _mm256_add_ps(av, bv);
        _mm256_storeu_ps(&c[i], cv);
    }
    
    // Remainder: process 1–7 elements with scalar code
    for (; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

This is boilerplate but unavoidable.

---

## 11.6 Reading Vectorized Assembly

When you inspect the assembly, you should recognize SIMD instructions.

### Common x86-64 Instructions

- `movups` / `movupd`: Load/store unaligned (packed singles/doubles)
- `movaps` / `movapd`: Load/store aligned (packed singles/doubles)
- `addps` / `addpd`: Add packed singles/doubles
- `mulps` / `mulpd`: Multiply packed singles/doubles
- `fmadd231ps`: Fused multiply-add (a = a + (b * c))
- `vbroadcastss`: Broadcast one scalar to all lanes
- `vperm2f128`: Permute 128-bit lanes within a 256-bit register
- `vpshufd`: Shuffle 32-bit elements
- `blendps`: Conditional blend (select between two registers per lane)

### ARM NEON

- `ldr q0, [x0]`: Load 128-bit from address in x0
- `fadd v0.4s, v0.4s, v1.4s`: Add 4 single-precision floats
- `fmul v0.4s, v0.4s, v1.4s`: Multiply 4 single-precision floats
- `fcvtzs v0.4s, v0.4s`: Convert float to signed int

### Example: Vectorized Dot Product

A dot product in scalar C++:

```cpp
float dot(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) {
        sum += a[i] * b[i];
    }
    return sum;
}
```

Compiled with `-O3` on x86-64, the assembly might look like:

```asm
    xorps   xmm3, xmm3              ; sum0 = 0
    xorps   xmm4, xmm4              ; sum1 = 0
    xorps   xmm5, xmm5              ; sum2 = 0
    xorps   xmm6, xmm6              ; sum3 = 0
.L_loop:
    movups  xmm0, [rdi + rax]       ; Load a[i..i+3]
    movups  xmm1, [rsi + rax]       ; Load b[i..i+3]
    mulps   xmm0, xmm1              ; Multiply 4 pairs
    addps   xmm3, xmm0              ; Accumulate into sum0
    
    movups  xmm0, [rdi + rax + 16]  ; Load a[i+4..i+7]
    movups  xmm1, [rsi + rax + 16]  ; Load b[i+4..i+7]
    mulps   xmm0, xmm1
    addps   xmm4, xmm0              ; Accumulate into sum1
    
    ; ... (repeat for sum2, sum3 if unrolled 4 times)
    
    add     rax, 32
    cmp     rax, rcx
    jl      .L_loop
    
    addps   xmm3, xmm4              ; Fold partial sums
    addps   xmm3, xmm5
    addps   xmm3, xmm6
```

Notice: four independent accumulators (`xmm3`, `xmm4`, `xmm5`, `xmm6`). This breaks the dependency chain, allowing the CPU to execute multiple `addps` instructions in parallel.

---

## 11.7 Worked Example: Dot Product with Intrinsics

Let's implement a dot product of two float arrays, benchmark it, and compare three versions: scalar, auto-vectorized, and manual AVX.

### Scalar Baseline

```cpp
#include <vector>
#include <cstdio>
#include <chrono>

float dot_scalar(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) {
        sum += a[i] * b[i];
    }
    return sum;
}

int main() {
    constexpr size_t N = 10'000'000;
    std::vector<float> a(N), b(N);
    
    // Initialize
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i);
        b[i] = static_cast<float>(i + 1);
    }
    
    // Benchmark
    auto t0 = std::chrono::steady_clock::now();
    float result = 0.0f;
    for (int iter = 0; iter < 100; ++iter) {
        result = dot_scalar(a.data(), b.data(), N);
    }
    auto t1 = std::chrono::steady_clock::now();
    
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
    std::printf("Scalar: %.2f ms, result = %f\n", ms / 100.0, result);
}
```

Compile and run:

```bash
clang++ -O3 -std=c++20 dot_scalar.cpp -o dot_scalar
./dot_scalar
```

Expected output (on a 2020+ CPU):

```
Scalar: 1.23 ms, result = 333333283333500000.000000
```

### Auto-Vectorized Version

The same loop, compiled with `-O3`, auto-vectorizes. To verify, add the flag:

```bash
clang++ -O3 -fopt-info-vec -std=c++20 dot_scalar.cpp
```

Output will confirm: `vectorized loop (vectorization factor: 4)` or similar.

Performance: typically **3–4× faster** than the scalar baseline with no code changes.

### Manual AVX Version

```cpp
#include <vector>
#include <cstdio>
#include <chrono>
#include <immintrin.h>  // AVX

float dot_avx(const float* a, const float* b, size_t n) {
    __m256 sum_v = _mm256_setzero_ps();  // Four accumulators, all zero
    
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 av = _mm256_loadu_ps(&a[i]);
        __m256 bv = _mm256_loadu_ps(&b[i]);
        __m256 prod = _mm256_mul_ps(av, bv);  // Multiply 8 pairs
        sum_v = _mm256_add_ps(sum_v, prod);    // Accumulate
    }
    
    // Horizontal reduction: sum all 8 elements in the vector
    // Shuffle and add to fold the sum
    __m128 sum_high = _mm256_extractf128_ps(sum_v, 1);
    __m128 sum_low = _mm256_castps256_ps128(sum_v);
    sum_low = _mm_add_ps(sum_low, sum_high);
    
    __m128 sum_shuf = _mm_shuffle_ps(sum_low, sum_low, _MM_SHUFFLE(2, 3, 0, 1));
    sum_low = _mm_add_ps(sum_low, sum_shuf);
    sum_shuf = _mm_shuffle_ps(sum_low, sum_low, _MM_SHUFFLE(1, 0, 3, 2));
    sum_low = _mm_add_ps(sum_low, sum_shuf);
    
    float result = _mm_cvtss_f32(sum_low);
    
    // Handle remainder
    for (; i < n; ++i) {
        result += a[i] * b[i];
    }
    
    return result;
}

int main() {
    constexpr size_t N = 10'000'000;
    std::vector<float> a(N), b(N);
    
    for (size_t i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i);
        b[i] = static_cast<float>(i + 1);
    }
    
    auto t0 = std::chrono::steady_clock::now();
    float result = 0.0f;
    for (int iter = 0; iter < 100; ++iter) {
        result = dot_avx(a.data(), b.data(), N);
    }
    auto t1 = std::chrono::steady_clock::now();
    
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
    std::printf("AVX: %.2f ms, result = %f\n", ms / 100.0, result);
}
```

Compile:

```bash
clang++ -O3 -std=c++20 dot_avx.cpp -o dot_avx
./dot_avx
```

Expected output:

```
AVX: 0.25 ms, result = 333333283333500000.000000
```

**Speedup:** ~4.9× over scalar. The auto-vectorized version was already 3–4× faster, and manual AVX buys an additional 20–30% by using the wider register (8 floats vs 4).

---

## 11.8 When SIMD Is the Wrong Tool

SIMD is powerful, but it is not always the bottleneck. Before reaching for intrinsics, ask:

### 1. Is the algorithm fundamentally inefficient?

If you are iterating 100 million times when you could iterate 1 million with a smarter algorithm, SIMD cannot save you. Optimize the algorithm first.

Example: summing a linked list. SIMD cannot parallelize irregular, pointer-chasing access. Restructure the data as an array first.

### 2. Are cache misses the real bottleneck?

Profile with `perf`:

```bash
perf record -e L1-dcache-load-misses,cycles ./your_program
perf report
```

If the L1 miss rate is 30%, SIMD will not help; you are bandwidth-limited, and the data layout is the problem (see Chapter 8). Fix the cache behavior first.

### 3. Is the loop iteration count small?

If you loop 100 times, the overhead of setting up SIMD vectors and doing horizontal reduction is not worth the speedup.

### 4. Is the code path frequently executed?

If the function is called once per second, optimizing it with SIMD is wasted effort. Profile. Focus on the hot paths (Chapter 7).

### 5. Are there unpredictable branches?

```cpp
for (size_t i = 0; i < n; ++i) {
    if (a[i] > threshold) {
        result[i] = expensive_computation(a[i]);
    } else {
        result[i] = 0.0f;
    }
}
```

If the branch is unpredictable, SIMD gains are minimal. The CPU cannot execute 8 branches in parallel if it does not know which lane will take which branch. Restructure to avoid branches (use conditional moves, max, etc.).

---

## 11.9 Tradeoffs

| Approach | Effort | Speedup | Portability | Maintenance |
|----------|--------|---------|-------------|-------------|
| Scalar code | Minimal | 1× (baseline) | Universal | Easy |
| Auto-vectorized (`-O3`) | None | 3–4× | x86/ARM with compiler support | Easy |
| `#pragma omp simd` | Low | 2–6× | GCC/Clang/MSVC | Moderate |
| `std::simd` | Low | 4–8× | C++26, experimental | Easy (future-proof) |
| Intrinsics (`_mm256_*`) | High | 4–32× | Single ISA (x86, ARM, etc.) | Hard (ISA-specific) |
| Google Highway | Medium | 4–32× | Universal (SSE, AVX, NEON, SVE) | Moderate |
| GPU/ISPC | Very high | 100–1000× | Specialist hardware | Hard |

---

## 11.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "SIMD is just for HPC and floating-point." | SIMD works on integers, booleans, and pointers. Any parallel-friendly operation benefits. |
| "The compiler will always vectorize my loop." | Only if the loop meets strict conditions: no aliasing, no dependencies, contiguous access, predictable trip count. |
| "Intrinsics are portable." | Intrinsics are portable across compilers (on the same ISA) but not across architectures. Code for AVX will not compile on ARM. |
| "I should use intrinsics for every loop." | Most of the time, auto-vectorization or `std::simd` suffice. Intrinsics are for performance-critical code where the compiler fails. |
| "SIMD always gives you 8× speedup." | Speedup depends on data layout, trip count, instruction latency, memory bandwidth, and CPU cache behavior. Typical gains are 3–8×, not 32×. |
| "SIMD is better than algorithmic improvements." | No. If you can cut the iteration count in half with a smarter algorithm, that beats 4× SIMD speedup. Prioritize: algorithm first, then cache, then SIMD. |
| "Aligned memory is always necessary." | Aligned loads are faster, but unaligned loads (`_loadu_ps`) work fine. They incur a small penalty. For critical loops, enforce alignment. |

---

## 11.11 Exercises

1. **Verify auto-vectorization on your machine.** Write a simple loop (e.g., element-wise addition). Compile with `-O3 -fopt-info-vec`. Confirm the vectorization factor. Then add `restrict` pointers or modify the loop to break vectorization, and recompile to see the remark disappear.

2. **Profile a real loop.** Find or write a loop that iterates over 1 million floats. Measure with scalar code, then enable `-O3` and measure again. Calculate the speedup. (Expected: 3–5×.)

3. **Write manual SIMD.** Implement a dot product with manual AVX intrinsics (see §11.7). Benchmark against auto-vectorized and scalar. Document the horizontal reduction (the trickiest part).

4. **Measure alignment impact.** Allocate two arrays: one aligned to 32 bytes, one unaligned. Fill both with data, then benchmark a tight loop with `_mm256_load_ps` (aligned) and `_mm256_loadu_ps` (unaligned). Measure the performance difference.

5. **Identify why the compiler refused.** Find a loop in your code that does not vectorize. Compile with `-fopt-info-vec-missed` to see the reason. Fix one issue (e.g., add `restrict`, remove data dependency, simplify control flow) and recompile to confirm vectorization.

6. **Explore Google Highway.** Download Highway, read the documentation, and rewrite a simple function (e.g., element-wise multiply, 1D convolution) with Highway. Benchmark on your platform. Note that Highway abstracts the ISA, so the same code works on SSE, AVX, NEON, and SVE.

---

## 11.12 Summary

SIMD is a CPU feature that executes one instruction on multiple independent data values in parallel. On modern x86-64, AVX processes 8 floats per instruction; on ARM NEON, 4 floats per instruction. With the right data layout (SoA, contiguous, aligned), even simple loops can see 3–8× speedups with no algorithmic changes.

The compiler auto-vectorizes many loops at `-O3`, but only if conditions are met: no pointer aliasing, no data dependencies between iterations, contiguous memory access, and predictable trip count. When auto-vectorization fails, you can use `#pragma omp simd`, `std::simd`, or low-level intrinsics.

SIMD is a tool, not a universal answer. Before reaching for intrinsics, profile. Fix the algorithm first, cache locality second, SIMD third. A well-designed scalar loop outperforms a poorly-designed vectorized one.

---

## 11.13 What's Next

Now you understand three levels of the performance stack: memory layout (Chapter 8), branch prediction (Chapter 9), and SIMD (this chapter). These are orthogonal optimizations; each can yield 2–8× speedups independently, and they compound.

The next chapter descends one more level: cache misses and the memory hierarchy. You will learn what "poor cache locality" really means, how to measure it, and how to fix it using profilers and access pattern analysis. After that, we close Part 8 with real-world performance engineering: profiling entire systems, finding bottlenecks, and making data-driven optimization decisions.

---

**[← Previous: Branch Prediction](10-branch-prediction.md)** · **[↑ Part 8](README.md)** · **[Next: Cache Misses (Deeper) →](12-cache-misses.md)**
