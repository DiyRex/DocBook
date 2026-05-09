# Chapter 27 — Composition vs Inheritance

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what inheritance actually does in memory and at the dispatch level — both "is-a" subtyping and implementation reuse.
2. Understand the Fragile Base Class problem and predict when inherited code will break silently.
3. Define the Liskov Substitution Principle and recognize violations in real code.
4. Compare inheritance and composition mechanically — dispatch cost, coupling, testability, extensibility.
5. Recognize when inheritance is the right tool and when it is a trap.
6. Implement composition patterns in C++20 — forwarding, delegation, strategy, decorator.
7. Refactor an inheritance hierarchy to composition and evaluate the tradeoffs.

Twenty-five years of object-oriented programming literature says "favor composition over inheritance." This chapter explains why, not through aesthetics or philosophy, but mechanically. And it identifies the narrow cases where inheritance is the right answer.

---

## 27.1 What Inheritance Actually Does

Inheritance is not one thing; it is two things, often conflated:

### Thing 1: Interface Inheritance

Derived class is a subtype of the base class. Client code can use a `Derived*` anywhere a `Base*` is expected.

```cpp
class Logger {
public:
    virtual void log(std::string_view msg) = 0;
    virtual ~Logger() = default;
};

class FileLogger : public Logger {
public:
    void log(std::string_view msg) override;
};

class ConsoleLogger : public Logger {
public:
    void log(std::string_view msg) override;
};

// Polymorphic use: the interface is the contract.
void processRequest(Logger& log) {
    log.log("processing...");
    // At this call site, we don't know if it's FileLogger or ConsoleLogger.
    // We only know Logger's contract.
}
```

In memory, this works via the **virtual table (vtable)**. Each class with a virtual function gets a static table of function pointers. Each instance holds a `vptr` (pointer to its vtable). A virtual call dereferences the vptr, then the function pointer within it.

```cpp
// Conceptually:
struct Logger_vtable {
    void (*log)(Logger*, std::string_view);
    void (*destructor)(Logger*);
};

struct Logger {
    Logger_vtable* vptr;
};

struct FileLogger : public Logger {
    std::string filename;
    // Inherits vptr; it points to FileLogger_vtable
};

struct FileLogger_vtable {
    void (*log)(Logger*, std::string_view) = &FileLogger::log;  // Points to FileLogger's impl
    void (*destructor)(Logger*) = &FileLogger::~FileLogger;
};

// At call site:
void processRequest(Logger& log) {
    log.vptr->log(&log, "processing...");
    // First indirection: load log.vptr (CPU cache hit, ~4 cycles)
    // Second indirection: load function pointer from vtable (~4 cycles)
    // Branch prediction: predict the branch in the pipeline (~1-2 cycles)
    // Total cost: ~10-20 CPU cycles on modern hardware, but only when cache misses.
}
```

The cost is paid once per virtual call. For tight loops calling virtual functions millions of times, this adds up. For normal application code (calling a virtual function a few thousand times), the cost is negligible compared to what the function itself does.

The contract says: "you can use me wherever my base class is expected, and I will behave according to my overridden methods." The abstraction is the interface; the subclass is a different implementation of that interface.

### Thing 2: Implementation Inheritance

Derived class reuses code from the base class without overriding it. Base class methods are inherited verbatim.

```cpp
class Logger {
public:
    void log(std::string_view msg) {
        auto now = std::chrono::system_clock::now();
        auto timestamp = std::chrono::format("{:%Y-%m-%d %H:%M:%S}", now);
        doLog(timestamp, msg);
    }
    
protected:
    virtual void doLog(std::string_view timestamp, std::string_view msg) = 0;
};

class FileLogger : public Logger {
protected:
    void doLog(std::string_view timestamp, std::string_view msg) override {
        file << timestamp << " " << msg << "\n";
    }
};
```

Here, `FileLogger` inherits the timestamp logic from `Logger` without repeating it. The derived class reuses the template. This is often called the **Template Method pattern**.

**The key problem:** In C++, you cannot have one without the other. If you inherit from a class, you get both subtyping *and* code reuse, whether you want them or not. Java separated them (abstract classes for interface, regular classes for reuse). C++ did not.

Both forms create coupling. Interface inheritance couples the client to the abstraction (by design, this is usually good). Implementation inheritance couples the derived class to the base class's internal structure and behavior — and this is where problems start.

---

## 27.2 What Composition Actually Does

Composition means: one object owns another, and delegates to it.

```cpp
class Logger {
public:
    void log(std::string_view msg) {
        auto now = std::chrono::system_clock::now();
        auto timestamp = std::chrono::format("{:%Y-%m-%d %H:%M:%S}", now);
        sink_->write(timestamp, msg);
    }
    
    void setSink(std::unique_ptr<Sink> sink) {
        sink_ = std::move(sink);
    }
    
private:
    std::unique_ptr<Sink> sink_;
};

class Sink {
public:
    virtual void write(std::string_view timestamp, std::string_view msg) = 0;
    virtual ~Sink() = default;
};

class FileSink : public Sink {
public:
    explicit FileSink(std::string_view filename);
    void write(std::string_view timestamp, std::string_view msg) override;
};
```

The `Logger` class owns a `Sink`. It does not inherit from `Sink`; it calls methods on it. The `Sink` interface says "I will write timestamped messages." Different `Sink` subclasses — `FileSink`, `ConsoleSink`, `NetworkSink` — implement that contract.

In memory:
```cpp
struct Logger {
    std::unique_ptr<Sink> sink_;  // Pointer to a Sink subclass
};

struct FileSink : public Sink {
    // Only implements the Sink interface.
    // Does not inherit logging logic.
};
```

**The key difference:** The `Logger` does not inherit from `Sink`. They are separate. The `Logger` *contains* a `Sink` and calls it. This is looser coupling: the `Sink` is swappable, mockable, testable in isolation.

---

## 27.3 The Fragile Base Class Problem

Here is the core danger of implementation inheritance.

Imagine a base class:

```cpp
class Queue {
public:
    void enqueue(int x) {
        buffer_.push_back(x);
    }
    
    int dequeue() {
        int x = buffer_.front();
        buffer_.erase(buffer_.begin());
        return x;
    }
    
    bool isEmpty() const {
        return buffer_.empty();
    }
    
protected:
    std::vector<int> buffer_;
};
```

Now someone derives a class that overrides `enqueue` to add logging:

```cpp
class LoggingQueue : public Queue {
public:
    void enqueue(int x) override {
        std::cout << "Enqueuing " << x << "\n";
        Queue::enqueue(x);
    }
};
```

The assumption is: "every enqueue will be logged, because I overrode the method." This is true as long as the base class only uses `enqueue` internally. But what if, a year later, the base class author decides to optimize by supporting both ends of the queue?

```cpp
class Queue {
public:
    void enqueue(int x) {
        buffer_.push_back(x);
    }
    
    int dequeue() {
        // Optimization: alternate which end we take from
        bool takeFromFront = --numOps_ % 2 == 0;
        if (takeFromFront) {
            int x = buffer_.front();
            buffer_.erase(buffer_.begin());
            return x;
        } else {
            int x = buffer_.back();
            buffer_.pop_back();
            return x;
        }
    }
    
    void priorityEnqueue(int x) {  // New method!
        std::cout << "Priority enqueue called\n";
        buffer_.insert(buffer_.begin(), x);  // Bypasses enqueue()!
    }
    
private:
    int numOps_ = 0;
};
```

The `LoggingQueue` still compiles, still runs, and the derived class code is unchanged. But now:

```cpp
LoggingQueue q;
q.enqueue(1);           // Logs: "Enqueuing 1" ✓
q.enqueue(2);           // Logs: "Enqueuing 2" ✓
q.priorityEnqueue(99);  // Logs nothing! ✗ Contract broken.
```

The new method `priorityEnqueue` adds items directly to the buffer, bypassing the `enqueue` override. The contract "all items added to the queue are logged" is broken, silently. No compiler error. No runtime exception. Just wrong behavior.

But wait, there is more. The original author of `Queue` might have intended something different:

```cpp
class Queue {
public:
    void enqueue(int x) {
        if (x < 0) throw std::invalid_argument("negative");
        buffer_.push_back(x);
    }
    
    // Later, for performance, add a method that skips validation:
    void enqueueUnsafe(int x) {
        buffer_.push_back(x);
    }
};

class LoggingQueue : public Queue {
public:
    void enqueue(int x) override {
        std::cout << "Enqueuing " << x << "\n";
        Queue::enqueue(x);  // Throws on negative.
    }
};

// Oops:
LoggingQueue q;
q.enqueueUnsafe(-1);  // No log, no validation. But user expected both.
```

This is the **Fragile Base Class problem**: changes to the base class's internal contract — which methods call which, what invariants are maintained, which methods can be skipped — can break derived classes in ways that are:

1. **Silent** (no compiler error, no runtime exception).
2. **Subtle** (the code still runs, just with wrong semantics).
3. **Surprising** (the derived class author did nothing wrong).
4. **Cascading** (a change in one base class breaks multiple derived classes, and only one of them is tested today).

---

## 27.4 Liskov Substitution Principle

To manage inheritance safely, the **Liskov Substitution Principle (LSP)** states:

> **A subclass must be usable wherever its base class is used, without surprising the caller.**

More formally: if `S` is a subtype of `T`, then objects of type `S` may be substituted for objects of type `T` without breaking the program.

**What "without surprise" means:**

1. **Narrowing of preconditions is forbidden.** If the base method accepts any integer, the override must accept any integer. It cannot say "only positive integers."

2. **Widening of postconditions is forbidden.** If the base method promises to return a value in `[0, 100)`, the override must also. It cannot promise `[0, 200)`.

3. **Weakening of invariants is forbidden.** If the base class maintains an invariant ("this container is always sorted"), the subclass must maintain it too.

4. **Throwing new exceptions is forbidden.** If the base method does not throw, the override must not throw either (or must throw only subclasses of exceptions the base throws).

Examples of LSP violation:

```cpp
class Bird {
public:
    virtual void fly() = 0;
};

class Penguin : public Bird {
public:
    void fly() override {
        throw std::runtime_error("Penguins cannot fly");
    }
};

// Caller expects Bird to fly. Penguin::fly throws.
// This violates LSP.
```

```cpp
class Shape {
public:
    virtual double area() const = 0;
};

class Circle : public Shape {
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }
    
private:
    double r_;
};

class Rectangle : public Shape {
public:
    Rectangle(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
    
    // Now someone adds:
    void setWidth(double w) { w_ = w; }  // Modifies immutable contract!
    
private:
    double w_, h_;
};

// Contract: Shape::area() returns the area. It's immutable.
// Rectangle adds setWidth(), which mutates it.
// A caller who cached the area gets a stale value.
// LSP violation.
```

**The deeper rule:** if you inherit from a class, you are *promising to maintain its entire contract*, including behaviors that are not explicitly virtual. This is a strong constraint.

---

## 27.5 Why Composition Avoids These Problems

With composition, the `Logger` and `Sink` are separate abstractions. The `Logger` does not inherit from `Sink`; it owns one.

```cpp
class Logger {
public:
    void log(std::string_view msg) {
        auto now = std::chrono::system_clock::now();
        auto timestamp = std::chrono::format("{:%Y-%m-%d %H:%M:%S}", now);
        sink_->write(timestamp, msg);
    }
    
private:
    std::unique_ptr<Sink> sink_;
};

class Sink {
public:
    virtual void write(std::string_view timestamp, std::string_view msg) = 0;
    virtual ~Sink() = default;
};

class FileSink : public Sink { /* ... */ };
class ConsoleSink : public Sink { /* ... */ };
```

If the `Sink` interface changes — say, `write` gains a parameter — the `Logger` must adapt, but it does *not* break silently. The compiler will tell you `Logger::log` now calls the wrong signature. You see the problem immediately.

If `Sink` gains a new method, `Logger` is unaffected. It still calls `write` the same way.

The contract is clearer: `Logger` says "I will delegate to a Sink that implements the Sink interface." Nothing more. No invisible dependencies on internal structure.

**Loose coupling:** The `Sink` can be replaced at runtime.

```cpp
auto logger = std::make_shared<Logger>();
logger->setSink(std::make_unique<FileSink>("/var/log/app.log"));

// Later, for testing:
logger->setSink(std::make_unique<MemorySink>());  // Captures to memory.
```

With inheritance, you cannot do this; `LoggingQueue` is always a `Queue`, and you cannot change its base at runtime.

---

## 27.6 When Inheritance Is The Right Tool

Inheritance is not always wrong. There are cases where it is the right answer.

### Case 1: True Is-A Relationship

If the relationship is genuinely "subtype," inheritance is clear and correct.

```cpp
class Animal {
public:
    virtual void makeSound() const = 0;
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void makeSound() const override { std::cout << "Woof\n"; }
};

class Cat : public Animal {
public:
    void makeSound() const override { std::cout << "Meow\n"; }
};
```

Here, `Dog` *is* an `Animal`. The relationship is not "Dog has an Animal"; it's "Dog is a kind of Animal." Inheritance makes the contract obvious.

Contrast with composition, which would be awkward:

```cpp
class Dog {
public:
    void makeSound() const { animal_->makeSound(); }
private:
    std::unique_ptr<Animal> animal_;  // Weird. Dog is not an Animal?
};
```

The composition version obscures the relationship. The inheritance version clarifies it.

**The rule:** if callers will use `Dog*` in place of `Animal*` without translation, inheritance is right. If the relationship is really "Dog delegates to an Animal," use composition.

### Case 2: Framework Inversion of Control

Frameworks often use inheritance as the hook for customization. The framework calls into your code.

```cpp
class RequestHandler {
public:
    virtual void handle(const HttpRequest& req, HttpResponse& resp) = 0;
    virtual ~RequestHandler() = default;
};

class UserController : public RequestHandler {
public:
    void handle(const HttpRequest& req, HttpResponse& resp) override {
        // Your code here.
        resp.status = 200;
        resp.body = "OK";
    }
};

// Framework:
std::unique_ptr<RequestHandler> handler = std::make_unique<UserController>();
handler->handle(incoming_request, response);
```

Here, the framework owns the base class (`RequestHandler`), and you provide the subclass (`UserController`). The framework calls `handle()` via the virtual interface. This is the **Template Method pattern** in its purest form.

Composition would not work cleanly here because the framework controls the creation and ownership logic. Inheritance is the hook mechanism.

### Case 3: Single-Implementation Interfaces

Sometimes a class has one and only one concrete subclass, and you want to enforce an abstraction without the runtime cost.

```cpp
class StringImpl {
public:
    virtual size_t length() const = 0;
    virtual char at(size_t i) const = 0;
    virtual ~StringImpl() = default;
};

class BasicString : public StringImpl {
public:
    explicit BasicString(std::string_view s) : data_(s) {}
    
    size_t length() const override { return data_.size(); }
    char at(size_t i) const override { return data_[i]; }
    
private:
    std::string data_;
};
```

This is less common in practice. Composition with a single implementation is usually clearer. But in some cases — especially in compiler internals or abstract syntax trees — a single virtual interface with a single subclass is acceptable.

### Case 4: CRTP (Curiously Recurring Template Pattern)

This is not traditional inheritance, but it uses inheritance syntax to achieve static polymorphism:

```cpp
template<typename Derived>
class Base {
public:
    void action() {
        static_cast<Derived*>(this)->doAction();
    }
};

class Derived : public Base<Derived> {
public:
    void doAction() { std::cout << "Derived action\n"; }
};
```

Here, the base class calls a method on the derived class, but it is resolved at compile time (no vtable). This avoids the virtual call overhead while maintaining a strong contract.

CRTP is right when:
1. You want inheritance's contract clarity.
2. You need zero virtual overhead.
3. The derived class is known at instantiation time (you cannot swap it at runtime).

---

## 27.7 Composition Patterns in C++

Composition is flexible. Here are the common patterns.

### Pattern 1: Simple Delegation

One object owns another and forwards calls.

```cpp
class Logger {
public:
    explicit Logger(std::unique_ptr<Sink> sink) : sink_(std::move(sink)) {}
    
    void info(std::string_view msg) {
        sink_->write("INFO", msg);
    }
    
    void error(std::string_view msg) {
        sink_->write("ERROR", msg);
    }
    
private:
    std::unique_ptr<Sink> sink_;
};
```

The `Logger` is the **wrapper**; it owns the `Sink` and forwards method calls to it. The `Sink` is the **component**.

### Pattern 2: Strategy Pattern

A family of algorithms are encapsulated, and the algorithm is selected at runtime.

```cpp
class SortStrategy {
public:
    virtual void sort(std::vector<int>& data) = 0;
    virtual ~SortStrategy() = default;
};

class QuickSort : public SortStrategy {
public:
    void sort(std::vector<int>& data) override {
        // Quicksort implementation
    }
};

class MergeSort : public SortStrategy {
public:
    void sort(std::vector<int>& data) override {
        // Mergesort implementation
    }
};

class Sorter {
public:
    explicit Sorter(std::unique_ptr<SortStrategy> strategy)
        : strategy_(std::move(strategy)) {}
    
    void sort(std::vector<int>& data) {
        strategy_->sort(data);
    }
    
    void setStrategy(std::unique_ptr<SortStrategy> strategy) {
        strategy_ = std::move(strategy);
    }
    
private:
    std::unique_ptr<SortStrategy> strategy_;
};
```

The `Sorter` does not inherit from `SortStrategy`; it owns one and calls it. At runtime, you can swap the strategy.

### Pattern 3: Decorator Pattern

An object wraps another and adds behavior.

```cpp
class DataSource {
public:
    virtual std::string read() = 0;
    virtual ~DataSource() = default;
};

class FileDataSource : public DataSource {
public:
    explicit FileDataSource(std::string_view path) : path_(path) {}
    std::string read() override { /* Read from file */ }
private:
    std::string path_;
};

class EncryptedDataSource : public DataSource {
public:
    explicit EncryptedDataSource(std::unique_ptr<DataSource> source)
        : source_(std::move(source)) {}
    
    std::string read() override {
        auto data = source_->read();
        return decrypt(data);  // Add behavior.
    }
    
private:
    std::unique_ptr<DataSource> source_;
    std::string decrypt(const std::string& data) { /* ... */ }
};
```

The `EncryptedDataSource` owns a `DataSource` (it could be a `FileDataSource` or another decorator). It calls `read()` on it, then decrypts the result. This chains decorators elegantly:

```cpp
auto base = std::make_unique<FileDataSource>("/data.txt");
auto encrypted = std::make_unique<EncryptedDataSource>(std::move(base));
auto compressed = std::make_unique<CompressedDataSource>(std::move(encrypted));

std::string data = compressed->read();  // Decompresses, then decrypts, then reads.
```

With inheritance, you would need a multi-level hierarchy, which explodes combinatorially.

### Pattern 4: Adapter Pattern

An object wraps another to present a different interface.

```cpp
class OldLogger {
public:
    void writeLog(const char* text) { /* Old API */ }
};

class LoggerAdapter : public Logger {
public:
    explicit LoggerAdapter(OldLogger& old) : old_(old) {}
    
    void log(std::string_view msg) override {
        old_.writeLog(std::string(msg).c_str());
    }
    
private:
    OldLogger& old_;
};
```

The `LoggerAdapter` owns (or references) an `OldLogger` and presents a new interface. This lets old code work with new abstractions.

A real-world example: you have legacy code that writes to stdout directly. New code expects a `Logger` interface. Rather than rewriting the legacy code, you wrap it:

```cpp
class StdoutAdapter : public Logger {
public:
    void log(std::string_view msg) override {
        printf("%s\n", std::string(msg).c_str());  // Legacy API
    }
};

// Inject it:
auto logger = std::make_unique<StdoutAdapter>();
processRequest(*logger);  // Uses new interface, calls old API.
```

The adapter pattern is not just for compatibility; it is also a bridge between different conceptual models. For example, a database connection pool might be wrapped to present the interface of a single connection:

```cpp
class PoolAdapter : public DbConnection {
public:
    explicit PoolAdapter(ConnectionPool& pool) : pool_(pool) {}
    
    ResultSet query(std::string_view sql) override {
        auto conn = pool_.acquire();
        auto result = conn->query(sql);
        pool_.release(std::move(conn));
        return result;
    }
    
private:
    ConnectionPool& pool_;
};
```

The caller sees a `DbConnection`, but the adapter manages the pool underneath.

---

## 27.8 Worked Example: From Inheritance to Composition

Let's refactor a real inheritance hierarchy to composition and compare.

### Starting Point: Inheritance Hierarchy

```cpp
class Logger {
public:
    void log(std::string_view msg) {
        auto timestamp = getTimestamp();
        doLog(timestamp, msg);
    }
    
    void setMinLevel(Level lvl) { minLevel_ = lvl; }
    
protected:
    virtual void doLog(std::string_view timestamp, std::string_view msg) = 0;
    
    std::string getTimestamp() const {
        auto now = std::chrono::system_clock::now();
        return std::chrono::format("{:%Y-%m-%d %H:%M:%S}", now);
    }
    
    Level minLevel_ = Level::Info;
};

class FileLogger : public Logger {
protected:
    explicit FileLogger(std::string_view path) : path_(path), file_(path) {}
    
    void doLog(std::string_view timestamp, std::string_view msg) override {
        file_ << timestamp << " " << msg << "\n";
        file_.flush();
    }
    
private:
    std::string path_;
    std::ofstream file_;
};

class RotatingFileLogger : public FileLogger {
protected:
    explicit RotatingFileLogger(std::string_view path, size_t maxSize)
        : FileLogger(path), maxSize_(maxSize) {}
    
    void doLog(std::string_view timestamp, std::string_view msg) override {
        if (file_.tellp() > maxSize_) {
            rotateFile();
        }
        FileLogger::doLog(timestamp, msg);
    }
    
private:
    void rotateFile() { /* Rename file, open new one */ }
    size_t maxSize_;
};
```

**Problems:**

1. Three-level hierarchy is fragile. Changes to `Logger` or `FileLogger` affect `RotatingFileLogger`.
2. Constructors must chain through the hierarchy.
3. Cannot easily test `RotatingFileLogger` without a real file.
4. Cannot combine rotation with other behaviors (e.g., compression).
5. If you want filtering (only log errors), you must add it to `Logger`, affecting all subclasses.

### Refactored: Composition with Policies

```cpp
class RotationPolicy {
public:
    virtual bool shouldRotate(size_t currentSize) const = 0;
    virtual void rotate(std::ofstream& file) = 0;
    virtual ~RotationPolicy() = default;
};

class SizeBasedRotation : public RotationPolicy {
public:
    explicit SizeBasedRotation(size_t maxSize) : maxSize_(maxSize) {}
    
    bool shouldRotate(size_t currentSize) const override {
        return currentSize > maxSize_;
    }
    
    void rotate(std::ofstream& file) override {
        // Rotate logic
    }
    
private:
    size_t maxSize_;
};

class NoRotation : public RotationPolicy {
public:
    bool shouldRotate(size_t) const override { return false; }
    void rotate(std::ofstream&) override {}
};

class Sink {
public:
    virtual void write(std::string_view timestamp, std::string_view msg) = 0;
    virtual ~Sink() = default;
};

class FileSink : public Sink {
public:
    FileSink(std::string_view path,
             std::unique_ptr<RotationPolicy> rotation = std::make_unique<NoRotation>())
        : path_(path), file_(path), rotation_(std::move(rotation)) {}
    
    void write(std::string_view timestamp, std::string_view msg) override {
        if (rotation_->shouldRotate(file_.tellp())) {
            rotation_->rotate(file_);
        }
        file_ << timestamp << " " << msg << "\n";
        file_.flush();
    }
    
private:
    std::string path_;
    std::ofstream file_;
    std::unique_ptr<RotationPolicy> rotation_;
};

class Logger {
public:
    explicit Logger(std::unique_ptr<Sink> sink) : sink_(std::move(sink)) {}
    
    void log(std::string_view msg) {
        auto timestamp = getTimestamp();
        sink_->write(timestamp, msg);
    }
    
private:
    std::unique_ptr<Sink> sink_;
    
    std::string getTimestamp() const {
        auto now = std::chrono::system_clock::now();
        return std::chrono::format("{:%Y-%m-%d %H:%M:%S}", now);
    }
};
```

**Usage:**

```cpp
auto rotation = std::make_unique<SizeBasedRotation>(1024 * 1024);  // 1MB
auto sink = std::make_unique<FileSink>("/var/log/app.log", std::move(rotation));
auto logger = std::make_unique<Logger>(std::move(sink));

logger->log("Application started");
```

**Advantages:**

1. **Testability:** Mock the `Sink` and `RotationPolicy` independently.
   ```cpp
   class MockSink : public Sink {
   public:
       void write(std::string_view timestamp, std::string_view msg) override {
           messages.push_back(std::string(msg));
       }
       std::vector<std::string> messages;
   };
   
   // Unit test:
   void testLoggingIncludesTimestamp() {
       auto mockSink = std::make_unique<MockSink>();
       auto& mockRef = *mockSink;
       auto logger = std::make_unique<Logger>(std::move(mockSink));
       
       logger->log("test");
       
       ASSERT_EQ(mockRef.messages.size(), 1);
       ASSERT_THAT(mockRef.messages[0], ContainsSubstring("2024-"));
   }
   ```
   
   With inheritance, you cannot test `RotatingFileLogger` without a real file. With composition, the test is fast and deterministic.

2. **Extensibility:** Add new sinks without touching `Logger`.
   ```cpp
   class NetworkSink : public Sink {
   public:
       explicit NetworkSink(std::string_view host, int port)
           : socket_(host, port) {}
       
       void write(std::string_view timestamp, std::string_view msg) override {
           std::string formatted = std::format("{} {}", timestamp, msg);
           socket_.send(formatted);
       }
       
   private:
       TcpSocket socket_;
   };
   
   // Use it without modifying Logger:
   auto sink = std::make_unique<NetworkSink>("logs.example.com", 5140);
   auto logger = std::make_unique<Logger>(std::move(sink));
   ```

3. **Composition of behaviors:** Chain sinks to combine effects.
   ```cpp
   class FilteringSink : public Sink {
   public:
       explicit FilteringSink(std::unique_ptr<Sink> inner, Level minLevel)
           : inner_(std::move(inner)), minLevel_(minLevel) {}
       
       void write(std::string_view timestamp, std::string_view msg) override {
           if (shouldLog(msg)) {
               inner_->write(timestamp, msg);
           }
       }
       
   private:
       std::unique_ptr<Sink> inner_;
       Level minLevel_;
       
       bool shouldLog(std::string_view msg) const {
           // Parse level from msg, check against minLevel_
           return true;
       }
   };
   
   // Chain them:
   auto fileSink = std::make_unique<FileSink>("/var/log/app.log");
   auto filtered = std::make_unique<FilteringSink>(std::move(fileSink), Level::Warning);
   auto networkAndFile = std::make_unique<NetworkSink>("example.com", 5140);
   
   // This is not possible cleanly with inheritance without a combinatorial explosion.
   ```

4. **Runtime swapping:** Change the sink or rotation at runtime.
   ```cpp
   auto logger = std::make_shared<Logger>(std::make_unique<ConsoleSink>());
   
   // During testing, replace it:
   logger->setSink(std::make_unique<MockSink>());
   
   // In production, switch to file-based:
   logger->setSink(std::make_unique<FileSink>("/var/log/app.log"));
   ```

---

## 27.8a Costs and Hidden Assumptions

Before choosing an approach, understand the costs:

**Inheritance costs:**
- **Virtual call overhead:** Each virtual call is ~10-20 CPU cycles (though modern CPUs mitigate this). For a tight loop calling millions of times, this adds up. Most application code does not hit this limit.
- **Coupling cost:** The derived class is tightly coupled to the base class's internal structure. A base class method calls another base class method, and a derived class overrides one of them; the behavior depends on the internal call order. If the call order changes, the derived class breaks.
- **Hierarchy cost:** Once you build a three-level hierarchy, adding a fourth level or moving behavior between levels becomes expensive. The hierarchy is a prison.
- **Testing cost:** You cannot easily test a derived class in isolation; you must instantiate the entire hierarchy.

**Composition costs:**
- **Indirection cost:** If both the wrapper and the wrapped object are polymorphic, you have two vtable lookups per operation. This is slower than one, but still negligible in most code.
- **Boilerplate cost:** Forwarding methods are visible and must be written. This is intentional — visibility is a feature, not a bug.
- **Semantic cost:** The relationship is less obvious. A `FileLogger` "is a" `Logger` by inheritance is clearer than a `Logger` "contains a" `FileSink` at first glance. But clarity at first glance is a poor goal; clarity on the fifth read, when the codebase is changing, matters more.

**CRTP costs:**
- **Compile-time cost:** Template instantiation is slow. A moderately complex CRTP hierarchy can add noticeably to compile times.
- **Runtime flexibility:** You cannot swap implementations at runtime. The derived type must be known at instantiation.
- **Complexity:** CRTP is harder to read. It is a tool for performance-critical code, not for everyday use.

---

## 27.9 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Inheritance** | Clear subtyping; single vtable; compact in memory; framework integration (inversion of control) | Fragile base class; tight coupling; hard to test; difficult to combine behaviors; cannot swap at runtime | True is-a relationships; framework hooks; single implementation of interface |
| **Composition** | Loose coupling; swappable at runtime; easier to test; combines behaviors; cleaner separation of concerns | Extra indirection (two vtable lookups if both layers are polymorphic); more forwarding code; less obvious relationship | Most production code; code that changes frequently; testable abstractions |
| **CRTP (Static Polymorphism)** | Zero virtual overhead; compile-time contract; no vtable bloat | Cannot swap at runtime; longer compile times; harder to read; only works when derived type is known at instantiation | Performance-critical code; compile-time strategies (e.g., algorithm selection) |
| **Mixin (Multiple Inheritance)** | Reuse code from multiple bases | Fragile; diamond problem; complex semantics; even fewer guarantees than single inheritance | Rare; when it appears, ask if composition could work instead |

---

## 27.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Inheritance is for code reuse." | Inheritance conflates interface and implementation. Composition is usually better for reuse because you do not inherit unwanted contracts. If all you want is to reuse code, make it a free function, a static utility, or a component. If all you want is to implement an interface, use composition with polymorphism. Inheritance should only be used for is-a relationships. |
| "Composition has more overhead because of extra indirection." | Modern CPUs cache vtable lookups and branch-predict virtual calls efficiently. With both composition and inheritance, you are paying for polymorphism. The real difference is *maintenance and testing*, not cycles. CRTP exists for the rare case where virtual overhead matters. Measure before optimizing. |
| "If I use composition, I lose the is-a relationship." | No. You *communicate* the relationship differently. An `EncryptedDataSource` "is a" `DataSource` — it implements the interface, passes it to functions expecting `DataSource*`. The relationship is proven by the interface contract, not the inheritance syntax. Inheritance is one way to express is-a; composition + interface is another. |
| "Composition requires more boilerplate (forwarding methods)." | True. But forwarding methods are explicit, visible, and easy to test. Inherited methods are implicit, invisible unless you read the base class, and can break silently. The boilerplate is the price of clarity. You pay for visibility up front to avoid paying for debugging later. This is a good trade. |
| "LSP violations are rare in practice." | They are common. Any time a subclass adds state that the base class does not know about, and the base class calls a virtual method, LSP can be violated silently. Any time a subclass overrides a non-virtual method, it can break the base class's assumptions. This happens constantly in real codebases — which is why codebases gradually refactor toward composition as they grow. |
| "I should use inheritance for flexibility." | The reverse is true. Inheritance locks you into a hierarchy at compile time. Composition lets you change the inner object (the strategy, the sink, the backend) at runtime. Composition is strictly more flexible. The only reason to choose inheritance for "flexibility" is misunderstanding. |
| "Composition means giving up on abstraction." | No. Composition *enforces* abstraction more strictly by making contracts explicit (the owned object's interface) rather than implicit (inherited methods you forget to read). The abstraction is clearer because the boundaries are visible. |
| "Virtual calls are slow, so I should avoid polymorphism." | Virtual calls are ~10-20 cycles. Most real work takes much more. A database query, a network send, a file write — these are milliseconds. The virtual call cost is noise. Premature optimization for virtual call overhead is a real bug in real codebases (people choose worse architectures to avoid a 20-cycle cost). Measure first. |
| "I should use CRTP instead of virtual functions to avoid vtable overhead." | Only if you have measured and found that virtual call overhead is a bottleneck. CRTP has its own costs: compile time, template bloat, no runtime polymorphism. Use it when you *know* you need zero-overhead abstraction, not because you are afraid virtual calls might be slow. |

---

## 27.11 Exercises

1. **Identify a fragile base class.** Find a real codebase (open-source is fine). Find a base class with non-virtual methods that call virtual methods. Identify one concrete way the base class could be changed such that a subclass would break silently without changes to the subclass itself. Bonus: write a test that would catch this breakage before the change is deployed.

2. **LSP violation analysis.** Examine the standard library's `std::istream` and `std::istringstream`. Does `istringstream` violate LSP? What is the symptom? (Hint: `seekg` on a stringstream behaves differently than on a file stream.) Why is this a violation, and what would a composition-based design do differently?

3. **Refactor to composition.** Take an existing inheritance hierarchy (three levels, preferably from a real codebase) and refactor it to composition with policies. Count:
   - Lines of code added/removed
   - Number of virtual calls per operation (use `nm` and objdump if needed)
   - Ease of writing unit tests (write three unit tests before and after)
   - Ability to combine behaviors at runtime (show three combinations that would have required new subclasses in the inheritance version)

4. **CRTP tradeoff.** Write the same generic container twice — once with virtual inheritance, once with CRTP. Measure:
   - Code size (use `size` or `wc` on the compiled object)
   - Compile time (use `-ftime-report` in GCC or `-Rpass` in Clang)
   - Runtime performance on a tight loop (does inlining happen? Use `objdump -d` to verify)
   - Ability to store instances in a heterogeneous collection (try it; what must you do differently?)

5. **Interface design.** Design an interface for "things that can be persisted to disk." Consider: (a) serialization to JSON, (b) serialization to binary, (c) resumable uploads, (d) compression. Would inheritance or composition fit better for supporting all four? Defend your choice in two detailed paragraphs, citing the costs and benefits from this chapter.

6. **Conceptual.** A coworker says "let's inherit from our HTTP client so we can add connection pooling, retry logic, and circuit-breaker pattern." What questions would you ask before agreeing? (Hint: think about what happens when the HTTP client library is updated.)

7. **Code reading.** Find an open-source C++ project with 50k+ lines. Search for inheritance hierarchies with three or more levels. Estimate: (a) how many are genuine is-a relationships? (b) how many could be refactored to composition? (c) for the ones that are genuine is-a, are there any LSP violations you can spot?

8. **Design decision.** You are designing a configuration system. Should the config object inherit from a base class `Config` that provides `get()`, `set()`, and `validate()`? Or should it own a `ConfigBackend` that provides these? What are the differences in terms of: (a) testability, (b) ability to load from multiple sources at once, (c) ability to switch backends at runtime?

---

## 27.12 Summary

Inheritance and composition are both tools for managing abstraction. Inheritance is simpler when the relationship is genuinely "is-a" and never needs to change. Composition is safer and more flexible in everything else.

The Fragile Base Class problem is not theoretical; it is the reason that most large codebases gradually shift from inheritance to composition as they grow. Liskov Substitution Principle violations are silent and common. Composition avoids both by making contracts explicit and allowing runtime swapping.

Use inheritance for true subtypes and framework hooks. Use composition for everything else. When you do inherit, be meticulous about Liskov Substitution — it is the only thing preventing your derived class from breaking in production.

---

> **[← Previous: Chapter 26 — What Objects Are In Memory](04-what-objects-are-in-memory.md)** · **[↑ Part 3](README.md)** · **[Next: Chapter 28 — Interfaces and Contracts →](06-interfaces-and-contracts.md)**
