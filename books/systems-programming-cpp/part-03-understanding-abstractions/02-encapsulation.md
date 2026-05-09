# Chapter 24 — Encapsulation From First Principles

Encapsulation is not "make fields private." That is a mechanical consequence, not the point. The actual idea is: **make the invariants of an object unbreakable from outside it**. An invariant is a predicate that must be true between every public method call. If callers cannot see or modify the object's state directly, they cannot violate the invariant. The invariant is guaranteed by the type.

This chapter deconstructs encapsulation from first principles, shows why it matters for reasoning about code, and explores the tradeoffs between different ways to enforce it. By the end, you will understand not just *how* encapsulation works in C++, but *why* the specific mechanisms (private, getters, tell-don't-ask, opaque pointers) exist and when to use each one.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define encapsulation precisely: a contract that says "if you respect my public interface, I guarantee my invariants hold."
2. Identify the invariant of a class and verify that every public method preserves it.
3. Explain why private is not security or performance—it is documentation enforced by the compiler.
4. Recognize when getter/setter pairs are anti-encapsulation and how to redesign for better interfaces.
5. Apply the Tell-Don't-Ask principle: ask objects to act, not to expose state for external decision-making.
6. Use opaque pointers and the pimpl idiom to enforce encapsulation across translation unit boundaries.
7. Design a class whose invariants are provably unbreakable by any conforming caller.

---

## 24.1 What Is an Invariant?

An invariant is a predicate about an object's state that is *always true* between public method calls, never true between them, and impossible to break from outside.

Examples:

- A `Fraction` object whose denominator is never zero. The invariant: `denominator != 0`.
- A `BoundedQueue<T>` with a fixed capacity whose size never exceeds capacity. The invariant: `size <= capacity`.
- A `BankAccount` whose balance is always non-negative (if the bank forbids overdrafts). The invariant: `balance >= 0`.
- A `Mutex` in locked state. The invariant: at most one thread holds the lock.
- A `LinkedList<T>` whose nodes are all reachable by following `next` pointers from the head. The invariant: no cycles, no orphaned nodes.

The invariant is the contract between the class and its callers. The class promises: "If you use only my public methods and pass valid arguments, my invariant will always hold." Callers promise: "I will not reach into your private state and break things."

**Without encapsulation, this contract is unenforceable.** If all fields are public, a caller can write:

```cpp
Fraction f(1, 2);
f.denominator = 0;  // invariant broken; now f is nonsensical
int result = f.numerator / f.denominator;  // disaster
```

**With encapsulation, the contract is enforced by the compiler.**

```cpp
Fraction f(1, 2);
f.denominator = 0;  // compile error: denominator is private
```

The invariant is now a machine-checkable guarantee.

---

## 24.2 Local Reasoning: The Real Benefit

The deepest benefit of encapsulation is not safety from accidents—it is the ability to *reason about an object without reasoning about all its callers*.

Suppose you are debugging a `BankAccount` class. You notice the invariant `balance >= 0` is broken. Where could this have happened?

**Without encapsulation (all public fields):**

You must search *every line of code that uses BankAccount* to find the one that set `balance` to a negative value. There could be thousands of such lines. There could be thousands of places that do:

```cpp
account.balance -= amount;
```

You must check every one to see if `amount` could be larger than `balance`. And there could be code paths you did not see: code in other files, in third-party libraries, in code that was merged last week. Reasoning is impossible.

**With encapsulation (private balance):**

The only place `balance` can be modified is *inside the `BankAccount` class*. You open the header file, find the `withdraw` method (and a few others), and read them. Each one must check the invariant before modifying. The bug is in one of those methods. You have narrowed the search space from "all code that uses BankAccount" to "ten methods in BankAccount itself."

This is **local reasoning**: you can understand the invariant by reading the class, not by reading all its callers.

At scale, this is the difference between a debuggable system and chaos. In a 100-engineer company, thousands of call sites exist. Without encapsulation, a one-line bug in a class's invariant requires understanding thousands of call sites. With encapsulation, it requires reading the class.

---

## 24.3 Public, Private, Protected — Mechanically

C++ enforces encapsulation through three access specifiers. They are **not** about security or performance. They are **documentation enforced by the compiler**.

This is an important distinction. Many junior programmers think `private` means "the compiler prevents anyone from ever reading or modifying this," as if private fields are somehow hidden in a locked vault. They are not. At runtime, in memory, `private` and `public` fields are indistinguishable. `private` merely prevents the *textual reference* to the name from outside the class. It is a compile-time restriction, not a runtime one.

This serves a crucial purpose, though: it makes intent explicit and prevents *accidental* violations. A developer who wants to access a private field must do so deliberately (via a cast), which signals that they are doing something unusual. This is a social contract: "if you bypass this access specifier, you are breaking the class's encapsulation, and you own the consequences."

### `private`

A private member can only be named inside the class itself (and in friends, which we will skip).

```cpp
class BankAccount {
private:
    double balance_;  // can only be accessed here
    
public:
    void withdraw(double amount) {
        if (amount <= balance_) {
            balance_ -= amount;  // allowed; inside the class
        }
    }
};

int main() {
    BankAccount acc;
    acc.balance_ = -1000;  // compile error: balance_ is private
}
```

The compiler enforces this. The field is still in memory at the same address; the access specifier does not change the memory layout or runtime behavior. It only prevents *textual reference* to the name.

### `public`

A public member can be named anywhere.

```cpp
class Point {
public:
    double x, y;  // anyone can read or write
};

int main() {
    Point p;
    p.x = 3.0;  // allowed
    p.y = 4.0;  // allowed
}
```

No invariant protection; any code can make `x` or `y` invalid.

### `protected`

A protected member can be named inside the class and in derived classes. We will skip inheritance for now; for this chapter, treat `protected` as a tool for larger hierarchies where a base class wants to expose implementation details to derived classes.

### The Key Insight

**Access specifiers prevent you from naming a member from outside the class. They do not prevent understanding what is in memory.** A determined programmer could:

```cpp
BankAccount acc;
double* balance_ptr = (double*)&acc;  // cast and access
*balance_ptr = -1000;  // break the invariant
```

This is possible but requires explicit malice (a cast). The access specifier makes accidents hard and signals intent: "if you are casting away const and access specifiers to modify this, you are doing something unusual."

**Therefore: private is not about preventing skilled programmers from cheating. It is about making the invariant's contract explicit and preventing accidental violations.**

---

## 24.4 A Critical Distinction: Data Hiding vs. Invariant Protection

Before diving into getters and setters, we need to clarify a subtle but important point. There are two reasons to make a field private:

1. **Data hiding:** preventing callers from seeing the internal representation. Example: you represent a temperature in Kelvin internally but want callers to think in Celsius.

2. **Invariant protection:** preventing callers from putting the object into an invalid state. Example: a Fraction's denominator must never be zero.

**These are different goals, and they require different design patterns.**

If your goal is data hiding but the field has no invariant (e.g., it is computed from other fields or it can be any value), a getter is appropriate:

```cpp
class Rectangle {
private:
    double width_, height_;
    
public:
    double width() const { return width_; }  // getter; no invariant
    double height() const { return height_; }
    double area() const { return width_ * height_; }  // computed; no setter needed
};
```

If your goal is invariant protection, a getter/setter pair is insufficient. You need methods that enforce the invariant:

```cpp
class Fraction {
private:
    int num_, denom_;  // invariant: denom_ != 0
    
public:
    // No getter/setter! Instead, a method that enforces the invariant:
    void simplify();  // reduces to lowest terms
    
    // Operations that make sense on fractions:
    Fraction operator+(const Fraction& other) const;
    Fraction operator*(const Fraction& other) const;
    
    int numerator() const { return num_; }  // OK to expose; read-only
    int denominator() const { return denom_; }  // OK to expose; read-only
};
```

Notice: a const getter that returns the immutable denominator is fine. The problem is a *setter* that allows it to become zero.

---

## 24.5 Why Getters and Setters Are Often Anti-Encapsulation

A common pattern is to expose every field through a pair: a getter and a setter.

```cpp
class BankAccount {
private:
    double balance_;
    
public:
    double getBalance() const { return balance_; }
    void setBalance(double b) { balance_ = b; }
};
```

This looks like encapsulation—the field is private—but it is **not**. The getter exposes the entire state; the setter allows any code to modify it. This is equivalent to a public field with two extra function calls:

```cpp
BankAccount acc;
acc.setBalance(-1000);  // breaks invariant; setter allows it
double b = acc.getBalance();
```

A caller can now do anything to the account without the class's consent. The invariant is broken. The class has no more control than if `balance_` were public.

**Why is this bad?**

1. **It exposes implementation details.** If you later decide `balance_` should be represented differently (e.g., as an integer number of cents instead of a double, to avoid floating-point errors), you must change the getter/setter signature and recompile every caller. The class's implementation is leaked.

2. **It makes refactoring impossible.** Suppose you want to add a log entry every time the balance changes. With a setter, you can override it:

```cpp
void setBalance(double b) {
    if (b != balance_) {
        log("balance changed from " << balance_ << " to " << b);
        balance_ = b;
    }
}
```

But now the setter allows *any* value, including negative balances. You have not enforced the invariant.

3. **It passes the burden of consistency to the caller.** Instead of the class guaranteeing the invariant, every caller must remember to validate. This is the opposite of encapsulation.

**The better pattern: tell the object what to do, not what its state should be.**

```cpp
class BankAccount {
private:
    double balance_;
    
public:
    bool withdraw(double amount) {
        if (amount <= balance_) {
            balance_ -= amount;
            log("withdrew " << amount);
            return true;
        }
        log("withdrawal denied: insufficient funds");
        return false;
    }
    
    void deposit(double amount) {
        if (amount > 0) {
            balance_ += amount;
            log("deposited " << amount);
        }
    }
    
    bool canWithdraw(double amount) const {
        return amount <= balance_;
    }
};
```

Now the invariant is enforced inside the class. Callers cannot break it:

```cpp
BankAccount acc;
acc.withdraw(500);  // succeeds or fails; invariant preserved either way
acc.deposit(1000);  // invariant preserved
acc.canWithdraw(100);  // query without modifying
```

The class owns the invariant. Callers trust the class to maintain it. This is encapsulation.

**Why the BankAccount design is superior:**

- The invariant (non-negative balance) is enforced in one place: inside the class.
- Callers cannot even *express* the operation "set balance to -1000"; the method simply does not exist.
- If the business rule changes (e.g., "allow overdrafts up to $500"), you change one method. All callers automatically benefit.
- Logging, auditing, and side effects (like updating a transaction history) can be added in the method without changing callers.
- The public interface matches the problem domain ("withdraw", "deposit") rather than the implementation ("get", "set").

---

## 24.6 The Tell-Don't-Ask Principle

An antipattern in object-oriented programming is **Ask-Then-Do**: the caller asks the object for state, makes a decision, and tells the object to act.

```cpp
// BAD: Ask-Then-Do
if (account.getBalance() >= amount) {
    account.setBalance(account.getBalance() - amount);
} else {
    // handle error
}
```

This spreads the invariant check across two pieces of code: the caller's check and the setter. If either is wrong, the invariant breaks. And the caller must understand `BankAccount`'s semantics, duplicating knowledge.

**The alternative: Tell-Don't-Ask.** The caller tells the object what it wants to do and lets the object decide whether it can.

```cpp
// GOOD: Tell-Don't-Ask
if (account.withdraw(amount)) {
    // withdrawal succeeded
} else {
    // withdrawal failed; handle error
}
```

The `withdraw` method encapsulates the decision: "can I withdraw this amount?" and the action: "perform the withdrawal." The caller does not ask; it tells and gets back a result.

Benefits:

1. **The invariant is owned by the class.** Only `BankAccount::withdraw` decides if the invariant can be broken temporarily (during the operation).

2. **Decisions are localized.** The rule "you cannot overdraw" is enforced in one place: inside `withdraw`.

3. **Callers cannot accidentally trigger invalid state transitions.** A caller cannot withdraw from the account by modifying `balance_` directly.

4. **The public interface matches the business model.** Callers think in terms of "withdraw" and "deposit," not "read balance, compute, write balance."

Tell-Don't-Ask is not just a style preference; it is a consequence of proper encapsulation. If the class owns its invariants, callers must ask the class to maintain them.

**Practical guidance:**

- Ask-Then-Do: used when the decision is the caller's domain. Example: a UI that queries the model to update the display.
- Tell-Don't-Ask: used when the decision belongs to the object. Example: a domain object that enforces business rules.

The boundary is not always clear. But a good heuristic: if the decision involves the object's invariants, it should be inside the object (Tell-Don't-Ask). If the decision is about presentation or orchestration, it can be outside (Ask-Then-Do is acceptable).

---

## 24.7 Encapsulation Across Module Boundaries

Encapsulation inside a class is one thing. Encapsulation across translation units (header files, libraries) is another. At module boundaries, the contract must be enforced not just at compile time but also at link time and runtime.

The problem: when you expose a class's *layout* in a header file, clients compile based on that layout. If you change the layout (add a field, remove a field, reorder), every client must recompile.

```cpp
// account.h
class BankAccount {
private:
    double balance_;     // layout exposed in header
    int account_id_;
    std::string owner_;
};
```

A client compiles this header and learns the exact size and offset of every field. If you later add a new field:

```cpp
// account.h (modified)
class BankAccount {
private:
    double balance_;
    int account_id_;
    std::string owner_;
    std::vector<Transaction> history_;  // new field
};
```

The client's compiled code still expects the old layout. When it accesses `account_id_`, it reads from the wrong offset (because `history_` was inserted before it). Disaster.

The solution: **opaque pointers** (pimpl, "pointer to implementation").

```cpp
// account.h
class BankAccount {
private:
    class Impl;  // forward declaration; clients do not know what Impl is
    std::unique_ptr<Impl> pimpl_;
    
public:
    BankAccount();
    ~BankAccount();
    bool withdraw(double amount);
    void deposit(double amount);
    bool canWithdraw(double amount) const;
};

// account.cpp
class BankAccount::Impl {
public:
    double balance_;
    int account_id_;
    std::string owner_;
    std::vector<Transaction> history_;
};

BankAccount::BankAccount() : pimpl_(std::make_unique<Impl>()) {}
BankAccount::~BankAccount() = default;  // unique_ptr destructor runs automatically

bool BankAccount::withdraw(double amount) {
    if (amount <= pimpl_->balance_) {
        pimpl_->balance_ -= amount;
        pimpl_->history_.push_back(Transaction::withdraw(amount));
        return true;
    }
    return false;
}
```

Now:

1. The header exposes only a pointer (`pimpl_`), which has a fixed size (8 bytes on 64-bit systems).
2. The actual layout (`Impl`) is in the .cpp file, hidden from clients.
3. You can add fields, remove fields, or reorder fields in `Impl` without recompiling clients.
4. The invariant is enforced in the .cpp file, where `Impl` is defined.

This is encapsulation at the module level. The class's public interface (the declarations in the header) is separate from its implementation (the definition in the .cpp file).

**Trade-off:** an extra pointer indirection on every method call (dereferencing `pimpl_`). For a fast operation like `withdraw`, this overhead is negligible. For very tight loops where every cycle matters, it can show up in benchmarks. But the benefit—being able to change implementation without recompiling the world—usually outweighs the cost.

In modern C++, this idiom is often applied at the library/platform boundary: client-facing headers are minimal and stable, while implementation details live in .cpp files behind opaque pointers. This is why many large libraries (standard library implementations, Qt, Boost) use pimpl for classes shipped publicly.

---

## 24.8 Worked Example: BoundedQueue<T>

Let us design a template class `BoundedQueue<T>` that stores elements in a fixed-size circular buffer. The invariant is: **size never exceeds capacity, and the queue is always valid (no corruption).**

### The Specification

- Capacity is fixed at construction time.
- `push(x)` adds `x` to the back; fails (returns false) if full.
- `pop()` removes from the front; fails if empty.
- `size()` returns the number of elements.
- `empty()` and `full()` are predicates.
- Exception safety: strong guarantee (if an operation fails, the queue is unchanged).

### The Implementation

```cpp
#include <memory>
#include <stdexcept>

template <typename T>
class BoundedQueue {
private:
    std::unique_ptr<T[]> buffer_;
    size_t capacity_;
    size_t front_;   // index of first element
    size_t size_;    // number of elements
    
    // Invariant check (for debugging; removed in release builds)
    void check_invariant() const {
        assert(size_ <= capacity_);
        assert(front_ < capacity_);
        // if size_ == capacity_, buffer is full but valid
    }
    
public:
    explicit BoundedQueue(size_t capacity)
        : buffer_(std::make_unique<T[]>(capacity)),
          capacity_(capacity),
          front_(0),
          size_(0) {
        if (capacity == 0) {
            throw std::invalid_argument("capacity must be > 0");
        }
    }
    
    // Deleted copy (non-copyable for simplicity)
    BoundedQueue(const BoundedQueue&) = delete;
    BoundedQueue& operator=(const BoundedQueue&) = delete;
    
    // Move is allowed
    BoundedQueue(BoundedQueue&& other) noexcept
        : buffer_(std::move(other.buffer_)),
          capacity_(other.capacity_),
          front_(other.front_),
          size_(other.size_) {
        other.front_ = 0;
        other.size_ = 0;
    }
    
    BoundedQueue& operator=(BoundedQueue&& other) noexcept {
        if (this != &other) {
            buffer_ = std::move(other.buffer_);
            capacity_ = other.capacity_;
            front_ = other.front_;
            size_ = other.size_;
            other.front_ = 0;
            other.size_ = 0;
        }
        return *this;
    }
    
    ~BoundedQueue() = default;  // unique_ptr cleans up
    
    // Push: add to back
    bool push(const T& value) {
        check_invariant();
        
        if (size_ >= capacity_) {
            check_invariant();
            return false;  // full
        }
        
        size_t back_index = (front_ + size_) % capacity_;
        buffer_[back_index] = value;  // may throw (T's copy constructor)
        size_++;  // only increment after successful copy
        
        check_invariant();
        return true;
    }
    
    // Pop: remove from front
    bool pop(T& out) {
        check_invariant();
        
        if (size_ == 0) {
            check_invariant();
            return false;  // empty
        }
        
        out = buffer_[front_];
        front_ = (front_ + 1) % capacity_;
        size_--;
        
        check_invariant();
        return true;
    }
    
    // Pop without retrieving the value
    bool pop() {
        T dummy;
        return pop(dummy);
    }
    
    size_t size() const {
        check_invariant();
        return size_;
    }
    
    size_t capacity() const { return capacity_; }
    
    bool empty() const {
        check_invariant();
        return size_ == 0;
    }
    
    bool full() const {
        check_invariant();
        return size_ == capacity_;
    }
    
    // Peek at front without removing
    bool front(T& out) const {
        check_invariant();
        if (size_ == 0) return false;
        out = buffer_[front_];
        check_invariant();
        return true;
    }
};
```

### Verification of Invariants

Let us trace through `push` and `pop` to show that the invariant is preserved:

**Invariant: `size_ <= capacity_`**

- After construction: `size_ = 0`, `capacity_ > 0`. Invariant holds.
- After `push`: `size_` increments only if `size_ < capacity_`. Invariant holds.
- After `pop`: `size_` decrements. Invariant holds.
- Exception: if `T`'s copy constructor throws (in `push`), the `buffer_[back_index] = value` fails and `size_` is never incremented. The invariant is preserved (strong exception guarantee).

**Invariant: `front_ < capacity_`**

- After construction: `front_ = 0`. Invariant holds.
- After `pop`: `front_ = (front_ + 1) % capacity_`. Since `front_ < capacity_` and we add 1, we get at most `capacity_ - 1 + 1 = capacity_`, then modulo brings it back to `[0, capacity_)`. Invariant holds.

**Invariant: the queue is logically contiguous (no gaps)**

- We store elements in a circular buffer. `front_` is the index of the first element. The size is `size_`. Elements are at indices `front_, (front_+1)%cap, (front_+2)%cap, ..., (front_+size_-1)%cap`. No gaps; no corruption.

**Invariant: no corruption under exception**

The `push` method is carefully designed to preserve the invariant even if `T`'s copy constructor throws:

```cpp
bool push(const T& value) {
    if (size_ >= capacity_) return false;  // check before modifying
    
    size_t back_index = (front_ + size_) % capacity_;
    buffer_[back_index] = value;  // may throw; state not yet modified
    size_++;  // only increments on success
    
    return true;
}
```

The key: we compute `back_index` and attempt the copy before incrementing `size_`. If the copy throws, `size_` is unchanged; the invariant is intact. This is the strong exception guarantee: if an operation fails, the object is as if the operation never happened.

### Why External Code Cannot Break the Invariant

Every public method preserves the invariant. No external code can break it because `buffer_`, `capacity_`, `front_`, and `size_` are private. If a caller could write:

```cpp
queue.size_ = queue.capacity_ + 1;  // break invariant
```

the invariant would break. But this is impossible; `size_` is private. The only way to modify `size_` is through `push` or `pop`, which preserve the invariant.

This is the entire point of encapsulation: the class's contract is maintained not by hope or discipline, but by the compiler. A developer cannot *accidentally* break `BoundedQueue`'s invariant. They could maliciously cast and modify private fields, but that is deliberate, documented, unusual, and clearly breaking the contract.

### Usage

```cpp
int main() {
    BoundedQueue<int> q(3);
    
    q.push(10);
    q.push(20);
    q.push(30);
    
    assert(q.full());
    assert(q.push(40) == false);  // fails; full
    
    int x;
    q.pop(x);
    assert(x == 10);  // FIFO
    
    q.push(40);  // now succeeds
    
    assert(q.size() == 3);
    
    // Exception safety: if T's constructor throws, nothing changes
    BoundedQueue<std::string> sq(2);
    sq.push("hello");
    sq.push("world");
    
    // If the next push's string constructor throws, sq is unchanged
    // (strong exception guarantee achieved by checking size_ before modifying)
    
    return 0;
}
```

---

## 24.8a Real-World Examples: Encapsulation in the Standard Library

The C++ standard library is full of well-encapsulated classes. Examining them teaches good design:

**`std::vector<T>`:** The capacity and size are separate. The invariant is `size <= capacity`. The vector exposes `size()` and `capacity()` as const getters because these values do not have an invariant—they can be any non-negative value. But it does not expose `capacity()` as a setter. Instead, it exposes `reserve(size_t new_capacity)`, which says "ensure capacity is at least this large." This is Tell-Don't-Ask: you tell the vector what you need, and it decides whether to reallocate.

**`std::string`:** Similar to vector. It exposes `length()` but not a `setLength()`. To modify the string, you use methods like `append`, `erase`, `replace`. Each method is responsible for maintaining the invariant (the null terminator at the end, the correct length).

**`std::map<K, V>`:** It is a balanced tree. The invariant is that the tree is balanced (every node's children differ in height by at most 1) and ordered (left subtree < node < right subtree). None of this is exposed. Callers use `insert`, `erase`, `find`. The map maintains its invariants internally.

**`std::lock_guard<Mutex>`:** The invariant is "the mutex is locked and will be unlocked when this guard is destroyed." This is RAII. The guard has no public methods except those that access the locked resource. You cannot "unlock early" because there is no `unlock()` method. The destructor does it automatically.

Each of these classes would be much less useful if they exposed setters for internal fields. Their value comes from owning their invariants.

---

## 24.9 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **All public fields** | Direct access; no overhead; simple syntax | Invariants unenforceable; callers must maintain consistency; refactoring breaks callers | Never in production code. Acceptable only for trivial data holders (`struct Point { float x, y; }`). |
| **Private fields + public getters/setters** | Looks like encapsulation; easy to add logging later | Exposes implementation; still allows invariant violations; couples callers to field layout | Rarely. Better to design the public interface around behavior, not state. Valid only for very simple cases where setter genuinely enforces invariant. |
| **Private fields + public methods (Tell-Don't-Ask)** | Invariant fully controlled by class; callers cannot break it; public interface matches business logic; refactoring is local | Requires more thought about what operations are valid; method names matter; no direct field access | Default choice. Use for anything with a non-trivial invariant. |
| **Opaque pointer (pimpl)** | Implementation hidden from clients; can change internal fields without recompiling callers; maximum encapsulation | Extra pointer indirection on every method call; more code (separate Impl class); complexity in move/copy semantics | Large libraries shipped to many clients. When breaking internal changes must not force client recompilation. When you want to hide complexity. |
| **Protected fields in base class** | Derived classes can access; less restrictive than private | Breaks encapsulation between base and derived; harder to maintain invariant in a hierarchy | Rarely; prefer private with virtual methods. |

---

## 24.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Encapsulation is about hiding implementation details so users never know how it works." | Encapsulation is about making invariants unbreakable. The user does not need to know *how* the invariant is enforced, but they must know *what* the invariant is. |
| "Private fields are for security." | Private fields prevent *accidental* violations by external code. They are not secure against determined attackers (who can cast, or read memory). They are documentation enforced by the compiler. |
| "A class with all getters and setters is well-encapsulated." | No. If every field has a getter and setter, the invariant is exposed and unprotected. Encapsulation requires the class to control what state transitions are valid. |
| "Encapsulation is a C++ feature." | Encapsulation is a principle applicable to any language. C++ enforces it with access specifiers. Python enforces it by convention (`_private`). Go enforces it with capitalization (`Private` vs `public`). The principle is universal. |
| "I should never expose any data." | Correct for mutable data (fields that can change). But immutable data (read-only properties, capacity that never changes after construction) can be exposed with less risk. The rule is: if changing it would break the invariant, it must be private and controlled. |
| "Tell-Don't-Ask means the class should make *all* decisions." | Tell-Don't-Ask means the class should enforce *its own invariants*. If the decision is orthogonal to the invariant (e.g., how to display the queue), the class can let the caller decide. |

---

## 24.11 Exercises

1. **Identify an invariant.** Take a class you have written or used (e.g., `std::vector`, `std::map`, a database connection pool). Describe its invariant in one sentence. Now list three public methods and explain why each one preserves the invariant.

2. **Break and fix an invariant.** Write a `Temperature` class that stores temperature in Kelvin (which must be >= 0). Make it with all public fields, then have a colleague (or yourself) try to break the invariant (write `temp.kelvin = -100;`). Then rewrite with private fields and a single public method `setKelvin(double k)` that validates. Show that the invariant is now unbreakable.

3. **Getter/setter anti-pattern.** Write a `BadBankAccount` class with `getBalance()`, `setBalance()`. Now write a transaction that should be atomic: "transfer $100 from account A to B." Show that with the getter/setter interface, the transfer can be broken mid-way (A's balance is decremented but B's is not), violating the invariant "total money is conserved." Then rewrite `BankAccount` with a `transfer` method that atomically updates both accounts.

4. **Apply Tell-Don't-Ask.** You have a `Player` class with health and max_health. A caller wants to apply healing. Antipattern: `player.setHealth(player.getHealth() + heal_amount)` (or clamping in the caller). Better: write a `heal(int amount)` method that clamps internally. Now extend it: write a `takeDamage(int amount)` method. Observe how Tell-Don't-Ask makes the public interface match game logic ("heal", "take damage") rather than state access ("get health", "set health").

5. **Implement pimpl.** Take the `BoundedQueue<T>` example and rewrite it with pimpl (opaque pointer). Put the actual buffer and index fields in a separate `Impl` class in the .cpp file. Compile and verify that changing the layout of `Impl` does not require recompiling callers of `BoundedQueue` (only relink). What is the memory overhead?

6. **Design an invariant-preserving class.** Create a `ValidatedString` class that stores a string and enforces a minimum and maximum length. The invariant: `min_length <= length <= max_length`. Write the constructor, `set(const string&)` method, and `get() const` method such that the invariant is always true. Verify that no caller can break it. Now add an `append(const string&)` method that respects the length constraint.

7. **Conceptual: refactoring for encapsulation.** You have inherited a class with many public fields and a getter/setter for each. Describe a process for refactoring it to proper encapsulation: (a) identify the invariant(s), (b) design the minimal public interface needed by callers, (c) migrate callers one at a time (this may be the most work), (d) delete the old getters/setters. What are the risks? How would you test that the refactoring is correct?

---

## 24.12 Summary

Encapsulation is the design principle that makes large systems maintainable: a class owns its invariants and ensures they are never broken by external code. By hiding mutable state (private fields) and exposing only behavior that preserves the invariant (public methods), the class becomes a black box whose correctness can be verified locally. This local reasoning scales to thousands of call sites—a change to the class's implementation does not require understanding all its callers, only the class itself. Getters and setters are anti-encapsulation; instead, design the public interface around what the object can *do*, not what state it stores. Tell-Don't-Ask. At module boundaries, opaque pointers enforce encapsulation across translation units, allowing internal changes without recompiling clients. A well-encapsulated class is one where the invariant is as obvious from the header as it is from the implementation.

---

**[← Previous: What Is An Abstraction](01-what-is-an-abstraction.md)** · **[↑ Part 3](README.md)** · **[Next: Why Classes Exist →](03-why-classes-exist.md)**
