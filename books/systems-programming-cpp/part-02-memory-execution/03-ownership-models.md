# Chapter 13 — Ownership Models

## Opening

Here is a single question that shapes language design more than perhaps any other: **who is responsible for freeing this?**

That one question splits languages into camps. In C, it is the responsibility of the programmer — you allocate with `malloc`, you must free with `free`, or memory leaks. In Java, it is the responsibility of the runtime — you never call free, the garbage collector does. In C++, it depends on the type — a stack object frees when it leaves scope, a smart pointer frees when no references remain. In Rust, the compiler knows — exactly one owner is permitted, and the compiler enforces it.

None of these answers is "obvious" or "right." Each is an engineering choice that costs something and buys something. This chapter is about understanding those choices and how they ripple through language design, performance, and the bugs you'll encounter.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define **ownership** as the obligation to release a resource, and distinguish it from **access** (borrowing).
2. Identify the five major ownership patterns — C convention, C++ unique/shared, Rust borrow-checked, GC'd languages — and recognize them in real code.
3. Explain **move semantics** in C++ concretely: what rvalue references are, what `std::move` actually does, and why moved-from objects must remain valid.
4. Apply **linear** and **affine types** informally to reason about resource safety without formal type theory.
5. Implement a small resource-owning class in C++ with proper move, copy deletion, and destructors.
6. Recognize the tradeoffs each ownership pattern makes and predict which bugs are likely in each.

---

## 13.1 What Ownership Actually Means

Start with a definition:

> **Ownership is the right and obligation to release a resource.**

That is different from *access*. If I give you a pointer to my memory, you have *access* — you can read it, write it, use it. But you do not own it; you are not obligated to free it. When you access borrowed memory and then return to me, I am still responsible for its cleanup.

Three patterns emerge from this distinction:

### Transfer of ownership

```cpp
int* allocate() {
    return new int(42);  // Allocate memory, return pointer
}

int main() {
    int* ptr = allocate();  // I now own ptr
    // ... use ptr ...
    delete ptr;  // I am responsible for cleanup
    ptr = nullptr;  // Good practice: null after delete
}
```

Here, `allocate()` creates a resource and hands ownership to the caller. The caller must remember to clean it up. Forget, and you leak.

### Shared ownership

Multiple parties can own the same resource, and it is freed only when the last owner releases it. Reference counting is the mechanism:

```cpp
std::shared_ptr<int> allocate() {
    return std::make_shared<int>(42);  // Allocate, return shared_ptr
}

int main() {
    auto ptr1 = allocate();  // Reference count = 1
    {
        auto ptr2 = ptr1;  // Reference count = 2
        // ptr2 out of scope: count = 1
    }
    // ptr1 still valid; resource still alive
    // ptr1 out of scope: count = 0, resource freed
}
```

### Borrowing (lending without transfer)

You give someone access to a resource without transferring ownership. The borrower promises not to extend the lifetime:

```cpp
void use(int* borrowed) {
    // I do not own borrowed
    // I must not free it
    // I must not store it (beyond the call)
    std::cout << *borrowed << std::endl;
}

int main() {
    int value = 42;
    use(&value);  // Lend the address
    // Caller is still responsible for cleanup (which happens at scope end)
}
```

In C and classic C++, borrowing is *convention* — enforced by comments, discipline, and code review. In Rust, it is *law* — enforced by the compiler.

---

## 13.2 Five Patterns Across Languages

Different languages solve the ownership problem differently. Each solution has cascading consequences.

### Pattern 1: C — Convention Only

In C, ownership is a *social contract* written in comments and naming:

```c
// Caller owns the returned pointer and must call free()
int* create_buffer(size_t size) {
    return malloc(size);
}

// Takes ownership of the buffer and frees it
void destroy_buffer(int* buffer) {
    free(buffer);
}

// Borrows the buffer; caller retains ownership
void process_buffer(int* buffer, size_t size) {
    for (size_t i = 0; i < size; i++) {
        // use buffer[i]
    }
    // Do not free; you don't own it
}

int main() {
    int* buf = create_buffer(100);
    process_buffer(buf, 100);  // Borrow
    destroy_buffer(buf);        // Transfer ownership & cleanup
}
```

The pattern works *if everyone follows it*. The compiler has no idea who owns what; the responsibility is entirely in the programmer's head.

**Failure mode:** If someone forgets to free, you leak. If someone frees twice, you corrupt the heap. If someone borrows and frees, you use-after-free.

### Pattern 2: C++ Unique Ownership with `std::unique_ptr`

C++ adds the ability to express ownership *in the type system*. `unique_ptr` says: "exactly one owner, and when the owner dies, so does the resource."

```cpp
#include <memory>

std::unique_ptr<int> allocate() {
    return std::make_unique<int>(42);  // Unique owner, caller gets it
}

int main() {
    std::unique_ptr<int> ptr = allocate();
    std::cout << *ptr << std::endl;  // Use it
    // ptr goes out of scope: destructor runs, free() called automatically
    // No manual delete needed
}
```

Uniqueness is enforced by move semantics. You *cannot* copy a `unique_ptr`:

```cpp
std::unique_ptr<int> ptr1 = std::make_unique<int>(42);
std::unique_ptr<int> ptr2 = ptr1;  // Compiler error: no copy allowed
std::unique_ptr<int> ptr2 = std::move(ptr1);  // OK: move transfers ownership
// ptr1 is now empty (points to nullptr)
```

The compiler guarantees: at any moment, exactly one variable owns the resource. When that variable dies, the resource is freed.

**Cost:** You must use move semantics to transfer ownership; it's more explicit than C but less flexible than shared ownership.

### Pattern 3: C++ Shared Ownership with `std::shared_ptr`

When multiple holders genuinely need to own a resource, `shared_ptr` uses reference counting:

```cpp
std::shared_ptr<int> allocate() {
    return std::make_shared<int>(42);  // Ref count = 1
}

int main() {
    auto ptr1 = allocate();  // Ref count = 1
    {
        auto ptr2 = ptr1;  // Ref count = 2
        std::cout << ptr1.use_count() << std::endl;  // 2
    }  // ptr2 destroyed; ref count = 1
    std::cout << ptr1.use_count() << std::endl;  // 1
}  // ptr1 destroyed; ref count = 0, resource freed
```

Unlike `unique_ptr`, you *can* copy a `shared_ptr`. Copying increments the reference count; destruction decrements it. The resource lives as long as *at least one* pointer to it exists.

**Cost:** Atomic reference-count operations are slower than unique ownership. If you create a cycle (A points to B, B points to A), both will leak. You need `weak_ptr` to break cycles.

### Pattern 4: Rust — Borrow Checking Enforced by the Compiler

Rust goes further: it enforces ownership rules *at compile time*, with no runtime overhead.

```rust
fn main() {
    let mut vec = vec![1, 2, 3];  // vec owns the allocation
    
    // Borrow immutably
    let ref1 = &vec;
    let ref2 = &vec;
    println!("{:?} {:?}", ref1, ref2);  // Both are valid
    
    // Now mutable borrow
    vec.push(4);  // OK: no other borrow active
    // println!("{:?}", ref1);  // ERROR: ref1 is invalidated
}
```

The compiler tracks *who* owns each piece of memory and *who* can borrow it. Rules:

1. **Exactly one owner** (like `unique_ptr`).
2. **Either many immutable borrows OR one mutable borrow**, but not both.
3. **Borrows must not outlive the owner** (checked at compile time).

These rules are enforced entirely at compile time. There is no runtime overhead; the binary is as fast as manual C++ code but with memory safety guarantees.

```rust
fn borrow_immutable(vec: &Vec<i32>) {
    // I don't own vec; I'm borrowing it immutably
    println!("{}", vec.len());
}

fn borrow_mutable(vec: &mut Vec<i32>) {
    // I don't own vec; I'm borrowing it mutably (I can change it)
    vec.push(4);
}

fn main() {
    let mut vec = vec![1, 2, 3];
    borrow_immutable(&vec);  // Immutable borrow
    borrow_mutable(&mut vec);  // Mutable borrow (only when no other borrow is active)
    // Both borrows end when the function returns
}
```

**Cost:** The learning curve is steep; some programs that are easy in C++ are difficult in Rust because the compiler is more conservative. The compiler's rules are sound but not all safe programs are expressible directly.

### Pattern 5: Garbage-Collected Languages — Ownership Erased

Languages like Java, Python, Go, and JavaScript delegate all ownership to the runtime. You allocate; the runtime tracks what is reachable, and frees what is not.

```java
public Object allocate() {
    return new Object();  // No explicit free; GC handles it
}

public static void main(String[] args) {
    Object obj = allocate();
    // Use obj
    // obj goes out of scope
    // Eventually (when the GC runs), the GC sees obj is unreachable and frees it
    // You have no control over *when*
}
```

Ownership is a *runtime* property, not a static one. The GC runs periodically, traces all reachable objects (starting from roots like stack variables), and frees anything not reached.

How does the GC know what is reachable? It starts from the roots (all live pointers on the stack and in CPU registers), then follows references: if an object A contains a pointer to object B, and A is reachable, then B is reachable. Anything not reachable is dead and can be freed. The beauty is you never have to think about when that happens; it is *automatic*.

```java
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    strings.add("hello");
    strings.add("world");
    
    // If we lose all references to 'strings' without calling clear()...
    strings = null;  // or go out of scope
    
    // ... the GC will eventually trace from the stack, see it's not referenced,
    // and free the List and all its String contents. No manual cleanup.
}
```

**Cost:** Garbage collection causes unpredictable pauses. You use more memory than necessary (the GC needs "slack" to run efficiently). You lose control over *when* cleanup happens. In latency-sensitive systems (financial trading, real-time control), those pauses can be catastrophic.

**Benefit:** You cannot leak memory (under normal circumstances — cycles of otherwise-unreachable objects *can* leak) and you cannot use-after-free. The entire class of memory-management bugs is eliminated from the programmer's mental load. This is why GC'd languages scale well for large teams: most junior engineers cannot accidentally corrupt the heap.

---

## 13.3 Move Semantics in C++ Concretely

Modern C++ (C++11 and later) introduced **rvalue references** and **move semantics** to support efficient transfer of ownership without the overhead of deep copying. This is one of the most important additions to C++ in decades, and also one of the most frequently misunderstood. This section makes it concrete.

### The Problem: Copies Are Expensive

Before C++11:

```cpp
std::vector<int> create_vector() {
    std::vector<int> v(1000000);  // Million elements
    // ... fill v ...
    return v;  // Copy the entire million-element vector to the caller!
}

int main() {
    std::vector<int> vec = create_vector();  // Expensive copy
}
```

`return v` calls the copy constructor: it allocates new memory, copies all million elements. Then it destroys the temporary. Waste.

### The Solution: Move Constructor

The move constructor says: "instead of copying, steal the innards."

```cpp
class Vector {
    int* data;
    size_t size;
    size_t capacity;

public:
    // Copy constructor: deep copy (expensive)
    Vector(const Vector& other) 
        : data(new int[other.size]), size(other.size), capacity(other.capacity) {
        std::copy(other.data, other.data + other.size, data);
    }

    // Move constructor: steal the internals (cheap)
    Vector(Vector&& other) noexcept 
        : data(other.data), size(other.size), capacity(other.capacity) {
        // Leave other in a valid but empty state
        other.data = nullptr;
        other.size = 0;
        other.capacity = 0;
    }

    ~Vector() {
        delete[] data;  // Safe: either we own it, or it's nullptr
    }
};

int main() {
    Vector v = create_vector();  // Move constructor: cheap! Steal the data.
}
```

Key observations:

1. **The `&&` syntax** means "rvalue reference" — a reference to a temporary or something you're about to move.
2. **`std::move(x)` is a cast.** It is *not* a function that does anything; it simply casts `x` to an rvalue reference, telling the compiler "treat this as a temporary."
3. **The moved-from object must remain valid.** After move, the object must be in a valid state (even if empty). It must be safe to destroy.

### What `std::move` Actually Does

Here is the key insight that clears up the confusion: **`std::move` is not a function that does anything. It is a cast.**

```cpp
std::unique_ptr<int> ptr1 = std::make_unique<int>(42);
std::unique_ptr<int> ptr2 = std::move(ptr1);  // Move ptr1 into ptr2

// What just happened:
// 1. std::move(ptr1) cast ptr1 to an rvalue reference
// 2. The compiler sees an rvalue reference and calls the move constructor
// 3. ptr2's move constructor ran: ptr2._ptr = ptr1._ptr; ptr1._ptr = nullptr;
// 4. ptr1 is now empty (points to nullptr)
// 5. ptr2 owns the original pointer

std::cout << ptr1.get() << std::endl;  // nullptr: it's empty now
std::cout << ptr2.get() << std::endl;  // 0x... : ptr2 owns it
```

`std::move` itself does *nothing* at runtime. Look at its definition:

```cpp
template<typename T>
typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}
```

It is entirely a compile-time cast. The compiler sees the rvalue reference and calls the move constructor instead of the copy constructor. The magic is in the *type* (rvalue reference), not in any runtime code. This is why move is as fast as manual pointer manipulation — it literally is manual pointer manipulation, just with the compiler ensuring you're doing it safely.

### Moved-From Objects Must Be Valid (But May Be Empty)

The rule: after move, the moved-from object must be in a *valid but unspecified state*. It must be safe to destroy, assign, or query (though the query may return surprising values).

```cpp
class Resource {
    int* data;
    size_t size;

public:
    Resource() : data(nullptr), size(0) {}
    
    Resource(size_t sz) : data(new int[sz]), size(sz) {}

    // Move constructor
    Resource(Resource&& other) noexcept 
        : data(other.data), size(other.size) {
        other.data = nullptr;  // Leave it valid: empty
        other.size = 0;
    }

    ~Resource() {
        delete[] data;  // Safe: null check not needed (delete nullptr is safe)
    }

    size_t get_size() const { return size; }  // Safe to call post-move
};

int main() {
    Resource r1(100);
    Resource r2 = std::move(r1);
    
    r1.get_size();  // Returns 0: r1 is empty after move
    delete r1;  // Safe: destructor deletes nullptr, which is a no-op
}
```

**Bad practice:** leaving the moved-from object in an *invalid* state:

```cpp
// BAD: After move, r1.data still points to now-freed memory
Resource(Resource&& other) noexcept : data(other.data), size(other.size) {
    // Do NOT do this: other.data = other.data;  // oops, forgot to null it
}
```

If you return from main without clearing the reference, the destructor runs on the moved-from object and double-frees the memory.

---

## 13.4 Linear and Affine Types (Briefly)

Ownership can be understood through **linear** and **affine types**, concepts from type theory that formalize the notion of "you can only use this once":

- **Linear type**: used *exactly once*. Once you pass it to a function, you cannot use it again. At the end of its lifetime, it must be used exactly once.
- **Affine type**: used *at most once*. Like linear, but you can also *not* use it at all (drop it without using). More permissive.

Rust's ownership model is essentially affine types + borrow semantics + region inference:

```rust
fn consume(vec: Vec<i32>) {
    // Takes ownership; vec is consumed (used exactly once, here)
    println!("{:?}", vec);
}

fn main() {
    let vec = vec![1, 2, 3];
    consume(vec);  // Ownership transferred; vec is consumed
    // println!("{:?}", vec);  // ERROR: vec was consumed above; it no longer exists
}
```

`consume(vec)` is affine: the function consumes the value exactly once. After the call, `vec` is no longer available. It cannot be used again, and the compiler enforces it.

Compare to borrowing, which is not affine:

```rust
fn borrow(vec: &Vec<i32>) {
    // Borrows; vec is NOT consumed
    println!("{:?}", vec);
}

fn main() {
    let vec = vec![1, 2, 3];
    borrow(&vec);  // Borrow; vec is still available
    borrow(&vec);  // Can borrow again
    println!("{:?}", vec);  // Still available
    
    // vec goes out of scope and is properly dropped (exactly once)
}
```

Here, `vec` is borrowed (not consumed) by each call, so you can call `borrow` multiple times. When `vec` goes out of scope at the end of `main`, it is dropped *exactly once* — the affine property is maintained.

The affine type system is how Rust expresses "who owns this resource, and is the owner still alive?" at the type level, with no runtime overhead. The compiler tracks ownership statically, proving at compile time that no use-after-free can occur. This is the theoretical foundation behind the borrow checker: affine types + lifetime tracking.

---

## 13.5 Worked Example: A Simple Buffer Type

Let's implement a small `Buffer` class in C++ that owns dynamically allocated memory. We'll see what happens if we forget each of the five special member functions.

### Correct Implementation

```cpp
class Buffer {
private:
    uint8_t* data;
    size_t size;

public:
    // Constructor
    Buffer(size_t sz) : size(sz) {
        data = new uint8_t[sz];
    }

    // Destructor: free the owned memory
    ~Buffer() {
        delete[] data;
    }

    // Deleted copy constructor: no copying allowed (unique ownership)
    Buffer(const Buffer&) = delete;

    // Deleted copy assignment: no copying allowed
    Buffer& operator=(const Buffer&) = delete;

    // Move constructor: steal the innards
    Buffer(Buffer&& other) noexcept : data(other.data), size(other.size) {
        other.data = nullptr;  // Leave other valid but empty
        other.size = 0;
    }

    // Move assignment: clean up self, then steal other's internals
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data;  // Clean up what we owned
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }

    uint8_t* get() { return data; }
    size_t get_size() const { return size; }
};

int main() {
    Buffer buf(100);
    buf.get()[0] = 42;
    
    Buffer buf2 = std::move(buf);  // Move: buf is now empty
    buf2.get()[0];  // OK: buf2 owns it now
    // buf.get();  // Would be nullptr
    
    return 0;  // buf2 destructor runs, frees memory
}
```

### What Happens If You Forget Move Constructor

```cpp
class BadBuffer {
    uint8_t* data;
    size_t size;

public:
    BadBuffer(size_t sz) : size(sz) { data = new uint8_t[sz]; }
    ~BadBuffer() { delete[] data; }
    
    // MISSING: move constructor and move assignment
};

int main() {
    BadBuffer buf1(100);
    BadBuffer buf2 = std::move(buf1);  // Uses default move (memberwise copy)
    
    // buf1.data and buf2.data point to the SAME memory!
    // When buf1 is destroyed, delete[] runs on that memory
    // When buf2 is destroyed, delete[] runs again on that memory
    // Double-free! Crash.
}
```

The compiler generates a default move constructor that does memberwise move (copying pointers). Both objects think they own the same memory. Disaster.

### What Happens If You Forget Destructor

```cpp
class LeakyBuffer {
    uint8_t* data;
    size_t size;

public:
    LeakyBuffer(size_t sz) : size(sz) { data = new uint8_t[sz]; }
    // MISSING: destructor
};

int main() {
    {
        LeakyBuffer buf(100);
    }  // buf goes out of scope, no destructor to free data
    
    // The allocated 100 bytes are never freed. Leak!
}
```

### What Happens If You Copy When You Shouldn't

```cpp
class WrongCopyBuffer {
    uint8_t* data;
    size_t size;

public:
    WrongCopyBuffer(size_t sz) : size(sz) { data = new uint8_t[sz]; }
    ~WrongCopyBuffer() { delete[] data; }
    
    // WRONG: user-defined copy that assumes ownership
    WrongCopyBuffer(const WrongCopyBuffer& other) 
        : data(other.data), size(other.size) {
        // Shallow copy: both point to the SAME memory
    }
};

int main() {
    WrongCopyBuffer buf1(100);
    WrongCopyBuffer buf2 = buf1;  // Shallow copy
    
    // buf1 and buf2 both point to the same data
    // When buf1 is destroyed, delete[] runs
    // When buf2 is destroyed, delete[] runs again on freed memory
    // Use-after-free! Crash.
}
```

The lesson: **ownership requires all five special members be thought through**: destructor, copy constructor, copy assignment, move constructor, move assignment. Mess any of them up and you get leaks, double-free, or use-after-free.

---

## 13.6 Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **C convention** | Maximum flexibility; you own the design | Completely manual; easy to leak/double-free; no compiler help | Systems code where flexibility and explicit control are paramount |
| **C++ `unique_ptr`** | Zero-cost abstraction; no runtime overhead; type-safe ownership | Must use move semantics; cannot share; more verbose than GC | Default choice in modern C++ for single ownership |
| **C++ `shared_ptr`** | Can share ownership; safe reference counting | Atomic operations overhead; cycles leak; more memory use | When genuinely multiple parties own a resource |
| **Rust borrow-check** | Memory safety guaranteed at compile time; no runtime cost; elegant; prevents entire classes of bugs | Steep learning curve; some safe programs are hard to express; slow compile times | New code where memory safety is non-negotiable |
| **Garbage collection** | No leaks (under normal circumstances); no use-after-free; simple programming model | Unpredictable pauses; higher memory use; less control over cleanup timing | Rapid iteration, rapid scaling (Java, Go); business logic not latency-critical |

---

## 13.7 Common Misconceptions

1. *"Move is faster because it doesn't copy."* — **Move is fast because it does *less work* (stealing pointers instead of copying buffers), but `std::move` itself is zero-cost. It's literally just a cast.**

2. *"Garbage collection eliminates memory bugs."* — **GC eliminates leaks and use-after-free, but not cycles, not dangling pointers in C-FFI, not memory exhaustion bugs, and not race conditions on mutable state. It's safer than manual memory, not magic.**

3. *"Rust's borrow checker means you can't write efficient code."* — **The borrow checker is a compile-time analysis. It has no runtime cost. Unsafe code exists for the rare cases the checker is too conservative. You can write Rust that is as fast as C.**

4. *"I should always use `shared_ptr` to be safe."* — **`shared_ptr` has overhead (atomic operations, reference count allocations). Use `unique_ptr` by default; only reach for `shared_ptr` when you genuinely need to share. This is called "ownership in breadth vs depth."**

5. *"Once I call `std::move(x)`, x is invalid and I can't use it anymore."* — **`x` remains *valid but empty*. You can still call methods on it, destroy it, or reassign it. You just can't assume it contains useful data.**

---

## 13.8 Exercises

1. **Implement a simple `DynamicArray` class in C++** with all five special members (destructor, copy constructor, copy assignment, move constructor, move assignment). The class should own a dynamically allocated array of integers. Test it: (a) create an array of 100 elements, (b) move it to another variable and verify the original is empty, (c) assign one `DynamicArray` to another and verify the left-hand side is cleaned up first, (d) create one in a scope, let it go out of scope, and confirm no crash or leak (use valgrind or similar if available).

2. **Convert C code to C++.** Take a piece of C code that uses explicit `malloc`/`free` (you can write a simple linked-list implementation or matrix manipulation code). Rewrite it using `unique_ptr` or `shared_ptr`. Remove all manual `delete`. Verify it compiles and runs without leaks. Commit to the constraint: no `new`, no `delete`, only `std::make_unique` or `std::make_shared`.

3. **Reason about Rust.** Write a Rust program with three functions: (a) `consume(vec: Vec<i32>)` that takes ownership and prints it; (b) `borrow_imm(vec: &Vec<i32>)` that borrows immutably and prints; (c) `borrow_mut(vec: &mut Vec<i32>)` that borrows mutably and pushes a new element. In `main`, call them in sequence. Then deliberately violate the borrow rules (e.g., call borrow_imm and then borrow_mut without a scope separation) and document the compiler error you get.

4. **Spot the ownership bug.** Look at a legacy C++ codebase (or write one intentionally with the bug). Find a function that allocates memory and returns a raw pointer with ambiguous ownership — something like `int* get_data()`. Determine: does the caller own it? Does the callee retain ownership? Can you tell from the signature alone, or must you read implementation details? Propose a fix using smart pointers, and rewrite the function signature to make ownership explicit.

5. **Trade-off analysis.** Imagine writing a multithreaded in-memory cache service (like Memcached). Sketch the ownership model and memory management strategy in C (raw pointers + reference counting), C++ with `unique_ptr`, C++ with `shared_ptr`, Rust, and Java. For each, estimate: (a) the cognitive load to get it right, (b) the likelihood of memory bugs (leaks, use-after-free, double-free), (c) the runtime overhead. Which feels most natural for shared, cached data that may be accessed from many threads? Which most dangerous?

---

## 13.9 Summary

Ownership is a fundamental question that shapes language design: **who frees this memory?** Different languages give radically different answers, and each answer cascades into design decisions about types, semantics, syntax, runtime overhead, and the bugs you'll encounter.

**C** says: you do, by social convention — comments, naming, and discipline. Maximum flexibility, maximum responsibility. Easy to leak, use-after-free, and double-free if you slip.

**C++** offers a spectrum: the type system can *express* ownership via `unique_ptr` (one owner), `shared_ptr` (many owners), and RAII (owner is the scope). With move semantics in C++11+, you get safety close to garbage collection with the performance of manual memory management. But you must understand the five special members and move semantics deeply.

**Rust** says: the compiler enforces ownership. Exactly one owner (enforced at compile time), borrows must not outlive the owner (checked statically), and the affine type system guarantees you cannot use-after-free or leak. No runtime overhead. The learning curve is steep because the compiler is strict, but once it compiles, a whole class of bugs is eliminated.

**Garbage-collected languages** (Java, Python, Go, JavaScript) say: the runtime does it. You allocate; the GC traces reachable objects and frees the rest. No leaks, no use-after-free. The cost: unpredictable pauses, higher memory use, and loss of deterministic cleanup. Excellent for rapid iteration and large teams.

None is universally "best." The tradeoff space is constrained: you cannot have Rust's compile-time safety *and* Python's dynamism *and* C's flexibility simultaneously. You choose which you optimize for.

Modern C++ (with `unique_ptr`, `shared_ptr`, move semantics, RAII) recovers much of the safety of garbage collection without the runtime overhead — you get deterministic cleanup and zero-cost abstractions, often matching C's performance while exceeding it in safety. This is why C++ is still the right tool for systems that must be fast, safe, and long-lived.

The next chapter, Lifetimes, takes this further: in languages like Rust, ownership is not just about cleanup, it is about *proving that the owner is still alive* for a borrow to be valid. Lifetime inference is how Rust achieves memory safety at compile time without requiring you to manually tag every reference with how long it must live. It is the final puzzle piece in the ownership model.

---

## 13.10 What's Next

[← Previous: Manual Memory Management](02-manual-memory-management.md) · [↑ Part 2](README.md) · [Next: Lifetimes →](04-lifetimes.md)
