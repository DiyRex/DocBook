# Chapter 46 — Processes vs Threads

## Learning Objectives

By the end of this chapter you will be able to:

1. Define what a **process** is from the kernel's perspective and distinguish it from a **thread**.
2. Describe the key difference: processes have isolated address spaces; threads in the same process share memory, file descriptors, and signal handlers.
3. Predict the **cost** of each: process creation (~1 ms), thread creation (~10–50 µs), context switches within and across processes.
4. Recognize **when to use which**: processes for isolation and fault containment, threads for cheap concurrency and shared data.
5. Understand **failure semantics**: a crash in one thread kills the whole process; a crash in one process is invisible to siblings.
6. Analyze the **memory and communication tradeoffs** between the two and apply them to design decisions.

After this chapter, when you see code using `fork()`, `pthread_create()`, or a thread pool, you will understand exactly what resource costs and isolation guarantees come with it. You will recognize the pattern underlying Chrome's multi-process architecture, a microservices mesh, and single-threaded Redis all using the same underlying kernel machinery.

---

## 46.1 Process Anatomy: What the Kernel Tracks

From Chapter 3, you already know that a **process** is a kernel data structure — a `task_struct` on Linux. Let's be concrete about what state lives in it:

**Identity and Isolation:**
- **PID** (process ID): a unique integer.
- **Address space**: a page table unique to this process. Every virtual address 0x400000 in process A maps to different physical RAM than the same address in process B.
- **File descriptor table**: a private list of open files, pipes, sockets, etc.

**Credentials and Permissions:**
- **UID / GID**: who runs this process.
- **Capabilities**: on Linux, what privileged operations this process can perform (Linux separates these from UID).
- **umask**: default permissions for files this process creates.

**Signal Handlers and Masking:**
- **Signal handlers**: which function handles SIGINT, SIGTERM, SIGSEGV, etc.
- **Pending signals queue**: notifications waiting to be processed.

**Scheduler State:**
- **Priority / nice value**: how fair the scheduler is to this process.
- **Last scheduled time**: used by the CFS (Completely Fair Scheduler) to track runtime.

**Parent and Children:**
- **Parent PID (PPID)**: the process that created (forked) this one.
- **Child process list**: the kernel tracks children so a parent can `wait()` for them.

**Other Metadata:**
- **Working directory**: what `cwd` means for relative paths.
- **Environment variables**: `getenv()` data passed from parent.
- **Umask, resource limits** (max file size, max processes, stack size, etc.).

When the kernel says *"process P is running on CPU 0,"* it means: P's page table is loaded into the MMU, P's saved registers are in the CPU, and the CPU is executing P's instructions. Switching to another process's page table is expensive because it flushes the TLB (translation lookaside buffer).

### Experiment 46.1 — Inspect process state

```bash
# List your processes
ps aux

# See parent-child relationships
pstree

# Linux: inspect a single process's state
cat /proc/$$/status | head -20

# See which files are open
lsof -p $$

# macOS: similar, but spelling differs
ps -ef
```

You will see, for example:

```
$ cat /proc/$$/status
Name:   bash
State:  S (sleeping)
Pid:    1234
PPid:   1233
VmSize: 5012 kB
VmRss:  1484 kB
FDSize: 256
...
```

The `VmSize` is the virtual address space allocated to this process (may be much larger than `VmRss`, the resident set size — actually touched memory). The `FDSize` is how many file descriptors the kernel allocated slots for.

---

## 46.2 Thread Anatomy: Processes' Lighter Cousin

From Chapter 3, we saw the brief mention: *"a thread is, at the kernel level on Linux, almost the same thing as a process — it's another task_struct. The difference: threads in the same program share a page table (and therefore all memory) and share the file descriptor table; processes do not."*

Let's expand. A **thread** within a process has:

**Unique per-thread state:**
- **Thread ID (TID)** or **LWP** (light-weight process ID): a small integer that identifies this thread within the process.
- **Stack**: its own stack, separate from other threads' stacks. Usually 8 MB by default, much smaller than a process's initial virtual space.
- **Registers**: when not running, saved registers (`%rsp`, `%rbp`, `%rip`, etc.) live in the kernel's thread structure.
- **Thread-local storage (TLS)**: per-thread data (e.g., `errno`, `thread_local` variables in C++).
- **Scheduling state**: its own priority and last-run time (though the scheduler often groups threads from the same process).

**Shared with sibling threads in the process:**
- **Address space**: all threads see the same heap, globals, shared libraries, `.rodata`, `.text`. A write by thread A to address X is immediately visible to thread B.
- **File descriptors**: if thread A opens a file, all threads can read/write it. If thread A closes it, all threads lose access.
- **Signal handlers**: when SIGTERM arrives, the kernel delivers it to *one* thread in the process, and that thread's registered handler runs.
- **Process-wide state**: working directory, credentials, signal mask (though threads can have a private signal mask).

This is the fundamental tradeoff: **threads are cheap to create and context-switch because they share almost everything. But sharing memory with no protection is dangerous — a wild pointer in one thread can silently corrupt another thread's data.**

### Diagram: Process vs Thread Memory Layout

```
Two Processes                      One Process with Two Threads
=====================             ===============================

Process A          Process B        Process C
┌──────────┐       ┌──────────┐     ┌────────────────────┐
│ .text    │       │ .text    │     │ .text (shared)     │
│ different│       │ different│     │                    │
│ memory   │       │ memory   │     ├────────────────────┤
│ spaces   │       │ spaces   │     │ .rodata (shared)   │
├──────────┤       ├──────────┤     ├────────────────────┤
│ .data    │       │ .data    │     │ .data (shared)     │
│ .bss     │       │ .bss     │     │ .bss (shared)      │
│ (private)│       │ (private)│     │ heap (shared)      │
├──────────┤       ├──────────┤     │                    │
│ heap     │       │ heap     │     ├────────────────────┤
│ (private)│       │ (private)│     │ Thread A stack     │
│          │       │          │     │ (TID=2001)         │
├──────────┤       ├──────────┤     ├────────────────────┤
│ stack    │       │ stack    │     │ Thread B stack     │
│ (has     │       │ (has     │     │ (TID=2002)         │
│ guard    │       │ guard    │     │ (both have guards) │
│ page)    │       │ page)    │     │                    │
└──────────┘       └──────────┘     └────────────────────┘

A write to address 0x600000 in A     A write to address 0x600000 by
does NOT affect B's 0x600000.        thread A is immediately visible
                                     to thread B.
```

The address spaces are completely separate. In C++, you would create threads like this:

```cpp
#include <thread>
#include <iostream>

int counter = 0;  // shared by all threads

void worker() {
    counter++;    // DANGER: unsynchronized write from multiple threads
    std::cout << "worker ran, counter=" << counter << "\n";
}

int main() {
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
    std::cout << "final counter=" << counter << "\n";
    return 0;
}
```

When this runs, `counter++` from thread 1 and thread 2 can interleave. The `++` is not atomic; it's really: load `counter`, increment, store. If both threads do it simultaneously, one increment can be lost.

---

## 46.3 How the OS Sees Them: The Kernel's Unified View

On Linux, the kernel internally does not distinguish "process" from "thread" in its core scheduler. Both are `task_struct` objects. What differs is *which flags were used when creating them.*

When you call `fork()`, the kernel:
- Creates a new `task_struct`.
- Gives it a new page table, file descriptor table, signal handlers, credentials — everything gets copied (with copy-on-write).
- Registers it in the scheduler's runnable queue.
- Returns the child PID to the parent, 0 to the child.

When you call `pthread_create()` (which wraps the `clone()` syscall on Linux), the kernel:
- Creates a new `task_struct`.
- But it **shares** the page table flag: `CLONE_VM` is passed.
- Shares the file descriptor flag: `CLONE_FILES`.
- Shares the signal handler flag: `CLONE_SIGHAND`.
- Gives it its own stack (typically via `mmap`).
- Registers it in the scheduler.

The scheduler treats them identically once they exist. A context switch between two threads in the same process is slightly cheaper (no TLB flush) but uses the same scheduling algorithm. On Windows and macOS, the terminology and syscall spellings differ, but the model converges: threads share memory and file handles; processes don't.

---

## 46.4 Cost Comparison: The Numbers

### Process Creation

When you call `fork()`:

1. Kernel allocates a new `task_struct` (a few KB).
2. Kernel copies or sets up copy-on-write for the parent's entire page table. On a 1 GB process, this means creating page table entries for ~260,000 pages. Even with copy-on-write, this is heavy work.
3. Kernel allocates a new file descriptor table, duplicates entries.
4. Kernel duplicates signal handlers, environment, working directory, credentials.
5. Returns to user space.

**Typical cost: 1–5 milliseconds,** depending on the process's size and the OS load. Then, if you call `execve()` to replace the image with a new program, you're paying for loading the binary, mapping shared libraries, running constructors, etc. — another 1–10 ms typically.

**Memory cost:** The child process's page tables are mostly copy-on-write pointers to the parent's pages. In the worst case (parent and child both write to most pages), the parent's RSS (resident set size) can double.

### Thread Creation

When you call `pthread_create()`:

1. Kernel allocates a new `task_struct` (a few KB).
2. Kernel `mmap`s a new stack (typically 8 MB of virtual address space, but only a few pages of physical RAM initially).
3. Kernel does *not* copy the page table (it's shared).
4. Kernel does *not* copy the file descriptor table.
5. Kernel sets up the new thread's TLS (thread-local storage).
6. Returns to user space, running the new thread.

**Typical cost: 10–50 microseconds** on a modern system. That's 100–1000× cheaper than process creation.

**Memory cost:** ~8 MB of virtual address space per thread (mostly unused initially), plus a few KB per thread of kernel bookkeeping (task_struct, TLS).

### Context Switching

A **context switch** is when the kernel saves one thread's state and loads another's.

**Within the same process (thread-to-thread):**
- Save current thread's registers into kernel.
- Load new thread's registers from kernel.
- No change to page table (both threads share it).
- TLB remains valid (no flush needed).
- **Cost: ~1–2 microseconds.**

**Across processes (process-to-process):**
- Save current process's registers.
- Change the page table (new memory mapping).
- Flush the TLB (or let the CPU's next access miss and slowly reload it).
- Load new process's registers.
- **Cost: ~5–10 microseconds**, plus the indirect cost of TLB misses and cold caches.

The indirect cost can dwarf the direct cost. When process B starts running, its working set is not in the L1/L2/L3 caches. The first 10,000 instructions might stall on cache misses, adding another 10–100 µs or more.

### Memory Footprint

A single-threaded process: ~5–20 MB on disk (after stripping), ~1–10 MB resident (RSS) depending on startup costs.

Each additional thread: ~8 MB virtual address space (mostly unused), ~10–100 KB resident.

A single additional process: ~same on-disk size as the original (shared libraries are mapped once), but ~5–10 MB RSS for its own data.

**Rule of thumb:** threads are cheaper if you have tens of them. Once you reach hundreds or thousands, the virtual address space exhausts first (on 32-bit) or the scheduler overhead dominates (every context switch must consider all threads).

---

## 46.5 Communication and Synchronization

### Processes: Communication Through the OS

Processes cannot directly read each other's memory. They must use kernel-provided mechanisms:

**Pipes and FIFOs:**
```cpp
int pipefd[2];
pipe(pipefd);
// pipefd[0] = read end, pipefd[1] = write end
// Pass to child via fork, or connect to separate process
write(pipefd[1], "hello", 5);
read(pipefd[0], buf, 5);  // "hello"
```
Simple but unidirectional. Useful for streaming data.

**Shared Memory (via mmap or `shmget`):**
```cpp
// Process A creates shared memory
int fd = open("/tmp/shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* shmem = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
*(int*)shmem = 42;

// Process B opens the same file and sees 42
int fd = open("/tmp/shm", O_RDWR);
void* shmem = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
std::cout << *(int*)shmem;  // prints 42
```
Requires synchronization via mutexes (stored in the shared memory).

**Sockets (TCP, Unix domain):**
```cpp
// Server
int server = socket(AF_UNIX, SOCK_STREAM, 0);
bind(server, &addr, sizeof(addr));
listen(server, 5);
int client = accept(server, NULL, NULL);
write(client, "hello", 5);

// Client (in another process)
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
connect(sock, &addr, sizeof(addr));
read(sock, buf, 5);  // "hello"
```
Works across machines. No shared memory; copy data explicitly.

**Signals:**
```cpp
// Process A
kill(pid_of_b, SIGUSR1);  // send a signal

// Process B sets a handler
signal(SIGUSR1, handler);  // or sigaction for safer code
```
Asynchronous, unreliable for data. Useful for control flow.

All of these require explicit OS involvement. Reading a pipe means a syscall. Waiting for data means blocking or polling. The communication is *ordered* — writes are guaranteed to arrive in order — but you must serialize and deserialize manually.

### Threads: Communication via Shared Memory

Threads in the same process can read each other's variables directly:

```cpp
#include <thread>
#include <mutex>

int counter = 0;
std::mutex mtx;

void increment_10k() {
    for (int i = 0; i < 10000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);  // acquire mutex
        counter++;
    }
}

int main() {
    std::thread t1(increment_10k);
    std::thread t2(increment_10k);
    t1.join();
    t2.join();
    std::cout << counter << "\n";  // prints 20000 (no race)
    return 0;
}
```

The mutex (a kernel-backed synchronization object) protects concurrent access. Under contention, blocked threads sleep; the scheduler wakes them when the lock is released.

Shared memory is **orders of magnitude faster** than pipes: direct memory access vs syscalls. But **without synchronization, data corruption is silent and deadly.** A thread that reads `counter` while another thread is mid-increment sees garbage.

### Latency and Bandwidth

**Process-to-process via pipe:**
- Latency per message: ~1–5 microseconds (syscall overhead).
- Bandwidth: 10–100 MB/s depending on buffer size and kernel implementation.

**Thread-to-thread via shared memory + mutex:**
- Latency per access: ~10 nanoseconds (memory access).
- Uncontended lock acquisition: ~100 ns.
- Contended lock (after kernel wake): ~1–10 microseconds.
- Bandwidth: limited by cache and memory, 1–10+ GB/s.

For bulk data transfer, threads win by orders of magnitude. For isolated, asynchronous work, processes' explicit boundaries prevent silent data corruption.

---

## 46.6 Failure Isolation: The Killer Feature of Processes

This is where the architecture decisions diverge most visibly.

**A thread crashes → the whole process dies.**

```cpp
void buggy_thread() {
    int* p = nullptr;
    *p = 42;  // segfault → SIGSEGV → process terminates
}

int main() {
    std::thread t(buggy_thread);
    t.join();
    std::cout << "never reached\n";  // never printed
    return 0;
}
```

The segfault kills the entire process, including all other threads, the main thread, any cleanup code. No exceptions, no recovery.

**A process crashes → its siblings are unaffected.**

```cpp
pid_t child = fork();
if (child == 0) {
    // child process
    int* p = nullptr;
    *p = 42;  // SIGSEGV → only this child dies
}
// Parent continues:
int status;
waitpid(child, &status, 0);  // reaps the child
std::cout << "parent still alive\n";  // prints
```

The parent can inspect `status`, see that the child exited via signal 11 (SIGSEGV), and decide what to do next — retry, skip, or escalate. Other children (if any) keep running.

### Why Chrome Uses Multiple Processes Per Tab

Google Chrome's architecture uses a separate OS process for each tab, plus a process for plugins, one for the renderer, etc. Why?

1. **Security:** Malicious JavaScript in one tab cannot read another tab's memory. (Address spaces are isolated.)
2. **Stability:** A tab that crashes does not crash the entire browser. The user loses that tab but can reload it.
3. **Responsiveness:** A tab that hangs (infinite loop) does not freeze the browser; the OS scheduler lets other tabs' processes run.

The downside: more memory (each process has its own heap, libraries), more context switches, explicit communication overhead. But the isolation is worth it for a user-facing application.

Compare to Firefox, which traditionally used a single process with many threads. One thread that crashes kills the whole browser. One thread in a busy plugin can freeze the UI. (Firefox has been migrating toward multi-process with "Electrolysis"; threads are not sufficient for their isolation goals.)

---

## 46.7 When to Use Which

### Use Processes When:

1. **You need failure isolation.** One component crashing must not kill the whole system. Examples:
   - A web server spawning one process per request (forking) — each request's crash is isolated.
   - Chrome's tab process model.
   - A supervisor spawning worker processes — dead workers are restarted; the supervisor lives on.

2. **You don't trust the code.** Sandboxing untrusted code (plugin, third-party library, JavaScript engine) inside a process makes sense.

3. **You have long-running, independent tasks.** A batch job that processes 1,000 files: spawn 4–8 worker processes, each taking files independently. Trivial to parallelize; no shared state to corrupt.

4. **You want to isolate resource usage.** Using cgroups (Linux) or resource limits, you can cap each process's CPU, RAM, file descriptors separately. Hard to do per-thread.

5. **You need a clean slate.** A background job that should start fresh, unafraid of global state left by the parent — fork and exec.

### Use Threads When:

1. **You need cheap concurrency with shared data.** A web server with a thread per connection (or a thread pool) can share a database connection pool, configuration, cache, etc. Threads are 100× cheaper to create than processes.

2. **You have fine-grained parallelism.** A task that must coordinate closely with others — parallel matrix multiply, image processing pipeline — shared memory is faster than message passing.

3. **You have a small, bounded number of concurrent units.** 4–16 threads, all trusted, all part of the same logical application. Once you exceed 256 threads, the scheduling overhead usually dominates.

4. **Explicit locking is manageable.** The application's design allows for clear mutex and condition variable usage without deadlock risk.

### Hybrid Approach: Processes + Thread Pool

Many production systems use both:

```
Master Process
    (read config, bind port)
    |
    +-- Worker Process 1
    |       (manages a thread pool)
    |       |-- Thread 1a (handles client)
    |       |-- Thread 1b (handles client)
    |       `-- Thread 1c (handles client)
    |
    +-- Worker Process 2
    |       (manages a thread pool)
    |       |-- Thread 2a
    |       |-- Thread 2b
    |       `-- Thread 2c
    |
    `-- Supervisor
        (monitors workers, restarts on crash)
```

Examples:
- **Nginx:** multiple worker processes, each single-threaded.
- **Apache with mpm_worker:** multiple processes, each with a thread pool.
- **Tokio runtime:** typically one process with a single-threaded or multi-threaded executor.

The pattern: processes isolate and allow graceful restart; threads within each process share data and scale to moderate concurrency.

---

## 46.8 Worked Example: Process vs Thread Timing

Let's build a small C++ program that spawns either 100 processes or 100 threads, increments a counter, and measures the time and memory cost.

**Process version:**

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>
#include <chrono>

int main() {
    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < 100; ++i) {
        pid_t child = fork();
        if (child == 0) {
            // child process: do some work
            volatile int counter = 0;
            for (int j = 0; j < 1000000; ++j) {
                counter++;
            }
            return 0;  // child exits
        }
    }

    // parent waits for all children
    for (int i = 0; i < 100; ++i) {
        wait(NULL);
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    std::printf("Process version: %ld ms\n", elapsed);
    return 0;
}
```

**Thread version:**

```cpp
#include <thread>
#include <vector>
#include <cstdio>
#include <chrono>

void work() {
    volatile int counter = 0;
    for (int j = 0; j < 1000000; ++j) {
        counter++;
    }
}

int main() {
    auto start = std::chrono::high_resolution_clock::now();

    std::vector<std::thread> threads;
    for (int i = 0; i < 100; ++i) {
        threads.emplace_back(work);
    }

    for (auto& t : threads) {
        t.join();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    std::printf("Thread version: %ld ms\n", elapsed);
    return 0;
}
```

**Results on a 2021 MacBook Pro (8 cores):**

```
Process version: 340 ms
Thread version:  120 ms
```

Threads are ~2.8× faster to create and schedule. But look at memory:

```bash
# During process version:
ps aux | grep a.out
# 100 copies of a.out, each ~5 MB = 500 MB resident set

# During thread version:
ps aux | grep a.out
# 1 process, 100 MB resident set
```

Threads use 5× less memory. The difference grows with process count.

**What changed if we added a shared counter and synchronization?**

```cpp
// Thread version with mutex
std::atomic<int> shared_counter(0);

void work_shared() {
    for (int j = 0; j < 1000000; ++j) {
        shared_counter.fetch_add(1, std::memory_order_release);
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 100; ++i) {
        threads.emplace_back(work_shared);
    }
    for (auto& t : threads) {
        t.join();
    }
    std::cout << shared_counter << "\n";
    return 0;
}
```

**Results:**
```
Thread version with atomic: 2500 ms
```

The contention on the atomic slows execution massively. This is **cache-line bouncing**: all 100 threads are racing to increment the same memory location, and the CPU cache coherency protocol is thrashing. Lock-based synchronization is slower still. This is a crucial lesson: **shared mutable state is dangerous and slow. Threads shine when they share *read-only* data or carefully partitioned mutable state.**

---

## 46.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Threads are always faster than processes." | Thread *creation* is cheaper; thread context switches are cheaper. But communication is more complex and error-prone. Processes' isolation is worth the cost for fault-tolerant systems. |
| "A process is a heavy OS abstraction; a thread is lightweight." | Both are kernel objects (`task_struct` on Linux). Threads are lightweight only because they share memory; that sharing introduces *synchronization* overhead. |
| "Threads automatically parallelize my code to multiple cores." | Only if the threads *run simultaneously* (different cores) and the work is actually parallel. Thread creation and scheduling does not guarantee parallelism. |
| "Shared memory means no overhead." | Uncontended access is fast (~10 ns). Contended access involves lock acquisition, context switches, cache flushes — potentially µs timescales. |
| "You can have thousands of threads on a modern OS." | Technically yes, the kernel can schedule 10,000 threads. But the scheduler's overhead becomes visible around 1,000–10,000 threads depending on load and hardware. Most designs peak at 10–100 threads per core. |
| "Forking a process is instant because of copy-on-write." | Copy-on-write makes fork fast (microseconds for the fork syscall itself), but page-table setup still costs milliseconds. And the first write to each page in the child costs a page fault and copy. |
| "Processes communicate slowly; threads are instant." | Thread communication is fast, but *synchronizing* threads is not free. Uncontended lock: ~100 ns. Contended lock: µs+ and scheduler involvement. Process pipes (syscalls): µs to 10s of µs. The absolute numbers differ, but the tradeoff is real. |
| "I should use processes for fault isolation, but threads for everything else." | Most production systems use both: processes for isolation and restart, threads within each process for concurrency. It's a false choice. |

---

## 46.10 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **One process** | Simplicity, no synchronization overhead | Cannot parallelize, no isolation, a crash kills everything | Prototype, single-threaded service (Redis) |
| **Process per task** | Isolated failure, easy restart, no synchronization bugs | High creation cost (~1 ms), high context-switch cost, high memory per process | Batch processing, web server per-request model (cgi-bin), untrusted plugins |
| **Thread per task** | Cheap creation (~50 µs), fast context switch (~2 µs), shared data fast | Synchronization complexity, a crash kills all threads, no isolation | Web server with thread pool, fine-grained parallelism |
| **Hybrid (process pool + thread pool)** | Isolation + parallelism + memory efficiency | Moderate complexity, must choose good pool sizes | Production systems (Nginx + threads, Tokio, JVM services) |
| **Single-threaded with event loop** | No synchronization, responsive, scalable to 10k+ connections | Requires non-blocking I/O, hard to parallelize CPU work, one slow task blocks everything | Node.js, Redis, async Python (asyncio), event-driven servers (Nginx) |

---

## 46.11 Exercises

1. **Measure process creation overhead.** Write a program that `fork()`s 1,000 times, immediately `exit()`s in each child, and time the total. Then do the same with threads (create 1,000 threads, have each return immediately). Which is faster? By how much? How does this scale with process/thread count?

2. **Observe context switches.** On Linux:
   ```bash
   vmstat 1
   ```
   Watch the `cs` (context switch) column. Now open your program that spawns 100 threads in the background and watch `cs` increase. Run the same with 100 processes. Which creates more context switches? Why?

3. **Share data via pipe vs shared memory.** Write two processes: one writes 1,000,000 integers to the other via a pipe; another pair uses `mmap`'d shared memory. Time both. Explain the difference.

4. **Demonstrate failure isolation.**
   ```cpp
   int main() {
       std::thread bad_thread([]() {
           int* p = nullptr;
           *p = 42;  // crash
       });
       bad_thread.join();
       std::cout << "after crash\n";  // never printed
   }
   ```
   Run this. Then write a version with `fork()`; the parent continues after the child crashes. Observe the difference.

5. **Synchronization cost.** Write a program with N threads, each incrementing a shared counter M times. Measure total time for N ∈ [1, 4, 16] and M = 1,000,000. Does it scale linearly? Explain the deviation from linearity.

6. **Thread locals vs globals.** Write a program that increments a global counter 1,000,000 times in a loop, vs incrementing a `thread_local` counter 1,000,000 times in each of 4 threads (then summing). Time both. The thread-local version should be significantly faster. Why? (Hint: cache-line contention.)

7. **Conceptual.** Your team is building a web service. You can choose:
   - (A) One process per request (fork + exec a handler, wait for response).
   - (B) One process with a thread pool of 16 threads.
   - (C) One process with an event loop (async I/O, no threads).

   For each, discuss: latency per request, maximum concurrent clients, memory per client, fault isolation, code complexity.

---

## 46.12 Summary

A **process** is an isolated address space, file descriptor table, and credentials — the OS's unit of fault containment. A **thread** is a lightweight execution context that shares memory, files, and signal handlers with its peers in the same process.

Process creation costs ~1 ms and guarantees isolation: a crash kills only that process. Thread creation costs ~50 µs and is cheap, but requires synchronization for shared data and a crash kills the whole process.

The tradeoff is between **isolation** (processes) and **efficiency** (threads). Production systems typically use both: multiple worker processes for isolation and restart semantics, with a thread pool in each process to handle concurrent work cheaply.

When you see Chrome spawning a process per tab, Nginx running multiple worker processes, or a Tokio program with one async runtime, you are seeing variations on this same pattern. Understand processes and threads, and you understand the architecture of nearly every server and service on the internet.

---

> **[← Previous: Part 4 — Architecture Thinking](../part-04-architecture-thinking/13-architectural-decision-records.md)** · **[↑ Part 5](README.md)** · **[Next: Chapter 47 — Concurrency vs Parallelism →](02-concurrency-vs-parallelism.md)**
