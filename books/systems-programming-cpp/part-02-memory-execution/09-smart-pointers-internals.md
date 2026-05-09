# Chapter 9 — Smart Pointers Internals

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a smart pointer is: **a stack object that owns a heap object, using RAII to free the heap object when the stack object dies**.
2. Distinguish `unique_ptr` (sole ownership, zero overhead, move-only) from `shared_ptr` (shared ownership, reference-counted, non-movable without care).
3. Understand the **control block**: what it contains, how `make_shared` fuses allocations, and why shared ownership has a cost.
4. Recognize why two `shared_ptr` objects can form a cycle that leaks, and how `weak_ptr` breaks it by being non-owning.
5. Explain **`enable_shared_from_this`** and when you need it: creating a `shared_ptr` to `this` after construction.
6. Reason about the cost model: `unique_ptr` is free; `shared_ptr` incurs atomic refcount operations and cache-line bouncing under contention.
7. Write a minimal `MyUniquePtr` and `MySharedPtr` with a control block, and explain why their sizes differ.

This chapter is where the abstraction of "automatic memory management" meets the machinery underneath. Smart pointers are RAII applied to memory: by binding a heap object's lifetime to a stack object's lifetime, we automate what used to be manual `new` and `delete`. Understanding how they work is essential for writing safe, efficient C++.

---

## 9.1 The Core Idea: RAII Applied to Memory

Recall from Chapter 5 that RAII is a contract: **the constructor acquires a resource, the destructor releases it, and the language guarantees the destructor runs**. A smart pointer applies this contract to heap objects.

```cpp
int* p = new int(42);      // manual: must remember to delete
delete p;                  // must happen, and only once

std::unique_ptr<int> up(new int(42));  // automatic: delete happens in ~unique_ptr()
// p goes out of scope; destructor runs; heap int is freed
```

The type `unique_ptr<int>` is a stack object — 8 bytes on 64-bit, just like a raw pointer. The destructor calls `delete` on the stored pointer. When `up` dies, so does the heap int.

That's the entire abstraction: move the responsibility for deletion from the programmer to the type system.

---

## 9.2 unique_ptr — Zero-Overhead Exclusive Ownership

A `unique_ptr<T>` is a pointer you own exclusively. It is **move-only**: you can transfer ownership by moving, but you cannot copy.

```cpp
std::unique_ptr<int> p1(new int(42));
std::unique_ptr<int> p2 = std::move(p1);  // p1 is now empty; p2 owns the int
// p2 goes out of scope; the int is deleted
// p1 goes out of scope; it was already empty, so nothing happens
```

### Size and Layout

`unique_ptr<T>` is typically **exactly the size of a raw pointer**: 8 bytes. It stores one `T*` and nothing else. The compiler removes all checks.

```cpp
std::unique_ptr<int> up;
assert(sizeof(up) == sizeof(int*));    // true
```

This is the "zero overhead" promise: you get automatic ownership without paying for it at runtime. Compare to `shared_ptr`, which is typically 16 bytes (two pointers: the object pointer and the control block pointer).

### Release and Reset

You can **release ownership** (get the raw pointer back, relinquishing the obligation to delete):

```cpp
int* p = up.release();     // up is now null; you must delete p
delete p;                  // your responsibility now
```

Or **reset** to a different pointer:

```cpp
up.reset(new int(50));     // deletes the old int, stores the new one
up.reset();                // deletes the int, stores nullptr
```

### Custom Deleters

Not all pointers are freed by `delete`. C APIs return pointers to be freed by special functions:

```cpp
FILE* f = fopen("data.txt", "r");
// ... use f
fclose(f);
```

With a custom deleter, `unique_ptr` handles this:

```cpp
std::unique_ptr<FILE, decltype(&fclose)> f(fopen("data.txt", "r"), &fclose);
// f goes out of scope; fclose(f.get()) is called automatically
```

The deleter is part of the template type (`decltype(&fclose)`), so it's bound at compile time with **zero runtime overhead**. The custom deleter is often an empty class, and the compiler inlines the call.

### Array Specialization

`unique_ptr<T[]>` is a specialization for arrays:

```cpp
std::unique_ptr<int[]> arr(new int[100]);
int x = arr[5];     // operator[] instead of operator*
// destructor calls delete[], not delete
```

### Move Semantics and Efficiency

`unique_ptr` is movable but not copyable. This enforces the invariant: at any moment, exactly one `unique_ptr` owns the heap object.

```cpp
std::unique_ptr<int> factory() {
    return std::make_unique<int>(42);  // constructs and returns
}

std::unique_ptr<int> p = factory();    // move constructor, no copy
```

The move constructor simply transfers the pointer:

```cpp
template <typename T>
class unique_ptr {
    T* ptr_;
public:
    unique_ptr(unique_ptr&& other) noexcept : ptr_(other.release()) {}
    // other.ptr_ is now nullptr
};
```

That's it. No atomic operations, no reference counting. Just transfer the pointer.

---

## 9.3 shared_ptr — Reference-Counted Shared Ownership

A `shared_ptr<T>` is a pointer to an object that may be owned by multiple `shared_ptr` instances. When the last one dies, the object is deleted.

The mechanism: **atomic reference counting**. Every time you copy a `shared_ptr`, the reference count increments. Every time one dies, the count decrements. When it reaches zero, the object is deleted.

### The Control Block

`shared_ptr` is not just a pointer. It holds **two pointers**:

```cpp
template <typename T>
class shared_ptr {
    T* ptr_;               // pointer to the object
    ControlBlock* cb_;     // pointer to the reference count and deleter
public:
    // ...
};
```

The **control block** is a heap structure that holds:

```cpp
struct ControlBlock {
    std::atomic<int> strong_count;   // number of shared_ptr instances
    std::atomic<int> weak_count;     // number of weak_ptr instances, plus 1
    Deleter* deleter;                // how to free the object
    Allocator* allocator;            // how to free this control block itself
};
```

This is why `shared_ptr` is typically 16 bytes: one `T*` (8) and one `ControlBlock*` (8).

### Copy and Destruction

When you copy a `shared_ptr`:

```cpp
std::shared_ptr<int> p1(new int(42));    // strong_count = 1
std::shared_ptr<int> p2 = p1;            // strong_count = 2
std::shared_ptr<int> p3 = p1;            // strong_count = 3
```

Each copy increments the **atomic** `strong_count`. When you destroy a `shared_ptr`:

```cpp
{
    std::shared_ptr<int> p = ...;       // strong_count = ?
}   // p destructor runs, decrements strong_count
```

The destructor does:

```cpp
template <typename T>
shared_ptr<T>::~shared_ptr() {
    if (cb_ && cb_->strong_count.fetch_sub(1, std::memory_order_release) == 1) {
        // We were the last owner
        cb_->deleter(ptr_);   // delete the int
        // Decrement weak_count too...
        // If weak_count reaches 1 (the control block itself), free the control block
    }
}
```

The atomic decrement uses `fetch_sub` with `memory_order_release` to ensure visibility across threads. The memory barrier is the cost: cache-line bouncing when multiple threads compete.

### make_shared

If you construct a `shared_ptr` naively, you allocate twice: once for the object, once for the control block.

```cpp
std::shared_ptr<int> p(new int(42));  // two allocations
```

`make_shared` fuses them into one:

```cpp
std::shared_ptr<int> p = std::make_shared<int>(42);  // one allocation
```

Internally, `make_shared` allocates a structure that contains both the control block *and* the object, in a single heap block. This is faster and uses less memory.

**ASCII diagram** (memory layout with `make_shared`):

```
Heap allocation (single block)
+----------------------------------+
| ControlBlock (16 bytes)          |
|  - strong_count: 1               |
|  - weak_count: 1                 |
|  - deleter pointer               |
|  - allocator pointer             |
+----------------------------------+
| int value: 42                    |
+----------------------------------+
```

The `shared_ptr<int>` holds:
- `ptr_` = address of the int (inside the allocation)
- `cb_` = address of the ControlBlock (start of the allocation)

Both point into the same heap block. On destruction, the deleter knows to free the whole block (not just the int).

### Atomic Cost

The atomic operations on `strong_count` are **not free**. On x86:

```
fetch_sub(1, memory_order_release):
  mov   rax, [rsi]          ; load from control block
  lock  dec qword [rsi]     ; atomic decrement, memory barrier
  cmp   rax, 1              ; check if we were last
```

The `lock` prefix forces a read-modify-write cycle on the cache line. If multiple threads are decrementing the same `shared_ptr` (sharing the same control block), the cache line bounces between cores. Under contention, this is measurable overhead.

In single-threaded code or when the `shared_ptr` is copied across thread boundaries rarely, the cost is negligible. In code where many threads frequently increment and decrement the same `shared_ptr`, the cost can dominate.

---

## 9.4 weak_ptr — Breaking Cycles

Consider a graph where each node holds `shared_ptr` to its children:

```cpp
struct Node {
    int value;
    std::shared_ptr<Node> left;
    std::shared_ptr<Node> right;
    // no parent pointer yet
};

auto root = std::make_shared<Node>();
auto child = std::make_shared<Node>();
root->left = child;
// Now child is owned by root (strong_count = 1)
// root is still owned by our stack variable (strong_count = 1)
```

What if the child needs to point back to its parent?

```cpp
struct Node {
    int value;
    std::shared_ptr<Node> left;
    std::shared_ptr<Node> right;
    std::shared_ptr<Node> parent;  // back-pointer
};

auto root = std::make_shared<Node>();
auto child = std::make_shared<Node>();
root->left = child;
child->parent = root;    // CYCLE
```

Now we have a cycle:
- `root`'s strong_count = 2 (the stack variable and `child->parent`)
- `child`'s strong_count = 1 (the `root->left`)

When the stack variable goes out of scope:

```cpp
{
    auto root = std::make_shared<Node>();
    auto child = std::make_shared<Node>();
    root->left = child;
    child->parent = root;
}
// root destructor runs:
//   root->strong_count.fetch_sub(1) => now 1 (not zero!)
//   root is not deleted because child->parent still holds it
// child destructor runs:
//   child->strong_count.fetch_sub(1) => now 0
//   child is deleted, but child->parent is destroyed by the destructor
//   root->strong_count.fetch_sub(1) => now 0
//   root is deleted
// But wait: at what point is child deleted?
```

Actually, this scenario *does* work — but it's fragile. If we add more complexity (the child holds a `shared_ptr` to its right child, which holds a `shared_ptr` to its parent), the cycle can persist.

**The problem:** two `shared_ptr` objects can point to each other, holding each other's reference count above zero, preventing either from being deleted. This is a memory leak, even though you're using `shared_ptr`.

**The solution:** use `weak_ptr`.

A `weak_ptr<T>` is a non-owning observer of a `shared_ptr<T>`. It does **not** increment the strong count; instead, it increments a **weak count**.

```cpp
struct Node {
    int value;
    std::shared_ptr<Node> left;
    std::shared_ptr<Node> right;
    std::weak_ptr<Node> parent;   // non-owning back-pointer
};

auto root = std::make_shared<Node>();
auto child = std::make_shared<Node>();
root->left = child;
child->parent = root;    // weak_ptr assignment; weak_count++, but not strong_count
```

Now:
- `root`'s strong_count = 1 (just the stack variable)
- `child`'s strong_count = 1 (just `root->left`)
- `child->parent`'s weak_count = 1 (observing root)

When the stack variable dies:

```cpp
{
    auto root = std::make_shared<Node>();
    auto child = std::make_shared<Node>();
    root->left = child;
    child->parent = root;
}
// root goes out of scope; strong_count.fetch_sub(1) => 0
// root is deleted immediately
// child->parent is now dangling (the weak_ptr sees "expired")
// child goes out of scope; strong_count.fetch_sub(1) => 0
// child is deleted
```

The weak pointer doesn't prevent deletion. But you can **upgrade** it to a shared pointer if the object is still alive:

```cpp
if (auto parent = child->parent.lock()) {
    // parent is a shared_ptr<Node>; we have shared ownership for this scope
    std::cout << parent->value << "\n";
} else {
    // the original parent was deleted
    std::cout << "parent is gone\n";
}
```

The `.lock()` method atomically checks if the strong_count is still nonzero. If yes, it increments the count and returns a valid `shared_ptr`. If no, it returns a null `shared_ptr`.

**When to use `weak_ptr`:**

- **Parent pointers in trees**: children own parents (via `shared_ptr`), parents observe children (via `weak_ptr`).
- **Observer patterns**: the subject holds `shared_ptr` to listeners; listeners hold `weak_ptr` to the subject to avoid cycles.
- **Caches**: the cache holds `weak_ptr` to cached objects; if the object is deleted elsewhere, the cache entry silently expires.
- **Callbacks**: a callback mechanism where the callback holder doesn't own the target; if the target is deleted, the callback is safe to ignore.

---

## 9.5 enable_shared_from_this

There's a footgun in C++: you cannot safely construct a `shared_ptr` to `this` inside a member function.

```cpp
struct Node {
    std::shared_ptr<Node> get_self() {
        return std::shared_ptr<Node>(this);  // WRONG
    }
};

auto n = std::make_shared<Node>();
auto self = n->get_self();
// Now we have TWO shared_ptr instances:
//   n: ptr_ = address of Node, cb_ = some control block
//   self: ptr_ = address of Node, cb_ = DIFFERENT control block!
// They think they own the same object but have separate reference counts.
// When self dies, it deletes the Node.
// When n dies, it tries to delete a dead Node. Boom.
```

The issue: `std::make_shared<Node>()` creates *one* control block. But `std::shared_ptr<Node>(this)` creates a *second* control block, because it has no way to know that the object already has a control block.

The fix: **inherit from `std::enable_shared_from_this<T>`**.

```cpp
struct Node : std::enable_shared_from_this<Node> {
    std::shared_ptr<Node> get_self() {
        return shared_from_this();  // correct
    }
};

auto n = std::make_shared<Node>();
auto self = n->get_self();
// Both n and self point to the same control block.
```

### How enable_shared_from_this Works

When you construct a `shared_ptr<T>` where `T` inherits from `enable_shared_from_this<T>`, the `shared_ptr` constructor detects this and stashes the control block pointer inside the `enable_shared_from_this` base object.

```cpp
template <typename T>
class enable_shared_from_this {
    mutable std::weak_ptr<T> weak_this_;
    // stashed by shared_ptr constructor
};
```

Later, `shared_from_this()` uses that stashed weak pointer to create a new `shared_ptr` with the *correct* control block:

```cpp
template <typename T>
std::shared_ptr<T> enable_shared_from_this<T>::shared_from_this() {
    return std::shared_ptr<T>(weak_this_);
}
```

This is a `shared_ptr` constructor that takes a `weak_ptr`. It increments the strong count of the *existing* control block, not a new one.

**When to use `enable_shared_from_this`:**

- You have a class whose instances are always managed by `shared_ptr`.
- Inside a member function, you need to hand out a `shared_ptr` to yourself.
- You are implementing a callback or observer pattern where the object must register itself.

---

## 9.6 Cost Model

### unique_ptr: Free

`unique_ptr<T>` has **zero runtime cost** over a raw pointer:

- **Size**: 8 bytes (one pointer). Possible compiler optimization: if the deleter is the default `delete`, the compiler may deduplicate the `delete` instruction across multiple deleters.
- **Copy / move**: move is trivial (transfer the pointer); copy is deleted (compile-time error).
- **Destruction**: one `delete` call, inlined.

In optimized builds, `unique_ptr` is as efficient as manual `new` / `delete`, except the deletion is *guaranteed* to happen.

### shared_ptr: Not Free

`shared_ptr<T>` has **measurable costs**:

- **Size**: 16 bytes (two pointers: object and control block).
- **Copy**: increments atomic counter. Load + atomic increment + memory barrier. ~10–20 CPU cycles in uncontended code.
- **Destruction**: atomic decrement + memory barrier. May trigger object deletion (additional destructor call). May trigger control block deallocation.
- **Cache pressure**: the control block is a separate heap allocation, potentially in a different cache line than the object.

**When shared_ptr cost matters:**

- **Single-threaded, high-frequency copying**: if you're copying `shared_ptr` thousands of times per second and threads are not involved, you pay the atomic cost unnecessarily. Consider whether you actually need shared ownership.
- **Contended reference counts**: if multiple threads frequently copy and destroy the same `shared_ptr` (the same control block), the atomic operations cause cache-line bouncing, serializing increments across cores.
- **Per-element overhead**: storing `shared_ptr<T>` in a large vector and copying the vector is expensive. Consider storing `T*` pointers if ownership is elsewhere.

**When shared_ptr cost is negligible:**

- **Moderate copying**: if you copy a `shared_ptr` a few times and then destroy it, the overhead is dwarfed by the work it does.
- **Thread-isolated pointers**: if each thread has its own `shared_ptr` to an object (not shared), there is no contention, and the atomic operations are as fast as regular memory operations.
- **make_shared**: using `make_shared` fuses allocations, reducing heap fragmentation and improving cache locality compared to `new` + `shared_ptr(new ...)`.

---

## 9.7 Minimal Implementation

To understand smart pointers, write minimal versions.

### MyUniquePtr

```cpp
template <typename T>
class MyUniquePtr {
    T* ptr_;

public:
    MyUniquePtr(T* p = nullptr) : ptr_(p) {}
    
    ~MyUniquePtr() {
        delete ptr_;
    }

    // Move-only semantics
    MyUniquePtr(MyUniquePtr&& other) noexcept : ptr_(other.release()) {}
    
    MyUniquePtr& operator=(MyUniquePtr&& other) noexcept {
        reset(other.release());
        return *this;
    }

    // Deleted copy
    MyUniquePtr(const MyUniquePtr&) = delete;
    MyUniquePtr& operator=(const MyUniquePtr&) = delete;

    T* get() const { return ptr_; }
    T* release() {
        T* tmp = ptr_;
        ptr_ = nullptr;
        return tmp;
    }

    void reset(T* p = nullptr) {
        delete ptr_;
        ptr_ = p;
    }

    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
};
```

Size:
```cpp
assert(sizeof(MyUniquePtr<int>) == sizeof(int*));  // 8 bytes
```

### MySharedPtr

```cpp
template <typename T>
class MyControlBlock {
public:
    std::atomic<int> strong_count;
    std::atomic<int> weak_count;

    MyControlBlock() : strong_count(1), weak_count(0) {}
    virtual ~MyControlBlock() = default;
    virtual void delete_object() = 0;
};

template <typename T>
class MyConcreteControlBlock : public MyControlBlock<T> {
    T* ptr_;
public:
    MyConcreteControlBlock(T* p) : ptr_(p) {}
    void delete_object() override {
        delete ptr_;
    }
};

template <typename T>
class MyWeakPtr;  // forward declare

template <typename T>
class MySharedPtr {
    T* ptr_;
    MyControlBlock<T>* cb_;

    friend class MyWeakPtr<T>;

public:
    MySharedPtr(T* p = nullptr) {
        if (p) {
            cb_ = new MyConcreteControlBlock<T>(p);
            ptr_ = p;
        } else {
            cb_ = nullptr;
            ptr_ = nullptr;
        }
    }

    MySharedPtr(const MySharedPtr& other) : ptr_(other.ptr_), cb_(other.cb_) {
        if (cb_) cb_->strong_count++;
    }

    MySharedPtr& operator=(const MySharedPtr& other) {
        if (this != &other) {
            if (cb_) {
                cb_->strong_count--;
                if (cb_->strong_count == 0) {
                    cb_->delete_object();
                }
            }
            ptr_ = other.ptr_;
            cb_ = other.cb_;
            if (cb_) cb_->strong_count++;
        }
        return *this;
    }

    ~MySharedPtr() {
        if (cb_) {
            cb_->strong_count--;
            if (cb_->strong_count == 0) {
                cb_->delete_object();
                // For simplicity, we leak the control block itself
                // Real std::shared_ptr also decrements weak_count
            }
        }
    }

    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get() const { return ptr_; }
};
```

Size:
```cpp
assert(sizeof(MySharedPtr<int>) == 2 * sizeof(int*));  // 16 bytes
```

### Comparison

```cpp
MyUniquePtr<int> u(new int(42));
assert(sizeof(u) == 8);   // Just the pointer

MySharedPtr<int> s(new int(42));
assert(sizeof(s) == 16);  // Pointer + control block pointer

MySharedPtr<int> s2 = s;  // Increments strong_count
// s and s2 both own the int; it's deleted when both are destroyed
```

The size difference is fundamental: `unique_ptr` needs only a pointer; `shared_ptr` needs two (object + control block).

---

## 9.8 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Smart pointers are for beginners; serious code uses raw pointers." | Smart pointers are how production C++ is written. Raw pointers are for non-owning, optional references only. |
| "`shared_ptr` solves all memory management problems." | `shared_ptr` has overhead and does not solve cycles. Default to `unique_ptr`; use `shared_ptr` only when ownership is genuinely shared. |
| "I can create a `shared_ptr` to `this` in any member function." | You cannot, unless the class inherits from `enable_shared_from_this`. Doing so creates two control blocks and corrupts the reference count. |
| "`weak_ptr` is slow because it dereferences pointers." | `weak_ptr` is as fast as `shared_ptr`. The `.lock()` call is a single atomic read; it is not slow. |
| "If I move a `shared_ptr`, it's faster because there's no copy." | Moving a `shared_ptr` still increments the strong count (in the move constructor). Move is a no-op atomic increment; copy is a no-op atomic increment. They have the same cost. |
| "The control block is allocated separately; I can't control where." | You can use `make_shared`, which allocates the control block and object together in one allocation. This is the preferred idiom. |
| "`delete[]` is required for arrays; `unique_ptr<T[]>` is not necessary." | `unique_ptr<T[]>` is essential. A raw `delete` on a `new[]` pointer is undefined behavior. Always use the array specialization. |

---

## 9.9 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| Raw pointer (`T*`) | No overhead; familiar syntax; required for C interop. | No ownership info; manual lifetime; easy to create dangling pointers. | Non-owning, optional references (e.g., "maybe points to something alive elsewhere"). |
| `unique_ptr<T>` | Zero overhead; sole ownership encoded in type; move semantics; safe by construction. | Move-only; cannot share ownership without transferring it; slightly verbose (must `std::move`). | Exclusive ownership; factory functions; PIMPL. |
| `shared_ptr<T>` | Shared ownership; safe lifetimes; reference counting automatic. | 16-byte overhead; atomic operations; does not prevent cycles; thread-safety cost under contention. | Multiple owners of the same object; returning ownership to caller; cache entries. |
| `weak_ptr<T>` | Non-owning observer; breaks cycles; detects expiration. | More verbose (must `.lock()`); mental model requires understanding shared/weak duality. | Parent pointers in trees; observer patterns; caches; callbacks. |
| Intrusive refcount | Single allocation; no separate control block; works without std library. | Requires code in the pointee; not composable (can't add refcount to a third-party class). | Performance-critical systems; embedded; when you control the object type. |
| Manual `new`/`delete` | Total control; required for certain C patterns. | Dangling pointers; leaks; use-after-free; every function that creates an object must document ownership. | Legacy code; very rare cases where you need manual control over timing. |

---

## 9.10 Worked Example: A Cache with Weak Pointers

Here's a realistic use case: a cache that does not prevent garbage collection.

```cpp
template <typename Key, typename Value>
class WeakCache {
    std::unordered_map<Key, std::weak_ptr<Value>> cache_;

public:
    std::shared_ptr<Value> get(const Key& k) {
        auto it = cache_.find(k);
        if (it != cache_.end()) {
            if (auto val = it->second.lock()) {
                // Cache hit; object still alive
                return val;
            } else {
                // Object was deleted; remove stale entry
                cache_.erase(it);
            }
        }
        return nullptr;
    }

    void insert(const Key& k, std::shared_ptr<Value> val) {
        cache_[k] = val;  // stores weak_ptr; does not extend val's lifetime
    }

    void clear() {
        cache_.clear();
    }
};
```

Usage:

```cpp
WeakCache<std::string, std::string> cache;

{
    auto value = std::make_shared<std::string>("expensive data");
    cache.insert("key1", value);
    
    auto cached = cache.get("key1");
    assert(cached && *cached == "expensive data");
}  // value goes out of scope; the string is deleted

auto missing = cache.get("key1");
assert(missing == nullptr);  // cache entry expired; was removed by lock() failure
```

The cache holds `weak_ptr` to values. When a value is deleted elsewhere, the weak pointer expires. The next cache access detects this and cleans up. The cache does not prevent cleanup; it is merely an *observer*.

---

## 9.11 Exercises

1. **Size and layout.** Write a program that prints `sizeof(unique_ptr<int>)`, `sizeof(shared_ptr<int>)`, and `sizeof(weak_ptr<int>)`. Then, using a debugger or memory dump, find the control block in memory and verify its layout (strong count, weak count). Print the addresses of a shared pointer, its pointee, and its control block; verify they are three distinct locations.

2. **Cycle detection.** Write a simple graph (two nodes pointing to each other) using `shared_ptr` for all pointers. Observe that they never get deleted (memory leak). Then, change one pointer to `weak_ptr`. Verify that the nodes are deleted when they should be. (Hint: add destructors that print when they run.)

3. **Atomic cost.** Write a small program that creates a single `shared_ptr` and has 4 threads each copy and destroy it 1 million times. Measure elapsed time. Then, repeat with 4 separate `shared_ptr` instances (no contention on the control block). How much slower is the contended case?

4. **enable_shared_from_this.** Write a callback registry:
   ```cpp
   struct Listener : std::enable_shared_from_this<Listener> {
       void on_event() {
           register_callback([this] { do_work(); });
       }
   };
   ```
   Verify that `shared_from_this()` works inside `on_event()`. Then, remove `enable_shared_from_this` and try `std::shared_ptr<Listener>(this)`; observe the crash or corruption.

5. **make_shared vs new.** Profile two loops:
   ```cpp
   // Loop 1: make_shared
   for (int i = 0; i < 1000000; i++) {
       auto p = std::make_shared<std::vector<int>>(100);
   }
   
   // Loop 2: new
   for (int i = 0; i < 1000000; i++) {
       auto p = std::shared_ptr<std::vector<int>>(new std::vector<int>(100));
   }
   ```
   Time both. Why is `make_shared` faster? (Hint: count allocations and check cache locality.)

6. **Conceptual: Custom deleters.** A C library function `Resource* create_resource()` returns a pointer that must be freed with `destroy_resource(ptr)`. Write a factory function that returns `std::unique_ptr<Resource>` with the correct deleter. Show how the deleter is embedded in the type.

---

## 9.12 Summary

A smart pointer is a stack object that owns a heap object, using RAII to delete the heap object when the stack object dies. **`unique_ptr` provides exclusive ownership with zero overhead; `shared_ptr` provides shared ownership via atomic reference counting, paying a measurable but usually acceptable cost.** Cycles in `shared_ptr` graphs leak; break them with `weak_ptr`. You cannot safely construct a `shared_ptr` to `this` without inheriting from `enable_shared_from_this`. Use `make_shared` to fuse the allocations of the control block and object.

By using smart pointers, you move the burden of memory management from the programmer to the type system. The type encodes the ownership story: `unique_ptr<T>` says "I own T"; `T*` says "I have optional access to T"; `T&` says "T is guaranteed to be alive as long as I am." This discipline eliminates the vast majority of C++ memory bugs.

---

**[← Previous: Chapter 8 — Data-Oriented Design](08-data-oriented-design.md)** · **[↑ Part 2](README.md)** · **[Next: Chapter 10 — Intrusive Reference Counting →](10-intrusive-reference-counting.md)**
