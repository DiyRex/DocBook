# Chapter 25 — Why Classes Exist

When someone asks "what are classes for?", the textbook answer is "object-oriented programming." That answer is wrong and misleading. Classes are not about OOP. They are a **bundling mechanism** — a way to package data and the operations that maintain its invariants into one named entity that the compiler understands. Understanding this distinction is the foundation for writing C++ that is clean, efficient, and robust.

A class is the smallest unit in C++ that can own a resource correctly. It is the mechanism that makes RAII possible, that enables access control, and that ties semantics to types rather than trusting programmer discipline. This chapter deconstructs what classes actually *do*, when they are the right tool, and when they are not.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define a class precisely as a bundling of data and operations, with the compiler enforcing that operations are called correctly on the data.
2. Compare C-style struct-plus-functions with C++ classes and explain what the class adds (invariant enforcement, RAII, access control, special members).
3. Understand the memory layout of a class: data members take space, member functions do not (they are code in `.text`).
4. Recognize when classes are the right choice (resource ownership, invariants) and when they are not (data-only payloads, pure algorithms).
5. Distinguish aggregate types from encapsulated types and decide when to use `struct` versus `class`.
6. Build a simple RAII class from scratch and verify that it maintains its invariants.
7. Explain the six special member functions and why they matter to correctness.

---

## Before Classes: C-Style Structures

C has structs and functions. A struct is a bag of data; functions operate on that data. This pattern is simple and works, but has no compiler-enforced connection between the data and the operations.

Example: a stack in C.

```c
// C: struct definition
typedef struct {
    int* items;
    int size;
    int capacity;
} stack_t;

// C: free functions that operate on the struct
stack_t stack_create(int capacity) {
    stack_t s = {0};
    s.items = malloc(capacity * sizeof(int));
    s.capacity = capacity;
    s.size = 0;
    if (!s.items) { /* error handling */ }
    return s;
}

void stack_push(stack_t* s, int x) {
    if (s->size >= s->capacity) {
        // resize or error
    }
    s->items[s->size++] = x;
}

int stack_pop(stack_t* s) {
    if (s->size == 0) {
        // error
    }
    return s->items[--s->size];
}

void stack_destroy(stack_t* s) {
    free(s->items);
    s->items = NULL;
    s->size = 0;
    s->capacity = 0;
}
```

This works. But notice the problems:

1. **No compiler enforcement.** I can call `stack_push(s, 5)` or `stack_pop(s)`, but nothing prevents me from directly modifying `s->size` or `s->items` and corrupting the invariant.
2. **Easy to forget cleanup.** I can create a `stack_t`, use it, and forget to call `stack_destroy()`. The memory leaks. The compiler does not help.
3. **No RAII.** If I create a stack and then an exception (or error condition) occurs, I have to remember to clean up manually. The cleanup does not happen automatically.
4. **Naming burden.** Every function must carry the type name: `stack_create`, `stack_push`, `stack_pop`, `stack_destroy`. The connection is by convention, not enforced.
5. **Copy semantics unclear.** If I copy a `stack_t`, do I get a deep copy or a shallow copy? The struct has a pointer; copying the struct by default makes a shallow copy — two structs pointing to the same array. Disaster.

This pattern survives in production C code, but scaling it is error-prone. The burden of correctness is on the programmer, not the language.

---

## What a Class Adds

A C++ class fixes all of these problems in one move. Here is the same stack in C++:

```cpp
class Stack {
private:
    int* items_;
    int size_;
    int capacity_;
    
public:
    // Constructor: acquire resources
    explicit Stack(int capacity)
        : items_(new int[capacity]), size_(0), capacity_(capacity) {
        if (!items_) throw std::bad_alloc();
    }
    
    // Destructor: release resources (automatic on scope exit)
    ~Stack() {
        delete[] items_;
    }
    
    // Delete copy to prevent double-free
    Stack(const Stack&) = delete;
    Stack& operator=(const Stack&) = delete;
    
    // Allow move for efficient transfer
    Stack(Stack&& other) noexcept
        : items_(other.items_), size_(other.size_), capacity_(other.capacity_) {
        other.items_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }
    
    Stack& operator=(Stack&& other) noexcept {
        if (this != &other) {
            delete[] items_;
            items_ = other.items_;
            size_ = other.size_;
            capacity_ = other.capacity_;
            other.items_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        }
        return *this;
    }
    
    // Operations on the stack
    void push(int x) {
        if (size_ >= capacity_) {
            // resize or throw
        }
        items_[size_++] = x;
    }
    
    int pop() {
        if (size_ == 0) throw std::underflow_error("stack empty");
        return items_[--size_];
    }
    
    int size() const { return size_; }
    bool empty() const { return size_ == 0; }
};
```

What changed?

1. **Access control.** Data members are `private`. No code outside the class can directly touch `items_`, `size_`, or `capacity_`. All interaction must go through the public member functions.
2. **Constructor.** The `Stack(int capacity)` constructor is called automatically when a `Stack` object is created. It acquires the resource (allocates the array). If allocation fails, it throws. The object either exists with a valid state or does not exist at all.
3. **Destructor.** The `~Stack()` destructor runs automatically when the object leaves scope. It releases the resource. No manual `stack_destroy()` call needed. No forgetting, no leaks, no exception handling required — the destructor runs even during exception unwinding.
4. **Special members.** The copy constructor and copy assignment are explicitly deleted to prevent shallow copies. The move constructor and move assignment transfer ownership without copying. All of this is part of the type contract.
5. **Invariants enforced by the compiler.** The private data members and public interface mean: if you declare a `Stack`, you are guaranteed that the three members (`items_`, `size_`, `capacity_`) are consistent. You cannot corrupt one without the compiler knowing. And you cannot forget to call the destructor.

This is the **entire point** of a class: to tie the data and its operations together, so the compiler can help you keep them consistent.

---

## Memory Layout of a Class

A frequent misconception: classes are "fatter" than structs because they are more complex. False. Let us check.

```cpp
class Stack {
private:
    int* items_;     // pointer: 8 bytes (64-bit)
    int size_;       // int: 4 bytes
    int capacity_;   // int: 4 bytes
};

struct Point {
    int x;           // int: 4 bytes
    int y;           // int: 4 bytes
    int z;           // int: 4 bytes
};

int main() {
    std::cout << sizeof(Stack) << "\n";      // 16 bytes
    std::cout << sizeof(Point) << "\n";      // 12 bytes
    std::cout << std::alignment_of<Stack> << "\n"; // likely 8
}
```

The `Stack` is 16 bytes: 8 for the pointer, 4 for `size_`, 4 for `capacity_`, plus padding to align the structure. The padding is not wasted space — it is alignment required by the CPU. The processor loads and stores data most efficiently when it is aligned to its word size. An 8-byte pointer must start at an 8-byte-aligned address. The compiler inserts padding between members to satisfy alignment constraints.

Member functions do not contribute to the size. They are compiled as ordinary functions, stored in the `.text` segment of the binary, and called like any other function. All instances of the `Stack` class share the same member function code. The `this` pointer is passed implicitly as the first argument.

```cpp
// Compiler translates:
Stack s(10);
s.push(5);

// Into something like (conceptually):
Stack s(10);
Stack::push(&s, 5);  // 'this' passed implicitly as first parameter
```

A member function call is just a function call with an implicit pointer argument. No overhead beyond the function call itself. Member functions are not special in memory; they are code, shared across all instances of the class. If you declare a million `Stack` objects, they all share the same `push()`, `pop()`, and other member function code in the binary. Only the data members (the `items_` pointer, `size_`, `capacity_`) differ per object.

This is a critical insight: **the complexity of your class interface (how many methods it has) does not affect the size of each instance. Only the data members do.** A class with 50 methods and a class with 1 method, if they have the same data members, have the same per-instance size.

---

## When Classes Are the Wrong Tool

Classes are powerful, but not always the right choice. Three common cases where they are not:

### Case 1: Data-Only Payloads

If your type is purely data with no invariants — like a point in 3D space, or a coordinate pair — a plain `struct` is clearer:

```cpp
struct Point {
    float x, y, z;
};

Point p = {1.0f, 2.0f, 3.0f};
```

No invariant to enforce. No special behavior needed. A `struct` with public members is honest and lightweight. Using a `class` with private data, getters, and setters is boilerplate that adds nothing.

### Case 2: Pure Algorithms

If you are writing a function that takes inputs and produces outputs without maintaining state, use a free function, not a class:

```cpp
// Good: free function
std::vector<int> sort_descending(const std::vector<int>& v) {
    auto result = v;
    std::sort(result.rbegin(), result.rend());
    return result;
}

// Unnecessary: wrapping in a class
class DescendingSorter {
public:
    std::vector<int> sort(const std::vector<int>& v) {
        auto result = v;
        std::sort(result.rbegin(), result.rend());
        return result;
    }
};
```

The class adds a layer of indirection without benefit. C++ encourages free functions — they are first-class citizens, not "features" only available inside a class.

### Case 3: External Resources That Are Not Your Problem

If you are using a library-provided RAII wrapper, do not wrap it again:

```cpp
// Bad: unnecessary wrapping
class MyConnection {
private:
    std::unique_ptr<DatabaseConnection> conn_;
public:
    MyConnection(const std::string& url) : conn_(std::make_unique<DatabaseConnection>(url)) {}
    void execute(const std::string& query) { conn_->execute(query); }
};

// Good: just use the wrapper
auto conn = std::make_unique<DatabaseConnection>(url);
conn->execute(query);
```

If the library has already wrapped the resource in a class, respect that abstraction. Do not add more layers.

---

## Aggregate vs Encapsulated Classes

The C++ standard distinguishes two kinds of classes: **aggregate types** and **encapsulated types**. This is not just a naming difference; it changes what the compiler can do for you.

### Aggregate Types

An aggregate type is a class or struct with:
- No private or protected members
- No user-declared constructors
- No virtual functions or virtual base classes
- Members (if any) are all public

Examples:

```cpp
struct Point {
    float x, y;  // public, no constructor
};

struct Rectangle {
    Point top_left, bottom_right;  // public, no constructor
};
```

Aggregates can be **aggregate-initialized**:

```cpp
Point p = {1.0f, 2.0f};
Rectangle r = {{0, 0}, {10, 10}};
```

The compiler generates a default constructor (that default-initializes members), and does not require you to write one. Aggregates are simple, lightweight, and transparent.

### Encapsulated Types

An encapsulated type has private data and public operations. Examples:

```cpp
class Stack {
private:
    int* items_;
    int size_;
    int capacity_;
public:
    explicit Stack(int capacity);
    ~Stack();
    void push(int x);
    int pop();
};

class Logger {
private:
    std::ofstream file_;
    std::mutex mu_;
public:
    explicit Logger(const std::string& filename);
    void log(const std::string& message);
};
```

Encapsulated types hide their internals. They maintain invariants. You cannot aggregate-initialize them. You must call the constructor explicitly. This is the cost of safety.

### The Choice: `struct` vs `class`

In C++, the only difference between `struct` and `class` is default access level: `struct` defaults to public, `class` defaults to private. So technically:

```cpp
struct Stack { };           // members are public by default
class Stack { };            // members are private by default
```

But the names carry convention:

- Use `struct` for aggregate types (pure data, no invariants).
- Use `class` for encapsulated types (private data, public operations, invariants).

This is taste, not correctness. But following the convention makes your code readable to other programmers. They will see `struct Point` and expect a simple data container. They will see `class Logger` and expect to understand the public API, not the internals.

---

## Constructors, Destructors, and the Special Member Functions

When you declare a class, the compiler generates six special member functions automatically if you do not declare them. These are not optional niceties; they are fundamental to how C++ manages object lifetimes.

1. **Default constructor:** `Stack();` Constructs an object with default-initialized members. If not declared, the compiler generates one that default-initializes each member (primitive types get indeterminate values, class members get their own default constructors).
2. **Destructor:** `~Stack();` Destroys the object and cleans up resources. If not declared and the class has members that need destruction, the compiler generates one that calls their destructors.
3. **Copy constructor:** `Stack(const Stack&);` Constructs a copy by copying each member.
4. **Copy assignment operator:** `Stack& operator=(const Stack&);` Assigns one object to another by copying each member.
5. **Move constructor:** `Stack(Stack&&);` Constructs by moving (stealing resources from the source).
6. **Move assignment operator:** `Stack& operator=(Stack&&);` Assigns by moving.

As Chapter 15 explained (RAII), the rules are:

- **Rule of Zero:** If your class manages no resources (all members are standard types or smart pointers), let the compiler generate all six. The generated code will be correct because standard types know how to copy and move themselves.
- **Rule of Three:** If you write a destructor (because you manage a resource), you *must* write the copy constructor and copy assignment. Why? If you manage a raw pointer and the compiler generates a copy constructor, it will do memberwise copy — both objects will point to the same memory. When the first is destroyed, it deletes the pointer. The second is left with a dangling pointer. Use-after-free. This is why the rule exists: if you care enough about cleanup to write a destructor, care enough to define how copying works.
- **Rule of Five:** If you write the rule of three, also write move constructor and move assignment. Move operations are optimizations that transfer ownership without copying. In modern C++, they are expected.
- **Modern practice:** Use smart pointers (`std::unique_ptr`, `std::shared_ptr`) and let the compiler generate everything. Smart pointers know how to copy and move themselves correctly.

For our `Stack` class with a raw pointer:

```cpp
class Stack {
private:
    int* items_;  // pointer: needs custom destruction
    int size_;
    int capacity_;
public:
    Stack(int capacity);           // Explicit constructor
    ~Stack();                       // Explicit destructor (rule of three / five)
    Stack(const Stack&) = delete;  // Forbid copy: would cause double-delete
    Stack& operator=(const Stack&) = delete;
    Stack(Stack&& other) noexcept; // Move constructor: transfer ownership
    Stack& operator=(Stack&& other) noexcept; // Move assignment
};
```

Because `items_` is a raw pointer, we cannot let the compiler generate copy operations (it would create a shallow copy and cause double-delete). So we delete them, making `Stack` non-copyable. We do write move operations to allow efficient transfer of the allocation without copying.

If we had used `std::unique_ptr<int[]>`, we could use the Rule of Zero:

```cpp
class Stack {
private:
    std::unique_ptr<int[]> items_;  // smart pointer handles destruction
    int size_;
    int capacity_;
public:
    explicit Stack(int capacity);  // Explicit constructor
    // All six special members are compiler-generated and correct:
    // - destructor calls ~items_, which deletes the array
    // - copy is deleted automatically (unique_ptr forbids copy)
    // - move is generated and transfers the unique_ptr ownership
};
```

This modern approach is preferred. You gain RAII safety without writing special members.

---

## Worked Example: From C-Style Stack to C++ Class

Let us convert the C stack from earlier into a robust C++ class and observe the benefits.

**Step 1: Wrap the data and constructor in a class.**

```cpp
class Stack {
private:
    int* items_;
    int size_;
    int capacity_;
public:
    explicit Stack(int capacity) {
        items_ = new int[capacity];
        if (!items_) throw std::bad_alloc();
        size_ = 0;
        capacity_ = capacity;
    }
    
    void push(int x) {
        if (size_ >= capacity_) {
            throw std::overflow_error("stack full");
        }
        items_[size_++] = x;
    }
    
    int pop() {
        if (size_ == 0) {
            throw std::underflow_error("stack empty");
        }
        return items_[--size_];
    }
};
```

**Step 2: Add the destructor.**

```cpp
~Stack() {
    delete[] items_;
}
```

Now memory is freed automatically on scope exit.

**Step 3: Handle copy and move.**

```cpp
// Forbid copy (avoid double-free)
Stack(const Stack&) = delete;
Stack& operator=(const Stack&) = delete;

// Allow move (transfer ownership)
Stack(Stack&& other) noexcept
    : items_(other.items_), size_(other.size_), capacity_(other.capacity_) {
    other.items_ = nullptr;
    other.size_ = 0;
    other.capacity_ = 0;
}

Stack& operator=(Stack&& other) noexcept {
    if (this != &other) {
        delete[] items_;
        items_ = other.items_;
        size_ = other.size_;
        capacity_ = other.capacity_;
        other.items_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }
    return *this;
}
```

**Now observe what we gained:**

- **No way to leak.** The destructor runs when the object leaves scope, even if an exception is thrown.
- **No way to corrupt.** The private data members and public interface make it impossible to accidentally break the invariant from outside.
- **No way to accidentally copy wrong.** Copy is deleted; if you try, the compiler rejects it. Move is explicit; if you want to transfer ownership, you use move semantics.
- **Exception-safe by default.** If `push()` throws, the stack's state is unchanged and the destructor still runs. The object is always valid.

Compare to the C version: C has none of these guarantees. You have to remember to call destroy, manually prevent shallow copies, and handle exceptions yourself.

---

## Tradeoffs

| Choice | Pros | Cons | When to use |
|---|---|---|---|
| **Encapsulated class** (private data, public methods) | Invariants enforced; resource ownership clear; RAII works; access control; copy/move semantics explicit | More boilerplate; requires understanding special members; slightly heavier API surface | Resources (file, lock, connection); state that must be kept consistent |
| **Aggregate struct** (public data) | Simple; transparent; aggregate initialization; no boilerplate | No invariants; hard to evolve (adding validation breaks existing code); no special member control | Pure data: points, colors, coordinates, POD structures |
| **Free functions + struct** (C-style) | Flexible; minimal coupling; explicit at call site | No compiler enforcement; easy to forget cleanup; copy semantics unclear; burden on programmer | Rare in modern C++; only when interop with C is required |
| **Template-based generic class** (std::vector, std::array) | Works with any type; compile-time polymorphism; optimal codegen | Complex implementation; compiler error messages can be cryptic; template specialization needed for performance | Container types; generic algorithms |
| **Inheritance-based class hierarchy** (virtual functions) | Polymorphism; loose coupling to derived types; extensible; familiar to OOP practitioners | Runtime dispatch cost; vtable memory; virtual functions leak abstraction; harder to optimize | Framework plugin systems; heterogeneous collections (rarely the best choice for most problems) |

---

## Common Misconceptions

**Misconception 1: "Classes are for object-oriented programming."**

Partially true, but deeply misleading. OOP is a *programming style* — inheritance, polymorphism, message passing, virtual dispatch. Classes are a *bundling mechanism*. You can use classes without OOP (many successful C++ programs do: containers, smart pointers, RAII wrappers are not "OOP" in any meaningful sense). A `std::vector<T>` is a class, but it is not used polymorphically. `std::unique_ptr<T>` is a class template managing a resource, not pretending to be something else. And conversely, you can write code that mimics "OOP" with free functions and structs (though not well). The real benefit of classes is **tying data to operations and letting the compiler enforce their consistency**, independent of inheritance or virtual dispatch.

**Misconception 2: "Classes are always better than structs because they hide details."**

False. Hiding details is only valuable if there are invariants to protect. A `struct Point { float x, y; }` does not hide anything because there is nothing to hide. Invariants? If `x` and `y` can be any floating-point value, there is no invariant. Requiring getters and setters adds verbosity without benefit, and makes evolution harder (changing internal representation breaks the API). A `class Logger` with private state makes sense because the logger has invariants: if the file is open, certain preconditions hold; writes must be synchronized. If you find yourself writing a `class` with public getters and setters for every field, you have confused encapsulation with access control. Just make it a `struct`.

**Misconception 3: "You have to write all six special member functions."**

False. Modern C++ practice is the Rule of Zero: use smart pointers (`std::unique_ptr`, `std::shared_ptr`) and standard containers, and let the compiler generate everything. You only write special members when you have a unique resource management need that standard wrappers do not cover. And even then, consider: can you wrap your resource in a `unique_ptr` or a standard library class? If yes, you can still use Rule of Zero.

**Misconception 4: "Classes are slower than structs."**

False. A class and a struct with the same members have the identical memory layout and access cost. Member functions are ordinary functions; the compiler generates the same code as free functions. Modern compilers inline member functions as aggressively as free functions (sometimes more, because they can see the definition). There is zero inherent speed difference. If you observe a difference, it is because the code structure differs (different algorithms, different call patterns), not because one is a class and the other is a struct.

**Misconception 5: "If I make data private, I must provide getters and setters."**

False. Getters and setters are overused and often a sign of confused design. Private data should be accessed only through methods that make logical sense to the problem domain (e.g., `stack.push()`, `logger.log()`). If you find yourself writing `int getSize()` and `void setSize(int)`, the class is probably the wrong abstraction. The class should not expose mutable state directly; it should expose operations. If the only operations are "get the field" and "set the field," you do not have an abstraction — you have a thin wrapper. Just make it a struct or reconsider your design.

**Misconception 6: "Classes are a C++ invention to support OOP and do not apply in other languages."**

False. Encapsulation and bundling are universal programming concepts. Rust has `struct` and `impl` blocks for bundling data and methods. Go has `struct` with receivers (methods). Python has `class`. Java has `class`. C# has `class`. The idea is older than any language: group related data and operations together so the compiler (or runtime) can help you keep them consistent. C++ makes it the default and makes it work especially well with resource ownership (RAII), but the principle transcends C++.

---

## Exercises

1. **Convert C to C++.** Take the C `stack_t` example from this chapter and convert it to a C++ class. (a) Add a constructor and destructor. (b) Delete copy operations. (c) Add move constructor and move assignment. (d) Write a function that creates a stack, pushes some values, and throws an exception mid-way. Verify (using a memory profiler or custom allocation tracking) that the destructor runs and the memory is freed even when the exception is thrown.

2. **Aggregate vs encapsulated.** Write two versions of a `Color` type: (a) as a `struct Color { uint8_t r, g, b; };` (b) as a `class Color` with private members and `setRGB()`, `getRed()` methods. For each, write a small program that uses it. Which is more readable? When would you choose each?

3. **Memory layout.** Define a class with an `int`, a `double`, a pointer, and a `char`. Use `sizeof()` and `offsetof()` to determine the actual memory layout. Then change the declaration order of members and re-measure. Explain why the layout changes. What does this tell you about data alignment?

4. **Special members.** Write a class that manages a `std::string` resource. (a) Write the default constructor, destructor, copy constructor, copy assignment, move constructor, move assignment. (b) Now rewrite it using `std::unique_ptr<std::string>` and apply the Rule of Zero. Compare the code size and complexity.

5. **RAII for a custom resource.** Design a class `Lock` that acquires a mutex in the constructor and releases it in the destructor. Write a function that creates multiple `Lock` objects in nested scopes and throws an exception at various points. Verify (by logging or counting lock/unlock calls) that every lock is released, in the correct order, even when exceptions occur.

6. **Conceptual: invariants.** A coworker writes a class `BankAccount` with public members `balance` and `account_id`. They ask: "Why should I make these private and add getters/setters?" Write a detailed answer that: (a) explains what invariants a bank account might have, (b) shows how public members break those invariants, (c) shows how encapsulation protects them, (d) discusses the cost-benefit trade-off.

---

## Summary

A class is a bundling of data and operations with compiler-enforced consistency. It is the mechanism that makes RAII possible — tying resource acquisition to construction and release to destruction, with automatic cleanup even during exceptions. Classes are not primarily an OOP feature; they are a tool for ensuring that data and its operations stay in sync. Use them when you need invariants and resource ownership. Use structs for pure data. Use free functions for algorithms. The choice matters, but the principle is the same: match the language construct to the problem's structure.

---

**[← Previous: Chapter 24 — Encapsulation From First Principles](02-encapsulation.md)** · **[↑ Part 3](README.md)** · **[Next: Chapter 26 — What Objects Are In Memory →](04-what-objects-are-in-memory.md)**
