# Chapter 12 — Manual Memory Management

Understanding manual memory management is not about writing C code in 2025. It is about understanding the contract that every allocation and deallocation makes, why that contract is dangerous, and what the cost is when it breaks. Even if you write in Rust or use smart pointers exclusively, the bugs you encounter will trace back to this layer. The allocator is where your machine's actual constraints become impossible to ignore.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish the three standard allocation APIs in C++ — `malloc`/`free`, `new`/`delete`, and `new[]`/`delete[]` — and explain why mixing them is undefined behavior.
2. Explain placement new and understand when you need to allocate memory separately from object construction, with real examples from arena allocators and aligned allocation.
3. Recognize the four bug families in manual memory management — memory leaks, double-frees, use-after-free, and dangling pointers — and identify each at runtime using AddressSanitizer and Valgrind.
4. Understand the role of allocators in C++, from `std::allocator` through custom allocators to polymorphic allocators, and know when you would actually write one.
5. Apply defensive patterns that survive even without RAII — setting freed pointers to null, ownership comments, and the `goto cleanup` pattern that C used for decades.
6. Reason about the tradeoffs among raw allocations, RAII, arena allocators, and garbage collection in specific contexts.

---

## The Three APIs

C++ inherits manual memory management from C and adds its own layer on top. Understanding all three APIs, and why mixing them is a contract violation, is the foundation of this chapter.

Before C++, there was no abstraction layer over `malloc` and `free`. The C runtime provided them, the OS provided the underlying system calls, and memory management was entirely manual — as it still is in C. C++ wrapped this with `new` and `delete` to integrate object lifecycle (construction and destruction) with allocation, but it did not replace the C layer; it stacked on top of it. This layering is why mixing them is such a sharp edge case.

### `malloc` and `free`: The C Layer

`malloc(size_t nbytes)` asks the operating system for a block of untyped, uninitialized memory. It returns a `void*` — a raw address, no type information. The OS may return memory via `sbrk` (expanding the process's heap) or `mmap` (mapping a new anonymous region). The returned pointer is the lowest address of a contiguous block of at least `nbytes` bytes; accessing beyond `nbytes` is undefined behavior.

```cpp
int* p = (int*)malloc(sizeof(int) * 10);
if (!p) {
    // allocation failed; out of memory
    perror("malloc");
    return -1;
}
p[5] = 42;
free(p);
p = nullptr;    // good practice: prevent use-after-free
```

`free(void* ptr)` returns memory to the OS for reuse. It expects `ptr` to be an address that malloc returned. The size is *not* passed as an argument; malloc maintains internal metadata (usually just before the allocated block) tracking the allocation size. This metadata is why `free` is cheap — no need to search; the size lives next to the block.

**Key properties:**
- Operates on untyped memory. No constructor or destructor runs.
- Null-safe: `free(nullptr)` is a no-op.
- Memory returned to free is available for reuse; it may or may not be returned to the OS (depends on allocator and OS behavior). Modern allocators keep freed memory for reuse to avoid system call overhead.
- Very fast in the common case: modern allocators (jemalloc, mimalloc) use thread-local arenas, making the common-case free O(1) with no locking.
- The contract: every pointer passed to `free` must have come from `malloc` (or `realloc`, or aligned_alloc). Passing a stack pointer, a data-segment pointer, or a garbage pointer to `free` is UB.

### `new` and `delete`: The C++ Layer

C++ wraps allocation with object lifecycle. The syntax is deceptive in its simplicity:

```cpp
MyClass* p = new MyClass(arg1, arg2);  // 1. allocate 2. construct
delete p;                               // 1. destruct 2. deallocate
p = nullptr;                            // good practice
```

`new T(args)` does two distinct things:
1. Calls `operator new(sizeof(T))`, which typically calls `malloc(sizeof(T))` (but can be overridden per-class or globally).
2. Calls the constructor `T::T(args)` at the allocated address.

Both must succeed for the object to exist. If the constructor throws, the allocated memory is deallocated automatically (the exception is safe).

`delete p` does two distinct things:
1. Calls the destructor `T::~T()`.
2. Calls `operator delete(p)`, which typically calls `free(p)`.

The order matters. Destructor runs first (while `p` still points to valid memory and can access member data); then the memory is freed.

**Key difference from malloc/free:** constructors and destructors run. If your destructor has side effects — closing a file, releasing a lock, deallocating sub-objects — you *must* use `delete`, not `free`. Using `free` will skip the destructor and leak or corrupt resource state.

```cpp
struct File {
    FILE* handle;
    File(const char* path) { handle = fopen(path, "r"); }
    ~File() { if (handle) fclose(handle); }
};

File* f = new File("data.txt");  // constructor: fopen runs, opens the file
delete f;                          // destructor: fclose runs; file handle released

// if you malloc(sizeof(File)) + constructor,
// then free() instead of delete: destructor never runs
// and the file handle is left open (resource leak)
```

This is why C++ separates `new`/`delete` from `malloc`/`free`. The language recognized that memory lifecycle and object lifecycle are not the same thing.

### `new[]` and `delete[]`: Array Variants

When allocating an array, you must use `new[]` and `delete[]`:

```cpp
MyClass* arr = new MyClass[10];  // 10 objects constructed
delete[] arr;                     // 10 destructors run, then deallocate
```

The `[]` variants are semantically different from single-object allocation. For types with non-trivial destructors, `new[]` must store the count somewhere so that `delete[]` knows how many times to call the destructor. This count is typically stored in a hidden overhead slot just before the pointer, consuming extra memory and adding a memory read on delete.

For types with trivial destructors (int, float, pointers, POD structs), the compiler optimizes this away — `new[]` and `new` become equivalent.

```cpp
int* p = new int[100];
delete[] p;     // no overhead for int; just free

struct POD { int x; };
POD* arr = new POD[50];
delete[] arr;   // no overhead; POD is trivial

struct NONTRIVIAL {
    std::string s;
    ~NONTRIVIAL() { /* clean up string */ }
};
NONTRIVIAL* arr = new NONTRIVIAL[20];
delete[] arr;   // hidden count stored; overhead ~8 bytes
```

Mixing them is undefined behavior with severe consequences:

```cpp
MyClass* arr = new MyClass[10];
delete arr;          // UB: destructor runs on only one object
                     // 9 destructors never run; their resources leak
                     // memory allocated for 10 objects may be leaked
                     // or cause corruption when reused

MyClass* obj = new MyClass();
delete[] obj;        // UB: destructor runs once; then delete[] tries to read
                     // the array count from address (obj - 8), which was
                     // never written. Reads garbage; heap metadata corrupts.
```

The compiler cannot catch this error (it would require data-flow analysis across complex control flow). AddressSanitizer catches it at runtime.

### Comparison Table

| API | Signature | Returns | Uninitialized | Destructors | Size Tracking |
|---|---|---|---|---|---|
| `malloc` | `void* malloc(size_t)` | `void*` | Yes | No | Internal metadata |
| `free` | `void free(void*)` | — | — | No | Implicit |
| `new` | `T* new T(args)` | `T*` | No | Yes (calls `T()`) | Implicit |
| `delete` | `delete T*` | — | — | Yes (calls `~T()`) | Implicit |
| `new[]` | `T* new T[n]` | `T*` | No | Yes (n times) | Implicit, per-object |
| `delete[]` | `delete[] T*` | — | — | Yes (n times) | Implicit, per-object |

**Never mix:** `malloc` with `delete`, `new` with `free`, `new` with `delete[]`, `new[]` with `delete`. Each pair assumes a matching partner.

---

## Placement new and Aligned Allocation

Real systems often separate *allocation* (obtaining a block of memory) from *construction* (building an object at that address).

### Placement new: Construct in Pre-Allocated Space

Placement new syntax:

```cpp
void* placement_addr = my_buffer;
MyClass* obj = new(placement_addr) MyClass(arg1, arg2);
```

This calls the constructor at an explicit address, without calling `malloc` or `operator new`. The memory must already exist and be properly aligned. The destructor still runs; deallocation is *your* responsibility. Placement new is not a memory allocation — it is a *construction* call at a specific address.

**Real use case: arena allocators.** Instead of calling `malloc` for every small object, an arena pre-allocates a large block and doles out chunks:

```cpp
class Arena {
    char* buffer;
    size_t cursor;
    size_t capacity;
public:
    Arena(size_t size) : buffer(new char[size]), cursor(0), capacity(size) {}
    
    template<typename T, typename... Args>
    T* allocate(Args&&... args) {
        size_t needed = sizeof(T);
        if (cursor + needed > capacity) throw std::bad_alloc();
        
        void* slot = buffer + cursor;
        cursor += needed;
        return new(slot) T(std::forward<Args>(args)...);
    }
    
    ~Arena() { delete[] buffer; }
};
```

Objects are constructed via placement new into the arena's buffer. Destructors run when you explicitly call them (the arena is responsible for bookkeeping):

```cpp
Arena arena(4096);
MyClass* obj = arena.allocate<MyClass>(arg);
// ...
obj->~MyClass();  // explicit destructor call
// arena itself freed in destructor; all at once
```

This pattern is powerful because:
- **Zero allocation overhead**: no per-object `malloc` call, no per-object header, no fragmentation.
- **Cache-friendly**: objects are contiguous in memory.
- **Bulk deallocation**: destroying the arena frees everything in one operation.
- **Suitable for scoped allocations**: temporary parse trees, request-scoped objects, game frame allocators.

The cost is inflexibility: you cannot free a single object from the arena (only in LIFO order with some variants), and you must manually track object lifetimes and call destructors.

### Aligned Allocation

Memory must be aligned: addresses must be multiples of certain powers of two. A 16-byte SIMD vector should start at an address divisible by 16; a 64-byte cache line should be cache-aligned.

C++17 added `std::aligned_alloc`:

```cpp
void* p = std::aligned_alloc(64, 1024);    // allocate 1024 bytes, aligned to 64
MyClass* obj = new(p) MyClass();
obj->~MyClass();
free(p);    // free is OK for aligned_alloc
```

Note: `aligned_alloc` requires the allocation size to be a multiple of the alignment (POSIX rule). If you need non-multiple sizes, some systems provide `memalign` (deprecated) or you must round up manually.

C (POSIX) provides `posix_memalign`:

```cpp
void* p;
if (posix_memalign(&p, 64, 1024)) {
    perror("posix_memalign");
    return -1;
}
// p is now aligned to 64 bytes; 0 on success, error code on failure
free(p);
```

C++ also provides `std::aligned_storage`, for stack allocation:

```cpp
alignas(64) char buffer[1024];    // buffer starts at 64-byte boundary
MyClass* obj = new(buffer) MyClass();
```

The `alignas` keyword instructs the compiler to place the buffer at the given alignment. This is a compile-time directive, not a runtime function, so it is zero-cost.

**Why alignment matters:**

- **SIMD**: SSE/AVX instructions (and ARM NEON) require aligned operands. Unaligned access is slow (hardware fault + retry) or illegal on strict-alignment architectures.
- **ABI contracts**: some calling conventions and library ABIs require specific alignments. C++ standard layout and data layout contracts depend on alignment. Misaligned memory breaks `std::memcpy` optimizations and causes subtle bugs.
- **False sharing**: if two threads' data occupy the same cache line (64 bytes on modern CPUs), both threads compete for that line's coherency. Every write by one thread invalidates it in the other's cache, causing repeated fetches. Separating data to different cache lines (aligning to 64 bytes) eliminates this contention and can multiply performance by 5x in pathological cases.
- **Hardware prefetching**: some CPUs prefetch more aggressively for aligned access.

---

## The Four Bug Families

Manual memory management concentrates bugs into four pathologies. Know them by sight.

### Bug 1: Memory Leak

You allocate memory but never deallocate it. The memory is lost when the program exits; while it's running, total memory grows until OOM.

```cpp
void process_file(const char* path) {
    char* buffer = (char*)malloc(4096);
    FILE* fp = fopen(path, "r");
    // ... read from fp into buffer ...
    fclose(fp);
    // forgot to free(buffer)
}  // memory leaks; buffer is never deallocated
```

**At runtime:** memory usage climbs. The program becomes sluggish. Eventually, `malloc` returns null or the OS kills the process.

**Detection:**

```bash
valgrind --leak-check=full ./program
```

Output will show "definitely lost: 4096 bytes":

```
==12345== 4,096 bytes in 1 blocks are definitely lost in loss record 1
==12345==    at 0x...: malloc (vg_replace_malloc.c:...)
==12345==    by 0x...: process_file (yourfile.cpp:5)
==12345==    by 0x...: main (yourfile.cpp:25)
```

AddressSanitizer also detects leaks (with `-fsanitize=address -fsanitize=leak`):

```bash
g++ -g -fsanitize=address -fsanitize=leak yourfile.cpp
./a.out
```

Reports `SUMMARY: AddressSanitizer: 4096 byte(s) leaked in 1 allocation(s)`.

### Bug 2: Double-Free

You deallocate the same memory twice.

```cpp
int* p = (int*)malloc(sizeof(int));
free(p);
free(p);    // UB: second free
```

**At runtime:** the allocator's free-list metadata corrupts. The next `malloc` may return a pointer to memory that's already in use. Writes through the second pointer silently corrupt other allocations.

**Detection with AddressSanitizer:**

```bash
g++ -g -fsanitize=address yourfile.cpp
./a.out
```

Output:

```
==12345==ERROR: AddressSanitizer: attempting double-free on 0x60300000eff0
    #0 0x... in free (/path/a.out+...)
    #1 0x... in main yourfile.cpp:5
```

AddressSanitizer catches this immediately.

### Bug 3: Use-After-Free

You dereference a pointer to memory that has been deallocated.

```cpp
int* p = (int*)malloc(sizeof(int));
*p = 42;
free(p);
*p = 100;    // UB: p is dangling
```

**At runtime:** the freed memory might be overwritten by another allocation. You read or write garbage, or corrupt other objects. Sometimes the program crashes; sometimes it silently produces wrong results.

**Detection with AddressSanitizer:**

```bash
g++ -g -fsanitize=address yourfile.cpp
./a.out
```

Output:

```
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x... at pc
    #0 0x... in main yourfile.cpp:6 write of size 4
```

AddressSanitizer quarantines freed memory to catch this. It's one of its most important catches.

### Bug 4: Dangling Pointer

A pointer outlives the object it points to, but the memory has not been freed yet. The memory still contains *something*, but not what you expect.

```cpp
struct Data {
    int x;
};

Data* make_dangling() {
    Data d = {42};
    return &d;      // d is stack-allocated; will be destroyed on return
}

int main() {
    Data* p = make_dangling();
    printf("%d\n", p->x);   // UB: p points to stack memory that has been reused
}
```

**At runtime:** the stack frame was reused. `p->x` reads whatever happens to be at that address now — random stack garbage, or (by chance) a sensible value that corrupts your understanding.

**Detection:** AddressSanitizer can catch some cases (when the stack frame is quickly reused), but dangling pointers are harder to detect than use-after-free because the memory is not freed. Valgrind, with `--track-origins=yes`, can help track down where the bad pointer originated. Rust's borrow checker catches this at compile time.

---

## Allocators in C++

C++ provides a pluggable allocation interface. Instead of always calling `malloc`, code can request allocation from an `Allocator` object.

### `std::allocator`

The default, minimal allocator. It wraps `new` and `delete`:

```cpp
template<typename T>
class std::allocator {
public:
    T* allocate(size_t n);
    void deallocate(T* p, size_t n);
    // ... construct/destroy (C++17 deprecated)
};

std::vector<int> v;    // uses std::allocator<int> by default
```

### Custom Allocators

You can provide your own. STL containers accept an allocator template parameter:

```cpp
template<typename T>
class PoolAllocator {
    std::vector<T> pool;
    std::vector<bool> available;
public:
    T* allocate(size_t n) {
        if (n != 1) throw std::bad_alloc();    // only allocate singletons
        for (size_t i = 0; i < pool.size(); ++i) {
            if (available[i]) {
                available[i] = false;
                return &pool[i];
            }
        }
        throw std::bad_alloc();
    }
    void deallocate(T* p, size_t n) {
        ptrdiff_t idx = p - &pool[0];
        available[idx] = true;
    }
};

std::vector<int, PoolAllocator<int>> v;   // custom allocator
```

**When to write a custom allocator:**
- **Memory pools**: objects of fixed size, reused in bulk.
- **Thread-local allocation**: one allocator per thread, no locking.
- **Restricted heap**: allocation from a pre-allocated region (embedded systems, real-time).
- **Tracking**: count allocations, measure fragmentation, enforce limits.

Most of the time, the default `std::allocator` or a standard third-party one (jemalloc, mimalloc, tcmalloc) is sufficient. Custom allocators are a specialized tool.

### Polymorphic Allocators (C++17)

`std::polymorphic_allocator` allows runtime polymorphism over allocation strategies:

```cpp
#include <memory_resource>

std::pmr::vector<int> v1;  // uses default resource (usually malloc-based)

char buffer[4096];
std::pmr::monotonic_buffer_resource mbr(buffer, sizeof(buffer));
std::pmr::vector<int> v2(&mbr);   // uses the monotonic buffer
```

Instead of allocator type being a compile-time template parameter (which forces all containers of different types to use different allocators), the memory resource is a runtime pointer. This enables switching allocators without recompiling or changing container types. It's particularly useful in scenarios where you want different containers to share the same memory pool.

**Common memory resources:**
- `std::pmr::new_delete_resource()` — wraps `new` and `delete`.
- `std::pmr::monotonic_buffer_resource` — allocates from a pre-provided buffer; never frees individual allocations (only when the resource is destroyed).
- `std::pmr::synchronized_pool_resource` — thread-safe pool allocator.
- `std::pmr::unsynchronized_pool_resource` — non-thread-safe pool allocator (faster).

Real use case: in a server that processes many requests, each request can use its own `monotonic_buffer_resource` backed by a large pre-allocated buffer. All temporary objects (parsed JSON, intermediate data structures) are allocated into this resource. When the request completes, the resource is destroyed in O(1) time, freeing everything at once — no per-object deallocation overhead.

---

## Defensive Patterns Before RAII

Modern C++ (post-C++11) uses RAII to manage memory automatically (Chapter 15). Before that — and in C code, and in legacy C++ — people used defensive patterns to reduce bugs.

### Pattern 1: Set to Nullptr After Free

```cpp
free(p);
p = nullptr;
```

This prevents use-after-free if `free(p)` is called again later:

```cpp
free(p);    // deallocate
p = nullptr;

// ... later ...

free(p);    // free(nullptr) is safe
```

It doesn't prevent all use-after-free (dereferencing `p` between free and null-setting still crashes), but it catches one common pattern.

### Pattern 2: Ownership Comments

Without type-system support (like Rust's `&` vs `owned`), humans document memory ownership:

```cpp
// allocate with malloc; caller must free
void* extract_region(Image* img, int x, int y, int w, int h);

// called struct; do not free; lifetime tied to parent
int* get_cache(Database* db);

// takes ownership; caller must not free; function will delete
void process_and_store(Widget* widget);
```

These comments are checked by code review, not by the compiler. Easy to miss; easy to violate. But better than nothing.

### Pattern 3: `goto cleanup`

C uses structured error handling with `goto`:

```cpp
int process_file(const char* path) {
    FILE* fp = nullptr;
    char* buffer = nullptr;
    int result = -1;
    
    fp = fopen(path, "r");
    if (!fp) goto cleanup;
    
    buffer = (char*)malloc(4096);
    if (!buffer) goto cleanup;
    
    // ... use fp and buffer ...
    result = 0;
    
cleanup:
    if (buffer) free(buffer);
    if (fp) fclose(fp);
    return result;
}
```

All exit paths (success, or error at any step) pass through `cleanup`, ensuring resources are released. Every `if (!condition) goto cleanup;` check is an early exit; the cleanup section is a catch-all. This pattern is verbose and unfamiliar to modern developers, but it's the reason C code survived for decades before exceptions and destructors.

Modern C++ replaces this with RAII (scope guards, smart pointers, destructors); but the pattern is still found in large C codebases and (ironically) in kernel code and systems that cannot use exceptions.

### The Discipline of Manual Management

Beyond these patterns, manual memory management requires discipline. Every allocation must have an owner — a function or struct that is responsible for its deallocation. Ownership must be documented (via comments or type system). Every path through a function must deallocate what it allocated or pass ownership to a caller.

The patterns above reduce bugs, but they do not prevent all bugs. This is why RAII exists: to move the responsibility from the programmer's shoulders to the language's automated mechanisms.

---

## Worked Example

Here is a real program with a leak and a double-free. We'll detect them with AddressSanitizer.

```cpp
// file: memleak.cpp
#include <cstdlib>
#include <cstdio>
#include <string.h>

struct Record {
    char* name;
    int age;
};

Record* create_record(const char* name, int age) {
    Record* r = (Record*)malloc(sizeof(Record));
    r->name = (char*)malloc(strlen(name) + 1);
    strcpy(r->name, name);
    r->age = age;
    return r;
}

void delete_record(Record* r) {
    free(r->name);
    free(r);
}

int main() {
    // Leak: create two records, delete only one
    Record* r1 = create_record("Alice", 30);
    Record* r2 = create_record("Bob", 25);
    delete_record(r1);
    // r2 leaked: name and Record both not freed
    
    // Double-free: delete the same record twice (contrived, but shows the bug)
    Record* r3 = create_record("Carol", 35);
    delete_record(r3);
    delete_record(r3);    // UB: second free(r3) and free(r3->name)
    
    return 0;
}
```

Compile and run with AddressSanitizer:

```bash
g++ -g -fsanitize=address -fsanitize=leak memleak.cpp -o memleak
./memleak
```

Output:

```
=================================================================
==12345==ERROR: AddressSanitizer: attempting double-free on 0x60200000eff0
    #0 0x... in free (...)
    #1 0x... in delete_record memleak.cpp:19
    #2 0x... in main memleak.cpp:35
    #3 0x... in __libc_start_main (...)

Address 0x60200000eff0 is located inside of 48-byte region [0x60200000efc0,0x60200000eff0)
freed by thread T0 here:
    #0 0x... in free (...)
    #1 0x... in delete_record memleak.cpp:19
    #2 0x... in main memleak.cpp:33

SUMMARY: AddressSanitizer: double-free memleak.cpp:35
```

If we comment out the second `delete_record(r3)`, we get the leak report:

```
=================================================================
==12345==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 48 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc (...)
    #1 0x... in create_record memleak.cpp:9
    #2 0x... in main memleak.cpp:28

Direct leak of 4 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc (...)
    #1 0x... in create_record memleak.cpp:10
    #2 0x... in main memleak.cpp:28

SUMMARY: LeakSanitizer: 52 byte(s) leaked in 2 allocation(s).
```

The stack traces pin down the exact lines. This is precisely what makes ASan invaluable during development: instead of discovering memory bugs in production via crashes or memory bloat, you find them locally with full context.

**Lesson from the example:** the double-free was caught immediately. The leak was found by ASan's leak detector (enabled with `-fsanitize=leak`). In real code, leaks are often less obvious — they accumulate over thousands of requests. Valgrind would report the same leaks but with a different format and considerably slower execution. For interactive development, AddressSanitizer's speed is a major advantage.

The `Record` structure above has two pointers: `name` and the implicit record itself. Forgetting to free `name` is a classic mistake. In modern C++, you would write:

```cpp
struct Record {
    std::string name;  // destructor frees the string automatically
    int age;
};
```

And the entire leak disappears. This is what RAII buys you: the language enforces cleanup through object lifetime semantics, not through your discipline. The next chapter covers ownership in depth; after that, Chapter 15 covers RAII and smart pointers that automate all of this.

---

## Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| Raw `malloc`/`free` | Minimal overhead; flexible; only mechanism in C | No type safety; no RAII; easy to leak/double-free | C code; low-level systems where custom allocation is essential |
| `new`/`delete` | Type-safe; calls constructors/destructors; same performance as malloc | Still manual; still leakable; mixes allocation and construction | Legacy C++ (pre-C++11) when RAII unavailable |
| RAII (smart pointers, stack-allocated) | Scope-based automation; compile-time safety; no leaks if done right | Steeper learning curve; move semantics required; some patterns (circular references) need careful handling | Modern C++ (default choice) |
| Arena allocator | Defers all deallocation to end; no fragmentation; cache-friendly | Inflexible; can't free individual objects; must manually call destructors | Temporary objects in tight loops; games; parsers with parse-tree lifetime |
| Garbage collection (Java, Go, Python) | No manual deallocation; no use-after-free; simple programming model | Pause times (STW); unpredictable allocation latency; space overhead (for GC metadata) | Languages where GC is standard; systems where pause times tolerable |
| Custom allocator | Fine-grained control; can optimize for specific patterns | Complexity; must maintain invariants; can amplify bugs if wrong | Embedded/real-time; performance-critical tight loops; memory-constrained systems |

---

## Common Misconceptions

*1. "If I use `new`, I don't need to worry about memory."*

False. `new`/`delete` still leaks if you forget the delete, still double-frees if you call delete twice, still dangling if you delete and then dereference. They add constructor/destructor support, but do not automate deallocation. Use RAII (smart pointers) for automation.

*2. "Free-list fragmentation is the allocator's problem, not mine."*

Partially true. Modern allocators (jemalloc, mimalloc) are sophisticated. But pathological allocation patterns (allocate 1MB, free 512KB, allocate 256KB, repeat) can fragment any allocator. If memory usage climbs mysteriously, profile allocator behavior with tools like `valgrind --tool=massif`.

*3. "AddressSanitizer will catch all memory bugs."*

No. ASan excels at use-after-free and double-free. But it cannot catch all leaks in long-running programs (memory that *should* be freed but never is found), and it cannot prove absence of UB. Use it together with code review and Valgrind for leak detection.

*4. "Aligned allocation is only for SIMD."*

False. False sharing on cache lines (multiple threads accessing the same 64-byte cache line) is a performance disaster. Alignment-based padding is essential for multi-threaded code. Also, some ABIs and library contracts mandate alignment that has nothing to do with SIMD.

*5. "Placement new is rarely used; I can ignore it."*

Mostly true for application code, but placement new underlies many important patterns: arena allocators, `std::vector`'s internal storage, custom memory pools. Understanding it is essential for reading advanced codebases.

*6. "I can use `reinterpret_cast<T*>(ptr)` to treat any memory block as a T."*

Casting a pointer is free, but treating unaligned or uninitialized memory as a T is undefined behavior. If T requires alignment (often true for types with SIMD members or alignment-sensitive ABI requirements), unaligned access may fault or silently corrupt. Use placement new to construct at a known address, not a cast.

---

## Exercises

1. **Spot the bug.** Identify the memory error(s) in each:

   ```cpp
   // (a)
   int* p = (int*)malloc(sizeof(int) * 10);
   int* q = new int[10];
   delete p;      // should be free(p)
   
   // (b)
   int* arr = new int[100];
   delete arr;    // should be delete[]
   
   // (c)
   Record* r = create_record("name");
   process_record(r);
   delete_record(r);
   if (!r) {      // dangling check; but r is freed
       printf("null\n");
   }
   delete_record(r);
   ```

2. **Leak or double-free?** Compile and run under AddressSanitizer:

   ```cpp
   void buggy_process(int n) {
       int* data = (int*)malloc(sizeof(int) * n);
       if (n == 0) {
           return;        // leak: data not freed
       }
       // ... use data ...
       free(data);
   }
   ```

   Add logic to prevent the leak without RAII.

3. **Arena allocator.** Implement a simple stack-like allocator:

   ```cpp
   class StackAllocator {
       char* buffer;
       size_t pos, capacity;
   public:
       StackAllocator(size_t cap);
       void* allocate(size_t size);
       void reset();  // free all at once
   };
   ```

   Then use placement new to allocate objects into it. Call destructors manually. Benchmark allocation speed vs `malloc`.

4. **Placement new with alignment.** Allocate a 64-byte-aligned array of 100 integers. Print the address; verify alignment with `printf("%p mod 64 = %ld\n", p, (long)p % 64)`.

5. **Comparative analysis.** Write the same short program (e.g., read, parse, store 1000 JSON objects) in:
   - C with `malloc`/`free`.
   - C++ with `new`/`delete`.
   - C++ with `std::unique_ptr` / RAII.
   - Valgrind all three with `--leak-check=full`. Compare.

---

## Summary

Manual memory management is the bedrock of C and C++. Its three APIs — `malloc`/`free`, `new`/`delete`, and `new[]`/`delete[]` — must never be mixed; each API has a specific partner and calling the wrong one results in undefined behavior. Placement new separates allocation from construction, enabling arena allocators and precise control over object lifetime. Four bug families — leaks, double-frees, use-after-free, and dangling pointers — concentrate the risks of manual management. Each is detectable at runtime with AddressSanitizer and Valgrind; development workflows that omit these tools suffer repeated discoveries of memory bugs in production.

Aligned allocation matters for correctness (SIMD, ABI contracts) and performance (false sharing on cache lines). Custom allocators optimize for specific patterns — arena allocators for temporary objects, pool allocators for fixed-size reuse — but add complexity and are rarely necessary for application code.

Defensive patterns (null-setting, ownership comments, `goto cleanup`) reduce bugs before RAII. They are still found in large C codebases and systems code that cannot use exceptions. In modern C++, RAII and smart pointers replace them entirely, moving memory management responsibility from programmer discipline to language-enforced semantics.

The deepest lesson: **understanding manual memory management teaches you where all memory bugs come from**. Even in languages with garbage collection, even with RAII and smart pointers, you will encounter memory bugs. They will be either violation of the preconditions (use-after-move in smart pointers, reference to freed objects in GC'd languages) or architectural oversights (circular references not being collected, resources not being released). Knowing this layer gives you the diagnostic framework to understand them.

---

**[← Previous: Stack vs Heap Internals](01-stack-vs-heap-internals.md)** · **[↑ Part 2](README.md)** · **[Next: Ownership Models →](03-ownership-models.md)**
