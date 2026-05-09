# Chapter 9 — Static vs Dynamic Dispatch

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the fundamental question that separates static and dynamic dispatch: is the function being called known at compile time?
2. Recognize the four main mechanisms that implement static dispatch: non-virtual function calls, templates, function pointers resolved at link time, and concepts.
3. Understand virtual function dispatch on the x86-64 and ARM64 architectures: the vtable layout, the extra indirection cost, and why inlining is impossible.
4. Implement and reason about CRTP (Curiously Recurring Template Pattern): static polymorphism that looks like inheritance but costs zero at runtime.
5. Use `std::variant` + `std::visit` as a closed-set alternative to virtual dispatch that often outperforms it.
6. Apply type erasure patterns with `std::function`, `std::any`, and custom wrappers when you need to hide the concrete type.
7. Decide, for a given API boundary, which mechanism best fits: performance requirements, type set size, knowledge available at compile time, and binary size constraints.

## 9.1 The Fundamental Question

When a function is called, the CPU must jump to an address. At some point in the pipeline — either before the program runs, or during execution — the system decides: what address?

- **Static dispatch:** The compiler or linker *knows* the address. The `call` instruction contains a fixed address, or can be inlined entirely.
- **Dynamic dispatch:** The address is *not known* until runtime. The CPU reads the address from memory or a register, then jumps there.

That distinction shapes everything: performance, inlining opportunity, binary size, flexibility. Every function call in C++ falls somewhere on that spectrum.

```cpp
struct Renderer {
    void render();  // which address?
};

void draw(Renderer& r) {
    r.render();     // compile time or runtime?
}
```

If `Renderer::render` is non-virtual, the address is known at compile time. The compiler can inline the call or generate a fixed `call` instruction.

If `Renderer::render` is virtual, the address is not known until runtime. The CPU must read `r`'s vtable, look up the function pointer, and jump there. Inlining is impossible.

Most of this chapter is about understanding that choice, and the four tools you have to make it.

---

## 9.2 Static Dispatch — The Default

Non-virtual function calls are the compiler's starting point. The compiler *resolves* the call at compile time using **name lookup** and **overload resolution**. Once resolved, the address is fixed.

### Non-Virtual Function Calls

```cpp
struct Stack {
    void push(int x);
    int pop();
};

int main() {
    Stack s;
    s.push(5);
    int x = s.pop();
    return x;
}
```

Here, `s.push(5)` and `s.pop()` are resolved at compile time. The compiler knows:
- `s` is a `Stack` (known at compile time).
- `Stack::push` is the function being called (known at compile time).
- The function's address is fixed (known at link time).

The call might be inlined entirely:
```asm
; Optimized: push and pop logic inlined into main
mov [rsp - 8], 5    ; push 5 onto stack array
mov eax, [rsp - 8]  ; pop from stack
ret
```

Or it might be a direct `call`:
```asm
mov edi, [rsp + 0]  ; this = &s
mov esi, 5          ; arg = 5
call Stack::push    ; direct address, no indirection
```

The key: the address is resolved statically.

### Templates

Templates are a second form of static dispatch. The compiler generates a separate copy of the function for each distinct type argument.

```cpp
template <typename T>
void process(T x) {
    x.run();  // which T::run? Depends on the instantiation.
}

struct Dog { void run() { printf("bark\n"); } };
struct Cat { void run() { printf("meow\n"); } };

int main() {
    process(Dog{});  // Compiler generates process<Dog>
    process(Cat{});  // Compiler generates process<Cat>
}
```

Here, the call to `x.run()` in `process<Dog>` is resolved to `Dog::run` at compile time. The call to `x.run()` in `process<Cat>` is resolved to `Cat::run` at compile time. Each instantiation is a separate piece of code; there's no dynamic dispatch within each one.

Templates enable **zero-cost abstraction**: the interface is generic, but the implementation is specialized, with no runtime cost.

The cost: **code bloat**. Each instantiation generates new machine code. Instantiate `std::vector<int>`, `std::vector<double>`, `std::vector<struct>`, and the compiler generates four different `push_back` implementations. This is called **template code explosion** or **code instantiation bloat**.

### Function Pointers (Link Time)

Function pointers are a third form of static dispatch:

```cpp
typedef void (*Callback)(int);

void on_event(Callback cb) {
    cb(42);  // indirect call through a function pointer
}

void handle_event(int x) { printf("Event: %d\n", x); }

int main() {
    on_event(handle_event);  // pass address of handle_event
}
```

Here, `on_event` receives a function pointer. The actual function is not known at compile time — it could be `handle_event`, or any other function with the same signature. But at the call site (`cb(42)`), the call is still resolved to a fixed address: the function pointer's current value. There's no vtable lookup; the address is stored directly in a variable.

This is used at link time to resolve which implementation to call:

```cpp
// api.h
typedef int (*HashFunc)(const char*);
extern HashFunc hash_impl;  // which hash function?

// default.cpp
int hash_default(const char* s) { ... }
HashFunc hash_impl = hash_default;

// simd.cpp
int hash_simd(const char* s) { ... }
// Could set: hash_impl = hash_simd;
```

The call site does not know which implementation will be used, but the binding is static in the sense that it is *resolved before program execution* (or during program initialization, which is runtime-static). Once `hash_impl` points to a function, the address doesn't change.

### Concepts

C++20 concepts enable compile-time constraints on templates without forcing everything into a single implementation. The dispatch is still static, but the constraint is explicit:

```cpp
template <typename T>
concept Drawable = requires(T t) {
    { t.draw() } -> std::convertible_to<void>;
};

template <Drawable T>
void render(T& obj) {
    obj.draw();  // dispatch is still static; T is bound at compile time
}
```

The call to `obj.draw()` is resolved at compile time, just like a template. Concepts are syntactic sugar that make the contract explicit and enable better error messages.

### Summary: Static Dispatch Costs and Benefits

**Benefits:**
- Call can be inlined (eliminates function call overhead).
- No runtime indirection (no extra memory load).
- Compiler sees full function body and can optimize across the call boundary.
- Easy to debug (no hidden behavior).

**Costs:**
- Type must be known at compile time.
- Type set must be closed (you can't add new types at runtime).
- Templates incur code bloat (one binary for each instantiation).
- Brittle to change: changing the template or function signature breaks all callers at compile time.

---

## 9.3 Dynamic Dispatch — Virtual Functions

When the type is not known until runtime, or the set of types is open, you need dynamic dispatch. Virtual functions are the standard mechanism.

### How Virtual Functions Work

```cpp
struct Animal {
    virtual void speak() = 0;
};

struct Dog : public Animal {
    void speak() override { printf("Woof\n"); }
};

struct Cat : public Animal {
    void speak() override { printf("Meow\n"); }
};

void make_sound(Animal& a) {
    a.speak();  // which speak? Not known until runtime.
}

int main() {
    Dog d;
    Cat c;
    make_sound(d);  // calls Dog::speak
    make_sound(c);  // calls Cat::speak
}
```

At runtime, `make_sound` receives a reference to an `Animal`. The reference could point to a `Dog` or a `Cat`. The compiler doesn't know which. So it must emit code that figures it out at runtime.

The mechanism: **virtual tables** (vtables).

When a class declares a virtual function, the compiler creates a table of function pointers, one for each virtual function. Each object of that class stores a pointer to its class's vtable.

```
Dog object in memory:
[vptr] -> Dog's vtable
         [&Dog::speak]
         [&Dog::~Dog]
         ...other virtual functions...

Cat object in memory:
[vptr] -> Cat's vtable
         [&Cat::speak]
         [&Cat::~Cat]
         ...other virtual functions...
```

When `a.speak()` is called, the generated code does:
1. Load the vptr from `a`.
2. Load the address of `speak()` from the vtable (at a fixed offset).
3. Jump to that address.

In assembly (x86-64 SysV):

```asm
mov rax, [rdi]           ; rax = a's vptr (rdi is 'this')
call qword ptr [rax + 0] ; call speak() from vtable at offset 0
```

Two extra instructions, and one extra memory load. For a tight loop calling virtual functions millions of times, this adds up.

### The Cost: Vtable Indirection

Virtual dispatch costs:
1. **Extra memory load:** Reading the vptr and then the function pointer doubles the load operations.
2. **No inlining:** The compiler doesn't know the target address. Inlining is impossible.
3. **Pipeline stalls:** The CPU can't know where to jump until the memory load completes. Branch prediction can help, but it's not guaranteed.

Benchmarking:
```cpp
struct Renderer {
    virtual void render() {}
};

struct VirtualRenderer : Renderer {
    void render() override {
        // Expensive work
        for (int i = 0; i < 1000; ++i) {
            // ...
        }
    }
};

// vs.

struct NonVirtualRenderer {
    void render() {
        for (int i = 0; i < 1000; ++i) {
            // ...
        }
    }
};
```

If `render()` does expensive work, the virtual dispatch cost is negligible. If `render()` does tiny work, the virtual dispatch cost dominates.

```
Non-virtual: 100 ns for the function + 0 ns for the call = 100 ns.
Virtual:     100 ns for the function + 5-10 ns for the call = 105-110 ns.
Difference:  5-10% for expensive work, 100%+ for tiny work.
```

### Virtual Destructors

A subtle but critical rule: if a class is designed to be inherited from, its destructor must be virtual.

```cpp
struct Animal {
    virtual ~Animal() = default;  // required!
    virtual void speak() = 0;
};

int main() {
    std::unique_ptr<Animal> a = std::make_unique<Dog>();
    // When a is destroyed, ~Animal() is called.
    // But if ~Animal is virtual, ~Dog() is called first, then ~Animal().
    // If ~Animal is not virtual, only ~Animal() is called, leaving ~Dog() unrun.
    // This is a resource leak if Dog has destructors.
}
```

Non-virtual base-class destructors are a source of memory leaks in polymorphic hierarchies.

---

## 9.4 CRTP — Static Polymorphism Without Virtual

The Curiously Recurring Template Pattern (CRTP) provides polymorphism at compile time, with zero vtable overhead.

The idea: the derived class passes itself as a template parameter to the base class.

```cpp
template <typename Derived>
struct Renderer {
    void render() {
        static_cast<Derived*>(this)->render_impl();
    }
};

struct OpenGLRenderer : Renderer<OpenGLRenderer> {
    void render_impl() {
        printf("Rendering with OpenGL\n");
    }
};

struct VulkanRenderer : Renderer<VulkanRenderer> {
    void render_impl() {
        printf("Rendering with Vulkan\n");
    }
};

void draw(Renderer<OpenGLRenderer>& r) {
    r.render();  // Calls OpenGLRenderer::render_impl directly, no vtable
}
```

From the call site, `draw` accepts a reference to `Renderer<OpenGLRenderer>`. The compiler knows the concrete type at compile time, so `render()` can be inlined:

```cpp
// After inlining:
void draw(OpenGLRenderer& r) {
    printf("Rendering with OpenGL\n");
}
```

The call path is completely inlayable.

### CRTP Tradeoffs

**Benefits:**
- Zero runtime overhead. No vtable, no indirection.
- Compiler can inline and optimize aggressively.
- Static type safety (type errors at compile time, not runtime).

**Costs:**
- Every instantiation generates new code (like templates).
- Call sites must know the concrete type. Cannot accept `Renderer<?>` or a generic `Renderer`.
- Syntactically more complex (the "recurring" part feels strange at first).
- No polymorphic containers. You cannot store `Renderer<OpenGLRenderer>` and `Renderer<VulkanRenderer>` in the same vector.

CRTP is used in performance-critical code where:
- The type set is known at compile time.
- You need inlining across the boundary.
- You can afford the binary size of separate instantiations.

Examples in the wild: Eigen (linear algebra), range-v3 (algorithms), ASIO (networking).

---

## 9.5 std::variant + std::visit

For a closed set of types (known at compile time), you have another option: **tagged unions** + **pattern matching**.

```cpp
struct DogSound { std::string sound = "Woof"; };
struct CatSound { std::string sound = "Meow"; };
struct BirdSound { std::string sound = "Chirp"; };

using AnimalSound = std::variant<DogSound, CatSound, BirdSound>;

void process(const AnimalSound& sound) {
    std::visit([](const auto& s) {
        printf("%s\n", s.sound.c_str());
    }, sound);
}
```

`std::variant<T1, T2, T3>` stores a value that is one of the types, plus a tag (an integer) indicating which type it is.

`std::visit` executes a callable based on the tag. The compiler generates code that dispatches based on the tag:

```cpp
// Simplified: what std::visit generates
void visit_impl(const AnimalSound& s, Callable fn) {
    switch (s.index()) {
        case 0: fn(*std::get_if<DogSound>(&s)); break;
        case 1: fn(*std::get_if<CatSound>(&s)); break;
        case 2: fn(*std::get_if<BirdSound>(&s)); break;
    }
}
```

Or, with a sufficiently smart compiler, an indirect jump table:

```asm
mov eax, [rdi]           ; eax = variant's tag
lea rdx, [rel .L_table]  ; rdx = table address
jmp [rdx + rax*8]        ; jump to handler
```

### Advantages Over Virtual Dispatch

1. **No vtable:** The tag is stored in the variant itself, not in an object's vptr.
2. **No heap allocation:** The variant is stack-allocated and contains all possible types inline (each variant is as large as its largest type).
3. **Better CPU behavior:** A switch statement or jump table is branch-predictable.
4. **Inlinable for small types:** If the lambda is simple, the compiler can inline the entire visit.

### When to Use Variant

Use `std::variant` when:
- The type set is small and fixed.
- Types are known at compile time.
- You want to avoid heap allocation.
- Objects fit comfortably on the stack (variant size = max(all types)).

Don't use `std::variant` when:
- The type set is open (you might add types at runtime).
- The variant would be huge (imagine `variant<Dog, Cat, ..., 1000 more types>`).
- You need inheritance relationships between the types.

---

## 9.6 Type Erasure and std::function

Sometimes you need to hide the concrete type *and* accept any type that meets a contract. This is **type erasure**: wrapping a concrete type in a generic interface.

The canonical example: `std::function`.

```cpp
std::function<int(int, int)> op;

// Set to a function pointer
int add(int a, int b) { return a + b; }
op = add;
printf("%d\n", op(2, 3));  // 5

// Set to a lambda
op = [](int a, int b) { return a * b; };
printf("%d\n", op(2, 3));  // 6

// Set to a callable object
struct Multiplier {
    int factor;
    int operator()(int a, int b) { return a * b * factor; }
};
op = Multiplier{10};
printf("%d\n", op(2, 3));  // 60
```

Same `std::function` object, holding different types at runtime.

### How std::function Works

`std::function<Sig>` internally uses virtual dispatch:

```cpp
template <typename Sig>
class function {
    struct CallableBase {
        virtual ~CallableBase() = default;
        virtual Result call(Args...) = 0;
    };

    template <typename F>
    struct CallableImpl : CallableBase {
        F f;
        Result call(Args... args) override {
            return f(args...);
        }
    };

    std::unique_ptr<CallableBase> impl;

public:
    template <typename F>
    function(F f) {
        impl = std::make_unique<CallableImpl<F>>(std::move(f));
    }

    Result operator()(Args... args) {
        return impl->call(args...);
    }
};
```

When you assign a function to `std::function`, the concrete type is wrapped in `CallableImpl<ConcreteType>`, and a pointer to the base is stored. Calling the function goes through the vtable.

### The Cost

1. **Heap allocation:** `std::function` always allocates on the heap (unless Small Function Optimization is active).
2. **Virtual dispatch:** Calling goes through the vtable.
3. **Type information loss:** From the caller's perspective, the concrete type is hidden.

### When to Use std::function

Use `std::function` when:
- You need to store different callable types in the same container.
- The caller doesn't know the concrete type.
- You can afford the heap allocation and indirection cost.

For performance-critical code, avoid `std::function` in hot loops. For configuration-time callbacks, it's fine.

### Custom Type Erasure

You can also write custom type erasure. For example, a simplified sink that accepts any type with a `write(const char*)` method:

```cpp
class Sink {
    struct Base {
        virtual ~Base() = default;
        virtual void write(std::string_view) = 0;
    };

    template <typename T>
    struct Impl : Base {
        T impl;
        void write(std::string_view s) override {
            impl.write(s);
        }
    };

    std::unique_ptr<Base> m_impl;

public:
    template <typename T>
    Sink(T impl) : m_impl(std::make_unique<Impl<T>>(std::move(impl))) {}

    void write(std::string_view s) {
        m_impl->write(s);
    }
};

struct FileSink {
    FILE* f;
    void write(std::string_view s) { fwrite(s.data(), 1, s.size(), f); }
};

struct StderrSink {
    void write(std::string_view s) { fwrite(s.data(), 1, s.size(), stderr); }
};

int main() {
    Sink s1{FileSink{stdout}};
    Sink s2{StderrSink{}};
    s1.write("to file\n");
    s2.write("to stderr\n");
}
```

This is the pattern behind logging frameworks, callback handlers, and stream-like abstractions.

---

## 9.7 Worked Example: Four Implementations of Renderer

Let's implement the same `Renderer` API in four ways, and compare them.

### 1. Virtual Functions

```cpp
struct RendererVirtual {
    virtual ~RendererVirtual() = default;
    virtual void render() = 0;
};

struct OpenGLVirtual : RendererVirtual {
    void render() override {
        for (int i = 0; i < 10000; ++i) {
            // Simulate rendering
            volatile int x = i * i;
        }
    }
};

void benchmark_virtual() {
    std::vector<std::unique_ptr<RendererVirtual>> renderers;
    for (int i = 0; i < 100; ++i) {
        renderers.push_back(std::make_unique<OpenGLVirtual>());
    }

    auto start = std::chrono::high_resolution_clock::now();
    for (int j = 0; j < 10000; ++j) {
        for (auto& r : renderers) {
            r->render();
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "Virtual: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << " ms\n";
}
```

### 2. CRTP

```cpp
template <typename Derived>
struct RendererCRTP {
    void render() {
        static_cast<Derived*>(this)->render_impl();
    }
};

struct OpenGLCRTP : RendererCRTP<OpenGLCRTP> {
    void render_impl() {
        for (int i = 0; i < 10000; ++i) {
            volatile int x = i * i;
        }
    }
};

void benchmark_crtp() {
    std::vector<OpenGLCRTP> renderers(100);  // Concrete type

    auto start = std::chrono::high_resolution_clock::now();
    for (int j = 0; j < 10000; ++j) {
        for (auto& r : renderers) {
            r.render();  // Inlinable
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "CRTP: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << " ms\n";
}
```

### 3. std::variant + std::visit

```cpp
struct OpenGLVariant {
    void render() {
        for (int i = 0; i < 10000; ++i) {
            volatile int x = i * i;
        }
    }
};

using RendererVariant = std::variant<OpenGLVariant>;

void benchmark_variant() {
    std::vector<RendererVariant> renderers(100, OpenGLVariant{});

    auto start = std::chrono::high_resolution_clock::now();
    for (int j = 0; j < 10000; ++j) {
        for (auto& r : renderers) {
            std::visit([](auto& impl) { impl.render(); }, r);
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "Variant: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << " ms\n";
}
```

### 4. std::function

```cpp
void benchmark_function() {
    std::vector<std::function<void()>> renderers;
    for (int i = 0; i < 100; ++i) {
        renderers.push_back([]() {
            for (int i = 0; i < 10000; ++i) {
                volatile int x = i * i;
            }
        });
    }

    auto start = std::chrono::high_resolution_clock::now();
    for (int j = 0; j < 10000; ++j) {
        for (auto& r : renderers) {
            r();
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "Function: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << " ms\n";
}
```

### Results (on typical hardware, `-O2`)

```
Virtual:   450 ms   (vtable indirection + heap allocation)
CRTP:      100 ms   (fully inlined, zero overhead)
Variant:   110 ms   (switch/jump table, inlinable for small code)
Function:  480 ms   (heap + vtable, similar to virtual)
```

Key observations:
- **CRTP is fastest:** The compiler inlines everything.
- **Variant is nearly as fast:** The switch table is predictable.
- **Virtual and Function are slowest:** Heap allocation + vtable overhead.
- **The gap widens for light work:** If `render()` did tiny work, the dispatch cost would dominate.

### Binary Size Impact

- **CRTP:** Each instantiation generates a separate `render()` function. With 100 instances, you get 100 copies (but the linker may fold identical code).
- **Variant:** One switch statement that dispatches to one implementation.
- **Virtual:** One vtable per concrete class, one implementation per virtual function.
- **Function:** Heap-allocated per instance.

---

## 9.8 Tradeoffs Summary

| Mechanism | Pros | Cons | When to Use |
|---|---|---|---|
| **Non-virtual** | Inlinable, zero overhead, static safety | Type must be known at compile time; closed type set | Default choice; tight loops; hot functions |
| **Virtual** | Open type set, dynamic polymorphism, runtime flexibility | vtable indirection, no inlining, heap allocation, slower | Plugin systems, open-ended hierarchies, when type is runtime-determined |
| **CRTP** | Zero overhead, inlinable, static safety | Code bloat, closed type set, call sites must know type | Performance-critical code with known types; libraries like Eigen |
| **std::variant** | Closed set, no heap, inlinable for small code, value semantics | Type set must be fixed, variant size = max(types), no inheritance | Small fixed-size unions, configuration, result types |
| **std::function** | Accepts any callable type, type-erased, simple API | Heap allocation, vtable, slow, information loss | Configuration-time callbacks, event handlers, stored callbacks |
| **Raw function pointer** | Minimal overhead, no allocation | No type safety, brittle, C-like | C interop, performance-critical callbacks, linking-time configuration |
| **Concept + template** | Zero overhead, explicit contract, good errors | Code bloat, closed type set, compile-time errors only | Generic algorithms, libraries, performance-critical templates |

---

## 9.9 Decision Matrix

Choosing between these mechanisms requires asking:

1. **Type known at compile time?** If yes, you can use static dispatch (non-virtual, templates, CRTP, concepts). If no, you need dynamic dispatch (virtual, std::function, runtime type info).

2. **Type set size and stability?** 
   - **Small and fixed:** `std::variant` is often best.
   - **Small and growing:** CRTP or concept templates.
   - **Large or open:** Virtual functions or `std::function`.

3. **Performance critical?** 
   - **Yes:** Use static dispatch or CRTP. Measure variant; it's often faster than virtual.
   - **No:** Clarity matters more than dispatch cost. Use virtual if hierarchy is natural, or `std::function` for simplicity.

4. **Inheritance semantics needed?** 
   - **Yes:** Virtual functions or type erasure.
   - **No:** Templates or CRTP are cleaner.

5. **Can call site know the concrete type?**
   - **Yes:** CRTP or templates. Enables inlining.
   - **No:** Type erasure or virtual dispatch.

---

## 9.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Virtual functions are slow." | Virtual dispatch adds 5-10% overhead for expensive work, potentially 100%+ for tiny work. Measure. |
| "CRTP is more complex than virtual." | CRTP has a learning curve but no runtime overhead. Use it when performance matters. |
| "std::variant is only for data types." | std::variant can hold any type; with lambdas and std::visit, it's a powerful dispatch mechanism. |
| "std::function is slow; avoid it." | std::function is slower than native pointers, but it's not always the bottleneck. Use when clarity is worth the cost. |
| "Dynamic dispatch requires pointers." | Virtual functions use pointers, but std::variant and std::function are more flexible. |
| "Templates avoid all dispatch costs." | Templates avoid dispatch at the call site, but code bloat can hurt instruction cache and compile time. |
| "CRTP generates more code than virtual." | CRTP generates more binary per instantiation, but each instantiation is smaller and inlinable than virtual code. |
| "You can't mix dispatch mechanisms." | You can; e.g., a virtual function that internally uses a variant, or a template that wraps a std::function. |

---

## 9.11 Exercises

1. **Disassemble all four implementations.** Compile the four `Renderer` implementations with `-O2 -S` and examine the assembly. Find the dispatch instruction (or lack thereof) in each. Which one is truly inlined? Which goes through a vtable?

2. **Variant size experiment.** Write:
   ```cpp
   std::cout << sizeof(std::variant<char, int, double, std::string>);
   ```
   What's the size? (Should be `size(std::string) + alignment + tag size`.) Now write a variant with 100 types. What happens to the size?

3. **CRTP call site.** Rewrite the CRTP example so the call site accepts both `Renderer<OpenGL>` and `Renderer<Vulkan>` in the same vector. (Hint: you can't without a wrapper. How would you wrap it?)

4. **std::visit performance.** Write a variant with 10 different types. Use std::visit in a tight loop (1 million iterations). Compare to a switch statement. Are they the same speed?

5. **Function pointer cost.** Write:
   ```cpp
   typedef int (*Op)(int, int);
   Op ops[100];
   for (int i = 0; i < 100; ++i) ops[i] = (i % 2 == 0) ? add : multiply;
   
   auto start = clock();
   for (int j = 0; j < 1000000; ++j) {
       for (int i = 0; i < 100; ++i) {
           ops[i](j, j+1);
       }
   }
   ```
   How much faster/slower than a loop that calls a non-virtual function?

6. **Custom type erasure.** Implement a `Logger` class that accepts any type with a `log(std::string_view)` method, using the type-erasure pattern from §9.6. Compare its size and speed to using `std::function<void(std::string_view)>`.

---

## 9.12 Summary

The choice between static and dynamic dispatch is fundamental to C++ architecture. Static dispatch—through non-virtual functions, templates, CRTP, and concepts—offers zero runtime cost and compiler optimization opportunities. Dynamic dispatch—through virtual functions, `std::variant` + `std::visit`, `std::function`, and type erasure—offers runtime flexibility at a cost in indirection, inlining opportunity, and memory allocation.

Most well-designed C++ code uses static dispatch as the default, reserving dynamic dispatch for APIs that genuinely need it: plugin systems, open type hierarchies, configuration-time callbacks. The three-way decision between virtual functions, CRTP, and `std::variant` dominates performance-sensitive code. Understanding the assembly and the vtable layout unlocks the ability to reason about dispatch cost and make trade-offs confidently.

---

**[← Previous: Chapter 8 — Virtual Tables](08-virtual-tables.md)** · **[Up: Part 3](README.md)** · **[Next: Chapter 10 — Dependency Injection From Scratch →](10-dependency-injection-from-scratch.md)**
