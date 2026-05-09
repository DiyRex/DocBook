# Chapter 14 — Lifetimes

The most common cause of crashes and undefined behavior in unsafe code is not a logic error in your algorithm. It is not a type mismatch. It is asking to read an object after the interval during which it is legal to read it. That interval is the object's **lifetime**, and this chapter teaches you to reason about lifetimes precisely.

A lifetime is the interval during which it is legal to read or write this object. Most bugs in unsafe code are lifetime bugs in disguise.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish **storage duration** (automatic, static, dynamic, thread-local) from **object lifetime** (initialization to destruction), and recognize that they are related but not identical.
2. Predict the lifetime of any C++ object given how and where it is declared, allocated, and destroyed.
3. Identify and trace the three most common dangling pointer patterns: returning a reference to a local, storing a pointer to a temporary, and iterator invalidation.
4. Detect lifetime bugs with dynamic analysis tools — AddressSanitizer (ASan), Valgrind — and understand what each tool reports.
5. Compare lifetime models across languages: Rust's encoded lifetimes, GC's unbound lifetimes, Go's simplifications.
6. Refactor a program with dangling references to be correct and safe.

---

## 14.1 Storage Duration vs Object Lifetime

C++ defines four **storage durations** — categories describing *when* the memory backing an object is allocated and deallocated:

1. **Static storage** — allocated once at program load, deallocated at program exit.
2. **Automatic storage** — allocated when entering scope, deallocated when exiting scope.
3. **Dynamic storage** — allocated on request (via `new` or `malloc`), deallocated on request (via `delete` or `free`).
4. **Thread-local storage** — one allocation per thread, allocated at thread start, deallocated at thread exit.

**Object lifetime** is different: it is the interval from **initialization** (constructor completes) to **destruction** (destructor starts).

The two concepts are related because storage duration sets the boundary. An object with automatic storage duration lives at most as long as the block containing its declaration. But the object's lifetime may be *shorter* — if you explicitly call the destructor before exiting the block, or if the constructor threw an exception and was never completed.

This distinction matters because **dangling references happen when you hold a pointer or reference to an object whose lifetime has ended, while its storage duration has not.** This is why C++ requires you to manually track lifetimes in unsafe code — the type system alone cannot prove that a pointer is valid.

### Storage Duration in the Address Space

Recall Chapter 5's memory layout. Storage durations map directly:

- **Static**: lives in `.data` or `.bss`, with address known at compile time.
- **Automatic**: lives on the stack, address determined at runtime (depends on call depth).
- **Dynamic**: lives in the heap, address determined by the allocator at the moment of allocation.
- **Thread-local**: lives in TLS (thread-local storage), with offset known at compile time but a distinct copy per thread.

```cpp
int global = 1;                    // static storage, lives in .data
thread_local int per_thread = 2;   // thread-local storage, lives in TLS

void foo() {
    int local = 3;                 // automatic storage, lives on stack
    int* heap = new int(4);        // dynamic storage, lives in heap
}
```

When you exit `foo`, `local` is deallocated (stack shrinks); `heap` is not (must `delete` explicitly). Static storage persists for the entire program run. Thread-local persists for the thread's lifetime.

---

## 14.2 The Four Storage Classes Concretely

Let's examine each class in detail, focusing on the contracts they impose.

### Static Storage

```cpp
int counter = 0;           // global, .data
static int internal = 0;   // internal linkage, .data
const int K = 100;         // may be .rodata if truly constant

void increment() {
    counter++;             // modifying static storage, safe (one copy per process)
}
```

**Allocation**: once, at program load.  
**Deallocation**: at program exit, in reverse order of initialization.  
**Address**: known at link time.  
**Visibility**: global (or `static` for internal linkage, visible only in this translation unit).  
**Thread implications**: all threads see the same copy; modifications are data races unless protected by synchronization.

**Lifetime contract**: static storage objects are initialized before `main()` runs (unless they have dynamic initialization, in which case they're initialized on first use or at program load). They are valid from that point until `atexit()` handlers run.

The key risk: **static initialization order fiasco**. If global A's initializer calls a function that uses global B, and B hasn't been initialized yet, you have undefined behavior.

```cpp
// a.cpp
int a() { return b + 1; }   // assumes b is initialized
int result = a();           // calls a()

// b.cpp
int b = 42;                 // b's init order relative to result is undefined

// main.cpp
extern int b, result;
int main() {
    // if a() was called before b() initialized, result is garbage
    return result;
}
```

**Threading**: not safe for mutation from multiple threads without locking.

### Automatic Storage

```cpp
void foo() {
    int local = 1;                // automatic storage, lives on stack
    std::vector<int> v;           // automatic storage for the vector object
    for (int i = 0; i < 10; ++i) {
        int inner = i;            // another automatic storage
    }                             // inner goes out of scope here
    // v's destructor runs when v goes out of scope
}                                 // local goes out of scope, stack shrinks
```

**Allocation**: when entering the scope (the address is determined by the stack pointer at that moment).  
**Deallocation**: when exiting the scope (destructor runs, stack shrinks).  
**Address**: determined at runtime, dependent on call depth. Highly cache-efficient (stack is hot).  
**Visibility**: local to the block.  
**Thread implications**: each thread has its own stack, so automatic storage is thread-safe for mutations *within that thread*.

**Lifetime contract**: the lifetime of an automatic object is guaranteed to end when its block exits, and guaranteed to begin when its declaration is reached (assuming the constructor doesn't throw). This is the foundation of RAII (Resource Acquisition Is Initialization).

**Risk**: returning a reference or pointer to an automatic object, where the caller uses it after the block exits. This is a **dangling reference**.

```cpp
int& bad_return() {
    int local = 42;
    return local;              // dangerous; local will be deallocated at }
}

int main() {
    int& ref = bad_return();   // ref now points to deallocated stack memory
    std::cout << ref;          // undefined behavior
}
```

**Threading**: automatic objects are safe; no synchronization needed. But if you take the address and pass it to another thread, the other thread must not access it after your thread exits or the block ends.

### Dynamic Storage

```cpp
void foo() {
    int* p = new int(42);      // dynamic storage
    // ...
    delete p;                  // deallocate; p is now invalid
    // p = nullptr;            // good practice, but the pointer itself is automatic
}
```

**Allocation**: when you call `new` (or `malloc`), at a time you control.  
**Deallocation**: when you call `delete` (or `free`), at a time you control. **No automatic deallocation.**  
**Address**: determined by the allocator; typically unpredictable (depends on fragmentation history).  
**Visibility**: determined by the pointer's scope (the pointer itself is usually automatic or static).  
**Thread implications**: multiple threads can allocate from the same heap, but synchronization is handled by the allocator (it locks). Reading/writing the *object* from multiple threads is not synchronized; you must add your own locks.

**Lifetime contract**: the lifetime begins when `new` completes and ends when `delete` is called. **There is no automatic boundary.** You must manually track when to delete, or use a smart pointer (a wrapper with automatic storage that deletes dynamically).

**Risk**: every error in manual memory management — use after free, double free, forget to delete (memory leak), delete too early (dangling pointers remain).

**Modern solution**: `std::unique_ptr<T>` and `std::shared_ptr<T>` give dynamic storage automatic deallocation. They are objects with automatic storage that call `delete` in their destructor. This is a wrapper that *combines* automatic storage's guarantee with dynamic storage's flexibility.

```cpp
{
    std::unique_ptr<int> p(new int(42));   // automatic storage for the pointer
    // dynamic storage for the int (allocated by new, held by unique_ptr)
    // ...
}  // p goes out of scope, its destructor calls delete on the int
```

**Threading**: dynamic storage is thread-safe only if you serialize access. The allocator is thread-safe (locked internally), but the object itself requires your synchronization.

### Thread-Local Storage

```cpp
thread_local int per_thread = 0;   // TLS storage

void worker() {
    per_thread = 1;                // modifies *this thread's* copy
}

// In main:
std::thread t1(worker);
std::thread t2(worker);
// t1 and t2 each modify their own per_thread copy; no data race
```

**Allocation**: once per thread, at thread start (or on first use, depending on the implementation).  
**Deallocation**: when the thread exits.  
**Address**: constant within a thread, different across threads. The compiler emits `mov %fs:offset` (x86-64) to access it.  
**Visibility**: appears global but is per-thread.  
**Thread implications**: reads and writes are safe (each thread has its own copy). But the variable itself is not visible to other threads without explicit synchronization.

**Lifetime contract**: TLS variables are initialized (once) before any code in that thread runs. They are valid until the thread exits.

**Risk**: passing a pointer to a TLS variable to another thread and using it there — you're reading another thread's private storage, which is a data race.

---

## 14.3 Temporary Objects and Lifetime Extension

A **temporary** is an unnamed object created to hold a value, typically:

- The result of an expression: `x + y` creates a temporary to hold the sum.
- A function's return value: `foo()` creates a temporary for the return value.
- An explicit construction: `std::string("hello")` creates a temporary string.

By default, a temporary's lifetime is the end of the statement:

```cpp
std::string temp = std::string("hello");   // temporary created, copied, destroyed at ;
std::cout << temp << std::endl;            // temp now empty if move was used
```

### Lifetime Extension with const T&

If you bind a temporary to a `const` reference, its lifetime extends to the reference's scope:

```cpp
const std::string& ref = std::string("hello");  // temporary lifetime extended
std::cout << ref << std::endl;                   // valid; temporary still alive
```

Why does this work? Because `const` promises you won't modify the temporary, and the compiler can guarantee it lives as long as the reference. This is a explicit exception to the "temporary dies at `;`" rule.

### Lifetime Extension with auto

If you use `auto` to deduce a type and the value is a temporary, auto will *not* extend the lifetime:

```cpp
auto str = std::string("hello");  // auto deduces std::string, not const string&
                                  // temporary is copied, original is destroyed
std::cout << str << std::endl;    // str is the copy (valid)

auto& ref = std::string("hello"); // auto& deduces string&, but temporary dies at ;
std::cout << ref << std::endl;    // undefined behavior; dangling reference
```

The rule: **only `const T&` and `const T&&` extend the lifetime of a temporary. Non-const references cannot bind to temporaries (C++03 onward), and `auto` does not extend.**

### Worked Example: Returning a Temporary

```cpp
const char* get_string_BAD() {
    std::string s = "hello";
    return s.c_str();   // c_str() returns const char*, valid only during s's lifetime
}                       // s destroyed here, pointer now dangling

int main() {
    const char* p = get_string_BAD();
    std::cout << p << std::endl;   // undefined behavior
}
```

The fix: return a copy or a `std::string`:

```cpp
std::string get_string_GOOD() {
    std::string s = "hello";
    return s;   // returns a copy or moves s (temporary), lifetime extends to assignment
}

int main() {
    std::string s = get_string_GOOD();
    std::cout << s << std::endl;   // valid
}
```

---

## 14.4 Dangling References — Anatomy

A **dangling reference** is a pointer or reference to an object whose lifetime has ended. The three most common patterns:

### Pattern 1: Returning a Reference to a Local

```cpp
int& get_answer() {
    int local = 42;
    return local;           // WRONG: returning reference to automatic storage
}                           // local deallocated at }

int main() {
    int& ref = get_answer();
    std::cout << ref;       // undefined behavior; ref points to deallocated stack
}
```

**Detection**: easy to spot in review. Compiler warnings help; AddressSanitizer detects the use.

**Fix**: return by value:

```cpp
int get_answer() {
    int local = 42;
    return local;           // copy or move; caller gets a valid copy
}
```

### Pattern 2: Storing a Pointer to a Temporary

```cpp
struct Node {
    const std::string* name;
    Node(const std::string& n) : name(&n) {}   // WRONG if n is a temporary
};

int main() {
    {
        std::string temp = "Alice";
        Node node(temp);
        std::cout << *node.name << std::endl;   // valid; temp still alive
    }                                           // temp destroyed
    // node.name now dangling
    std::cout << *node.name << std::endl;       // undefined behavior
}
```

**Why it's subtle**: the constructor correctly takes `const std::string&`, which binds to temporaries. But it stores the pointer, which outlives the temporary.

**Fix**: store a copy:

```cpp
struct Node {
    std::string name;
    Node(const std::string& n) : name(n) {}    // copy the string
};
```

Or use a proper ownership model:

```cpp
struct Node {
    std::unique_ptr<std::string> name;
    Node(const std::string& n) : name(std::make_unique<std::string>(n)) {}
};
```

### Pattern 3: Iterator Invalidation

Iterators are pointers to elements in a container. If the container reallocates (grows), iterators to the old storage are dangling.

```cpp
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);             // vector reallocated; it is now dangling

std::cout << *it << std::endl;  // undefined behavior
```

**Why it happens**: `std::vector` grows by doubling capacity. When capacity is exceeded, it allocates a larger array and moves all elements. The old addresses are invalid.

**Detection**: AddressSanitizer catches the out-of-bounds dereference.

**Fix**: don't cache iterators across mutations:

```cpp
std::vector<int> v = {1, 2, 3};
v.push_back(4);
auto it = v.begin();        // get iterator *after* mutations
std::cout << *it << std::endl;  // valid
```

Or use indices (not invalidated by reallocation):

```cpp
std::vector<int> v = {1, 2, 3};
int index = 0;
v.push_back(4);
std::cout << v[index] << std::endl;  // valid
```

---

## 14.5 Detecting Lifetimes Bugs with Tools

### AddressSanitizer (ASan)

Compile with `-fsanitize=address`. ASan instruments all memory accesses to detect:

- Use-after-free
- Heap buffer overflows
- Stack buffer overflows
- Initialization-order issues

```cpp
// compile: clang++ -fsanitize=address -g test.cpp -o test
int& dangling() {
    int local = 42;
    return local;
}

int main() {
    int& ref = dangling();
    std::cout << ref << std::endl;   // ASan reports use-after-free
}
```

Output (abbreviated):

```
=================================================================
==12345==ERROR: AddressSanitizer: stack-use-after-return on address
    0x7fff12345678 at pc 0x000000401234 bp 0x7fff12345600 sp 0x7fff12345590
READ of size 4 at 0x7fff12345678 thread T0
    #0 0x401233 in main test.cpp:8
Address 0x7fff12345678 is located in stack of thread T0 at offset 32 in frame
    #0 0x401000 in dangling()
    This frame has 1 object(s):
      [32, 36) 'local' <== Memory access at offset 32 is inside this variable
=================================================================
```

ASan pinpoints the exact variable and access. This is one of the most valuable tools for hunting lifetime bugs.

### Valgrind

Valgrind's `--leak-check=full` and `--track-origins=yes` catch memory leaks and use-after-free:

```bash
valgrind --leak-check=full --track-origins=yes ./test
```

Valgrind is slower than ASan but works on more platforms (ARM, older x86, etc.). Both should be in your CI.

### Static Analysis

Modern C++ compilers and tools like `clang-analyzer` can sometimes warn about obvious lifetime bugs:

```cpp
int& bad() { int x = 1; return x; }
// warning: reference to local variable 'x' returned [-Wreturn-reference]
```

But static analysis cannot catch all dangling references (the problem is undecidable in general). Always pair with dynamic analysis.

---

## 14.6 Lifetimes in Other Languages

### Rust: Lifetimes in the Type System

Rust encodes lifetimes explicitly in types, making them part of the compile-time contract:

```rust
fn get_string(s: &String) -> &String {
    s  // okay; return reference to input
}

fn bad_return() -> &String {
    let s = String::from("hello");
    &s  // compile error: s doesn't live long enough
}
```

In Rust, every reference carries a **lifetime parameter** `'a`:

```rust
fn get_string<'a>(s: &'a String) -> &'a String {
    s
}
```

The compiler proves that the returned reference lives at least as long as the input. If you try to return a reference to a local, the compiler rejects it at compile time. There is no way to write the dangling reference pattern in safe Rust.

This comes at a cost: Rust's borrow checker is sophisticated and sometimes rejects correct programs that it cannot prove safe. But the guarantee is absolute: **if it compiles, there are no use-after-free bugs.**

### Go: Managed Lifetimes (Garbage Collected)

Go uses garbage collection. Objects are allocated with `new()` or `:=` and remain valid until unreachable:

```go
func get_string() string {
    s := "hello"
    return s  // okay; string is copied to the caller
}

func get_pointer() *string {
    s := "hello"
    return &s  // okay; GC will keep s alive while reachable
}
```

Go's escape analysis determines whether an object can be stack-allocated or must be heap-allocated. If you take a pointer to a local that escapes (is returned or stored), the compiler automatically heap-allocates it. You never write `new` explicitly for this; the compiler decides.

The cost: GC overhead and the inability to precisely control when objects are freed. But there are no manual lifetime bugs.

### Java: Implicit Heap Allocation

Java heap-allocates almost everything and relies on GC:

```java
String getString() {
    String s = "hello";
    return s;  // okay; GC keeps alive
}
```

Like Go, but Java does not stack-allocate anything by default (primitives are an exception).

### Python: Reference Counting

Python uses reference counting: when an object's reference count reaches zero, it is deallocated immediately. This is simpler than GC but suffers from cycles (A refers to B, B refers to A, neither gets freed). Python includes a cycle detector.

```python
def get_string():
    s = "hello"
    return s  # okay; reference count > 0 while in scope
```

---

## 14.7 Worked Example: A Dangling Reference and Its Fix

### The Bug

```cpp
#include <iostream>
#include <vector>

class Cache {
public:
    const std::string& get(int key) {
        auto it = data.find(key);
        if (it != data.end()) {
            return it->second;  // WRONG: returning reference to map's internal value
        }
        // If key not found, create a temporary and return a reference to it
        temp = "not found";
        return temp;            // WRONG: returning reference to member (ok)
    }

private:
    std::unordered_map<int, std::string> data;
    std::string temp;
};

int main() {
    Cache cache;
    cache.data[1] = "Alice";
    
    const std::string& ref = cache.get(1);
    cache.data[2] = "Bob";     // inserting into map can rehash -> invalidate iterators/refs
    
    std::cout << ref << std::endl;  // undefined behavior; ref may be dangling
}
```

The bug: `std::unordered_map` may rehash when you insert, invalidating all references to its values. The reference returned is to a value inside the map, but the map's internal structure can change.

### Diagnosis

```bash
$ clang++ -fsanitize=address -g cache.cpp -o cache
$ ./cache
=================================================================
==123==ERROR: AddressSanitizer: heap-use-after-free on address
    0x60200000dfb0 at pc 0x000000401234
READ of size 8 at 0x60200000dfb0 thread T0
...
```

ASan reports the rehash invalidated the reference.

### The Fix

Return by value instead:

```cpp
class Cache {
public:
    std::string get(int key) {  // returns by value
        auto it = data.find(key);
        if (it != data.end()) {
            return it->second;  // copy; lifetime ends at caller's scope
        }
        return "not found";     // temporary is copied
    }

private:
    std::unordered_map<int, std::string> data;
};
```

Now the caller owns the copy, and insertions into the map don't affect it.

For performance-critical code, return `std::string_view` if the caller will consume it immediately:

```cpp
std::string_view get_view(int key) {
    auto it = data.find(key);
    if (it != data.end()) {
        return it->second;  // view into the map's value
    }
    return "not found";     // view into a string literal (static storage)
}

int main() {
    std::string s = cache.get_view(1);  // must copy immediately, before next insertion
}
```

But now the caller must *not* insert before using the view. This is fragile — the value-returning version is safer.

---

## 14.8 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Scope-based (RAII, automatic)** | No manual tracking; exception-safe; deterministic cleanup | Limited to lexical scope; stack-allocated limits size | Default for single-threaded code, small objects, owned resources |
| **Explicit lifetimes (Rust's `'a`)** | Compile-time checked; zero-cost at runtime; no GC pauses | Complex type syntax; borrow checker rejects some valid programs; learning curve | Performance-critical systems; embedded; zero false-positive safety |
| **Garbage collection (Go, Java)** | No manual lifetime tracking; safe by default; simpler type system | GC pauses; unpredictable performance; memory overhead; cycles | Application servers; high-level code where GC pause is acceptable |
| **Manual (C malloc/free)** | Explicit control; simple implementation; maximum flexibility | Easy to get wrong; use-after-free; leaks; difficult to refactor | Legacy code; extremely performance-sensitive kernels (rare) |
| **Smart pointers (unique_ptr, shared_ptr)** | Automatic deallocation; owner is explicit; move semantics | Shared ownership (shared_ptr) has overhead; reference cycles leak | Default for dynamic storage in modern C++; polymorphic objects |

---

## 14.9 Common Misconceptions

1. **"Lifetime and scope are the same thing."** No. Scope is lexical (where the name is visible). Lifetime is semantic (when the object is valid). A name can go out of scope while the object is still valid (captured by a lambda, held in a global). An object can lose lifetime while in scope (freed early).

2. **"Static storage means the variable never changes."** No. Static storage means it lives for the entire program. The value can change. `static int counter = 0;` is in static storage and can be incremented. Static storage doesn't imply `const`.

3. **"If I see an address, I know the lifetime."** Partially true. The address hints at the storage duration: high address (stack), low/medium address (executable region or heap), thread-local register (TLS). But the actual lifetime depends on explicit deallocation (dynamic storage) or block structure (automatic storage).

4. **"I can extend a temporary's lifetime by storing it in a const variable."** No. `const std::string x = temporary;` creates a copy; the temporary dies at `;`. You cannot store "the object" via `const` alone. You *can* extend a temporary by binding it to `const T&`, but that only extends it to the reference's scope, not the variable's scope.

5. **"Returning a reference is always wrong."** No. Returning a reference to a parameter (if valid for the caller's use) is fine:
   ```cpp
   std::string& getRef(std::string& s) { return s; }  // okay; caller owns s
   ```
   Returning a reference to a local is wrong. Returning a reference to a static or a parameter is safe (if the parameter is guaranteed to outlive the return).

6. **"Smart pointers solve all lifetime problems."** Smart pointers solve *ownership transfer* and *deallocation* problems. They do not prevent dangling references to the object. `shared_ptr` can still dangle if you store a pointer to the object and the smart pointer is destroyed separately.

---

## 14.10 Exercises

1. **Trace a lifetime.** Write a program with:
   - A global `int global`.
   - A function with automatic `int local` and dynamic `int* heap`.
   - A lambda that captures by reference.

   Print addresses and explain which regions they're in. When is each object valid?

2. **Find the bug.** Given this code, identify the dangling reference and rewrite it safely:
   ```cpp
   std::vector<std::string>& build_list() {
       std::vector<std::string> v;
       v.push_back("hello");
       return v;
   }
   ```

3. **Iterator invalidation in detail.** Write a loop that inserts into a `std::vector` while iterating over it. Compile with ASan. What does ASan report? Fix it three ways:
   - Cache the size before the loop.
   - Use indices instead of iterators.
   - Use a separate vector for insertions.

4. **Temporary lifetime extension.** Write code demonstrating:
   - A temporary bound to `const T&` (valid).
   - A temporary bound to `auto` (what happens?).
   - A temporary returned from a function and bound immediately (valid or not?).

5. **Rust's lifetime syntax.** Read a Rust example with lifetime parameters (from the official book, Chapter 10). Rewrite it in C++ using smart pointers or value semantics. What did Rust's type system buy you?

6. **Static initialization order.** Create two translation units where one global's initializer depends on another's, in the wrong order. Observe the bug, then fix it using the "Schwarz Counter" pattern or lazy initialization.

---

## 14.11 Summary

A lifetime is the interval during which it is legal to read or write an object. It begins at the end of the constructor and ends at the start of the destructor (or explicit destruction). Storage duration sets the boundary, but the actual lifetime can be shorter if the object is explicitly destroyed.

The four storage durations — static, automatic, dynamic, thread-local — each have different deallocation guarantees and threading implications. Most lifetime bugs arise from holding a pointer or reference to an object whose lifetime has ended: returning a reference to a local, storing a pointer to a temporary, or using an iterator after a container reallocation.

Modern C++ tools (ASan, smart pointers) prevent many lifetime bugs. Rust proves lifetimes at compile time; Go and Java avoid the problem with garbage collection; C++ requires you to reason carefully, but offers control and determinism.

Mastery of lifetimes is mastery of the invisible bugs that crash production systems. The next chapter, RAII, shows how to make lifetimes *automatic* within C++'s type system.

---

**[← Previous: Chapter 13 — Ownership Models](03-ownership-models.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 15 — RAII Explained Deeply →](05-raii-explained-deeply.md)**
