# Chapter 15 — RAII Explained Deeply

RAII has the worst name in computing. "Resource Acquisition Is Initialization" is technically true but obscures the actual idea, which is simpler and more powerful: **tie resource lifetime to object lifetime so the destructor cleans up automatically**. That is the entire concept. The compiler's rules about stack discipline and exception unwinding make it work.

This chapter deconstructs RAII from first principles. You will see that it is not a C++ feature — it is a fundamental pattern that Rust calls `Drop`, that Go approximates with `defer`, that Python formalizes with `with`, and that languages without it (Java before try-with-resources, C#'s `using`) have to fake with special syntax. C++ made it *automatic* because of how C++ handles destructors and the stack.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define RAII precisely: constructor acquires, destructor releases, and stack discipline guarantees cleanup on scope exit.
2. Explain why destructors run during exception unwinding and why this is what makes RAII robust against errors.
3. Build a custom RAII wrapper around any resource (files, locks, memory, sockets) and prove it cleans up even when exceptions occur.
4. Distinguish between the Rule of Zero (let the compiler generate everything), the Rule of Three (write custom copy/move), and the Rule of Five (write all five).
5. Compare RAII to `try/finally`, `defer`, and `with` statements in other languages and understand why RAII is integrated into the type system rather than added as syntax.
6. Recognize common misconceptions: that RAII only manages memory, that it requires writing destructors, that it has a performance cost.

---

## The Core Idea

Here is RAII without the jargon:

1. **Resource Acquisition in Constructor:** When you construct an object, it acquires a resource — a file handle, a mutex lock, allocated memory, a socket, a database connection.
2. **Automatic Cleanup in Destructor:** When the object goes out of scope, the destructor runs and releases the resource.
3. **Exception Safety:** Because destructors run during stack unwinding, the resource is released *even if an exception is thrown*.

That is all. But that simple idea eliminates entire categories of bugs: resource leaks, use-after-free, missing lock releases, forgotten frees.

The mechanism relies on three facts about C++:

- **Stack objects are destroyed when they leave scope.** The moment a variable goes out of scope, its destructor is called. This is not a suggestion; it is guaranteed by the language. Every local object, every temporary, every member of a class being destroyed — all have their destructors invoked.
- **Destructors run during exception unwinding.** When an exception propagates through a frame, the compiler runs destructors of all local objects in reverse order of construction before popping that frame off the stack. If the standard library, the OS, or an application-defined exception handler catches the exception, it may continue execution; but regardless, the current frame's resources are always cleaned.
- **The compiler generates destructors for you.** If you don't write a destructor, the compiler generates one that calls the destructor of each member variable in reverse order of declaration. This is true even if you do not explicitly request it.

Combine these three facts and you get automatic resource cleanup for free, without any special syntax.

This is powerful because *you cannot accidentally forget the cleanup*. The compiler enforces it. In languages without guaranteed destructor semantics (or without automatic destruction on scope exit), cleanup becomes a matter of discipline — you must remember to call the cleanup function. Forgetting is a silent bug that may not manifest for months. In C++, forgetting is impossible; the compiler simply does not allow it.

---

## Three Canonical Examples

### Example 1: File Handle

Without RAII, managing a file is error-prone:

```cpp
// BAD: manual cleanup, easy to leak
FILE* f = fopen("data.txt", "r");
if (!f) return;
// ... do work ...
if (some_error) {
    printf("error!\n");
    return;  // oops, forgot to fclose; leak
}
// ... more work ...
fclose(f);
```

The problem: if we return or throw early, we forget to close. The fix is RAII:

```cpp
// GOOD: automatic cleanup via RAII
class ScopedFile {
private:
    FILE* handle_;
public:
    explicit ScopedFile(const char* path, const char* mode) {
        handle_ = fopen(path, mode);
        if (!handle_) throw std::runtime_error("fopen failed");
    }
    
    ~ScopedFile() {
        if (handle_) fclose(handle_);
    }
    
    // delete copy; file is unique
    ScopedFile(const ScopedFile&) = delete;
    ScopedFile& operator=(const ScopedFile&) = delete;
    
    FILE* get() const { return handle_; }
};

void read_file() {
    ScopedFile f("data.txt", "r");
    
    if (some_error) {
        printf("error!\n");
        return;  // destructor ~ScopedFile() runs automatically; fclose called
    }
    // ... work ...
}  // end of scope: ~ScopedFile() called; fclose guaranteed
```

Even if an exception is thrown inside `read_file()`, the `ScopedFile` destructor runs, the file is closed, and no leak occurs.

### Example 2: Mutex Lock

A mutex without RAII requires manual lock/unlock, which is easy to get wrong:

```cpp
// BAD: easy to forget unlock or unlock in wrong order
std::mutex mu;

void critical_section() {
    mu.lock();
    // ... do work ...
    if (error) {
        printf("oops\n");
        return;  // forgot to unlock; deadlock or priority inversion
    }
    // ... more work ...
    mu.unlock();
}
```

RAII solves it:

```cpp
std::mutex mu;

void critical_section() {
    std::lock_guard<std::mutex> lock(mu);  // constructor calls mu.lock()
    // ... do work ...
    if (error) {
        printf("oops\n");
        return;  // destructor ~lock_guard() calls mu.unlock()
    }
    // ... more work ...
}  // destructor runs; mu.unlock() guaranteed
```

`std::lock_guard<T>` is an RAII wrapper around a mutex. Constructor calls `mu.lock()`, destructor calls `mu.unlock()`. No way to forget; no way to deadlock by returning early or throwing; no way to double-lock.

### Example 3: Memory Allocation

The oldest and simplest RAII: smart pointers.

```cpp
// BAD: manual new/delete, easy to leak
void process() {
    int* data = new int[1000];
    if (compute(data) < 0) {
        printf("failed\n");
        return;  // leak; never delete
    }
    delete[] data;
}

// GOOD: automatic cleanup via unique_ptr
void process() {
    std::unique_ptr<int[]> data(new int[1000]);
    if (compute(data.get()) < 0) {
        printf("failed\n");
        return;  // destructor ~unique_ptr() calls delete[]; no leak
    }
}  // destructor runs; delete[] guaranteed
```

`std::unique_ptr<T>` wraps a pointer. Constructor stores it, destructor calls `delete`. That is RAII applied to memory.

---

## Stack Unwinding and Exception Safety

The magic of RAII is exception unwinding. When an exception is thrown, the C++ runtime walks back through the call stack, running destructors of all local objects as it goes.

### How Unwinding Works

The mechanism is specified in the C++ standard and enforced by the runtime:

1. **Exception thrown.** A `throw` statement constructs an exception object on a special exception-handling heap (distinct from the main heap). It then initiates stack unwinding.

2. **Unwind tables consulted.** The compiler generates unwind information — tables that map every instruction address to the set of live local objects at that point. On Linux/macOS (ELF/Mach-O), this is in `.eh_frame` (Exception Handling Frame). On Windows, it is in `.pdata` and `.xdata`. This information is not optional; it is generated unless exceptions are disabled.

3. **Frame by frame:** For each stack frame, the unwinder (part of the runtime library, `libunwind` on Linux, DWARF-based on macOS) identifies all local variables and temporary objects in that frame by consulting the unwind table.

4. **Destructors in reverse order:** The destructors are called in *reverse order of construction*. If a function constructed `A`, then `B`, then `C`, the destructors run as `~C()`, `~B()`, `~A()`. This order is critical: if `C`'s destructor calls something that touches `B`'s state, `B` is still valid. This LIFO (last-in-first-out) order mirrors construction.

5. **Frame popped:** After all destructors of a frame run, the frame is removed from the stack, and the next frame up is processed. The `rsp` (stack pointer) and `rbp` (frame pointer) are restored to their state before the frame was entered.

6. **Catch block found:** If a `catch` handler matches the exception type, unwinding stops at that frame. The handler is executed, and normal execution resumes. If no handler matches, unwinding continues up the call stack. If no handler is found at all, the program terminates (typically by calling `std::terminate()`).

### Example: Multiple Objects Unwinding

```cpp
class Resource {
public:
    Resource(const char* name) : name_(name) {
        printf("[%s] acquired\n", name);
    }
    ~Resource() {
        printf("[%s] released\n", name);
    }
private:
    const char* name_;
};

void deep_call() {
    Resource r1("File");
    Resource r2("Lock");
    Resource r3("Memory");
    
    printf("Throwing...\n");
    throw std::runtime_error("oops");
}

int main() {
    try {
        deep_call();
    } catch (const std::exception& e) {
        printf("Caught: %s\n", e.what());
    }
    printf("Done\n");
}

// Output:
// [File] acquired
// [Lock] acquired
// [Memory] acquired
// Throwing...
// [Memory] released     <- ~r3, reverse order
// [Lock] released       <- ~r2
// [File] released       <- ~r1
// Caught: oops
// Done
```

Observe the order: objects are released in *reverse* order of construction. `r3` (Memory) was constructed last, so it is destroyed first. This is guaranteed. If code runs in a destructor and touches another object, that object is still valid because it was constructed before.

### Why This Guarantee Matters

This guarantee exists because C++17 specifies that exception handling must run destructors. (Pre-standard C++ had less robust guarantees, which is why you still see legacy code with manual try/finally workarounds.) The guarantee is backed by the runtime, the compiler's code generation, and the unwind tables. 

If a language does not guarantee destructors run during unwinding (some old C, some embedded systems without exception support), RAII breaks. The resource may not be cleaned up. The C++ standard's decision to guarantee this is the reason RAII is reliable — it is not a best-effort heuristic, it is a contract enforced by the language.

---

## RAII vs `try/finally` vs `defer` vs `with`

RAII is not unique to C++. Other languages solve the same problem differently:

### Java / C# / Python `try/finally`

Java requires manual cleanup with `finally`:

```java
// Java
FileReader f = new FileReader("data.txt");
try {
    // ... work ...
} finally {
    f.close();  // guaranteed to run
}
```

The `finally` block is always entered before exiting the `try`, whether normally or via exception. But this is **syntax** — the language enforces it, not the type system. You still have to *remember* to write the `finally` block.

### Go's `defer`

Go adds `defer` as a syntax for "run this before exiting the function":

```go
// Go
f, _ := os.Open("data.txt")
defer f.Close()  // guaranteed to run before return

// ... work ...
// defer runs here, before function returns
```

`defer` statements are queued, and all queued `defer`s run in LIFO order before the function returns (or panics). Like `finally`, it requires syntax. But unlike RAII, `defer` is explicit at every function level — you must *see* the `defer` statement to know cleanup will happen.

### Python's `with` statement

Python formalizes cleanup via context managers:

```python
# Python
with open("data.txt") as f:
    # ... work ...
    # __exit__ guaranteed to run here
```

Objects used with `with` must implement `__enter__` (constructor) and `__exit__` (cleanup). Again, syntax is required — you must use `with` to get cleanup.

### Why RAII is different

RAII integrates cleanup into the **type system**, not syntax. When you declare a `ScopedFile f`, cleanup is *guaranteed by the type*. You cannot forget; the compiler enforces it. No `try/finally`, no `defer`, no `with` required. Just declare the object and use it; the destructor runs automatically.

This is why Rust made `Drop` (Rust's RAII) a core language feature: because tying cleanup to types, not syntax, scales better. It is also why languages without guaranteed destructor semantics must resort to syntax — the type system cannot guarantee cleanup if destructors are not guaranteed to run.

---

## The Rule of Zero, Three, Five

When you write a class, you decide: do I need to write custom copy, move, and/or destructor code?

### Rule of Zero

If your class does not manage any resources directly, let the compiler generate everything:

```cpp
struct Point {
    float x, y;
};

// Default-generated:
// Point(const Point&);  // memberwise copy
// Point(Point&&);       // memberwise move
// ~Point();             // destroys members (trivial for built-ins)
// Point& operator=(const Point&);
// Point& operator=(Point&&);

Point p1{1.0f, 2.0f};
Point p2 = p1;  // copy; fine, just copies floats
Point p3 = std::move(p1);  // move; fine, just moves floats
```

No manual code needed. The compiler-generated code does the right thing.

### Rule of Three

If your class manually acquires a resource in the constructor (like `new`), you must write:

1. **Destructor** — release the resource.
2. **Copy constructor** — decide if copying is allowed and what it means.
3. **Copy assignment operator** — handle assignment.

```cpp
class Buffer {
private:
    int* data_;
    size_t size_;
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {}
    
    // (1) Destructor
    ~Buffer() { delete[] data_; }
    
    // (2) Copy constructor
    Buffer(const Buffer& other) 
        : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + other.size_, data_);
    }
    
    // (3) Copy assignment
    Buffer& operator=(const Buffer& other) {
        if (this == &other) return *this;
        delete[] data_;
        data_ = new int[other.size_];
        size_ = other.size_;
        std::copy(other.data_, other.data_ + other.size_, data_);
        return *this;
    }
};
```

If you write only the destructor and not the copy constructor/assignment, the compiler generates those for you, and they do memberwise copy — which is wrong for pointers. Copying a `Buffer` would copy the `data_` pointer, leaving two `Buffer` objects pointing to the same array. When the first is destroyed, it `delete[]`s the array; the second now has a dangling pointer. Use-after-free. This is a critical bug:

```cpp
void process() {
    Buffer b1(10);
    {
        Buffer b2 = b1;  // compiler-generated copy: bad!
        // b2.data_ = b1.data_ (pointer copy only)
    }  // b2's destructor runs: delete[] b2.data_
       // but b2.data_ == b1.data_!
    // b1.data_ is now dangling
    b1[0] = 5;  // use-after-free; undefined behavior
}
```

The rule of three enforces: if you care enough about resource management to write a destructor, care enough to define copy semantics explicitly. Either allow copy (and write the copy operations to duplicate the resource), or forbid it (`= delete`).

### Rule of Five

Modern C++ (C++11+) adds move semantics. If you write copy, you may also need to write move:

1. **Destructor**
2. **Copy constructor**
3. **Copy assignment**
4. **Move constructor**
5. **Move assignment**

```cpp
class Buffer {
private:
    int* data_;
    size_t size_;
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {}
    ~Buffer() { delete[] data_; }
    
    Buffer(const Buffer& other);
    Buffer& operator=(const Buffer& other);
    
    // (4) Move constructor: take ownership from temporary
    Buffer(Buffer&& other) noexcept 
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // (5) Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this == &other) return *this;
        delete[] data_;
        data_ = other.data_;
        size_ = other.size_;
        other.data_ = nullptr;
        other.size_ = 0;
        return *this;
    }
};

Buffer create_buffer() {
    Buffer b(100);
    // ... fill ...
    return b;  // move constructor: avoids copying the array
}

void use() {
    Buffer b = create_buffer();  // move, not copy
}
```

Move constructors and move assignments allow efficient transfer of ownership without copying. When you return a local object or pass an rvalue, the compiler uses the move constructor, transferring the allocation instead of copying it. This is why modern C++ can return large objects efficiently.

Or, **Rule of Zero with `unique_ptr`:**

If you use `unique_ptr` to manage the resource, the compiler-generated copy is deleted and move is generated correctly:

```cpp
class Buffer {
private:
    std::unique_ptr<int[]> data_;
    size_t size_;
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {}
    
    // No need to write anything; compiler generates:
    // ~Buffer() : calls ~data_
    // Buffer(Buffer&& other) = default;  // move ok
    // copy is deleted automatically (unique_ptr forbids it)
};
```

This is the modern C++ way: **use smart pointers and let the compiler generate everything**. If you find yourself writing the rule of five, ask: could I use `unique_ptr` or `shared_ptr` instead?

---

## Worked Example: Building `ScopedFile` from Scratch

Let us build a complete RAII file wrapper from first principles and verify that exceptions do not leak it. This example is deliberately detailed to show every decision:

### The Implementation

```cpp
#include <cstdio>
#include <stdexcept>
#include <string>

class ScopedFile {
private:
    FILE* handle_;
    
    // Delete copy operations: a file is unique
    ScopedFile(const ScopedFile&) = delete;
    ScopedFile& operator=(const ScopedFile&) = delete;
    
public:
    // Constructor: acquisition
    explicit ScopedFile(const std::string& path, const char* mode) {
        handle_ = std::fopen(path.c_str(), mode);
        if (!handle_) {
            throw std::runtime_error(
                std::string("fopen failed: ") + path
            );
        }
    }
    
    // Destructor: release
    ~ScopedFile() {
        if (handle_) {
            std::fclose(handle_);
            handle_ = nullptr;
        }
    }
    
    // Move semantics
    ScopedFile(ScopedFile&& other) noexcept 
        : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
    
    ScopedFile& operator=(ScopedFile&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = other.handle_;
            other.handle_ = nullptr;
        }
        return *this;
    }
    
    FILE* get() const { return handle_; }
    bool is_open() const { return handle_ != nullptr; }
};
```

Let us trace through the key design decisions:

- **Constructor acquires:** `fopen` is called; if it fails, an exception is thrown. The object either has a valid handle or it does not exist (exception propagates).
- **Destructor releases:** `fclose` is guaranteed to run when the object leaves scope or is destroyed.
- **No copy:** We delete the copy operations because a file handle is unique. Two `ScopedFile` objects cannot own the same handle; if allowed, the first to be destroyed would close the file, leaving the second with a closed handle.
- **Move is allowed:** Moving transfers ownership — the source object's handle becomes null, the destination gets the handle. This is safe and efficient.

### Using It with Exceptions

Now let us use it in a function that might throw:

```cpp
int count_lines(const std::string& filename) {
    ScopedFile f(filename, "r");  // constructor: file opened
    
    int count = 0;
    char buffer[256];
    while (std::fgets(buffer, sizeof(buffer), f.get())) {
        ++count;
        if (count > 1000) {
            // Simulate an error mid-way
            throw std::runtime_error("file too large");
        }
    }
    
    return count;
    // destructor ~ScopedFile() runs here on normal return
}

int main() {
    try {
        int n = count_lines("bigfile.txt");
        printf("Lines: %d\n", n);
    } catch (const std::exception& e) {
        printf("Error: %s\n", e.what());
        // If exception thrown in count_lines, the destructor still ran
        // The file is closed; no leak
    }
    return 0;
}
```

### Proof of Correctness

Whether `count_lines` returns normally or throws:

1. **Normal return:** at the closing brace of `count_lines`, `f` goes out of scope. The destructor `~ScopedFile()` is invoked automatically. `fclose(handle_)` is called. The file is closed.

2. **Exception at runtime:** suppose `count > 1000` is true, and `throw` executes. The C++ runtime begins stack unwinding. It walks the stack frame of `count_lines`, identifies local variable `f`, and calls `~ScopedFile()`. This happens *before* exiting the frame. The file is closed. Then the frame is popped and unwinding continues toward `main`, where the `catch` block handles the exception.

**No leak in either path.** The destructor is guaranteed to run. No manual try/finally required. No chance of forgetting to close. The compiler enforces it.

This is the power of RAII: the *type* `ScopedFile` guarantees cleanup. You cannot declare a `ScopedFile` and accidentally leak it. The compiler prevents it.

---

## Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| RAII (constructor acquires, destructor releases) | Automatic cleanup; works during exceptions; tightly binds resource to type; no syntax overhead; compiler enforces it | Requires understanding destructors and stack discipline; custom copy/move may be needed; less explicit than defer/finally | Default choice in modern C++ |
| Manual cleanup (explicit free/close/unlock) | Explicit at call site; no hidden behavior; works in any language | Easy to forget; error-prone; control-flow dependent (early returns, exceptions leak resources); silent bugs | Rare; only when RAII is impossible or forbidden (e.g., C90, embedded without exceptions) |
| `try/finally` or `try`/`catch`/`finally` | Explicit cleanup block; straightforward; readable | Easy to forget the block entirely; requires syntax overhead; multiple exits require multiple blocks; not tied to type system | Languages without RAII (Java, C#, old Python); languages where RAII is not integrated |
| `defer` statement | Clean syntax; all cleanup listed at function start; LIFO order | Not per-type; must remember syntax; harder to reason about delayed execution across function boundaries; easy to defer at wrong scope | Go; Rust (via `defer!` macro); languages that added this syntax |
| `with` statement or context managers | Per-variable; explicit; natural syntax; works well for single-resource functions | Requires syntax and custom `__enter__`/`__exit__` boilerplate; nesting can become unreadable; not enforced by type system | Python; some modern languages; good for one-off resource management |
| Smart pointers (`unique_ptr`, `shared_ptr`) | RAII applied to memory; zero overhead; compiler-generated code optimal | Adds pointer indirection (one level of load); mental model overhead for `shared_ptr` (atomic ref counting); requires heap allocation | Default for dynamic memory in modern C++; universal for resource ownership in C++ |

---

## Common Misconceptions

1. **"RAII only manages memory."** False. RAII manages *any* resource: files, locks, sockets, database connections, memory buffers, GPU allocations, thread-local storage, custom protocol state machines. Any resource that must be cleaned up can be wrapped in an RAII class. The name "Resource Acquisition Is Initialization" is deliberately general. Many teams ship RAII wrappers for company-specific resources (e.g., "take a connection from the pool in the constructor, return it in the destructor").

2. **"You have to write destructors."** False (mostly). If you use standard RAII wrappers (`std::unique_ptr`, `std::lock_guard`, `std::fstream`), the compiler generates the destructor for you, or the class provides it. You only write destructors when building your own RAII wrappers. And even then, if you use smart pointers internally, the compiler generates the destructor.

3. **"RAII has a runtime cost."** False. A destructor call is a function call, which the compiler can inline. A well-optimized destructor (like `~unique_ptr` checking if the pointer is null and calling `delete`) compiles to a few instructions. Modern CPUs execute this in a single cycle. The "cost" is comparable to a conditional branch in a normal function. Debugging overhead (reading destructors in a debugger during exception unwinding) is real but not a runtime cost; it is a development-time cost.

4. **"RAII is a workaround for memory management."** Backwards. RAII is a *fundamental pattern* for managing any resource, not just memory. C++ makes it automatic through destructors and stack discipline. Other languages bolt it on with special syntax (`try/finally`, `defer`, `with`). Rust made Drop (Rust's RAII) a core language feature, proving it is not specific to C++. RAII is not a hack; it is the right way to think about resource lifetimes in any language.

5. **"Exception safety is about using RAII."** Incomplete. RAII is *one piece* of exception safety, but not the whole story. You also need to think about: (a) **strong exception guarantee** — if an operation fails, the object's state is unchanged (transaction-like); (b) **basic exception guarantee** — if an operation fails, the object is in a valid state (no leaks, no corruption); (c) **no-throw guarantee** — the operation never throws. RAII provides the infrastructure (automatic cleanup); the rest is careful design (not modifying state until you're sure you won't fail, etc.).

6. **"Move semantics breaks RAII."** False. Move is perfectly compatible with RAII. When you move an object `x` to `y`, ownership of the resource is transferred from `x` to `y`. The moved-from object `x` is left in a valid but unspecified state (often, a "null" state). When `~x()` is called later (because `x` goes out of scope), it is safe — it will see the null state and do nothing. The key rule: **the move constructor must leave the source in a state where `~source()` is safe to call**. If you follow this rule, move and RAII compose seamlessly.

---

## Practical Patterns in Production Code

RAII is not just a theoretical tool. Modern C++ libraries are built on it. Here are patterns you will see:

### Pattern 1: Scoped Guard (Acquire at Construction, Release at Destruction)

The simplest pattern. Acquire the resource in the constructor, release in the destructor. Examples: `std::lock_guard`, `std::unique_ptr`, file objects, connection pool borrowers.

```cpp
{
    std::unique_ptr<Connection> conn = pool.acquire();  // acquire
    conn->execute("SELECT ...");
}  // release on scope exit
```

### Pattern 2: Nested Scopes for Scoped Cleanup

Declare RAII objects in nested scopes to control their lifetime precisely:

```cpp
void process_in_transaction() {
    {
        std::lock_guard<std::mutex> lock(db_mu);
        db.begin_transaction();
        // ... work ...
        db.commit();
    }  // lock released; transaction closed
    // subsequent work can run without lock
}
```

### Pattern 3: Moving for Ownership Transfer

Use move semantics to transfer resource ownership without copying:

```cpp
std::unique_ptr<int[]> create_buffer(size_t n) {
    return std::make_unique<int[]>(n);  // move, not copy
}

auto buf = create_buffer(1000);  // buf now owns the allocation
```

### Pattern 4: Rule of Zero with Smart Pointers

Let the compiler generate everything by using smart pointers:

```cpp
class ImageCache {
private:
    std::unordered_map<std::string, std::shared_ptr<Image>> cache_;
    // No destructor, copy constructor, copy assignment, move constructor, move assignment
    // Compiler generates all five; they call the generated code for cache_
};
```

### Pattern 5: RAII Adapters for C APIs

Wrap old C APIs in RAII for safety:

```cpp
class PQConnection {
    PGconn* conn_;
public:
    PQConnection(const char* connstr) {
        conn_ = PQconnectdb(connstr);
        if (!conn_) throw std::runtime_error("PQ connect failed");
    }
    ~PQConnection() { if (conn_) PQfinish(conn_); }
    // ... methods ...
};
```

---

## Exercises

1. **Build an RAII wrapper for `malloc`/`free`.** Create a `MallocBuffer<T>` class that allocates a block of `T` via `malloc` in the constructor and frees it in the destructor. Include move constructor/move assignment. Write a function that allocates a buffer, fills it, throws if a condition fails mid-way, and verify the allocation is freed without leak. (Hint: use a custom allocator wrapper to track allocations, or log every malloc/free and count them.)

2. **Verify exception safety.** Write a function that creates three RAII objects (file, lock, memory) in sequence and throws at various points: (a) before creating any, (b) between the first and second, (c) between the second and third, (d) after all three. Use a test harness that counts allocated resources before and after each throw. Verify that no resources leak at any throw point and that destructors run in the correct (reverse) order.

3. **Distinguish Rule of Zero from Rule of Five.** Write two classes: (a) `Vector<T>` that manages a dynamic array (write rule-of-five: destructor, copy constructor, copy assignment, move constructor, move assignment); (b) `Counter` that wraps a `std::atomic<int>` (let the compiler generate everything). For each, explain why compiler-generated code is correct or incorrect.

4. **Compare RAII to `defer` (in Go or Rust).** Take the `ScopedFile` example from this chapter and rewrite it in Go using `defer` or in Rust using `Drop` and `impl Drop`. Compare: (a) which is clearer to read? (b) which is easier to accidentally misuse? (c) when would you use one over the other? (d) what happens if you forget the defer/Drop in each language?

5. **Measure destructor cost.** Write a tight loop that creates and destroys 1,000,000 RAII objects (e.g., `std::lock_guard<std::mutex>` with a dummy mutex). Benchmark with optimizations on (`-O2`) and off (`-O0`). Measure the wall-clock time. Then write a no-op class that does nothing and benchmark that. Is the difference measurable? What does the assembly show? Why is the real destructor's cost so small?

6. **Conceptual: the lifetime problem.** A coworker writes code like this:
   ```cpp
   Resource* r = new Resource();
   use(r);
   delete r;  // manual cleanup
   ```
   They ask: "Why should I use RAII instead?" Write a detailed explanation connecting to this chapter: (a) the specific bugs this code is vulnerable to (early return, exception, forgotten delete), (b) how RAII prevents each, (c) why RAII is automatic and does not require remembering syntax, (d) why RAII is enforceable by the compiler.

---

## Summary

RAII is the tying of resource lifetime to object lifetime. The destructor—guaranteed to run when the object leaves scope, even during exception unwinding—releases the resource. This simple idea eliminates resource leaks, dangling pointers, and forgotten unlocks without any special syntax or language runtime. Modern C++ leverages this through smart pointers and standard library wrappers, but the pattern is universal: any language that wants robust resource management must tie cleanup to types, not syntax. Understanding RAII is understanding how safe systems are built.

---

**[← Previous: Chapter 14 — Lifetimes](04-lifetimes.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 16 — Memory Fragmentation →](06-memory-fragmentation.md)**
