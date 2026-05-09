# Chapter 53 — Coroutines

A coroutine is a function that can suspend itself and resume later. That sentence sounds simple. The mechanism behind it — and the variations across languages — is one of the deeper ideas in modern concurrency.

A normal function call pushes a stack frame and runs to completion. A coroutine call captures the in-progress state somewhere and returns to its caller without finishing. The state can be resumed later, from exactly where it left off, with all local variables intact. This simple idea has profound implications: one OS thread can drive thousands of concurrent tasks; you can write sequential-looking code that is actually interleaved; and the boundary between "function" and "state machine" becomes blurry.

By the end of this chapter you will understand what happens when a coroutine suspends, why different languages choose different implementations, how to write C++20 coroutines, and what tradeoffs you are accepting when you use them.

---

## 53.1 What Suspension Actually Means

In Chapter 7, we defined the call stack: a chain of frames, each representing one active function call. The frame contains the return address, saved registers, local variables, and outgoing arguments. At the moment a function calls another, a new frame is pushed. At the moment a function returns, its frame is popped.

This model assumes a strict LIFO (last-in-first-out) order: frames are created and destroyed in a nested, hierarchical way. The outermost frame (in `main`) is created first and destroyed last. All the frames in between follow strict nesting rules.

**A coroutine breaks this rule.** Instead of returning (which pops its frame), it suspends (which keeps its frame somewhere safe) and returns control to its caller without destroying itself. Later, that saved state is resumed, and the coroutine picks up exactly where it left off.

Here is a diagram comparing the two:

```
Normal Function Flow        Coroutine Flow
========================    ====================

main calls f1               main calls coro1
  |                           |
  +-> f1's frame exists        +-> coro1's frame exists
  |                           |
  +-> f1 calls f2             +-> coro1 calls co_await (suspends)
  |    |                      |
  |    +-> f2's frame exists   +-> coro1 returns to main
  |    |                       |    (frame is saved somewhere)
  |    +-> f2 returns          |
  |    (f2's frame destroyed)  |
  |                           |
  +-> f1 continues            main continues...
  |                           main later resumes coro1
  +-> f1 returns               |
  (f1's frame destroyed)       +-> coro1's frame is restored
                              |
  Call stack is now empty      +-> coro1 continues from co_await
                              |
                              +-> coro1 may co_await again
                              |
                              +-> coro1 co_return
                              (frame is destroyed)
```

The key difference: a coroutine's frame is not on the call stack. It is on the heap. The heap frame persists across suspension. The OS thread's stack is freed up to run other work (or other coroutines).

---

## 53.2 Stackful vs Stackless: Two Philosophies

There are two fundamental approaches to implementing coroutines: **stackful** and **stackless**.

### Stackful Coroutines

A **stackful** coroutine has its own stack — a separate region of memory, usually allocated on the heap and sized by the programmer.

When a stackful coroutine suspends, the entire stack (all the frames on it) is preserved. When it resumes, the stack is restored, and execution continues from the saved point.

**Example systems:**
- **Lua coroutines.** Each coroutine has its own heap-allocated stack.
- **Boost.Coroutine** (C++). Programmatic allocation of a stack (usually 64 KB to 1 MB).
- **Go goroutines (early versions).** Originally, each goroutine had a dedicated, growable stack (changed later).
- **Fibers (Windows, game engines).** User-allocated stacks managed by the program.

**Cost model:**
- Context switch: ~50–100 ns (just save and restore `rsp` and a few other registers).
- Memory: fixed per coroutine (e.g., 64 KB minimum). Scales poorly to millions.
- Unwinding: simple — just pop frames normally.

**Conceptual clarity:**
Stackful coroutines feel like threads. You can have a C function at the bottom, call into C++, which calls into a library, which suspends — and on resumption, all the frames are there, unchanged. From the perspective of nested function calls, nothing strange happened.

### Stackless Coroutines

A **stackless** coroutine does not have its own stack. Instead, its state — the local variables and execution point — is stored in a heap-allocated structure. There is no "nested call stack" within the coroutine frame.

When a stackless coroutine suspends, the OS thread's stack is completely unwound (or freed to do other work). When it resumes, a new stack frame is pushed, and the heap state is restored.

**Example systems:**
- **C++20 coroutines.** The compiler generates a state machine on the heap.
- **Rust `async fn`.** Similar: a `Future` that holds saved locals.
- **JavaScript `async`/`await`.** Coroutines as closures with saved state.
- **Python `async def`.** Similar to JavaScript.

**Cost model:**
- Context switch: zero (just a function call).
- Memory: per local variable crossing a suspension point (typically tens to hundreds of bytes). Scales to millions.
- Unwinding: forced by the language. You cannot have a C function suspend; only the top-level coroutine can suspend.

**Language enforcement:**
Stackless coroutines are less forgiving. In C++20, you cannot suspend inside an arbitrary C function; the compiler tracks which functions are "coroutines" and only they can use `co_await`. The state machine is built by the compiler, not by you.

### Side-by-Side Comparison

```
╔═══════════════════════╦═══════════════════════╗
║  Stackful             ║  Stackless            ║
╠═══════════════════════╬═══════════════════════╣
║ Own stack per coro    ║ Shared thread stack   ║
║ Suspend from deep     ║ Suspend only at top   ║
║ ~100 ns switch        ║ ~0 ns switch          ║
║ ~64 KB min memory     ║ ~100 bytes memory     ║
║ Millions: limited     ║ Millions: practical   ║
║ Familiar debugging    ║ State machine complex ║
╚═══════════════════════╩═══════════════════════╝
```

**When to choose:**
- **Stackful:** If you are wrapping existing synchronous libraries (database drivers, file I/O) and need to suspend from within them without modifying them.
- **Stackless:** If you are designing a new system from scratch, can control the suspension points, and need to handle massive concurrency.

Modern systems almost always choose stackless when building a runtime from scratch. The per-coroutine memory cost is too attractive at scale.

---

## 53.3 C++20 Coroutines: The Standard

C++20 introduced coroutines via three keywords: `co_await`, `co_yield`, and `co_return`. These are fundamentally different from keyword like `async` in JavaScript — they are not tied to a runtime. Instead, they are a **compiler transformation** that you can use to build your own runtimes, or integrate with existing ones.

### Three Keywords

**`co_await`**: Suspend the coroutine, wait for the result of an awaitable, and continue.

```cpp
result = co_await some_async_operation();
```

The compiler generates a state machine that:
1. Suspends before `some_async_operation()`.
2. Stores the awaitable somewhere.
3. Resumes when the awaitable is ready, extracting its result.

**`co_yield`**: Suspend the coroutine and return a value to the caller.

```cpp
co_yield value;
```

Used in generators. The coroutine suspends, the value is returned to the caller (who is typically in a loop calling `next()` or equivalent), and when the caller calls next again, the coroutine resumes from right after the `co_yield`.

**`co_return`**: Suspend the coroutine and set the final result.

```cpp
co_return 42;
```

Like `return`, but for coroutines. This is where the coroutine ends.

### The Promise Type: Customization Point

Every C++20 coroutine is associated with a **promise type**. The promise is the object that controls the coroutine's behavior: how it suspends, how it resumes, what it returns, how exceptions are handled.

For a coroutine returning `Task<T>`, the promise type is typically `Task<T>::promise_type`. The compiler calls specific methods on the promise:

```cpp
template <typename T>
class Task {
public:
    struct promise_type {
        // Called when the coroutine starts
        void* operator new(size_t size) { return malloc(size); }
        void operator delete(void* ptr) { free(ptr); }
        
        Task get_return_object() {
            // Called to construct the Task returned to the caller
            return Task(std::coroutine_handle<promise_type>::from_promise(*this));
        }
        
        std::suspend_never initial_suspend() {
            // std::suspend_never: don't suspend before running
            return {};
        }
        
        std::suspend_always final_suspend() noexcept {
            // std::suspend_always: suspend when the coroutine ends
            return {};
        }
        
        void unhandled_exception() {
            // Called if an exception escapes the coroutine
            exception = std::current_exception();
        }
        
        void return_value(T value) {
            // Called when co_return value is executed
            result = value;
        }
        
        std::suspend_always yield_value(T value) {
            // Called when co_yield value is executed (generators only)
            result = value;
            return {};
        }
        
        T result;
        std::exception_ptr exception;
    };
    
    std::coroutine_handle<promise_type> handle;
};
```

This is dense, but the key insight is: **every aspect of coroutine behavior is customizable via the promise type.**

### A Tiny C++20 Generator

Here is a complete, compilable example of a coroutine that yields integers (a generator):

```cpp
#include <coroutine>
#include <iostream>
#include <optional>

template <typename T>
class Generator {
public:
    struct promise_type {
        T current;
        bool done = false;
        
        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        
        std::suspend_always yield_value(T value) {
            current = value;
            return {};
        }
        
        void return_void() {
            done = true;
        }
        
        void unhandled_exception() {}
    };
    
    std::coroutine_handle<promise_type> handle;
    
    Generator(std::coroutine_handle<promise_type> h) : handle(h) {}
    
    ~Generator() {
        if (handle) handle.destroy();
    }
    
    // Move-only
    Generator(const Generator&) = delete;
    Generator(Generator&& other) noexcept : handle(other.handle) {
        other.handle = nullptr;
    }
    
    bool next() {
        if (!handle || handle.done()) return false;
        handle.resume();
        return !handle.done();
    }
    
    T value() { return handle.promise().current; }
};

// The coroutine itself
Generator<int> count_to_three() {
    co_yield 1;
    co_yield 2;
    co_yield 3;
}

int main() {
    auto gen = count_to_three();
    
    while (gen.next()) {
        std::cout << gen.value() << "\n";
    }
    
    return 0;
}

// Output:
// 1
// 2
// 3
```

**Walking through it:**

1. `count_to_three()` is a coroutine (it contains `co_yield`).
2. When called, it doesn't run immediately. Instead, the compiler generates a state machine (the promise) on the heap and returns a `Generator` handle.
3. `initial_suspend()` returns `suspend_never`, so the coroutine starts running right away (before the first `co_yield`).
4. The first `co_yield 1` calls `promise.yield_value(1)`, sets `current = 1`, and suspends.
5. `gen.next()` calls `handle.resume()`, which jumps back into the coroutine at the first `co_yield`, resumes execution, and hits the second `co_yield 2`.
6. This repeats until the end of the function, where `promise.return_void()` is called.

### An Async Task

Here is a sketch of an async task (like `Task<T>` in Chapter 49):

```cpp
#include <coroutine>
#include <optional>
#include <functional>

template <typename T>
class AsyncTask {
public:
    struct promise_type {
        std::optional<T> result;
        std::function<void()> continuation;  // What to do when resuming
        
        AsyncTask get_return_object() {
            return AsyncTask{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        
        std::suspend_always initial_suspend() {
            // Don't run immediately; let the caller decide when to resume
            return {};
        }
        
        std::suspend_always final_suspend() noexcept {
            return {};
        }
        
        void return_value(T value) {
            result = value;
        }
        
        void unhandled_exception() {
            // Simplified; real code would store the exception
        }
        
        auto await_transform(AsyncTask& other) {
            // This is called when we co_await another AsyncTask
            // Simplified version; a real one would chain continuations
            struct Awaiter {
                AsyncTask& task;
                
                bool await_ready() const {
                    return task.handle.done();  // Is the task done?
                }
                
                void await_suspend(std::coroutine_handle<>) {
                    // Set up chaining: when 'task' completes, resume us
                }
                
                T await_resume() {
                    return *task.handle.promise().result;
                }
            };
            return Awaiter{other};
        }
    };
    
    std::coroutine_handle<promise_type> handle;
    
    AsyncTask(std::coroutine_handle<promise_type> h) : handle(h) {}
    
    ~AsyncTask() {
        if (handle) handle.destroy();
    }
    
    bool is_ready() const {
        return handle.done();
    }
    
    T get() {
        return *handle.promise().result;
    }
};

// Example coroutine
AsyncTask<int> fetch_number() {
    co_return 42;
}

// Nested coroutine: wait for another
AsyncTask<int> combine() {
    int x = co_await fetch_number();
    int y = co_await fetch_number();
    co_return x + y;
}
```

The key point: `co_await` integrates with the promise type's `await_transform`. The promise decides what happens when you `await` something else.

---

## 53.4 How Other Languages Encode Coroutines

### JavaScript: async/await (Stackless, Event-Driven)

In JavaScript, every `async` function returns a **Promise**. The runtime (the event loop) manages a queue of ready Promises and a set of pending I/O operations (via the OS, Node.js libuv, or the browser's I/O system).

```javascript
async function read_both_files() {
    const a = await fetch("a.txt");
    const b = await fetch("b.txt");
    return a + b;
}
```

The JavaScript engine compiles this to:
- A state machine with three states (start, after first fetch, after second fetch).
- Each `await` is a suspension point.
- The Promise machinery (`.then()` chaining internally) is the mechanism.

The event loop is invisible to the programmer, but it is always there, driven by the JavaScript engine.

**Cost:** Each `async` function allocates a Promise object on the heap. Thousands are fine.

### Python: async/await (Stackless with Legacy Generator Syntax)

Python adopted JavaScript-style `async`/`await` in Python 3.5. Earlier, Python used **generators** (via `yield`) for coroutines, which were then composed via `yield from`.

Modern Python:

```python
async def read_both_files():
    a = await read_file("a.txt")
    b = await read_file("b.txt")
    return a + b
```

This compiles to a state machine, similar to JavaScript. The event loop (via `asyncio.run()` or another async runtime like `trio` or `anyio`) drives the coroutines.

**Legacy Python (generators as coroutines):**

```python
@coroutine
def read_both_files():
    a = yield from read_file("a.txt")
    b = yield from read_file("b.txt")
    return a + b
```

Here, `yield` is overloaded: normally it produces a value, but in a generator-based coroutine, it means "suspend and await the result of this other generator."

**Cost:** Coroutines are generator objects on the heap. Python's `asyncio` is single-threaded; all coroutines share one event loop per process (or thread, if you use multiple threads).

### Rust: async/await (Stackless with Traits)

Rust does not have a built-in async runtime. Instead, C++20-style, `async` is a **compiler transformation**. An `async fn` in Rust returns a `Future` (equivalent to a Promise or coroutine handle).

```rust
async fn read_both_files() -> String {
    let a = read_file("a.txt").await;
    let b = read_file("b.txt").await;
    format!("{}{}", a, b)
}
```

Compiles to:

```rust
fn read_both_files() -> impl Future<Output = String> {
    async move {
        // State machine
    }
}
```

A `Future` is a trait:

```rust
pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

The event loop calls `poll()` on Futures until they return `Poll::Ready`. This is identical to JavaScript Promises and C++ coroutine handles conceptually, but it is a library abstraction, not a language feature.

**Cost:** Futures are heap-allocated state machines (via `Box<dyn Future>` or monomorphized inline).

**Flexibility:** Because Futures are a trait, you can implement custom Future types. This is both a blessing (zero-cost abstractions) and a curse (complex generic debugging).

---

## 53.5 Why Suspension Points Matter for Locals

This is the most common source of bugs in coroutines: **a local variable that crosses a suspension point lives on the heap, not the stack.**

In a normal function:

```cpp
void normal_function() {
    int x = 42;
    do_something(&x);  // Pass a stack-local pointer
    // x is destroyed when the function returns
}
```

The pointer to `x` is valid only while the function is running.

In a coroutine:

```cpp
AsyncTask<int> async_function() {
    int x = 42;
    int y = co_await some_operation(&x);  // DANGER: x is in the coroutine state!
    return y;
}
```

Here is what the compiler does:

1. Recognizes that `x` crosses the `co_await` point (it is live after the await).
2. Moves `x` into the coroutine state object (on the heap).
3. The `some_operation` function receives a pointer to the heap-allocated `x`, not the stack.

This is usually fine, but there are gotchas:

**Gotcha 1: Dangling pointers in callbacks.**

```cpp
AsyncTask<void> bad_coroutine() {
    int local_var = 10;
    co_await register_callback(&local_var);  // Callback receives heap addr
    // After await, local_var is still alive in the state
}
```

If the callback is registered to run *after* the coroutine finishes, you have a use-after-free.

**Gotcha 2: References escaping the coroutine.**

```cpp
AsyncTask<void> bad_coroutine() {
    int stack_var = 10;
    int& ref = stack_var;
    co_await something();
    // ref now points into the coroutine state (not the stack), but the original
    // reference was supposed to track a stack variable. Confusing!
}
```

The reference is not invalidated (the coroutine state persists), but the semantic expectation is broken.

**Rule of thumb:** Assume all locals in a coroutine are on the heap after a suspension point. Do not pass pointers to coroutine locals to callbacks that outlive the await. Use value semantics instead.

---

## 53.6 Generators vs Async Coroutines

The mechanism is identical, but the consumer is different.

### Generators (Pull-based)

A **generator** suspends via `co_yield` and returns a value to the caller. The caller is responsible for pulling values:

```cpp
Generator<int> integers() {
    for (int i = 1; i <= 3; i++) {
        co_yield i;
    }
}

int main() {
    auto gen = integers();
    while (gen.next()) {
        std::cout << gen.value() << "\n";
    }
}
```

The generator is **pull-based**: the caller decides when to fetch the next value.

**Use cases:** Lazily iterating a large sequence, implementing range-based algorithms, tree traversal.

### Async Coroutines (Push-based)

An **async coroutine** suspends via `co_await` and waits for external work (I/O, timers, etc.). The **runtime** is responsible for pushing the coroutine forward when the work completes:

```cpp
AsyncTask<int> fetch_from_network() {
    auto response = co_await http_get("example.com");
    co_return response.status_code;
}

int main() {
    auto task = fetch_from_network();
    // task is running on the event loop automatically
    // (or we explicitly call run(task))
}
```

The coroutine is **push-based**: an external event (I/O completion) causes progress.

**Conceptual difference:** Generators are passive (data structures you iterate). Async coroutines are active (scheduled tasks running in a runtime).

**Implementation:** The difference is in the promise type. Generators use `co_yield` and return values lazily. Async tasks use `co_await` and integrate with a runtime's event loop.

---

## 53.7 Worked Example: A Minimal C++20 Generator and Task

Here is a complete, runnable example demonstrating both a generator and a simple async task:

```cpp
#include <coroutine>
#include <iostream>
#include <optional>
#include <queue>
#include <functional>

// ============================================================================
// GENERATOR
// ============================================================================

template <typename T>
class Generator {
public:
    struct promise_type {
        T current_value;
        
        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        
        void return_void() {}
        void unhandled_exception() {}
    };
    
    std::coroutine_handle<promise_type> handle;
    
    Generator(std::coroutine_handle<promise_type> h) : handle(h) {}
    
    ~Generator() {
        if (handle) handle.destroy();
    }
    
    Generator(const Generator&) = delete;
    Generator(Generator&& other) noexcept : handle(other.release()) {}
    
    bool next() {
        if (!handle) return false;
        handle.resume();
        return !handle.done();
    }
    
    T value() const { return handle.promise().current_value; }
    
private:
    std::coroutine_handle<promise_type> release() {
        auto tmp = handle;
        handle = nullptr;
        return tmp;
    }
};

// ============================================================================
// ASYNC TASK (simplified, single-threaded event loop)
// ============================================================================

class SimpleEventLoop;

template <typename T>
class Task {
public:
    struct promise_type {
        std::optional<T> result_value;
        std::function<void()> continuation;
        
        Task get_return_object() {
            return Task{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        
        std::suspend_always initial_suspend() { return {}; }
        
        std::suspend_always final_suspend() noexcept {
            return {};
        }
        
        void return_value(T value) {
            result_value = value;
        }
        
        void unhandled_exception() {}
        
        // Simple await_transform for other Tasks
        auto await_transform(Task<T>& other) {
            struct TaskAwaiter {
                Task<T>& task;
                
                bool await_ready() {
                    return task.handle.done();
                }
                
                void await_suspend(std::coroutine_handle<>) {
                    // In a real implementation, we'd chain continuations
                    // For now, assume the awaited task is already done
                }
                
                T await_resume() {
                    return task.get_result();
                }
            };
            return TaskAwaiter{other};
        }
    };
    
    std::coroutine_handle<promise_type> handle;
    
    Task(std::coroutine_handle<promise_type> h) : handle(h) {}
    
    ~Task() {
        if (handle) handle.destroy();
    }
    
    Task(const Task&) = delete;
    Task(Task&& other) noexcept : handle(other.release()) {}
    
    bool is_done() const {
        return handle.done();
    }
    
    T get_result() const {
        return *handle.promise().result_value;
    }
    
private:
    std::coroutine_handle<promise_type> release() {
        auto tmp = handle;
        handle = nullptr;
        return tmp;
    }
};

// ============================================================================
// COROUTINES
// ============================================================================

// Generator: yields integers
Generator<int> count_to_three() {
    std::cout << "[count_to_three] Starting\n";
    co_yield 1;
    std::cout << "[count_to_three] After yield 1\n";
    co_yield 2;
    std::cout << "[count_to_three] After yield 2\n";
    co_yield 3;
    std::cout << "[count_to_three] After yield 3\n";
}

// Async task: simple computation
Task<int> simple_task() {
    std::cout << "[simple_task] Starting\n";
    co_return 42;
}

// ============================================================================
// MAIN
// ============================================================================

int main() {
    std::cout << "=== GENERATOR EXAMPLE ===\n";
    auto gen = count_to_three();
    
    while (gen.next()) {
        std::cout << "Got: " << gen.value() << "\n";
    }
    
    std::cout << "\n=== ASYNC TASK EXAMPLE ===\n";
    auto task = simple_task();
    
    // Simulate running the task (in a real runtime, the event loop does this)
    while (!task.is_done()) {
        task.handle.resume();
    }
    
    std::cout << "Task result: " << task.get_result() << "\n";
    
    return 0;
}

// Output:
// === GENERATOR EXAMPLE ===
// [count_to_three] Starting
// Got: 1
// [count_to_three] After yield 1
// Got: 2
// [count_to_three] After yield 2
// Got: 3
// [count_to_three] After yield 3
//
// === ASYNC TASK EXAMPLE ===
// [simple_task] Starting
// Task result: 42
```

This example is intentionally simplified (the Task's `await_transform` does not actually chain coroutines; a real one would). But it shows:

1. How a generator yields values and suspends.
2. How an async task suspends and resumes.
3. The promise type as the customization point.

---

## 53.8 Tradeoffs: Stackful vs Stackless vs Threads vs Callbacks

| Model | Memory/Coro | Context Switch | Suspension Depth | Tooling | Best For |
|-------|-------------|-----------------|------------------|---------|----------|
| **OS Thread** | 8 MB | 1–10 µs | Unlimited (stack frame per call) | Debuggers work fully; stack traces clear | Few high-concurrency tasks; CPU-bound; existing sync code |
| **Stackful Coroutine** | 64 KB–1 MB | ~100 ns | Unlimited (owns stack) | Debuggers mostly work; pseudo-stack visible | Wrapping sync libraries; medium concurrency; locality important |
| **Stackless Coroutine** | 100 bytes–10 KB | ~0 ns (just function call) | Limited to "await points" | Debuggers need async frame support; state machine implicit | Massive concurrency; I/O-bound; new code designed for async |
| **Callback/Promise Chain** | 100 bytes–1 KB | ~0 ns | Limited to explicit chaining | Nightmare to debug ("pyramid of doom"); no stack context | Legacy systems; specific libraries; simple tasks |

**Practical guidance:**

- **Use threads** if you have a small number of long-running tasks (< 1000), or if you need true parallelism on multiple cores for CPU-bound work.
- **Use stackless coroutines** if you need massive concurrency (>10,000 tasks) and you are doing primarily I/O.
- **Use stackful coroutines** if you are integrating with existing synchronous libraries (database drivers, legacy code) and need to suspend from deep within them.
- **Avoid bare callbacks** if at all possible. Use a coroutine/Future abstraction instead.

---

## 53.9 Common Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Coroutines are syntactic sugar for callbacks." | Coroutines preserve the call stack's structure: locals, scoping, exception handling all work. Callbacks shatter this. |
| "Stackless coroutines can suspend anywhere." | Stackless coroutines can only suspend at explicit `co_await` / `await` points. Suspension from deep function calls requires stackful coroutines. |
| "Switching coroutines is free." | For stackless coroutines: yes, it's just a function call. For stackful: ~100 ns. For OS threads: 1–10 µs (plus cache misses). |
| "Coroutines are always faster than threads." | Coroutines are faster for I/O; threads are faster for CPU-bound work. Coroutines require a runtime; threads are OS-native. |
| "You can have billions of coroutines." | You can have millions (limited by heap memory). Billions would require terabytes of RAM. |
| "Debugging a coroutine is like debugging a thread." | Harder. The "call stack" is not contiguous; it is a state machine on the heap. Modern debuggers are catching up but support varies. |
| "The promise type is magic." | It is pure C++ boilerplate: the compiler calls your methods. You define the behavior entirely. |
| "C++20 coroutines are production-ready." | The core language feature is stable, but the ecosystem of libraries and tooling is still maturing (as of 2025). |

---

## 53.10 Exercises

1. **Anatomy of a simple generator.** Write a generator that yields the Fibonacci sequence (1, 1, 2, 3, 5, 8, ...). Trace through the state machine manually: which case in the switch is executed on each `next()` call?

2. **Generator with early termination.** Add a check to your Fibonacci generator: stop if the value exceeds 1000. Verify that calling `next()` after the termination point returns false.

3. **Stackful vs stackless memory.** Write a program that spawns 10,000 coroutines, each holding a 1 KB buffer. Estimate the memory cost:
   - Stackful (assuming 64 KB stack each): 10,000 × 64 KB = 640 MB.
   - Stackless (assuming 1 KB locals + overhead): 10,000 × 2 KB = 20 MB.
   Compare to actual RSS (resident set size) using `ps` or a memory profiler.

4. **Locals crossing await points.** Write a coroutine that uses a local variable before and after an `await`. Compile with debug symbols and inspect the generated code (e.g., `objdump -d` or `llvm-objdump`) to confirm that the local is stored in the coroutine state, not the stack.

5. **Promise type customization.** Modify the Task example so that it tracks how many times `resume()` was called. Add a `resume_count()` method and verify that a task with three `co_await` points is resumed exactly three times.

6. **Generator-based range.** Implement a generator that produces a range of numbers (like Python's `range(start, stop, step)`). Use it in a range-based for loop (via a helper adapter that implements `begin()` and `end()`).

7. **Gotcha: dangling references.** Write a coroutine that takes a reference parameter, awaits something, and then uses the reference. Pass a stack-local variable via the reference. Show (via asan or valgrind) that the reference is safe because it now points into the coroutine state, not the original stack.

---

## 53.11 Summary

**A coroutine is a suspendable function.** It captures its state when suspended and resumes from exactly that point later.

**Two implementation styles:** Stackful coroutines have their own heap-allocated stack (familiar, but memory-heavy). Stackless coroutines store state in a heap object and share the OS thread's stack (compact, scales to millions).

**C++20 coroutines are stackless.** They compile to a state machine controlled by a promise type. The three keywords — `co_await`, `co_yield`, `co_return` — are compiler transformations, not tied to any runtime.

**Locals that cross suspension points live on the heap,** not the stack. This is usually transparent but is the source of the most common coroutine bugs (passing pointers to coroutine-local variables to callbacks that outlive the await).

**Generators pull values lazily; async coroutines are pushed by a runtime.** Both use the same mechanism; the difference is in the promise type and the consumer.

**Coroutines are not threads.** They are far cheaper (memory and context-switch time) but require an explicit runtime (or event loop) to drive them. A single OS thread can host thousands of coroutines; the tradeoff is that CPU-bound work starves the loop.

---

> **[← Previous: Event Loops](07-event-loops.md)** · **[↑ Part 5](README.md)** · **[Next: How Python AsyncIO Works →](09-how-python-asyncio-works.md)**
