# Chapter 10 — Dependency Injection From Scratch

Dependency Injection is the most over-explained pattern in software. Strip away the framework jargon—the containers, the attributes, the reflection—and it is two words: "pass dependencies in." This chapter explains why that is enough, and when a container actually buys you something.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define Dependency Injection precisely as a discipline of passing an object's dependencies as parameters rather than having it create them.
2. Explain why DI matters: local reasoning and testability.
3. Recognize the three forms of DI—constructor, setter, and method injection—and choose the right one for each situation.
4. Build a realistic C++ program using manual DI with no framework.
5. Identify when a DI container solves a real problem versus when it adds complexity.
6. Recognize and reject the Service Locator anti-pattern.
7. Understand how popular frameworks (Spring, .NET, Laravel) implement DI and why C++ lags behind.

---

## 10.1 What Dependency Injection Actually Is

Before we define it, let's see what the absence of DI looks like:

```cpp
class ReportService {
private:
    Database db;        // created internally
    Clock clock;        // created internally
    
public:
    ReportService() {
        db.connect("localhost:5432");
        // clock is default-constructed
    }
    
    std::string generate(std::string_view user_id) {
        auto data = db.query("SELECT * FROM events WHERE user_id = ?", user_id);
        auto now = clock.now();
        return format_report(data, now);
    }
};
```

This class *creates its own dependencies*. To use it:

```cpp
ReportService service;
auto report = service.generate("user123");
```

Now here is the same class with DI:

```cpp
class ReportService {
private:
    IDatabase& db;
    IClock& clock;
    
public:
    ReportService(IDatabase& db, IClock& clock) : db(db), clock(clock) {}
    
    std::string generate(std::string_view user_id) {
        auto data = db.query("SELECT * FROM events WHERE user_id = ?", user_id);
        auto now = clock.now();
        return format_report(data, now);
    }
};
```

Now the dependencies are *passed in*:

```cpp
RealDatabase db;
SystemClock clock;
ReportService service(db, clock);
auto report = service.generate("user123");
```

That's it. That is the pattern.

**Dependency Injection is passing a class's dependencies as constructor (or method) parameters instead of having the class create them internally.**

Everything that follows—containers, factories, auto-resolution—is mechanism built on top of this simple discipline.

---

## 10.2 Why It Matters: Local Reasoning and Tests

Without DI, reasoning about a class requires understanding its entire transitive closure of dependencies. To understand `ReportService`, you must understand `Database` (how does `connect()` work? what if it fails?), and `Clock`, and anything those depend on. Understanding becomes global reasoning.

With DI, reasoning becomes *local*. `ReportService` says: "give me an `IDatabase` and an `IClock`, and I will work with what you give me." The implementation doesn't care whether the database is real, in-memory, or a mock. It treats its dependencies as contracts (Chapter 28).

### The Testing Argument

Without DI:

```cpp
TEST(ReportServiceTest, GeneratesReport) {
    ReportService service;
    auto report = service.generate("user123");
    // This test hits the real database, requires test infrastructure,
    // is slow, and is fragile (depends on database state).
}
```

With DI:

```cpp
class MockDatabase : public IDatabase {
public:
    std::vector<Record> query(std::string_view sql, std::string_view param) override {
        // Return canned data
        return { Record{"event1", "2024-05-09"}, Record{"event2", "2024-05-09"} };
    }
};

class MockClock : public IClock {
public:
    Timestamp now() override {
        return Timestamp{2024, 5, 9, 14, 30, 0};  // fixed time
    }
};

TEST(ReportServiceTest, GeneratesReport) {
    MockDatabase db;
    MockClock clock;
    ReportService service(db, clock);
    auto report = service.generate("user123");
    
    EXPECT_THAT(report, ContainsSubstring("event1"));
    EXPECT_THAT(report, ContainsSubstring("2024-05-09"));
    // Fast, deterministic, no side effects.
}
```

The test is now a *unit test*—it tests the class in isolation. With DI, the class is a pure function of its inputs and its dependencies. Without DI, it is tangled with its creation graph.

This difference scales. In a system with 50 classes, DI turns each class into a testable unit. Without it, testing a class often means testing 10 classes because they are all coupled through creation.

---

## 10.3 Three Forms of Injection

### Form 1: Constructor Injection (Preferred)

Dependencies are passed to the constructor. They live for the lifetime of the object.

```cpp
class ReportService {
private:
    IDatabase& db_;
    IClock& clock_;
    
public:
    ReportService(IDatabase& db, IClock& clock) : db_(db), clock_(clock) {}
};
```

**Pros:**
- Dependencies are visible at construction time. The object is never in a broken state.
- Dependencies are immutable after construction (if you keep them as `const` references or store as `const` pointers).
- The constructor signature documents what the object needs.

**Cons:**
- If a dependency becomes optional later, you must change the constructor signature (a breaking change).
- Constructors can become long with many dependencies (a code smell that you have too many concerns).

### Form 2: Setter Injection (For Optional Dependencies)

Some dependencies are optional—a logger, a cache, a metrics reporter. You can inject them after construction:

```cpp
class ReportService {
private:
    IDatabase& db_;
    IClock& clock_;
    ILogger* logger_ = nullptr;  // optional
    
public:
    ReportService(IDatabase& db, IClock& clock) : db_(db), clock_(clock) {}
    
    void set_logger(ILogger* logger) {
        logger_ = logger;
    }
    
    std::string generate(std::string_view user_id) {
        if (logger_) logger_->info("generating report");
        // ...
    }
};
```

**Pros:**
- Handles optional dependencies cleanly.
- Allows reconfiguration after construction.

**Cons:**
- The object can be partially initialized (missing the optional dependency).
- Setters are often overlooked; a reader might not realize a dependency exists.

Use setter injection sparingly. It is a signal that something is optional, and optional should be rare.

### Form 3: Method Injection (For Per-Call Dependencies)

Some dependencies vary per call. Pass them as method parameters:

```cpp
class ReportService {
private:
    IClock& clock_;
    
public:
    ReportService(IClock& clock) : clock_(clock) {}
    
    // The database comes from the caller; it might change per call
    std::string generate(std::string_view user_id, IDatabase& db) {
        auto data = db.query("SELECT * FROM events WHERE user_id = ?", user_id);
        auto now = clock_.now();
        return format_report(data, now);
    }
};
```

**Pros:**
- Flexible; the caller can use different databases for different calls.

**Cons:**
- Dependencies are not visible in the constructor signature.
- Harder to reason about (what if different threads pass different databases?).

Use method injection for dependencies that truly vary per call. For stable dependencies, prefer constructor injection.

---

## 10.4 Doing DI in Plain C++ (No Framework)

Let's build a realistic program: a service that generates daily activity reports. It needs:

1. A database to fetch user events.
2. A clock to timestamp reports.
3. A JSON serializer to format output.

All three are dependencies. We'll use constructor injection.

#### Step 1: Define Interfaces

```cpp
// database.h
class IDatabase {
public:
    virtual ~IDatabase() = default;
    
    struct Event {
        std::string id;
        std::string type;
        std::string timestamp;
    };
    
    // Precondition: user_id is non-empty
    // Postcondition: returns events for the user, or empty if none found
    virtual std::vector<Event> fetch_events(std::string_view user_id) = 0;
};

// clock.h
class IClock {
public:
    virtual ~IClock() = default;
    
    // Postcondition: returns current UTC timestamp as ISO8601 string
    virtual std::string now() = 0;
};

// serializer.h
class ISerializer {
public:
    virtual ~ISerializer() = default;
    
    // Precondition: data is valid
    // Postcondition: returns JSON string
    virtual std::string serialize(const std::vector<IDatabase::Event>& events) = 0;
};
```

#### Step 2: Implement Real Versions

```cpp
// real_database.h
class RealDatabase : public IDatabase {
private:
    std::string connection_string_;
    
public:
    explicit RealDatabase(std::string_view conn) : connection_string_(conn) {}
    
    std::vector<Event> fetch_events(std::string_view user_id) override {
        // In a real program, this would use a database driver.
        // For now, simulate.
        if (user_id == "user123") {
            return {
                Event{"ev1", "login", "2024-05-09T10:00:00Z"},
                Event{"ev2", "api_call", "2024-05-09T10:30:00Z"},
            };
        }
        return {};
    }
};

// system_clock.h
class SystemClock : public IClock {
public:
    std::string now() override {
        auto t = std::time(nullptr);
        auto tm = *std::gmtime(&t);
        char buf[30];
        std::strftime(buf, sizeof(buf), "%Y-%m-%dT%H:%M:%SZ", &tm);
        return std::string(buf);
    }
};

// json_serializer.h
class JsonSerializer : public ISerializer {
public:
    std::string serialize(const std::vector<IDatabase::Event>& events) override {
        std::string result = "[";
        for (size_t i = 0; i < events.size(); ++i) {
            if (i > 0) result += ",";
            result += R"({"id":")" + events[i].id + R"(","type":")" 
                    + events[i].type + R"(","timestamp":")" 
                    + events[i].timestamp + R"("})";
        }
        result += "]";
        return result;
    }
};
```

#### Step 3: The Service (Using Injected Dependencies)

```cpp
// report_service.h
class ReportService {
private:
    IDatabase& db_;
    IClock& clock_;
    ISerializer& serializer_;
    
public:
    ReportService(IDatabase& db, IClock& clock, ISerializer& serializer)
        : db_(db), clock_(clock), serializer_(serializer) {}
    
    // Precondition: user_id is non-empty
    // Postcondition: returns a JSON report or an error message
    std::string generate_daily_report(std::string_view user_id) {
        auto events = db_.fetch_events(user_id);
        auto report_json = serializer_.serialize(events);
        auto timestamp = clock_.now();
        
        return R"({"user_id":")" + std::string(user_id) 
             + R"(","generated_at":")" + timestamp 
             + R"(","events":)" + report_json + "}";
    }
};
```

#### Step 4: Wire Everything in main()

```cpp
int main() {
    // Create all dependencies
    RealDatabase db("postgresql://localhost:5432/app");
    SystemClock clock;
    JsonSerializer serializer;
    
    // Inject them into the service
    ReportService service(db, clock, serializer);
    
    // Use the service
    auto report = service.generate_daily_report("user123");
    std::cout << report << "\n";
    
    return 0;
}
```

This is the entire pattern. No framework, no container, no magic.

#### Step 5: Testing

```cpp
// test_report_service.cpp
class MockDatabase : public IDatabase {
public:
    std::vector<Event> fetch_events(std::string_view user_id) override {
        if (user_id == "user123") {
            return {
                Event{"ev1", "login", "2024-05-09T10:00:00Z"},
            };
        }
        return {};
    }
};

class MockClock : public IClock {
public:
    std::string now() override {
        return "2024-05-09T14:30:00Z";
    }
};

class MockSerializer : public ISerializer {
public:
    std::string serialize(const std::vector<IDatabase::Event>& events) override {
        return "[\"mock\"]";
    }
};

TEST(ReportServiceTest, GeneratesReportForExistingUser) {
    MockDatabase db;
    MockClock clock;
    MockSerializer serializer;
    
    ReportService service(db, clock, serializer);
    auto report = service.generate_daily_report("user123");
    
    EXPECT_THAT(report, ContainsSubstring("user123"));
    EXPECT_THAT(report, ContainsSubstring("2024-05-09T14:30:00Z"));
    EXPECT_THAT(report, ContainsSubstring("[\"mock\"]"));
}

TEST(ReportServiceTest, HandlesNonexistentUser) {
    MockDatabase db;
    MockClock clock;
    MockSerializer serializer;
    
    ReportService service(db, clock, serializer);
    auto report = service.generate_daily_report("nonexistent");
    
    // The service should still generate a report, but with empty events
    EXPECT_THAT(report, ContainsSubstring("nonexistent"));
    EXPECT_THAT(report, ContainsSubstring("[]"));
}
```

That's manual DI. The service is testable, the dependencies are visible, the reasoning is local.

---

## 10.5 When You Want a Container

Manual DI works well for small systems. As a system grows, the object graph becomes large and the wiring in `main()` becomes tedious.

Imagine a system with 100 classes, where each depends on 3–5 others. Wiring them manually means hand-coding a dependency graph resolver. This is busywork that a container can automate.

A **DI container** is a program that:

1. **Inspects** your classes to discover their dependencies (via constructor parameters, annotations, or configuration).
2. **Resolves** the dependency graph automatically.
3. **Instantiates** objects in the right order.
4. **Manages lifetimes** (singleton, transient, scoped).

#### Example: Wiring 10 Classes Without a Container

```cpp
// main.cpp — no container
int main() {
    // Create the graph by hand
    RealDatabase db("postgresql://localhost:5432/app");
    SystemClock clock;
    JsonSerializer serializer;
    ReportService reports(db, clock, serializer);
    
    UserRepository user_repo(db);
    AuthService auth(user_repo);
    
    NotificationService notifications(db);
    Logger logger;
    
    ReportController controller(reports, auth, notifications, &logger);
    
    AnalyticsService analytics(db, &logger);
    MetricsCollector metrics(analytics);
    
    HttpServer server(controller, metrics, &logger);
    server.listen(8080);
    
    return 0;
}
```

This works, but it is fragile:
- If you change a constructor signature, you must update the wiring.
- If you add a class, you must find the right place to instantiate it.
- If you forget to instantiate something, you get a runtime error (or worse, a null pointer).

#### Example: The Same Wiring With a Container (Pseudocode)

Most languages have a container framework. C++ has options (Boost.DI, Hypodermic), but they are less prevalent than in other languages. For illustration, here is how it might look:

```cpp
// container.cpp — hypothetical container
auto builder = ContainerBuilder();

builder.register<IDatabase>(RealDatabase::create("postgresql://localhost:5432/app"));
builder.register<IClock>(SystemClock);
builder.register<ISerializer>(JsonSerializer);
builder.register<ReportService>();  // auto-resolved

builder.register<UserRepository>();
builder.register<AuthService>();

builder.register<NotificationService>();
builder.register<Logger>();

builder.register<ReportController>();
builder.register<AnalyticsService>();
builder.register<MetricsCollector>();
builder.register<HttpServer>();

auto container = builder.build();
auto server = container.resolve<HttpServer>();
server->listen(8080);
```

The container reads the constructor signatures of each class and automatically wires them. You declare what you have, and the container figures out the graph.

#### The Cost

DI containers trade two things:

1. **Startup time**: the container must inspect and resolve the graph. This can add milliseconds to startup (significant for serverless, CLI tools, or hot-reload in tests).
2. **Static analysis**: the compiler can't verify that the graph is complete. A typo in a class name or a missing registration might not be caught until runtime.

For large systems, these costs are worth paying. For small systems or systems with tight startup requirements, manual DI is often cleaner.

---

## 10.6 Service Locator: The Anti-Pattern

Before we dismiss service locators, let's see how they look:

```cpp
class ServiceLocator {
private:
    static std::map<std::string, void*> registry;
    
public:
    static void register_service(const std::string& name, void* service) {
        registry[name] = service;
    }
    
    template<typename T>
    static T* get_service(const std::string& name) {
        return static_cast<T*>(registry[name]);
    }
};

// Usage:
class ReportService {
private:
    IDatabase* db_;
    IClock* clock_;
    
public:
    ReportService() {
        // Pull dependencies from the locator
        db_ = ServiceLocator::get_service<IDatabase>("database");
        clock_ = ServiceLocator::get_service<IClock>("clock");
    }
};

// In main():
RealDatabase db("postgresql://localhost:5432/app");
SystemClock clock;
ServiceLocator::register_service("database", &db);
ServiceLocator::register_service("clock", &clock);

ReportService service;
```

This *looks like* DI: dependencies are not created internally. But it is the **opposite** of DI. Here's why:

1. **Dependencies are hidden**: looking at the `ReportService` constructor, you have no idea what it needs. You must read the constructor body to find the `ServiceLocator` calls.
2. **Global state**: the service locator is a global registry. All code depends on it. If you change the registry, you affect everything.
3. **Hard to test**: to test `ReportService`, you must set up the service locator, register mocks, then instantiate the service. The test is coupled to the locator's API.
4. **Runtime failures**: if a dependency is not registered, the service constructor silently creates a broken object. The error happens later, when the dependency is used.

**Rule: Never use a service locator when you can use constructor injection.** If you find yourself reaching for a service locator, reconsider your design.

A service locator is sometimes justified for frameworks (a web framework needs to be flexible), but for application code, it is a code smell.

---

## 10.7 How Popular Frameworks Do It

Other languages have more sophisticated DI tooling. Let's look at how they work, so you understand the concepts when you encounter them.

### Java (Spring Framework)

Spring is the de facto DI container for Java. It uses **annotations** and **reflection**:

```java
@Service
public class ReportService {
    private final IDatabase db;
    private final IClock clock;
    
    @Autowired
    public ReportService(IDatabase db, IClock clock) {
        this.db = db;
        this.clock = clock;
    }
}

@Configuration
public class AppConfig {
    @Bean
    public IDatabase database() {
        return new RealDatabase("postgresql://localhost:5432/app");
    }
    
    @Bean
    public IClock clock() {
        return new SystemClock();
    }
}
```

Spring scans the classpath at startup, finds classes annotated with `@Service`, inspects their constructors, and auto-wires them. If a constructor parameter is of type `IDatabase`, Spring looks for a bean of that type and injects it.

**Pros:** Little boilerplate; powerful.
**Cons:** Reflection is slow at startup; harder to debug; magic.

### C# / .NET DI (Built-in)

Since .NET Core, the DI container is built into the framework:

```csharp
var services = new ServiceCollection();
services.AddScoped<IDatabase>(sp => new RealDatabase("postgresql://localhost:5432/app"));
services.AddScoped<IClock, SystemClock>();
services.AddScoped<ReportService>();

var serviceProvider = services.BuildServiceProvider();
var service = serviceProvider.GetRequiredService<ReportService>();
```

This is explicit (you register each service) but readable. The container uses reflection to resolve types.

**Pros:** Fast; built-in; conventions over configuration.
**Cons:** Still uses reflection; verbose registrations for large systems.

### PHP (Laravel Container)

Laravel uses **auto-resolution** based on constructor type hints:

```php
class ReportService {
    public function __construct(IDatabase $db, IClock $clock) {
        $this->db = $db;
        $this->clock = $clock;
    }
}

// In a service provider:
$this->app->bind(IDatabase::class, RealDatabase::class);
$this->app->singleton(IClock::class, SystemClock::class);

// Later, anywhere in the app:
$service = app(ReportService::class);  // Container auto-resolves
```

Type hints tell the container what to inject. If a dependency needs a specific configuration (like a database connection string), you bind it explicitly.

**Pros:** Clean; minimal boilerplate.
**Cons:** Magic with type hints; slow if introspection is naive.

### C++ (Limited Options)

C++ has no dominant DI framework. Options include:

- **Boost.DI**: compile-time DI via templates. Zero runtime overhead, but complex metaprogramming.
- **Hypodermic**: runtime DI container with reflection-like macros.
- **Manual wiring**: write the graph in `main()`.

Most C++ codebases use manual wiring because:
1. Reflection is not in the C++ standard (though reflection is being proposed).
2. Startup time matters in systems code.
3. C++ encourages static typing and compile-time safety.

---

## 10.8 Worked Example: A Complete Application

Let's build a small service: a task queue processor. It reads tasks from a queue, processes them, and logs results.

#### Interfaces

```cpp
// queue.h
class ITaskQueue {
public:
    virtual ~ITaskQueue() = default;
    
    struct Task {
        std::string id;
        std::string type;
        std::string payload;
    };
    
    virtual std::optional<Task> dequeue() = 0;
    virtual void acknowledge(const std::string& task_id) = 0;
};

// processor.h
class ITaskProcessor {
public:
    virtual ~ITaskProcessor() = default;
    
    // Precondition: task is valid
    // Postcondition: returns true if processed successfully, false otherwise
    virtual bool process(const ITaskQueue::Task& task) = 0;
};

// logger.h
class ILogger {
public:
    virtual ~ILogger() = default;
    
    enum Level { Debug, Info, Warn, Error };
    virtual void log(Level level, std::string_view message) = 0;
};
```

#### Implementations

```cpp
// in_memory_queue.h
class InMemoryQueue : public ITaskQueue {
private:
    std::queue<Task> tasks_;
    
public:
    void enqueue(const Task& t) {
        tasks_.push(t);
    }
    
    std::optional<Task> dequeue() override {
        if (tasks_.empty()) return std::nullopt;
        auto task = tasks_.front();
        tasks_.pop();
        return task;
    }
    
    void acknowledge(const std::string& task_id) override {
        // In-memory queue doesn't need to do anything
    }
};

// default_processor.h
class DefaultProcessor : public ITaskProcessor {
private:
    ILogger& logger_;
    
public:
    explicit DefaultProcessor(ILogger& logger) : logger_(logger) {}
    
    bool process(const ITaskQueue::Task& task) override {
        logger_.log(ILogger::Info, "Processing task: " + task.id);
        // Simulate some work
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        logger_.log(ILogger::Info, "Completed task: " + task.id);
        return true;
    }
};

// console_logger.h
class ConsoleLogger : public ILogger {
public:
    void log(Level level, std::string_view message) override {
        const char* level_str[] = {"[DEBUG]", "[INFO]", "[WARN]", "[ERROR]"};
        std::cout << level_str[level] << " " << message << "\n";
    }
};
```

#### The Worker Service

```cpp
// worker.h
class Worker {
private:
    ITaskQueue& queue_;
    ITaskProcessor& processor_;
    ILogger& logger_;
    std::atomic<bool> running_{false};
    
public:
    Worker(ITaskQueue& queue, ITaskProcessor& processor, ILogger& logger)
        : queue_(queue), processor_(processor), logger_(logger) {}
    
    void start() {
        running_ = true;
        logger_.log(ILogger::Info, "Worker starting");
        
        while (running_) {
            auto task = queue_.dequeue();
            if (!task) {
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
                continue;
            }
            
            try {
                bool success = processor_.process(*task);
                if (success) {
                    queue_.acknowledge(task->id);
                    logger_.log(ILogger::Info, "Task acknowledged: " + task->id);
                } else {
                    logger_.log(ILogger::Warn, "Task failed: " + task->id);
                }
            } catch (const std::exception& e) {
                logger_.log(ILogger::Error, std::string("Exception: ") + e.what());
            }
        }
    }
    
    void stop() {
        running_ = false;
    }
};
```

#### Main: Manual DI

```cpp
int main() {
    // Create dependencies
    InMemoryQueue queue;
    ConsoleLogger logger;
    DefaultProcessor processor(logger);
    Worker worker(queue, processor, logger);
    
    // Pre-load some tasks
    queue.enqueue({
        "task-1",
        "send-email",
        R"({"to":"user@example.com","subject":"Hello"})"
    });
    queue.enqueue({
        "task-2",
        "generate-report",
        R"({"user_id":"user123"})"
    });
    
    // Run in a background thread
    auto worker_thread = std::thread([&worker]() {
        worker.start();
    });
    
    // Let it run for a bit
    std::this_thread::sleep_for(std::chrono::seconds(1));
    worker.stop();
    
    worker_thread.join();
    logger.log(ILogger::Info, "Worker stopped");
    
    return 0;
}
```

#### Testing

```cpp
class MockQueue : public ITaskQueue {
private:
    std::vector<Task> tasks_;
    
public:
    MockQueue(const std::vector<Task>& tasks) : tasks_(tasks) {}
    
    std::optional<Task> dequeue() override {
        if (tasks_.empty()) return std::nullopt;
        auto task = tasks_.back();
        tasks_.pop_back();
        return task;
    }
    
    void acknowledge(const std::string& task_id) override {}
};

class MockProcessor : public ITaskProcessor {
public:
    std::vector<std::string> processed_ids;
    
    bool process(const ITaskQueue::Task& task) override {
        processed_ids.push_back(task.id);
        return true;
    }
};

class MockLogger : public ILogger {
public:
    std::vector<std::string> messages;
    
    void log(Level level, std::string_view message) override {
        messages.push_back(std::string(message));
    }
};

TEST(WorkerTest, ProcessesTasks) {
    std::vector<ITaskQueue::Task> tasks = {
        {"task-1", "type1", "payload1"},
        {"task-2", "type2", "payload2"},
    };
    
    MockQueue queue(tasks);
    MockLogger logger;
    MockProcessor processor;
    
    Worker worker(queue, processor, logger);
    
    // Process one task manually
    if (auto task = queue.dequeue()) {
        processor.process(*task);
    }
    
    EXPECT_EQ(processor.processed_ids.size(), 1);
    EXPECT_EQ(processor.processed_ids[0], "task-2");
}
```

This is a complete application with DI. The worker is testable, dependencies are explicit, and the main function is simple.

---

## 10.9 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **No DI (statics/singletons)** | Simple code; no boilerplate | Hard to test; global state; hidden dependencies | Small scripts, one-off programs |
| **Manual constructor DI** | Explicit; testable; no framework | Verbose wiring in main(); must manage graph | Medium systems, performance-critical systems |
| **DI container** | Automatic resolution; less boilerplate; scales to large systems | Startup overhead; harder to debug; magic | Large systems, many classes, frequent changes |
| **Service locator** | Looks flexible | Hidden dependencies; hard to test; global state; runtime failures | **Don't use this** |

---

## 10.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "DI requires a framework." | No. DI is passing parameters. Frameworks automate the wiring, but the pattern is pure discipline. |
| "DI is only for testing." | Testing is a major benefit, but DI also improves code clarity and modularity. |
| "A container makes code more testable." | A container makes wiring less tedious, but testability comes from DI, not from the container. |
| "If I use a container, I don't need to understand the dependency graph." | A poorly designed graph is still poorly designed. A container doesn't fix bad architecture. |
| "Service locators are a form of DI." | No. Locators hide dependencies and reverse the dependency direction (everything depends on the locator). |
| "DI means I can't use singletons." | Singletons can be injected. The difference is that you pass them in, not that they don't exist. |
| "Interfaces are required for DI." | No. You can inject concrete classes. Interfaces are useful for testing, but not strictly required. |

---

## 10.11 Exercises

1. **Refactor for testability.** Take a class from a codebase you know that creates its own dependencies (a database, a logger, an external API client). Refactor it to use constructor injection. Write a test that passes a mock. Did your test become simpler?

2. **Design a small DI container.** Write a function template that can resolve a simple dependency graph. It should:
   - Take a template parameter T.
   - Inspect T's constructor.
   - Resolve all parameters of that constructor.
   - Return an instance of T.
   
   (Hint: This is hard in C++. See why containers are frameworks, not libraries.)

3. **Spot the service locator.** Find an instance of a service locator pattern in a codebase you know (a global registry, a static method that returns instances, a factory that hides where things come from). Rewrite it using constructor injection.

4. **Compare frameworks.** Read the Spring (Java) and .NET DI container documentation. What problems are they solving? What tradeoffs do they make? How would you implement the same features in C++?

5. **Manual wiring at scale.** Write a `main()` that manually wires 10 classes with interdependencies (A depends on B and C, B depends on D, etc.). Count the lines. Now imagine 100 classes. At what point would a container become worthwhile for you?

6. **Lifetime management.** Some dependencies are singletons (one instance for the whole program), some are transient (new instance each time), some are scoped (one per request). Write three versions of a ReportService: one with each lifetime. When would you use each?

---

## 10.12 Summary

Dependency Injection is passing a class's dependencies as constructor (or method) parameters instead of having it create them. This simple discipline makes code testable, modular, and reasoned about locally.

In small systems, manual wiring in `main()` is clear and straightforward. As systems grow, a DI container (or a careful architecture) automates the wiring. The pattern is the same; the scale differs.

Service locators look like DI but are not—they hide dependencies and create global state. Avoid them.

Most languages have sophisticated DI tooling because the pattern is powerful. C++ has less, partly because reflection is not standard and partly because C++ values startup time and static analysis. Use manual DI, or reach for a library like Boost.DI if you need automation.

The key insight: **DI is a discipline, not a framework. A framework is optional; the discipline is not.**

---

> **[← Previous: Chapter 9 — Why Abstractions Exist](09-why-abstractions-exist.md)** · **[↑ Part 3](README.md)** · **[Next: Part 4 — Architecture Thinking](../part-04-architecture-thinking/README.md)** *(coming soon)*
