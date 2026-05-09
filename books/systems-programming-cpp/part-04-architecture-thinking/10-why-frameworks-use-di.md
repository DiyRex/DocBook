# Chapter 10 — Why Frameworks Use DI

Spring, Laravel, .NET, NestJS, Angular—every modern framework has a DI container at its core. This is not fashion. It is the answer to a specific problem: **how do you let users of your framework declare what they need without the framework knowing those requirements in advance?**

In the previous chapter, we saw that dependency injection is simple: pass dependencies as parameters. In a system with 100 classes, manual wiring becomes impossible. A container solves that, but the solution introduces tradeoffs that every architect must understand.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Articulate the wiring problem that frameworks face and why it requires automation.
2. Explain what a DI container actually does: registration, resolution, and lifetime management.
3. Recognize auto-wiring patterns in real frameworks and understand the hidden costs.
4. Design or select a container based on your system's constraints (startup time, compilation speed, debuggability).
5. Identify when a framework's DI design is a good fit and when it adds more complexity than it removes.
6. Understand the distinction between a container solving a *real* problem and a container creating ceremony.

---

## 10.1 The Wiring Problem at Scale

Suppose you are designing a web framework. Your users will write services, controllers, repositories, middleware, background jobs. Each of these has dependencies. The framework has no way to know in advance what those dependencies are—they are application code the framework has never seen.

Without automation, users must manually wire their entire dependency graph. Here is what that looks like:

```cpp
// Framework user's main.cpp — 10 services
int main() {
    RealDatabase db("postgresql://localhost");
    RealCache cache("redis://localhost");
    UserRepository user_repo(db, cache);
    AuthService auth(user_repo);
    NotificationService notif(db);
    ReportService reports(db, cache);
    
    EmailProvider email;
    SMSProvider sms;
    AlarmService alarms(email, sms, notif);
    
    MetricsCollector metrics;
    Logger logger;
    
    UserController controller(auth, user_repo, &logger);
    ReportController report_ctrl(reports, &logger);
    AlarmController alarm_ctrl(alarms, &logger);
    
    AnalyticsService analytics(metrics, user_repo);
    BatchProcessor batch(analytics, alarms, user_repo);
    
    HttpServer server;
    server.register_controller(controller);
    server.register_controller(report_ctrl);
    server.register_controller(alarm_ctrl);
    server.register_background_service(batch);
    
    server.listen(8080);
    return 0;
}
```

This works for 10 services. Now multiply by 10. With 100 services, this becomes:

- **Hundreds of lines of wiring code** (often split across multiple files, making the graph opaque)
- **Fragile**: change a single constructor signature, and the wiring breaks everywhere that class is instantiated
- **Error-prone**: if you forget to instantiate a dependency, you get a null pointer at runtime, not a compile error
- **Duplication**: if two features both need an `AuthService`, it appears twice in the wiring code (or you create a global variable to avoid duplication, which is worse)
- **Operational**: adding a new service requires finding the right place in `main()` to create it, which is not obvious in a large file

**The core problem:** the framework cannot automate this because the framework does not own the application code. The user owns it. Yet the user should not have to maintain hundreds of lines of wiring boilerplate.

The DI container solves this by **letting the user declare what they have** and **the framework figuring out how to wire it**. The user writes code that registers classes, the container inspects their constructors, and the container builds the graph automatically.

---

## 10.2 What a Container Actually Does

A DI container is not magic—it is a program that does four concrete things:

### 1. Registration

The user tells the container what classes exist and how to build them:

```python
# Pseudocode: Laravel-like registration
container.bind('database', lambda: RealDatabase('postgresql://localhost'))
container.singleton('cache', RedisCache)  # one instance for the whole app
container.transient('user_repo', UserRepository)  # new instance each time
container.bind('auth', AuthService)
container.bind('notif', NotificationService)
```

Or in a more declarative style (Spring, .NET, NestJS):

```java
@Configuration
public class AppConfig {
    @Bean
    public IDatabase database() {
        return new RealDatabase("postgresql://localhost");
    }
    
    @Bean
    @Scope("singleton")
    public RedisCache cache() {
        return new RedisCache();
    }
}
```

The container stores these registrations in a map: key = type, value = factory function or class.

### 2. Inspection (Reflection)

The container inspects each registered class's constructor to discover its dependencies:

```python
# Pseudocode: container inspects UserRepository
ctor_params = inspect(UserRepository.__init__)
# Result: [IDatabase, ICache]
```

Most modern languages have reflection APIs for this. C++ lacks standard reflection, which is one reason C++ has fewer DI frameworks.

### 3. Resolution (Graph Walking)

When you ask the container for an instance of `UserRepository`, the container:

1. Looks up the registration for `UserRepository`
2. Inspects its constructor and sees it needs `IDatabase` and `ICache`
3. Recursively resolves those dependencies
4. Instantiates `IDatabase` (via the factory)
5. Instantiates `ICache` (via the factory)
6. Instantiates `UserRepository` with those resolved dependencies
7. Returns the graph

This is a recursive depth-first walk. If there is a cycle (A depends on B, B depends on A), the container should detect it and throw an error early.

### 4. Lifetime Management

The container manages how long instances live. Most frameworks support three scopes:

- **Singleton**: one instance for the entire application. Useful for stateless services (database connections, cache clients, repositories).
- **Transient**: a new instance each time it is requested. Useful for stateful objects that must not be shared (request handlers, form objects).
- **Request-scoped**: one instance per HTTP request. Useful for request-local state (current user, request ID, transaction). After the request, the instance is discarded.

```python
# Pseudocode
container.singleton('database', RealDatabase)  # one instance, shared
container.transient('request_handler', RequestHandler)  # new each time
container.request_scoped('current_user', CurrentUserProvider)  # one per HTTP request
```

The container stores singleton instances in a map and reuses them. For transient, it creates a new instance each time. For request-scoped, it ties instances to the request context (usually via thread-local storage or a map keyed by request ID).

---

## 10.3 Auto-Wiring: The Leap Forward

Manual registration is better than manual wiring, but it is still tedious. Most frameworks go further: **auto-wiring**. The framework automatically wires dependencies based on type hints alone.

Here is how it works:

```python
# User's code — no explicit registration
class UserRepository:
    def __init__(self, db: IDatabase, cache: ICache):
        self.db = db
        self.cache = cache

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

class UserController:
    def __init__(self, service: UserService):
        self.service = service
```

The framework scans the application, finds these classes, and notices they are injectable (they are classes with type hints). The framework registers them automatically:

```python
# Framework does this internally
for cls in scan_application():
    if has_constructor_params(cls):
        container.register(cls)  # auto-register
```

Later, when a request arrives, the framework constructs `UserController`. It inspects the constructor, sees it needs `UserService`, recursively resolves `UserService` (which needs `UserRepository`), resolves `UserRepository` (which needs `IDatabase` and `ICache`), and wires the whole graph.

**The magic:** the user writes code that looks like normal classes. The framework handles the wiring. No explicit registration, no ceremony.

**The cost:** the magic is hidden. If auto-wiring fails, the error message is often cryptic. The framework resolves by *type*, not by *name*, so two implementations of the same interface require explicit registration or annotations to disambiguate.

---

## 10.4 Auto-Wiring Pitfalls: Ambiguity and Scope

### Ambiguity: Multiple Implementations

Suppose two classes implement `IDatabase`:

```python
class PostgresDatabase(IDatabase):
    pass

class MySQLDatabase(IDatabase):
    pass
```

The container sees `IDatabase` and has two choices. Which one do you want? Auto-wiring breaks.

Frameworks solve this with **annotations** (Spring's `@Qualifier`, .NET's `[ServiceName]`):

```java
@Service
public class UserRepository {
    private final IDatabase db;
    
    @Autowired
    public UserRepository(@Qualifier("postgres") IDatabase db) {
        this.db = db;
    }
}

@Configuration
public class Config {
    @Bean("postgres")
    public IDatabase postgresDb() {
        return new PostgresDatabase();
    }
}
```

Annotations are metadata. They tell the container "when you see a type hint for `IDatabase`, use the bean named 'postgres'." This is more boilerplate than pure auto-wiring, but it remains less than manual wiring.

### Scope Mixing: A Dangerous Bug

Consider this scenario:

```python
# Bad: mixing scopes
class RequestContext:
    """Stores the current user. One per request."""
    def __init__(self):
        self.user = None

class UserRepository:
    """Stateless service. One instance for the app."""
    def __init__(self, context: RequestContext):
        self.context = context
```

A singleton `UserRepository` holds a reference to a request-scoped `RequestContext`. This works for the first request: `context.user` is set. On the second request, a new `RequestContext` is created, but `UserRepository` still holds the old one. `context.user` is stale.

```python
# Sequence
Request 1: UserRepository gets RequestContext(user=alice)
Request 2: New RequestContext(user=bob) created, but UserRepository still has the old one
Result: UserRepository sees user=alice when it should see user=bob. Silent data leak.
```

This is a common bug in DI frameworks. The fix is to have the singleton depend on a *provider* of the request-scoped object, not the object itself:

```python
# Correct: use a provider
class UserRepository:
    def __init__(self, context_provider: Callable[[], RequestContext]):
        self.context_provider = context_provider
    
    def get_current_user(self):
        return self.context_provider().user  # always gets the current request's context
```

Or, the framework can inject a proxy that routes to the correct instance based on the current request context. Either way, the user must understand scopes to avoid this trap.

---

## 10.5 Modules, Providers, Configurations

Frameworks organize registration in different ways. Each is trying to solve the same problem: grouping related registrations and making the dependency graph readable.

### Spring (Java): `@Configuration` and `@Bean`

```java
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        return new PostgresDataSource("postgresql://localhost");
    }
    
    @Bean
    public UserRepository userRepository(DataSource ds) {
        return new UserRepository(ds);
    }
}

@Configuration
public class ServiceConfig {
    @Bean
    public UserService userService(UserRepository repo) {
        return new UserService(repo);
    }
}
```

A `@Configuration` class is a collection of `@Bean` methods. Each method returns an instance or builds one. Methods can depend on other beans (Spring auto-wires the parameters). This is explicit and readable.

### Laravel (PHP): Service Providers

```php
class AppServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->singleton(IDatabase::class, function ($app) {
            return new PostgresDatabase('postgresql://localhost');
        });
        
        $this->app->bind(UserRepository::class, function ($app) {
            return new UserRepository($app->make(IDatabase::class));
        });
    }
}
```

A `ServiceProvider` is a class with `register()` and `boot()` methods. The framework calls these during bootstrap. You can bind services manually (shown above) or rely on auto-wiring.

### .NET Core: `IServiceCollection`

```csharp
var services = new ServiceCollection();

// Register with different lifetimes
services.AddSingleton<IDatabase, PostgresDatabase>();
services.AddScoped<IUserRepository, UserRepository>();
services.AddTransient<IUserService, UserService>();

// Or with a factory
services.AddSingleton<ICache>(sp => new RedisCache("redis://localhost"));

var provider = services.BuildServiceProvider();
var service = provider.GetRequiredService<IUserService>();
```

Fluent API. Reads like a declarative list of what the app needs. The container uses reflection to resolve the graph.

### NestJS (TypeScript): Modules

```typescript
@Injectable()
export class UserRepository {
  constructor(private db: Database) {}
}

@Module({
  providers: [Database, UserRepository, UserService],
  exports: [UserService],
})
export class UserModule {}

@Module({
  imports: [UserModule],
  controllers: [UserController],
})
export class AppModule {}
```

NestJS uses TypeScript decorators and a module system. `@Injectable()` marks a class as injectable. The module declares what it provides and what it exports. NestJS auto-wires based on constructor parameters. This is higher-level: the framework manages both dependency wiring *and* module organization.

### Common Theme

All four frameworks do the same thing: let the user declare what they have, then the framework resolves the graph. The syntax differs, but the concept is identical.

---

## 10.6 The Costs of Container-Based DI

Containers are powerful, but they are not free. Here are the real costs:

### Cost 1: Startup Time

The container must scan the application, inspect classes, and resolve the graph. This happens at startup. For a small app (10 services), it is microseconds. For a large app (1,000 services), it is milliseconds.

Startup time matters for:
- **CLI tools**: a command that takes 500ms to run, of which 400ms is DI setup, is unusable.
- **Lambda / serverless**: cold start latency is paid by the user. 500ms of DI overhead per request is a cost.
- **Tests**: if each test instantiates the container, slow startup delays test suites.

Compiled, compile-time DI (like Boost.DI in C++) can eliminate this cost. Runtime DI always pays it.

### Cost 2: Compiled Errors vs. Runtime Errors

With manual wiring, the compiler checks the graph. If you forget to instantiate something, the compiler complains:

```cpp
// Compile error
UserController controller(/* forgot to pass UserService */);
```

With a DI container, the graph is discovered at runtime. If you forget to register a class, the container fails:

```python
# Runs fine, but fails at startup when container is built
container.resolve(UserService)  # Throws: "No registration for UserRepository"
```

You discover the error only when the container boots, not when you compile. For large applications, this is manageable (you boot the app once and catch errors). For rapid iteration, this is frustrating.

### Cost 3: Stack Traces With Magic

When the container resolves the graph, it creates instances through factory methods and reflection. The stack trace is noisy:

```
Error: Cannot instantiate UserRepository
  at Container.resolve()
  at Container._resolve_dependencies()
  at Container._resolve_dependencies()
  at Container._resolve_dependencies()
  at Framework.startup()
```

The actual error—"you forgot to bind IDatabase"—is buried. Modern frameworks try to improve this (Spring's error messages are quite good), but it remains harder to debug than explicit wiring.

### Cost 4: Hidden Behavior

A framework's DI container often comes with "magic." Spring scans the classpath for `@Service` annotations. Laravel auto-resolves type hints. NestJS injects decorators. This magic makes simple cases frictionless but makes complex cases harder to reason about.

Example: Spring's `@Autowired` can inject into fields:

```java
@Service
public class UserService {
    @Autowired  // Hidden dependency; not in constructor
    private ILogger logger;
}
```

This is convenient, but the dependency is not visible in the constructor signature. A reader must search for `@Autowired` fields to understand what the class needs.

---

## 10.7 When a Container Solves a Real Problem vs. When It Adds Ceremony

A useful heuristic: **does the container enable something that manual wiring cannot, or does it merely reduce boilerplate?**

### Real Problem: Wiring at Unbounded Scale

If you have 200 services with complex, transitive dependencies, manual wiring becomes a maintenance burden. Each new service requires finding the right place in the wiring code. Each refactor (renaming, moving) requires many edits. A container eliminates that friction.

**Example:** A large SaaS app with many microservices, each with dozens of services. The container is worth it.

### Real Problem: Runtime Reconfiguration

Some applications need to swap implementations at runtime:

```python
# At startup, decide which database to use
if config.database == "postgres":
    container.bind(IDatabase, PostgresDatabase)
else:
    container.bind(IDatabase, MySQLDatabase)
```

A container makes this easy. Manual wiring requires conditional code in `main()`, which is less clean.

### Ceremony Without Benefit: Small Applications

A CLI tool with 5 services? A script that runs once a day? A library that other code uses? Manual wiring is simpler:

```python
# Simple, readable, no magic
def main():
    db = Database("sqlite:///:memory:")
    service = MyService(db)
    result = service.do_work()
    print(result)
```

A container adds overhead and indirection without solving a real problem.

### Ceremony Without Benefit: Frameworks Forcing DI

Some frameworks require all code to be injectable:

```typescript
// NestJS: every service must be a class
@Injectable()
export class MyService {
  constructor(private logger: Logger) {}
}
```

If you have a simple utility function, wrapping it in a service just to make it injectable is ceremony.

---

## 10.8 Worked Example: A 30-Line Container in TypeScript

To demystify how containers work, here is a minimal implementation:

```typescript
// Simple DI container
class Container {
  private registrations = new Map<string, any>();
  private singletons = new Map<string, any>();

  // Register a type with a factory function
  register<T>(key: string, factory: (container: Container) => T) {
    this.registrations.set(key, factory);
  }

  // Register a singleton
  singleton<T>(key: string, factory: (container: Container) => T) {
    if (!this.singletons.has(key)) {
      this.singletons.set(key, factory(this));
    }
    return this;
  }

  // Resolve a registered type
  resolve<T>(key: string): T {
    // Check if it's a singleton
    if (this.singletons.has(key)) {
      return this.singletons.get(key);
    }

    // Look up the factory
    const factory = this.registrations.get(key);
    if (!factory) {
      throw new Error(`No registration for ${key}`);
    }

    // Call the factory and return the result
    return factory(this);
  }
}
```

Usage:

```typescript
class Database {
  constructor(connectionString: string) {
    this.connectionString = connectionString;
  }
}

class UserRepository {
  constructor(private db: Database) {}

  getUser(id: string) {
    // Hit db
  }
}

class UserService {
  constructor(private repo: UserRepository) {}

  getUser(id: string) {
    return this.repo.getUser(id);
  }
}

// Setup
const container = new Container();

container.singleton('db', () => new Database('postgresql://localhost'));
container.register('user_repo', (c) => new UserRepository(c.resolve('db')));
container.register('user_service', (c) => new UserService(c.resolve('user_repo')));

// Resolve
const userService = container.resolve<UserService>('user_service');
userService.getUser('123');
```

Walk through resolution:
1. `container.resolve('user_service')` is called
2. The container looks up the factory for `user_service`
3. The factory needs `UserRepository`, which calls `c.resolve('user_repo')`
4. That factory needs `Database`, which calls `c.resolve('db')`
5. `db` is a singleton, so it is created once and cached
6. The container returns the fully wired graph: `UserService -> UserRepository -> Database`

That's all a container does. Real containers add:
- Auto-wiring (reflect to discover constructor parameters)
- Lifetime management (request-scoped, etc.)
- Circular dependency detection
- Better error messages

But the core is just graph resolution.

---

## 10.9 Lifetimes in Depth

Understanding lifetimes is critical for avoiding bugs. Here are the patterns:

### Singleton

One instance for the entire application. Created once, shared everywhere.

```python
container.singleton('db', Database)
```

**Use case:** stateless services (database connections, cache clients, repositories, configuration objects).

**Danger:** if a singleton holds mutable state, that state is shared across requests. Bugs:

```python
class BadService:
    def __init__(self):
        self.cache = {}  # Shared across all requests
    
    def process(self, user_id):
        if user_id not in self.cache:
            self.cache[user_id] = fetch_expensive_data(user_id)
        return self.cache[user_id]
```

User A's data is cached and served to User B. This is a security vulnerability.

### Transient

A new instance each time. Never cached.

```python
container.transient('request_handler', RequestHandler)
```

**Use case:** stateful objects (request handlers, form objects, DTOs).

**Cost:** if many parts of the app need the same transient object, it is recreated many times. This is sometimes inefficient but is the safe default for stateful objects.

### Request-Scoped

One instance per HTTP request. Created when the request arrives, discarded when the request ends.

```python
container.request_scoped('current_user_provider', CurrentUserProvider)
```

**Use case:** request-local state (current user, request ID, request-local cache).

**Implementation:** the container ties instances to the request context. In a web framework, this is usually thread-local storage (in single-threaded frameworks) or a map keyed by request ID.

```python
# Pseudocode: request-scoped in a web framework
class Container:
    def request_scoped(self, key, factory):
        # Store a factory, not an instance
        self.request_scoped_factories[key] = factory
    
    def resolve_request_scoped(self, key):
        request_id = current_request.id
        # Check if this request already has an instance
        cache = self.request_cache.get(request_id)
        if cache and key in cache:
            return cache[key]
        # Create a new instance for this request
        instance = self.request_scoped_factories[key](self)
        self.request_cache[request_id][key] = instance
        return instance
```

---

## 10.10 When You Don't Need a Container

Containers are not required. Many real applications do fine without them:

### CLI Tools and Scripts

```python
# No container needed
def main():
    db = Database()
    service = UserService(db)
    result = service.process()
    print(result)

if __name__ == '__main__':
    main()
```

A single `main()` function is enough. Startup time matters; the container is overhead.

### Libraries

If you are building a library (not an application), do not assume the consumer has a container:

```cpp
// Bad: library couples to a specific container
class LibraryClass {
    LibraryClass() : logger_(ServiceLocator::get<ILogger>()) {}
};

// Good: library expects dependencies passed in
class LibraryClass {
    LibraryClass(ILogger& logger) : logger_(logger) {}
};
```

Libraries should be container-agnostic. Let the application (which uses the library) manage wiring.

### Small Microservices

A microservice with 10 services can use manual wiring in a single `main()` file. The wiring is explicit and fast to read:

```python
def main():
    # Create all dependencies
    db = Database()
    cache = Cache()
    auth = AuthService(db)
    user_repo = UserRepository(db, cache)
    # ... create all 10 services
    
    # Start the app
    app = start_app(auth, user_repo, ...)
```

Explicit, readable, no magic. As the service grows, you can introduce a container later.

### Performance-Critical Code

Startup time must be minimal:
- Real-time systems
- Embedded systems
- Serverless with tight latency budgets

Manual wiring is faster.

---

## 10.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Frameworks force DI." | Frameworks make DI the smoothest path. Most allow opting out; it is just less convenient. |
| "A container makes code more testable." | The container is irrelevant to testability. DI (passing dependencies in) makes code testable. The container just automates the wiring. |
| "Containers are always faster than manual wiring." | Compiled-time containers (Boost.DI, compile-time reflection) are zero-cost. Runtime containers (Spring, Laravel, .NET) add startup overhead. |
| "If I use a container, I don't need to understand my dependency graph." | Wrong. A poorly designed graph is still poorly designed. Containers don't fix architecture; they just scale the wiring. |
| "More containers are better for modularity." | False. Modularity comes from clear boundaries and explicit contracts. A container is a tool for managing those boundaries, not a substitute for good design. |
| "Containers eliminate global state." | No. A container *itself* is global state. It manages dependencies, but the container object is a singleton. |
| "Auto-wiring is always simpler than explicit registration." | Auto-wiring is simpler for common cases but is harder to debug when it breaks. Explicit registration is verbose but predictable. |

---

## 10.12 Tradeoffs Table

| Approach | Pros | Cons | Best for |
|---|---|---|---|
| **Manual wiring in main()** | Simple; explicit; fast startup; easy to debug | Boilerplate for large graphs; fragile if signatures change | Small apps, CLI tools, libraries, real-time systems |
| **Runtime DI container** | Scales to large systems; auto-wiring reduces boilerplate; runtime reconfiguration | Startup overhead; runtime errors instead of compile errors; harder to debug; indirection | Large apps, many services, rapid changes, web applications |
| **Compile-time DI (Boost.DI)** | Zero-cost abstraction; errors at compile time; explicit | Complex C++ metaprogramming; steep learning curve | C++ systems with strict performance budgets |
| **Framework-provided container** | Integrated with framework; framework idioms work seamlessly | Vendor lock-in; if the container doesn't fit your needs, you are stuck | Using the framework (Rails, Django, Spring, etc.) |

---

## 10.13 Exercises

1. **Trace the resolution.** Write a dependency graph with 5 classes where A depends on B and C, B depends on D, and C depends on D. Manually trace how a container would resolve `A()`. What is the order of instantiation?

2. **Spot scope mixing.** In a real codebase you use, find a place where a singleton depends on a request-scoped service (or vice versa). Describe the bug this could cause. How would you fix it?

3. **Compare framework registration styles.** Read the documentation for Spring `@Bean`, Laravel `ServiceProvider`, and .NET `IServiceCollection`. Write the same 3-class dependency graph in all three. What syntax differences matter? What concepts are the same?

4. **Build a container with auto-wiring.** Extend the 30-line TypeScript container to support auto-wiring based on constructor parameter names. (Hint: use JavaScript reflection to inspect constructor parameters.)

5. **Startup time audit.** In an app you know, measure the startup time spent on DI. Turn off the DI container and wire manually in `main()`. How much time did you save? Is it meaningful for your use case?

6. **Design a wiring strategy for a large system.** You are building an app with 200 services, 8 features, and a shared core. How would you organize registrations? Would you use a single container, multiple containers (per feature), or a mix? Justify.

7. **Scope design.** Write a service that must not be a singleton (it has mutable request-specific state) but is also too expensive to be transient (it holds a database connection). What scope should it have? How would you implement it in your chosen framework?

---

## 10.14 Summary

Frameworks use DI containers because they solve a real problem: at scale (100+ services), manual wiring becomes a maintenance burden. A container automates the tedious parts—graph resolution, lifetime management—while letting developers focus on business logic.

But containers are not free. They add startup overhead, push errors from compile time to runtime, and hide what is happening beneath layers of abstraction. The skill is matching the tool to the problem: use a container when manual wiring would be worse, use manual wiring when the container would add more friction than it removes.

The core insight: **containers do not make DI magical. They make it scalable.** The discipline—passing dependencies in—is what matters. The container is an engineering convenience, not a design principle.

---

> **[← Previous: Chapter 9 — Dependency Injection From Scratch](09-dependency-injection-from-scratch.md)** · **[↑ Part 4](README.md)** · **[Next: Chapter 11 — Boundaries—When To Split Logic →](11-boundaries-when-to-split.md)**
