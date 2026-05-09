# Chapter 48 — Schedulers

A scheduler is the mechanism that decides which thread or task runs next on a given CPU core. On a busy system, thousands of tasks want to run but only a handful of cores exist. Every concurrent system has at least one scheduler — sometimes several layered. The scheduler's choices shape latency, fairness, and throughput in ways that remain invisible until something breaks.

Most developers interact with schedulers without thinking about them: you spawn a thread, the OS schedules it. You await a coroutine, the async runtime schedules it. But understanding *how* schedulers work — their tradeoffs, their blind spots, their failure modes — separates those who can reason about concurrent systems from those who can only profile and hope.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the difference between **preemptive** and **cooperative** scheduling, and why each model has fans and critics.
2. Describe how **Linux CFS** (Completely Fair Scheduler) actually assigns CPU time using vruntime, weight, and the runqueue.
3. Recognize the four patterns of schedulers in modern runtimes: **OS preemptive** (kernel threads), **async cooperative** (Tokio, Erlang), **M:N hybrid** (Go), and **work-stealing** (thread pools).
4. Explain why **I/O integration** matters: how non-blocking I/O and epoll/kqueue let a scheduler move one thread's wait time off the CPU.
5. Trace a concrete scenario: a CPU-bound task and an I/O-bound task competing for one core, and predict which one the scheduler favors.
6. Identify scheduling problems in production systems using `top`, `pidstat`, and `perf sched`.

---

## 48.1 The Scheduler's Job

At its core, a scheduler solves a multiplexing problem: *given more tasks than CPUs, which one runs now?*

The answer has consequences:

- **Latency.** How long does a high-priority task wait before getting CPU time?
- **Throughput.** Do we finish more total work per unit time, or do we waste time context-switching?
- **Fairness.** Do all tasks get a fair share, or can one hog the CPU?
- **Responsiveness.** Does the system feel snappy to a user, or laggy?

A scheduler cannot optimize all of these simultaneously. It must choose. That choice is architecture.

### The universal model

Nearly all schedulers follow this structure:

1. **Runnable queue**: maintain a list of tasks ready to run (waiting for CPU, not blocked on I/O or a lock).
2. **Running state**: one task per CPU core is currently executing.
3. **Blocking events**: when a task blocks (on I/O, a lock, a sleep), it leaves the running state and enters a blocked queue.
4. **Wakeup**: when the blocking event completes, the task re-enters the runnable queue.
5. **Preemption or yielding**: the running task either yields voluntarily or is forcibly interrupted (preempted) to make room for another.

The *structure* of the runnable queue — is it global? Per-CPU? A lock-free data structure? — and the *policy* for picking the next task from it are where schedulers differ.

---

## 48.2 Preemptive vs Cooperative Scheduling

### Preemptive Scheduling

In **preemptive** scheduling, the scheduler can interrupt a running task *at any point* in its execution. The OS sets a timer; when it fires, the CPU enters an interrupt handler, saves the current task's state, and invokes the scheduler to pick the next one.

Example: Linux on a single core. A task is running. A timer fires every 4 milliseconds. The kernel saves registers, picks another task, loads its registers, resumes. The original task is stopped mid-instruction (conceptually) and has no say in it.

**Pros:**

- No task can starve others by hogging the CPU.
- Fair: each task gets slices in rough proportion to its priority weight.
- Simple to reason about: you don't need to think about where in your code another task might preempt you.
- Responsive: a high-priority task can run even if a lower-priority one was scheduled first.

**Cons:**

- Context switches are expensive: save registers, load page table, flush TLB, cold cache.
- Hard to debug: a race condition can happen between any two instructions; reproducing it requires running long enough to hit the unlucky timing.
- Requires privilege: only the kernel can set timers and interrupt execution. User code cannot control when it is preempted.

### Cooperative Scheduling

In **cooperative** scheduling, a task must explicitly *yield* control before another task runs. It yields by making a blocking syscall (`read`, `write`, `epoll_wait`), by calling an explicit `yield()` function, or by awaiting an async operation.

Example: Go's goroutines or Rust's async/await. A goroutine runs until it performs a channel operation, a function call marked `go`, or a blocking syscall. At that point, the Go runtime scheduler takes a chance to pick another goroutine. Between those yield points, nothing else runs.

**Pros:**

- No context-switch overhead: the CPU stays in the same address space, TLB is warm, cache is warm.
- Simple to debug: you know exactly where other tasks can run (at yield points marked in code).
- Scales to many more tasks: a process can have millions of goroutines; only thousands of threads.
- Deterministic: given the same input and yield sequence, the schedule is the same (important for testing).

**Cons:**

- One task can starve others: if a task does CPU-bound work without yielding, it holds the CPU forever.
- Requires discipline: the programmer must ensure frequent yield points, or the system becomes unresponsive.
- Incompatible with blocking syscalls: if a goroutine calls a blocking syscall without the runtime knowing, it blocks the entire OS thread it's running on, and other goroutines on that thread are stuck.
- Complex implementation: the runtime must coordinate yielding with I/O readiness; async runtimes like Tokio are intricate.

### In Practice

Most modern systems are **hybrid**:

- **OS level:** preemptive kernel-level threads. The kernel schedules threads via timers.
- **Language level:** cooperative tasks or coroutines on top. Go's runtime multiplexes M goroutines onto N OS threads using cooperative scheduling among goroutines, but preemption at the OS thread level (since Go threads are kernel threads).

The boundary between layers is important: if the bottom layer is preemptive and the top is cooperative, you get responsiveness from preemption (the OS will interrupt a runaway goroutine when its OS thread's timeslice expires) but efficiency from cooperation (most of the time, goroutines yield cleanly without forcing a context switch).

---

## 48.3 OS Schedulers: Linux CFS

Linux's **Completely Fair Scheduler** (CFS) is the dominant preemptive kernel scheduler on Linux. Understanding it illuminates how real production systems work.

### The core idea: vruntime

CFS does not track absolute time. Instead, it tracks **vruntime** — "virtual runtime," a weighted measure of how much CPU time a process has consumed.

The formula:

```
vruntime = actual_runtime / weight
```

where `weight` depends on priority (nice value on Linux). Higher-priority processes have higher weights; their vruntime accumulates slower for the same amount of actual runtime.

The scheduler maintains a **red-black tree** of all runnable processes, ordered by vruntime. The process with the *smallest* vruntime gets to run next. While it runs, its actual runtime increases; its vruntime increases accordingly (at a rate determined by its weight). Eventually, another process's vruntime becomes smaller; the scheduler preempts and switches.

### Example

Two processes: A (priority 0, weight 1024) and B (priority -10, weight 2048, higher priority).

Both start with vruntime = 0.

- A runs for 1 millisecond: vruntime = 1 / 1024 ≈ 0.001.
- B runs for 1 millisecond: vruntime = 1 / 2048 ≈ 0.0005.

B's vruntime is smaller, so if both are runnable, B gets picked next. Over time, B accumulates more actual runtime while keeping its vruntime lower, ensuring it gets more CPU percentage. *Fairness through fairness.*

### Preemption

Every few milliseconds (by default, about once per millisecond per core), a timer interrupt fires. The kernel checks: does the currently running process have the smallest vruntime among all runnable processes? If not, preempt and switch.

The key: preemption is *frequent* (millisecond-scale), so responsiveness is good, but not so frequent that the overhead dominates. A balance.

### Priority and nice

You control a process's weight via the **nice value** (range -20 to +19). Lower (more negative) values are higher priority. `nice -10 ./my-program` runs `my-program` with higher priority, and it will get more CPU time relative to other processes.

**But nice is relative fairness**, not absolute. It does not mean "get 99% of CPU." It means "get CPU time in proportion to your weight." On a single core, if A has weight 1024 and B has weight 2048, and both are always runnable, A gets roughly 1/(1+2) = 33% of CPU and B gets 66%.

### cgroups and CPU shares

For containerization (Docker, Kubernetes), Linux provides **cgroups** (control groups). You can set per-container CPU quotas (e.g., "this container gets 50% of CPU") and the kernel enforces it. This is a higher-level abstraction built on top of CFS weights.

---

## 48.4 Cooperative Async Runtimes

Go, Tokio (Rust), and Erlang each implement *language-level* schedulers on top of OS threads. The strategy is always: use a small pool of OS threads, and multiplex many more user-level tasks onto them cooperatively.

### Go's M:N Scheduler

Go has M goroutines multiplexed onto N OS threads (M >> N). The runtime maintains a global runqueue and per-thread local runqueues.

When a goroutine makes a blocking syscall, the Go runtime knows about it (it wraps syscalls) and does not block the OS thread. Instead:

1. The goroutine issues the syscall.
2. The Go runtime stashes the goroutine in a "blocked syscall" state.
3. The OS thread picks another goroutine from the queue and resumes running.
4. When the blocked syscall completes, the Go runtime wakes the goroutine and puts it back in a runqueue.

This trick — intercepting syscalls and not blocking the OS thread — is why Go can efficiently run hundreds of thousands of goroutines. Each one is very cheap: just a stack and some bookkeeping, not a full OS thread.

**But there's a catch:** if a goroutine does CPU-bound work without yielding, it holds its OS thread. Other goroutines on that thread do not run until the kernel preempts the thread (after a few milliseconds). Modern Go inserts "preemption points" in compiled code to force yields more frequently.

### Tokio's Work-Stealing Scheduler

Tokio is an async runtime for Rust. Tasks are lightweight futures that yield control when they await I/O or other async operations.

Tokio's scheduler uses **work-stealing**: each worker thread has its own queue of tasks. When a thread runs out of tasks, instead of idling, it *steals* tasks from another thread's queue. This balances load without a global lock.

```
Thread 0 queue: [T1, T2, T3, T4, T5]
Thread 1 queue: [T6, T7]
Thread 2 queue: []

Thread 2 is idle. It steals from Thread 0's queue:
Thread 2 queue: [T3, T4, T5]  (stolen from end of Thread 0)
Thread 0 queue: [T1, T2]
```

Work-stealing is elegant because:

- Most of the time, threads work on their own queue (cache-hot, no synchronization).
- When threads go idle, they naturally rebalance load.
- No global lock is needed; only the stealing thread and the victim thread interact.

---

## 48.5 Work Stealing in Detail

Work-stealing is worth understanding because it is the dominant pattern in modern parallel systems (Java's ForkJoinPool, Rust's Rayon, thread-pool libraries everywhere). Let's see why it scales.

### The naive approach: global queue

```
          [Global Lock]
              |
          [Task Queue]
         /    |    \
    Thread0 Thread1 Thread2
```

Every thread contends on a single queue. With N threads, lock contention grows. On a 16-core machine, the lock becomes a bottleneck. Throughput plateaus.

### Work-stealing approach

```
    Thread0       Thread1       Thread2
    [Queue]       [Queue]       [Queue]
       |             |             |
      Idle → Steal → Victim
```

Each thread has its own queue. Most operations are on the local queue (no synchronization). When a thread goes idle, it selects a victim (another thread) and steals half of that thread's tasks.

The stealing thread and the victim thread need to coordinate (via atomic operations), but this is rare: only when the stealer goes idle, which is infrequent if tasks are plenty. Most of the time, threads run uncontended.

**Scaling property:** work-stealing scales nearly linearly up to the number of cores, because the common case (local queue access) is lock-free.

### ASCII diagram

```
Time 0:
T0: [Task, Task, Task] (busy)    T1: [Task] (busy)    T2: [] (idle)

Time 1:
T0: [Task, Task] (busy)          T1: [Task] (busy)    T2: [Task] (busy, stole)

Time 2:
T0: [Task] (busy)                T1: [Task] (busy)    T2: [Task] (busy)
```

---

## 48.6 I/O and Schedulers

One of the most important ideas: **the scheduler must know about I/O readiness.** Otherwise, it can waste CPU scheduling a task that cannot run because it's waiting for a network packet.

### Blocking I/O: the naïve approach

```cpp
// Thread 0
int data = read(socket_fd, buf, 1024);  // Blocks. Thread is unrunnable.
process(data);

// Thread 1
int data = read(other_fd, buf, 1024);   // Also blocks.

// OS sees: 2 threads runnable, but both are blocked in syscalls.
// The scheduler keeps them on the CPU, burning cycles, waiting for I/O.
// No other task can run (even if we want it to).
```

This is wasteful. On a web server with 1000 connections, you'd have 1000 threads, all blocked, all consuming scheduler overhead.

### Non-blocking I/O + epoll/kqueue

The better approach: mark FDs non-blocking and use a **multiplexer** (epoll on Linux, kqueue on BSD/macOS).

```cpp
// Single thread
fcntl(fd1, F_SETFL, O_NONBLOCK);
fcntl(fd2, F_SETFL, O_NONBLOCK);
fcntl(fd3, F_SETFL, O_NONBLOCK);

epoll_fd = epoll_create();
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd1, ...);
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd2, ...);
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd3, ...);

while (true) {
    epoll_event events[64];
    int n = epoll_wait(epoll_fd, events, 64, -1);  // Wait for *any* FD to be ready.
    for (int i = 0; i < n; ++i) {
        int fd = events[i].data.fd;
        if (events[i].events & EPOLLIN) {
            read(fd, buf, 1024);  // Won't block; FD is ready.
        }
    }
}
```

Now: one thread waits on the syscall `epoll_wait`, which blocks until *any* of the 1000 FDs becomes ready. When it does, `epoll_wait` returns, and we can read from that FD without blocking. The OS scheduler sees one thread (the event-loop thread), not 1000.

**This is why Nginx (a server handling millions of connections) is so efficient: it uses epoll and handles many connections with a small number of threads.**

### async/await integration

In async runtimes like Tokio or Python asyncio, the integration is tight:

```rust
// Tokio
let data = read_socket(fd).await;  // When await is hit, the task yields.
process(data);

// Internally:
// 1. read_socket returns a Future that is not yet ready.
// 2. await checks: is the future ready? No. Yield the task.
// 3. The runtime puts the task in the epoll "wait for this FD" list.
// 4. epoll_wait is called for all tasks waiting on I/O.
// 5. When the FD is ready, epoll wakes, and the runtime resumes this task.
```

The beauty: **the task's yield point aligns with OS I/O readiness.** No busy-waiting, no polls, no wasted context switches. The scheduler integrates with the OS's multiplexer.

---

## 48.7 Priority and Fairness

Different systems make different choices about priority and fairness.

### Real-time priorities

Some tasks are latency-critical: a robot's actuator control, a video game's render loop, a trading system's order execution. For these, **real-time (RT) scheduling classes** exist.

On Linux, `SCHED_FIFO` (first in, first out) and `SCHED_RR` (round-robin) are real-time classes. A real-time task *always* runs ahead of a non-real-time task, regardless of when they arrived.

```bash
chrt -f 1 ./critical-task &      # FIFO, priority 1 (high)
chrt -f 50 ./less-critical &     # FIFO, priority 50 (lower within FIFO)
./normal-task &                  # Default, lower than both
```

**Caution:** giving a task real-time priority is powerful and dangerous. If a real-time task spins in a loop, the entire system becomes unresponsive.

### Fairness vs responsiveness

CFS tries to be **fair**: each task gets CPU time proportional to its weight. But fairness and responsiveness can conflict:

- A large batch job (e.g., a 10-second matrix multiplication) is "fair" if it gets its 10 seconds of CPU time alongside other processes.
- But a user hitting a key on a terminal expects a response in *milliseconds*, not waiting for the batch job to yield.

Real systems add **IO scheduling classes** to the kernel:

- `SCHED_NORMAL` (CFS, default): fair to all tasks.
- `SCHED_BATCH`: even more fair; intended for batch jobs. Slightly lower responsiveness.
- `SCHED_IDLE`: run only when nothing else is runnable. Vacuum the disk cache in the background? Use `SCHED_IDLE`.

---

## 48.8 Worked Example: CPU-Bound vs I/O-Bound on One Core

Let's trace what happens when two tasks compete on a single CPU.

**Setup:**

- CPU-bound task A: runs a loop, no I/O, no yielding.
- I/O-bound task B: issues HTTP requests, waits for responses.
- One CPU core.
- Linux CFS scheduler with default timeslice (about 3 ms per task).

**Sequence:**

1. Time 0 ms: A and B both runnable. A's vruntime = 0, B's vruntime = 0. A is picked (arbitrary tie-break).

2. Time 0-3 ms: A runs on the CPU. Its vruntime increases to ~0.003 (assuming weight 1024).

3. Time 3 ms: Timer fires. Scheduler checks: does B (vruntime 0) have smaller vruntime than A (vruntime 0.003)? Yes. Preempt A, run B.

4. Time 3 ms: B issues a read syscall (network I/O). B is marked sleeping. The scheduler looks for another runnable task. Only A is runnable. Run A.

5. Time 3-6 ms: A runs again. Its vruntime increases to ~0.006.

6. Time 6 ms: Timer fires. A is still the only runnable task. It continues (or is preempted and immediately re-run). No switch happens.

7. Time 50 ms: B's I/O completes. B is marked runnable. B's vruntime is still ~0 (it was blocked, not running). Scheduler preempts A. Run B.

8. Time 50 ms: B processes the response and issues another read. B sleeps again. A resumes.

**The result:**

- A runs almost continuously (only interrupted when B's I/O completes).
- B gets CPU time only when it's not blocked on I/O.
- Fairness is achieved: B does not accumulate much vruntime because it spends time blocked. A accumulates vruntime, so when B wakes, B is scheduled.

**Key insight:** CFS is *fair*, but it is fair among runnable tasks. A blocked task does not accumulate runtime, so when it wakes, it gets priority again. This is usually correct: I/O-bound tasks should be responsive.

---

## 48.9 Observing Schedulers in Production

To understand what is happening on your system, use tools:

### Linux: top, pidstat, perf

```bash
# See system-wide context switches
top -H
# Look at "context switches" per second (cs column).

# Per-process scheduler stats
pidstat -w 1
# Shows: cswch/s (voluntary context switches, e.g., from syscalls),
#        nvcswch/s (involuntary, from timer preemption).

# Detailed scheduling events
perf sched record -- ./my-program
perf sched report
```

A high rate of involuntary context switches suggests the scheduler is preempting frequently (many tasks competing). A low rate suggests tasks are yielding cooperatively or are blocked on I/O.

### macOS: Activity Monitor, fs_usage

```bash
# Detailed syscall trace
sudo dtrace -n 'syscall:::entry { @[execname] = count(); }' -c ./my-program

# Context switch events (advanced)
sudo powermetrics --samplers cpu_power -n 1
```

---

## 48.10 Tradeoffs

| Scheduler | Pros | Cons | When to use |
|---|---|---|---|
| **OS preemptive (CFS, Windows)** | Fair, responsive, no task can starve others | Context switch overhead, complex debugging (races happen at any instruction), cache pollution | General-purpose systems, servers, desktops |
| **Cooperative async (Tokio, async/await)** | Efficient (few context switches), millions of tasks, simple debugging (races at known points) | One task can starve others if it doesn't yield, requires async-aware I/O library, harder mental model | Services with many concurrent I/O connections, latency-sensitive systems |
| **M:N hybrid (Go)** | Goroutines feel like threads (simple model), lightweight, many concurrency patterns work naturally | Subtle starvation if goroutines are CPU-bound, interaction with blocking syscalls requires runtime awareness | Services with mixed I/O and compute, developers new to async |
| **Work-stealing thread pools** | Scales well, load-balances automatically, few lock bottlenecks | More complex implementation, requires tasks to be divisible, tricky for I/O | Parallel computation (embarrassingly parallel), data-parallel problems |

---

## 48.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "More threads = the OS schedules better." | Past the number of cores, more threads = more context-switch overhead and lock contention. A thread pool with N = # cores is usually optimal. Excess threads hurt. |
| "async is faster than threads." | Async is *lighter* (smaller memory, fewer context switches) and can scale to more concurrency. But if both are I/O-bound, the throughput is similar. The win is efficiency and responsiveness, not raw speed. |
| "Preemptive is always better than cooperative." | Preemptive is fairer and more responsive but more expensive. Cooperative is cheaper and simpler to debug. Choice depends on the workload. |
| "The scheduler is always fair." | Schedulers are fair *among runnable tasks*. A task that blocks on I/O is not considered runnable and doesn't accrue scheduler debt. Design depends on this. |
| "Changing niceness will make my task faster." | Niceness affects fairness, not absolute speed (unless other tasks are starving). On a quiet system, nice value does nothing. On a busy system, high priority gets more of the contended CPU. |
| "Real-time scheduling means my task always runs." | Real-time means "among real-time tasks, priority matters" and "no fair-scheduler preemption." But the kernel can still preempt you for hardware interrupts, and if you're CPU-bound, you still get timeslices. |
| "The OS can see all the work my program is doing." | No. The OS only sees blocking syscalls (or, with modern preemption points, frequent yields). Purely CPU-bound work is invisible to the scheduler until the timeslice ends. Yield points matter. |

---

## 48.12 Exercises

1. **Compare priorities.** Write two programs: one that does `while(true) { sum++; }` (CPU-bound), one that does `sleep(100ms); printf("tick\n")` repeatedly (I/O-bound). Run them together with the default scheduler. Then run the CPU-bound one with `nice +10` (lower priority). Observe with `top`: does the I/O-bound one become more responsive?

2. **Context switch cost.** Write a program that spawns N threads, each doing a tight loop. Measure wall-clock time. Increase N from 1 to 2x the number of cores and plot throughput. Where does it peak? Beyond the peak, what happens? (This demonstrates context-switch overhead.)

3. **Block vs spin.** Write a program with two approaches to waiting for 1 second:
   - Approach A: `sleep(1)` (blocking syscall).
   - Approach B: `while(clock() < target) {}` (busy-wait loop).
   
   Time both. Monitor CPU usage with `top`. Explain the difference. Which one does the scheduler prefer, and why?

4. **epoll vs threads.** Write an HTTP server that handles 1000 concurrent connections using two approaches:
   - Approach A: 1000 threads, each calling blocking `read()` on its connection.
   - Approach B: 1 event-loop thread using epoll, handling all 1000 connections.
   
   Send a slow client (sends bytes at 1 Baud per second) to each server. Measure: latency for the other 999 connections. Which server is more responsive?

5. **Go goroutine yielding.** Write a Go program with a CPU-bound goroutine that does not yield:
   ```go
   go func() {
       for i := 0; i < 1e9; i++ {
           sum += i  // No yield point here.
       }
   }()
   
   for {
       fmt.Println("main")
       time.Sleep(100 * time.Millisecond)
   }
   ```
   Run it. Does "main" print regularly? If not, when does it start? (This demonstrates that long-running CPU loops can starve other goroutines until the Go runtime's preemption point is hit.)

6. **Scheduler state inspection.** On Linux, run:
   ```bash
   pidstat -w 1 &
   your-multi-threaded-program
   ```
   Capture the `cswch/s` and `nvcswch/s` columns. Compare a program with 1 thread vs 10 threads vs 100 threads. What is the relationship between thread count and context switches?

7. **Conceptual.** Your web service is experiencing high latency on P99 requests. You suspect the scheduler. What metrics would you check? How would you know if the culprit is (a) context-switch overhead, (b) priority inversion (a low-priority task blocking a high-priority one), or (c) I/O contention?

---

## 48.13 Summary

Schedulers are the invisible hand allocating CPU time. Preemptive schedulers (Linux CFS, Windows) give fairness and responsiveness at the cost of context-switch overhead. Cooperative schedulers (async runtimes) are efficient and scale to millions of tasks but require discipline to avoid starvation. Modern systems often blend them: preemptive at the OS level, cooperative at the language level.

Understanding your scheduler means understanding what happens to tasks that block on I/O (they yield the CPU, allowing others to run), where race conditions can occur (in preemptive systems, anywhere; in cooperative systems, at yield points), and how to tune for latency (priority, nice value, cgroup CPU shares) versus throughput (fewer threads, better yields).

The concrete tools — `top`, `pidstat`, `perf sched` — are not optional. They are the language for talking about what is actually happening on your machine.

---

> **[← Previous: Concurrency vs Parallelism](02-concurrency-vs-parallelism.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Async Runtime Internals →](04-async-runtime-internals.md)**
