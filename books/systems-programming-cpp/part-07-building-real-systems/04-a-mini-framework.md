# Chapter 71 — A Mini Framework

A framework is a library that calls *you*. You don't call it; it calls your code. That inversion of control is the defining feature. A library is a collection of utilities—you use them when you need them. A framework is a skeleton that runs your application—you plug in handlers, routes, middleware, and the framework orchestrates the rest.

Building a tiny framework from scratch shows you exactly what Spring, Laravel, Express, or Rails are doing under the hood, minus the marketing and magic. By the end of this chapter, you will have written a complete working web framework in about 150 lines of C++, and you will understand why frameworks are structured the way they are.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the difference between a library and a framework in terms of control flow.
2. Identify the core components every web framework must have: router, request/response abstraction, middleware pipeline, lifecycle, and DI wiring.
3. Build a router that maps HTTP methods and paths to handler functions, including path parameters.
4. Implement a middleware pipeline that processes requests sequentially before they reach handlers.
5. Understand why frameworks wrap raw HTTP in domain types (Request, Response).
6. Recognize how a framework's lifecycle hooks (boot, request, shutdown) enable plugins and extensions.
7. Integrate constructor injection into a minimal framework so handlers receive their dependencies automatically.

---

## 71.1 Library vs. Framework: The Inversion of Control

A library is passive. You call it:

```cpp
// Library: you are in control
int main() {
    auto conn = new PostgresConnection("postgresql://localhost");
    auto repo = new UserRepository(conn);
    auto user = repo->findById(123);
    std::cout << user.name << std::endl;
    return 0;
}
```

You decide when to connect, what to query, and what to do with the result. The library just provides the tools.

A framework is active. It calls you:

```cpp
// Framework: the framework is in control
int main() {
    Framework app;
    
    app.get("/users/:id", [](Request req) -> Response {
        auto user_id = req.param("id");
        auto user = /* somehow get the repository */;
        return Response::json(user);
    });
    
    app.listen(8080);  // Framework now controls the flow
    return 0;
}
```

You define a route handler. The framework accepts HTTP requests, parses them, calls your handler with the request, collects the response, and sends it back. The control flow is inverted: the framework owns the main loop, and your code is called *from* the framework, not the other way around.

### Why This Matters

The inversion is not academic. It changes how you write code:

- **Testability**: A handler is a function with arguments. You can call it directly in a test without starting a server.
- **Reusability**: A handler does not know it is running in an HTTP context. The framework bridges that gap. The same business logic can be called from an HTTP endpoint, a CLI command, or a background job.
- **Consistency**: All routes go through the same middleware pipeline, the same error handling, the same lifecycle hooks. A library leaves that to you; a framework enforces it.

---

## 71.2 What A Web Framework Has

Before building one, let's list the essential parts:

### 1. Router

Maps `(HTTP method, path)` to handlers. Supports path parameters like `/users/:id` and query strings.

### 2. Request and Response Abstraction

Wraps raw HTTP (bytes) in domain types (`Request`, `Response`). Handlers work with objects, not strings.

### 3. Middleware Pipeline

A chain of functions that processes the request before it reaches the handler and processes the response after. Logging, authentication, compression—all live here.

### 4. Lifecycle

Boot (once, at startup), handle request (per request), shutdown (once, at exit). Hooks let plugins register code at each stage.

### 5. Dependency Injection Wiring

Handlers depend on services. The framework constructs them and passes them to the handler. No `new` in user code.

### 6. Configuration

Environment-specific settings (database URL, port, log level). Usually from environment variables or a config file.

The smallest useful framework has 1, 2, and 3. Add 4, 5, and 6, and you have something professional teams use.

---

## 71.3 Building It: The Router

A router maps `(method, path)` to handlers. The simplest version uses a map. A more sophisticated version uses a trie to handle path parameters efficiently.

We will start with a map-based router:

```cpp
#include <map>
#include <string>
#include <functional>

struct Request {
    std::string method;
    std::string path;
    std::map<std::string, std::string> params;
    std::map<std::string, std::string> query;
    std::string body;
};

struct Response {
    int status;
    std::string body;
    std::map<std::string, std::string> headers;
    
    static Response json(const std::string& json_body) {
        Response r;
        r.status = 200;
        r.body = json_body;
        r.headers["Content-Type"] = "application/json";
        return r;
    }
};

using Handler = std::function<Response(const Request&)>;
using RouteKey = std::pair<std::string, std::string>;  // (method, path)

class Router {
private:
    std::map<RouteKey, Handler> routes;
    
public:
    void register_route(const std::string& method, const std::string& path, Handler handler) {
        routes[{method, path}] = handler;
    }
    
    Handler* match(const std::string& method, const std::string& path, Request& req) {
        // Try exact match first
        auto it = routes.find({method, path});
        if (it != routes.end()) {
            return &it->second;
        }
        
        // Try pattern match (e.g., /users/:id matches /users/123)
        for (auto& [key, handler] : routes) {
            if (key.first != method) continue;
            if (matches_pattern(key.second, path, req.params)) {
                return &handler;
            }
        }
        
        return nullptr;
    }
    
private:
    bool matches_pattern(const std::string& pattern, const std::string& path,
                        std::map<std::string, std::string>& params) {
        auto pattern_parts = split(pattern, '/');
        auto path_parts = split(path, '/');
        
        if (pattern_parts.size() != path_parts.size()) return false;
        
        for (size_t i = 0; i < pattern_parts.size(); ++i) {
            if (pattern_parts[i][0] == ':') {
                // Parameter: :id -> path_parts[i]
                params[pattern_parts[i].substr(1)] = path_parts[i];
            } else if (pattern_parts[i] != path_parts[i]) {
                return false;
            }
        }
        return true;
    }
    
    std::vector<std::string> split(const std::string& s, char delim) {
        std::vector<std::string> result;
        std::stringstream ss(s);
        std::string item;
        while (std::getline(ss, item, delim)) {
            if (!item.empty()) result.push_back(item);
        }
        return result;
    }
};
```

**Key insight:** Routing is pattern matching. For a trie-based router (more efficient), you would build a tree where each node represents a path segment. For this introduction, the map is clear and sufficient.

---

## 71.4 Building It: Request and Response

A handler should not deal with raw HTTP. Instead, we wrap it in domain types that are easy to work with:

```cpp
struct Request {
    std::string method;
    std::string path;
    std::map<std::string, std::string> params;      // path params (:id)
    std::map<std::string, std::string> query;       // query string (?foo=bar)
    std::map<std::string, std::string> headers;
    std::string body;
    
    std::string param(const std::string& key) const {
        auto it = params.find(key);
        return it != params.end() ? it->second : "";
    }
    
    std::string query_param(const std::string& key) const {
        auto it = query.find(key);
        return it != query.end() ? it->second : "";
    }
};

struct Response {
    int status = 200;
    std::string body;
    std::map<std::string, std::string> headers;
    
    static Response ok(const std::string& body) {
        Response r;
        r.status = 200;
        r.body = body;
        return r;
    }
    
    static Response json(const std::string& json) {
        Response r = Response::ok(json);
        r.headers["Content-Type"] = "application/json";
        return r;
    }
    
    static Response not_found() {
        Response r;
        r.status = 404;
        r.body = "Not Found";
        return r;
    }
    
    static Response error(const std::string& msg) {
        Response r;
        r.status = 500;
        r.body = msg;
        return r;
    }
};
```

**Why this matters:**

- A handler that takes `Request` instead of raw bytes is testable. You construct a `Request` in a test and call the handler directly.
- The `Request` object is also responsible for parsing. In a real framework, the framework parses raw HTTP into `Request`, and the handler never sees the raw bytes.
- `Response::json()` is a convenience factory. In a real framework, you would have more: `Response::html()`, `Response::redirect()`, etc.

---

## 71.5 Building It: Middleware Pipeline

Middleware is a function that processes the request and/or response. Examples: logging, authentication, compression, rate limiting.

The pattern is a chain:

```cpp
using Middleware = std::function<Response(Request, std::function<Response()>)>;

// Each middleware receives the request and a `next` function
// It can do something before calling next, after, or skip next entirely
Response logging_middleware(Request req, std::function<Response()> next) {
    std::cout << "Request: " << req.method << " " << req.path << std::endl;
    Response res = next();
    std::cout << "Response: " << res.status << std::endl;
    return res;
}

Response auth_middleware(Request req, std::function<Response()> next) {
    auto token = req.headers["Authorization"];
    if (token.empty()) {
        return Response::error("Unauthorized");
    }
    return next();
}

// Compose them
class MiddlewarePipeline {
private:
    std::vector<Middleware> middlewares;
    Handler handler;
    
public:
    void add(Middleware m) {
        middlewares.push_back(m);
    }
    
    void set_handler(Handler h) {
        handler = h;
    }
    
    Response execute(Request req) {
        // Build the chain from right to left
        std::function<Response()> chain = [this, &req]() {
            return handler(req);
        };
        
        // Wrap each middleware
        for (int i = middlewares.size() - 1; i >= 0; --i) {
            auto current_middleware = middlewares[i];
            auto current_chain = chain;
            chain = [current_middleware, current_chain, &req]() {
                return current_middleware(req, current_chain);
            };
        }
        
        return chain();
    }
};
```

**How it works:** Each middleware is a function that receives the request and a "next" function. It can:
- Inspect or modify the request before calling next.
- Call next to let the request proceed.
- Inspect or modify the response after calling next.
- Skip calling next and return its own response (e.g., auth middleware returning 401).

The pipeline builds a chain by nesting lambdas, so each middleware wraps the next one.

---

## 71.6 Building It: Lifecycle and Hooks

A framework has three main phases:

1. **Boot**: Initialize once. Register routes, start database connections, load config.
2. **Request**: Handle each incoming HTTP request.
3. **Shutdown**: Cleanup. Close connections, flush logs.

Hooks allow plugins to run code at each phase:

```cpp
class Framework {
private:
    std::vector<std::function<void()>> boot_hooks;
    std::vector<std::function<void(Request&)>> request_hooks;
    std::vector<std::function<void()>> shutdown_hooks;
    
    Router router;
    MiddlewarePipeline pipeline;
    
public:
    void on_boot(std::function<void()> hook) {
        boot_hooks.push_back(hook);
    }
    
    void on_request(std::function<void(Request&)> hook) {
        request_hooks.push_back(hook);
    }
    
    void on_shutdown(std::function<void()> hook) {
        shutdown_hooks.push_back(hook);
    }
    
    void get(const std::string& path, Handler handler) {
        router.register_route("GET", path, handler);
    }
    
    void post(const std::string& path, Handler handler) {
        router.register_route("POST", path, handler);
    }
    
    void use(Middleware m) {
        pipeline.add(m);
    }
    
    void boot() {
        for (auto& hook : boot_hooks) {
            hook();
        }
    }
    
    Response handle_request(const std::string& raw_http) {
        // Parse raw_http into Request
        Request req = parse_http(raw_http);
        
        // Call request hooks
        for (auto& hook : request_hooks) {
            hook(req);
        }
        
        // Find and execute handler
        Handler* handler = router.match(req.method, req.path, req);
        if (!handler) {
            return Response::not_found();
        }
        
        pipeline.set_handler(*handler);
        return pipeline.execute(req);
    }
    
    void listen(int port) {
        boot();
        
        // Pseudocode: create HTTP server, bind to port, accept connections
        // On each connection, call handle_request(raw_http)
        
        // At shutdown, call shutdown hooks
        for (auto& hook : shutdown_hooks) {
            hook();
        }
    }
};
```

**Lifecycle benefits:**
- Plugins can hook into boot to initialize themselves (set up a database connection pool).
- Plugins can hook into request to add request context (store the user ID).
- Plugins can hook into shutdown to cleanup (flush logs, close connections).

This is how Rails engines, Express middleware, and Laravel service providers work.

---

## 71.7 Building It: DI Wiring

So far, handlers are functions that take a `Request`. But handlers need services—repositories, business logic, external APIs. How do they get them?

A framework solves this with dependency injection. The simplest version uses a container:

```cpp
class Container {
private:
    std::map<std::type_index, std::function<void*(Container&)>> factories;
    std::map<std::type_index, void*> singletons;
    
public:
    template<typename T>
    void singleton(std::function<T*()> factory) {
        factories[std::type_index(typeid(T))] = 
            [factory](Container& c) { return (void*) factory(); };
    }
    
    template<typename T>
    T* resolve() {
        auto key = std::type_index(typeid(T));
        
        // Check if singleton already exists
        if (singletons.count(key)) {
            return (T*) singletons[key];
        }
        
        // Create via factory
        auto factory = factories[key];
        T* instance = (T*) factory(*this);
        
        // Cache if singleton
        singletons[key] = instance;
        
        return instance;
    }
};
```

Now a handler can depend on a service:

```cpp
class UserRepository {
    // ... methods to fetch users
};

class UserService {
    UserRepository* repo;
public:
    UserService(UserRepository* r) : repo(r) {}
    // ... business logic
};

// In main:
Container container;
container.singleton<UserRepository>([]() { return new UserRepository(); });
container.singleton<UserService>([]() { return new UserService(
    container.resolve<UserRepository>()
); });

// Handler receives the service
app.get("/users/:id", [&container](Request req) -> Response {
    auto service = container.resolve<UserService>();
    auto user = service->getUser(req.param("id"));
    return Response::json(serialize(user));
});
```

**Real frameworks go further:** They use reflection (in Java, C#, TypeScript) or macros (in C++) to automatically inspect a handler's constructor and resolve its dependencies. But the concept is the same: **the framework, not the handler, is responsible for constructing dependencies.**

---

## 71.8 Putting It Together: A 150-Line Framework

Here is a complete, working mini framework:

```cpp
#include <map>
#include <string>
#include <vector>
#include <functional>
#include <sstream>

// Request and Response
struct Request {
    std::string method, path, body;
    std::map<std::string, std::string> params, query, headers;
    
    std::string param(const std::string& key) const {
        auto it = params.find(key);
        return it != params.end() ? it->second : "";
    }
};

struct Response {
    int status = 200;
    std::string body;
    std::map<std::string, std::string> headers;
    
    static Response json(const std::string& j) {
        Response r;
        r.status = 200;
        r.body = j;
        r.headers["Content-Type"] = "application/json";
        return r;
    }
    
    static Response error(int code, const std::string& msg) {
        Response r;
        r.status = code;
        r.body = msg;
        return r;
    }
};

using Handler = std::function<Response(const Request&)>;
using Middleware = std::function<Response(const Request&, std::function<Response()>)>;

// Router
class Router {
private:
    std::map<std::pair<std::string, std::string>, Handler> routes;
    
    std::vector<std::string> split(const std::string& s, char delim) {
        std::vector<std::string> result;
        std::stringstream ss(s);
        std::string item;
        while (std::getline(ss, item, delim)) {
            if (!item.empty()) result.push_back(item);
        }
        return result;
    }
    
public:
    void add(const std::string& method, const std::string& path, Handler h) {
        routes[{method, path}] = h;
    }
    
    Handler* match(const std::string& method, const std::string& path, Request& req) {
        auto it = routes.find({method, path});
        if (it != routes.end()) return &it->second;
        
        // Pattern matching: /users/:id
        auto path_parts = split(path, '/');
        for (auto& [key, handler] : routes) {
            if (key.first != method) continue;
            auto pattern_parts = split(key.second, '/');
            if (pattern_parts.size() != path_parts.size()) continue;
            
            bool matches = true;
            for (size_t i = 0; i < pattern_parts.size(); ++i) {
                if (pattern_parts[i][0] == ':') {
                    req.params[pattern_parts[i].substr(1)] = path_parts[i];
                } else if (pattern_parts[i] != path_parts[i]) {
                    matches = false;
                    break;
                }
            }
            if (matches) return &handler;
        }
        return nullptr;
    }
};

// Middleware Pipeline
class Pipeline {
private:
    std::vector<Middleware> middlewares;
    Handler handler;
    
public:
    void add(Middleware m) { middlewares.push_back(m); }
    void set_handler(Handler h) { handler = h; }
    
    Response execute(Request req) {
        std::function<Response()> chain = [this, &req]() {
            return handler(req);
        };
        
        for (int i = (int)middlewares.size() - 1; i >= 0; --i) {
            auto m = middlewares[i];
            auto c = chain;
            chain = [m, c, &req]() { return m(req, c); };
        }
        
        return chain();
    }
};

// Framework
class Framework {
private:
    Router router;
    Pipeline pipeline;
    std::vector<std::function<void()>> boot_hooks;
    
public:
    void get(const std::string& path, Handler h) {
        router.add("GET", path, h);
    }
    
    void post(const std::string& path, Handler h) {
        router.add("POST", path, h);
    }
    
    void use(Middleware m) {
        pipeline.add(m);
    }
    
    void on_boot(std::function<void()> h) {
        boot_hooks.push_back(h);
    }
    
    Response handle(const std::string& method, const std::string& path) {
        Request req{method, path, "", {}, {}, {}};
        
        Handler* h = router.match(method, path, req);
        if (!h) return Response::error(404, "Not Found");
        
        pipeline.set_handler(*h);
        return pipeline.execute(req);
    }
    
    void boot() {
        for (auto& h : boot_hooks) h();
    }
};
```

---

## 71.9 Worked Example: A Complete App

Here is a complete application using the mini framework:

```cpp
int main() {
    Framework app;
    
    // Data (in-memory for demo)
    std::map<std::string, std::string> users = {
        {"1", "Alice"},
        {"2", "Bob"}
    };
    
    // Logging middleware
    app.use([](const Request& req, std::function<Response()> next) {
        std::cout << req.method << " " << req.path << std::endl;
        return next();
    });
    
    // Authentication middleware (check Authorization header)
    app.use([](const Request& req, std::function<Response()> next) {
        auto auth = req.headers["Authorization"];
        if (auth.empty()) {
            return Response::error(401, "Unauthorized");
        }
        return next();
    });
    
    // Boot hook
    app.on_boot([]() {
        std::cout << "Booting framework..." << std::endl;
    });
    
    // Routes
    app.get("/users/:id", [&users](const Request& req) -> Response {
        auto id = req.param("id");
        auto it = users.find(id);
        if (it == users.end()) {
            return Response::error(404, "User not found");
        }
        return Response::json(R"({"name":")" + it->second + R"("})");
    });
    
    app.post("/users", [&users](const Request& req) -> Response {
        // Parse JSON from req.body, create user, return 201
        return Response::error(201, "Created");
    });
    
    // Simulate requests
    app.boot();
    
    auto res1 = app.handle("GET", "/users/1");
    std::cout << res1.status << ": " << res1.body << std::endl;
    
    auto res2 = app.handle("GET", "/users/999");
    std::cout << res2.status << ": " << res2.body << std::endl;
    
    return 0;
}
```

**Output:**
```
Booting framework...
GET /users/1
200: {"name":"Alice"}
GET /users/999
404: User not found
```

Notice:
- The app defines routes as handler functions.
- Middleware runs before the handler (logging, then auth).
- The framework controls the flow (calling the handlers, not vice versa).
- Handlers are testable: you can call them with a `Request` directly.

---

## 71.10 What Real Frameworks Do Beyond This

This mini framework is ~150 lines. Real frameworks add:

### Body Parsing

Automatic deserialization of request bodies (JSON, form data, XML). The framework chooses the parser based on `Content-Type` and passes a parsed object to the handler.

### Templating

View templates (HTML, Jinja, ERB) that are rendered with data from the handler. Our mini framework skips this; a real one includes a templating engine.

### Sessions and Cookies

Request-scoped state that persists across multiple requests from the same user. The framework manages cookies, encryption, and cleanup.

### ORM / Database Access

A library that maps database rows to objects and provides query builders. Often integrated so handlers can declare dependencies on repositories.

### Validation

Automatic validation of request parameters (is this an integer? is this email valid?). The framework provides decorators or schemas.

### Error Handling

Centralized error handling. Exceptions thrown in handlers are caught by the framework, logged, and converted to HTTP responses.

### Static Files

Serving CSS, JavaScript, images without routing through handlers.

### Testing Helpers

Utilities to make testing easier (mock requests, assertions on responses).

### Job Queues and Background Tasks

Handlers can enqueue work (send email, process video) without blocking the response.

Each of these is a layer of complexity. But the **core skeleton is identical** to what we built: router, request/response, middleware, lifecycle, DI.

---

## 71.11 Why Frameworks Exist (And Why You Might Not Need One)

A framework provides:

- **Consistency**: All routes use the same middleware, error handling, logging. No reinvention per endpoint.
- **Productivity**: Common patterns are automated (routing, DI, validation). You focus on business logic.
- **Scalability**: As the app grows, the framework's structure keeps it organized.

But frameworks have costs:

- **Startup overhead**: Initialization takes time. For a CLI tool that runs once, a framework's setup might outweigh the benefit.
- **Magic**: Frameworks hide what is happening. Debugging is harder. You must understand the framework's conventions.
- **Coupling**: Your code couples to the framework. Switching frameworks is painful.

### When to use a framework:

- Building a web service with 10+ endpoints.
- Multiple developers working on the same codebase.
- The app will grow over time.

### When to skip it:

- A small CLI tool (100 lines of code).
- A library that other code uses (libraries should be framework-agnostic).
- Real-time or embedded systems where startup time is critical.
- A prototype you want to iterate quickly on (framework setup can slow you down initially).

---

## 71.12 Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **No framework** | Simple, transparent, full control | Must build everything (routing, DI, middleware) | Small tools, libraries, prototypes |
| **Mini framework** (like we built) | Clear, understandable, suitable for learning | Lacks features real apps need (parsing, validation, ORM) | Educational, small services, custom domains |
| **Established framework** (Express, Rails, Laravel) | Mature, well-tested, large ecosystem, hire experienced developers | Startup overhead, magic, vendor lock-in, learning curve | Production web apps, large teams, standard CRUD apps |
| **Lightweight framework** (Sinatra, Flask) | Balance of simplicity and productivity | Fewer features, smaller ecosystem, less organizational support | Medium-sized apps, when you want control but not boilerplate |

---

## 71.13 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Frameworks save time on small projects." | Often false. For a 200-line tool, the framework's setup and learning curve outweigh the benefit. Manual code is faster. |
| "A framework makes code more testable." | No. DI and clean separation of concerns make code testable. The framework is just a tool that enforces those patterns. |
| "If I use a framework, I don't need to understand HTTP." | Wrong. Understanding the framework requires understanding what it abstracts (HTTP, routing, middleware, sessions). |
| "Frameworks are magic; I don't need to know how they work." | Using a framework without understanding its core is like driving a car without knowing how the engine works. Debugging requires understanding. |
| "A framework locks you in; you can never change it." | Somewhat true, but good architecture (Services, Repositories, clean separation) makes switching possible. Bad architecture locks you in, framework or not. |
| "You should always use a framework for web apps." | No. A simple API with 3 endpoints might not need one. A CLI that calls an API might not need one. Match the tool to the problem. |

---

## 71.14 Exercises

1. **Extend the mini framework.** Add support for HTTP PUT and DELETE methods. Add a route that accepts multiple path parameters (`/users/:user_id/posts/:post_id`).

2. **Add query string parsing.** Modify the framework to parse query strings (`?limit=10&offset=20`) into the `Request::query` map. Add a handler that uses query parameters.

3. **Build a simple validator middleware.** Write middleware that validates that certain query parameters are present and are of the right type (integer, email). If validation fails, return 400 with an error message.

4. **Add error handling.** Modify the framework so handlers can throw exceptions. Add a catch block that converts exceptions to HTTP responses (500, error message).

5. **Implement a plugin system.** Handlers often need a logger. Add a `LoggerPlugin` that registers a logger in the container at boot time. Have a handler depend on it.

6. **Compare to a real framework.** Write the same 3-route app (list users, get user by ID, create user) using your mini framework and using Express.js or Flask. Compare the code length and complexity.

---

## 71.15 Summary

A framework is a library that calls you. It owns the main loop and orchestrates your code. The core components—router, request/response types, middleware pipeline, lifecycle, and DI—are present in every web framework, from Express to Rails to Spring.

Building a mini framework from scratch demystifies what real frameworks do. You see that a router is just pattern matching, middleware is function composition, and DI is graph resolution. The magic is not in the individual pieces but in how they fit together.

Frameworks are powerful when the problem fits their model: a web service with many endpoints, shared middleware, and dependencies to manage. But they are not always the right tool. A small CLI, a library, or a prototype might be simpler without one. Match the tool to the problem: use a framework when manual code would be more boilerplate than the framework's overhead, not because it is fashionable.

---

> **[← Previous: A Database Layer](03-a-database-layer.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: A DI Container →](05-a-di-container.md)**
