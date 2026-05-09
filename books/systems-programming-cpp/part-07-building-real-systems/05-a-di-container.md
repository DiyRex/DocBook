# Chapter 72 — A DI Container

In Chapter 10 (Part 1), we learned that dependency injection is just passing parameters. In Chapter 10 (Part 4), we learned why frameworks add containers: because wiring 100+ classes by hand is tedious and error-prone.

This chapter builds a real C++ DI container from scratch—small enough to understand completely, large enough to be useful. We will see how type erasure handles heterogeneous factories, how lifetimes map to storage, and what pitfalls exist when real code tries to use one.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a DI container does: type registration, resolution, and lifetime management.
2. Implement a container that handles three lifetimes: singleton, transient, and scoped.
3. Use type erasure (`std::any`, `std::type_index`) to store factories of different types in a single registry.
4. Register dependencies by interface type, by factory function, or as pre-constructed instances.
5. Implement recursive resolution: when resolving a type, automatically resolve its dependencies.
6. Detect common errors: missing registrations, cycles, and scope violations.
7. Recognize when a container is the right tool and when manual wiring is simpler.

---

## 72.1 What Goes In, What Comes Out

A DI container is a registry. You tell it three things:

1. **"This interface maps to this implementation"**: `container.register<ILogger>(ConsoleLogger);`
2. **"Create instances using this factory"**: `container.registerFactory<IDatabase>([](Container& c) { return std::make_unique<PostgreSQL>(...); });`
3. **"This object already exists; always return it"**: `container.registerInstance<IConfig>(myConfig);`

When you ask for a type, the container:
1. Looks up the registration (or fails if missing).
2. For factories, recursively resolves all parameters.
3. Returns an instance respecting the lifetime (create new, return singleton, cache per scope).

Here is a mental model:

```
┌─────────────────────────────────┐
│ DI Container (Type → Factory)   │
├─────────────────────────────────┤
│ ILogger    → ConsoleLogger      │
│ IDatabase  → PostgreSQL factory │
│ IConfig    → (instance)         │
└─────────────────────────────────┘
                 │
                 │ resolve<ILogger>()
                 │ 1. Look up ILogger
                 │ 2. ConsoleLogger has no deps
                 │ 3. Create instance
                 ↓
            ConsoleLogger
             (singleton)
```

---

## 72.2 Lifetimes

A lifetime policy determines *when* an instance is created and *how many* exist at once. Three patterns cover most cases.

Understanding lifetimes is crucial because choosing the wrong lifetime is one of the most subtle bugs in systems that use containers. A singleton that captures a scoped dependency, for example, silently violates the scoped lifetime and can leak data between requests.

### Singleton
One instance, forever, shared by everyone.

```cpp
container.registerSingleton<ILogger>(ConsoleLogger);
```

The container creates the instance on first resolve. Subsequent requests return the same instance.

Use for: stateless utilities (logging, configuration, time providers), shared resources (database connections, thread pools).

**Pitfall**: If a singleton holds a reference to a scoped or transient dependency, it becomes a "captive dependency"—the scoped lifetime is violated.

```cpp
// WRONG: LogService is singleton but depends on RequestContext (scoped)
class LogService {
    RequestContext& ctx_;  // Scoped; should not be in singleton
public:
    LogService(RequestContext& ctx) : ctx_(ctx) {}
};
```

### Transient
A new instance every time.

```cpp
container.registerTransient<IRequestHandler>(RequestHandler);
```

Every call to `resolve<IRequestHandler>()` returns a fresh instance. No caching.

Use for: stateful objects with per-request or per-operation state.

**Cost**: Creation overhead. If your type is expensive to build, transient is wrong.

### Scoped
One instance per scope. A scope is a logical boundary (an HTTP request, a transaction, a batch job).

```cpp
container.registerScoped<IRequestContext>(RequestContext);
```

Within a scope, the first resolve creates the instance; subsequent resolves return the same instance. When the scope ends, the instance is discarded.

Use for: request-local state (current user, transaction), expensive stateful objects tied to a lifetime, database transactions that must complete atomically.

Example in a web handler:

```cpp
// HTTP request arrives; create a scope
{
    auto scope = container.createScope();
    
    // Multiple resolves return the same instance
    auto user = scope.resolve<IUserContext>();
    auto txn = scope.resolve<ITransaction>();
    
    // Handler uses the same user and transaction throughout
    handler.process(scope, user, txn);
    
    // Scope exits; user and transaction are cleaned up
}
// Next HTTP request gets a fresh scope and fresh instances
```

**Implementation**: A scope holds a cache. When you enter a scope, create an empty cache. When you resolve a scoped type, check the cache first; if missing, create and cache it.

```cpp
// Pseudocode: scope management
class Scope {
    std::unordered_map<std::type_index, std::any> cache_;
public:
    template<typename T>
    std::shared_ptr<T> resolve(Container& container) {
        auto key = std::type_index(typeid(T));
        if (cache_.count(key)) {
            return std::any_cast<std::shared_ptr<T>>(cache_[key]);
        }
        auto instance = container.resolve<T>();  // recursive
        cache_[key] = instance;
        return instance;
    }
};
```

The scope is typically created at the boundary of a logical unit of work and destroyed when that unit completes. A web framework creates a scope per request, a background job processor creates a scope per job.

### Lifetime Mapping to Storage

| Lifetime | Storage | Creation | Deletion |
|---|---|---|---|
| Singleton | `std::any` in container | On first resolve | Never (or at container destruction) |
| Transient | Stack or heap (caller owns) | On every resolve | Immediately after return |
| Scoped | `std::any` in scope cache | On first resolve per scope | When scope destroyed |

---

## 72.3 Registration API

### Type-to-Type Registration

Register a concrete type implementing an interface:

```cpp
struct ILogger {
    virtual ~ILogger() = default;
    virtual void log(std::string_view msg) = 0;
};

struct ConsoleLogger : ILogger {
    void log(std::string_view msg) override { std::cout << msg << "\n"; }
};

Container container;
container.register<ILogger, ConsoleLogger>();
```

The container infers that `ILogger` maps to `ConsoleLogger` and calls `ConsoleLogger()` (default constructor) to create instances.

### Factory Registration

For custom construction logic, register a factory:

```cpp
container.registerFactory<IDatabase>([](Container& c) {
    auto host = c.resolve<IConfig>().database_host();
    return std::make_unique<PostgreSQL>(host);
});
```

The factory is a callable: `std::function<T(Container&)>`. It receives the container to resolve other dependencies. Return a `std::unique_ptr<T>` for ownership clarity.

### Instance Registration

If you already have an instance, register it as a singleton:

```cpp
MyLogger logger("app.log");
container.registerInstance<ILogger>(logger);
```

The container will always return this exact instance.

---

## 72.4 Resolution: Walking the Graph

When you call `container.resolve<T>()`, the container must:

1. Look up T's registration.
2. Get a factory.
3. Inspect the factory's parameters and resolve them recursively.
4. Call the factory with resolved dependencies.
5. Return or cache the result based on lifetime.

Here is pseudocode:

```cpp
template<typename T>
std::shared_ptr<T> resolve() {
    auto key = std::type_index(typeid(T));
    
    // 1. Look up registration
    if (!registrations_.count(key)) {
        throw std::runtime_error("Type not registered");
    }
    
    auto& reg = registrations_[key];
    
    // 2. Check lifetime (singleton?)
    if (reg.lifetime == Lifetime::Singleton) {
        if (singletons_.count(key)) {
            return std::any_cast<std::shared_ptr<T>>(singletons_[key]);
        }
    }
    
    // 3. Call factory
    auto instance = reg.factory();
    
    // 4. Cache if singleton
    if (reg.lifetime == Lifetime::Singleton) {
        singletons_[key] = instance;
    }
    
    return instance;
}
```

### Recursive Resolution

The factory receives the container itself, so it can recursively resolve its own dependencies:

```cpp
container.registerSingletonFactory<OrderService>([](Container& c) {
    // The factory asks the container for its dependencies
    auto userRepo = c.resolve<IUserRepository>();
    auto orderRepo = c.resolve<IOrderRepository>();
    auto logger = c.resolve<ILogger>();
    
    // Construct with resolved dependencies
    return std::make_shared<OrderService>(userRepo, orderRepo, logger);
});
```

When you call `resolve<OrderService>()`, the container:
1. Looks up `OrderService`.
2. Calls its factory.
3. The factory calls `resolve<IUserRepository>()`, which recursively resolves.
4. The factory calls `resolve<IOrderRepository>()`, which recursively resolves.
5. The factory calls `resolve<ILogger>()`, which recursively resolves.
6. The factory constructs `OrderService` with all three dependencies.
7. The container caches it (if singleton) and returns it.

The hard part: **how does the factory know about dependencies?** In typed languages, reflection answers this. C++ has no standard reflection, so containers use:

- **Explicit factory** (we used above): the user writes the factory, which explicitly calls `resolve<>()` for each dependency.
- **Manual registration of factory parameters**: the user tells the container what parameters the factory needs.
- **Compile-time reflection** (Boost.DI): templates inspect constructor signatures at compile time.

We will use explicit factories. They are verbose but clear.

---

## 72.5 Type Erasure in C++

The core challenge: a container stores factories of *different* types in a single map. The map key is `std::type_index` (a runtime type identifier). The map value must hold any factory.

We use `std::any` for this:

```cpp
struct Registration {
    std::any factory;           // Holds std::function<std::shared_ptr<T>(Container&)>
    Lifetime lifetime;
};

std::unordered_map<std::type_index, Registration> registrations_;
```

When you register, you store the typed factory in `std::any`:

```cpp
template<typename T, typename Impl>
void register_() {
    auto factory = [](Container& c) -> std::shared_ptr<T> {
        return std::make_shared<Impl>();
    };
    
    auto key = std::type_index(typeid(T));
    registrations_[key] = Registration{
        factory,
        Lifetime::Singleton
    };
}
```

When you resolve, you cast back:

```cpp
template<typename T>
std::shared_ptr<T> resolve() {
    auto key = std::type_index(typeid(T));
    auto& reg = registrations_[key];
    
    auto factory_any = reg.factory;
    auto factory = std::any_cast<std::function<std::shared_ptr<T>(Container&)>>(factory_any);
    
    return factory(*this);
}
```

The cast can fail if you register the wrong way or resolve the wrong type. This is the price of type erasure: you lose compile-time type checking.

---

## 72.6 A Minimal Implementation

Here is a working C++20 container in ~120 lines:

```cpp
#include <any>
#include <functional>
#include <memory>
#include <typeindex>
#include <unordered_map>
#include <stdexcept>

enum class Lifetime {
    Singleton,
    Transient,
    Scoped
};

class Container {
public:
    // Register a concrete type
    template<typename T, typename Impl = T>
    void registerSingleton() {
        registerSingletonFactory<T>([](Container&) {
            return std::make_shared<Impl>();
        });
    }

    // Register with a custom factory
    template<typename T>
    void registerSingletonFactory(
        std::function<std::shared_ptr<T>(Container&)> factory
    ) {
        auto key = std::type_index(typeid(T));
        registrations_[key] = {
            std::any(factory),
            Lifetime::Singleton
        };
    }

    // Register transient (new instance each time)
    template<typename T, typename Impl = T>
    void registerTransient() {
        registerTransientFactory<T>([](Container&) {
            return std::make_shared<Impl>();
        });
    }

    template<typename T>
    void registerTransientFactory(
        std::function<std::shared_ptr<T>(Container&)> factory
    ) {
        auto key = std::type_index(typeid(T));
        registrations_[key] = {
            std::any(factory),
            Lifetime::Transient
        };
    }

    // Register an instance (singleton)
    template<typename T>
    void registerInstance(std::shared_ptr<T> instance) {
        auto key = std::type_index(typeid(T));
        singletons_[key] = instance;
    }

    // Resolve: get an instance
    template<typename T>
    std::shared_ptr<T> resolve() {
        auto key = std::type_index(typeid(T));

        // Check if singleton already exists
        if (singletons_.count(key)) {
            return std::any_cast<std::shared_ptr<T>>(singletons_[key]);
        }

        // Look up registration
        if (!registrations_.count(key)) {
            throw std::runtime_error(
                std::string("Type not registered: ") + key.name()
            );
        }

        auto& reg = registrations_[key];

        // Get factory and call it
        auto factory = std::any_cast<std::function<std::shared_ptr<T>(Container&)>>(
            reg.factory
        );
        auto instance = factory(*this);

        // Cache if singleton
        if (reg.lifetime == Lifetime::Singleton) {
            singletons_[key] = instance;
        }

        return instance;
    }

private:
    struct Registration {
        std::any factory;
        Lifetime lifetime;
    };

    std::unordered_map<std::type_index, Registration> registrations_;
    std::unordered_map<std::type_index, std::any> singletons_;
};
```

Usage:

```cpp
struct ILogger {
    virtual ~ILogger() = default;
    virtual void log(std::string_view msg) = 0;
};

struct ConsoleLogger : ILogger {
    void log(std::string_view msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

struct IDatabase {
    virtual ~IDatabase() = default;
    virtual std::string query(std::string_view sql) = 0;
};

struct MockDatabase : IDatabase {
    std::string query(std::string_view sql) override {
        return "mock result";
    }
};

struct UserService {
    std::shared_ptr<IDatabase> db_;
    std::shared_ptr<ILogger> logger_;

    UserService(std::shared_ptr<IDatabase> db, std::shared_ptr<ILogger> logger)
        : db_(db), logger_(logger) {}

    void process() {
        logger_->log("Starting process");
        auto result = db_->query("SELECT * FROM users");
        logger_->log(result);
    }
};

int main() {
    Container c;

    c.registerSingleton<ILogger, ConsoleLogger>();
    c.registerSingleton<IDatabase, MockDatabase>();

    // Manual factory: UserService depends on IDatabase and ILogger
    c.registerSingletonFactory<UserService>([](Container& c) {
        return std::make_shared<UserService>(
            c.resolve<IDatabase>(),
            c.resolve<ILogger>()
        );
    });

    auto service = c.resolve<UserService>();
    service->process();

    return 0;
}
```

Output:
```
[LOG] Starting process
[LOG] mock result
```

---

## 72.7 Auto-Wiring: Why C++ Doesn't Have It

Languages like Java, C#, and Python can auto-discover constructor parameters at runtime. The framework uses reflection to inspect the constructor signature, look up registrations for each parameter type, and wire them automatically.

```java
// Java with Spring (auto-wiring)
public class UserService {
    @Inject
    private IDatabase db;
    
    @Inject
    private ILogger logger;
    // Spring auto-discovers these and wires them
}
```

C++ has no standard runtime reflection. You cannot ask "what are the constructor parameters of `UserService`?" at runtime. The compiler knows, but that information is stripped from the binary.

**Option 1: Compile-time reflection (Boost.DI)**

Boost.DI uses templates and SFINAE to inspect constructors at compile time:

```cpp
auto injector = di::make_injector(
    di::bind<ILogger>().to<ConsoleLogger>(),
    di::bind<IDatabase>().to<MockDatabase>()
);
auto service = injector.create<UserService>();
```

Boost.DI inspects `UserService`'s constructor at compile time, deduces the types of its parameters, and auto-wires them. This is zero-cost (the compiler generates the exact wiring code you would write by hand) but requires C++ expertise.

**Option 2: Manual factory (what we did above)**

You write factories explicitly. Verbose, but clear and straightforward. No magic.

**Option 3: Code generation**

Generate registration code from source (like Java annotations). Requires build-system integration and source inspection.

For most C++ systems, manual factories are the right choice. Boost.DI is excellent for performance-critical code where zero-cost abstraction matters. Auto-wiring adds value mostly in large, dynamically typed systems where the boilerplate of manual registration becomes genuinely burdensome.

The C++ philosophy is explicit over implicit. Manual factories embody that: you see exactly what is wired where. This clarity is worth the boilerplate in most cases.

---

## 72.8 Scoped Resolution

To implement request-scoped instances, introduce a `Scope` object:

```cpp
class Scope {
public:
    template<typename T>
    std::shared_ptr<T> resolve(Container& container) {
        auto key = std::type_index(typeid(T));

        // Check scope cache
        if (cache_.count(key)) {
            return std::any_cast<std::shared_ptr<T>>(cache_[key]);
        }

        // Resolve and cache
        auto instance = container.resolve<T>();
        cache_[key] = instance;
        return instance;
    }

private:
    std::unordered_map<std::type_index, std::any> cache_;
};
```

Usage:

```cpp
{
    Scope scope;
    auto user1 = scope.resolve<IUserContext>(container);
    auto user2 = scope.resolve<IUserContext>(container);
    // user1 and user2 are the same instance
}
// scope destroyed; user1 and user2 invalid
```

To integrate scoped into the container, add scoped registrations and track active scopes:

```cpp
class Container {
    // ... existing code ...

    template<typename T>
    void registerScoped(std::function<std::shared_ptr<T>(Container&)> factory) {
        auto key = std::type_index(typeid(T));
        registrations_[key] = { std::any(factory), Lifetime::Scoped };
    }

    std::shared_ptr<Scope> currentScope_;

public:
    void setCurrentScope(std::shared_ptr<Scope> scope) {
        currentScope_ = scope;
    }
};
```

In a web framework, set a new scope at the start of each request, clear it at the end.

---

## 72.9 Error Detection

Real containers provide helpful error messages. The minimal container we built above detects some errors; real code should add more.

### Missing Registration

```cpp
c.resolve<IMissingType>();
// Throws: "Type not registered: 14IMissingTypeE"
```

The error message includes the mangled type name (because `std::type_index::name()` returns the mangled name on most implementations). Unmangle it with `c++filt`:

```bash
$ c++filt 14IMissingTypeE
IMissingType
```

**Better error message**: Store a human-readable type name with each registration.

```cpp
struct Registration {
    std::string type_name;           // "ILogger"
    std::any factory;
    Lifetime lifetime;
};

// In resolve():
if (!registrations_.count(key)) {
    throw std::runtime_error(
        "Type not registered: " + registrations_[key].type_name
    );
}
```

### Type Mismatch at Cast

When you call `std::any_cast<T>()` on a value stored as `U`, it throws `std::bad_any_cast`. This happens if you register a service as `ILogger` but try to resolve it as `IDatabase`.

```cpp
c.registerSingleton<ILogger, ConsoleLogger>();

try {
    auto db = c.resolve<IDatabase>();  // resolves to ILogger, cast fails
} catch (const std::bad_any_cast&) {
    std::cerr << "Type cast failed\n";
}
```

To prevent this, check the type at registration time or use a type tag:

```cpp
struct Registration {
    std::type_index registered_type;
    std::any factory;
    Lifetime lifetime;
};

// At resolution time:
if (std::type_index(typeid(T)) != registration.registered_type) {
    throw std::runtime_error("Type mismatch");
}
```

### Circular Dependencies

If A depends on B and B depends on A, the container will infinite-loop or exhaust stack. Detect it:

```cpp
// Pseudocode: detect cycle
std::unordered_set<std::type_index> resolution_stack;

template<typename T>
std::shared_ptr<T> resolve_impl(std::unordered_set<std::type_index> stack) {
    auto key = std::type_index(typeid(T));
    if (stack.count(key)) {
        throw std::runtime_error("Circular dependency: " + key.name());
    }
    
    stack.insert(key);
    
    auto key = std::type_index(typeid(T));
    auto& reg = registrations_[key];
    auto factory = std::any_cast<std::function<std::shared_ptr<T>(Container&)>>(reg.factory);
    
    // Call factory, which may recursively resolve
    return factory(*this);
}
```

The factory receives the container and calls `resolve()`. To pass the stack through, you would need to refactor the container to track it. A simpler approach: set a recursion depth limit.

```cpp
class Container {
    static constexpr int MAX_DEPTH = 32;
    int depth_ = 0;
    
    template<typename T>
    std::shared_ptr<T> resolve() {
        if (depth_ >= MAX_DEPTH) {
            throw std::runtime_error("Recursion depth exceeded (circular dependency?)");
        }
        ++depth_;
        try {
            // ... resolve logic ...
        } catch (...) {
            --depth_;
            throw;
        }
        --depth_;
        return result;
    }
};
```

This is a stopgap. Real cycle detection requires tracking the call stack.

---

## 72.10 Pitfalls and Anti-Patterns

### Captive Dependencies

A singleton holds a reference to a scoped service. The scoped lifetime is violated.

```cpp
// WRONG
class SingletonService {
    std::shared_ptr<ScopedContext> ctx_;  // Scoped, shouldn't be in singleton
public:
    SingletonService(std::shared_ptr<ScopedContext> ctx) : ctx_(ctx) {}
};
```

The scope ends, but the singleton still holds the old reference. This is a bug.

**Fix**: Inject a factory, not an instance.

```cpp
class SingletonService {
    std::function<std::shared_ptr<ScopedContext>()> ctx_factory_;
public:
    SingletonService(std::function<std::shared_ptr<ScopedContext>()> factory)
        : ctx_factory_(factory) {}
    
    void do_work() {
        auto ctx = ctx_factory_();  // Get current scope's instance
    }
};
```

### Service Locator Anti-Pattern

Avoid injecting the container itself:

```cpp
// WRONG
class Service {
    Container& container_;
public:
    Service(Container& c) : container_(c) {}
    
    void process() {
        auto logger = container_.resolve<ILogger>();
        // Hides dependencies; hard to test
    }
};
```

This is a service locator. It hides what the service actually needs. Tests cannot know what to mock. Dependencies are implicit.

**Fix**: Inject what the service needs.

```cpp
// RIGHT
class Service {
    std::shared_ptr<ILogger> logger_;
public:
    Service(std::shared_ptr<ILogger> logger) : logger_(logger) {}
    
    void process() {
        logger_->log("Processing");
        // Dependencies are explicit
    }
};
```

### Runtime-Only Errors

With type erasure, many errors happen at runtime, not compile time.

```cpp
c.registerSingleton<ILogger, ConsoleLogger>();
c.resolve<IDatabase>();  // Runtime error: not registered
```

In statically typed systems with auto-wiring, this error is caught at compile time. With a manual container, mistakes slip through.

**Mitigation**: Write tests that exercise all registrations.

```cpp
TEST(ContainerTest, AllTypesResolvable) {
    Container c = setupContainer();
    EXPECT_NO_THROW(c.resolve<ILogger>());
    EXPECT_NO_THROW(c.resolve<IDatabase>());
    EXPECT_NO_THROW(c.resolve<UserService>());
}
```

---

## 72.11 Worked Example: A Simple Web Handler

Here is a complete example: an HTTP request handler that uses the container to wire dependencies:

```cpp
#include <iostream>
#include <memory>

// Interfaces
struct ILogger {
    virtual ~ILogger() = default;
    virtual void log(std::string_view msg) = 0;
};

struct IDatabase {
    virtual ~IDatabase() = default;
    virtual std::string getUser(int id) = 0;
};

struct IAuthenticator {
    virtual ~IAuthenticator() = default;
    virtual bool isAuthorized(int user_id) = 0;
};

// Implementations
struct ConsoleLogger : ILogger {
    void log(std::string_view msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

struct MockDatabase : IDatabase {
    std::string getUser(int id) override {
        return "User" + std::to_string(id);
    }
};

struct SimpleAuthenticator : IAuthenticator {
    bool isAuthorized(int user_id) override {
        return user_id > 0;
    }
};

// Service
struct UserHandler {
    std::shared_ptr<IDatabase> db_;
    std::shared_ptr<ILogger> logger_;
    std::shared_ptr<IAuthenticator> auth_;

    UserHandler(
        std::shared_ptr<IDatabase> db,
        std::shared_ptr<ILogger> logger,
        std::shared_ptr<IAuthenticator> auth
    ) : db_(db), logger_(logger), auth_(auth) {}

    std::string handle(int user_id) {
        logger_->log("Handling request for user " + std::to_string(user_id));
        
        if (!auth_->isAuthorized(user_id)) {
            logger_->log("Authorization failed");
            return "Unauthorized";
        }
        
        auto user = db_->getUser(user_id);
        logger_->log("Retrieved: " + user);
        return user;
    }
};

// Setup container
Container setupContainer() {
    Container c;
    
    c.registerSingleton<ILogger, ConsoleLogger>();
    c.registerSingleton<IDatabase, MockDatabase>();
    c.registerSingleton<IAuthenticator, SimpleAuthenticator>();
    
    c.registerSingletonFactory<UserHandler>([](Container& c) {
        return std::make_shared<UserHandler>(
            c.resolve<IDatabase>(),
            c.resolve<ILogger>(),
            c.resolve<IAuthenticator>()
        );
    });
    
    return c;
}

// Main
int main() {
    auto container = setupContainer();
    auto handler = container.resolve<UserHandler>();
    
    std::cout << handler->handle(123) << "\n";
    std::cout << handler->handle(-1) << "\n";
    
    return 0;
}
```

Output:
```
[LOG] Handling request for user 123
[LOG] Retrieved: User123
User123
[LOG] Handling request for user -1
[LOG] Authorization failed
Unauthorized
```

---

## 72.12 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **No container; manual wiring in main()** | Explicit; debuggable; fast startup; no magic; all errors caught at link time | Boilerplate grows with app size; fragile if signatures change; tedious to refactor | Small apps, CLI tools, embedded systems, performance-critical code |
| **Custom container (like ours)** | Tailored to your needs; simple to understand; zero overhead if you don't use it; educational | Maintenance burden; errors at runtime; no auto-wiring; no type safety across registrations | Medium apps with specific lifetime requirements; teaching DI concepts |
| **Boost.DI** | Zero-cost; compile-time errors; auto-wiring; type-safe; mature; production-tested | Steep learning curve; compile-time complexity; heavy templates; slower builds | C++ systems with strict performance budgets and large dependency graphs; teams familiar with advanced C++ |
| **Framework container (NestJS, Spring, .NET)** | Integrated with framework; many conventions; community support; extensive docs | Vendor lock-in; large startup overhead; harder to debug magic; less portable | Using the framework; large web applications; rapid development over raw performance |

The decision tree is simple:
- **Fewer than 20 services?** Manual wiring is clearer.
- **20–100 services with complex lifetimes?** Custom container or Boost.DI.
- **100+ services, large team, web app?** Framework container.
- **Performance or startup time critical?** Manual wiring or Boost.DI, never runtime containers.

---

## 72.13 Common Misconceptions

| Misconception | Reality |
|---|---|
| "A container makes code testable." | A container automates wiring, but testability comes from DI (passing dependencies in). You can inject mocks without a container. |
| "Containers eliminate boilerplate." | They eliminate wiring boilerplate but add registration boilerplate. For small apps, manual wiring is simpler. |
| "If I use a container, I don't need to understand the dependency graph." | A poorly designed graph is still poorly designed. The container scales wiring, not architecture. |
| "Type erasure is always slower than typed factories." | `std::any_cast` is a runtime operation, but the actual work (calling the factory) dominates. The overhead is typically negligible. |
| "Singletons are fine; the container manages them." | A singleton is still global state. A container doesn't fix the design problems of globals. Use singletons deliberately, not as a default. |
| "A container must support auto-wiring." | No. Java and C# have runtime reflection. C++ doesn't. Manual factories are fine and offer clarity. |

---

## 72.14 Testing a Container

Once you have a container, test it like any other component. Key areas:

### Test Registration and Resolution

```cpp
TEST(ContainerTest, RegisterAndResolveSimpleType) {
    Container c;
    c.registerSingleton<ILogger, ConsoleLogger>();
    
    auto logger = c.resolve<ILogger>();
    EXPECT_NE(logger, nullptr);
}

TEST(ContainerTest, MissingTypeThrows) {
    Container c;
    
    EXPECT_THROW(c.resolve<IMissingType>(), std::runtime_error);
}
```

### Test Lifetimes

```cpp
TEST(ContainerTest, SingletonReturnsSameInstance) {
    Container c;
    c.registerSingleton<ILogger, ConsoleLogger>();
    
    auto log1 = c.resolve<ILogger>();
    auto log2 = c.resolve<ILogger>();
    
    EXPECT_EQ(log1.get(), log2.get());  // Same pointer
}

TEST(ContainerTest, TransientReturnsNewInstance) {
    Container c;
    c.registerTransient<ILogger, ConsoleLogger>();
    
    auto log1 = c.resolve<ILogger>();
    auto log2 = c.resolve<ILogger>();
    
    EXPECT_NE(log1.get(), log2.get());  // Different pointers
}
```

### Test Factories and Dependencies

```cpp
struct IRepository {
    virtual ~IRepository() = default;
    virtual std::string query() = 0;
};

struct MockRepository : IRepository {
    std::string query() override { return "mock"; }
};

struct Service {
    std::shared_ptr<IRepository> repo_;
    Service(std::shared_ptr<IRepository> repo) : repo_(repo) {}
    std::string process() { return repo_->query(); }
};

TEST(ContainerTest, ResolvesWithDependencies) {
    Container c;
    c.registerSingleton<IRepository, MockRepository>();
    c.registerSingletonFactory<Service>([](Container& c) {
        return std::make_shared<Service>(c.resolve<IRepository>());
    });
    
    auto svc = c.resolve<Service>();
    EXPECT_EQ(svc->process(), "mock");
}
```

---

## 72.15 Exercises

1. **Add cycle detection.** Extend the container to detect circular dependencies. Register A depends on B, B depends on A. Call `resolve<A>()` and verify it throws with a clear error message.

2. **Implement scoped lifetimes.** Add a `Scope` class to the container. `registerScoped<T>()` should cache instances per scope. Verify that two resolves within a scope return the same instance, and a new scope creates a new instance.

3. **Write a factory with multiple dependencies.** Register a service that depends on three other services (like the `UserHandler` example). Use a lambda factory to resolve all of them. Verify the service receives all dependencies.

4. **Test the container thoroughly.** Write unit tests that verify:
   - A registered type can be resolved.
   - An unregistered type throws.
   - A singleton is cached (same instance on multiple resolves).
   - A transient creates a new instance each time.
   - A factory receives the container and can recursively resolve.

5. **Build a test double.** Create a mock implementation of `ILogger` that records all logged messages. Register it with the container. In a test, resolve it, use it in a service, and verify the service's actions were logged correctly.

6. **Lifetime mismatch detection.** Design a scenario where a singleton incorrectly holds a scoped dependency. Implement a check in the container that detects this violation and throws an error at registration time.

7. **Compare with manual wiring.** Write two versions of `main()`: one that uses the container to wire 10 services, one that manually wires them all. Time the startup of each. How much overhead does the container add? Is it meaningful?

8. **Explore Boost.DI.** Install Boost and read the Boost.DI tutorial. Rewrite the `UserHandler` example using Boost.DI's auto-wiring. How much boilerplate does it eliminate? What did you have to learn to use it? When would you choose it over a manual container?

---

## 72.16 Summary

A DI container is a registry that automates dependency resolution. It stores factories keyed by type, handles three lifetimes (singleton, transient, scoped), and recursively resolves the dependency graph.

C++ containers use type erasure (`std::any`, `std::type_index`) because the language has no runtime reflection. This means manual factory registration and runtime type errors—tradeoffs that are acceptable for many systems. The containers we build today are simple enough to fit in a single file and understand completely.

A container is optional. Frameworks like Spring or Django use them to handle complexity at scale. For small systems, manual wiring in `main()` is often clearer and faster. The key skill is knowing when to reach for a container (50+ interdependent services) and when to stick with explicit code (anything smaller).

The lifetime tradeoff is real and subtle: a singleton is fast and shared but captures dependencies at risk of leaking across logical boundaries, a scoped service is isolated per logical boundary (perfect for request-local state), and a transient is fresh but expensive. Choosing the right lifetime for each dependency is as important as building the container itself. Getting it wrong leads to security vulnerabilities (one user's data leaked to another), dangling references, or unnecessary object creation chewing through CPU time and memory.

**The fundamental insight**: a container does not make DI magical or automatic. It makes wiring less tedious. The discipline—passing dependencies in—is what matters. The container is an engineering convenience, not a design principle. Mastering this discipline is more important than mastering any container technology. If you can manually wire a system clearly, adding a container is a refactoring convenience, not a necessity.

---

> **[← Previous: A Mini Framework](04-a-mini-framework.md)** · **[↑ Part 7](README.md)** · **[Next: An Event Bus →](06-an-event-bus.md)**
