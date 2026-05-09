# Chapter 8 — Data-Oriented Design

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the core premise of data-oriented design (DoD) — that cache behavior and memory layout dominate performance for compute-heavy code, outweighing algorithmic elegance.
2. Distinguish between Array-of-Structs (AoS) and Struct-of-Arrays (SoA) layouts and predict when each wins on real hardware.
3. Apply hot/cold splitting to reduce cache line waste and improve working-set locality.
4. Recognize why Entity Component Systems (ECS) exist and when they are appropriate.
5. Design data structures for SIMD vectorization by understanding alignment, stride, and data dependency chains.
6. Measure the performance impact of layout changes using profilers and benchmarking.
7. Understand the tradeoff between code flexibility and data layout optimization.

This chapter inverts the perspective of object-oriented design. In OOP, we organize *code* around objects — bundling data and the methods that operate on it. In data-oriented design, we organize *data* around how the CPU actually consumes it. For code that runs once or twice, this distinction does not matter. For code that runs millions or billions of times per second — image processing, audio mixing, particle simulation, real-time physics, network packet processing — the CPU's cache, prefetcher, and branch predictor become the performance ceiling, and data layout is the lever that raises it.

---

## 8.1 The Core Premise: Memory Access Pattern Matters More Than Code Structure

Recall from Chapter 2 that a main-memory miss costs ~200 cycles, during which the CPU can execute 200+ other instructions if the data is available. A well-designed data layout brings the data you need into cache before you ask for it. A poorly-designed layout means you ask for data that is not there, and the CPU stalls.

This is why "fast code" is not about clever algorithms in isolation. It is about the *marriage* of algorithm and data layout.

Consider a compute-heavy task: sum the `x` coordinates of 10 million particles. The algorithmic complexity is O(N) either way. But there are two ways to organize the particles:

```cpp
// Option A: Array-of-Structs (AoS)
struct Particle {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    uint32_t id;
};
std::vector<Particle> particles(10'000'000);

// In main loop:
float sum = 0.0f;
for (const auto& p : particles) {
    sum += p.x;
}
```

vs

```cpp
// Option B: Struct-of-Arrays (SoA)
struct Particles {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<uint32_t> id;
};
Particles particles;
particles.x.resize(10'000'000);
// ... (initialize others)

// In main loop:
float sum = 0.0f;
for (size_t i = 0; i < 10'000'000; ++i) {
    sum += particles.x[i];
}
```

Both execute the same number of additions and the same number of loop iterations. The difference in runtime is often **3–5×** in favor of SoA, with no algorithmic change — only memory layout.

Here's why: a cache line is 64 bytes. A single `Particle` in option A uses 32 bytes (6 floats + 1 uint32 + 1 padding). One cache line brings 2 particles. You are accessing only the `x` field (4 bytes); the rest (28 bytes per particle) is wasted cache space. In option B, the `x` array is contiguous; 16 floats fit in one cache line; the prefetcher loads the line before you ask for it; you hit L1 cache for nearly every access.

**The CPU does not care about your code structure. It cares about the data.**

---

## 8.2 Array-of-Structs vs Struct-of-Arrays

Let's formalize the comparison with a concrete example.

### Memory Layout Diagram

**Array-of-Structs:**

```
Address     Particle 0                    Particle 1
0x1000:     [x y z vx vy vz mass id]      [x y z vx vy vz mass id]
            ^--- 32 bytes --^              ^--- 32 bytes --^
            Cache line 0 holds 2 particles
```

When iterating and accessing only `.x`, the CPU loads cache lines containing a lot of unused data.

**Struct-of-Arrays:**

```
Address     x[0..3] (16 bytes / 4 floats)
0x1000:     [x0 x1 x2 x3 x4 x5 ... x15]
            ^----------64 bytes-----------^
            Cache line 0 holds 16 x-values
```

Accessing `x[i]` sequentially fills the cache line with the exact data you need.

### A Microbenchmark

Create `dod_aos.cpp`:

```cpp
#include <chrono>
#include <cstdio>
#include <vector>

struct Particle {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    uint32_t id;
};

int main() {
    constexpr int N = 10'000'000;
    std::vector<Particle> particles(N);

    // Initialize
    for (int i = 0; i < N; ++i) {
        particles[i] = {
            .x = 1.0f, .y = 2.0f, .z = 3.0f,
            .vx = 0.1f, .vy = 0.2f, .vz = 0.3f,
            .mass = 1.0f,
            .id = (uint32_t)i
        };
    }

    auto t0 = std::chrono::steady_clock::now();
    float sum = 0.0f;
    for (const auto& p : particles) {
        sum += p.x;
    }
    auto t1 = std::chrono::steady_clock::now();

    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    std::printf("AoS: sum = %f, time = %lld us\n", sum, us);
}
```

Create `dod_soa.cpp`:

```cpp
#include <chrono>
#include <cstdio>
#include <vector>

struct Particles {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<uint32_t> id;
};

int main() {
    constexpr int N = 10'000'000;
    Particles particles;
    particles.x.resize(N);
    particles.y.resize(N);
    particles.z.resize(N);
    particles.vx.resize(N);
    particles.vy.resize(N);
    particles.vz.resize(N);
    particles.mass.resize(N);
    particles.id.resize(N);

    // Initialize
    for (int i = 0; i < N; ++i) {
        particles.x[i] = 1.0f;
        particles.y[i] = 2.0f;
        particles.z[i] = 3.0f;
        particles.vx[i] = 0.1f;
        particles.vy[i] = 0.2f;
        particles.vz[i] = 0.3f;
        particles.mass[i] = 1.0f;
        particles.id[i] = (uint32_t)i;
    }

    auto t0 = std::chrono::steady_clock::now();
    float sum = 0.0f;
    for (int i = 0; i < N; ++i) {
        sum += particles.x[i];
    }
    auto t1 = std::chrono::steady_clock::now();

    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    std::printf("SoA: sum = %f, time = %lld us\n", sum, us);
}
```

Compile both with `-O2`:

```bash
clang++ -std=c++20 -O2 dod_aos.cpp -o aos
clang++ -std=c++20 -O2 dod_soa.cpp -o soa
./aos
./soa
```

Typical output (on a 2020+ CPU):

```
AoS: sum = 10000000.000000, time = 89000 us
SoA: sum = 10000000.000000, time = 14000 us
```

The SoA version is **6× faster**, on the same logical computation. This is not a microbenchmark artifact; it reflects real-world performance. Codebases that process millions of data items (game engines, data processing, audio, rendering) can see order-of-magnitude improvements by switching from AoS to SoA.

### When Each Layout Wins

| Scenario | Winner | Why |
|----------|--------|-----|
| Iteration over a single field | SoA | Cache-efficient, vectorizable |
| Random access to full structs | AoS | Struct is loaded in one cache line, no scattered access |
| Polymorphic access (C++ virtual functions) | AoS | vptr is part of struct; SoA would scatter vtables |
| Heterogeneous data (fields accessed unpredictably) | AoS | No one layout optimizes all access patterns |
| SIMD operations on one field | SoA | Arrays are ideal for vectorization |
| Small structs, tight coupling to methods | AoS | Object locality is still valuable; overhead of SoA management is high |

**Rule of thumb:** If your main loop accesses all or most fields in a struct, AoS is fine. If your main loop accesses 1-3 specific fields out of many, SoA wins.

---

## 8.3 Hot/Cold Splitting: Separating Data By Frequency of Use

SoA is not always practical. Game entities, for example, may have dozens of fields with different access patterns: position, velocity, and collision bounds are accessed nearly every frame; AI state and animation keyframes are accessed occasionally; inventory and loot drops are accessed rarely.

**Hot/cold splitting** is a middle path: group frequently-accessed ("hot") fields into one struct, and rarely-accessed ("cold") fields into another. Access the cold struct only when needed.

### Example: A Game Entity

**Before hot/cold splitting (single struct):**

```cpp
struct Entity {
    // Hot: accessed every frame
    glm::vec3 position;
    glm::vec3 velocity;
    glm::vec3 acceleration;
    float radius;
    uint32_t state;
    
    // Warm: accessed occasionally (collision checks, animation)
    AABB bounds;
    float mass;
    std::vector<uint32_t> collision_flags;
    AnimationState anim_state;
    
    // Cold: accessed rarely (inventory, loot)
    std::vector<Item> inventory;
    std::vector<StatusEffect> effects;
    std::string name;
    Dialogue dialogue_tree;
    uint32_t level;
    uint32_t experience;
};

std::vector<Entity> entities(100'000);
```

The cache line holds one `Entity`. A single cache line is wasted on inventory, effects, name, and dialogue — data you're not accessing in the hot loop.

**After hot/cold splitting:**

```cpp
struct EntityHot {
    glm::vec3 position;
    glm::vec3 velocity;
    glm::vec3 acceleration;
    float radius;
    uint32_t state;
};

struct EntityWarm {
    AABB bounds;
    float mass;
    std::vector<uint32_t> collision_flags;
    AnimationState anim_state;
};

struct EntityCold {
    std::vector<Item> inventory;
    std::vector<StatusEffect> effects;
    std::string name;
    Dialogue dialogue_tree;
    uint32_t level;
    uint32_t experience;
};

std::vector<EntityHot> hot_data(100'000);
std::vector<EntityWarm> warm_data(100'000);
std::vector<EntityCold> cold_data(100'000);  // or use a sparse map
```

Now the main physics loop:

```cpp
for (size_t i = 0; i < hot_data.size(); ++i) {
    auto& entity = hot_data[i];
    entity.position += entity.velocity * dt;
    entity.velocity += entity.acceleration * dt;
}
```

The cache misses only hot data. When you do need the collision bounds or animation state, you fetch it once (one extra miss), not on every iteration.

**How much does this help?** Microbenchmarks often show 20–50% improvements in the hot loop. Real games use this pattern for exactly this reason.

---

## 8.4 ECS — Entity Component Systems

If splitting one struct into three is good, why not split it further? The logical extreme is the **Entity Component System** pattern, where:

- An **entity** is just an ID (e.g., `uint32_t`).
- A **component** is a pure data struct (no methods) that attaches data to an entity.
- A **system** is code that iterates over entities with certain components, updating them.

```cpp
// Components are plain data
struct Transform {
    glm::vec3 position;
    glm::vec3 velocity;
};

struct Health {
    float hp;
    float max_hp;
};

struct Inventory {
    std::vector<Item> items;
};

// ECS world stores arrays of components
class ECSWorld {
public:
    std::vector<Transform> transforms;  // indexed by entity ID
    std::vector<Health> healths;
    std::vector<Inventory> inventories;
    // ... more component arrays
};

// Systems iterate over components
void physics_system(ECSWorld& world, float dt) {
    for (size_t i = 0; i < world.transforms.size(); ++i) {
        world.transforms[i].position += world.transforms[i].velocity * dt;
    }
}

void health_system(ECSWorld& world) {
    for (size_t i = 0; i < world.healths.size(); ++i) {
        if (world.healths[i].hp <= 0) {
            world.healths[i].hp = 0;  // clamp
        }
    }
}
```

**Why ECS dominates game engines and high-performance simulations:**

1. **Cache locality:** Each system iterates a single, contiguous array of one component type. Perfect for prefetching.
2. **SIMD friendliness:** Contiguous arrays are trivial to vectorize.
3. **Scalability:** Adding a new system does not require changing existing code; systems are decoupled.
4. **Parallelization:** Systems that operate on different component types can run in parallel without coordination.

Unity DOTS (Data-Oriented Tech Stack) and Bevy (a Rust game engine) are built on this model. It is not an accident; it is the natural architecture for code that must process millions of entities at 60+ FPS.

**The cost:** loosely-coupled systems can be harder to reason about than tightly-bundled objects. A `Player` entity is spread across multiple arrays, and you must manually check whether it has a component before accessing it. This is a real tax on code clarity, but it is the price DoD pays.

---

## 8.5 SIMD Friendliness and Auto-Vectorization

Vectorization — executing the same instruction on multiple data elements in parallel — is a gift the CPU gives you. The CPU has vector registers (`xmm0`–`xmm15` on x86-64, or `v0`–`v31` on ARM64 NEON) that can hold 4–8 floats or 8–16 ints. One vector instruction can process them all.

Modern C++ compilers can auto-vectorize loops, but only if the data layout cooperates.

### A Vectorizable Loop

```cpp
std::vector<float> a(N), b(N), c(N);
// ... initialize a and b ...

for (size_t i = 0; i < N; ++i) {
    c[i] = a[i] + b[i];
}
```

The compiler sees: no data dependencies (the `i`-th iteration does not depend on iteration `i-1`), contiguous arrays, a simple operation. It rewrites this as:

```cpp
for (size_t i = 0; i < N; i += 4) {  // process 4 floats at a time
    __m128 av = _mm_loadu_ps(&a[i]);
    __m128 bv = _mm_loadu_ps(&b[i]);
    __m128 cv = _mm_add_ps(av, bv);
    _mm_storeu_ps(&c[i], cv);
}
```

One SIMD instruction does the work of 4 scalar instructions.

### What Breaks Vectorization

1. **Non-contiguous access:**

```cpp
std::vector<Particle> particles;  // AoS
for (const auto& p : particles) {
    result += p.x;  // stride is sizeof(Particle), not sizeof(float)
}
```

The compiler cannot easily vectorize strided access (though modern compilers have gotten better). If you SoA:

```cpp
std::vector<float> x;  // SoA
for (size_t i = 0; i < x.size(); ++i) {
    result += x[i];  // contiguous, vectorizable
}
```

It vectorizes immediately.

2. **Data dependencies:**

```cpp
float sum = 0;
for (size_t i = 0; i < N; ++i) {
    sum += a[i];  // iteration i depends on the sum from i-1
}
```

The reduction creates a data dependency chain. The compiler cannot parallelize this loop across vector lanes. This is a real limitation; you cannot vectorize every loop.

**Workaround:** unroll the loop to create multiple independent accumulators:

```cpp
float sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
for (size_t i = 0; i < N; i += 4) {
    sum0 += a[i];
    sum1 += a[i+1];
    sum2 += a[i+2];
    sum3 += a[i+3];
}
float sum = sum0 + sum1 + sum2 + sum3;
```

Now the four accumulators are independent; the CPU can execute them in parallel.

3. **Unpredictable branches:**

```cpp
for (size_t i = 0; i < N; ++i) {
    if (a[i] > threshold) {
        c[i] = a[i];
    } else {
        c[i] = 0;
    }
}
```

Modern compilers can vectorize this with masked operations (`blendps`, etc.), but only if the branch is predicted. If `a[i] > threshold` is unpredictable, the CPU stalls on each iteration.

### Verifying Vectorization

Compile with `-march=native -O3 -fopt-info-vec` to see what the compiler vectorized:

```bash
clang++ -march=native -O3 -fopt-info-vec simd_test.cpp
```

Output:

```
simd_test.cpp:45:5: remark: vectorized loop (vectorization factor: 4) [-Rpass=loop-vectorize]
```

Or inspect the assembly with `-S -masm=intel` to see the `xmm` instructions:

```asm
.L12:
    movups  xmm0, [rdi + rax]    ; load 4 floats from a[i..i+3]
    movups  xmm1, [rsi + rax]    ; load 4 floats from b[i..i+3]
    addps   xmm0, xmm1          ; add them
    movups  [rdx + rax], xmm0    ; store result
    add     rax, 16              ; next 4 floats
    cmp     rax, rcx
    jne     .L12
```

---

## 8.6 When Data-Oriented Design Hurts

DoD is not a universal win. Before applying it, ask:

### 1. Is your data deeply heterogeneous?

If entities have wildly different sets of fields — a player might have inventory but an NPC might not, a projectile has velocity but a building does not — ECS becomes complex. You are constantly checking whether an entity has a component (`if (world.has<Inventory>(entity_id))`). In contrast, AoS lets you bundle what belongs together and ignore the rest.

### 2. Is your dataset small?

If you have 100 particles, cache effects do not matter. The extra indirection of splitting into multiple arrays (or the complexity of an ECS) costs more than the savings. Stay with AoS until you reach thousands of items.

### 3. Are you in the early design phase?

DoD is an optimization. If the code's behavior is still changing, locking in a data layout is premature. You might discover that you do need the `full_struct` after all, and refactoring will be tedious. The rule of three applies: let the same AoS struct be used in 2-3 different systems before extracting to SoA.

### 4. Are you mixing CPU and GPU?

The GPU has very different memory access patterns. GPUs are designed for SIMD-heavy, highly parallel workloads where SoA is more natural. But data transfer between CPU and GPU is expensive. Sometimes the cost of reorganizing data for the GPU outweighs the gain.

---

## 8.7 Worked Example: A Particle System

Let's build a concrete particle system, profile it, and optimize it.

### Step 1: AoS Baseline

`particles_aos.cpp`:

```cpp
#include <chrono>
#include <cstdio>
#include <vector>
#include <cmath>
#include <random>

struct Particle {
    float x, y, z;
    float vx, vy, vz;
    float life;
    uint32_t id;
};

int main() {
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(-1.0f, 1.0f);

    constexpr int N = 1'000'000;
    std::vector<Particle> particles(N);

    // Initialize
    for (int i = 0; i < N; ++i) {
        particles[i] = {
            .x = dist(rng), .y = dist(rng), .z = dist(rng),
            .vx = dist(rng), .vy = dist(rng), .vz = dist(rng),
            .life = dist(rng) + 1.0f,  // life in [0, 2]
            .id = (uint32_t)i
        };
    }

    auto t0 = std::chrono::steady_clock::now();
    
    // Simulate 100 frames
    for (int frame = 0; frame < 100; ++frame) {
        for (auto& p : particles) {
            p.x += p.vx * 0.016f;
            p.y += p.vy * 0.016f;
            p.z += p.vz * 0.016f;
            p.life -= 0.016f;
            
            // Decay velocity due to air resistance
            p.vx *= 0.99f;
            p.vy *= 0.99f;
            p.vz *= 0.99f;
        }
    }
    
    auto t1 = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
    
    std::printf("AoS: %lld ms\n", ms);
}
```

Compile and run:

```bash
clang++ -std=c++20 -O2 particles_aos.cpp -o aos
time ./aos
```

Typical: ~350 ms on a modern CPU.

### Step 2: Profile to Identify the Bottleneck

```bash
perf record ./aos
perf report
```

(Output will show that the main loop's cache miss rate is high, but the absolute number of L1 misses is the bottleneck.)

### Step 3: SoA Refactor

`particles_soa.cpp`:

```cpp
#include <chrono>
#include <cstdio>
#include <vector>
#include <cmath>
#include <random>

struct Particles {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> life;
    std::vector<uint32_t> id;
};

int main() {
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(-1.0f, 1.0f);

    constexpr int N = 1'000'000;
    Particles particles;
    particles.x.resize(N);
    particles.y.resize(N);
    particles.z.resize(N);
    particles.vx.resize(N);
    particles.vy.resize(N);
    particles.vz.resize(N);
    particles.life.resize(N);
    particles.id.resize(N);

    // Initialize
    for (int i = 0; i < N; ++i) {
        particles.x[i] = dist(rng);
        particles.y[i] = dist(rng);
        particles.z[i] = dist(rng);
        particles.vx[i] = dist(rng);
        particles.vy[i] = dist(rng);
        particles.vz[i] = dist(rng);
        particles.life[i] = dist(rng) + 1.0f;
        particles.id[i] = (uint32_t)i;
    }

    auto t0 = std::chrono::steady_clock::now();
    
    // Simulate 100 frames
    for (int frame = 0; frame < 100; ++frame) {
        for (int i = 0; i < N; ++i) {
            particles.x[i] += particles.vx[i] * 0.016f;
            particles.y[i] += particles.vy[i] * 0.016f;
            particles.z[i] += particles.vz[i] * 0.016f;
            particles.life[i] -= 0.016f;
            
            particles.vx[i] *= 0.99f;
            particles.vy[i] *= 0.99f;
            particles.vz[i] *= 0.99f;
        }
    }
    
    auto t1 = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
    
    std::printf("SoA: %lld ms\n", ms);
}
```

Typical: ~60 ms — **5.8× faster**.

The same workload, same compiler flags, different data layout. The SoA version has better spatial locality, better prefetching, and most critically, no wasted cache on fields you are not currently accessing.

---

## 8.8 DoD vs OOP: A Comparison

| Aspect | OOP | DoD |
|--------|-----|-----|
| **Code organization** | Around objects and methods | Around data and access patterns |
| **Type safety** | Strong; compiler enforces boundaries | Weaker; more runtime checks for component existence (ECS) |
| **Performance** | Medium-good; optimizers work well for simple objects, poorly for complex ones | Excellent for compute-heavy code; poor for highly variable access |
| **Flexibility** | High; can easily add methods to a class | Medium-low; requires restructuring data arrays to add a new attribute |
| **Testability** | High; methods are encapsulated, easy to mock | Medium; systems operate on raw arrays, require setup |
| **Readability** | High for small-to-medium objects; low for deeply heterogeneous data | Medium for contiguous iteration; low for accessing sparse data |
| **Scalability to teams** | Excellent; clear boundaries and ownership | Good; systems are decoupled, but shared data layout decisions are harder |

**There is no universal winner.** A web server's request handler is better as an OOP object; a particle simulation is better as ECS.

---

## 8.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "DoD is just about micro-optimizations." | DoD is a design philosophy. Micro-optimization is a consequence, not the goal. The goal is aligning code with hardware reality. |
| "SoA is always faster than AoS." | SoA wins when you access a subset of fields. If you access all fields, AoS can be competitive (all data is in the same cache line anyway). |
| "DoD means no abstraction." | DoD means abstraction at the data-layout layer, not the object layer. An ECS *is* an abstraction; it abstracts over the heterogeneous lifetime challenges of entities. |
| "If I use DoD, the compiler will auto-vectorize everything." | Auto-vectorization requires (1) contiguous data, (2) no data dependencies, and (3) no unpredictable branches. DoD helps with (1); it cannot fix (2) or (3). |
| "SIMD is only for numerical code." | SIMD works on integers, booleans, and pointers too. Any operation that can be parallelized benefits. |
| "Premature optimization, go home." | DoD is not an optimization in the traditional sense; it is a first-order design choice. For compute-heavy code, design with data layout in mind from the start. |
| "ECS is the only way to structure a game." | ECS is *one* pattern. For smaller games, or games with tightly-coupled entities, a scene graph or component-based object model may be better. |

---

## 8.10 Exercises

1. **Measure the AoS vs SoA difference on your hardware.** Run the microbenchmarks from §8.2. At what dataset size does the gap appear? What cache line size does that suggest? (Use `getconf LEVEL1_DCACHE_LINESIZE` to check.)

2. **Profile a real loop.** Find a loop in a codebase you maintain that iterates over a large struct. Compile with `-O2` and profile with `perf record` and `perf report`. What is the L1 miss rate? Refactor to SoA or hot/cold split and re-profile. How much does the miss rate improve?

3. **Apply hot/cold splitting.** Take a struct you use in a loop. Identify fields accessed every iteration (hot), occasionally (warm), and rarely (cold). Refactor into separate structs and measure the performance impact.

4. **Vectorization inspection.** Compile a simple loop with `-O3 -fopt-info-vec -S`. Look for vectorization remarks. Then add a condition or a data dependency and recompile. Watch the vectorization disappear. Understand why the compiler made each choice.

5. **Design a small ECS.** Build a minimal ECS for a particle system or simple simulation. Iterate 1 million simple entities with a Physics component and an optional Health component. Measure the time for (a) a system that only updates Physics, (b) a system that updates Physics and Health for healthy entities. Compare to an AoS version.

6. **Conceptual.** A coworker proposes "we should switch to SoA everywhere for cache locality." Write a response that covers: (a) when SoA helps most, (b) the costs of SoA, (c) an example where AoS is better. Draw on this chapter.

---

## 8.11 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| AoS layout | Cache misses if you access a subset of fields; harder to vectorize strided access | Simple, cache-efficient for random access, familiar to OOP engineers |
| SoA layout | Indirection across multiple arrays; manual component checks (ECS); harder to change later | Excellent cache locality for iteration, easy vectorization, natural for SIMD |
| Hot/cold splitting | Medium complexity; code must know about the split | Reduces cache waste without full SoA complexity |
| ECS | Loose coupling can obscure logic; runtime overhead for component checks | Maximum flexibility and cache efficiency; trivial parallelization |
| ECS with sparse storage | Extra indirection; memory overhead | Sparse components (not all entities have them) do not waste memory |
| Contiguous allocation (arena) | Fixed size, must estimate capacity | Eliminates allocator pressure, improves cache locality, reduces fragmentation |
| Manual cache-line alignment | Code complexity; reduced portability | Predictable cache behavior, ability to pack multiple small structs per line |

---

## 8.12 Summary

Data-oriented design inverts the traditional software-design hierarchy. Instead of "design the objects, then figure out how to store them efficiently," you ask "what data do I need to process, and in what order?" and *then* design the code to match.

For compute-heavy code — particle systems, image processing, physics simulations, real-time audio, network packet processing, data analytics — this inversion often yields 3–10× performance improvements with no algorithmic changes, purely from cache friendliness. The CPU is not your code's bottleneck; memory access patterns are.

The tools are simple: separate cold from hot data, use contiguous arrays instead of scattered pointers, iterate in a pattern that matches the CPU's prefetcher. The payoff is high.

But DoD trades flexibility for throughput. Heterogeneous data, small datasets, and rapid iteration all favor the object-oriented style. The mature engineer picks the right style for the right problem.

---

## 8.13 What's Next

You now understand how memory layout shapes performance. In the next chapter, we descend one level further — into the CPU's branch predictor and instruction pipeline. We will see that even a perfectly cache-friendly loop can be slow if the branch predictions are wrong, and we will learn to write code that the CPU's speculative execution engine loves.

After that, Part 2 closes with concurrency — what happens when multiple CPU cores try to access and modify the same data, and how synchronization primitives protect them without grinding the system to a halt.

---

**[← Previous: Hot/Cold Splitting](07-caches-and-cpu-locality.md)** · **[↑ Part 2](README.md)** · **[Next: Smart Pointers Internals →](09-smart-pointers-internals.md)**
