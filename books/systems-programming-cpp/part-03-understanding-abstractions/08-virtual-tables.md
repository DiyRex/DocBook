# Chapter 8 — Virtual Tables

A **vtable** is an array of function pointers. A **vptr** is a pointer to that array. Dispatching a virtual call is two memory loads. Once you internalize that picture — `load vptr from object; load function pointer from vptr; call it` — all the special cases (multiple inheritance, virtual destructors, RTTI) make sense.

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a vtable is, where it lives, and what a vptr is.
2. Predict the layout of a vtable for a class with virtual methods and understand how inheritance and overriding affect it.
3. Trace, by hand, a virtual method call: load vptr, load function pointer, call it. Estimate the cycle cost.
4. Explain why virtual destructors exist and what goes wrong without them.
5. Understand how multiple inheritance creates multiple vptrs and when pointer adjustments (thunks) are needed.
6. Recognize that RTTI (`typeid`, `dynamic_cast`) adds type_info pointers to the vtable infrastructure.
7. Reason about when virtual calls are fast enough, when devirtualization helps, and when the cost matters.
8. Inspect a compiled binary with `objdump` or Godbolt to find and verify the actual vtables.

---

## 8.1 The Basic Picture

A class with virtual methods gets a **virtual function table** — a static, compile-time construct.

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
    virtual double area() const = 0;
};

class Circle : public Shape {
private:
    double radius;
public:
    Circle(double r) : radius(r) {}
    void draw() const override { /* ... */ }
    double area() const override { return 3.14159 * radius * radius; }
};

class Square : public Shape {
private:
    double side;
public:
    Square(double s) : side(s) {}
    void draw() const override { /* ... */ }
    double area() const override { return side * side; }
};
```

The compiler builds one vtable per class. For `Circle`, it looks conceptually like:

```
CircleVTable (resides in .rodata, read-only):
  [0] -> &Circle::~Circle
  [1] -> &Circle::draw
  [2] -> &Circle::area
```

For `Square`:

```
SquareVTable (in .rodata):
  [0] -> &Square::~Square
  [1] -> &Square::draw
  [2] -> &Square::area
```

For `Shape` (abstract), there is *still* a vtable in case a pointer to Shape points to a concrete subclass:

```
ShapeVTable (in .rodata):
  [0] -> &Shape::~Shape (or pure function stub)
  [1] -> &Shape::draw (pure function stub)
  [2] -> &Shape::area (pure function stub)
```

Each *instance* of a class carries a **vptr** — a hidden pointer to its vtable, usually placed in the object's layout at offset 0.

Layout of a `Circle(5.0)`:

```
offset 0:   vptr -> CircleVTable   (8 bytes on x86-64)
offset 8:   radius (double, 8 bytes)
total:      16 bytes
```

### Visual summary

```
Memory layout:

.rodata (read-only)
+----------------------------+
| Shape's vtable             |
|  [0] -> &Shape::~Shape     |
|  [1] -> &Shape::draw       |
|  [2] -> &Shape::area       |
+----------------------------+

| Circle's vtable            |
|  [0] -> &Circle::~Circle   |
|  [1] -> &Circle::draw      |
|  [2] -> &Circle::area      |
+----------------------------+

Heap / Stack
+----------------------------+
| Circle instance            |
|  +0:  vptr ----+           |
|  +8:  radius   |           |
+---+---+--------+           |
    |                        |
    +------> [Circle vtable] |
              in .rodata <---+
```

---

## 8.2 A Virtual Call Step-By-Step

When you call `p->draw()` where `p` is a `Shape*` pointing to a `Circle`:

```cpp
Shape* p = new Circle(5.0);
p->draw();   // which draw? Shape's or Circle's?
```

The compiler generates:

```asm
; Assume p is in rdi (x86-64 calling convention)
mov    rax, [rdi]          ; load vptr from p (offset 0)
call   qword ptr [rax + 8]  ; load function pointer at [vptr + 8];
                            ; (8 = 1 * sizeof(function pointer));
                            ; offset 8 is the draw() slot
```

In C-like pseudocode:

```c
typedef void (*VFuncPtr)(void*);

struct Shape {
    VFuncPtr* vptr;
};

void virtual_call_draw(Shape* p) {
    VFuncPtr* vtable = p->vptr;           // load vptr
    VFuncPtr  func = vtable[1];           // [1] = index of draw() slot
    func((void*)p);                       // call, passing object as this
}
```

What happened:

1. **First load:** `load vptr from object` — dereferencing the object to read its vptr. One memory load, likely cache-hit (vptr is at offset 0).
2. **Second load:** `load function pointer from vptr[index]` — dereferencing the vtable to get the actual function address. This is also likely a cache-hit because vtables are small and in `.rodata`.
3. **Indirect call:** `call [function address]` — a CPU branch. Modern CPUs predict this: if the call site is *monomorphic* (always called with the same concrete type), the branch predictor learns the target and predicts correctly. If the call site is *polymorphic* (different types at different times), the predictor may thrash, and you pay a mispredict penalty (~15 cycles on modern x86).

Total best case: **5 cycles** (load + load + call, all cache-hits, correct prediction).
Total worst case: **20+ cycles** (cache miss + mispredict).
In practice: **5-15 cycles** depending on cache and branch predictor state.

Compare to a **direct call** (non-virtual): 1-2 cycles for the `call` itself, no indirection, fully inlinable. Virtual calls cannot be inlined (unless the compiler can prove the type — which is rare outside of `final` classes).

---

## 8.3 Inheritance and Overriding

When a derived class overrides a virtual method, it does *not* get a new slot in the vtable — it replaces the entry.

```cpp
class Animal {
public:
    virtual ~Animal() = default;
    virtual void speak() const = 0;      // slot [1]
};

class Dog : public Animal {
public:
    void speak() const override {        // replaces slot [1]
        std::cout << "Woof!\n";
    }
};

class Cat : public Animal {
public:
    void speak() const override {        // replaces slot [1] in Cat's vtable
        std::cout << "Meow!\n";
    }
};
```

Vtables:

```
AnimalVTable:
  [0] -> &Animal::~Animal
  [1] -> &Animal::speak (pure stub)

DogVTable:
  [0] -> &Dog::~Dog
  [1] -> &Dog::speak       <- overridden; points to Dog's implementation

CatVTable:
  [0] -> &Cat::~Cat
  [1] -> &Cat::speak       <- overridden; points to Cat's implementation
```

The indices are *stable*: a call to `p->speak()` always uses index [1], regardless of whether `p` points to a `Dog` or `Cat`. At runtime, the call loads `vptr[1]`, which may be `&Dog::speak` or `&Cat::speak`, and jumps there.

If a derived class *adds* a new virtual method:

```cpp
class Dog : public Animal {
public:
    void speak() const override { /* ... */ }
    virtual void fetch() const { /* ... */ }   // new virtual
};
```

DogVTable:

```
  [0] -> &Dog::~Dog
  [1] -> &Dog::speak
  [2] -> &Dog::fetch       <- new slot
```

But now a `Dog*` cannot be passed to code that expects `Animal*` and calls some imaginary `p->fetch()` — `Animal`'s vtable does not have a [2] slot. This is why C++ has *static* polymorphism via templates and *dynamic* polymorphism via inheritance: dynamic polymorphism requires all types to agree on method signatures and slot indices.

### Vtable layout inheritance

The derived-class vtable is a modified copy of the base vtable:

```cpp
class Shape { virtual void draw() = 0; virtual void fill() = 0; };
class Polygon : public Shape { void draw() override; virtual void setBorder(); };
class Triangle : public Polygon { void setBorder() override; };

ShapeVTable:       PolygonVTable:      TriangleVTable:
  [0] dtor          [0] dtor            [0] dtor
  [1] draw          [1] draw (overridden) [1] draw (inherited from Polygon)
  [2] fill          [2] fill            [2] fill
                    [3] setBorder       [3] setBorder (overridden)
```

---

## 8.4 Virtual Destructors

The destructor is always in vtable slot [0] — special, but not magical.

Here's why it must be virtual:

```cpp
class Base {
public:
    ~Base() { std::cout << "~Base\n"; }  // not virtual
    virtual void work() = 0;
};

class Derived : public Base {
public:
    Derived() { p = new int[1000]; }
    ~Derived() {
        delete[] p;
        std::cout << "~Derived\n";
    }
private:
    int* p;
};

int main() {
    Base* b = new Derived();
    delete b;   // calls which destructor?
}
```

Without `virtual ~Base()`: the compiler generates `delete` as:

```asm
mov    rdi, [rsi]        ; rsi = b; load vptr
mov    rax, [rdi + ?]    ; load destructor slot [0]
                         ; BUT — Base has no virtual destructor!
call   &Base::~Base      ; direct call to Base's destructor
```

Result: only `~Base()` runs. `p` is never freed. Memory leak. Worse, if `Derived` held a file handle or lock, it would never be released.

With `virtual ~Derived()`: both destructors run because the vtable entry is overridden:

```
BaseVTable:
  [0] -> &Base::~Base

DerivedVTable:
  [0] -> &Derived::~Derived    <- VIRTUAL, so loaded from vptr
```

At runtime, loading from `vptr[0]` gives `&Derived::~Derived`, which runs first; then the compiler-generated cleanup calls the base destructor.

**Rule of thumb:** Any class with a virtual method needs a virtual destructor, even if it does nothing:

```cpp
class Base {
public:
    virtual ~Base() = default;   // empty, but required
    virtual void work() = 0;
};
```

---

## 8.5 Multiple Inheritance

A class with multiple base classes has *multiple vptrs* — one for each base.

```cpp
class IDrawable {
public:
    virtual ~IDrawable() = default;
    virtual void draw() = 0;
};

class IPhysical {
public:
    virtual ~IPhysical() = default;
    virtual void applyForce(double f) = 0;
};

class Ball : public IDrawable, public IPhysical {
public:
    void draw() override { /* ... */ }
    void applyForce(double f) override { /* ... */ }
private:
    double mass;
};
```

Layout of `Ball`:

```
offset  0:  vptr_IDrawable  -> BallVTable_IDrawable
offset  8:  vptr_IPhysical  -> BallVTable_IPhysical
offset 16:  mass (double)
```

Two vtables:

```
BallVTable_IDrawable:
  [0] -> &Ball::~Ball
  [1] -> &Ball::draw

BallVTable_IPhysical:
  [0] -> &Ball::~Ball
  [1] -> &Ball::applyForce
```

When you call a method through the `IDrawable*` interface:

```cpp
IDrawable* d = new Ball();
d->draw();
```

The compiler uses the first vptr (offset 0). When you call through `IPhysical*`:

```cpp
IPhysical* p = new Ball();
p->applyForce(5.0);
```

The vptr is offset 8.

### Pointer adjustment (thunks)

If you cast from `Ball*` to `IPhysical*`, the pointer must be *adjusted*:

```cpp
Ball* b = new Ball();
IPhysical* p = b;   // pointer must be adjusted by +8 bytes
```

Under the hood, the compiler emits code to adjust `rdi` (the pointer) by 8, so it points to the `vptr_IPhysical` slot instead of the object's start.

When you call `p->applyForce()`, the vptr is loaded from that adjusted offset, and the call dispatch works correctly.

**Thunks** are small wrapper functions that adjust `this` before calling the actual implementation:

```asm
; Thunk for Ball::~Ball called via IPhysical interface
thunk_Ball_dtor_IPhysical:
    sub rdi, 8          ; adjust this pointer back to object start
    jmp &Ball::~Ball    ; tail-call the real destructor
```

This is automatic; you do not write thunks. The compiler inserts them in the vtable.

### Diagram: multiple inheritance

```
Memory:

Stack:  Ball* b = new Ball()
        ---
        b points to: [offset 0]

Heap:
        +--------+
    0:  |vptr_D| ----+
        +--------+    |
    8:  |vptr_P| --+ |
        +--------+  | |
   16:  |mass   |   | |
        +--------+  | |
                    | |
.rodata:            | |
        +-----------+-+  BallVTable_IDrawable
        | [0] -> &Ball::~Ball (thunk adjusts +8)
        | [1] -> &Ball::draw
        +---+
            |
            +----------+  BallVTable_IPhysical
                       | [0] -> &Ball::~Ball
                       | [1] -> &Ball::applyForce
```

---

## 8.6 RTTI (Run-Time Type Information)

`typeid()` and `dynamic_cast` require knowing the actual type of an object at runtime. This type information is stored *in the vtable* (or alongside it).

Modern implementations store a pointer to a `std::type_info` object in a *hidden slot* (often slot [-1]) of each vtable:

```cpp
Ball b;
const std::type_info& ti = typeid(b);
std::cout << ti.name() << "\n";  // output: "Ball" or mangled name
```

At runtime, the compiler generates:

```cpp
// typeid(b) becomes:
const std::type_info* info = b.vptr[-1];  // hidden slot
return *info;
```

The cost: one extra load (to fetch the type_info pointer from the hidden slot) and then possibly a name string comparison if you call `.name()`.

`dynamic_cast` is more expensive:

```cpp
IDrawable* d = new Ball();
Ball* b = dynamic_cast<Ball*>(d);  // legal cast?
```

Implementation:

1. Load the type_info from both the source object and the target type.
2. Check if they match or if there's an inheritance relationship.
3. If yes, return a possibly-adjusted pointer; if no, return nullptr.

This involves string comparison or a traversal of the class hierarchy. Not terrible, but more expensive than a vtable call. (Some implementations use a hash or bitmap for fast rejection.)

**Cost:** A `dynamic_cast` is roughly 10-100 cycles depending on inheritance depth and implementation. Use it for *rare* type checks, not in hot loops.

---

## 8.7 Cost vs Direct Call

Let us quantify:

| Call type | Best case | Typical | Worst case | Notes |
|-----------|-----------|---------|-----------|-------|
| Direct call (e.g., `foo()`) | 1 cycle | 2-3 | 5 | Inlinable; branch predicted. |
| Virtual call, monomorphic | 3-5 | 5-10 | 20+ | Same type at call site; predictor learns it. |
| Virtual call, polymorphic | 5 | 15 | 30+ | Different types; branch mispredict; cache miss. |
| `dynamic_cast` | 10 | 50-100 | 100+ | Type check; string/hierarchy traversal. |
| Virtual dtor + derived dtor | 5 | 10 | 20 | Two calls; first dispatched, second direct. |

**The myth:** "vtable lookups are slow."

**The reality:** The lookup itself (two loads) is 5 cycles in the fast path. The real cost is:

1. **Missed inlining.** Direct calls are inlined; virtual calls are not. Inlining enables further optimizations that dwarf the vtable cost.
2. **Branch misprediction.** If the call site is polymorphic, the predictor thrashes, costing 15+ cycles.
3. **Cache misses.** If the vtable or the function code is cold, you pay memory latency.

In a tight loop calling the same virtual method on the same type, the predictor learns the pattern, cache is hot, and the vtable overhead is negligible — comparable to a direct call.

In a loop calling different types at each iteration, you pay the mispredict penalty repeatedly.

---

## 8.8 Devirtualization

The compiler can **devirtualize** a virtual call — replace it with a direct call — if it can prove the type statically.

```cpp
Circle c(5);
c.draw();   // compiler knows c is a Circle, not a Shape*
```

The compiler can emit a direct call to `Circle::draw`, skipping the vtable entirely.

More sophisticated devirtualization:

```cpp
void process(Shape& s) {
    s.draw();  // virtual dispatch
}

void main() {
    Circle c(5);
    process(c);  // can the compiler devirtualize?
}
```

Modern compilers with **whole-program optimization** (LTO, link-time optimization) can see that `process` is *only* called with `Circle`, and devirtualize the call inside `process`. The cost: longer compile time.

At `-O2` without LTO, most compilers cannot devirtualize polymorphic calls — they play it safe.

At `-O3` or with `-flto`, many can.

### Final classes

Marking a class `final` is a hint to devirtualize:

```cpp
class FinalCircle final : public Shape {
public:
    void draw() const override { /* ... */ }
};
```

Any virtual call on a `FinalCircle` can be devirtualized because the compiler *knows* there are no further subclasses. Some compilers treat this as a devirtualization hint; others do not.

---

## 8.9 Worked Example

Let us define `Shape`, `Circle`, `Square` and inspect the compiled binary:

```cpp
// shapes.h
#pragma once

class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
    virtual double area() const = 0;
};

class Circle : public Shape {
private:
    double radius;
public:
    Circle(double r) : radius(r) {}
    void draw() const override;
    double area() const override;
};

class Square : public Shape {
private:
    double side;
public:
    Square(double s) : side(s) {}
    void draw() const override;
    double area() const override;
};

// shapes.cpp
#include "shapes.h"
#include <iostream>

void Circle::draw() const {
    std::cout << "Drawing circle with radius " << radius << "\n";
}

double Circle::area() const {
    return 3.14159 * radius * radius;
}

void Square::draw() const {
    std::cout << "Drawing square with side " << side << "\n";
}

double Square::area() const {
    return side * side;
}

// main.cpp
#include "shapes.h"
#include <iostream>

int main() {
    Circle c(5.0);
    Square s(4.0);

    Shape* shapes[] = {&c, &s};

    for (int i = 0; i < 2; ++i) {
        shapes[i]->draw();
        std::cout << "Area: " << shapes[i]->area() << "\n";
    }

    return 0;
}
```

Compile with debugging info:

```bash
g++ -g -O2 -o shapes shapes.cpp main.cpp
```

Inspect the vtables:

```bash
objdump -d shapes | grep -A 50 "vtable"
```

Or, better, use `nm` to find vtables:

```bash
nm -C shapes | grep vtable
```

Output (mangled names expanded with `-C`):

```
0000000000004000 d Circle::_ZTV6Circle
0000000000004020 d Square::_ZTV6Square
0000000000004040 d Shape::_ZTV5Shape
```

The `d` flag means "data in initialized segment". Now use `objdump` to read the vtable:

```bash
objdump -s --section=.rodata shapes | grep -A 30 "vtable"
```

You will see the vtable contents — the 8-byte pointers to each virtual method, laid out in order.

For a more direct view, use **Godbolt** (godbolt.org):

1. Paste the code.
2. Set compiler to `g++ 14.1`.
3. Set `-O2`.
4. Click "Demangle identifiers" to see unmangled names.
5. Search for `.LC0:` (start of `.rodata` section) and scroll down — you will see the vtable addresses.

---

## 8.10 Tradeoffs

| Approach | Cost | Benefit | When to use |
|----------|------|---------|-------------|
| Virtual methods | 5-15 cycles per call; code size for vtables | Polymorphic behavior; decoupled implementations | Default for extensible types (Shape hierarchy) |
| Direct calls | 1-2 cycles | Inlinable; predictable | Known static type; performance-critical path |
| Final classes | Slight code-generation cost | Enables devirtualization | Explicitly seal a hierarchy to allow optimization |
| Non-virtual destructor | None | Smaller vtable | Only in base classes with no virtual methods; risky |
| Multiple inheritance | Multiple vptrs; pointer adjustment overhead | Multiple interfaces per object | Explicit interface separation (IDrawable + IPhysical) |
| RTTI enabled | Type_info pointers in vtable; dynamic_cast cost | Runtime type checking; safer downcasts | When polymorphic types need identity checks; validation code |
| RTTI disabled (`-fno-rtti`) | Smaller vtables; faster devirtualization | Smaller binary, slightly faster virtual calls | Embedded / bare-metal code; no `typeid` or `dynamic_cast` |

---

## 8.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Virtual functions are always slow." | The vtable lookup itself (~5 cycles) is fast. The cost comes from missed inlining and branch misprediction on polymorphic call sites. |
| "Each virtual method adds a vtable." | No. One vtable per *class*, with one entry per virtual method. Derived classes reuse the base vtable structure. |
| "Virtual calls prevent inlining." | Correct. The call target is unknown until runtime, so inlining is impossible. This is the real performance tax. |
| "Virtual destructors always slow you down." | The dtor itself is called once per object, not per method call. The vtable lookup for the dtor is negligible in practice. |
| "Multiple inheritance creates multiple vtables." | Correct; one vtable per base class hierarchy. Pointer adjustment (thunks) handles casts. |
| "RTTI is free." | No. `typeid` adds type_info pointers; `dynamic_cast` does runtime type checking. Use them sparingly in hot code. |
| "You can't know the performance of a virtual call without profiling." | True and false. The *best case* is predictable (~5 cycles). The *worst case* depends on your data patterns and hardware. Profile to find which. |

---

## 8.12 Exercises

1. **Implement a simple vtable manually.**
   Write a C program that manually implements the vtable dispatch pattern:
   ```c
   typedef struct Shape {
       struct ShapeVTable* vptr;
   } Shape;
   
   typedef struct {
       void (*draw)(Shape*);
       double (*area)(Shape*);
   } ShapeVTable;
   
   typedef struct {
       Shape base;
       double radius;
   } Circle;
   
   void circle_draw(Shape* s) { /* ... */ }
   double circle_area(Shape* s) { /* ... */ }
   
   ShapeVTable circle_vtable = {
       .draw = circle_draw,
       .area = circle_area,
   };
   ```
   Implement `Circle` and `Square`, then call methods through a `Shape*` pointer. Compile and trace the vtable dispatch in the assembly.

2. **Inspect a vtable with objdump.**
   Compile this:
   ```cpp
   class A { virtual void f() = 0; virtual void g() = 0; };
   class B : public A { void f() override; void g() override; };
   int main() { B b; }
   ```
   Use `objdump -C -t <binary>` to find the vtable symbols. Use `objdump -s --section=.rodata <binary>` to see the vtable bytes. How many entries? Which are which?

3. **Virtual destructor experiment.**
   Write two versions of a class with resources:
   - Version A: non-virtual destructor.
   - Version B: virtual destructor.
   
   In each, define a derived class that holds a pointer to dynamically allocated memory. In `main`, delete through a base pointer. In which version is the memory freed?

4. **Polymorphic call sites and branch prediction.**
   Write a loop that calls `shape->area()` on an array of 1000 shapes, where half are circles and half are squares. Time the loop:
   - Version A: interleaved (circle, square, circle, square, ...).
   - Version B: grouped (all circles, then all squares).
   
   Version B should be faster (better branch prediction). By how much? Why?

5. **Devirtualization with final.**
   Compile this with and without `-flto`:
   ```cpp
   class Shape { public: virtual void draw() = 0; };
   class FinalCircle final : public Shape { void draw() override; };
   
   int main() {
       FinalCircle c;
       for (int i = 0; i < 1'000'000; ++i) c.draw();
   }
   ```
   Without LTO, is the loop inlined? With LTO? Use `-O2 -S -masm=intel` and search for the `call` instruction to `draw()`.

6. **RTTI cost.**
   Write a loop calling `dynamic_cast<Derived*>(p)` on a polymorphic base pointer, where the cast succeeds. Compare to a version using a virtual method `getType()` to identify the object without casting. Which is faster?

7. **Conceptual: design tradeoffs.**
   A colleague proposes removing virtual methods from a large class hierarchy to improve performance. Discuss:
   (a) What performance is gained? (Avoid vtable lookups, enable inlining.)
   (b) What design flexibility is lost? (Polymorphism; late binding.)
   (c) What alternatives exist? (Template metaprogramming; CRTP; static polymorphism.)
   (d) When would each choice make sense?

---

## 8.13 Summary

A vtable is a static array of function pointers, one per class. Each instance carries a vptr to its class's vtable. A virtual method call is two loads (vptr, then function pointer) and an indirect call — approximately 5 cycles in the fast path, 15+ cycles if the branch predictor mispredicts.

The cost is usually not the lookup itself, but the inability to inline virtual calls and the branch prediction penalty when a call site is polymorphic. Virtual destructors are essential in any class with virtual methods to ensure derived destructors run. Multiple inheritance adds multiple vptrs and pointer adjustments (thunks). RTTI adds type information pointers and enables runtime type checking. Modern compilers can devirtualize calls on `final` classes or when whole-program optimization is enabled.

Understanding vtables — where they live (`.rodata`), how they dispatch (two loads), when they thrash the branch predictor (polymorphic call sites) — removes the mystery from C++'s dynamic polymorphism and lets you reason about performance empirically rather than by superstition.

---

> **[← Previous: Polymorphism Internals](07-polymorphism-internals.md)** · **[↑ Part 3](README.md)** · **[Next: Static vs Dynamic Dispatch →](09-static-vs-dynamic-dispatch.md)**
