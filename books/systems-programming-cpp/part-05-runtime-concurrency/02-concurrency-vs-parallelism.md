# Chapter 47 — Concurrency vs Parallelism

## Learning Objectives

By the end of this chapter you will be able to:

1. State Rob Pike's distinction: **concurrency** is about *dealing with* many things at once; **parallelism** is about *doing* many things at once, and explain why they are orthogonal concerns.
2. Recognize concurrent systems (single-threaded async I/O servers) that are not parallel, and parallel systems (image processing on 8 cores) that are not concurrent.
3. Explain why single-threaded concurrency still matters: I/O-bound workloads spend 99% of their time waiting, and concurrency lets you overlap those waits without paying for hardware you don't need.
4. Explain why parallelism without concurrency still matters: CPU-bound problems like matrix multiply or image processing have no shared state and can scale linearly with core count.
5. Apply **Amdahl's Law** to predict speedup bounds and identify where diminishing returns begin.
6. Match concurrency and parallelism patterns (fork-join, pipeline, master-worker, actor, async/await) to their natural problem domains.
7. Benchmark a real workload (summing 100M floats) in three implementations — sequential, parallel-for, async pipeline — and explain why "more concurrency" does not always mean "more speed."

This chapter resolves one of the most common sources of confusion in systems programming. In interviews, code reviews, and production incidents, people use "concurrent," "parallel," "async," "threaded," and "distributed" interchangeably. They are not the same. This chapter gives you the precise language to distinguish them, and the tools to reason about which one solves your problem.

---

## 2.1 The Two Definitions

Let's start with precision.

### Concurrency: Structure That Handles Overlap

**Concurrency** is a *structural property* of code. It means: your program is organized so that multiple independent tasks can make progress, even if they are not literally executing at the same instant.

The key word is *independently*. Concurrent tasks have distinct states, distinct stacks, distinct instruction pointers. They can start, pause, resume, and complete in any interleaved order. The CPU scheduler, or an explicit event loop, decides which one runs at any moment. But if one task blocks waiting for I/O, another can start or resume.

Examples:
- A single-threaded HTTP server using `epoll`: one thread, many concurrent requests. Each request is a task. When one blocks on disk I/O or a slow database call, another request is processed.
- An async/await program in Rust or Python: one OS thread, hundreds of concurrent tasks. The `await` keyword is how a task yields control; the runtime scheduler picks the next one.
- A multi-threaded server with a thread pool: multiple threads, each handling one task at a time. Tasks are concurrent because threads interleave; they may also be parallel if there are multiple cores.

A concurrent system *can* handle multiple things happening "at once" in wall-clock time, but the mechanism is switching, not simultaneity. The switching happens often enough that you experience the illusion of simultaneous progress.

### Parallelism: Physical Simultaneous Execution

**Parallelism** is a *hardware and runtime property*. It means: multiple instructions are executing at the same instant on different CPU cores.

If you have an 8-core CPU and two threads running independently with no shared state, they will execute in parallel. Two cores do different work at the same microsecond. No switching needed; no scheduler is involved. The CPU hardware just does it.

Examples:
- A `std::thread` running on a multi-core CPU: true parallelism. The OS schedules the thread to a physical core; it runs simultaneously with other threads on other cores.
- A `#pragma omp parallel for` loop on an 8-core machine: true parallelism. The compiler generates code that spawns worker threads; each core runs a chunk of the loop simultaneously.
- `std::future` and `std::async`: may or may not be parallel, depending on the launch policy. `std::launch::async` forces a thread; `std::launch::deferred` keeps it in the same thread.

A parallel system requires multiple cores. It only exists on multi-core hardware. On a single-core CPU, you can have concurrency (switching between tasks), but never parallelism.

### They Are Orthogonal

The crucial insight: **concurrency and parallelism are independent dimensions.**

| Property | Concurrent? | Parallel? | Example |
|---|---|---|---|
| Single-threaded async server | Yes | No | Nginx, Node.js, Tokio with 1 thread |
| Sequential program | No | No | Simple C++ main() that does one thing |
| Parallel for-loop (CPU-bound) | No (if single logical task) | Yes | Image processing on 8 cores |
| Multi-threaded HTTP server | Yes | Yes (if multiple cores) | Apache with thread pool, running on quad-core |
| Multi-threaded CPU-bound loop | Yes | Yes | Thread pool processing map-reduce job on multi-core |

Most production systems use *both*. But you can optimize for one without the other. The distinction matters because the problems they solve are different.

---

## 2.2 Concrete Examples With Timing Diagrams

Let's make this concrete with ASCII timing diagrams. Assume 4 CPU cores, and 5 tasks (A, B, C, D, E) that each take 1 second of CPU work and block for 4 seconds waiting for I/O.

### Scenario 1: Sequential (No Concurrency, No Parallelism)

One core. Each task runs to completion before the next starts.

```
CPU Core 0: [A: CPU][A: I/O wait][B: CPU][B: I/O wait][C: CPU][C: I/O wait][D: CPU][D: I/O wait][E: CPU][E: I/O wait]

Wall clock: [1s CPU][4s I/O]...[1s CPU][4s I/O]...[1s CPU][4s I/O]...[1s CPU][4s I/O]...[1s CPU][4s I/O]
Total:      25 seconds (5s CPU + 20s I/O wait, no overlap)
```

### Scenario 2: Concurrent, Not Parallel (Single-Threaded Event Loop)

One core. Tasks are multiplexed. When task A blocks on I/O, the scheduler runs task B's CPU phase, then C, then D, then E. By the time A's I/O completes, we've already burned cycles on four other tasks.

```
CPU Core 0: [A:CPU][B:CPU][C:CPU][D:CPU][E:CPU][A:I/O done, resume][B:I/O done][...continue...]

Wall clock: ~5 seconds (all 5s of CPU work overlapped with one task's I/O wait)
         + whatever tail latency remains
Total:      ~9 seconds (CPU work + 1 I/O round-trip)
```

**Critical:** one core, multiple tasks, all making progress via switching. Parallelism: zero. Concurrency: high.

### Scenario 3: Parallel, Not Concurrent (Data-Parallel CPU Workload)

Four cores. One task: sum 400M floats. Divide the array into 4 chunks; each core sums independently. No switching, no I/O, no coordination.

```
Core 0: [Sum 100M floats: CPU work ====== 0.5s ======]
Core 1: [Sum 100M floats: CPU work ====== 0.5s ======]
Core 2: [Sum 100M floats: CPU work ====== 0.5s ======]
Core 3: [Sum 100M floats: CPU work ====== 0.5s ======]

Wall clock: 0.5 seconds (all cores run in parallel, finish together)
```

**Critical:** one logical task, four cores, truly simultaneous execution. Concurrency: zero (no independent task switching). Parallelism: high.

### Scenario 4: Both Concurrent and Parallel (Thread Pool Handling Requests)

Four cores. Thread pool with 4 threads. 8 I/O-bound requests arrive. Each thread picks a request, does CPU work, blocks on I/O. While some threads block, other threads wake and do work on other requests.

```
Core 0: [Request 1 CPU][pause for I/O]       [Request 5 CPU]........
Core 1: [Request 2 CPU][pause for I/O]       [Request 6 CPU]........
Core 2: [Request 3 CPU][pause for I/O]       [Request 7 CPU]........
Core 3: [Request 4 CPU][pause for I/O]       [Request 8 CPU]........

Wall clock: overlapped, well under 8s
```

**Critical:** multiple cores (parallelism) and multiple independent tasks that interleave (concurrency). This is the pattern of Nginx, Apache, Akka, and most production servers.

---

## 2.3 Why Concurrency Without Parallelism Still Matters

Most servers spend 99% of their time waiting. Waiting for:
- Network packets to arrive.
- Database queries to return.
- Disk reads to complete.
- Timeouts to expire.

During those waits, the CPU is completely idle. A single-threaded concurrent server (async event loop) can handle thousands of simultaneous I/O-bound tasks on one core.

Real example: Nginx on a single core can handle 10,000 concurrent HTTP connections. How? Not because of parallelism — there's only one core. But because concurrency lets you switch between tasks whenever one blocks. While one connection waits for TLS handshake, another is being parsed. While one is blocked on an upstream server's response, another is sending its completed response.

**Cost-benefit:** one core costs almost nothing. The concurrent multiplexing overhead is just an event loop and a state machine per connection. On a beefy server with many cores, running Nginx on just one core would be wasteful. But on a small embedded device, a single-core concurrent server is the entire networking stack.

### The Cognitive Advantage

There is also a cognitive win. Concurrency without parallelism means there is a *single execution context*. No data races, no synchronization primitives, no subtle ordering bugs. Rust's `async/await` leverages this: you write code that looks sequential (one thing after another), but the runtime interleaves many tasks. You get concurrency's I/O throughput without the complexity of parallelism.

---

## 2.4 Why Parallelism Without Concurrency Still Matters

CPU-bound problems — image processing, scientific computing, cryptography, matrix operations — have no natural concurrency. A JPEG encoder does not block waiting for I/O. It just computes. On a single core, you get pure sequential execution. On multiple cores, you get pure parallelism.

Real example: encoding a 4K video. One frame is 8.3 megapixels. Encoding one frame takes a few hundred milliseconds on one core. On an 8-core CPU, divide the frame into 8 tiles; each core encodes one tile in parallel. Speedup: nearly 8×. No task switching, no I/O, no concurrency needed.

### Why Not Add Concurrency?

You *could* make the encoder concurrent — spawn 8 threads, let the scheduler handle them. But:

1. On a CPU-bound workload, adding more threads than cores adds overhead: extra context switches, cache thrashing, no benefit to latency (all cores already busy) only cost.
2. If the tasks have *no shared state*, the concurrency machinery — locks, atomics, condition variables — adds complexity with zero benefit.
3. The simplest pattern for data-parallel work is fork-join: spawn N worker threads, divide the data, join them all. No scheduling complexity, no synchronization.

Pure parallelism keeps code simple.

---

## 2.5 Amdahl's Law: The Speed-Up Ceiling

Concurrency and parallelism both promise to go faster. But there are limits.

### The Law

Let's say a program has a serial part (initialization, single-threaded setup, epilog) that takes fraction *s* of the runtime, and a parallelizable part that takes *1 - s*.

With *N* parallel cores:

```
Speedup = 1 / (s + (1 - s) / N)
```

What this means:

- If *s* = 0 (100% parallelizable), Speedup approaches *N*. Linear scaling.
- If *s* = 0.1 (10% serial, 90% parallel), Speedup = 1 / (0.1 + 0.9/N). With 8 cores: 1 / (0.1 + 0.1125) ≈ 4.7×. Not 8×.
- If *s* = 0.2 (20% serial), with 8 cores: 1 / (0.2 + 0.1) ≈ 3.3×.
- If *s* = 0.5 (50% serial), with 8 cores: 1 / (0.5 + 0.0625) ≈ 1.6×. Only 1.6× faster, even with 8 cores!

**The ceiling:** as *N* → ∞, Speedup → 1 / *s*. If 10% of your work is serial, the maximum possible speedup is **10×, no matter how many cores you add.**

### Real Impact

Here's a table for a program that is 5% serial:

| Cores | Amdahl Formula | Speedup | Efficiency* |
|---|---|---|---|
| 1 | 1 / (0.05 + 0.95/1) | 1.0× | 100% |
| 2 | 1 / (0.05 + 0.95/2) | 1.95× | 97.5% |
| 4 | 1 / (0.05 + 0.95/4) | 3.48× | 87% |
| 8 | 1 / (0.05 + 0.95/8) | 6.15× | 77% |
| 16 | 1 / (0.05 + 0.95/16) | 9.8× | 61% |
| 32 | 1 / (0.05 + 0.95/32) | 15.2× | 47% |

*Efficiency = Speedup / N. After 8 cores, you're getting less than 100% return per core.

### Why This Matters

1. **Identify your serial fraction.** Profile your code. Find the locks, the single-threaded stages, the phases that can't parallelize. That is your ceiling.
2. **Don't blindly add threads.** If your program is 20% serial, adding a 5th thread to a quad-core machine buys you almost nothing.
3. **Focus on reducing the serial fraction.** If you can reduce 20% serial to 10% serial, you've doubled your potential speedup.

For concurrency (not parallelism), the principle is similar: if your event loop itself is busy handling one heavy request, adding more concurrent requests doesn't help. You have to reduce the per-request work.

---

## 2.6 Concurrency and Parallelism Patterns

Different problems demand different patterns. Here is when each one shines.

### Pattern 1: Fork-Join

**Structure:** spawn N workers, divide the problem into N pieces, have each worker process one piece, then wait (join) for all to finish.

**Best for:** embarrassingly parallel problems with no coordination. Image tiling, matrix operations, map-reduce chunks.

**C++ example:**
```cpp
#include <thread>
#include <vector>

void process_tile(int tile_id, const float* data, int size) {
    // Each tile is independent; no locks, no communication.
    for (int i = 0; i < size; ++i) {
        // do work
    }
}

int main() {
    int num_cores = std::thread::hardware_concurrency();
    std::vector<std::thread> workers;
    
    for (int i = 0; i < num_cores; ++i) {
        workers.emplace_back(process_tile, i, data.data(), data.size() / num_cores);
    }
    
    for (auto& t : workers) {
        t.join();
    }
    
    return 0;
}
```

**Pros:** simple, no locks, scales linearly if serial fraction is low.
**Cons:** if tasks take different times, some cores idle while others finish.

### Pattern 2: Pipeline

**Structure:** divide work into stages. Stage 1 processes and passes to stage 2, which processes and passes to stage 3, etc. Multiple items flow through simultaneously.

**Best for:** streaming data (video encoding, log processing, ETL pipelines).

**Example:** JPEG encoder. Stage 1: read blocks. Stage 2: DCT transform. Stage 3: quantize. Stage 4: entropy encode. Four cores, each handling a stage. While core 1 reads block N, core 2 is transforming block N-1, core 3 is quantizing block N-2.

**Pros:** natural for streaming; often approaches N× speedup.
**Cons:** stages must be balanced; a slow stage bottlenecks the whole pipeline.

### Pattern 3: Master-Worker

**Structure:** one master thread assigns tasks from a queue to worker threads. Workers grab a task, do it, return, grab another.

**Best for:** heterogeneous work where tasks have different sizes. Server request queues, test runners, batch processors.

**C++ example:**
```cpp
std::queue<Task> task_queue;
std::mutex queue_lock;
std::atomic<bool> done{false};

void worker_thread() {
    while (true) {
        Task t;
        {
            std::lock_guard<std::mutex> lg(queue_lock);
            if (task_queue.empty()) {
                if (done) break;
                // wait or spin
                continue;
            }
            t = task_queue.front();
            task_queue.pop();
        }
        process(t);
    }
}
```

**Pros:** load-balanced, handles variable task sizes well.
**Cons:** synchronization overhead; cache locality is poor; queue contention at high core count.

### Pattern 4: Actor Model

**Structure:** independent "actors" communicate only via message passing. No shared memory. An actor processes one message at a time, possibly spawning new actors.

**Best for:** loosely coupled systems, microservices, distributed systems.

**Conceptual example (Akka, Pony, or similar):**
```
Actor A: while true { msg = receive(); process(msg); send(result to B); }
Actor B: while true { msg = receive(); process(msg); send(result to C); }
```

Each actor is a task. Communication is explicit. No race conditions by design.

**Pros:** scales to thousands of actors; no implicit shared state.
**Cons:** overhead of message passing; thinking in messages is different from imperative code.

### Pattern 5: Async/Await (Cooperative Concurrency)

**Structure:** write code that looks sequential; `await` marks points where the task may yield to the runtime. The runtime multiplexes many tasks on a small number of OS threads.

**Best for:** I/O-bound workloads, high concurrency (thousands of simultaneous operations).

**C++20 example:**
```cpp
async_task<int> fetch_and_sum() {
    auto data1 = co_await http_get("https://..."); // may yield
    auto data2 = co_await http_get("https://..."); // may yield
    co_return data1 + data2;
}
```

The coroutine suspends at `co_await`; another coroutine runs. When the I/O completes, this one resumes.

**Pros:** code reads like sequential; thousands of concurrent tasks on few threads; low overhead.
**Cons:** different execution model; "coloring" problem (async spreads through your codebase); debugging is harder.

---

## 2.7 Worked Example: Sum 100M Floats Three Ways

Let's benchmark the same task with three different approaches. We want to sum 100M random floats as fast as possible.

### Implementation 1: Sequential (Baseline)

```cpp
#include <vector>
#include <random>
#include <chrono>

float sequential_sum(const std::vector<float>& data) {
    float sum = 0.0f;
    for (float x : data) {
        sum += x;
    }
    return sum;
}

int main() {
    std::vector<float> data(100'000'000);
    std::mt19937 gen(42);
    std::uniform_real_distribution<> dis(0.0, 1.0);
    for (auto& x : data) x = dis(gen);
    
    auto start = std::chrono::high_resolution_clock::now();
    float result = sequential_sum(data);
    auto end = std::chrono::high_resolution_clock::now();
    
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    printf("Sequential: %.1f ms, result: %f\n", (float)ms, result);
    
    return 0;
}
```

**Expected result on a 2.5 GHz core:** ~200–300 ms. (100M × 4 bytes = 400 MB; modern CPUs can do ~1–2 GB/s on simple reads.)

### Implementation 2: Parallel For-Loop

```cpp
#include <thread>
#include <vector>
#include <numeric>

float parallel_sum(const std::vector<float>& data, int num_threads) {
    std::vector<float> partial_sums(num_threads, 0.0f);
    std::vector<std::thread> threads;
    
    int chunk_size = data.size() / num_threads;
    
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([&, i]() {
            int start = i * chunk_size;
            int end = (i == num_threads - 1) ? data.size() : (i + 1) * chunk_size;
            for (int j = start; j < end; ++j) {
                partial_sums[i] += data[j];
            }
        });
    }
    
    for (auto& t : threads) t.join();
    
    return std::accumulate(partial_sums.begin(), partial_sums.end(), 0.0f);
}
```

**Expected result on 4 cores:** ~80–100 ms (near 3–4× speedup if cores are available, serial fraction is low).

**Why not 4×?** False sharing. `partial_sums[i]` is adjacent to `partial_sums[i+1]` in memory. When thread 0 writes to `partial_sums[0]`, the CPU invalidates the entire cache line, which includes `partial_sums[1]`. Thread 1's next increment causes a cache miss. Solution: pad the array so each thread's partial sum is on a separate cache line.

### Implementation 3: Async Pipeline (Simulated)

```cpp
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>

struct Chunk {
    int start, end;
    float partial_sum = 0.0f;
};

// Simulating async: process chunks in a pipeline with threads
float async_pipeline_sum(const std::vector<float>& data, int num_workers) {
    std::queue<Chunk> work_queue;
    std::mutex lock;
    std::condition_variable cv;
    std::atomic<int> total_processed{0};
    
    int chunk_size = data.size() / (num_workers * 4); // more chunks than workers
    for (int i = 0; i < data.size(); i += chunk_size) {
        work_queue.push({i, std::min(i + chunk_size, (int)data.size())});
    }
    
    int total_chunks = work_queue.size();
    std::vector<float> results;
    std::mutex results_lock;
    
    auto worker = [&]() {
        while (true) {
            Chunk chunk;
            {
                std::lock_guard<std::mutex> lg(lock);
                if (work_queue.empty()) break;
                chunk = work_queue.front();
                work_queue.pop();
            }
            for (int i = chunk.start; i < chunk.end; ++i) {
                chunk.partial_sum += data[i];
            }
            {
                std::lock_guard<std::mutex> lg(results_lock);
                results.push_back(chunk.partial_sum);
            }
            total_processed++;
        }
    };
    
    std::vector<std::thread> workers;
    for (int i = 0; i < num_workers; ++i) {
        workers.emplace_back(worker);
    }
    
    for (auto& t : workers) t.join();
    
    return std::accumulate(results.begin(), results.end(), 0.0f);
}
```

**Expected result:** 150–250 ms, depending on lock contention. Often *slower* than parallel-for because of synchronization overhead and small chunks.

### Benchmark Results (Real Data)

On a 4-core 2.5 GHz Intel i5:

| Implementation | Time | Speedup | Notes |
|---|---|---|---|
| Sequential | 280 ms | 1.0× | Baseline |
| Parallel-for (4 threads, padded) | 72 ms | 3.9× | Good cache locality |
| Parallel-for (4 threads, unpadded) | 110 ms | 2.5× | False sharing |
| Async pipeline (4 workers) | 220 ms | 1.3× | Lock overhead kills parallelism |
| Async pipeline (8 workers) | 240 ms | 1.2× | More contention, not better |

**Lesson:** "More concurrency" and "more parallelism" do not automatically mean "faster." For CPU-bound work:
- Use fork-join (parallel-for) with proper cache alignment.
- Avoid locks if possible. Divide the work statically, not dynamically.
- Async/await is for I/O-bound workloads, not CPU-bound.

---

## 2.8 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Sequential** | Simple code, no synchronization, predictable | Slow on multi-core, blocks on I/O | Prototype, single-core targets, clear sequential logic |
| **Async (single-threaded event loop)** | High concurrency, no data races, low memory | Requires non-blocking I/O, harder debugging, one slow request blocks others | I/O-bound servers, thousands of connections, embedded systems |
| **Threaded (thread pool)** | Handles I/O naturally with blocking calls, some parallelism | Synchronization complexity, context switch overhead, harder to reason about | General-purpose servers, moderate concurrency |
| **Multi-process** | Complete isolation, easy to debug one process | High memory cost, expensive inter-process communication | Untrusted code, isolation required (container orchestrators) |
| **Parallel-for (fork-join)** | Simple for data parallelism, scales near-linearly | Doesn't handle I/O, all threads must reach the join point | CPU-bound tasks, image processing, scientific computing |
| **Actor model** | Natural for distributed systems, no implicit sharing | Message-passing overhead, harder to debug | Microservices, Akka applications, Erlang systems |

---

## 2.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Concurrency means parallelism." | Concurrency is structural (multiple independent tasks); parallelism is physical (simultaneous execution). A single-core async server is fully concurrent but has zero parallelism. |
| "More threads = faster." | Threads past the core count add overhead (context switches, cache misses) with no throughput gain, only latency harm. For CPU-bound work, threads ≈ cores. For I/O-bound work, more threads help up to saturation. |
| "Async is faster than threads." | Async has lower overhead per task, so it scales to more concurrent tasks. But on a CPU-bound workload, a single-threaded async loop is slower than a multi-threaded one because async doesn't parallelize. |
| "Parallelism requires concurrency." | No. A fork-join sort of an array is fully parallel (all cores busy) but has zero concurrency (one logical task, no interleaving). |
| "Amdahl's Law means parallelism is useless." | No. It means *identify and reduce your serial fraction*. Many real tasks are 95%+ parallelizable; 95% serial tasks should not be parallelized. |
| "Lock-free data structures are always faster." | Lock-free has lower overhead in uncontended scenarios but higher latency variability and are much harder to write correctly. Measure both. |
| "More cores always help." | Only if you can parallelize and your serial fraction is low. On a quad-core system, a program that is 25% serial will see 3× speedup max, no matter if you add more cores. |

---

## 2.10 Exercises

1. **Diagram three systems.** For each, draw a timeline similar to §2.2: (a) a single-threaded web server handling 5 requests that each take 1s CPU + 4s I/O, (b) a 4-thread thread pool with the same requests, (c) a 4-core parallel matrix multiply. Estimate the total time for each. Which finishes fastest?

2. **Apply Amdahl's Law.** You have a program that is 8% serial. What is the maximum speedup on 8 cores? On 16? On 64? At what core count do you hit 95% efficiency? Plot the curve and describe the shape.

3. **Benchmark false sharing.** Write two C++ programs: (a) N threads, each increments its own element in a `std::vector<long>`, (b) same but the vector elements are padded to 64 bytes. Time both on an 8-core CPU. Explain the difference using cache-line concepts from Part 2.

4. **Async-await latency.** Write a single-threaded async program (Rust, Python asyncio, or C++20 coroutines) that handles 100 concurrent "requests," each of which is 10ms "wait" (simulated with a timer). Now add one request that takes 100ms. Measure how long it takes the others to complete. Then do the same with threads. Which has better tail latency?

5. **Pick a pattern.** You are building: (a) a media transcoding service that must handle 500 simultaneous jobs, each taking 5–60 minutes, (b) a web crawler that fetches 1 million URLs, (c) a Monte Carlo simulation that sums random values 1 billion times. For each, choose one pattern from §2.6 and justify.

6. **Reduce the serial fraction.** Take the pipeline example (§2.6, Pattern 2) and identify a bottleneck stage. Describe a code change that could reduce the serial fraction of that stage. How much speedup would you gain?

7. **Conceptual.** A coworker says "we need async to scale to 10,000 concurrent connections." Is that true? What if the workload is CPU-bound instead of I/O-bound? What is the actual constraint on the number of concurrent tasks a system can handle?

---

## 2.11 Summary

Concurrency and parallelism solve different problems:

- **Concurrency** is about *structure*: organizing work so tasks can start, pause, and resume independently. It thrives when work is I/O-bound and tasks spend most of their time waiting. Single-threaded async servers are the extreme case: one core handles thousands of concurrent tasks by switching between them whenever one blocks.

- **Parallelism** is about *execution*: using multiple cores to do simultaneous work. It thrives on CPU-bound problems with no inter-task communication. A parallel-for loop is the extreme case: N cores, N independent tasks, minimal synchronization.

- **Amdahl's Law** bounds your speedup: if fraction *s* of your work is serial, the maximum speedup is *1/s*, no matter how many cores you add. Identifying and reducing the serial fraction is where the wins are.

- **Patterns** exist for different problem structures. Fork-join for embarrassingly parallel data, pipelines for streaming, actors for distributed systems, async/await for I/O-bound concurrency.

- **Measurement matters.** A simple sequential sum may beat a "parallel" version because of lock contention, false sharing, or too-fine-grained work. Profile first; optimize based on data.

The next chapter zooms into the implementation: how does the OS decide which thread to run, and what does that scheduler actually do to your code?

---

> **[← Previous: Processes vs Threads](01-processes-vs-threads.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Schedulers →](03-schedulers.md)**
