# Chapter 51 — Cancellation Propagation

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain why forceful thread termination (`pthread_cancel`, `Thread.stop()`) breaks invariants and why cooperative cancellation is necessary.
2. Identify four models of cancellation — signal-based, drop-based, exception-based, and channel-based — and recognize them in real code.
3. Predict the bug that happens when a parent task is cancelled but its children are not, and design propagation rules to prevent it.
4. Implement structured concurrency patterns where cleanup is guaranteed on the way out of a scope.
5. Debug a server shutdown that is leaving zombie goroutines, using cancellation metrics to spot the fault.

---

## 51.1 The Hard Problem of Stopping

Starting work in a concurrent system is easy. You spawn a goroutine, launch a thread, enqueue a task. Stopping it cleanly is the hard problem.

The naive approach — *forcefully halt the task* — seems obvious. Many early systems offered it: POSIX had `pthread_cancel`, the JVM had `Thread.stop()`, Python had `thread.interrupt_main()`. They are all dangerous.

Here is why. Imagine a thread is halfway through this operation:

```cpp
void transfer(Account& from, Account& to, int amount) {
    Lock lock1(from.mutex);
    from.balance -= amount;         // <-- Thread cancelled here
    Lock lock2(to.mutex);
    to.balance += amount;
}
```

If cancellation fires between line 2 and line 4, the lock on `to.mutex` is never acquired. `to` is corrupted — the invariant "every account's balance is correct" is broken. Any subsequent operation that reads or writes `to` is based on false state.

This is the **invariant-breaking problem**. It is not theoretical:

- A cancelled transaction that held a write lock to a database page leaves that page locked forever, hanging all subsequent queries.
- A cancelled API handler that partially modified a shared data structure leaves the structure in an impossible state; the next request that reads it crashes.
- A cancelled network handler that released memory without freeing network buffers leaves sockets leaking.

The solution is not better cancellation signals. **The solution is to remove the signal and make cancellation a message the task reads and acts upon.** This is called *cooperative cancellation* — the task cooperates by checking a cancellation flag or receiving a "stop" message, unwinding cleanly, and releasing all locks and resources as it goes.

Every modern concurrent runtime has adopted this:

- Go: cancellation is a value in a context, read by the goroutine.
- Rust: dropping a future aborts it; destructors clean up.
- Java Virtual Machine (Project Loom): cancellation is a scope exit; cleanup happens in finalizers.
- Python asyncio: cancellation raises a `CancelledError` that unwinds the stack.
- JavaScript: `AbortController` signals cancellation; the task reads it.

The shift from forced to cooperative is a shift from "I will stop you" to "you must stop yourself."

---

## 51.2 Four Models of Cancellation

Cooperative cancellation has a contract: "the system will signal that you should stop, and you will stop cleanly." But the *mechanism* of that signal differs. Here are the four dominant models.

### Model 1: Signal-Based (Flag Polling)

The system sets a flag. The task polls it, typically in a loop.

```cpp
// In the runtime or parent:
std::atomic<bool> stop_flag = false;
stop_flag = true;  // Signal cancellation

// In the task:
while (!stop_flag) {
    // Do work
    process_item();
}
return;  // Unwind cleanly, release resources
```

**How it looks in real systems:**

Go's `context.Context` is this model:

```go
ctx, cancel := context.WithCancel(context.Background())
// ... start a goroutine
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return  // Cancelled, unwind
        case item := <-work_queue:
            process(item)
        }
    }
}(ctx)
// ... later
cancel()  // Signal cancellation
```

The goroutine polls `ctx.Done()` in a select; when it fires, the goroutine exits.

**Pros:** Simple; no exceptions; works in any language; allows time budgets (`context.WithTimeout`).

**Cons:** The task must explicitly check; forgotten checks mean the task ignores cancellation. Blocking I/O (e.g., `socket.read()` on TCP) does not wake when the flag is set — you must interrupt the I/O separately.

### Model 2: Drop-Based (Resource Destruction)

Dropping a reference to a task automatically cancels it. Used in Rust and languages with deterministic finalization.

```rust
{
    let task = tokio::spawn(async {
        loop {
            do_work().await;
        }
    });
    // task is running
}
// task goes out of scope, dropped
// task is cancelled immediately
```

**Pros:** Automatic; no explicit cancellation call needed; cleanup is guaranteed by the type system (destructors run). Solves the "forgot to cancel" problem.

**Cons:** The task must be expressible as a type (a future, a handle). Cancellation is immediate, not cooperative — the task is not asked to stop, it is aborted. Works well only if your "cleanup" is just deallocating memory; if you have transactions to roll back or locks to release, immediate destruction is unsafe.

### Model 3: Exception-Based (Unwinding Stack)

Cancellation throws an exception that the task catches and handles.

```python
# In the event loop:
task = asyncio.create_task(background_work())
# ... later
task.cancel()  # Raises CancelledError in the task

# In the task:
async def background_work():
    try:
        while True:
            await some_io()
    except asyncio.CancelledError:
        print("Cancelled, cleaning up")
        close_resources()
        raise  # Re-raise or return
```

**How it looks in JavaScript:**

```javascript
const controller = new AbortController();
const task = fetch(url, { signal: controller.signal });
// ... later
controller.abort();  // Propagates to the fetch
```

**Pros:** Synchronous, visible in the code; the try/except is obvious. The task is "asked" (via exception) to stop, not forced.

**Cons:** Exceptions have high CPU cost in some languages (e.g., C++ with RTTI). Not all APIs check the signal. Interleaving exception handlers with locks can still cause deadlocks.

### Model 4: Channel-Based (Receiving a Stop Message)

Cancellation is a message sent on a channel. The task reads it and exits.

```go
stop := make(chan struct{})
go func() {
    for {
        select {
        case <-stop:
            close_resources()
            return
        case item := <-work_queue:
            process(item)
        }
    }
}()
// ... later
close(stop)  // Send the stop signal
```

**Pros:** Explicit and familiar (same pattern as work); no global state; works well for producer-consumer patterns.

**Cons:** Requires plumbing channels through the call stack. More verbose than flag-based models.

---

## 51.3 The Propagation Problem

Cancellation becomes a systems problem when tasks spawn other tasks. A parent task that is cancelled should propagate that cancellation to all its children. Without explicit propagation, children keep running, orphaned.

Here is the bug, in Go:

```go
// Buggy: cancels parent but not children
func handle_request(ctx context.Context, id string) error {
    // ctx might be cancelled by the parent (HTTP handler timeout)
    
    // These goroutines are NOT cancelled when ctx is cancelled
    go fetch_profile(id)
    go fetch_history(id)
    go fetch_recommendations(id)
    
    return nil  // Parent returns, goroutines still running
}
```

If the HTTP handler's deadline is exceeded and `ctx` is cancelled, the parent returns immediately. The three goroutines still running in the background continue to fetch data, burn CPU, and hold connections open. After the handler returns, the client has already moved on (timeout); the work is wasted.

This is the **orphaned task** problem. In a server that handles 1000 requests/second, each with a forgotten child cancellation, you accumulate 1000s of zombie goroutines. Memory grows unbounded. Open sockets leak. The server becomes an resource sink.

**The propagation rule:** When a parent task receives a cancellation signal, it must immediately:

1. Stop accepting new work.
2. Signal all child tasks to cancel (recursively).
3. Wait for children to finish (unwind and release resources).
4. Release its own resources (locks, open files, connections).
5. Return or exit.

In code:

```go
func handle_request(ctx context.Context, id string) error {
    // Derive a child context; if parent is cancelled, this is too
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()  // Ensure cleanup
    
    // These goroutines inherit the cancellation
    errCh := make(chan error, 3)
    go func() { errCh <- fetch_profile(ctx, id) }()
    go func() { errCh <- fetch_history(ctx, id) }()
    go func() { errCh <- fetch_recommendations(ctx, id) }()
    
    var errs []error
    for i := 0; i < 3; i++ {
        if err := <-errCh; err != nil {
            errs = append(errs, err)
        }
    }
    
    // All children finished (cancelled or completed)
    return errs[0]  // Return early if any error
}
```

When the parent context is cancelled, `ctx` is also cancelled (both children receive the signal). The function waits for all three goroutines to finish before returning. No orphans.

---

## 51.4 Structured Concurrency

The propagation rule is manual and error-prone. Modern runtimes offer *structured concurrency* — a pattern where cancellation flows naturally through the call stack.

**Structured concurrency guarantee:** A parent task does not finish until all child tasks have finished. Cancellation propagates automatically. Resources are cleaned up in reverse order of allocation.

This is not a new idea. It is how sequential programming works:

```cpp
void main() {
    try {
        func_a();
        func_b();
        func_c();
    } catch (...) {
        // All three functions have exited (or thrown)
        // Their stack frames are unwound
    }
}
```

Each function is a "sequential scope." Before `main` returns, all three functions have finished. Structured concurrency applies the same principle to *concurrent* scopes:

```go
// Structured concurrency in Java (Project Loom):
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> fetch_profile(id));
    scope.fork(() -> fetch_history(id));
    scope.fork(() -> fetch_recommendations(id));
    scope.joinUntil(Instant.now().plus(Duration.ofSeconds(1)));
} catch (TimeoutException e) {
    // All three are cancelled; scope cleans up
}
```

Or in Kotlin coroutines:

```kotlin
coroutineScope {
    val profile = async { fetch_profile(id) }
    val history = async { fetch_history(id) }
    val recommendations = async { fetch_recommendations(id) }
    // coroutineScope does not exit until all three async tasks finish
}
// If any task throws, all are cancelled
```

**Why this is the dominant model:**

1. **Automatic propagation.** Cancellation flows from parent to children without explicit plumbing.
2. **No orphans.** The parent cannot exit while children are running.
3. **Cleanup by scope.** Exiting a scope (normally or via exception) triggers resource cleanup in reverse order.
4. **Familiar.** It mirrors sequential code structure; less cognitive load.

---

## 51.5 Cleanup On Cancellation

Cancellation is only half the problem. The other half is **what happens to resources when the task unwinds**.

A task might hold:

- A lock (mutex, semaphore, read-write lock).
- An open file or network socket.
- A database transaction (with partial writes).
- An allocated buffer or memory chunk.
- A reference to a shared data structure.

If the task is cancelled mid-operation, these resources must be released, or the system deadlocks or leaks.

**Strategy 1: RAII / Scope-Based Cleanup**

Resources are acquired in constructors and released in destructors. Exiting a scope (via exception or normal return) automatically releases them.

```cpp
{
    std::lock_guard<std::mutex> lock(mtx);  // Acquire
    do_work();
    // lock destroyed here (release)
}  // Exiting scope = automatic cleanup
```

This is why C++ adopted RAII. It is also why Rust's Drop trait is mandatory. In both languages, unwinding the stack from a cancellation automatically releases resources.

**Strategy 2: Explicit Cleanup (defer/finally)**

Resources are released in an explicit finally block, executed on the way out (whether normal or exceptional).

```java
FileWriter f = new FileWriter("output.txt");
try {
    do_work(f);
} finally {
    f.close();  // Always executed
}
```

Or in Go:

```go
lock := acquire_lock(resource)
defer lock.Release()  // Always executed, even if cancelled
do_work()
```

Both strategies have the same guarantee: cleanup code always runs.

**The bug that happens without either:**

```go
// Wrong: no cleanup on cancel
func process_with_lock(ctx context.Context) {
    lock.Lock()
    
    for {
        select {
        case <-ctx.Done():
            return  // LEAK: lock is never released
        case item := <-queue:
            process(item)
        }
    }
}
```

When `ctx` is cancelled, the function returns immediately. The lock is held forever. Any other goroutine that tries to acquire it blocks indefinitely.

The fix:

```go
func process_with_lock(ctx context.Context) {
    lock.Lock()
    defer lock.Unlock()  // Always released
    
    for {
        select {
        case <-ctx.Done():
            return  // Clean exit
        case item := <-queue:
            process(item)
        }
    }
}
```

---

## 51.6 Cancellation in HTTP Servers

An HTTP request is a natural parent task. The handler (the function that processes the request) may spawn background work (database queries, cache lookups, downstream API calls). If the client disconnects, the request is cancelled — the handler should cancel its children and exit.

**The problem:** Most servers do not propagate client disconnection by default.

Here is the naive server:

```go
// Without cancellation
http.HandleFunc("/data", func(w http.ResponseWriter, r *http.Request) {
    // Client disconnects while we're querying the database
    // We keep querying, burning CPU and DB connections
    data := query_database()
    w.Write(data)
})
```

If the client disconnects while `query_database()` is blocked, the handler still waits for the result, then tries to write it to an already-closed connection.

Here is the corrected version:

```go
http.HandleFunc("/data", func(w http.ResponseWriter, r *http.Request) {
    // r.Context() is cancelled when the client disconnects
    ctx := r.Context()
    
    // Pass cancellation to the query
    data, err := query_database_with_timeout(ctx, 5*time.Second)
    if errors.Is(err, context.Canceled) {
        return  // Client disconnected, exit cleanly
    }
    w.Write(data)
})
```

In Node.js:

```javascript
server.on('request', (req, res) => {
    const controller = new AbortController();
    
    // If client disconnects, abort all work
    req.on('close', () => controller.abort());
    
    fetch(upstream_url, { signal: controller.signal })
        .then(data => res.end(data))
        .catch(err => {
            if (err.name === 'AbortError') {
                // Client disconnected, cleanup already happened
            }
        });
});
```

The difference is dramatic in production. Without propagation, a slow upstream service blocks hundreds of client connections. With propagation, clients that give up free their resources immediately.

---

## 51.7 Worked Example: Three Downstream Calls with a Deadline

A server receives a request. It must fetch from three downstream services (profile service, order service, analytics service) and merge the results. The deadline is 200ms from the request start.

**Without cancellation propagation:**

```go
func get_user_profile(id string) (Profile, error) {
    profile := fetch_service(profile_url, id)       // No deadline
    orders := fetch_service(order_url, id)           // No deadline
    analytics := fetch_service(analytics_url, id)    // No deadline
    
    // If any service is slow, we wait for all
    // Requests pile up; connections leak
    return merge(profile, orders, analytics)
}
```

If the profile service is slow, the request waits 200ms (or more). The handler is blocked. The connection to the client is held open. If 1000 requests hit this slow path, 1000 connections are blocked.

**With cancellation propagation:**

```go
func get_user_profile(ctx context.Context, id string) (Profile, error) {
    // Timeout context; cancelled after 200ms
    ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()
    
    // Each service is passed the timeout context
    profileCh := make(chan Profile, 1)
    ordersCh := make(chan Orders, 1)
    analyticsCh := make(chan Analytics, 1)
    
    go func() {
        p, _ := fetch_service(ctx, profile_url, id)
        profileCh <- p
    }()
    go func() {
        o, _ := fetch_service(ctx, order_url, id)
        ordersCh <- o
    }()
    go func() {
        a, _ := fetch_service(ctx, analytics_url, id)
        analyticsCh <- a
    }()
    
    // Wait for all three or for timeout
    var profile Profile
    var orders Orders
    var analytics Analytics
    
    for i := 0; i < 3; i++ {
        select {
        case profile = <-profileCh:
        case orders = <-ordersCh:
        case analytics = <-analyticsCh:
        case <-ctx.Done():
            // 200ms elapsed; all three goroutines cancelled
            return Profile{}, ctx.Err()  // Return error immediately
        }
    }
    
    return merge(profile, orders, analytics)
}
```

When the 200ms deadline is reached:

1. `ctx.Done()` fires.
2. All three `fetch_service` calls are cancelled (they poll their context).
3. The goroutines unwind immediately, closing connections to downstream services.
4. The handler returns an error to the client.
5. The connection is released.

**The metrics:**

- **Without propagation:** Slow requests have long tail latency (p99 and p999 spike). Server CPU grows during slowdowns.
- **With propagation:** Slow requests are cancelled at the deadline. Tail latency is capped. Server CPU is stable even under slow downstream services.

---

## 51.8 Tradeoffs

| Model | Pros | Cons | When to use |
|---|---|---|---|
| **Signal-based (flag polling)** | Simple; no language overhead; works anywhere | Task must explicitly check; blocking I/O does not wake; latency from flag-set to check | Tight loops; simple tasks; languages without exceptions |
| **Drop-based (resource destruction)** | Automatic; no manual cancellation; type-safe | Immediate (not cooperative); requires deterministic destructors; not all cleanup is resource-safe | Rust; low-latency systems; tasks with no side effects |
| **Exception-based (unwinding)** | Synchronous; visible in code; familiar control flow | High CPU cost; requires exception-safe code; locks + exceptions = deadlock hazard | Languages with cheap exceptions (Go, Python); when cleanup is always the same |
| **Channel-based (stop message)** | Explicit; works well for producer-consumer; no global state | Verbose; requires plumbing channels through call stack; can deadlock if not careful | Go, Rust channels; loosely-coupled services |
| **Structured concurrency (scope-based)** | Automatic propagation; no orphans; natural cleanup; mirrors sequential code | Requires runtime support; limits concurrency patterns (must be fork-join); not all libraries support it | Modern servers (Go, Java Loom, Kotlin, Trio); services with deadline-driven work |

---

## 51.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Cancellation should be fast, so I'll use a flag check." | The bottleneck is usually not the flag check (nanoseconds) but the time from flag-set to the next check (milliseconds). For tight latency (< 10ms), care about how often you check, not what you check. |
| "If I cancel the parent, the children will be cancelled automatically." | Only if the runtime implements propagation (Go, Java Loom, Kotlin). Most runtimes (C++, raw threads) do not. You must pass cancellation explicitly. |
| "Structured concurrency is just fork-join." | Structured concurrency is fork-join *with automatic propagation and cleanup*. The structure matters because it enables the automation. |
| "Cleanup via finally/defer is the same as destructors." | Functionally similar; different failure modes. Finally blocks can throw (leaving cleanup incomplete); destructors usually cannot. RAII is stronger. |
| "I can ignore cancellation inside critical sections." | You can, but then you're trading latency (delayed shutdown) for simplicity. In a system where cancellation deadline matters (p99 latency SLA), ignored cancellation inside locks is a red flag. |

---

## 51.10 Exercises

1. **Spot the orphaned task.** Given a Go server handler with three spawned goroutines, identify which one is orphaned if the parent context is cancelled. Explain what resource leaks as a result.

2. **Implement the propagation rule.** Write a C++ function that spawns two threads and waits for both to finish. When the parent is signalled to stop (via a `std::atomic<bool>`), both children should immediately cancel and unwind. Show the cleanup code.

3. **Translate cancellation models.** Rewrite the same task — "fetch data with a 100ms timeout" — using four models: (a) signal-based, (b) exception-based, (c) channel-based, (d) structured concurrency. Which is clearest? Which is most error-prone?

4. **Measure the leak.** Write a server that receives requests. In the handler, spawn three goroutines that each sleep for 10 seconds. Without cancellation propagation, measure the number of goroutines as requests arrive and complete (use `runtime.NumGoroutine()`). Then add cancellation. How many goroutines survive now?

5. **Cleanup under pressure.** Write a task that holds a lock, acquires a file descriptor, and makes a network call, all with a deadline. If the deadline is exceeded, verify that all three are released in the correct order.

---

## 51.11 Summary

Cancellation is necessary but dangerous. Forced termination breaks invariants; cooperative cancellation requires the task to unwind cleanly. Four models exist — signal-based (flag polling), drop-based (resource destruction), exception-based (unwinding), and channel-based (stop message) — each with different latency and clarity tradeoffs. The hard problem is propagation: cancelling a parent must cancel all its children, or orphaned tasks accumulate. Structured concurrency solves this by making propagation automatic and cleanup mandatory. In HTTP servers, propagating client disconnection prevents zombie goroutines and bounds tail latency. Understanding these patterns is essential for building servers that remain stable under overload.

---

**[← Previous: Why Go Uses Context](05-why-go-uses-context.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Event Loops →](07-event-loops.md)**
