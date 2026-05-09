# Chapter 50 — Why Go Uses Context

Every Go HTTP handler in production code takes `ctx context.Context` as its first argument. The pattern is so ubiquitous that it feels like a language feature, not a library convention. It isn't a language feature. It is a solution to a hard concurrent systems problem: *in a distributed system, you need a way to say "give up — the work is no longer needed" — and that signal has to propagate through every goroutine the request spawned.*

This chapter examines what that problem actually is, how Go's `context` solves it, how other languages solve the same problem differently, and the design tradeoffs involved. By the end, you will understand not just how to use context, but why no concurrent system can live without something like it.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. **Identify the resource-leakage problem context solves** — why goroutines left running after a request is no longer needed cause cascading resource exhaustion.
2. **Explain what `context.Context` actually is** — an interface, not magic; the core mechanism is a closed channel.
3. **Trace how cancellation propagates** through parent and child contexts, and what guarantee Go provides about ordering.
4. **Implement cancellation patterns correctly** — passing context as the first parameter, never storing contexts in struct fields, always checking `Done()`.
5. **Reason about deadlines and timeouts** — including how they compose and what happens when a parent deadline is tighter than a child's.
6. **Design context values properly** — and understand why this feature is controversial even among Go advocates.
7. **Read equivalent cancellation systems in other languages** — Rust, Java, .NET, JavaScript — and recognize the convergent design.

---

## 50.1 The Problem: Resource Leaks in Distributed Systems

Start with a concrete scenario: an HTTP request arrives at a web server. The handler receives it, spawns three goroutines:

- One queries the user database.
- One queries the recommendation service.
- One queries the cache.

All three run concurrently. The handler waits for all three to complete, aggregates the results, and sends a response.

Now: the client disconnects after 500 milliseconds. They closed the browser tab, or their network failed. The response is never sent.

**Without cancellation:** those three goroutines keep running. The database query completes after 700ms; the result is written to a channel that no one is reading. The recommendation service keeps trying to send back a large result that will never be consumed. The cache query holds a lock on the cache that other requests are waiting for. The three goroutines hold:

- Database connections (limited resource; if enough requests disconnect, you run out).
- Memory for the result buffers (if the client disconnected before reading, the buffer is held until the goroutine finishes).
- Locks or semaphores (if the goroutine holds a lock while waiting for I/O, no one else can proceed).
- Network sockets (to the downstream services; each one is a file descriptor).

This is a **resource leak**, and it scales nonlinearly. If requests habitually disconnect after 500ms but take 5 seconds to complete, you have 10x the goroutines you need, 10x the database connections, 10x the memory. Your server crashes.

**With cancellation:** when the connection closes (or a timeout fires, or the client sends a cancellation signal), every goroutine that knows about the request can *check* whether to keep working. The three spawned goroutines can see "the request is no longer needed" and return early. The database connection returns to the pool. The memory is freed. The lock is released.

The key insight: **cancellation is not forced termination.** A goroutine does not die when you signal it to cancel. It receives a *suggestion* to stop, via a closed channel, and then it is responsible for stopping at a safe point. This is cooperative cancellation, and it is what makes resources predictable.

---

## 50.2 What Context Actually Is

Go's `context.Context` is an interface:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

Four methods. Let us understand what each does.

### `Done() <-chan struct{}`

Returns a channel that is **closed** when the context is cancelled. (It does not send a value; the channel closes, which all receivers see at once.) Reading from `Done()` before it closes will block. Reading from a closed channel always returns the zero value immediately.

This is the core mechanism of cancellation. A goroutine can do:

```go
select {
case result := <-workChan:
    // Work produced a result
    process(result)
case <-ctx.Done():
    // Context was cancelled; stop immediately
    return ctx.Err()
}
```

The `select` statement will unblock as soon as *either* the work completes or the context is cancelled, whichever comes first.

### `Err() error`

Returns the reason why the context was cancelled, or `nil` if it is not cancelled. Errors are:

- `context.Canceled` if the context was explicitly cancelled via the cancel function.
- `context.DeadlineExceeded` if the deadline passed.

Always check `Err()` *after* detecting that `Done()` is closed to find out why:

```go
<-ctx.Done()
if err := ctx.Err(); err == context.Canceled {
    // Handle explicit cancellation
} else if err == context.DeadlineExceeded {
    // Handle timeout
}
```

### `Deadline() (time.Time, bool)`

Returns the deadline (if one exists) and a boolean indicating whether it is set. Does not tell you *how much time is left*; you compute that yourself: `deadline.Sub(time.Now())`.

A deadline does not *itself* cancel the context. It is purely informational. The actual cancellation is triggered by a timer that calls the cancel function.

### `Value(key interface{}) interface{}`

A key-value store for request-scoped data. Typical use: storing a trace ID, a user identity, or request-specific configuration. We will cover this in §50.6.

---

## 50.3 How Cancellation Propagates

The standard library provides two functions to create child contexts:

```go
func WithCancel(parent Context) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithCancel(parent)` returns two things:

1. A new child context.
2. A `CancelFunc` — a function you call to cancel that context.

When you call the `CancelFunc`, it closes the child's `Done()` channel. Anyone waiting on `<-ctx.Done()` unblocks. Simple and mechanical.

**Parent-to-child propagation:** if the parent context is cancelled, the child is automatically cancelled too. The parent closing its `Done()` channel does not directly close the child's; instead, the child is created with awareness of its parent, and whenever the parent's `Done()` closes, the child's also closes.

**The guarantee:** cancellation is monotonic. Once a context is cancelled, it stays cancelled. A child context is cancelled if *any ancestor* is cancelled.

Here is a typical pattern:

```go
// HTTP handler
func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Parent context from the HTTP server
    
    // Create a child context with a 500ms timeout
    ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()
    
    // Spawn three workers
    go queryDatabase(ctx)
    go queryRecommendations(ctx)
    go queryCache(ctx)
    
    // Wait for results or timeout
    // ...
}

func queryDatabase(ctx context.Context) {
    result, err := db.QueryContext(ctx, "SELECT ...")
    if err != nil {
        // Handle err, which might be context.DeadlineExceeded
        return
    }
    // Use result
}
```

The `db.QueryContext()` function respects the context: if the deadline passes before the query completes, the driver will cancel the query and return `context.DeadlineExceeded`. The child context propagates the deadline down. If the HTTP server closes the connection before the handler returns, the top-level context is cancelled, which cancels the timeout context, which unblocks the workers.

---

## 50.4 Deadlines and Timeouts

`WithTimeout(parent, duration)` is syntactic sugar for `WithCancel` plus a timer. Under the hood:

```go
ctx, cancel := context.WithTimeout(parent, 500*time.Millisecond)
// Internally, this creates:
// 1. A child context with a cancel function
// 2. A timer that will call cancel() after 500ms
```

**Deadline composition:** if the parent has a deadline of 1 second from now, and you ask for a 5-second timeout, the child gets the tighter deadline (1 second). Go chooses the earlier deadline:

```go
parent, cancel1 := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel1()

child, cancel2 := context.WithTimeout(parent, 5*time.Second)
defer cancel2()

// child's effective deadline is 1 second, not 5 seconds
```

This is correct behavior because the parent's deadline is a hard constraint; the child cannot outlive it.

**One subtlety:** you should always `defer cancel()` even when using `WithTimeout`. The timer runs in the background; if you do not cancel it, it keeps running until the timeout expires, wasting CPU and goroutines. Calling cancel early saves resources.

---

## 50.5 Context Values

Context can store request-scoped key-value pairs:

```go
ctx = context.WithValue(ctx, "user_id", 42)
ctx = context.WithValue(ctx, "trace_id", "abc123")

// Later, in a nested function:
userID := ctx.Value("user_id")  // Returns 42
```

This is useful for propagating request identity (user ID, trace ID, request ID) through the call stack without passing explicit parameters.

**The mechanism is type-erased:** the value is stored as `interface{}`, and the key is also `interface{}`. This works but is error-prone:

```go
// Anti-pattern: string keys collide
ctx = context.WithValue(ctx, "user_id", 42)
// Later, in different code:
ctx = context.WithValue(ctx, "user_id", "alice")  // Overwrote previous!
```

**Best practice:** use custom types as keys to prevent collisions:

```go
type userIDKey struct{}

ctx = context.WithValue(ctx, userIDKey{}, 42)
ctx = context.WithValue(ctx, userIDKey{}, "alice")  // Different key type!

// Later:
userID := ctx.Value(userIDKey{})  // Returns 42
```

**Why this is controversial:** `Value()` makes the context a grab-bag of implicit dependencies. A function that calls `ctx.Value("x")` is now depending on data that is not visible in its signature. Newcomers to the function cannot easily discover what keys it looks for. It is "magic" in a language that prides itself on explicitness.

Go's convention is: **use context values sparingly, only for request-scoped metadata (trace IDs, user IDs, deadlines), never for application state.** The standard library's `context` package does not even provide a `context.WithValue` example in most tutorials — it is considered advanced.

---

## 50.6 Correct Usage Patterns

### Always Pass Context as the First Parameter

Go convention is strict: context is *always* the first parameter of a function:

```go
// Good
func FetchUser(ctx context.Context, userID int) (*User, error)
func WriteLog(ctx context.Context, message string)

// Incorrect
func FetchUser(userID int, ctx context.Context) (*User, error)
func FetchUser(ctx context.Context, userID ...int) (*User, error)
```

This is not enforced by the compiler; it is convention. But it is so consistent that an IDE can flag violations. The reason: it signals that the function respects cancellation and timeouts, making it safe to call from concurrent code.

### Never Store Contexts in Struct Fields

**Anti-pattern:**

```go
type Handler struct {
    ctx context.Context  // WRONG
    db  *sql.DB
}

func (h *Handler) Handle() {
    // Use h.ctx
}
```

Why? Because the same `Handler` instance is reused across requests, each with a different context. Storing the context in the struct means it persists after one request finishes and before the next begins, leading to subtle bugs.

**Correct pattern:**

```go
type Handler struct {
    db *sql.DB
}

func (h *Handler) Handle(ctx context.Context) {
    // Use ctx
}
```

Pass context as a parameter every time.

### Always Check `Done()` Before Expensive Work

If you are about to do something expensive (a database query, a network call), check the context first:

```go
// Good
func FetchData(ctx context.Context) {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }
    
    // Now safe to do expensive work
    return db.Query(ctx, ...)
}

// Also correct: many libraries (sql, http) check automatically
result, err := db.QueryContext(ctx, ...)
// If ctx is cancelled, err will be context.DeadlineExceeded or Canceled
```

The benefit: you do not waste CPU spinning up a query that will never be used.

### Honor `Done()` in Long-Running Loops

If you are doing work in a loop, check `Done()` regularly:

```go
// Good
func ProcessBatch(ctx context.Context, items []Item) error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        
        if err := process(item); err != nil {
            return err
        }
    }
    return nil
}

// Poor: does not check context until the entire batch is done
func ProcessBatchSlow(ctx context.Context, items []Item) error {
    for _, item := range items {
        if err := process(item); err != nil {
            return err
        }
    }
    return nil
}
```

The select statement has no-op overhead; the benefit is responsiveness.

---

## 50.7 Worked Example: HTTP Handler with Deadline and Cancellation

Here is a realistic handler that fans out to two services:

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Get the context from the HTTP request
    ctx := r.Context()
    
    // Create a child context with a 500ms deadline
    ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()
    
    // Channel to collect results
    type result struct {
        source string
        data   string
        err    error
    }
    results := make(chan result, 2)
    
    // Start two workers
    go func() {
        data, err := fetchFromServiceA(ctx)
        results <- result{"ServiceA", data, err}
    }()
    
    go func() {
        data, err := fetchFromServiceB(ctx)
        results <- result{"ServiceB", data, err}
    }()
    
    // Collect results or timeout
    var responses []result
    for i := 0; i < 2; i++ {
        select {
        case res := <-results:
            responses = append(responses, res)
        case <-ctx.Done():
            // Deadline exceeded or request cancelled
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
    }
    
    // Aggregate and return
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(responses)
}

func fetchFromServiceA(ctx context.Context) (string, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", "https://service-a.local/data", nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err  // err might be context.DeadlineExceeded
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}

func fetchFromServiceB(ctx context.Context) (string, error) {
    // Similar pattern
    req, _ := http.NewRequestWithContext(ctx, "GET", "https://service-b.local/data", nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}
```

**What happens when the client disconnects:**

1. The HTTP server detects the closed connection.
2. It cancels the context returned by `r.Context()`.
3. The timeout context (child of the server's context) also gets cancelled.
4. Both workers are blocked in `http.Do(req)`, waiting for the remote service.
5. The HTTP client driver sees the context is cancelled and returns early with an error.
6. Both workers send results (with errors) to the channel.
7. The handler's `select` statement is unblocked by the context cancellation.
8. The handler returns; all goroutines exit.

If neither worker finished by the 500ms deadline, the same flow happens, but triggered by the timer instead of the network close.

---

## 50.8 How Other Languages Solve This Problem

Cancellation is not a Go invention. Every concurrent language needs a way to stop work.

### Rust: Cancellation Tokens and Dropping Futures

Rust has no garbage collection, so it cannot use "closing a channel" as a signal. Instead, Tokio (Rust's async runtime) provides `CancellationToken`:

```rust
use tokio::task::CancellationToken;

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let token_child = token.child_token();
    
    let task = tokio::spawn(async move {
        tokio::select! {
            _ = long_running_task() => println!("Task done"),
            _ = token_child.cancelled() => println!("Cancelled"),
        }
    });
    
    // Cancel after 500ms
    tokio::time::sleep(Duration::from_millis(500)).await;
    token.cancel();
    
    task.await.unwrap();
}
```

Rust also has "dropping" — if a future is dropped (goes out of scope), its cleanup runs via the `Drop` trait. This is deterministic cancellation: no goroutine left behind because the future's `drop()` is called.

### Java: Thread.interrupt() and CompletableFuture.cancel()

Java's older thread model uses `Thread.interrupt()`:

```java
Thread worker = new Thread(() -> {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            doWork();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

worker.start();

// Later, cancel it
worker.interrupt();
```

The issue: interruption is a *suggestion* (like context), but not all operations respond to it. A blocking network call does not respond to interruption until the I/O completes. Modern Java uses `CompletableFuture` and `Virtual Threads` (since Java 19), which are closer to Go's model:

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return fetchData();
});

// After 500ms, cancel
future.completeExceptionally(new TimeoutException());
```

### .NET: CancellationToken

.NET's approach is nearly identical to Go's `Context`:

```csharp
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromMilliseconds(500));

try {
    var result = await FetchDataAsync(cts.Token);
} catch (OperationCanceledException) {
    // Timeout or cancellation
}

async Task<string> FetchDataAsync(CancellationToken ct) {
    using var client = new HttpClient();
    return await client.GetStringAsync("https://...", ct);
}
```

The `CancellationToken` is not an interface; it is a struct. But the semantics are identical: propagate a token through call chains; check it before expensive work; libraries cooperate by checking it.

### JavaScript: AbortController

JavaScript's approach mirrors the others:

```javascript
const controller = new AbortController();

setTimeout(() => controller.abort(), 500);

try {
    const response = await fetch('https://...', {
        signal: controller.signal
    });
} catch (err) {
    if (err.name === 'AbortError') {
        console.log('Request cancelled');
    }
}
```

The `AbortSignal` (the equivalent of context's `Done()`) can be passed to fetch, setTimeout, addEventListener, etc. The browser's event loop checks it; user code also checks `signal.aborted`.

### Pattern Recognition

Across all five languages, the design converges:

1. **Cancellation token/context** — an object representing a logical operation that can be cancelled.
2. **Parent-child hierarchy** — tokens compose; child tokens are cancelled if the parent is.
3. **Check-before-expensive-work** — a cooperative signal, not forced termination.
4. **Propagation through call chains** — each function receives the token/context and passes it downstream.
5. **Timeout support** — a special case of cancellation triggered by a timer.

The convergence suggests this is *the* right abstraction for concurrent systems.

---

## 50.9 Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| **Cooperative cancellation (signal, not forced stop)** | Resources are released cleanly; safe to cancel during I/O. | Requires every goroutine to check. Unresponsive code can ignore signals. |
| **Interface-based (Go) vs token struct (.NET)** | Pluggable; multiple implementations. | Slightly more indirection. |
| **Deadline support** | Uniform timeout across the entire call tree. | Requires comparing times at every level; clock drift can cause surprises. |
| **Context values for metadata** | Avoid cluttering function signatures. | Type erasure; implicit dependencies; easy to collide. |
| **Pass context as first parameter (convention)** | Readable; IDE support; consistency. | Enforced by social pressure, not the language. |
| **Parent-child cancellation** | Simple to reason about; prevents accidental cancellation of unrelated work. | Requires careful context creation; easy to pass the wrong context. |

---

## 50.10 Common Misconceptions

### Misconception 1: "Cancellation forces a goroutine to stop"

Reality: Cancellation is a *signal*. The goroutine must check `Done()` and stop itself. If a goroutine never checks, it never stops. This is by design — Go does not terminate threads mid-operation because it would leak resources.

```go
// Even with cancellation, this goroutine will not stop until 10 seconds have passed
func stubborn(ctx context.Context) {
    time.Sleep(10 * time.Second)  // Never checks ctx
    return
}
```

### Misconception 2: "Context values are a good way to pass data"

Reality: Context values are for *metadata*, not data. Metadata is request-scoped, usually system-level (user ID, trace ID), and is rarely used. Application data should be passed as parameters.

```go
// Good: metadata
ctx = context.WithValue(ctx, traceIDKey{}, "abc123")

// Poor: application data
ctx = context.WithValue(ctx, "config", appConfig)
ctx = context.WithValue(ctx, "database", db)
```

### Misconception 3: "Storing context in a struct is a micro-optimization"

Reality: It is wrong, not a micro-optimization. The same struct instance is reused across multiple requests, each with a different context. You will use the previous request's context for the current request, leading to unpredictable timeouts and cancellations.

### Misconception 4: "Defer cancel() is optional if the timeout fires anyway"

Reality: You should always defer cancel. The timer runs in the background (it is a real goroutine in Go's runtime). If you do not cancel it, it keeps running, holding a goroutine alive even after the timeout would have fired. Calling cancel explicitly stops the timer immediately.

---

## 50.11 Exercises

1. **Write a handler with cascading timeouts.** Create an HTTP handler that calls two downstream services. The overall deadline is 1 second. The first service has a 400ms budget; the second has 400ms. The handler should aggregate results or timeout early if either service exceeds its budget. (Hint: use two `WithTimeout` calls, or one parent timeout and split the time manually.)

2. **Detect and fix a context leak.** The following code has a subtle bug:
   ```go
   func processRequest(ctx context.Context) {
       ctx, cancel := context.WithCancel(ctx)
       // Missing: defer cancel()
       
       go func() {
           // Do work
       }()
   }
   ```
   What is the consequence? Rewrite it correctly.

3. **Implement a context value for request tracing.** Create a key type for storing trace IDs. Write two functions: one that creates a context with a trace ID, and another that logs a message with the trace ID from the context. Call both from a handler.

4. **Compare with another language.** Write the same handler (fan-out to two services with a deadline) in Go and either Rust (Tokio) or .NET. How do the error-handling and cancellation patterns differ? Which is easier to reason about?

5. **Build a worker pool with cancellation.** Write a worker pool (5 workers reading tasks from a channel) that properly shuts down when the context is cancelled. Ensure no goroutines leak.

---

## 50.12 Summary

Context is Go's solution to a universal problem: in a concurrent system, how do you signal that work is no longer needed, and how do you ensure that signal propagates through all the goroutines doing that work?

The answer is cooperative cancellation via a closed channel, organized hierarchically (parent and child contexts), with optional deadline support, and a convention to pass context as the first parameter of every function that respects it.

The design is not unique to Go. Rust, Java, .NET, and JavaScript all converge on similar mechanisms because the problem is real and the solution is correct. The convergence suggests that once a language reaches the level of concurrency complexity that Go targets, context or its equivalent becomes not optional but necessary.

The cost is cognitive: every function must be context-aware, and every goroutine must actively cooperate with cancellation. The benefit is that resources are released cleanly, and distributed systems do not collapse under cascading failures when clients disconnect.

---

> **[← Previous: Async Runtime Internals](04-async-runtime-internals.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Cancellation Propagation →](06-cancellation-propagation.md)**
