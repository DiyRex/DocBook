# Chapter 49 — Async Runtime Internals

Behind every `async`/`await` keyword, even in languages far apart as C++, Rust, JavaScript, and Python, the same machinery is running. An async function is not a thread. It is not a true function, either — it is a **state machine** compiled by the language. This chapter tears that machine apart, shows you the event loop that drives it, and explains the I/O notification tricks that keep thousands of coroutines from blocking an entire program.

By the end of this chapter you will understand what is happening *underneath* when you write `co_await` or `.await` — the state transitions, the waker callbacks, the kernel integration. You will be able to read async code without treating it as magic, and you will understand the real cost and benefit of async concurrency versus threads.

---

## 49.1 What `async` Actually Compiles To

Every `async` function, regardless of language, compiles to a **state machine**. Let's see what that looks like in C++ using coroutines.

Here is a simple async function that reads data from two file descriptors:

```cpp
// C++20 coroutine (real C++)
AsyncTask<std::string> read_both_files() {
    std::string a = co_await read_file("a.txt");
    std::string b = co_await read_file("b.txt");
    co_return a + b;
}
```

The compiler does not generate a single function with two `await` points that suspends and resumes magically. Instead, it generates a **state machine object** (often called a "promise" or "coroutine state") on the heap:

```cpp
// Pseudocode: what the compiler actually generates
struct read_both_files_state {
    int state = 0;  // 0: entry, 1: after first read, 2: after second read
    
    // Local variables that outlive awaits
    std::string a;
    std::string b;
    
    // The actual result
    std::string result;
    
    bool step() {
        switch (state) {
            case 0: {
                // Execute from start to first co_await
                // Issue the async read
                start_async_read("a.txt");
                state = 1;
                return false;  // suspended, waiting for I/O
            }
            case 1: {
                // Resume here when first read completes
                a = get_read_result();
                
                // Execute from first await to second await
                start_async_read("b.txt");
                state = 2;
                return false;  // suspended again
            }
            case 2: {
                // Resume here when second read completes
                b = get_read_result();
                
                // Execute after final await
                result = a + b;
                state = 3;
                return true;  // completed
            }
        }
        return true;
    }
};
```

Three key observations:

1. **Each `co_await` becomes a suspension point** — a case in the switch where the coroutine yields control back to the event loop.

2. **Local variables that cross an await point are stored in the state object** — they live on the heap, not the stack. This is crucial: the stack frame for `read_both_files` dies as soon as the first `co_await` suspends, but `a` and `b` must persist until they're read again at the next `step()` call.

3. **The coroutine is not a function; it is a callable object.** Resuming the coroutine means calling its `step()` method again.

This is why async functions are so cheap: they do not allocate OS threads. The state machine is a small heap allocation (typically a few hundred bytes). Thousands of them can exist simultaneously without blowing the kernel's thread limit.

---

## 49.2 The Event Loop: One Thread, Many Coroutines

Given that every coroutine is a state machine, something must drive those state machines forward. That something is the **event loop**.

Here is the skeleton of a single-threaded event loop in C++:

```cpp
class EventLoop {
private:
    std::queue<Task*> ready_queue;
    std::set<int> registered_fds;  // file descriptors waiting for I/O
    
public:
    void run() {
        while (!ready_queue.empty() || !registered_fds.empty()) {
            // Step 1: Run all ready tasks
            while (!ready_queue.empty()) {
                Task* task = ready_queue.front();
                ready_queue.pop();
                
                // Call the coroutine's step() method
                bool completed = task->step();
                
                if (!completed) {
                    // Coroutine suspended; it will be re-queued when I/O is ready
                    // (or by a timer, or by another task signaling it)
                }
            }
            
            // Step 2: Wait for I/O on any registered FDs
            // This blocks until at least one FD is ready, or a timer expires
            int num_ready = epoll_wait(epoll_fd, events, max_events, timeout);
            
            // Step 3: For each ready FD, wake the task(s) waiting on it
            for (int i = 0; i < num_ready; i++) {
                int fd = events[i].data.fd;
                Task* task = fd_to_task[fd];
                ready_queue.push(task);  // Re-queue for the next loop iteration
            }
        }
    }
};
```

The event loop is a single thread that:

1. **Polls the ready queue** — runs coroutines that have been woken up (either by I/O completion, a timer, or being spawned).
2. **Blocks on I/O** — tells the kernel "wake me when any of these file descriptors are ready for reading or writing." The kernel does not busy-spin; it truly sleeps the thread until one of the FDs has data.
3. **Re-queues** — when an FD is ready, the waiting coroutine is put back on the ready queue.

This is the fundamental insight of async: **one OS thread can drive millions of coroutines, because the thread sleeps between work, not the coroutines themselves.**

### Timeline Example

Here is a timeline of what happens when two coroutines try to read from two different files:

```
Time  Thread State              Ready Queue              Registered FDs
----  -------------------      -----------              ----------------
0     Running task A           [task_A]                 (empty)
1     task_A: "read fd=3"      (empty)                  [3, 4]
        Suspends at co_await
2     epoll_wait() blocks      (empty)                  [3, 4]
       (kernel sleeping)
3     Running task B           [task_B]                 [3, 4]
4     task_B: "read fd=4"      (empty)                  [3, 4]
        Suspends at co_await
5     epoll_wait() blocks      (empty)                  [3, 4]
       (kernel sleeping)
6     [Kernel: fd=3 has data]  (empty)                  [3, 4]
       epoll_wait() returns
7     task_A re-queued         [task_A]                 [4]
8     Running task A           (empty)                  [4]
       task_A: resume, read data, complete
9     [Kernel: fd=4 has data]  (empty)                  [4]
       epoll_wait() returns
10    task_B re-queued         [task_B]                 (empty)
11    Running task B           (empty)                  (empty)
       task_B: resume, read data, complete
```

The key: steps 2 and 5 are the entire thread sleeping. No busy-loop, no polling. The OS schedules the thread out; the kernel wakes it when work arrives. This is why async is so efficient for I/O-bound work.

---

## 49.3 I/O Readiness Notification

For the event loop to sleep efficiently, the OS must tell it when an I/O operation is ready. The mechanism varies by OS.

### Linux: epoll

```cpp
int epoll_fd = epoll_create1(0);

// Register interest in fd=3 for read readiness
struct epoll_event event;
event.events = EPOLLIN;  // "readable"
event.data.fd = 3;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, 3, &event);

// ... later, in the loop ...

struct epoll_event events[128];
int num_ready = epoll_wait(epoll_fd, events, 128, timeout_ms);

for (int i = 0; i < num_ready; i++) {
    int fd = events[i].data.fd;
    // fd is now ready
}
```

`epoll_wait` is not a syscall that checks all FDs — that would be O(n). Instead, `epoll` is a kernel data structure that maintains a ready list internally. When a network packet arrives for fd=3, the kernel adds fd=3 to that ready list *without leaving the kernel*. The next `epoll_wait` returns immediately, telling us which FDs are ready.

### macOS/BSD: kqueue

```cpp
int kq = kqueue();

struct kevent ev;
EV_SET(&ev, fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &ev, 1, NULL, 0, NULL);

// ... later ...

struct kevent events[128];
int num_ready = kevent(kq, NULL, 0, events, 128, &timeout);

for (int i = 0; i < num_ready; i++) {
    int fd = events[i].ident;
    // fd is now ready
}
```

`kqueue` is conceptually similar to `epoll` but more flexible. Instead of just FD readiness, you can register interest in timers, signals, process state changes, and filesystem events.

### Windows: I/O Completion Ports (IOCP)

Windows does not have `epoll`. Instead, it has:

```cpp
HANDLE iocp = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

// Associate a file handle with the completion port
CreateIoCompletionPort(file_handle, iocp, (ULONG_PTR)task, 0);

// Issue an async read
ReadFile(file_handle, buffer, size, &bytes_read, &overlapped);

// ... later, in the loop ...

OVERLAPPED_ENTRY entries[128];
ULONG num_ready;
GetQueuedCompletionStatusEx(iocp, entries, 128, &num_ready, timeout_ms, FALSE);

for (ULONG i = 0; i < num_ready; i++) {
    Task* task = (Task*)entries[i].lpCompletionKey;
    // I/O for task is complete
}
```

IOCP is fundamentally different: instead of telling you "fd=3 is readable," it tells you "your read on handle=3 completed with N bytes." This is more semantic; the OS knows you issued a read, and it reports the result, not just the readiness state.

### Linux (newer): io_uring

Modern Linux offers `io_uring`, which is faster than `epoll` for high-concurrency workloads:

```cpp
struct io_uring ring;
io_uring_queue_init(128, &ring, 0);

// Queue a read
struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buffer, size, 0);
io_uring_sqe_set_data(sqe, (void*)task);
io_uring_submit(&ring, 1);

// ... later ...

struct io_uring_cqe* cqe;
unsigned head;
io_uring_for_each_cqe(&ring, head, cqe) {
    Task* task = (Task*)io_uring_cqe_get_data(cqe);
    int result = cqe->res;  // bytes read, or error
}
```

`io_uring` is a shared ring buffer between user space and the kernel, designed for extremely high throughput (thousands of operations per loop).

---

## 49.4 Wakers: The Contract Between Runtime and Task

In systems like Rust's Tokio, every `Future` (Rust's name for coroutine) has a **Waker** — a callback mechanism that says "when the I/O you're waiting for is ready, call me."

Here is the Rust pattern (simplified):

```rust
// Pseudocode: Rust Future trait
pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        // cx contains a Waker
        // If not ready, store the waker somewhere and return Poll::Pending
        // If ready, return Poll::Ready(output)
    }
}

// A file-read future might look like:
struct ReadFuture {
    fd: i32,
    buffer: [u8; 1024],
    waker: Option<Waker>,
}

impl Future for ReadFuture {
    type Output = usize;  // bytes read
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<usize> {
        // Try a non-blocking read
        match nix::unistd::read(self.fd, &mut self.buffer) {
            Ok(n) => Poll::Ready(n),
            Err(EAGAIN) => {
                // Not ready yet. Store the waker.
                self.waker = Some(cx.waker().clone());
                // Also register this fd with the runtime's I/O driver
                RUNTIME.register_read(self.fd, cx.waker().clone());
                Poll::Pending
            }
            Err(e) => panic!("Read error: {}", e),
        }
    }
}
```

When the I/O driver (the part of the runtime interfacing with `epoll`/`kqueue`) detects that fd=3 is readable, it:

1. Finds the `Waker` that was registered for fd=3.
2. Calls `waker.wake()`.
3. This puts the task back on the ready queue.

The next time the event loop runs, the task's `poll()` method is called again, the non-blocking read succeeds, and the coroutine resumes.

This is the contract: **the Waker is not a callback that runs the task; it is a signal that says "check again, you are probably ready now."**

---

## 49.5 Single-Threaded vs Multi-Threaded Async

### Single-Threaded

A single-threaded event loop is the simplest and often the fastest for CPU-light workloads:

```
One OS thread
    |
    +-- Event loop
    |
    +-- Coroutine A (state machine)
    +-- Coroutine B (state machine)
    +-- Coroutine C (state machine)
    ...
```

**Pros:**
- No synchronization overhead. Tasks do not need mutexes, atomics, or `Send`/`Sync` bounds. Data sharing is just a shared reference.
- Cache-friendly. All tasks run on the same core; they share the L1/L2/L3 caches.
- Simple debugging. Stack traces are linear; there are no interleaving surprises.

**Cons:**
- CPU-bound work blocks the loop. If task A is computing prime factorization, tasks B and C cannot make progress.
- Cannot use multiple cores. One event loop = one core.

### Multi-Threaded

A multi-threaded runtime spawns multiple event loops, one per CPU core:

```
Thread 1           Thread 2           Thread 3
    |                  |                  |
Event loop 1       Event loop 2       Event loop 3
    |                  |                  |
Task A, C, ...     Task B, D, ...     Task E, F, ...
(shared work queue)
```

When a task is spawned, it is assigned to one of the event loops. If a task migrates between threads (e.g., suspended on Thread 1, resumed on Thread 2), it must be `Send`.

**Pros:**
- Full CPU utilization. CPU-bound tasks can run in parallel.
- Better throughput for mixed workloads.

**Cons:**
- Synchronization overhead. Tasks that share data must use `Arc<Mutex<T>>` or atomic operations.
- Debugging is harder. Task A might suspend on thread 1 and resume on thread 2; the "stack" is not contiguous.
- Potential for work-stealing overhead. If the runtimes are not careful, moving tasks between threads causes cache misses and lock contention.

**The pragmatic choice:** Most production runtimes (Tokio, asyncio, JavaScript event loops in Node) offer both. Python's asyncio is single-threaded by default but can spawn multiple threads. Node.js is single-threaded but uses a thread pool for some operations. Tokio defaults to multi-threaded but offers a single-threaded variant.

---

## 49.6 Backpressure: A Forgotten Crisis

Consider a producer-consumer scenario where a task produces data and puts it on a channel (a queue):

```cpp
async fn producer() {
    for i in 0..1000000 {
        channel.send(i).await;  // Sending is async
    }
}

async fn consumer() {
    while let Some(value) = channel.recv().await {
        // Do something slow with value
        sleep(1_ms);
    }
}
```

If the producer is fast and the consumer is slow, the channel's internal queue will grow unbounded:

```
Time    Queue Size    Producer State         Consumer State
0       0             Sending 0              Waiting for recv
1       1             Sending 1              Processing 0
2       2             Sending 2              Processing 1
3       3             Sending 3              Processing 2
...     ...           ...                    ...
1000    1000          Sending 1000           Processing 990 (still slow!)
10000   10000         Sending 10000          Still way behind
```

The queue will consume all available memory. The producer never blocks because `channel.send()` succeeds immediately (the queue has space). The consumer cannot catch up.

**Backpressure** is the solution: bounded channels.

```cpp
Channel<int> channel(capacity=100);  // Bounded to 100 items

async fn producer() {
    for i in 0..1000000 {
        channel.send(i).await;  // Blocks if queue is full
    }
}
```

Now:

```
Time    Queue Size    Producer State                Consumer State
0       0             Sending 0                     Waiting for recv
...     ...           ...                           ...
100     100           **BLOCKED** (queue is full)   Processing 90
101     100           **BLOCKED**                   Processing 91
...     ...           ...                           ...
102     99            **BLOCKED**                   Processing 92
103     100           Sending 101                   Processing 93
```

The producer blocks when the queue is full and waits for the consumer to make room. This prevents unbounded memory growth and ensures the producer and consumer move at compatible rates.

**Rule of thumb:** Always use bounded channels in async code. Unbounded channels are a DoS vector.

---

## 49.7 A Toy Async Runtime in C++

Here is a minimal async runtime demonstrating the core concepts (roughly 100 lines):

```cpp
// Real, compilable C++20 pseudo-async library
#include <queue>
#include <functional>
#include <optional>
#include <iostream>

struct Task {
    std::function<void()> run;
    
    Task(std::function<void()> f) : run(f) {}
};

class Runtime {
private:
    std::queue<Task> ready_queue;
    
public:
    void spawn(std::function<void()> task) {
        ready_queue.push(Task(task));
    }
    
    void execute() {
        while (!ready_queue.empty()) {
            Task task = ready_queue.front();
            ready_queue.pop();
            task.run();  // Execute one step
        }
    }
};

// Simulating two async tasks that yield and resume
Runtime rt;

struct Coroutine1State {
    int step = 0;
    
    void poll() {
        switch (step) {
            case 0:
                std::cout << "Coro1: Step 0\n";
                step = 1;
                rt.spawn([this] { this->poll(); });
                break;
            case 1:
                std::cout << "Coro1: Step 1\n";
                step = 2;
                rt.spawn([this] { this->poll(); });
                break;
            case 2:
                std::cout << "Coro1: Step 2 (done)\n";
                break;
        }
    }
};

struct Coroutine2State {
    int step = 0;
    
    void poll() {
        switch (step) {
            case 0:
                std::cout << "Coro2: Step 0\n";
                step = 1;
                rt.spawn([this] { this->poll(); });
                break;
            case 1:
                std::cout << "Coro2: Step 1\n";
                step = 2;
                rt.spawn([this] { this->poll(); });
                break;
            case 2:
                std::cout << "Coro2: Step 2 (done)\n";
                break;
        }
    }
};

int main() {
    Coroutine1State coro1;
    Coroutine2State coro2;
    
    rt.spawn([&coro1] { coro1.poll(); });
    rt.spawn([&coro2] { coro2.poll(); });
    
    rt.execute();
    
    return 0;
}

// Output:
// Coro1: Step 0
// Coro2: Step 0
// Coro1: Step 1
// Coro2: Step 1
// Coro1: Step 2 (done)
// Coro2: Step 2 (done)
```

This demonstrates:

1. Each coroutine is a state machine with a `poll()` method.
2. The runtime manages a queue of ready tasks.
3. Tasks yield by re-enqueueing themselves.
4. Interleaving is fair: both coroutines make progress in turn.

A real runtime would replace the simple function calls with actual I/O registration (epoll, etc.) and would use Wakers to re-queue tasks only when they are actually ready.

---

## 49.8 Where Async Hurts

### CPU-Bound Work Blocks the Loop

The single-threaded event loop is the Achilles heel of async. A CPU-bound coroutine starves all others:

```rust
async fn cpu_bound() {
    let result = expensive_computation();  // No await here; this blocks
    return result;
}

async fn io_bound() {
    let data = read_file("data.txt").await;
    return data;
}

#[tokio::main]
async fn main() {
    tokio::spawn(cpu_bound());
    tokio::spawn(io_bound());
    // Both run on the same thread.
    // If cpu_bound takes 10 seconds, io_bound waits 10 seconds.
}
```

**Solution:** Move CPU work to a thread pool. Tokio offers `tokio::task::spawn_blocking`:

```rust
async fn main() {
    tokio::spawn(tokio::task::spawn_blocking(|| expensive_computation()));
    tokio::spawn(io_bound());
    // Now they can run in parallel.
}
```

### Forgetting to Await

If a coroutine forgets to `await`, the I/O request is never issued, and the coroutine hangs invisibly:

```rust
async fn bad_read() {
    let result = read_file("data.txt");  // Forgot .await
    println!("{:?}", result);  // Prints: Future { ... } (not the data!)
}
```

Modern compilers warn about this, but older ones did not.

### Shared Mutable State Across Awaits

Mutable state shared between coroutines is a footgun. In single-threaded async, there is no data race (no true parallelism), but there is a logic error called the "re-entrancy bug":

```rust
let mut counter = 0;

async fn increment() {
    let current = counter;
    some_io().await;  // Suspends here
    counter = current + 1;  // But someone else incremented counter while we waited!
}

// Spawn two incrementers
tokio::spawn(increment());
tokio::spawn(increment());

// Expected: counter = 2
// Actual: counter = 1 (lost update)
```

The fix is to use a `Mutex`:

```rust
let counter = Arc::new(Mutex<i32>());

async fn increment() {
    let mut guard = counter.lock().await;  // Async lock
    *guard += 1;
}
```

### The Colored Function Problem

Async functions have a different signature from sync functions. A function that calls an async function must itself be async. This "colors" the entire call stack:

```rust
fn sync_read() -> String {
    read_file("data.txt").await  // ERROR: can't await in a sync function
}

async fn async_read() -> String {
    read_file("data.txt").await  // OK
}

fn sync_caller() {
    let data = async_read();  // ERROR: can't call async fn without await
}

async fn async_caller() {
    let data = async_read().await;  // OK
}
```

Once you go async, your entire call chain is async. This is why languages like Python allow you to mix sync and async, but you must be careful to run async code on the event loop.

---

## 49.9 Tradeoffs: Threading vs Async

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Thread per task** | Simple; true parallelism; blocking I/O works. | High memory (1-2 MB per thread); context switch overhead; synchronization complexity. | Small number of tasks; CPU-bound work; legacy code. |
| **Async single-threaded** | Minimal memory; simple sharing (no locks needed); excellent for I/O. | Blocks the entire loop on CPU work; requires discipline. | Thousands of I/O tasks on a single core. |
| **Async multi-threaded** | Full CPU utilization; efficient for mixed workloads. | Synchronization overhead; task migration cost; complex debugging. | Production servers handling both I/O and CPU work. |
| **M:N runtime** (user-space threads with async underneath) | Best of both: lightweight tasks with minimal sync overhead. | Complex to implement; language support required. | Go goroutines, Erlang processes. |

---

## 49.10 Common Misconceptions

**"Async is always faster than threads."**

Not true. Async is faster for I/O-bound workloads where you have thousands of tasks. For a few tasks with blocking I/O, threads are simpler and often faster (fewer state machine allocations, no event loop overhead).

**"Async functions are lazy; they do nothing until awaited."**

Partially true. An `async fn` returns a Future (or coroutine object) without running anything. But the Future *itself* is expensive — it allocates state on the heap. If you never await it, that allocation is wasted.

**"The event loop automatically scales to multiple cores."**

No. One event loop = one core. Production runtimes spawn one event loop per core. The scaling is explicit, not automatic.

**"Cancellation in async is like interrupts."**

Wrong. Cancellation is cooperative. A coroutine can only be cancelled at an `await` point. If a coroutine is doing CPU work, there is no mechanism to forcibly stop it short of killing the thread.

**"You can nest event loops."**

You cannot run `event_loop.run()` inside a task running on `event_loop`. This is a deadlock. Some runtimes (like Tokio) detect this and panic; others (like asyncio) silently hang.

---

## 49.11 Exercises

1. **State Machine Tracing.** Take a simple async function with two awaits and manually trace through its state machine. Write down the state values at each step. Verify that the state machine produces the correct result.

2. **Event Loop Simulation.** Implement a single-threaded event loop (like the toy example in 49.7) that can run two coroutines concurrently. Demonstrate that they interleave correctly.

3. **Backpressure.** Implement a bounded channel (queue with a size limit) and show how backpressure prevents unbounded memory growth. Measure queue depth over time with and without backpressure.

4. **Blocking Detection.** Write a program that spawns 100 async tasks. One task does CPU work for 1 second. Measure how long the other 99 tasks take to complete. Explain the result.

5. **Data Race in Async.** Write a shared counter that is incremented by 10 async tasks, each doing `increment().await` (with a dummy I/O in the middle). Show that the final count is less than 10 (demonstrating the lost-update bug). Then fix it with a Mutex and show the count is correct.

---

## 49.12 Summary

**An async function is not a function; it is a state machine.** The `async`/`await` syntax is sugar for a heap-allocated object whose `poll()` method is called repeatedly by an event loop. Each `await` point becomes a case in a switch statement where local variables are preserved in the state object.

**The event loop is a single OS thread that multiplexes thousands of coroutines.** It runs ready coroutines until they suspend (at an `await`), then blocks on I/O (via `epoll`, `kqueue`, or `io_uring`) until the OS signals that data is available. The Waker mechanism reconnects the I/O driver to the event loop: when I/O completes, the task is re-queued.

**Async is a tradeoff.** For I/O-bound workloads, async is memory-efficient and scales to thousands of concurrent tasks on a single thread. For CPU-bound work, async offers no benefit over threads and can actually hurt, because one slow coroutine blocks all others on the same thread. Production systems use async for I/O and threads (or thread pools) for CPU work.

**The colored function problem is real.** Once you call an async function, your entire call chain becomes async. This architectural cost is not always worth it for small programs or libraries that must remain synchronous-compatible.

---

> **[← Previous: Schedulers](03-schedulers.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Why Go Uses Context →](05-why-go-uses-context.md)**
