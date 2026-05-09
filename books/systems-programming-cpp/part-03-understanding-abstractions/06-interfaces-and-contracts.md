# Chapter 28 — Interfaces and Contracts

An **interface** is a promise about syntax: "these operations exist and take these arguments." A **contract** is a promise about behavior: "these operations do this, with these guarantees, under these conditions." The compiler enforces the first. Only careful design, documentation, and tests enforce the second. This chapter teaches you to read both, design both, and recognize when an interface is a lie because the contract is broken.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Design an abstract base class interface that is *minimal* and *honesty*-enforcing.
2. Explain why destructors in C++ virtual interfaces must be virtual, and what happens if they are not.
3. Recognize the cost of virtual dispatch — and when that cost is worth paying.
4. Use C++20 Concepts to define compile-time interfaces that enforce shape without polymorphism.
5. Distinguish between structural and nominal typing, and explain the tradeoff.
6. Document contracts precisely: preconditions, postconditions, invariants, and side effects that the type system cannot express.
7. Apply Design by Contract principles to your own interfaces.
8. Build both testable interfaces and testable implementations (real and mock).
9. Recognize when an interface has promised more than it can deliver, and how to repair that leak.

---

## 28.1 What an Interface Is (And Isn't)

An interface is not a collection of method signatures. It is a *promise about what you can do*.

Here is an honest interface:

```cpp
class IReader {
public:
    virtual ~IReader() = default;
    virtual size_t read(byte* buf, size_t nbytes) = 0;
};
```

This says: "I am a thing you can read from. Call `read()` with a buffer and a count, and you will get back a number of bytes read."

The question every interface designer must answer: **what does 'read()' actually promise?**

- Does it promise to fill the buffer? (No — it might return 0, meaning EOF.)
- Does it promise to block until data is available? (Maybe — a socket read does; a file read doesn't.)
- Does it promise to be thread-safe? (You must specify.)
- Does it promise to read exactly what you asked for? (No — it might read less.)
- Does it promise anything about the lifetime of the returned data? (No — but you should document that the buffer is yours immediately.)

An interface without these answers is a *lie*. The compiler will accept it. It will compile. At runtime, code will make assumptions that the interface never promised, and those assumptions will be wrong.

### The Three Parts of an Honest Interface

1. **The signature**: what the function takes and returns. (`read(byte*, size_t) -> size_t`)
2. **The precondition**: what must be true before you call it. (buf is valid; nbytes > 0)
3. **The postcondition**: what is guaranteed to be true after. (0 <= returned value <= nbytes; buf[0..returned value) is filled)

Type systems handle part 1. Contracts handle parts 2 and 3. This chapter teaches you both.

---

## 28.2 Virtual Interfaces: Runtime Polymorphism

When you want runtime polymorphism in C++, you define an abstract base class with pure virtual methods. Every subclass that provides a different behavior implements those methods.

### A Worked Example: `IFileSystem`

```cpp
class IFileSystem {
public:
    virtual ~IFileSystem() = default;

    // Precondition: path is non-empty, valid UTF-8
    // Postcondition: if returned non-null, *data is valid until Destroy() called
    //                or another read() call is made
    virtual std::optional<std::string> read(std::string_view path) = 0;

    // Precondition: path is non-empty; data is valid
    // Postcondition: either write succeeds and returns true, or fails and returns false
    // Guarantee: atomic at the logical level (you don't get partial writes visible to concurrent reads)
    virtual bool write(std::string_view path, std::string_view data) = 0;

    // Precondition: path is non-empty
    // Postcondition: returns true iff file exists
    virtual bool exists(std::string_view path) = 0;
};
```

Note the comments. The type signature alone is not enough. We've said:
- What the preconditions are (what must be true when the caller invokes)
- What the postconditions are (what is guaranteed after the call)
- Whether atomicity is promised
- Whether the returned pointer is stable or temporary

Now we implement this interface two ways:

```cpp
class RealFileSystem : public IFileSystem {
public:
    std::optional<std::string> read(std::string_view path) override {
        std::ifstream file(std::string(path), std::ios::binary);
        if (!file) return std::nullopt;
        
        std::string data((std::istreambuf_iterator<char>(file)),
                         std::istreambuf_iterator<char>());
        return data;
    }

    bool write(std::string_view path, std::string_view data) override {
        std::ofstream file(std::string(path), std::ios::binary);
        if (!file) return false;
        
        file.write(data.data(), data.size());
        return file.good();
    }

    bool exists(std::string_view path) override {
        return std::filesystem::exists(path);
    }
};

class MockFileSystem : public IFileSystem {
private:
    std::map<std::string, std::string> files_;
    
public:
    std::optional<std::string> read(std::string_view path) override {
        auto it = files_.find(std::string(path));
        if (it == files_.end()) return std::nullopt;
        return it->second;
    }

    bool write(std::string_view path, std::string_view data) override {
        files_[std::string(path)] = std::string(data);
        return true;
    }

    bool exists(std::string_view path) override {
        return files_.count(std::string(path)) > 0;
    }
};
```

Now your code can accept either:

```cpp
class ConfigLoader {
private:
    IFileSystem& fs_;
    
public:
    ConfigLoader(IFileSystem& fs) : fs_(fs) {}
    
    bool load(std::string_view path) {
        auto data = fs_.read(path);
        if (!data) return false;
        // parse *data
        return true;
    }
};

// In production:
RealFileSystem prod_fs;
ConfigLoader config(prod_fs);
config.load("/etc/app.conf");

// In tests:
MockFileSystem mock_fs;
ConfigLoader config(mock_fs);
mock_fs.write("/etc/app.conf", "key=value");
config.load("/etc/app.conf");
```

The interface *decouples* the code that uses the filesystem from the code that implements it. You can test without touching disk.

### Why the Destructor Must Be Virtual

When you inherit from a base class, the base class is responsible for cleaning up. If the destructor is not virtual, deleting through a base pointer deletes the base but *not* the derived class:

```cpp
class IFileSystem { /* ... */ };

class RealFileSystem : public IFileSystem {
private:
    std::vector<char> buffer_;  // this might not be freed!
    
public:
    ~RealFileSystem() {
        // cleanup
    }
};

// Danger:
IFileSystem* fs = new RealFileSystem();
delete fs;  // calls IFileSystem::~IFileSystem(), not RealFileSystem::~RealFileSystem()
```

If `IFileSystem::~IFileSystem()` is not virtual, the derived destructor is never called. Any cleanup the derived class needs (closing file handles, freeing memory, flushing buffers) is skipped. This is a silent resource leak.

Make the destructor virtual:

```cpp
class IFileSystem {
public:
    virtual ~IFileSystem() = default;  // virtual
};
```

This is sometimes called the "virtual destructor rule": **if a class is meant to be inherited from and deleted polymorphically, its destructor must be virtual.**

### The Cost of Virtual Dispatch

Every virtual call incurs a *vtable lookup*. When you call `fs->read(path)`, the CPU must:

1. Load the pointer to the vtable from the object.
2. Load the function pointer from the vtable at the right offset.
3. Call the function.

For tight loops calling the same virtual function millions of times, this can be measurable. For typical application code, it is not. Modern compilers often *devirtualize* — if they can prove the object's type statically, they emit a direct call instead.

**Rule of thumb:** Use virtual interfaces when polymorphism is the right abstraction (you have multiple implementations and you want to swap them at runtime). Use the cost of virtual dispatch as a tie-breaker only, never as a primary reason.

---

## 28.3 Concepts: Compile-Time Interfaces

C++20 introduced Concepts, which allow you to define *structural* constraints on template parameters. A Concept is a compile-time predicate: "does this type satisfy these constraints?"

Here is `IReader` as a Concept:

```cpp
template<typename T>
concept Reader = requires(T& reader, std::byte* buf, size_t nbytes) {
    { reader.read(buf, nbytes) } -> std::convertible_to<size_t>;
};
```

This says: "A type T is a Reader if you can call `read(buf, nbytes)` on it and get back something convertible to `size_t`."

Now you can write generic code that works with any Reader:

```cpp
template<Reader R>
std::optional<std::string> read_all(R& reader) {
    std::string result;
    std::byte buf[4096];
    
    while (true) {
        size_t n = reader.read(buf, sizeof(buf));
        if (n == 0) break;
        result.append(reinterpret_cast<char*>(buf), n);
    }
    
    return result;
}
```

No virtual dispatch. No indirection. The compiler generates a specialized version of `read_all` for each Reader type. If the compiler can inline `read()`, it will. The abstraction is "free."

Compare:

```cpp
// Virtual interface: one function, vtable lookup per call
void process(IReader& reader) {
    byte buf[1024];
    size_t n = reader.read(buf, sizeof(buf));  // vtable lookup
}

// Concept: potentially inlined, zero overhead
template<Reader R>
void process(R& reader) {
    byte buf[1024];
    size_t n = reader.read(buf, sizeof(buf));  // possibly inlined
}
```

### Tradeoff: Compile Time vs Runtime Flexibility

- **Virtual interfaces**: one binary, multiple implementations loaded at runtime.
- **Concepts**: one implementation per type, loaded at compile time.

Virtual is more flexible for dynamically loaded plugins. Concepts are more flexible for micro-optimizations. Choose based on what your architecture needs.

```cpp
// If you need to load implementations at runtime:
std::unique_ptr<IReader> reader = load_plugin("reader.so");

// If you know all implementations at compile time:
if (config.use_buffered) {
    process(BufferedReader{});
} else {
    process(DirectReader{});
}
```

---

## 28.4 Structural vs Nominal Typing

**Nominal typing** (C++, Java): you must explicitly say "this class implements this interface." The relationship is part of the declaration.

```cpp
class MyReader : public IReader {  // <- nominal: explicit declaration
    // ...
};
```

**Structural typing** (Go, Python): a type satisfies an interface if it has the right methods. The relationship is implicit.

```go
// Go: MyReader satisfies Reader if it has a Read method, without saying so
type MyReader struct { /* ... */ }
func (m *MyReader) Read(buf []byte) (int, error) { /* ... */ }
```

### Costs and Benefits

**Nominal (C++, Java)**
- **Pro**: compiler catches typos (you said you implement `IReader` but forgot `read()`).
- **Pro**: explicit, auditable — anyone reading the declaration knows the intent.
- **Con**: coupling — `MyReader` must know about `IReader` to implement it.
- **Con**: brittle — renaming the interface breaks all implementations.

**Structural (Go)**
- **Pro**: decoupling — `MyReader` doesn't know it implements anything.
- **Pro**: loose — the same type can satisfy multiple interfaces without declaring it.
- **Con**: silent — you might implement something you didn't intend.
- **Con**: refactoring is risky — changing a method signature might accidentally break an interface you didn't know you were implementing.

C++20 Concepts are a third way: **structural declarations**. You declare the interface (the Concept), but implementations don't mention it. The compiler checks satisfaction structurally.

```cpp
template<typename T>
concept Reader = requires(T& r, std::byte* b, size_t n) {
    { r.read(b, n) } -> std::convertible_to<size_t>;
};

struct MyReader {
    size_t read(std::byte* buf, size_t n);
    // <- doesn't say "implements Reader"
};

// Compiler checks satisfaction automatically
template<Reader R> void process(R&);
process(MyReader{});  // <- compiler verifies MyReader satisfies Reader
```

---

## 28.5 Contracts: What the Type System Cannot Express

A type says: "this is a `size_t`." A contract says: "this `size_t` is in the range [0, 1024)" or "this `size_t` represents the number of bytes in this buffer, no more, no less."

Types are checked by the compiler. Contracts are checked by careful engineers and by tests.

### Preconditions: What Must Be True Before

```cpp
// Precondition: i >= 0 and i < vec.size()
int get_nth(const std::vector<int>& vec, size_t i) {
    return vec[i];  // undefined behavior if i is out of range
}
```

The type `size_t` doesn't capture "less than size()". If you violate the precondition, the program might crash, might return garbage, might appear to work and fail later. The contract is a promise from the caller.

```cpp
// Better: document and check
int get_nth(const std::vector<int>& vec, size_t i) {
    if (i >= vec.size()) {
        throw std::out_of_range("index out of range");
    }
    return vec[i];
}

// Or: return an optional
std::optional<int> get_nth(const std::vector<int>& vec, size_t i) {
    if (i >= vec.size()) return std::nullopt;
    return vec[i];
}
```

### Postconditions: What Is Guaranteed After

```cpp
// Postcondition: returned value is in [min, max]
// Postcondition: returned value < original array size
int find_index(const std::vector<int>& vec, int target) {
    // returns -1 if not found, otherwise the index
    auto it = std::find(vec.begin(), vec.end(), target);
    if (it == vec.end()) return -1;
    return std::distance(vec.begin(), it);
}
```

The return type `int` doesn't capture "range." The type system allows any `int`. The contract narrows it.

### Invariants: What Stays True

An **invariant** is a property that holds at "visible points" in the object's lifetime — between public method calls, never in the middle of one.

```cpp
class Vector {
private:
    int* data_;
    size_t size_;
    size_t capacity_;
    
    // Invariant: data_ is allocated (or nullptr if capacity_ == 0)
    // Invariant: size_ <= capacity_
    // Invariant: all size_ elements are initialized
    
public:
    void push_back(int x) {
        if (size_ == capacity_) {
            // reallocate
        }
        data_[size_++] = x;
        // Invariant holds here for the caller
    }
};
```

If you violate the invariant (say, set `size_ > capacity_`), calling any other method will corrupt memory or crash.

### Side Effects and Guarantees

```cpp
// Postcondition: socket is open and connected
// Side effect: allocates memory for buffers
// Guarantee: subsequent reads will block if data isn't available
bool connect(const std::string& host, int port);

// Postcondition: if returned true, file descriptor is closed
// Side effect: file is no longer accessible
bool close();

// Guarantee: thread-safe for concurrent reads
// Guarantee: NOT thread-safe for concurrent writes
std::optional<std::string> read_config();
```

Document these explicitly. They shape how code using your interface must behave.

---

## 28.6 Design by Contract (Briefly)

Eiffel (a language rarely used today, but with one enduring contribution) formalized contracts into the language:

```eiffel
-- Precondition: the caller promises i >= 1
-- Postcondition: result = fibonacci(i-1) + fibonacci(i-2)
fibonacci(i: INTEGER): INTEGER
    require
        i >= 1
    do
        -- implementation
    ensure
        result >= 0
    end
```

The language enforces these. Violate a precondition, the program halts with a clear error. Violate a postcondition, same. This caught bugs that would otherwise hide in testing.

C++ doesn't have contract enforcement in the language (though C++23 has a proposal). But you can apply the principle:

```cpp
// Explicit precondition
void set_size(size_t n) {
    if (n < 0 || n > MAX_SIZE) {
        throw std::invalid_argument("size out of range");
    }
    // ...
}

// Explicit postcondition (assert)
void add(int x) {
    int old_size = size_;
    // ... implementation
    assert(size_ == old_size + 1);
}
```

**The principle:** preconditions are the caller's responsibility; postconditions are the function's responsibility. If the caller violates a precondition, it's their bug. If the function violates a postcondition, it's your bug.

This shifts the burden of reasoning. You don't have to defend against every possible input; you defend against inputs that violate the stated precondition, and you must deliver the stated postcondition.

---

## 28.7 Stable vs Unstable Interfaces

A **stable interface** makes promises about evolution. A **unstable interface** is internal and can change.

### Semantic Versioning (SemVer)

If your interface is public, follow SemVer:

- **MAJOR.MINOR.PATCH**
- **MAJOR bump**: breaking change (removed method, changed signature).
- **MINOR bump**: new method, new optional parameter, new precondition loosened.
- **PATCH bump**: bug fix, postcondition strengthened, invariant strengthened.

A client depending on version `2.3.0` can safely use `2.3.5` (PATCH — no breaking changes) and `2.5.0` (MINOR — new features, but old ones still work). They cannot safely use `3.0.0` (MAJOR — contracts changed).

```cpp
// v1.0: read(buf, nbytes) -> size_t
// v1.1: added read(buf, nbytes, timeout) -> std::optional<size_t>
// v2.0: removed read(buf, nbytes), now read() takes a buffer object
```

The v1.1 change is backward compatible. The v2.0 change is not.

### Internal Interfaces

Code internal to your module can be less stable. Refactor freely. But document: "this interface is internal to `filesystem/` and may change."

```cpp
// In filesystem/reader.h
class IReader { /* public interface */ };

// In filesystem/internal.h
class InternalBuffer { /* implementation detail, subject to change */ };
```

Anyone importing `filesystem/internal.h` is opting into instability. That's acceptable. Anyone importing `filesystem/reader.h` expects stability across versions.

---

## 28.8 Worked Example: A Testable Config Loader

Let's build a realistic interface and see how it decouples code from its dependencies.

```cpp
// ============ Interface ============

// Precondition: path is non-empty
// Postcondition: if returned value is non-empty, it is valid UTF-8 text
// Throws: std::runtime_error if the file exists but can't be read
// Guarantee: thread-safe for concurrent reads
std::optional<std::string> read_file(std::string_view path);

// ============ Production Implementation ============

std::optional<std::string> read_file(std::string_view path) {
    std::ifstream file(std::string(path));
    if (!file.is_open()) {
        if (std::filesystem::exists(path)) {
            throw std::runtime_error("file exists but cannot be opened");
        }
        return std::nullopt;  // file doesn't exist
    }
    
    std::stringstream buffer;
    buffer << file.rdbuf();
    return buffer.str();
}

// ============ The Service that Uses It ============

class ConfigService {
private:
    std::function<std::optional<std::string>(std::string_view)> reader_;
    
public:
    // Constructor injection: pass the reader as a dependency
    ConfigService(
        std::function<std::optional<std::string>(std::string_view)> reader =
            read_file
    ) : reader_(reader) {}
    
    // Precondition: path is non-empty
    // Postcondition: config is loaded and validated
    // Throws: std::runtime_error if config is invalid
    bool load_config(std::string_view path) {
        auto content = reader_(path);
        if (!content) {
            throw std::runtime_error("config file not found");
        }
        
        // parse and validate
        return parse_config(*content);
    }
};

// ============ Test Setup ============

TEST(ConfigServiceTest, LoadsConfigFromFile) {
    std::map<std::string, std::string> mock_files;
    mock_files["app.conf"] = "debug=true\nport=8080";
    
    auto mock_reader = [&](std::string_view path) -> std::optional<std::string> {
        auto it = mock_files.find(std::string(path));
        if (it == mock_files.end()) return std::nullopt;
        return it->second;
    };
    
    ConfigService config(mock_reader);
    EXPECT_TRUE(config.load_config("app.conf"));
}

TEST(ConfigServiceTest, ThrowsWhenFileNotFound) {
    ConfigService config([](std::string_view) { return std::nullopt; });
    EXPECT_THROW(config.load_config("missing.conf"), std::runtime_error);
}

TEST(ConfigServiceTest, ThrowsWhenFileCannotBeRead) {
    auto mock_reader = [](std::string_view) -> std::optional<std::string> {
        throw std::runtime_error("permission denied");
    };
    
    ConfigService config(mock_reader);
    EXPECT_THROW(config.load_config("forbidden.conf"), std::runtime_error);
}
```

Notice:
1. The interface is a promise: "give me a file path, get back a string or nothing."
2. The preconditions and postconditions are documented.
3. The ConfigService doesn't depend on the filesystem directly. It depends on the interface.
4. Tests inject a mock reader that returns canned data.
5. No disk I/O in tests. No flaky file permissions. No cleanup of temp files.

---

## 28.9 When Interfaces Leak

All non-trivial interfaces leak. The leak might be:

### Leak 1: Wrong Abstraction Boundary

You promised: "read a file."
What you delivered: "read a file from disk using the C standard library."

When someone asks "can I read from a network file?" or "can I read from a compressed archive?" your interface doesn't stretch.

**Fix**: Define the interface at the right boundary. Instead of "read_file", define "read_from_source" and let implementations decide what that source is.

### Leak 2: Missing Preconditions

```cpp
std::vector<int>& get_internal_buffer();  // precondition: don't modify it
```

You promise a buffer. You don't promise the caller won't modify it. If they do, bad things happen. But how do they know it's internal?

**Fix**: return `const std::vector<int>&` if the caller shouldn't modify it. Preconditions should be enforced by types when possible.

### Leak 3: Silent Failure

```cpp
bool initialize();  // returns false on failure. What failure?
```

The caller doesn't know whether initialization failed because memory is exhausted, the file doesn't exist, permission is denied, or the disk is full. They can't retry intelligently.

**Fix**: be more specific. Throw an exception with context. Return a `Result<T, Error>` with details.

```cpp
void initialize();  // throws std::runtime_error with context
// or
enum class InitError { NoMemory, FileNotFound, PermissionDenied };
Result<void, InitError> initialize();
```

### Leak 4: Undocumented Thread Safety

```cpp
std::string get_name();  // Is this thread-safe?
```

The caller doesn't know. They might call it concurrently from two threads and get corruption. The interface is a lie.

**Fix**: document it.

```cpp
// Thread-safe for concurrent reads
// NOT thread-safe for concurrent writes
std::string get_name();
```

---

## 28.10 Tradeoffs

| Choice | Cost | Benefit | When to use |
|---|---|---|---|
| **Virtual interface** | vtable lookup per call; runtime polymorphism; indirection | Multiple implementations; runtime swappability; plugin support | When you need to swap implementations at runtime or have genuinely different implementations. |
| **Concept (template)** | Compile-time code bloat; longer compilation | Zero overhead; inlining; specialization per type | When you know all implementations at compile time or need maximum performance. |
| **Function pointer / `std::function`** | Extra indirection; heap allocation for captures | Lightweight, decoupled implementations | When you have a few simple implementations and want simplicity. |
| **Structural typing (Go-like)** | Implicit; risky refactoring; hard to trace | Loose coupling; no pre-declaration required | When your language supports it and you accept the risks. |
| **Nominal typing (C++, Java)** | Explicit coupling; refactoring-heavy | Safe; compiler-checked; auditable | When you want explicit contracts and safety. |
| **Stable public interface** | Maintenance burden; slower evolution | Predictable; versioning guarantees | When your code is a library or service other teams depend on. |
| **Unstable internal interface** | Silent breakage across modules | Fast iteration; refactoring freedom | For implementation details and internal tools. |

---

## 28.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "An interface is just the list of methods." | An interface is the method signatures *plus* the preconditions, postconditions, and invariants. The type system captures the first; documentation captures the rest. |
| "Virtual functions are always slow." | Virtual functions have measurable cost, but modern CPUs with branch prediction and inlining handle them well. Profile before optimizing; don't optimize on principle. |
| "Concepts replace inheritance." | Concepts are structural; inheritance is nominal. They answer different questions. Use inheritance for polymorphic hierarchies; use Concepts for generic programming. |
| "A contract is just a comment." | A contract is a promise, documented or not. If you don't document it, you'll break it accidentally. If you do document it and break it, it's a bug. Either way, it matters. |
| "If the interface is stable, clients never need to check for breakage." | SemVer means MINOR and PATCH don't break existing clients. But they may change behavior subtly. Test your dependencies' upgrades in staging before production. |
| "Thread-safety is a property of the implementation, not the interface." | Thread-safety constraints on the caller (e.g., "not thread-safe for concurrent writes") are part of the contract. Document them in the interface. |
| "I should design the most flexible interface possible." | Flexibility has costs: more edge cases, more documentation, slower implementation. Design for the problem you have, not for all problems you might have. |

---

## 28.12 Exercises

1. **Find a leaky interface.** Pick a method in a codebase you know. List its preconditions (what must be true before calling it) and postconditions (what is guaranteed after). Are these documented? If not, write them. If yes, check if the implementation actually delivers them. Report what you find.

2. **Design a testable interface.** Write a class that processes data from a source (file, network, database — your choice). Define the interface that *separates* the processing logic from the source. Now write two implementations of that interface: one that is real, one that is a mock. Write three tests that use the mock. Can you test all the logic without accessing the real source?

3. **Nominal vs structural.** Write a C++ interface for a "shape" with area() and perimeter(). Now write two implementations: Rectangle and Circle. Then write a Concept that defines the shape interface. Use both — one with virtual inheritance, one with templates. Which is easier to use? Which is more flexible?

4. **Contract documentation.** Pick a method from the standard library (e.g., `std::vector::insert`, `std::string::find`). Read its documentation. List every precondition and postcondition you can identify. Now look at the implementation. Does the implementation enforce the preconditions, or does the caller? Does the implementation guarantee the postconditions?

5. **Stability and versioning.** Imagine you have published an interface as a public API. You now realize it has a design flaw. List three ways to fix it without breaking existing clients. For each, note whether it's a SemVer MINOR or MAJOR bump, and why.

6. **Thread-safety contract.** Write a simple mutable class (e.g., a counter, a cache). Decide on a thread-safety contract: is it thread-safe? If so, for what operations? Document it clearly in the class. Now implement it to match the contract. Finally, write a test that would fail if you broke the contract.

---

## 28.13 Summary

An interface is a promise about syntax and behavior. The compiler checks the syntax (the signature); you must check the behavior (the contract) through careful design, documentation, and testing.

In C++, you have two paths: virtual interfaces for runtime polymorphism (cost: indirection; benefit: swappability), and Concepts for compile-time polymorphism (cost: code bloat; benefit: zero overhead).

Contracts — preconditions, postconditions, invariants — are the parts of an interface that types don't capture. Document them. They are not optional, and they matter.

All interfaces leak. When they do, you must understand both sides of the boundary. Know your preconditions and postconditions; know the implementation; know why they mismatch. That knowledge is what separates debugging from guessing.

---

> **[← Previous: Chapter 27 — Composition vs Inheritance](05-composition-vs-inheritance.md)** · **[↑ Part 3](README.md)** · **[Next: Chapter 29 — Polymorphism Internals →](07-polymorphism-internals.md)**
