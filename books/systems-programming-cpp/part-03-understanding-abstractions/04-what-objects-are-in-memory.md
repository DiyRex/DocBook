# Chapter 26 — What Objects Are In Memory

## Learning Objectives

By the end of this chapter you will be able to:

1. Predict the memory layout of a C++ object — field order, alignment, padding — given its class definition. Use `offsetof` and `sizeof` to verify.
2. Explain why a `struct { char a; int b; }` occupies more bytes than a naive calculation suggests, and why this matters for performance.
3. Recognize when and where a compiler inserts a hidden `vptr` (virtual table pointer), and understand what a vtable is and where it lives.
4. Understand how multiple inheritance affects object layout and why diamond inheritance causes complications.
5. Explain object identity — why address matters more than value — and how this distinction shapes comparisons in C++ and other languages.
6. Reason about object lifetime: the precise window during which an object is safe to use, from first instruction of the constructor to last instruction of the destructor.
7. Compare object layout in C++ to memory models in Python, Java, and JavaScript to see that "objects are just typed memory" is a language-universal pattern.

When you write `Foo f;`, something concrete exists in RAM. It is not an abstract thing — it is bytes, at an address, arranged in a particular pattern. This chapter strips away the OOP vocabulary and shows you what the compiler actually does: it lays out each field of each base class in a specific order, adds padding for alignment, optionally inserts a pointer to a vtable, and then your "object" is just a typed memory region with housekeeping rules about how to read, write, copy, move, and destroy it.

---

## 26.1 A Plain Object Is a Struct

The simplest case: a class with only non-virtual members and no base classes.

```cpp
class Point {
public:
    int x;
    int y;
    void move(int dx, int dy);
};
```

The compiler does not put `move` in the object. `move` lives in `.text` as a function (really a thunk that takes an implicit `this` pointer). The object itself contains *only the data members*:

```
Memory layout of an object of type Point:
+-------+
| x     |  4 bytes (int)
+-------+
| y     |  4 bytes (int)
+-------+
Total: 8 bytes
```

Every instance of `Point` is exactly 8 bytes, arranged as two consecutive 4-byte integers. On a 32-bit system, the layout would be different (maybe 8 bytes total, or maybe 4 on a platform where `int` is 2 bytes), but the principle is identical: **the object is the data, arranged in declaration order**.

You can verify this:

```cpp
#include <cstdio>
#include <cstddef>

class Point {
public:
    int x;
    int y;
};

int main() {
    std::printf("sizeof(Point) = %zu\n", sizeof(Point));
    Point p;
    std::printf("address of p.x = %p\n", (void*)&p.x);
    std::printf("address of p.y = %p\n", (void*)&p.y);
    std::printf("offset of x = %zu\n", offsetof(Point, x));
    std::printf("offset of y = %zu\n", offsetof(Point, y));
}
```

Output on a typical x86-64 system:

```
sizeof(Point) = 8
address of p.x = 0x7fff5fbff8a0
address of p.y = 0x7fff5fbff8a4
offset of x = 0
offset of y = 4
```

The fields live side by side, with no gap. The second integer's address is exactly 4 bytes after the first. This is the *canonical* object layout.

---

## 26.2 Padding and Alignment

Now complicate: what if fields have different sizes?

```cpp
class Bad {
public:
    char a;      // 1 byte
    int b;       // 4 bytes
    char c;      // 1 byte
};
```

The naive layout would be: `a` (1) + `b` (4) + `c` (1) = 6 bytes. In reality:

```cpp
std::printf("sizeof(Bad) = %zu\n", sizeof(Bad));
std::printf("offsetof a = %zu\n", offsetof(Bad, a));
std::printf("offsetof b = %zu\n", offsetof(Bad, b));
std::printf("offsetof c = %zu\n", offsetof(Bad, c));
```

Output:

```
sizeof(Bad) = 12
offsetof a = 0
offsetof b = 4
offsetof c = 8
```

Why 12, not 6?

**Alignment.** An `int` on x86-64 must live at an address divisible by 4. An address at offset 1 is not aligned; the CPU would either fault (on strict-alignment architectures like ARM) or run slower (on x86, which tolerates misalignment). The compiler inserts *padding* — unused bytes — to make the layout correct.

Layout in memory:

```
offset 0:  a (1 byte)
offset 1-3: padding (3 bytes)  <- to align b to a 4-byte boundary
offset 4-7: b (4 bytes)
offset 8:  c (1 byte)
offset 9-11: padding (3 bytes) <- to align the struct itself
Total: 12 bytes
```

The struct itself must also be aligned. Its alignment is the *maximum* alignment of any member. Since `b` is an `int` with 4-byte alignment, the whole struct has 4-byte alignment. So after the last member, padding is added to make the total size a multiple of 4.

This is called **"padding for alignment"** and it is **entirely compiler-controlled**. You have no say in where the padding goes — the ABI (Application Binary Interface) for your CPU and OS dictates the rules. You *can* control it with `alignas` and `#pragma pack`, but:

```cpp
#pragma pack(1)  // DO NOT DO THIS IN NEW CODE
class Tight {
public:
    char a;
    int b;
    char c;
};
#pragma pack()
```

With `pack(1)`, the compiler will layout the struct without padding: `a` at 0, `b` at 1, `c` at 5. This saves 6 bytes but causes `b` to be misaligned — and on strict-alignment machines, reading `b` faults. On x86, it works but is slower (unaligned access). In modern C++, use `[[no_unique_address]]` for zero-overhead bases (Chapter 27) instead; avoid `#pragma pack`.

The right fix: reorder.

```cpp
class Good {
public:
    int b;   // 4 bytes, aligned
    char a;  // 1 byte
    char c;  // 1 byte
};
```

Layout:

```
offset 0-3: b (4 bytes)
offset 4:   a (1 byte)
offset 5:   c (1 byte)
offset 6-7: padding (2 bytes)
Total: 8 bytes
```

By putting the larger field first, you reduce wasted space from 12 to 8. In a data structure with millions of instances (network protocol headers, serialized records, game-engine entities), this difference is significant — **8 vs 12 bytes per object means 33% less cache pressure, 33% faster iteration, 33% more objects per page**.

### A Rule for Padding

Arrange fields in *decreasing order of alignment requirement* (which is typically their size, but use `alignof` to be sure):

```cpp
struct Optimal {
    double d;      // 8 bytes, 8-byte aligned
    int i;         // 4 bytes, 4-byte aligned
    char c1, c2;   // 1 byte each, 1-byte aligned
};
// sizeof: 16 bytes. No wasted padding.
```

vs.

```cpp
struct Suboptimal {
    char c1, c2;   // 1 byte each
    int i;         // 4 bytes (padded to offset 4)
    double d;      // 8 bytes (padded to offset 8)
};
// sizeof: 24 bytes. 8 bytes wasted.
```

### Explicit Alignment

To force stricter alignment (useful for SIMD or cache-line alignment):

```cpp
struct Aligned {
    alignas(64) char data[64];  // Placed on a 64-byte boundary
};
```

When aligned, the compiler ensures the object's address is a multiple of 64. This is useful for:
- Avoiding false sharing (two threads on different cores reading from the same cache line).
- Aligning to cache-line boundaries for performance-critical hot paths.

---

## 26.3 Adding a vtable

When a class has a virtual function, the compiler's job gets more complicated.

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() = 0;
};

class Circle : public Shape {
public:
    int radius;
    void draw() override;
};
```

The compiler inserts a hidden pointer — the **vptr** (virtual function table pointer) — at the beginning of the object. This pointer points to a read-only table of function pointers, one per virtual function.

Layout of a `Circle` object:

```
offset 0:     vptr (8 bytes on 64-bit) -> points to Circle's vtable in .rodata
offset 8:     radius (4 bytes)
offset 12-15: padding (4 bytes)
Total:        16 bytes
```

The vtable for `Circle` lives in `.rodata` and looks like:

```cpp
// Conceptually, what the compiler generates:
struct Circle_vtable {
    // Destructor
    void (*destructor)(Circle*);
    // draw
    void (*draw)(Circle*);
};
```

In assembly, it might look like:

```
.rodata
Circle::_ZTV6Circle:  // mangled name for Circle's vtable
    .quad Circle::~Circle()     // address of destructor
    .quad Circle::draw()        // address of Circle::draw
```

When you call a virtual function:

```cpp
Shape* s = new Circle();
s->draw();
```

The compiler generates:

```cpp
// Pseudocode of what draw() becomes:
void (*fn)(Shape*) = s->vptr->draw;
fn(s);  // s becomes the 'this' pointer
```

In assembly:

```asm
mov rax, [s]          ; load vptr
mov rax, [rax + 8]    ; load function pointer (draw is second in vtable)
call rax              ; indirect call through function pointer
```

### Size and Overhead

```cpp
class Empty {
    virtual ~Empty() = default;
};
std::printf("sizeof(Empty) = %zu\n", sizeof(Empty));
```

Output: `sizeof(Empty) = 8` (just the vptr; the destructor contributes nothing to the object's size).

```cpp
class Derived : public Empty {
public:
    int x;
};
std::printf("sizeof(Derived) = %zu\n", sizeof(Derived));
```

Output: `sizeof(Derived) = 16` (vptr from base + x + padding).

### Key Points

1. **Only if the class has a virtual function.** A plain class with no virtuals has no vptr.
2. **One vptr per class.** Multiple inheritance can cause multiple vptrs; we'll see that below.
3. **The vtable is global and read-only.** All instances of a class share the same vtable. It lives in `.rodata`, not in the object itself.
4. **The vptr is opaque to you.** You cannot read it directly (doing so is technically undefined behavior, though implementations let you cast it). The compiler manages it.
5. **The vptr is checked for type safety.** When you use `dynamic_cast<>`, the runtime walks the vptr and verifies the actual type. This is how `dynamic_cast` works.

---

## 26.4 Multiple Inheritance Layout

When a class inherits from multiple base classes, the compiler places each base's subobject end-to-end.

```cpp
class A {
public:
    int a;
    virtual void fa() {}
};

class B {
public:
    int b;
    virtual void fb() {}
};

class C : public A, public B {
public:
    int c;
};
```

Layout of a `C` object:

```
offset 0:     A's vptr (8 bytes)
offset 8:     A::a (4 bytes)
offset 12-15: padding
offset 16:    B's vptr (8 bytes)
offset 24:    B::b (4 bytes)
offset 28-31: padding
offset 32:    C::c (4 bytes)
offset 36-39: padding
Total:        40 bytes
```

`C` has **two** vptrs — one for the `A` part of the inheritance tree, one for the `B` part. This is necessary because the vtables are different (one has `fa`, the other `fb`).

When you cast a `C*` to an `A*` or `B*`, the pointer *must be adjusted* to skip over irrelevant vptrs and fields.

```cpp
C c;
A* ap = &c;   // Points to offset 0 (the A subobject)
B* bp = &c;   // Compiler adjusts: points to offset 16 (the B subobject)
// ap and bp have different numeric values, even though both point into the same C object.
```

This **pointer adjustment** on inheritance casts is why multiple inheritance is expensive and why many designs avoid it.

### Diamond Inheritance

If `C` inherits from both `A` and `B`, and both `A` and `B` inherit from a common base `D`, you have a diamond:

```
    D
   / \
  A   B
   \ /
    C
```

The naïve layout would put `D` twice — once as part of `A`, once as part of `B`. This wastes space and causes member ambiguity (which `D::x` do you mean?). Most of the time, this is a design mistake.

C++ offers **virtual inheritance** to solve this:

```cpp
class A : virtual public D { };
class B : virtual public D { };
class C : public A, public B { };
```

With virtual inheritance, `D` appears only once in `C`'s layout, but the mechanism to find it is indirect — via a **virtual base offset table**. This adds complexity and runtime cost. Modern designs avoid this by using composition instead of diamond inheritance.

---

## 26.5 Object Identity

A pointer's **value** — the numeric address — *is* the object's identity in C++.

```cpp
Foo f1, f2;
Foo* p1 = &f1;
Foo* p2 = &f2;

*p1 == *p2;   // may be true (equal *values*)
p1 == p2;     // false (different *addresses*, different objects)
p1 == &f1;    // true (same address, same identity)
```

Even if `f1` and `f2` have identical member values, they are different objects because they live at different addresses. The address is the identity. This is a fundamental C++ model: **identity is address; you cannot have identity without location**.

This matters for:

- **Ownership.** "The owner of a heap object" is "the code that holds its address" (or a handle derived from that address, like a `unique_ptr` or `shared_ptr`).
- **Equality semantics.** `f1 == f2` (comparing values) is different from `&f1 == &f2` (comparing identity). Many classes overload `operator==` to compare values; the default is address-based and usually not what you want.
- **Containers.** A set or map hashes or compares object *values*, not identities. If you put `Foo` objects (by value) in a `std::set`, the set compares their contents, not their addresses. If you store `Foo*` pointers, the set compares addresses.
- **Caching and memoization.** If you cache results by object identity (a common pattern in compilers and databases), you use the address as the key.

### Contrast with Garbage-Collected Languages

In Java, every variable holding an object is a reference (pointer in disguise). Identity is still address; the difference is the *garbage collector* manages the lifetime:

```java
Foo f1 = new Foo();
Foo f2 = new Foo();
f1.equals(f2);     // compares contents; depends on overridden equals()
f1 == f2;          // false (different identities)
```

If you want to compare identity in Java, you check `f1 == f2` (comparing references, which are addresses). If you want value equality, you call `f1.equals(f2)`.

In Python:

```python
f1 = Foo()
f2 = Foo()
f1 == f2  # calls __eq__; depends on implementation
f1 is f2  # checks identity (address); false
```

The pattern is universal: every language has identity (address) and value, and you must keep them distinct in your reasoning.

---

## 26.6 Object Lifetime in Memory

An object's lifetime is the window during which it is safe to use. For an object on the stack:

```cpp
{
    Foo f;
    f.do_something();  // safe: f is alive
}  // destructor ~Foo() is called here; f is dead
// using f here is undefined behavior
```

More precisely:

1. **Storage allocation.** Memory is allocated (by the stack, the heap, or static initialization).
2. **Constructor begins.** The constructor's first instruction executes. The object *exists* now, even if the constructor hasn't finished setting up all members.
3. **Constructor ends.** The object is fully initialized and usable.
4. **Destructor begins.** The destructor starts. The object is still alive for the destructor's duration.
5. **Destructor ends.** The last instruction of the destructor has executed. The object is dead.
6. **Storage deallocation.** The memory is returned to the pool or reused.

A subtle but important point: **the storage exists before the constructor runs**. If your constructor throws an exception midway:

```cpp
class Foo {
public:
    Foo() {
        allocate_resource();   // might throw
        setup_member();        // might throw
    }
    ~Foo() { deallocate_resource(); }
};

{
    Foo f;  // if allocate_resource() throws, storage is freed, destructor not called
}
```

If `allocate_resource()` throws, the constructor never completes, the destructor is never called, and the storage is freed. This is why RAII (Resource Acquisition Is Initialization) uses the destructor: the destructor is *guaranteed* to run if and only if the constructor completes.

For heap objects:

```cpp
Foo* p = new Foo();   // storage allocated, constructor runs
delete p;             // destructor runs, storage deallocated
// p is now a dangling pointer; using it is UB
```

A key rule: **do not use an object outside its lifetime**. The compiler cannot always catch this at compile time, but runtime tools like AddressSanitizer can.

---

## 26.7 Worked Example: vtable Layout

Let's build a small hierarchy and inspect the layout at the bit level.

```cpp
#include <cstdio>
#include <cstddef>

class Animal {
public:
    int age;
    virtual void speak() { std::printf("...\n"); }
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    int breed_id;
    void speak() override { std::printf("Woof!\n"); }
};

int main() {
    std::printf("sizeof(Animal) = %zu\n", sizeof(Animal));
    std::printf("sizeof(Dog)    = %zu\n", sizeof(Dog));
    std::printf("offsetof(Animal, age) = %zu\n", offsetof(Animal, age));
    std::printf("offsetof(Dog, breed_id) = %zu\n", offsetof(Dog, breed_id));

    Dog dog;
    std::printf("&dog = %p\n", (void*)&dog);
    std::printf("&dog.age = %p (offset %zu)\n", (void*)&dog.age, (char*)&dog.age - (char*)&dog);
    std::printf("&dog.breed_id = %p (offset %zu)\n", (void*)&dog.breed_id, (char*)&dog.breed_id - (char*)&dog);

    // Inspect the vptr (if you dare; this is technically UB but works on all major compilers)
    const void** vptr_addr = (const void**)&dog;
    std::printf("vptr = %p\n", *vptr_addr);

    return 0;
}
```

Typical output on x86-64:

```
sizeof(Animal) = 16
sizeof(Dog)    = 24
offsetof(Animal, age) = 8
offsetof(Dog, breed_id) = 16
&dog = 0x7fff5fbff8a0
&dog.age = 0x7fff5fbff8a8 (offset 8)
&dog.breed_id = 0x7fff5fbff8b0 (offset 16)
vptr = 0x7ffed9a00f80
```

The vptr is at offset 0; it's 8 bytes. The `age` field is at offset 8. The `breed_id` field is at offset 16.

Using `objdump -d` or Godbolt, you can see the vtable:

```
0000000000201020 <_ZTV3Dog>:
  201020:	00 00 00 00 00 00 00 00 	.quad	0x0
  201028:	20 10 20 00 00 00 00 00 	.quad	0x201020 (RTTI)
  201030:	?? ?? ?? ?? ?? ?? ?? ?? 	.quad	Dog::~Dog()
  201038:	?? ?? ?? ?? ?? ?? ?? ?? 	.quad	Dog::speak()
```

Each entry is a function pointer. When you call `dog.speak()`, the compiler:

1. Loads the vptr from offset 0 of the object.
2. Loads the function pointer from the vtable (offset 16 from vptr; `speak` is second after destructor).
3. Calls through the function pointer.

---

## 26.8 Objects in Other Languages

C++'s model — data + optional vptr, all contiguous — is not universal.

### Python

In Python, every object is a dictionary of attributes plus a reference to its class:

```python
f = Foo()
f.x = 5
```

The object is internally something like:

```cpp
struct PythonObject {
    PyObject_HEAD          // reference count, type pointer
    PyDictObject* dict;    // hash table of attributes
};
```

Methods are looked up in the class's dictionary, not in a per-class vtable. This makes Python objects larger and method lookup slower, but it allows runtime attribute assignment and introspection that C++ doesn't.

### Java

Java objects have a header (object lock, garbage-collection info, type pointer) followed by fields, similar to C++:

```java
new Foo()
```

Looks roughly like (in memory):

```cpp
struct JavaObject {
    uint64_t mark;         // GC + lock bits
    uint64_t klass_ptr;    // points to the Class object
    // fields follow
};
```

The `klass_ptr` is similar to a vptr, but it points to a full `Class` descriptor (methods, fields, superclasses) rather than just a vtable. Method dispatch is virtual by default in Java (all non-final methods are like C++'s virtual functions).

### JavaScript (V8)

JavaScript uses **hidden classes** (called "Maps" internally) to assign shape to objects dynamically:

```javascript
const f = { x: 5 };
f.y = 10;
```

The V8 engine tracks the shape: first `f` has shape `{x}`, then shape `{x, y}`. Objects with the same shape can share a hidden class, and property access can be optimized to a direct offset lookup instead of a hashtable query.

### C++ vs Others

C++'s model is the most *compact* (no per-object hashtable, no GC header unless explicitly requested via `std::shared_ptr`). It is the most *static* (field offsets are known at compile time, not runtime). But it requires you to declare all fields up front.

---

## 26.9 Tradeoffs

| Layout Model | Pros | Cons | When to Use |
|---|---|---|---|
| Plain struct (no virtual) | Smallest; zero vptr overhead; stable ABI | No polymorphism; cast safety is your problem | Performance-critical data; network protocol parsing |
| Virtual functions (with vptr) | Polymorphism; `dynamic_cast` safe; runtime dispatch | One vptr per class; indirect call overhead; bigger objects | Extensible class hierarchies; plugin systems |
| Multiple inheritance | Models complex relationships; multiple vptrs per class | Complex layout; pointer adjustment on casts; often avoidable | Rare; consider composition instead |
| Virtual inheritance | Solves diamond inheritance | Adds virtual base offset table; slow dispatch | Very rare; almost always a design mistake |
| Python-style (dict + class pointer) | Runtime flexibility; attributes can be added dynamically | Per-object overhead; method lookup slower | Dynamic languages where this is expected |
| Java-style (header + class pointer) | GC-friendly; compact; safe by default | Header overhead; less control over layout | Managed languages with GC |

---

## 26.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Every object has a vtable." | Only if the class has a virtual function. A plain struct has no vptr overhead. |
| "Padding is always wasted." | Padding enables fast, aligned access. Removing it (via `#pragma pack`) breaks alignment and may fault on strict-alignment architectures. |
| "Objects in C++ are references like in Java." | No. C++ objects are values (stored inline); when you write `Foo f;`, you get a full object on the stack. In Java, `Foo f = new Foo();` gives you a reference; the object is on the heap. |
| "The vptr is just another pointer I can dereference." | The vptr is opaque. Dereferencing it directly is undefined behavior. Use `dynamic_cast` to query the type safely. |
| "Multiple inheritance always means multiple vptrs." | Yes, in the current C++ standard. However, the compiler can optimize non-polymorphic base classes to avoid extra vptrs. |
| "Object identity is the same as object equality." | No. Identity is "the same address"; equality is "same value." Two different `Point(5, 5)` objects are equal but not identical. |
| "Destructors run immediately after the last user is done with the object." | Destructors run when the object's lifetime ends (scope exit for stack, `delete` for heap). If you still have a pointer to a dead object and use it, the destructor has already run and you have UB. |
| "I can move an object in memory by memcpy'ing its bytes." | Only if the object is trivially relocatable (no pointers to its own members, no invariants that depend on address). Most objects are safe to move, but some (e.g., with intrusive lists) are not. |

---

## 26.11 Exercises

1. **Predict and verify.**  Define this struct:
   ```cpp
   struct S {
       char a;
       double b;
       int c;
       char d;
   };
   ```
   Predict the order of fields in memory and the size. Use `offsetof` and `sizeof` to verify. Then reorder the fields for minimum size and confirm the new sizes.

2. **Pointer adjustment on inheritance.**  Build a class hierarchy:
   ```cpp
   class A { int a; virtual void fa() {} };
   class B { int b; virtual void fb() {} };
   class C : public A, public B { int c; };
   ```
   Create a `C` object. Take its address, cast it to `A*` and `B*`, and print the numeric pointer values. They will differ. Why? Verify the offsets with `offsetof`.

3. **vtable inspection.**  Write a program that creates a polymorphic object and prints the address of its vptr and the addresses of the functions in its vtable. Use `objdump -d` on the binary to find the vtable in `.rodata` and confirm the function pointers match.

4. **Padding waste.**  Design two versions of a struct with 5 fields of varying sizes (e.g., `char`, `int`, `double`, `short`, `char`). Layout one poorly; layout the other optimally. Compare their sizes.

5. **Lifetime tracing.**  Build a tracer class:
   ```cpp
   struct Traced {
       Traced(int id) : id(id) { printf("Construct %d\n", id); }
       ~Traced() { printf("Destruct %d\n", id); }
       int id;
   };
   ```
   Create a vector of `Traced` objects and observe constructor/destructor calls as the vector grows, reallocates, and is destroyed. What do you notice?

6. **Alignment and cache lines.**  Create an array of 1000 structs with two `int` fields. Time iteration over the array. Then reorder the fields and time again. Measure cache misses with `perf stat`. Can you observe the cache-line effect?

7. **Conceptual: object identity in other languages.**  In Java, `a == b` compares references (identity). In Python, `a is b` does the same. Explain: how is this different from C++'s `&a == &b`? How is it the same? Can you construct a program in both languages and C++ that demonstrates the distinction?

---

## 26.12 Summary

When you write `Foo f;`, a compiler lays out its data members in declaration order, with padding for alignment, and optionally inserts a hidden `vptr` if the class has virtual functions. The object is a contiguous region of memory, typed by the compiler and enforced by scope rules (for stack) or ownership rules (for heap). Object identity is the address; two objects at different addresses are different, even if their values are equal. The lifetime of an object is from the first instruction of the constructor to the last instruction of the destructor. Multiple inheritance complicates layout; virtual inheritance is rarely needed and often a sign of a design that should use composition instead. Understanding object layout makes memory bugs visible and predictable.

---

> **[← Previous: Why Classes Exist](03-why-classes-exist.md)**  ·  **[↑ Part 3](README.md)**  ·  **[Next: Composition vs Inheritance →](05-composition-vs-inheritance.md)**
