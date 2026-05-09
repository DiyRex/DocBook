# Chapter 65 — Why Rust Ownership Exists

## Opening

Rust's reputation is "the language with the borrow checker." That phrase makes the borrow checker sound like an obstacle, a punishment inflicted by a language designer with a grudge against pleasant programming. In truth, the borrow checker is a symptom. The real innovation is *ownership*.

Ownership is not new. C has it (implicit, in comments). C++ has it (in smart pointers and RAII). Java has it (erased by the garbage collector). Rust's genius is making ownership *explicit and enforceable at compile time*, with zero runtime overhead.

This chapter explains the problem ownership solves and why the industry has reached different answers. You will understand why Rust exists, what it trades away, and when the trade is worth making.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. State the three-language constraint: memory safety, performance, and concurrency. Explain which two of three any mainstream language actually achieves.
2. Identify the core problem ownership solves: proving at compile time that no pointer will be dereferenced after its target is freed.
3. Compare Rust's compile-time ownership enforcement to C++'s runtime assertions via smart pointers, and to garbage collection's dynamic tracing.
4. Reason about the tradeoffs: Rust's steeper learning curve vs. guaranteed memory safety; GC's simplicity vs. pauses and overhead.
5. Recognize patterns (graphs, doubly-linked lists, interior mutability) that make Rust's ownership rules difficult, and understand why.
6. Assess a problem domain and choose between Rust, C++, Java, or Python based on engineering constraints.

---

## 65.1 The Three-Language Tradeoff

Every language designer faces a trilemma. You can pick any two, but not all three:

1. **Memory safety**: no use-after-free, no buffer overflows, no dangling pointers, no data races.
2. **Performance**: no garbage collection pauses, no runtime overhead, deterministic cleanup.
3. **Concurrency**: safe handling of multiple threads accessing shared mutable state.

The industry has historically chosen two:

- **C and C++** (pre-smart-pointers): safe-false, fast-true, concurrent-false. Manual memory management is fast and flexible but unsafe. C's rule: "trust the programmer." C++ adds RAII, which is safer, but you must still understand lifetimes and move semantics. Concurrency requires explicit synchronization; data races are undefined behavior.

- **Java, Go, Python**: safe-true, fast-false, concurrent-maybe. Garbage collection eliminates use-after-free and dangling pointers, but costs pauses and memory overhead. Go's concurrency model is elegant (goroutines + channels), but a shared mutable `int` is still subject to data races unless you protect it. Python's GIL makes data races "less common" but not impossible on the C extension layer.

- **Rust**: safe-true, fast-true, concurrent-true. Ownership eliminates memory bugs at compile time with zero runtime cost. The borrow checker prevents data races by construction. Concurrency is safe: threads cannot access unprotected shared mutable state. The cost: a steeper learning curve and some programs that are logically safe become harder to express.

Rust is the first mainstream language to break the trilemma *at scale*.

---

## 65.2 The Core Problem: "Is This Pointer Valid?"

All memory safety bugs reduce to one question: **is this pointer valid right now?**

A pointer is valid if:

1. The memory it points to was allocated (not uninitialized).
2. The memory has not been freed.
3. The memory was allocated in the right way (e.g., not a stack pointer after the frame exits).

If any of these is false, dereferencing is undefined behavior. In the best case, you get a crash. In the worst, silent corruption.

The industry has three approaches to answering "is this pointer valid?":

### Approach 1: The Programmer Tracks It (C)

C says: your problem. You `malloc`, you track what you own, you `free` it. At compile time, the language doesn't know or care. At runtime, neither does the language. If you mess up, undefined behavior, and that's your fault.

```c
int* get_buffer(size_t size) {
    return malloc(size);  // Caller must free this
}

void process(int* data, size_t size) {
    // You do not own this; do not free
    for (size_t i = 0; i < size; i++) {
        data[i] += 1;
    }
}

int main() {
    int* buf = get_buffer(100);
    process(buf, 100);
    free(buf);    // Your responsibility
    // If you forget: memory leak
    // If you double-free: heap corruption
    // If you use-after-free: undefined behavior
}
```

Cost: maximum flexibility, maximum speed, maximum responsibility. Easy to leak and corrupt.

### Approach 2: The Runtime Tracks It (Java, Go, Python)

These languages say: we'll track it for you. You `new` an object; the runtime allocates it on the heap and watches it. When no part of your program can reach it anymore, the runtime frees it.

```java
public class Buffer {
    public int[] data;
    public Buffer(int size) {
        this.data = new int[size];
    }
}

public static void main(String[] args) {
    Buffer buf = new Buffer(100);
    process(buf);
    // When buf goes out of scope, the GC will eventually notice it's unreachable
    // and free it. You never think about it.
}
```

Cost: no leaks (under normal circumstances), no use-after-free, programmer simplicity. The runtime must stop the world periodically to trace and free, causing pauses. You trade latency determinism for safety.

### Approach 3: The Compiler Proves It (Rust)

Rust says: the compiler will prove at compile time that this pointer is valid. You write code that *looks* like you're responsible (like C), but the compiler verifies every access.

```rust
fn get_buffer(size: usize) -> Vec<i32> {
    vec![0; size]  // Vec owns the allocation
}

fn process(data: &[i32]) {
    // You borrow data; compiler proves you cannot access it after the borrow ends
    for i in 0..data.len() {
        println!("{}", data[i]);
    }
}

fn main() {
    let mut buf = get_buffer(100);
    process(&buf);
    // buf still owns the allocation; compiler proves no pointer to it exists
    // When buf goes out of scope, it is freed automatically (by the destructor)
}
```

The compiler tracks:

1. **Ownership**: exactly one variable owns each allocation.
2. **Borrowing**: you can lend `&buf` (shared reference) to multiple parties, or `&mut buf` (exclusive reference) to one party, but not both simultaneously.
3. **Lifetimes**: every reference must not outlive the owner.

If you violate any rule, the compiler rejects the code at compile time. If it compiles, you have no use-after-free, no buffer overflow (in safe code), no data race.

---

## 65.3 Why Ownership At Compile Time Matters

The killer insight: **proof at compile time scales to production.**

### The C/C++ Problem

In C++, you can use smart pointers to *assist* ownership:

```cpp
std::unique_ptr<Buffer> get_buffer(size_t size) {
    return std::make_unique<Buffer>(size);
}

int main() {
    {
        auto buf = get_buffer(100);
        // buf owns the allocation; destructor will free when it goes out of scope
    }
    // safe: buf's destructor ran, memory freed
}
```

But the compiler doesn't *prevent* you from breaking the rules. It is possible to:

1. Take a raw pointer from a smart pointer and use it after the smart pointer is destroyed.
2. Create a reference cycle where two `shared_ptr`s point to each other, and neither ever frees (because each holds a reference count from the other).
3. Move a smart pointer and forget which variable owns it, leading to confusion in a large codebase.

None of these errors are compile-time errors in C++. They are mistakes waiting to happen in code review or in production.

```cpp
// C++ allows this (compiles, but is wrong)
std::unique_ptr<int> ptr = std::make_unique<int>(42);
int* raw = ptr.get();  // Borrow the raw pointer

{
    auto ptr2 = std::move(ptr);  // Transfer ownership; ptr is now empty
}
// ptr2 destroyed; memory freed

std::cout << *raw << std::endl;  // use-after-free! Compiles, crashes at runtime.
```

### The Rust Solution

Rust prevents this at the type level:

```rust
let ptr = Box::new(42);  // ptr owns the allocation
let raw = &ptr;          // borrow as a reference

{
    let ptr2 = ptr;      // move; ptr is no longer usable
}
// ptr2 destroyed; memory freed

println!("{}", *raw);    // COMPILER ERROR: raw outlives ptr
```

The compiler rejects the code because it proves that `raw` cannot outlive `ptr`. The error message is:

```
error[E0505]: cannot move out of `ptr` because it is borrowed
   |
2 | let raw = &ptr;
  |           ---- borrow occurs here
3 | {
4 |     let ptr2 = ptr;
  |               ^^^ move occurs here
5 | }
  | - borrow ends here
```

The compiler is not guessing. It has traced the lifetimes and determined that the borrow is still active when you try to move. You cannot write the buggy code.

---

## 65.4 The Borrow Checker: Rust's Enforcement Mechanism

The borrow checker is a compile-time analysis that enforces three rules:

1. **Exactly one owner**: each value has one variable that owns it. You cannot have two variables that claim ownership of the same allocation.

2. **Shared XOR mutable**: at any moment, you can have many shared borrowers (`&T`) OR one exclusive borrower (`&mut T`), but not both. This prevents data races.

3. **Outliving prevention**: a reference must not outlive the data it points to. The compiler tracks lifetimes (usually implicitly) and rejects code where a reference could outlive its target.

Example: shared borrow (many readers, no writers):

```rust
fn main() {
    let vec = vec![1, 2, 3];
    
    let ref1 = &vec;  // Shared borrow #1
    let ref2 = &vec;  // Shared borrow #2
    
    println!("{:?} {:?}", ref1, ref2);  // Both valid; no conflict
}
```

Example: exclusive borrow (one writer, no readers):

```rust
fn main() {
    let mut vec = vec![1, 2, 3];
    
    let ref1 = &mut vec;  // Exclusive borrow
    ref1.push(4);         // Can mutate
    
    // let ref2 = &vec;    // COMPILER ERROR: cannot borrow as shared while ref1 is active
    
}  // ref1 ends here; shared borrow now allowed
```

Example: preventing data races:

```cpp
// C++: allows data race (compiles, race at runtime)
std::vector<int> vec = {1, 2, 3};
std::thread t1([&vec]() { vec.push_back(4); });  // Write
std::thread t2([&vec]() { std::cout << vec[0]; });  // Read
// Race: t1 reallocates while t2 reads
```

```rust
// Rust: prevents data race (compilation error)
let mut vec = vec![1, 2, 3];
let ref1 = &vec;  // Shared borrow in main thread
std::thread::spawn(move || {
    vec.push_back(4);  // COMPILER ERROR: vec is borrowed by ref1 in main
});
```

The Rust compiler rejects this because it proves the move is invalid: `vec` is borrowed by `ref1` in the main thread, so it cannot be moved into the spawned thread.

---

## 65.5 What Rust Pays For This Safety

Safety is not free. Rust's costs:

### 1. Steeper Learning Curve

The borrow checker is a new mental model. You must learn:

- Ownership transfer and move semantics (similar to C++'s move, but mandatory).
- Borrowing and lifetimes (mostly inferred, but explicit when ambiguous).
- When to use `Box<T>` (heap), `Rc<T>` (reference-counted), `Arc<T>` (atomic reference-counted for threads).
- Interior mutability patterns like `Cell<T>` and `Mutex<T>` (advanced escape hatches).

For a programmer trained in Python or Java, this is a significant shift. For a C++ programmer, it is recognizable but stricter.

### 2. Some Safe Programs Are Hard to Express

The compiler is conservative. It accepts all *unsafe* programs (in the safety sense) and rejects all unsound programs (which would break the safety guarantee). In between, it rejects some programs that are *logically safe* but cannot be statically proven so.

Examples:

**Doubly-linked lists**: In C++, you can write a doubly-linked list with `prev` and `next` pointers. In Rust, this is difficult because both pointers want to own the node, violating the "one owner" rule. The solution: `Rc<RefCell<Node>>` (reference-counted interior-mutable containers), which shifts some checks to runtime.

**Graphs with cycles**: Similar problem. A graph where node A points to B and B points to A cannot be expressed with simple ownership. You need `Rc` + `Weak` pointers (Rust's version of `weak_ptr`).

**Self-referential structs**: A struct that contains a pointer to itself is impossible in Rust without `unsafe` code. In C++, it is straightforward.

```cpp
// C++ is easy
struct Node {
    int data;
    Node* next;
};
```

```rust
// Rust: impossible to express safely without unsafe
struct Node {
    data: i32,
    // next: &Node,  // ERROR: self-referential; lifetime is ambiguous
}

// Workaround: heap + reference counting
struct Node {
    data: i32,
    next: Option<Rc<RefCell<Node>>>,  // Verbose, and Rc has runtime overhead
}
```

### 3. Compile Times

Rust's compiler does significant lifetime inference and borrow checking. For large codebases, compile times can exceed C++. This matters in rapid iteration workflows.

### 4. Boilerplate for Library APIs

When you write a function that takes a reference to data and returns a reference, the compiler often needs you to annotate lifetimes explicitly:

```rust
// The compiler cannot infer that the returned reference is to the input
fn get_first<'a>(vec: &'a Vec<i32>) -> &'a i32 {
    //                  ^a             ^a
    // The 'a says: "the returned reference lives as long as vec"
    &vec[0]
}
```

In many cases, the compiler infers lifetimes. But for APIs that are ambiguous (multiple input references), you must annotate. This is not hard, but it is visible boilerplate.

---

## 65.6 Comparison: Rust vs. C++ vs. GC vs. Python

| Language | Memory Safety | Performance | Concurrency | Learning Curve | When to Use |
|----------|---|---|---|---|---|
| **C** | No | Fast | Hard (manual locking) | Low (for systems programming) | Embedded, OS kernels, when you need maximum control |
| **C++** | Partial (smart pointers + RAII) | Fast | Hard (threads + locks, or async) | Medium-high | Systems programming, performance-critical code, long-lived projects with discipline |
| **Rust** | Yes | Fast | Yes (safe by construction) | High (borrow checker) | New projects where memory safety is non-negotiable; embedded; performance-critical; concurrent systems |
| **Java/Kotlin** | Yes (no manual memory) | Moderate (GC pauses) | Partial (data races possible, but easier to avoid) | Medium | Enterprise applications, rapid iteration, teams with varied experience |
| **Go** | Yes | Good (GC is fast but still pauses) | Good (goroutines + channels) | Low | Cloud infrastructure, microservices, applications where GC pause is acceptable |
| **Python** | Partial (GC, but C extensions can segfault) | Slow | Hard (GIL limits true parallelism) | Low | Data science, rapid iteration, scripting, business logic |

---

## 65.7 Worked Example: Use-After-Free in Four Languages

This program has a subtle use-after-free bug. Let's see how each language responds.

### The Bug: A Cache That Returns References

We want a cache that stores strings and returns them by reference. The bug: the cache may need to evict entries. When it does, any outstanding reference becomes invalid.

### C

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char* keys[10];
    char* values[10];
    int size;
} Cache;

const char* cache_get(Cache* c, const char* key) {
    for (int i = 0; i < c->size; i++) {
        if (strcmp(c->keys[i], key) == 0) {
            return c->values[i];  // Return reference to internal value
        }
    }
    return NULL;
}

void cache_set(Cache* c, const char* key, const char* value) {
    if (c->size >= 10) {
        // Simple eviction: remove first entry
        free(c->keys[0]);
        free(c->values[0]);  // BUG: if caller holds reference to this, it's now dangling
        for (int i = 0; i < c->size - 1; i++) {
            c->keys[i] = c->keys[i + 1];
            c->values[i] = c->values[i + 1];
        }
        c->size--;
    }
    c->keys[c->size] = malloc(strlen(key) + 1);
    strcpy(c->keys[c->size], key);
    c->values[c->size] = malloc(strlen(value) + 1);
    strcpy(c->values[c->size], value);
    c->size++;
}

int main() {
    Cache c = {0};
    cache_set(&c, "name", "Alice");
    
    const char* ref = cache_get(&c, "name");  // ref points to internal storage
    
    // Now fill the cache to trigger eviction
    for (int i = 0; i < 10; i++) {
        char key[20], val[20];
        sprintf(key, "key%d", i);
        sprintf(val, "val%d", i);
        cache_set(&c, key, val);
    }
    
    printf("%s\n", ref);  // use-after-free: ref points to freed memory
    return 0;
}
```

**Result**: undefined behavior. The program may crash, print garbage, or appear to work. The compiler does not warn; the bug is silent.

### C++

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

class Cache {
public:
    const std::string& get(const std::string& key) {
        return data.at(key);  // Return reference to map's internal value
    }
    
    void set(const std::string& key, const std::string& value) {
        if (data.size() >= 10) {
            // Eviction might rehash; references become invalid
        }
        data[key] = value;  // This may trigger rehash
    }
    
private:
    std::unordered_map<std::string, std::string> data;
};

int main() {
    Cache c;
    c.set("name", "Alice");
    
    const std::string& ref = c.get("name");  // Reference to map's internal value
    
    // Trigger evictions
    for (int i = 0; i < 10; i++) {
        c.set("key" + std::to_string(i), "val" + std::to_string(i));
    }
    
    std::cout << ref << std::endl;  // use-after-free; ref is dangling
    return 0;
}
```

**Result**: undefined behavior. The compiler does not prevent this. If you compile with AddressSanitizer (`-fsanitize=address`), it catches the error at runtime. Otherwise, it may silently corrupt memory.

### Java

```java
import java.util.HashMap;

public class Cache {
    private HashMap<String, String> data = new HashMap<>();
    
    public String get(String key) {
        return data.get(key);  // Return a reference (to an object on the heap)
    }
    
    public void set(String key, String value) {
        data.put(key, value);  // May trigger rehash, but GC keeps the string alive
    }
    
    public static void main(String[] args) {
        Cache c = new Cache();
        c.set("name", "Alice");
        
        String ref = c.get("name");  // Reference to the string object
        
        // Trigger evictions/rehashes
        for (int i = 0; i < 10; i++) {
            c.set("key" + i, "val" + i);
        }
        
        System.out.println(ref);  // SAFE: GC ensures the string is still alive
    }
}
```

**Result**: safe. The `String` object is kept alive by the GC. Even if the HashMap rehashes and the reference within the map changes, the string object itself is not freed because `ref` still references it.

### Rust

```rust
use std::collections::HashMap;

struct Cache {
    data: HashMap<String, String>,
}

impl Cache {
    fn get(&self, key: &str) -> &str {
        &self.data[key]  // Return reference to string inside map
    }
    
    fn set(&mut self, key: String, value: String) {
        self.data.insert(key, value);  // May rehash
    }
}

fn main() {
    let mut c = Cache { data: HashMap::new() };
    c.set("name".to_string(), "Alice".to_string());
    
    let ref1 = c.get("name");  // Borrow from c
    
    // Try to trigger eviction
    for i in 0..10 {
        c.set(format!("key{}", i), format!("val{}", i));
        // COMPILER ERROR: cannot borrow c as mutable while ref1 is borrowed
    }
    
    println!("{}", ref1);  // If it compiled, this would be safe
}
```

**Result**: compilation error. The compiler rejects the code because `c` is borrowed immutably by `ref1`, and then you try to borrow it mutably (via `set`) while the immutable borrow is still active. The error:

```
error[E0502]: cannot borrow `c` as mutable because it is also borrowed as immutable
   |
5 | let ref1 = c.get("name");
  |            - immutable borrow occurs here
8 |         c.set(format!("key{}", i), format!("val{}", i));
  |         ^ mutable borrow occurs here
```

The fix: drop the reference before calling `set`, or redesign the API to return owned data:

```rust
fn get(&self, key: &str) -> String {
    self.data[key].clone()  // Return owned copy
}
```

---

## 65.8 When Rust Is the Wrong Tool

Rust is powerful but not always the right choice:

1. **Rapid iteration and prototyping**: if your deadline is two weeks and you have never used Rust, Python or Go is faster. You will spend time fighting the borrow checker instead of building features.

2. **Team expertise**: if your team knows Java and Go well, and has never used Rust, the ramp-up time is significant. Rust has a steep learning curve; a junior engineer will be less productive for months.

3. **Hyper-dynamic data**: if your program allocates and deallocates thousands of structures per second with complex ownership patterns (financial tick data, game engine AI), the borrow checker may force awkward workarounds (`Rc<RefCell<T>>`), which have runtime overhead anyway.

4. **Rapid changes to APIs**: Rust's type system is strict. Changes to function signatures (e.g., adding a parameter to a callback) require updates throughout the call graph. For exploratory code, this is friction.

5. **Interop with dynamic languages**: if you must call Python, Lua, or JavaScript from Rust, the interop overhead is significant. For a dynamic scripting engine, a dynamic language is usually simpler.

6. **Very large existing codebase**: migrating a 10-million-line C++ codebase to Rust is not practical. Gradual adoption (Rust modules calling C++ via FFI) is possible but complex.

**In these cases, C++ with smart pointers and discipline, Go, or Java is often the better choice.**

---

## 65.9 Common Misconceptions

1. **"Rust is faster than C++"** — No. Both have zero-cost abstractions. The difference is compile-time vs. runtime checks. In practice, Rust and optimized C++ have similar performance. Rust's win is *safety*, not speed. In rare cases, the borrow checker prevents a more efficient design (e.g., you must clone instead of borrowing), so Rust can be slower. In other cases, Rust's guarantees enable optimizations that C++ cannot safely make.

2. **"The borrow checker is too strict; I can't write real code in Rust"** — The borrow checker is strict about *safe* code. For ~95% of programs, safe Rust is natural. For the remaining 5%, you write `unsafe` blocks (with care and documentation). Rust gives you the safety by default and an escape hatch for genuine need.

3. **"Rust prevents all memory bugs"** — Safe Rust prevents use-after-free, buffer overflow, and data races. It does not prevent logic errors, deadlocks, or resource leaks (in the sense of forgetting to close a file). It prevents the *memory* family of bugs.

4. **"I should always use `Rc<RefCell<T>>` for flexibility"** — `Rc<RefCell<T>>` shifts the borrow checker to runtime, recovering some flexibility but losing compile-time guarantees. Use it sparingly, only when the borrow checker cannot express your design. Most code should use simple ownership.

5. **"Rust has no garbage collection, so it must be faster"** — Rust's lack of GC means no pauses, which is good for latency. But GC languages (Go, Java) are often faster for throughput because the GC can compact memory and the CPU can be fully utilized. The lack of GC is about predictability, not raw speed.

6. **"Lifetimes are complicated; I should always write explicit lifetime annotations"** — Most lifetimes are inferred. Only complex function signatures need explicit annotations. If you find yourself writing `'a` everywhere, you probably have a design that the borrow checker doesn't like. Redesign first; annotate second.

---

## 65.10 Exercises

1. **Identify the tradeoff.** Given a task (e.g., "implement a web server," "write a game engine," "build a command-line tool"), choose one of: Rust, C++, Java, Go, Python. Justify each choice with 2–3 sentences on memory safety, performance, team expertise, and time to delivery.

2. **Rewrite in Rust.** Take the C++ Cache example above (the unsafe one) and rewrite it in Rust safely. Your implementation must pass compilation. Compare your design to the C++ version: what did you change? Why?

3. **Spot the lifetime issue.** Given a Rust function signature like:
   ```rust
   fn get_data(cache: &Cache, key: &str) -> &str { ... }
   ```
   Determine: is this safe? If the cache can rehash (evicting entries), why would it be unsafe? What would the borrow checker report if you tried to call `cache.insert()` after getting the reference?

4. **Compare GC vs. ownership.** Write the same program (a simple linked list or tree traversal) in Java and Rust. Measure runtime and memory. Which is faster? Why? (In most cases, Rust will be faster and use less memory, but GC can be competitive for single-iteration benchmarks.)

5. **Understand the constraint.** Read about Rust's `PhantomData`, `Rc<T>`, `Arc<T>`, `RefCell<T>`, and `Mutex<T>`. For each, write one sentence explaining when you'd use it and what borrow checker rule it helps you work around.

6. **Design for the borrow checker.** Sketch an API for a doubly-linked list in Rust. Explain why simple pointers don't work and what data structure (e.g., `Rc<RefCell<Node>>`, index-based, arena allocator) you'd choose. What are the tradeoffs?

---

## 65.11 Summary

Rust's ownership model solves the fundamental problem of memory safety: **proving at compile time that a pointer will never be dereferenced after the memory it points to is freed.** This proof comes with three rules: exactly one owner, shared XOR mutable borrowing, and lifetimes that prevent outliving.

The industry has historically chosen two of three: memory safety, performance, and concurrency. C/C++ chose performance and concurrency at the cost of safety. Java chose safety and concurrency at the cost of performance (GC pauses). Rust chose all three by enforcing ownership at compile time.

The cost is steep: Rust's learning curve is high, and some programs that are logically safe are difficult to express. Patterns like graphs, doubly-linked lists, and interior mutability require reaching for runtime checks or `unsafe` code. But for systems where memory safety is non-negotiable — operating systems, databases, compilers, embedded firmware — Rust is increasingly the right choice.

C++ with smart pointers and discipline can approach Rust's safety, but the burden is on the programmer. The compiler in C++ is your assistant; in Rust, it is your enforcer. For large teams and long-lived codebases where you cannot afford a single memory bug, Rust's enforcement is worth the friction.

The next chapter, Why C++ Is Difficult, explains why C++ remains relevant despite Rust's safety: flexibility, expressiveness, and a 40-year ecosystem that Rust has not yet matched.

---

> **[← Previous: Runtime GC Tradeoffs Across Languages](07-runtime-gc-tradeoffs.md)**  ·  **[↑ Part 6](README.md)**  ·  **[Next: Why C++ Is Difficult →](09-why-cpp-is-difficult.md)**
