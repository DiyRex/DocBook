# Chapter 52 — Event Loops

## Opening

Every async runtime — Node.js and libuv, Python's asyncio, Rust's Tokio, Go's scheduler, Boost.Asio — looks wildly different on the surface. Yet at the core, they all have the same skeleton: a loop that repeatedly polls a queue of ready work, blocks on I/O, and wakes tasks when data arrives. This chapter is what that skeleton looks like, how it works, and why it exists.

By the end of this chapter you will understand:

1. The **structure of an event loop** — the phases, the ready queue, the I/O multiplexing syscall.
2. How the loop **differs from a scheduler** — the loop is cooperative; the kernel scheduler is preemptive.
3. Why **microtasks and macrotasks exist** — they are bounds on how often different kinds of work can starve each other.
4. The **I/O multiplexing landscape** — `epoll`, `kqueue`, `io_uring`, IOCP — what each does and which one your runtime probably uses.
5. The **fundamental limitation** — one loop, one thread, no CPU work. Why production systems use thread pools alongside async.
6. How to **read and debug an event loop** when you run into real-world timing problems.

---

## 52.1 The Loop, Stripped Down

Here is an event loop in pseudocode. It is what every production runtime does, sometimes with names changed:

```cpp
// The event loop in 20 lines

while (!stopped) {
    // Phase 1: Run all tasks that are ready to run
    while (!ready_queue.empty()) {
        Task task = ready_queue.pop();
        task.run();  // May suspend (e.g., at co_await)
    }
    
    // Phase 2: Determine how long to wait for I/O
    int timeout_ms = 0;
    if (!ready_queue.empty()) {
        timeout_ms = 0;  // Don't block, ready tasks exist
    } else if (has_pending_timers()) {
        timeout_ms = next_timer_deadline_ms();  // Block until next timer
    } else if (has_registered_file_descriptors()) {
        timeout_ms = -1;  // Block indefinitely; something must arrive
    } else {
        stopped = true;  // No work left
        break;
    }
    
    // Phase 3: Block on I/O until file descriptors are ready
    events = epoll_wait(epoll_fd, timeout_ms);
    
    // Phase 4: For each FD that became ready, wake its task
    for (Event ev : events) {
        Task task = fd_to_task[ev.fd];
        ready_queue.push(task);
    }
    
    // Phase 5: Check timers; wake tasks that are ready
    for (Timer t : timers) {
        if (t.deadline_passed()) {
            ready_queue.push(t.task);
        }
    }
}
```

The loop alternates between two states:

1. **Runnable state**: tasks on the ready queue are executed.
2. **Sleeping state**: the loop blocks on I/O via a syscall like `epoll_wait`, waiting for FDs to become ready or timers to expire.

This is the fundamental difference from a preemptive scheduler (Chapter 3): **the event loop never interrupts a task**. A task runs until it voluntarily suspends (at a `co_await` or by explicitly yielding). The kernel scheduler, by contrast, interrupts any running task after a time slice (typically a few milliseconds).

### The Trade-Off

**Single-threaded event loop:**
- Pros: no context switch overhead; all tasks see the same thread-local state; simple mental model.
- Cons: one CPU core; CPU-bound work blocks all other tasks.

**Kernel preemption (normal threads):**
- Pros: automatic parallelism on multi-core; CPU-bound work runs in parallel.
- Cons: context switch overhead (~1–10 microseconds); cache thrashing; complex synchronization.

The event loop wins for thousands of I/O tasks on a single core. It loses for CPU-bound work or leveraging multiple cores. This is why production systems combine both: an event loop for I/O, and thread pools for CPU work.

---

## 52.2 Loop Phases: Node.js / libuv Example

The most explicit phase-based loop is libuv, which powers Node.js. It has six distinct phases:

```
┌─────────────────────────────────────────────┐
│          Event Loop Phases (libuv)          │
├─────────────────────────────────────────────┤
│ 1. Timers                                   │
│    Execute callbacks from setInterval,      │
│    setTimeout, etc., if deadline passed.    │
├─────────────────────────────────────────────┤
│ 2. Pending Callbacks                        │
│    Execute deferred I/O callbacks           │
│    (e.g., from previous syscall).           │
├─────────────────────────────────────────────┤
│ 3. Idle / Prepare                           │
│    Internal libuv maintenance.              │
├─────────────────────────────────────────────┤
│ 4. Poll                                     │
│    BLOCK on I/O via epoll_wait.             │
│    Wake when FDs are ready.                 │
├─────────────────────────────────────────────┤
│ 5. Check                                    │
│    Execute setImmediate callbacks.          │
│    (High-priority work post-I/O.)           │
├─────────────────────────────────────────────┤
│ 6. Close                                    │
│    Execute close callbacks.                 │
│    (Resource cleanup.)                      │
└─────────────────────────────────────────────┘
```

**Why phases?**

Without phases, the loop would be a free-for-all: whoever calls the loop next runs first. This leads to **starvation**. For example:

- If timers always run first, they might dominate the entire loop, starving I/O handlers.
- If the poll phase is too long, timers miss their deadline.

Phases bound how often each kind of work can run. In the libuv order above:

1. Timers run once per loop (even if many are due).
2. I/O runs once per loop (the poll phase blocks; pending I/O handlers run once).
3. Immediate callbacks (`setImmediate`) run once per loop.

This prevents any single category from starving the others.

### JavaScript Example: Visible Phases

JavaScript exposes this structure:

```javascript
// Runs in Phase 1 (timers)
setTimeout(() => console.log("T1"), 0);

// Runs in Phase 2/4 (pending callbacks / poll)
fs.readFile("file.txt", () => console.log("I/O"));

// Runs in Phase 5 (check)
setImmediate(() => console.log("Immediate"));

// Runs NOW (synchronous)
console.log("Sync");
```

Output (always):

```
Sync
Immediate
T1
I/O
```

The sync code runs first (not in any phase). Then the loop starts:

- Phase 1 (timers): `setTimeout` callback has not run yet? Skip this phase (deadline is 0ms, but we haven't checked the clock). Actually, in practice, it runs at the *end* of this iteration or the next, depending on timing. This is why `setTimeout(..., 0)` is *not* the same as running immediately — there's a phase boundary.
- Phase 5 (check): `setImmediate` *always* runs before timers on the next iteration (because it's Phase 5, and Phase 1 timers haven't arrived yet). **So `setImmediate` beats `setTimeout(..., 0)` every time.**
- Phase 4 (poll): I/O is pending. The loop blocks until the file is ready, reads it, and calls the callback.

This ordering is part of the language spec, not an accident. Understanding it is crucial for writing performant async code.

---

## 52.3 Microtasks vs Macrotasks

JavaScript has a hidden second loop inside each phase. Before proceeding to the next phase, all **microtasks** are drained.

**Microtasks** (run after every callback):
- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `process.nextTick()` (Node.js)
- `queueMicrotask()`
- MutationObserver callbacks

**Macrotasks** (one per phase iteration):
- `setTimeout`, `setInterval`, `setImmediate`
- `requestAnimationFrame` (browser)
- I/O callbacks (`fs.readFile`, network, etc.)

The structure is:

```
while (!stopped) {
    // Execute ONE macrotask (e.g., a timer callback)
    macrotask = macrotask_queue.pop();
    macrotask.run();
    
    // Drain ALL microtasks before continuing
    while (!microtask_queue.empty()) {
        microtask = microtask_queue.pop();
        microtask.run();
    }
}
```

### Why This Matters: Starvation

Microtasks can starve macrotasks:

```javascript
async function loop_forever() {
    while (true) {
        await Promise.resolve();  // Queues a microtask
    }
}

setTimeout(() => console.log("Never prints"), 0);

loop_forever();  // Starts
```

The microtask queue fills with infinite `Promise.resolve()` continuations. Each macrotask iteration consumes one timer callback and then drains *all* microtasks. But there are always more microtasks in the queue. The timer callback never runs.

**Output:** Nothing. The program runs forever.

This is a real problem. It's why JavaScript linters warn: "don't await in a loop without a macrotask boundary." You must do:

```javascript
async function fixed_loop() {
    while (true) {
        await new Promise(resolve => setTimeout(resolve, 0));  // Macrotask!
        // ... work ...
    }
}
```

Now `setTimeout` yields to the macrotask queue, allowing other timers to run.

### Rust / Tokio: A Different Model

Tokio doesn't have a microtask distinction. Instead, it has:

```rust
async fn work() {
    // Coroutine A runs here
    some_io().await;  // Suspension point
    // Coroutine B might run here
}
```

Tokio's scheduler switches at each `.await`, not at phase boundaries. In multi-threaded Tokio, work is load-balanced across threads. This is simpler but more expensive: you pay for thread synchronization instead of careful phase design.

---

## 52.4 I/O Multiplexing Underneath

The `epoll_wait` (or `kqueue`, `io_uring`, IOCP) syscall is the key syscall in every event loop. It is what allows one OS thread to wait on hundreds of file descriptors simultaneously.

### Linux: epoll

```cpp
// Create an epoll instance
int epfd = epoll_create1(0);

// Register interest in FD 3: "wake me when it's readable"
struct epoll_event ev = {
    .events = EPOLLIN,
    .data.fd = 3
};
epoll_ctl(epfd, EPOLL_CTL_ADD, 3, &ev);

// Block until at least one FD is ready or timeout expires
struct epoll_event events[128];
int num_ready = epoll_wait(epfd, events, 128, timeout_ms);

for (int i = 0; i < num_ready; ++i) {
    int fd = events[i].data.fd;
    // fd 3 is now readable; issue the read
    read(fd, buf, size);
}
```

**How it works inside the kernel:**

When you call `epoll_wait`, the kernel does *not* iterate through all 128 FDs checking if they're ready. Instead:

1. It has an internal data structure (a red-black tree or hash table) mapping FD -> watcher.
2. When a network packet arrives for FD 3 (an interrupt from the NIC), the kernel adds FD 3 to a ready list *inside the kernel*.
3. When `epoll_wait` is called, it returns that ready list immediately.
4. The next `epoll_wait` returns only the FDs that became ready since the last call.

This is O(1) per ready FD, not O(n). The kernel is doing the hard work; your loop just reaps the results.

### macOS / BSD: kqueue

```cpp
int kq = kqueue();

// Register: "wake when fd 3 becomes readable"
struct kevent ev;
EV_SET(&ev, 3, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &ev, 1, NULL, 0, NULL);

// Block until events
struct kevent events[128];
int num_events = kevent(kq, NULL, 0, events, 128, &timeout);

for (int i = 0; i < num_events; ++i) {
    int fd = events[i].ident;
    // fd is now ready
}
```

`kqueue` is more general than `epoll`. You can register interest in:
- File I/O (EVFILT_READ, EVFILT_WRITE)
- Timers (EVFILT_TIMER)
- Signals (EVFILT_SIGNAL)
- Processes (EVFILT_PROC)
- Filesystem events (EVFILT_VNODE)

This means a BSD/macOS event loop can use `kqueue` for everything — timers, I/O, signals. Linux requires separate mechanisms: `epoll` for I/O, `timerfd` for timers, `signalfd` for signals.

### Linux (Modern): io_uring

`io_uring` is faster than `epoll` for very high concurrency (thousands of operations per loop):

```cpp
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// Queue a read (does not execute immediately)
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, 4096, 0);
io_uring_sqe_set_data(sqe, (void*)task_id);

// Submit all pending reads to the kernel
io_uring_submit(&ring, 1);

// Wait for completions
struct io_uring_cqe *cqe;
unsigned head;
io_uring_for_each_cqe(&ring, head, cqe) {
    int task_id = (intptr_t)io_uring_cqe_get_data(cqe);
    int result = cqe->res;  // bytes read or error
    // Wake task_id
}
```

`io_uring` uses a **shared ring buffer** (mmap'd into both kernel and userspace). Instead of syscalls, you:

1. Write I/O requests directly into the submission queue (in userspace, no syscall).
2. Call `io_uring_submit` once to tell the kernel "process all queued requests."
3. Wait on the completion queue, which the kernel updates asynchronously.

This cuts syscall overhead drastically. Modern Linux runtimes (like Tokio with the experimental `uring` feature) are moving here.

### Windows: I/O Completion Ports (IOCP)

Windows doesn't have `epoll`. Instead:

```cpp
HANDLE iocp = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

// Associate a file with the completion port
CreateIoCompletionPort(file_handle, iocp, (ULONG_PTR)task_id, 0);

// Issue an async read
OVERLAPPED ov = {};
ReadFile(file_handle, buf, 4096, NULL, &ov);

// Wait for completion
OVERLAPPED_ENTRY entries[128];
ULONG num_entries;
GetQueuedCompletionStatusEx(iocp, entries, 128, &num_entries, timeout_ms, FALSE);

for (ULONG i = 0; i < num_entries; ++i) {
    ULONG_PTR task_id = entries[i].lpCompletionKey;
    OVERLAPPED *ov = entries[i].lpOverlapped;
    // I/O for task_id completed
}
```

IOCP is fundamentally different: instead of reporting readiness (epoll), it reports completion. You issue a read, the kernel does the read asynchronously, and you get notified when it's done with the result code and bytes transferred. This is closer to Libuv and Tokio's model.

---

## 52.5 Single-Loop Limitations

An event loop is elegant for I/O but breaks on CPU work.

### Problem 1: CPU Blocking

```javascript
async function cpu_work() {
    // No await here; this is synchronous CPU work
    let sum = 0;
    for (let i = 0; i < 1e9; ++i) sum += i;
    return sum;
}

async function io_work() {
    await fetch("https://example.com");
}

// Both scheduled on same loop
Promise.all([cpu_work(), io_work()]);
```

The loop starts `cpu_work()`. It runs synchronously for ~2 seconds (on a modern CPU). During those 2 seconds, `io_work()` cannot make progress — the loop is blocked.

**Output:** I/O takes ~2 seconds longer than it should.

### Problem 2: Stale Cache

Even worse, the loop blocks the entire process. Timer callbacks miss their deadline, signal handlers cannot run, and the UI (in a browser) freezes.

### The Solution: Thread Pools

Node.js uses a thread pool (via libuv) for blocking operations:

```javascript
// Uses a worker thread, not the main loop
const data = await fs.promises.readFile("huge-file.bin");

// Uses a worker thread for crypto, not the main loop
const hash = await crypto.subtle.digest("SHA-256", data);
```

Tokio offers `tokio::task::spawn_blocking`:

```rust
tokio::spawn(async {
    // Run expensive_computation on a worker thread, not this loop
    let result = tokio::task::spawn_blocking(|| expensive_computation())
        .await
        .unwrap();
});
```

C++ with Boost.Asio can dispatch to a `thread_pool`:

```cpp
boost::asio::thread_pool pool(4);  // 4 worker threads

boost::asio::post(pool, [] {
    expensive_computation();  // Runs on a worker thread
});
```

**Rule:** Never do CPU work on the event loop. Always dispatch to a thread pool.

---

## 52.6 Worked Example: A Toy Event Loop in C++

Here is a minimal single-threaded event loop using `epoll`, with a TCP server that echoes bytes:

```cpp
#include <iostream>
#include <unordered_map>
#include <queue>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <arpa/inet.h>

// Task: a connection waiting for I/O
struct Task {
    int fd;
    char buffer[4096];
    size_t bytes_buffered;
};

int main() {
    // Create epoll instance
    int epfd = epoll_create1(0);
    std::unordered_map<int, Task*> tasks;
    std::queue<Task*> ready_queue;
    
    // Create listening socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
    
    // Bind to port 8000
    struct sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(8000);
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 128);
    
    // Make non-blocking
    fcntl(listen_fd, F_SETFL, O_NONBLOCK);
    
    // Register listener with epoll
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
    
    std::cout << "Listening on port 8000...\n";
    
    // Event loop
    struct epoll_event events[128];
    while (true) {
        // Phase 1: Process ready queue
        while (!ready_queue.empty()) {
            Task* task = ready_queue.front();
            ready_queue.pop();
            
            if (task->fd == listen_fd) {
                // Accept new connection
                struct sockaddr_in peer = {};
                socklen_t peer_len = sizeof(peer);
                int client_fd = accept(listen_fd, (struct sockaddr*)&peer, &peer_len);
                if (client_fd >= 0) {
                    fcntl(client_fd, F_SETFL, O_NONBLOCK);
                    Task* client = new Task{client_fd, {}, 0};
                    tasks[client_fd] = client;
                    
                    struct epoll_event ev2;
                    ev2.events = EPOLLIN;
                    ev2.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev2);
                    
                    std::cout << "Client " << client_fd << " connected\n";
                }
            } else {
                // Read from client
                Task* task = tasks[task->fd];
                ssize_t n = read(task->fd, task->buffer + task->bytes_buffered,
                                 sizeof(task->buffer) - task->bytes_buffered);
                if (n > 0) {
                    task->bytes_buffered += n;
                    
                    // Echo back
                    ssize_t written = write(task->fd, task->buffer, task->bytes_buffered);
                    if (written > 0) {
                        task->bytes_buffered -= written;
                        if (task->bytes_buffered > 0) {
                            // Incomplete write; shift buffer
                            memmove(task->buffer, task->buffer + written, task->bytes_buffered);
                        }
                    }
                } else if (n < 0 && (errno != EAGAIN && errno != EWOULDBLOCK)) {
                    // Error or close
                    std::cout << "Client " << task->fd << " disconnected\n";
                    epoll_ctl(epfd, EPOLL_CTL_DEL, task->fd, NULL);
                    close(task->fd);
                    delete task;
                    tasks.erase(task->fd);
                }
            }
        }
        
        // Phase 2: Block on I/O
        int num_events = epoll_wait(epfd, events, 128, 1000);  // 1s timeout
        if (num_events < 0) break;  // Error
        
        // Phase 3: Re-queue ready tasks
        for (int i = 0; i < num_events; ++i) {
            int fd = events[i].data.fd;
            if (tasks.count(fd)) {
                ready_queue.push(tasks[fd]);
            } else {
                // Listener
                ready_queue.push(&(Task{listen_fd, {}, 0}));
            }
        }
    }
    
    close(listen_fd);
    close(epfd);
    return 0;
}
```

**To run:**

```bash
g++ -o echo_server echo_server.cpp
./echo_server &

# In another terminal
nc localhost 8000
# Type something; it echoes back
```

**The loop's phases:**

1. **Ready queue**: check if any tasks from the previous `epoll_wait` are still runnable.
2. **epoll_wait**: block until a socket is readable or a timeout expires.
3. **Re-queue**: for each ready socket, add its task back to the queue.

**Why it works:**

- One thread, one event loop, no synchronization.
- `epoll_wait` blocks the OS thread; the CPU does nothing while waiting.
- When a client sends data, the NIC interrupts; the kernel marks the FD ready; `epoll_wait` returns; the task runs.
- No CPU-bound work, so no starvation.

**Limitations:**

- All I/O is blocking at the high level (you must write to the buffer and read from it, which can block on large packets).
- No timeout management (timers are implemented via the `epoll_wait` timeout, which is crude).
- No multi-threading; one core only.

---

## 52.7 Common Patterns

### Run Until Complete

```cpp
Future<int> my_future = some_async_work();
int result = event_loop.run_until_complete(my_future);
```

The loop runs until `my_future` is resolved. This is how test harnesses and CLIs use async:

```python
# Python asyncio
import asyncio

async def main():
    result = await fetch_and_process()
    return result

result = asyncio.run(main())  # Blocks until main() completes
```

### Gather (Fan-out)

```cpp
std::vector<Future<int>> futures = {
    async_task_1(),
    async_task_2(),
    async_task_3()
};

auto [r1, r2, r3] = event_loop.gather(futures);  // Wait for all
```

The loop runs until all futures are resolved. Under the hood, they may execute concurrently on the loop (interleaved), or on separate threads if the runtime is multi-threaded.

### Select (First to Complete)

```cpp
int result = event_loop.select({
    timeout_in(1.0),      // First to complete wins
    read_from(socket),
    timer(5.0)
});
```

The loop runs all tasks until one completes. This is useful for timeouts and competitive async operations.

### Backpressure

```cpp
bounded_queue<int> q(capacity=100);

async void producer() {
    for (int i = 0; i < 1000000; ++i) {
        co_await q.send(i);  // Blocks if queue is full
    }
}

async void consumer() {
    while (auto v = co_await q.recv()) {
        co_await slow_operation(v);  // Slow
    }
}
```

Without backpressure (unbounded queue), the producer would enqueue millions of items before the consumer even starts, consuming all RAM. With a bounded queue, the producer waits.

### Idle Handlers

```cpp
event_loop.set_idle_handler([] {
    // Runs when no other work is pending
    housekeeping();
    garbage_collection();
});
```

Some runtimes (like libuv) have an idle phase for background work that doesn't block the loop.

---

## 52.8 Single-Loop vs Multi-Loop vs Threads: Tradeoffs

| Approach | Memory | Latency | CPU Utilization | Complexity |
|---|---|---|---|---|
| **Single event loop** | 1 MB (one OS thread) | ~1 μs per I/O (no context switch) | 1 core | Simple; no locks needed |
| **Multi-loop (one per core)** | N MB (N threads, load-balanced) | ~10 μs per I/O (possible migration) | N cores | Moderate; work-stealing, load balancing |
| **Thread pool (M:N)** | M MB (M threads, thousands of tasks) | ~100 μs per I/O (context switch) | M cores | Complex; mutexes, atomics, cache coherence |
| **Hybrid: async + thread pool** | (1+M) MB | ~1 μs (I/O) + ~100 μs (CPU) | N+M effective | Complex; dispatch logic |

**When to use:**

- **Single loop**: Single machine, thousands of concurrent I/O tasks, no CPU work, low latency required. (Node.js, asyncio on single thread).
- **Multi-loop**: Server with multiple cores, mixed I/O and some CPU work, willing to accept moderate complexity. (Tokio, Go, modern Java async).
- **Thread pool**: Legacy codebases, or when CPU work dominates. (C++ with `std::thread`, Java with thread pools).
- **Hybrid**: Production systems where I/O and CPU tasks coexist. (Tokio with `spawn_blocking`, Node.js with worker threads).

---

## 52.9 Common Misconceptions

**"Event loops are always single-threaded."**

No. Modern runtimes (Tokio, Go, Erlang) run one event loop per core, load-balanced. The loop itself is single-threaded (one loop per OS thread), but multiple loops run in parallel.

**"Async means no blocking."**

Async *prevents blocking the loop*. But an individual task can still block in a blocking syscall if it's on a worker thread. The loop doesn't wait for it.

**"The event loop schedules tasks fairly."**

Not at all. If one task does CPU work (no `await`), others starve. Fair scheduling requires preemption, which the loop doesn't have. This is a feature in some contexts (cache locality) and a bug in others (responsiveness).

**"Microtasks always run before macrotasks."**

In JavaScript, yes. But this is a language-specific detail. Other runtimes don't have this distinction.

**"You can nest event loops."**

You cannot call `event_loop.run()` inside a callback on that loop. It's a deadlock. Some runtimes panic; others hang silently.

**"I/O multiplexing is always faster than threads."**

For thousands of I/O tasks, yes. For a few dozen, threads are simpler and often faster (less allocation, no state machine overhead).

---

## 52.10 Exercises

1. **Loop simulation.** Implement a single-threaded event loop in your language. Spawn two tasks that each do: "print step 1, yield, print step 2, yield, print step 3, done." Verify they interleave correctly.

2. **Backpressure.** Implement a bounded queue. Spawn a producer that enqueues 10000 items (fast) and a consumer that dequeues slowly (e.g., 1ms per item). Without backpressure, measure queue depth; with a bounded queue, observe how the producer blocks.

3. **Timer starvation.** Write a loop that spawns many I/O tasks (reading from pipes or sockets). Add a timer callback. Measure how often the timer runs (it should be starved if I/O is constant).

4. **epoll trace.** Using `strace -e epoll_wait` or `dtrace`, run a real async program (e.g., `node` with a simple HTTP server). Watch the `epoll_wait` syscalls. How often does it block? What FDs are registered?

5. **CPU work vs I/O.** Spawn 100 async tasks: 99 do I/O (reading a file), 1 does CPU work (sleep or busy loop for 1 second). Measure how long the 99 I/O tasks take. Explain why. Repeat with CPU work on a worker thread (e.g., `tokio::task::spawn_blocking`). Does latency improve?

---

## 52.11 Summary

The event loop is the skeleton of every async runtime. It is a single OS thread that repeatedly polls a queue of ready tasks, executes them until they suspend, then blocks on I/O (via `epoll`, `kqueue`, or similar) until the kernel signals that work is available.

**The loop's advantage:** minimal overhead per task. One thread can drive millions of coroutines because the thread sleeps, not the coroutines.

**The loop's disadvantage:** CPU-bound work blocks all others. This is why production systems combine async loops with thread pools.

**Key mechanisms:**

- **Phases**: Bound how often different kinds of work run (timers, I/O, cleanup).
- **Microtasks vs macrotasks**: Prevent starvation in systems (like JavaScript) that distinguish them.
- **I/O multiplexing**: The kernel-level mechanism (`epoll`, `kqueue`, `io_uring`) that makes it possible to wait on many FDs in one syscall.

Once you understand the event loop, async systems are no longer magic. You can reason about timing, predict starvation, and diagnose performance problems.

---

> **[← Previous: Cancellation Propagation](06-cancellation-propagation.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Coroutines →](08-coroutines.md)**
