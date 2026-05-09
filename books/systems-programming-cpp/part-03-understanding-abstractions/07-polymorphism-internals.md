# Chapter 29 — Polymorphism Internals

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish four distinct mechanisms that all go by the name "polymorphism" — and predict which one is in play when the same code does different things on different types.
2. Explain how subtype polymorphism (virtual dispatch) works at runtime: the vptr, the vtable, and the cost of an indirect call.
3. Describe how parametric polymorphism (templates and generics) instantiate: compile-time vs. runtime, code bloat, and why generics in Java are not the same as templates in C++.
4. Trace overload resolution at compile time, and recognize where SFINAE and concepts prevent invalid specializations.
5. Understand structural polymorphism: how Go's itables are built and why they cost what they cost.
6. Predict performance: which dispatch happens at compile time (zero cost), which happens at runtime (one indirect load + branch), and which causes cache pressure.
7. Choose the right mechanism for a given problem: when to use virtual, when to use templates, when to use `std::variant`, when to just overload.

---

## The Four Polymorphisms

"Polymorphism" means "many shapes." In practice, it covers four distinct mechanisms. Each has different costs, different guarantees, and different use cases. **Understanding which one you're using is half the battle.**

### 1. Ad-Hoc Polymorphism (Overloading)

Same name, different code for different types. Resolved entirely at compile time.

```cpp
void print(int x) { std::cout << "int: " << x; }
void print(double x) { std::cout << "double: " << x; }
void print(const std::string& x) { std::cout << "string: " << x; }

print(42);        // calls print(int)
print(3.14);      // calls print(double)
print("hello");   // calls print(const std::string&)
```

The compiler picks the function during overload resolution. At runtime, there is no ambiguity — each call site is compiled to a direct jump to the correct function. **Cost: zero runtime dispatch.** **Drawback: new overloads must be written before the code is compiled.**

Operator overloading is ad-hoc polymorphism. So is function template specialization via SFINAE or concepts.

### 2. Parametric Polymorphism (Generics / Templates)

Write code once, it works for many types. In C++, this is templates. In Java, this is (nominally) generics, but type erasure breaks the abstraction.

```cpp
template <typename T>
void swap(T& a, T& b) {
    T temp = a;
    a = b;
    b = temp;
}

int x = 1, y = 2;
swap(x, y);           // instantiates swap<int>

std::string a = "hello", b = "world";
swap(a, b);           // instantiates swap<std::string>
```

The compiler instantiates the template body once for each distinct type at compile time. Each instantiation is a separate function. **Cost: zero runtime dispatch, but binary bloat and longer compile times.** **Drawback: code generation is visible at runtime (different instances may have different behavior).**

### 3. Subtype Polymorphism (Virtual Dispatch)

Base class pointer or reference holds a derived class object. Calls to virtual functions are resolved at runtime based on the actual type.

```cpp
class Shape {
public:
    virtual double area() = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double r;
public:
    double area() override { return 3.14159 * r * r; }
};

class Rectangle : public Shape {
    double w, h;
public:
    double area() override { return w * h; }
};

Shape* shapes[100];
shapes[0] = new Circle(5);
shapes[1] = new Rectangle(3, 4);

for (auto s : shapes) {
    std::cout << s->area() << "\n";  // calls Circle::area or Rectangle::area?
}
```

At runtime, the call to `area()` looks up the actual type of the object and calls the right function. **Cost: one memory load (vptr) + one indirect jump. Cheap, but not free.** **Benefit: new types can be added after compilation.**

### 4. Structural Polymorphism (Row Polymorphism / Duck Typing)

Code works on anything that has the right shape — not because of inheritance, but because it has the right methods or fields. Go's interfaces are the canonical example.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type File struct { /* ... */ }
func (f *File) Read(p []byte) (int, error) { /* ... */ }

type Buffer struct { /* ... */ }
func (b *Buffer) Read(p []byte) (int, error) { /* ... */ }

func consume(r Reader) {
    buf := make([]byte, 1024)
    r.Read(buf)
}

consume(&File{})    // works; File has Read
consume(&Buffer{})  // works; Buffer has Read
```

At compile time, the code does not know whether a `Reader` is a `File` or a `Buffer`. At runtime, an "itab" (interface table) is created, lazily, for each (interface, concrete type) pair, and holds pointers to the methods. **Cost: one memory load (itab) + one indirect jump, similar to virtual dispatch.** **Benefit: types opt in by shape, not by declaration.**

---

## How Subtype Polymorphism Is Implemented

Virtual dispatch is the most common runtime polymorphism in C++. Understanding it *exactly* — not just "vptr + vtable," but the cost, the branch prediction story, and the cache behavior — is essential.

### The Mechanism: vptr and vtable

When a class has a virtual function, the compiler generates an extra hidden pointer, the **vptr**, at the beginning of every object:

```cpp
class Shape {
public:
    virtual double area() = 0;
    virtual ~Shape() = default;
};

// In memory, a Shape looks like:
// [vptr] [data members]
```

The vptr points to a vtable — a static table of function pointers:

```cpp
// In compiled code, the vtable for Circle looks like:
// void* vtable_Circle[] = {
//   &Shape::~Shape,        // vtable[0] = destructor
//   &Shape::~~DELETING_DESTRUCTOR,  // vtable[1] = deleting destructor
//   &Circle::area,         // vtable[2] = area
//   ...
// };
```

When you call `shape->area()` where `shape` is a `Shape*`:

1. Load the vptr from the first bytes of the object. (One memory load.)
2. Index into the vtable using a compile-time-known offset. (Second memory load.)
3. Jump indirectly to the function pointer. (One indirect jump.)

In assembly, it looks like:

```asm
; rdi = shape pointer (by SysV calling convention)
mov     rax, [rdi]          ; rax = vptr
mov     rax, [rax + 8]      ; rax = function pointer at vtable[1] (area is offset 8)
call    rax                 ; indirect call
```

**Cost per call: two memory loads + one indirect jump.** On modern CPUs with L1 caches and speculative execution, this costs roughly 5–10 cycles when the branch is predicted correctly.

### The Branch Predictor Story

Modern CPUs have branch predictors that track indirect jumps. If `area()` is called repeatedly on the same type, the predictor learns the target and speculates ahead. **When the pattern is consistent (e.g., a loop of Circles followed by a loop of Rectangles), the predictor is nearly perfect and the call is essentially free.**

But if the pattern is unpredictable — alternating between many types randomly — the predictor mispredicts, flushes the pipeline, and costs 15+ cycles per call. This is the real cost of virtual dispatch: not the mechanism itself, but the branch misprediction penalty when the call target is not stable.

Example of good branch prediction:

```cpp
for (auto s : shapes) {
    s->area();  // predictor learns: mostly Circles, then Rectangles
}
```

Example of bad branch prediction:

```cpp
for (int i = 0; i < 100; i++) {
    for (int j = 0; j < shapes.size(); j++) {
        shapes[j]->area();  // random order every iteration
    }
}
```

### Vtable Size and Method Ordering

The vtable contains an entry for every virtual function. For a class hierarchy that is five levels deep with 50 virtual functions across all base classes, the vtable is 50 pointers. The vptr adds 8 bytes per object. For large objects, this is negligible. For small objects (e.g., a `Point` with only `x` and `y` doubles), the vptr can double the object size.

Virtual functions are indexed by declaration order, which means adding a new virtual function to a base class in the middle of the hierarchy can shift all the indices of methods below it — a form of fragility called **virtual function shadowing**. In practice, this rarely matters because vtables are built at link time and methods are indexed consistently.

---

## How Parametric Polymorphism Is Implemented

Templates in C++ instantiate at compile time. Generics in Java use type erasure at runtime. These are *not* the same thing.

### C++ Templates: Compile-Time Instantiation

When you write:

```cpp
template <typename T>
T max(T a, T b) {
    return a > b ? a : b;
}
```

And use it:

```cpp
int x = max(3, 5);           // instantiates max<int>
double y = max(3.14, 2.71);  // instantiates max<double>
```

The compiler generates *two separate functions* in the object file:

```cpp
// max<int>: (compiled separately)
int max_int(int a, int b) {
    return a > b ? a : b;
}

// max<double>: (compiled separately)
double max_double(double a, double b) {
    return a > b ? a : b;
}
```

Each instantiation is compiled with full type information, so the compiler can:
- Inline the comparison and branch.
- Use the most efficient machine instruction for the type.
- Specialize the code path (e.g., use SIMD for vectors).

**Cost: zero runtime dispatch.** The call to `max(x, y)` is a direct function call, no different from a non-template function.

**Drawback: binary bloat.** If you instantiate `max<T>` for 20 different types, you get 20 copies of the function body in the binary. For large templates (containers, algorithms), this bloat can be severe. C++2026 is exploring approaches to reduce this via "template modules," but the current state is: templates trade compile time and binary size for runtime speed.

### Java Generics: Type Erasure at Runtime

Java compiles generic code like this:

```java
public static <T> T max(T a, T b) {
    return ((Comparable<T>) a).compareTo(b) > 0 ? a : b;
}

Integer x = max(3, 5);           // calls max<Object>, result cast to Integer
Double y = max(3.14, 2.71);      // calls max<Object>, result cast to Double
```

The compiler **erases** the type parameter and replaces it with the bound (usually `Object`):

```java
// At runtime, only ONE function:
public static Object max(Object a, Object b) {
    return ((Comparable) a).compareTo(b) > 0 ? a : b;
}
```

The generic code compiles to:

1. Upcast arguments to `Object` (free — references have the same shape).
2. Call the single max function.
3. Downcast the result (free at runtime; may throw at runtime).

**Cost: boxing/unboxing and dynamic dispatch** (since `compareTo` is a virtual call). **Benefit: no binary bloat; one copy of the code for all types.**

The difference is profound: Java generics are a *compile-time* convenience that erases to runtime polymorphism, whereas C++ templates are a *compile-time code generation* mechanism. C++ templates can be inlined and optimized across type boundaries; Java generics cannot.

### Function Templates with Explicit Specialization

You can specialize a template for a specific type:

```cpp
template <typename T>
void sort(T* arr, int n) {
    // slow generic sort
}

template <>
void sort<int>(int* arr, int n) {
    // fast radix sort for integers
}
```

The compiler generates the generic version for all types *except* `int`, and generates the specialized version for `int`. This is ad-hoc polymorphism hidden inside parametric polymorphism: the function name is the same, but the implementation differs.

---

## How Ad-Hoc Polymorphism Is Implemented

Overload resolution happens at compile time. The compiler picks the best match according to a strict set of rules, and generates a direct call to the chosen function.

### Overload Resolution Rules

Given a call like:

```cpp
print(42);
print(3.14);
print("hello");
```

The compiler performs **overload resolution**:

1. Find all functions named `print` in scope.
2. For each candidate, compute the conversion cost from the argument type to the parameter type.
3. Pick the candidate with the lowest cost.

Conversion costs are ordered:
- Exact match (cost 0)
- Promotions (cost 1): `int` → `long`, `float` → `double`
- Standard conversions (cost 2): `int` → `double`
- User-defined conversions (cost 3): `MyClass` → `int` via `operator int()`

```cpp
void print(int);     // cost for print(42) = 0 (exact match)
void print(long);    // cost for print(42) = 1 (promotion)
void print(double);  // cost for print(3.14) = 0 (exact match)

print(42);      // best match: print(int), cost 0
print(3.14);    // best match: print(double), cost 0
```

If multiple candidates tie, the overload is **ambiguous** and the compiler rejects the code.

### Operator Overloading

Operator overloading is just overloading of special function names:

```cpp
struct Complex {
    double real, imag;
    
    Complex operator+(const Complex& other) const {
        return Complex{real + other.real, imag + other.imag};
    }
    
    Complex operator*(double scalar) const {
        return Complex{real * scalar, imag * scalar};
    }
};

Complex c{1, 2};
Complex d = c + Complex{3, 4};  // calls c.operator+(Complex{3, 4})
Complex e = c * 2.5;            // calls c.operator*(2.5)
```

The compiler rewrites `c + d` as `c.operator+(d)` and `c * 2.5` as `c.operator*(2.5)`. Overload resolution picks the best match. **No runtime dispatch — the call is direct.**

### SFINAE and Concepts as Guards

SFINAE is "Substitution Failure Is Not An Error." It allows you to write multiple template overloads and have the compiler silently discard specializations that do not work:

```cpp
// Overload 1: works for any type with operator<
template <typename T>
std::enable_if_t<std::is_same_v<decltype(std::declval<T>() < std::declval<T>()), bool>>
sort(T* arr, int n) {
    // general-purpose sort
}

// Overload 2: specialized for int
template <>
void sort<int>(int* arr, int n) {
    // fast radix sort
}

std::vector<int> v = {3, 1, 4};
sort(v.data(), v.size());  // picks overload 2 (exact specialization)
```

In C++20, concepts make this cleaner:

```cpp
template <typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};

template <Comparable T>
void sort(T* arr, int n) {
    // general sort for anything with <
}

template <>
void sort<int>(int* arr, int n) {
    // fast radix sort
}
```

The compiler checks if `T` satisfies `Comparable` and picks the specialization or the general version accordingly. **Still zero runtime cost.** The dispatch happens at compile time, during template instantiation.

---

## How Structural Polymorphism Is Implemented

Go's interfaces are the canonical example. A type does not explicitly inherit from an interface; it just needs to have the right methods.

### The itab: Interface Table

When a concrete type (say, `*File`) is assigned to an interface type (say, `Reader`), Go creates an **itab** — interface table — a small structure containing:

```go
type itab struct {
    inter *interfaceType  // metadata about Reader interface
    _type *runtimeType    // metadata about *File type
    hash  uint32          // cached hash for faster type assertions
    fun   [1]uintptr      // function pointers; fun[0] is Read
}
```

The itab is allocated and cached in a global hash table. If the same (interface, type) pair is assigned again, the cached itab is reused.

### Runtime Method Lookup

When you call `r.Read(buf)` where `r` is a `Reader`:

1. Load the itab from the interface value. (One memory load.)
2. Index into the `fun` array to get the method pointer. (Second memory load.)
3. Jump indirectly to the method. (One indirect jump.)

This is almost identical to C++ virtual dispatch:

```go
// conceptually:
r.Read(buf)
// compiles to:
itab := r.itab                    // load itab
Read := itab.fun[0]              // load Read function pointer
Read(r.data, buf)                 // call Read
```

**Cost: same as virtual dispatch — two memory loads + indirect jump.**

### Type Assertions

You can check the runtime type and cast:

```go
if f, ok := r.(*File); ok {
    // r really is a *File
}
```

This is compiled to:

```go
// load itab, check if it matches *File, cast if it does
```

Type assertions are not free — they involve a runtime type check — but they are rare compared to method calls, so the cost is negligible.

### Structural vs. Nominal Typing

The key difference from C++ virtual dispatch: **Go does not require an explicit inheritance relationship.** If `File` has a `Read` method with the right signature, it automatically satisfies `Reader`. This is called **structural** or **row** polymorphism.

C++ requires explicit inheritance (nominal typing):

```cpp
class Reader {
public:
    virtual void Read(char* buf, size_t n) = 0;
};

class File : public Reader {  // must explicitly inherit
public:
    void Read(char* buf, size_t n) override;
};
```

The benefit of structural typing: you can retrofit interfaces without changing existing code. The drawback: it is easier to satisfy an interface by accident (if `MyClass` happens to have a `Read` method with the wrong semantics, it will still match).

---

## Comparing Costs

Here is a table of the four polymorphisms and their costs:

| Mechanism | Dispatch | Cost | Inlinable | Binary Size | When Resolved |
|---|---|---|---|---|---|
| **Ad-hoc** | Compile time | Zero | Yes | Small | Compile time |
| **Parametric (C++)** | Compile time | Zero | Yes | Large (bloat) | Compile time |
| **Parametric (Java)** | Runtime | Moderate | No | Small | Runtime (type erasure) |
| **Subtype** | Runtime (vtable) | ~5–15 cycles | No | Small | Runtime (branch prediction) |
| **Structural** | Runtime (itab) | ~5–15 cycles | No | Small | Runtime (method lookup) |

**Key insight:** Compile-time dispatch (ad-hoc and parametric) can be inlined, moved, and optimized away entirely. Runtime dispatch (subtype and structural) cannot.

But runtime dispatch is not always bad. The branch predictor makes repeated calls to the same type nearly free. The cost is *misprediction* — when the type changes frequently and unpredictably.

---

## Worked Example: Computing Area

Let's implement the same operation — computing the area of a shape — three different ways, and compare the code generation.

### Version 1: Virtual Dispatch (Subtype Polymorphism)

```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double r;
public:
    Circle(double r) : r(r) {}
    double area() const override { return 3.14159 * r * r; }
};

class Rectangle : public Shape {
    double w, h;
public:
    Rectangle(double w, double h) : w(w), h(h) {}
    double area() const override { return w * h; }
};

double total_area(const Shape* shapes[], int n) {
    double sum = 0;
    for (int i = 0; i < n; i++) {
        sum += shapes[i]->area();  // virtual call
    }
    return sum;
}
```

Compiled to x86-64 (approximate):

```asm
total_area:
    xor     rax, rax            ; sum = 0
    xor     ecx, ecx            ; i = 0
    test    edx, edx            ; if n == 0?
    jle     .done
.loop:
    mov     rdi, [rsi + rcx*8]  ; rdi = shapes[i]
    mov     rdx, [rdi]          ; rdx = vptr
    mov     rdx, [rdx + 0]      ; rdx = vtable[0] = area method
    call    rdx                 ; indirect call to area()
    addsd   xmm0, rax           ; sum += return value (in xmm0 for double)
    inc     rcx
    cmp     rcx, rdx            ; i < n?
    jl      .loop
.done:
    ret
```

**Cost per iteration: one indirect call + branch prediction.**

### Version 2: Templates (Parametric Polymorphism)

```cpp
template <typename Shape>
double area(const Shape& s) {
    return s.area();
}

double total_area(const Circle circles[], int n) {
    double sum = 0;
    for (int i = 0; i < n; i++) {
        sum += area(circles[i]);  // direct call, likely inlined
    }
    return sum;
}
```

Compiled to x86-64 (approximate):

```asm
total_area:
    xor     rax, rax            ; sum = 0
    xor     ecx, ecx            ; i = 0
    test    edx, edx
    jle     .done
.loop:
    mov     rsi, [rsi + rcx*8]  ; rsi = circles[i]
    movsd   xmm0, [rsi + 0]     ; xmm0 = r
    mulsd   xmm0, xmm0          ; xmm0 *= xmm0
    mulsd   xmm0, [rip + constant_pi]  ; xmm0 *= pi
    addsd   rax, xmm0           ; sum += result
    inc     rcx
    cmp     rcx, rdx
    jl      .loop
.done:
    ret
```

Notice: the function call to `area()` has been **inlined** and optimized away. The loop body is now just arithmetic. **Cost per iteration: none — just the area calculation.**

### Version 3: std::variant + std::visit (Tagged Union)

```cpp
struct Circle { double r; };
struct Rectangle { double w, h; };

using Shape = std::variant<Circle, Rectangle>;

double area(const Shape& s) {
    return std::visit([](const auto& shape) {
        if constexpr (std::is_same_v<decltype(shape), const Circle&>) {
            return 3.14159 * shape.r * shape.r;
        } else {
            return shape.w * shape.h;
        }
    }, s);
}

double total_area(const Shape shapes[], int n) {
    double sum = 0;
    for (int i = 0; i < n; i++) {
        sum += area(shapes[i]);  // calls area, which visits the variant
    }
    return sum;
}
```

Compiled to x86-64 (approximate):

```asm
total_area:
    xor     rax, rax            ; sum = 0
    xor     ecx, ecx            ; i = 0
.loop:
    mov     rdx, [rsi + rcx*20] ; rdx = shapes[i].index (discriminant)
    mov     rdi, [rsi + rcx*20 + 8]  ; rdi = shapes[i].data
    cmp     rdx, 0              ; which type?
    je      .is_circle
.is_rectangle:
    movsd   xmm0, [rdi + 0]     ; w
    mulsd   xmm0, [rdi + 8]     ; * h
    jmp     .add
.is_circle:
    movsd   xmm0, [rdi + 0]     ; r
    mulsd   xmm0, xmm0          ; * r
    mulsd   xmm0, [rip + constant_pi]
.add:
    addsd   rax, xmm0
    inc     rcx
    cmp     rcx, rdx
    jl      .loop
    ret
```

A conditional branch replaces the indirect call. **Cost per iteration: one conditional branch (branch prediction).** If the shapes are arranged (all circles first, then all rectangles), the predictor learns the pattern and the branch is nearly free. If they are random, misprediction costs ~15 cycles.

### Comparison

| Version | Dispatch | Cost | Binary |
|---|---|---|---|
| Virtual | Indirect call | ~5–15 cycles (depends on type stability) | Small |
| Template | None (inlined) | ~1 cycle (just arithmetic) | Larger (code duplication per type) |
| Variant | Conditional branch | ~1 cycle (predicted) to ~15 cycles (mispredicted) | Small–medium |

**Insight:** Templates win on speed, but at the cost of code bloat. Virtual dispatch has stable moderate cost. `std::variant` is a middle ground: smaller than templates, faster than virtual when types are stable, but worse if types change randomly.

---

## Tradeoffs

| Mechanism | Pros | Cons | When to Use |
|---|---|---|---|
| **Ad-hoc (overload)** | Zero cost; inlinable; flexible | Must declare all overloads upfront | When you know all types at compile time (operators, numeric functions) |
| **Parametric (C++ template)** | Zero cost; inlinable; optimal codegen | Binary bloat; longer compile times; hard to debug | Containers, algorithms, generic numeric code |
| **Parametric (Java generic)** | No binary bloat | Runtime type erasure; boxing; slower | When you can tolerate a type-erased runtime |
| **Subtype (virtual)** | Add new types without recompiling | Runtime cost; branch misprediction hazard; requires inheritance hierarchy | Plugin architectures, framework base classes, large polymorphic systems |
| **Structural (Go interface)** | Loose coupling; can retrofit; simple | Runtime cost; less explicit contracts | When you want implicit contracts and loose coupling |
| **std::variant** | No virtual dispatch; predictable perf | Type-safe; but must enumerate all types upfront; code branching | When you have a small, fixed set of types and want to avoid virtual |

---

## Common Misconceptions

### 1. "Polymorphism is Object-Oriented Programming"

False. Parametric polymorphism (templates, generics) has nothing to do with inheritance, classes, or OOP. You can write highly polymorphic C++ code without a single virtual function or class hierarchy. Templates are polymorphism. Overloading is polymorphism. Neither requires objects.

### 2. "Virtual Functions Are Always Slow"

False. Modern branch predictors make virtual calls nearly free when the call target is stable. The real cost is misprediction. If your code calls `Circle::area()` 1000 times in a row, then `Rectangle::area()` 1000 times, the predictor learns both patterns and the cost is negligible. If your code alternates randomly, the cost is significant.

### 3. "Generics in Java Are Like Templates in C++"

False. Java generics use type erasure — the type is erased at runtime, and the code becomes a single instantiation for all types. C++ templates are instantiated per type at compile time. This has huge implications: C++ templates can be specialized, inlined, and optimized per type; Java generics cannot.

### 4. "std::variant Is a Bad Virtual Function"

Partly true, partly false. `std::variant` is *not* a virtual function — it is a tagged union with explicit switch-like behavior. It can be faster than virtual if the type is stable and the branch predictor learns the pattern. It is slower if the type changes randomly. It requires you to enumerate all possible types upfront, whereas virtual dispatch does not. Use `std::variant` for a fixed set of types; use virtual for extensible hierarchies.

### 5. "Go Interfaces Are More Efficient Than C++ Virtual Functions"

False. Go's itab-based dispatch is almost identical in cost to C++ virtual dispatch — both are one indirect load + one indirect jump. The difference is that Go allows structural typing (you don't have to explicitly inherit), whereas C++ requires nominal typing (you must explicitly inherit from the base class).

---

## Exercises

1. **Write three versions of `max(a, b)`:** one with `if`, one with a template, and one with a function pointer. Compile all three with `-O3` and examine the assembly. Which is fastest? Which is smallest?

2. **Branch Prediction Stress Test:** Write a program that calls a virtual function on an array of mixed shapes (Circle, Rectangle, Triangle in random order). Measure the time. Then sort the array so all Circles come first, then Rectangles, then Triangles. Measure again. By how much does branch prediction improve?

3. **Template Bloat:** Write a template function `print<T>()` that prints a value. Instantiate it with 20 different types (`int`, `double`, `std::string`, custom classes, etc.). Check the object file size. By how much did it grow compared to a non-template version?

4. **std::variant Performance:** Implement a simple calculator using (a) a base class with virtual `eval()`, (b) `std::variant<Add, Subtract, Multiply, Divide>` with `std::visit`, and (c) a switch statement on an enum. Benchmark all three with randomly generated expressions. Which is fastest? Which is most readable?

5. **Overload Resolution:** Write a class `Matrix` with overloaded `operator[]` for int (1D indexing), `operator()` for 2D indexing, and a templated `at<T>()` method for type-safe access. Call all three and verify the compiler picks the right one. Then add a `template <> operator[]<std::string>()` specialization and trace overload resolution.

---

## Summary

"Polymorphism" hides four distinct mechanisms: ad-hoc (overloading), parametric (templates/generics), subtype (virtual dispatch), and structural (duck typing). Each is resolved at a different stage — overloading at compile time during overload resolution, templates at compile time during instantiation, virtual dispatch at runtime via vtable lookup, and structural dispatch at runtime via itab lookup or equivalent. Compile-time dispatch has zero runtime cost and can be inlined away entirely. Runtime dispatch costs one indirect load and one indirect jump, mitigated by branch prediction when the call target is stable. **Choose the mechanism that fits your constraints:** templates when you want zero cost and can tolerate code bloat, virtual dispatch when you need extensibility and can accept the cost, `std::variant` when you have a small fixed set of types, overloading when all possibilities are known at compile time. Understanding which mechanism is in play — and its actual cost on modern hardware — is the difference between polyglot systems design and cargo-cult programming.

---

> **[← Previous: Chapter 28](06-interfaces-and-contracts.md)** · **[↑ Part 3](README.md)** · **[Next: Chapter 30 →](08-virtual-tables.md)**
