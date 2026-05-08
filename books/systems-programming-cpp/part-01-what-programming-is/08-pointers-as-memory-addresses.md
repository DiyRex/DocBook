# Chapter 8 — Pointers As Memory Addresses

## Learning Objectives

By the end of this chapter you will be able to:

1. Define a pointer precisely: a **typed view of a memory address**, and explain why both halves — the address and the type — matter.
2. Distinguish a pointer's **value**, a pointer's **type**, and the **object it refers to** — three things that beginners conflate constantly.
3. Reason about pointer arithmetic correctly: why `p + 1` adds `sizeof(*p)` bytes, what happens when you cross object boundaries, and why this is undefined behavior territory.
4. Explain the difference between a **pointer**, a **reference**, an **iterator**, an **array name**, and a **smart pointer** — and why these distinctions matter for ownership, safety, and code clarity.
5. Recognize the four classic pointer hazards (null, dangling, uninitialized, type-punned) and how each manifests at runtime.
6. Connect pointers to the higher-level concept of **identity vs value** that underlies everything from `equals()` overrides to ORM design.

This chapter is where many developers' relationship with C and C++ breaks down. We will build a model that survives — one that holds up not only in C++ but in *every* language that has references, including the "no pointers" languages like Java and Python.

---

## 8.1 The Definition That Actually Helps

A pointer is **a value that names a memory location, plus a type that says how to interpret what's there**.

```cpp
int  x = 42;
int* p = &x;
```

`p`'s value is "the address of `x`" — say, `0x7fff_abcd_1234`. `p`'s type is `int*`, which means: "the bytes at this address should be read/written as an `int`."

Three things, not one:

- **The pointer itself**: a 64-bit value (on a 64-bit system), 8 bytes, lives somewhere in memory or in a register.
- **The address it holds**: a number from `0` to `2^64 - 1`. This number is just a number.
- **The pointee**: the thing at that address. Maybe an `int`, maybe nothing, maybe garbage, maybe an `int` whose lifetime ended last microsecond.

These three are independent. The pointer is alive even if the pointee is dead. The address can be valid (mapped) but contain garbage. The type can claim "int" but the bytes can be from a different object that was placed there. Most pointer bugs are confusion among these three layers.

---

## 8.2 The Address Is a Number; the Type Is the Hat It's Wearing

The address `0x7fff_abcd_1234` is just a 64-bit integer. The CPU does not "know" what's at that address. Whether you treat the bytes there as an `int` or a `float` or a `char[4]` is determined by *the type of the pointer you read it through*.

```cpp
int   x = 0x40490FDB;  // bit pattern of float 3.14159...
float* fp = (float*)&x;
std::printf("%f\n", *fp);   // prints "3.14159..."
```

This is **type punning**. It's allowed in some forms (via `std::memcpy` or `std::bit_cast` in C++20, or unions in C) and undefined behavior in others (the cast above violates strict aliasing rules in C++).

The point: at the *machine* level, `int` and `float` and `char` are all just bytes. The *type system* — the compiler's bookkeeping — decides which interpretation is legal. Type-punning is the deliberate mismatch between the machine view and the type-system view, and it's a sharp tool.

---

## 8.3 Pointer Arithmetic

```cpp
int arr[5] = {10, 20, 30, 40, 50};
int* p = arr;          // p points to arr[0]
int* q = p + 2;        // q points to arr[2], NOT to (address of p) + 2 bytes
```

`p + 2` adds **`2 * sizeof(int) = 8` bytes** to the underlying address. The compiler scales the integer by the pointee's size. This is the whole point of types on pointers: they let you talk about *elements*, not *bytes*.

Two pointers can be subtracted to get the number of elements between them:

```cpp
ptrdiff_t d = q - p;   // d = 2
```

Pointer arithmetic is **only defined within a single array** (or one past its end). `&arr[0] + 5` is fine (and points one past the last element); `&arr[0] + 6` or `&otherArray[0] - &arr[0]` is undefined behavior, even if the addresses happen to be valid.

This rule comes from the language's contract that arrays are the only place pointer arithmetic is meaningful. The compiler may use this rule to optimize: it assumes you never go out of bounds, and may eliminate range checks accordingly. Cross the line and you've stepped off the contract; the compiler's assumptions can corrupt your program in surprising ways.

---

## 8.4 Pointers, References, Iterators, Arrays — What's the Difference?

C++ has many forms of "indirect access." They differ in important ways.

### Pointer (`T*`)
- **Mutable**: can be reassigned to point elsewhere; can be null.
- **Explicit**: written as `*p` or `p->x`.
- **No lifetime guarantees**: holds an address; the language does not check the pointee is still alive.
- The "low-level" tool. You use it when you must, otherwise prefer something higher-level.

### Reference (`T&`)
- **Bound at creation**, never reassigned.
- **Cannot be null** (in well-behaved code; you can construct an invalid reference via UB but you shouldn't).
- **Implicit**: written like `r = 5;` (no `*`).
- A reference is essentially a pointer with the discipline of "always points to a valid object" baked into the type system.

```cpp
int x = 1;
int& r = x;     // r refers to x, forever
r = 2;          // x is now 2
int& s = r;     // s refers to the same x
```

Under the hood, references are usually implemented as pointers. The semantic difference is in *what the language promises*.

### Iterator (`std::vector<T>::iterator`)
- Generalized pointer: can be pointed at sequence elements, can be advanced (`++it`), dereferenced (`*it`).
- For `vector` and `array`, iterators are typically just `T*` (or thin wrappers around it).
- For other containers (`map`, `list`), iterators are class types that overload `++`/`*`/`->` to do the right thing.
- Iterators **may be invalidated** by container modifications: a `vector::iterator` becomes dangling after the vector reallocates. Knowing the invalidation rules per container is core C++ knowledge.

### Array name (`T arr[N]`)
- An array name **decays** to a pointer to its first element in most contexts: `int arr[5]; int* p = arr;` works.
- But: `sizeof(arr)` is `5 * sizeof(int)`, while `sizeof(p)` is `sizeof(int*)`.
- Functions cannot take arrays by value; `void f(int arr[5])` is silently rewritten to `void f(int* arr)` and you've lost the size info.

### Smart pointer (`std::unique_ptr<T>`, `std::shared_ptr<T>`)
- An object that *owns* a pointee and frees it automatically.
- We will dedicate a chapter to smart pointers in Part 2; here, just know they exist as the modern, safe alternative to raw owning pointers.

The hierarchy in modern C++:

> Use **values** when you can. Use **references** when you need indirection without ownership. Use **`unique_ptr`/`shared_ptr`** when you need ownership. Use **raw pointers** only for non-owning, optional indirection (`T*` to mean "a pointer to an existing object somewhere, or null").

This rule, applied consistently, eliminates most C++ memory bugs. The language allows raw owning pointers for backward compatibility; modern code minimizes them.

---

## 8.5 Pointers in "No-Pointer" Languages

Java, Python, JavaScript, C#, Ruby — these languages don't have explicit pointer syntax. But they absolutely have pointers under the hood.

```java
Person p = new Person("Alice");
Person q = p;
q.name = "Bob";
System.out.println(p.name);   // prints "Bob"
```

`p` and `q` are not `Person` objects — they are *references*, which are just pointers in a different costume. Both refer to the same heap-allocated object.

In Python:
```python
xs = [1, 2, 3]
ys = xs
ys.append(4)
print(xs)   # [1, 2, 3, 4]
```

`xs` and `ys` are both references to the same list object.

The model is: in these languages, every variable that "holds an object" actually holds a pointer to a heap-allocated object. Primitive types (small ints, booleans) may be stored inline as a special case ("value types," "small int caching"). Everything else is by reference.

This is why these languages can have garbage collection: references are tracked, the GC can find them all, and collect what's unreachable. It's also why these languages have surprising aliasing — passing a list to a function gives the function a reference, and the function can mutate the original.

> The deepest pointer lesson: **almost every language has pointers. The differences are in syntax, in safety guarantees, and in lifetime management — not in the underlying mechanism.**

Once you see this, JavaScript's "passed by reference for objects, by value for primitives" stops being mysterious — it's just C++'s `T&` for objects and `T` for primitives. Java's `==` vs `.equals()` distinction is identity (pointer comparison) vs value comparison. Python's `is` vs `==` is the same distinction. Knowing pointers tells you what's happening in all of them.

---

## 8.6 The Four Classic Hazards

### Hazard 1: Null dereference

```cpp
int* p = nullptr;
*p = 5;      // segfault
```

The address `0x0` is never mapped (the kernel reserves a low region as unmapped specifically to catch this). The hardware faults; you get `SIGSEGV`.

Defenses:
- Don't store null in a "should always be valid" pointer; use a reference instead.
- For optional indirection, use `std::optional<T&>` or a wrapper that documents nullability.
- Check before dereferencing if the pointer is genuinely optional.
- Modern languages (Kotlin, Swift, Rust) make nullability part of the type system: `T?` is a different type from `T`, and the type checker forces you to handle null.

### Hazard 2: Dangling pointer

```cpp
int* p;
{
    int x = 5;
    p = &x;
}
*p = 10;     // x is gone; p is dangling
```

The pointee's lifetime ended; the memory is reused (or the stack frame collapsed); the pointer still has the old address. Reading or writing through it is UB.

This is the most pernicious memory bug because:
- It often *appears* to work in testing — the freed memory hasn't been overwritten yet.
- It corrupts state silently — the eventual crash, if any, is far from the cause.
- It's the basis of a huge fraction of security vulnerabilities (use-after-free).

Defenses:
- Don't store pointers across lifetime boundaries.
- Use `std::unique_ptr` / `std::shared_ptr` to tie pointer lifetimes to their pointees.
- Use Rust's borrow checker, which makes dangling references compile-time errors.
- AddressSanitizer (`-fsanitize=address`) catches most use-after-free at runtime.

### Hazard 3: Uninitialized pointer

```cpp
int* p;          // uninitialized; contains garbage
*p = 5;          // undefined; could segfault, could corrupt
```

In C++, local variables are not zero-initialized by default. A pointer declared without an initializer holds whatever bytes were in that stack slot last.

Defenses:
- Always initialize: `int* p = nullptr;` or `int* p = &x;`.
- Use `-Wall -Wuninitialized` (compiler diagnostics).
- Modern C++ style: declare variables at point of first use, with an initializer. The "declare at top of function, init later" idiom is a legacy C habit.

### Hazard 4: Type punning / misaligned access

```cpp
char buffer[8];
int* ip = (int*)buffer;
*ip = 42;     // may be unaligned; UB on strict-alignment archs
```

If `buffer` is not 4-byte aligned, the write is at best slow, at worst (on ARM without unaligned-access support enabled) a hardware fault. And the type-pun violates strict aliasing — the compiler is allowed to assume `int*` and `char*` don't overlap, leading to optimizations that produce nonsense.

Defenses:
- Use `std::memcpy(&value, buffer, sizeof(value))` for legal type punning.
- Use `std::bit_cast<T>(source)` in C++20 for cleaner syntax.
- Use `alignas(int)` to force alignment when needed.

---

## 8.7 Pointers in Memory: The Picture

A pointer is 8 bytes (on 64-bit) somewhere in memory. The 8 bytes encode an address. The address is interpreted relative to your process's virtual address space (Chapter 5).

```
   pointer p (lives in the stack frame of f)
   +-------------------+
   |  0x55a0c5e8b2b0   |     <- 8 bytes; the address of an int on the heap
   +-------------------+
            |
            v
   +---+
   | 7 |     <- the int *p
   +---+
   address 0x55a0c5e8b2b0 in the heap region
```

The pointer and the pointee are *two different memory locations*. The pointer can be on the stack while the pointee is on the heap. The pointer can be in `.data` while the pointee is in another process (via shared memory). The two are wired by the address; they have nothing else in common.

This is why a single object can have multiple pointers to it — there's nothing magical about pointers, just multiple slots holding the same address. It's also why `delete p` does not affect `q` if both pointed at the same thing — `delete` is about the pointee, not the pointer.

---

## 8.8 Pointer-to-Function

```cpp
int (*op)(int, int);   // pointer to a function taking two ints, returning int
op = &add;
int r = op(2, 3);       // calls add(2, 3) indirectly
```

A function pointer holds the *entry address* of a function in `.text`. Calling through it is a single indirect-call instruction (`call qword ptr [rax]`).

Function pointers are how:
- C does "polymorphism" (qsort takes a comparator function pointer).
- Plugin systems work (load a `.so`, look up a symbol, call through a pointer).
- Virtual functions in C++ are implemented (each polymorphic class has a vtable — an array of function pointers).
- Callbacks are passed in C-style APIs.

In modern C++, you usually use `std::function` (a type-erased wrapper that can hold function pointers, lambdas, or member-function pointers) or templates. But the underlying mechanism is always the same: a pointer to executable code.

---

## 8.9 Pointer-to-Member

C++ has a peculiar feature: pointer-to-member.

```cpp
struct Foo { int x; void greet(); };
int Foo::* px = &Foo::x;            // points to a member, not bound to an instance
Foo f; f.*px = 5;                    // dereference relative to f

void (Foo::* pg)() = &Foo::greet;
(f.*pg)();                           // call f.greet()
```

A pointer-to-member is *not* an address — it's an **offset** (for data members) or a **virtual-table index** (for virtual member functions). It's bound to an instance only when applied with `.*` or `->*`.

This is rare in modern code but worth recognizing: it's how serializers and reflection libraries enumerate fields generically, and how PIMPL-style compile-firewall designs sometimes work.

---

## 8.10 Smart Pointers Briefly

We're saving the deep dive for Part 2, but the model in one paragraph each:

- **`std::unique_ptr<T>`**: owns a `T`. Cannot be copied; can be moved. When the unique_ptr dies, it deletes the `T`. Zero overhead at runtime — same size as a raw pointer.
- **`std::shared_ptr<T>`**: owns a `T` jointly with other shared_ptrs. Maintains an atomic reference count; when the last one dies, the `T` is deleted. Larger (typically 16 bytes) and slower (atomic ops) than unique_ptr. Use only when ownership is genuinely shared.
- **`std::weak_ptr<T>`**: a non-owning observer of a shared_ptr's pointee. Can ask "is the pointee still alive?" and, if so, briefly upgrade to a shared_ptr. Essential for breaking cycles in shared_ptr graphs.

The *philosophy* of smart pointers — and of modern C++ — is: **make the type encode the ownership story**. A raw `T*` says nothing; a `unique_ptr<T>` says "I own this"; a `T&` says "I have access without ownership"; a `weak_ptr<T>` says "I might have access if it's still alive." The type system carries the documentation.

---

## 8.11 Identity vs Value

Pointers force you to confront a deep question: do you mean **the same object** or **an equal one**?

```cpp
int x = 5, y = 5;
int* p = &x;
int* q = &y;
p == q;       // false: different addresses
*p == *q;     // true: same value
```

This shows up everywhere:

- **Java**: `s1 == s2` compares references; `s1.equals(s2)` compares string contents. Forgetting this and using `==` for strings is the most common Java beginner bug.
- **Python**: `a is b` is identity (same object); `a == b` is value (`__eq__`).
- **Database design**: every row has a primary key (identity) — separate from its column values. Two rows can have identical column values but distinct identities.
- **ORMs**: a fetched object's *identity* is its DB primary key; its *value* is the snapshot of column data. Equality of ORM objects is a notoriously thorny design question.
- **Functional vs imperative**: pure functional programming says "values, not identities" — there's no concept of "the same list updated in place." Imperative programming relies on stable identities (mutate the user's profile in place).

When you see `==` overrides, `equals` methods, hash functions, and "deep copy vs shallow copy," you are dealing with identity vs value. Pointers are the place where this distinction is *forced* on you. Languages that hide pointers also hide the distinction — and bugs follow.

---

## 8.12 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| Raw pointer (`T*`) | No ownership info; nullable; manual lifetime | Smallest, fastest, most flexible; required for C interop |
| Reference (`T&`) | Cannot be reseated; cannot be null | Compile-time guarantee of validity; cleaner syntax |
| `unique_ptr<T>` | Move-only (must transfer to share) | Single owner; safe by construction; same speed as raw pointer |
| `shared_ptr<T>` | Larger, atomic ref counts cost; cycles leak | Multiple owners, safe lifetimes |
| GC reference (Java/Python) | Pause times; allocation pressure; per-object overhead | No manual lifetime management; no use-after-free |
| Borrow-checked (Rust) | Steeper learning curve; some patterns hard to express | Compile-time guarantee of no aliasing UB; no runtime overhead |
| Type-punning via `memcpy`/`bit_cast` | Verbose | Defined behavior; portable |
| Type-punning via reinterpret_cast | Brief | UB land; optimizer may surprise you |

---

## 8.13 Common Misconceptions

| Misconception | Reality |
|---|---|
| "A pointer is a number." | A pointer has a number *and* a type. Both matter to the language. |
| "Pointers are unsafe; references are safe." | References are pointers with one fewer footgun. They can still be used after lifetime ends if you write the code wrong. |
| "I don't use pointers; I write Python." | You use pointers; they're called "object references." They can be null, can be aliased, can hold stale references. |
| "Pointer arithmetic in C++ is well-defined like integer arithmetic." | Only within a single array. Outside that, UB, even if the addresses look valid. |
| "Pointer cast preserves the bytes." | The bit pattern is preserved. The compiler's assumptions about aliasing may not be. Strict-aliasing UB causes mysterious bugs at high optimization. |
| "`shared_ptr` solves all memory management." | It solves nothing for cycles, and it's overhead is real. Default to `unique_ptr`; reach for `shared_ptr` only when ownership is genuinely shared. |

---

## 8.14 Exercises

1. **Three things, one exercise.** Write code that demonstrates the difference between (a) the pointer's address (where the pointer lives), (b) the pointer's value (the address it holds), (c) the pointee. Print all three for a single `int* p = new int(42);`.

2. **Pointer arithmetic.** Predict the output:
   ```cpp
   int arr[5] = {10, 20, 30, 40, 50};
   int* p = arr;
   std::printf("%d %d %d\n", *p, *(p+2), *(arr+4));
   std::printf("%ld\n", (long)((char*)(p+1) - (char*)p));
   ```
   Run and confirm. Explain each output.

3. **Type punning, the right way and the wrong way.** Write three versions of "interpret a 4-byte buffer as a float":
   - With `reinterpret_cast<float*>`.
   - With `std::memcpy`.
   - With `std::bit_cast<float>` (C++20).
   Compile with `-O2 -fstrict-aliasing -Wall`. Do you get warnings? Run AddressSanitizer or UBSan; do they complain about any?

4. **Dangling-pointer demo.**
   ```cpp
   int* make_dangling() {
       int x = 42;
       return &x;
   }
   int main() {
       int* p = make_dangling();
       std::printf("%d\n", *p);   // UB
   }
   ```
   Compile with `-Wall`; the compiler will likely warn. Run it. Now run it under AddressSanitizer (`-fsanitize=address`). What does ASan report? Why is the warning so much more useful than the segfault?

5. **Reference vs pointer.** Write the same function (e.g., "increment by one") in three forms:
   ```cpp
   void inc_ptr(int* p);        // raw pointer
   void inc_ref(int& r);        // reference
   void inc_unique(std::unique_ptr<int>& u);
   ```
   Discuss: which one allows the caller to pass null? Which one most clearly says "I will not own this"? Which one is most idiomatic when calling code already has a `unique_ptr`?

6. **Identity vs value.**
   ```python
   a = [1, 2, 3]
   b = [1, 2, 3]
   c = a
   print(a == b, a is b)
   print(a == c, a is c)
   ```
   Predict the output. Run it. Connect to: in C++, what would `a == b` and `&a == &b` give for `std::vector`s?

7. **Conceptual.** Explain to a colleague: in Java, the line `String s = "hello";` involves a pointer. Where? What is `s`'s actual contents? What is the type? Why does Java not call this a "pointer"?

---

## 8.15 What's Next

You can now reason about pointers as typed views of addresses, distinguish them from references and smart pointers, see the four hazards before they bite, and recognize that "no pointer" languages just have hidden pointers everywhere.

Chapter 9 — **Why Abstractions Exist** — pulls the camera all the way back. We have spent eight chapters at the metal. Now we ask: given that all software is just CPU + memory + addresses, why do we have languages, libraries, frameworks, and design patterns at all? What problem do they actually solve?

Chapter 10 — **What Languages Actually Do** — closes Part 1 by viewing every programming language as a *negotiation* with the same underlying machine. After it, you'll have the tools to read any language's runtime documentation and quickly know where in this picture you are.

---


**[← Previous: Chapter 7 — Call Stack Deep Dive](07-call-stack-deep-dive.md)** · **[Up: Part 1](README.md)** · **Next: Chapter 9 — Why Abstractions Exist (coming soon)**
