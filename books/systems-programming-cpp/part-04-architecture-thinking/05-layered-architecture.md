# Chapter 37 — Layered Architecture

Layered architecture is the most ubiquitous organizational pattern in modern software. Walk into any web application, any CRUD service, any desktop program built in the last 20 years, and you will find layers. They are so common that most developers treat them as *the* way to organize code, the default, the obvious choice.

Done well, layers make a codebase easy to reason about. Each layer has a clear responsibility. Dependencies point one way. A developer can understand one layer without mastering all the others. Changes to persistence don't ripple into business logic. A unit test can mock the database layer without touching a line of infrastructure code.

Done poorly, layering is bureaucratic ceremony hiding a ball of mud. Every operation traverses a chain of six thin classes that add nothing — `UserRequest` → `UserController` → `UserService` → `UserRepository` → `UserMapper` → `UserQuery` — the indirection cost paid, the cognitive overhead incurred, with zero benefit. The "separation" is illusory because the layers are so tightly coupled that changing one layer requires changes in all the others. The abstraction has not scaled the cognition; it has just displaced it.

This chapter is about understanding when layers help and when they hurt, and the specific architectural patterns that make layering work in practice.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Describe the classic three-layer model (Presentation, Domain, Persistence) and explain why dependencies must point strictly downward.
2. Understand the hexagonal/ports-and-adapters architecture (Alistair Cockburn) and why it inverts the traditional dependency direction to make the domain independent of frameworks.
3. Recognize the concentric rings of Clean Architecture (Robert C. Martin) and see how it generalizes the dependency rule across levels of abstraction.
4. Identify the failure modes of strict layering: accidental complexity, unnecessary traversal, layer-as-pass-through, leaking persistence concerns.
5. Decide whether to enforce strict layers or allow pragmatic shortcuts (layer skipping) in your codebase.
6. Spot common anti-patterns: anemic domain models, service-only architecture, passive repositories, and what to do about them.
7. Implement a real, three-layer system in C++ and know exactly where each kind of code belongs.

---

## 5.1 The Classic Three-Layer Model

The most durable architectural pattern for business applications divides code into three layers:

```
┌─────────────────────────────────────┐
│   PRESENTATION LAYER                │
│   (HTTP handlers, CLI, UI screens)  │
└──────────────┬──────────────────────┘
               │ (depends on)
               ↓
┌─────────────────────────────────────┐
│   DOMAIN/BUSINESS LOGIC LAYER       │
│   (Rules, calculations, workflows)  │
└──────────────┬──────────────────────┘
               │ (depends on)
               ↓
┌─────────────────────────────────────┐
│   PERSISTENCE LAYER                 │
│   (Database, file storage, cache)   │
└─────────────────────────────────────┘
```

Each layer depends *only* on the layer immediately below it — never upward, never sideways.

### Why Strictly Downward Dependencies?

When a lower layer depends on a higher layer, you have created a cycle. Cycles make a codebase hostile to change: modifying code in the lower layer now requires understanding and potentially changing the higher layer. The two layers are no longer independent; they are locked together. Testing becomes a nightmare — you can't test the lower layer without pulling in the higher layer.

The rule is simple: **dependencies point downward or stop**. A layer can depend on what is below; it cannot depend on what is above.

### The Presentation Layer

The presentation layer handles all communication between the outside world and your system. This includes:

- HTTP request handlers (controllers in web frameworks)
- CLI argument parsing and output formatting
- Web socket connections
- Message queue consumers
- gRPC service implementations
- Desktop UI event handlers

The presentation layer's job is translation: convert external input (JSON, command-line flags, UI events) into calls to the domain layer, and convert the domain layer's output back into external format (HTTP responses, rendered HTML, terminal output, database state changes).

A good presentation layer is **thin**. It should not contain business logic. If your HTTP handler is 50 lines and mostly parsing, routing, error handling, and serialization — that is good. If it is 50 lines of business logic with some HTTP wrapper around it — you have violated the pattern.

Example of bad presentation layer (logic leaks upward):

```cpp
// BAD: Business logic in the controller
class UserController {
    void registerUser(const RegisterRequest& req) {
        // This is domain logic, not presentation logic
        if (req.email.length() < 5) {
            return error("Email too short");
        }
        if (!isValidEmailFormat(req.email)) {
            return error("Email invalid");
        }
        // ... 20 more validation rules
        auto user = db.createUser(req.email, req.password);
        // ...
    }
};
```

Example of good presentation layer (thin wrapper):

```cpp
// GOOD: Controller delegates to domain
class UserController {
    std::unique_ptr<UserService> service;
    
    RegisterResponse registerUser(const RegisterRequest& req) {
        auto result = service->registerUser(
            req.email, 
            req.password
        );
        
        if (!result.isSuccess()) {
            return RegisterResponse{
                .success = false,
                .error = result.error()
            };
        }
        
        return RegisterResponse{
            .success = true,
            .userId = result.userId()
        };
    }
};
```

### The Domain Layer

This layer is the heart of your system. It contains the rules, calculations, and workflows that make your software valuable. Examples:

- User registration logic (password rules, email verification)
- Financial calculations (interest, amortization, tax)
- Workflow state machines (order approval, job scheduling)
- Validation rules (domain-specific constraints)
- Business invariants (a transaction cannot reference a deleted account)

The domain layer **must not know about the presentation layer above or the persistence layer below**. It does not know that data comes from an HTTP request. It does not know that results go to a SQL database. It knows only its own concepts and rules.

This is the hardest rule to follow in practice, and it is worth the effort. A domain layer that is independent of frameworks and infrastructure becomes:

- **Testable without complex fixtures.** A test can call domain code directly, passing plain C++ objects, without setting up HTTP servers or databases.
- **Portable.** The same domain logic works whether you later add a gRPC endpoint, a CLI, or a background job consumer.
- **Durable.** The domain outlasts the framework. You can rewrite your web framework; the domain is unaffected.

Example domain layer:

```cpp
// The domain knows about users, not HTTP or databases
class UserService {
    std::unique_ptr<UserRepository> repo;
    std::unique_ptr<EmailValidator> validator;
    
public:
    struct RegisterResult {
        bool success;
        std::string userId;
        std::string errorMessage;
    };
    
    RegisterResult registerUser(
        const std::string& email,
        const std::string& password
    ) {
        // Domain rule: email must be valid
        if (!validator->isValid(email)) {
            return {false, "", "Invalid email"};
        }
        
        // Domain rule: password must meet strength requirements
        if (!meetsPasswordRequirements(password)) {
            return {false, "", "Password too weak"};
        }
        
        // Domain rule: email must be unique
        if (repo->userExistsByEmail(email)) {
            return {false, "", "Email already in use"};
        }
        
        // If all rules pass, create the user
        auto newUser = repo->createUser(email, hashPassword(password));
        return {true, newUser.id, ""};
    }

private:
    bool meetsPasswordRequirements(const std::string& pwd) {
        // Domain rule: at least 8 chars
        return pwd.length() >= 8;
    }
};
```

Notice: no HTTP, no JSON, no database code. Just domain rules in plain C++.

### The Persistence Layer

The persistence layer handles all interaction with durable storage. This includes:

- SQL database connections and queries
- Object-relational mapping
- Caching strategies
- File I/O
- Cache invalidation
- Transaction management

The persistence layer exposes **repositories** — abstractions that let the domain layer ask for data without knowing how that data is stored.

Example persistence layer:

```cpp
// The repository interface (owned by the domain)
class UserRepository {
public:
    virtual ~UserRepository() = default;
    virtual std::optional<User> getUserById(const std::string& id) = 0;
    virtual bool userExistsByEmail(const std::string& email) = 0;
    virtual User createUser(
        const std::string& email,
        const std::string& hashedPassword
    ) = 0;
};

// The implementation (owned by persistence layer)
class SqlUserRepository : public UserRepository {
    std::shared_ptr<Database> db;
    
public:
    std::optional<User> getUserById(const std::string& id) override {
        auto row = db->query(
            "SELECT id, email FROM users WHERE id = ?",
            {id}
        );
        if (!row) return std::nullopt;
        return User{row["id"], row["email"]};
    }
    
    bool userExistsByEmail(const std::string& email) override {
        auto row = db->query(
            "SELECT 1 FROM users WHERE email = ? LIMIT 1",
            {email}
        );
        return row.has_value();
    }
    
    User createUser(
        const std::string& email,
        const std::string& hashedPassword
    ) override {
        auto id = db->execute(
            "INSERT INTO users (email, password_hash) VALUES (?, ?)",
            {email, hashedPassword}
        );
        return User{id, email};
    }
};
```

The key insight: the domain layer depends on the `UserRepository` *interface*, not the `SqlUserRepository` implementation. The implementation can change — swap SQL for NoSQL, add a cache layer, mock it for tests — without touching domain code.

---

## 5.2 Hexagonal Architecture (Ports and Adapters)

Alistair Cockburn, in his 2005 paper "Hexagonal Architecture," reframed the three-layer model in a way that clarifies the dependency direction and makes the domain-centric principle more explicit.

In hexagonal architecture:

1. The **domain** is at the center — the core business logic.
2. The **ports** are the interfaces the domain exposes and the services the domain requires.
3. The **adapters** are implementations of those ports, connecting the domain to the outside world.

```
External System (Web)     External System (Database)
         |                           |
         ↓                           ↓
    HTTP Adapter            Database Adapter
         ↑                           ↑
         └──────────┬─────────────────┘
                    │ (ports)
                    ↓
            ┌───────────────┐
            │  DOMAIN CORE  │
            │  (Business    │
            │   Logic)      │
            └───────────────┘
```

This looks similar to the three-layer model, but the key difference is in how dependency is structured.

### Inbound Ports (Driving Adapters)

An inbound port is something the domain *provides* — an interface that the outside world uses to interact with your system.

- HTTP endpoint → UserController → UserService (inbound port)
- CLI command → CliHandler → UserService (inbound port)
- Message queue event → EventConsumer → UserService (inbound port)

The adapter translates external format into a call to the domain port.

### Outbound Ports (Driven Adapters)

An outbound port is something the domain *needs* — a service the domain calls to accomplish its work.

- UserRepository (domain needs to retrieve/store users)
- EmailSender (domain needs to send emails)
- Clock (domain needs current time)
- Logger (domain needs to record events)

The adapter implements the port using some external resource (database, SMTP, system clock, logging backend).

### Why Hexagonal Inverts Dependency

In the traditional three-layer model, the domain depends on repositories (the layer below). This is correct, but hexagonal architecture makes the inversion-of-control principle explicit: the domain *defines the interface* of what it needs. The persistence layer *implements* that interface. The persistence layer depends on the domain, not the reverse.

This is not just semantic — it has real consequences. When the domain defines what repository it needs, the domain remains central and independent. When the persistence layer is free to choose *how* to implement that interface, you gain flexibility.

Example: the domain defines what a logger must do:

```cpp
// Domain-owned port
class Logger {
public:
    virtual ~Logger() = default;
    virtual void info(const std::string& msg) = 0;
    virtual void error(const std::string& msg) = 0;
};
```

The persistence layer provides an implementation:

```cpp
// Persistence-owned adapter
class ConsoleLogger : public Logger {
    void info(const std::string& msg) override {
        std::cout << "[INFO] " << msg << "\n";
    }
    
    void error(const std::string& msg) override {
        std::cerr << "[ERROR] " << msg << "\n";
    }
};
```

Tests provide a mock implementation:

```cpp
// Test-owned adapter
class MockLogger : public Logger {
    std::vector<std::string> logs;
    
    void info(const std::string& msg) override {
        logs.push_back("INFO: " + msg);
    }
    
    void error(const std::string& msg) override {
        logs.push_back("ERROR: " + msg);
    }
    
public:
    const auto& getLogsSinceStart() const { return logs; }
};
```

All three satisfy the port. The domain does not care which. That is the hexagonal principle: domain defines the interface, others implement it.

---

## 5.3 Clean Architecture (Concentric Rings)

Robert C. Martin's "Clean Architecture" generalizes the same principle across more granular levels of abstraction. Instead of three layers, it defines four concentric rings, from innermost to outermost:

```
┌─────────────────────────────────┐
│  Frameworks & Drivers           │
│  (Rails, Django, Spring, DB)    │
├─────────────────────────────────┤
│  Interface Adapters             │
│  (Controllers, Presenters, Gateways) │
├─────────────────────────────────┤
│  Application Business Rules     │
│  (Use Cases, Interactors)       │
├─────────────────────────────────┤
│  Enterprise Business Rules      │
│  (Entities, Domain Objects)     │
└─────────────────────────────────┘
```

The dependency rule remains: **dependencies always point inward**. The outermost ring (frameworks) can know about the rings below, but the innermost ring (entities) knows nothing about anything above it.

- **Entities** are the core domain concepts — `User`, `Order`, `Transaction`. They contain business logic that is truly domain-universal; they would be the same logic in any implementation, any technology.
- **Use Cases** (sometimes called Interactors or Application Services) orchestrate entities to accomplish a specific user goal. A use case might combine multiple entities: "register a user" involves the User entity, the Email entity, the PasswordHash entity.
- **Interface Adapters** translate between external format and internal format. Controllers, presenters, repositories all live here.
- **Frameworks & Drivers** are the toolkits you depend on — web framework, database driver, logging library. They are outermost because changing them should not affect inner rings.

This is less about the number of layers and more about the principle: **policy depends on none of the details; details depend on policy**. Policy is the domain; details are everything else.

---

## 5.4 Why Strict Layering Fails In Practice

Layering is a powerful organizing principle. But strictly enforced, it creates problems.

### Problem 1: Traversal Tax

Consider a simple operation: retrieve a user by ID and return it as JSON.

In strict three-layer architecture:

```
Request Handler (Presentation)
    ↓
Service.getUserById(id)  (Domain)
    ↓
Repository.getUserById(id)  (Persistence)
    ↓
Database Query
    ↓
Return User object
    ↓
Return User object from Repository
    ↓
Return User object from Service
    ↓
Serialize User to JSON
    ↓
Return HTTP 200 + JSON
```

Seven function calls for an operation that logically is "fetch and return." Each call adds a tiny cost: a function prologue, argument passing, a return value, maybe a vtable lookup if using interfaces. Individually negligible. Stacked across thousands of requests, significant. More problematically, the call stack depth makes debugging harder.

### Problem 2: The Pass-Through Layer

Consider this domain service:

```cpp
class OrderService {
    std::unique_ptr<OrderRepository> repo;
    
public:
    std::optional<Order> getOrder(const std::string& id) {
        return repo->getOrder(id);
    }
};
```

This is a **pass-through layer**: the service does nothing but forward the call to the repository. It adds indirection without adding value. Yet strict layering requires it — the presentation layer must talk to the service, not the repository, even when the service does nothing.

### Problem 3: Leaking Persistence Concerns

The purest form of layering separates concerns perfectly. In reality, persistence decisions leak upward.

Example: pagination.

```cpp
// Domain-pure approach (impossible):
std::vector<User> getActiveUsers();

// Reality: must know about pagination
struct PageRequest {
    int pageNumber;
    int pageSize;
};

struct Page<T> {
    std::vector<T> items;
    int totalCount;
    int pageNumber;
};

Page<User> getActiveUsers(const PageRequest& req);
```

Pagination is a persistence concern (you don't paginate in memory, you paginate at the database layer), but the domain layer must know about it because the caller (presentation) needs pagination.

Similarly:

- **Sorting**: the database can sort efficiently; the domain can't. But the presentation wants to sort results.
- **Filtering**: same issue. The presentation wants to filter; filtering is efficient at the database; the domain must expose filter parameters.
- **Lazy loading**: whether an association is loaded eagerly or lazily is a persistence decision, but it affects domain object design.

### Problem 4: Anemic Domain Models

Strict layering often combines with a correlated anti-pattern: the **anemic domain model**. The domain layer contains data classes (entities) with no behavior — just getters and setters. All logic lives in service classes.

```cpp
// Anemic domain: data with no logic
class User {
public:
    std::string id;
    std::string email;
    std::string passwordHash;
    
    std::string getId() const { return id; }
    void setId(const std::string& i) { id = i; }
    // ... dozens of getters/setters
};

// All logic exiled to a service
class UserService {
    bool isPasswordValid(const User& u, const std::string& plain) {
        return bcrypt::verify(plain, u.passwordHash);
    }
};
```

This separates data and logic. The domain becomes a schema definition rather than a collection of behavioral concepts. Testing requires instantiating full objects and passing them to services. Refactoring is harder because logic is scattered across service classes instead of co-located with the data it operates on.

---

## 5.5 Pragmatic Layering: Allowing Downward Shortcuts

Real systems compromise between strict layering and practicality. The most sustainable approach is:

1. **Never depend upward.** The presentation layer never calls persistence directly, and the persistence layer never calls presentation.
2. **Allow skipping layers downward.** The presentation layer can call the persistence layer directly if the logic is simple enough (but must still serialize/deserialize correctly).
3. **Keep the domain layer separate.** Business rules always live in the domain layer.
4. **Don't force a layer to do nothing.** If a service is just forwarding to a repository, skip it.

Example: fetching a user by ID for display.

```cpp
// Strict three-layer (unnecessary traversal):
userController.handleGetUser(id) 
    → userService.getUser(id)
    → userRepository.getUser(id)

// Pragmatic (skipping a pass-through):
userController.handleGetUser(id)
    → userRepository.getUser(id)
    // (repository returns user, controller serializes to JSON)
```

The service is skipped because no business logic is involved — just retrieval and serialization.

But consider a more complex case: changing a password.

```cpp
// Pragmatic (still uses service because business logic is involved):
userController.handleChangePassword(userId, oldPwd, newPwd)
    → userService.changePassword(userId, oldPwd, newPwd)  
    // Service validates old password, checks new password rules,
    // logs the change, possibly invalidates sessions, etc.
    → userRepository.updatePasswordHash(userId, hash)
```

Here the service is essential because the business rules live there.

**The pragmatic rule:** do not enforce layers mechanically. Enforce the *dependencies* (never upward) and the *separation of concerns* (business logic in the domain, infrastructure in the infrastructure layer). Allow skipping layers when it adds no value.

---

## 5.6 Worked Example: BookShelf Application

Imagine a simple book management application. Users can:
- View their library
- Add a book
- Mark a book as read
- Get recommendations

Let's design it as three layers and see where code belongs.

### The Domain Layer

```cpp
// Domain concepts (no knowledge of HTTP or databases)
class Book {
public:
    std::string id;
    std::string title;
    std::string author;
    bool isRead = false;
    
    // Domain logic: can a book be marked as read?
    bool canMarkAsRead() const {
        return !isRead;
    }
    
    void markAsRead() {
        if (!canMarkAsRead()) {
            throw std::logic_error("Book already marked as read");
        }
        isRead = true;
    }
};

class BookRepository {
public:
    virtual ~BookRepository() = default;
    virtual std::optional<Book> getById(const std::string& id) = 0;
    virtual std::vector<Book> listByUser(const std::string& userId) = 0;
    virtual void save(const Book& b) = 0;
};

class BookService {
    std::unique_ptr<BookRepository> repo;
    
public:
    std::optional<Book> getBook(const std::string& id) {
        return repo->getById(id);
    }
    
    std::vector<Book> getUserLibrary(const std::string& userId) {
        return repo->listByUser(userId);
    }
    
    void markBookAsRead(const std::string& bookId) {
        auto book = repo->getById(bookId);
        if (!book) {
            throw std::logic_error("Book not found");
        }
        book->markAsRead();
        repo->save(*book);
    }
};
```

### The Persistence Layer

```cpp
class SqlBookRepository : public BookRepository {
    std::shared_ptr<Database> db;
    
public:
    std::optional<Book> getById(const std::string& id) override {
        auto row = db->query(
            "SELECT id, title, author, is_read FROM books WHERE id = ?",
            {id}
        );
        if (!row) return std::nullopt;
        
        Book b;
        b.id = row["id"];
        b.title = row["title"];
        b.author = row["author"];
        b.isRead = row["is_read"].asInt() == 1;
        return b;
    }
    
    std::vector<Book> listByUser(const std::string& userId) override {
        auto rows = db->query(
            "SELECT id, title, author, is_read FROM books WHERE user_id = ?",
            {userId}
        );
        
        std::vector<Book> result;
        for (const auto& row : rows) {
            Book b;
            b.id = row["id"];
            b.title = row["title"];
            b.author = row["author"];
            b.isRead = row["is_read"].asInt() == 1;
            result.push_back(b);
        }
        return result;
    }
    
    void save(const Book& b) override {
        db->execute(
            "UPDATE books SET is_read = ? WHERE id = ?",
            {b.isRead ? 1 : 0, b.id}
        );
    }
};
```

### The Presentation Layer

```cpp
class BookController {
    std::shared_ptr<BookService> service;
    
public:
    struct GetLibraryResponse {
        bool success;
        std::vector<Book> books;
    };
    
    GetLibraryResponse handleGetLibrary(const std::string& userId) {
        try {
            auto books = service->getUserLibrary(userId);
            return {true, books};
        } catch (const std::exception& e) {
            return {false, {}};
        }
    }
    
    struct MarkAsReadResponse {
        bool success;
        std::string error;
    };
    
    MarkAsReadResponse handleMarkAsRead(const std::string& bookId) {
        try {
            service->markBookAsRead(bookId);
            return {true, ""};
        } catch (const std::exception& e) {
            return {false, e.what()};
        }
    }
};
```

### Where Does Each Concern Live?

- **Domain:** Book logic (can it be marked as read?), workflow (mark it as read), invariants.
- **Persistence:** SQL queries, database connection, row-to-object mapping.
- **Presentation:** HTTP parsing, error translation, response serialization.

### What Happens When Persistence Concerns Leak Upward?

Imagine we want to optimize list operations with pagination. A naive approach:

```cpp
// BAD: Persistence detail leaks into domain
class BookService {
    struct Page<T> {
        std::vector<T> items;
        int totalCount;
    };
    
    Page<Book> getUserLibrary(
        const std::string& userId,
        int pageNumber,
        int pageSize
    ) {
        // Now service knows about pages (a database concept)
        // Testing the service requires setting up pagination
        // The domain is polluted with infrastructure
    }
};
```

The domain should not know about paging. But the presentation layer needs it. Solution: allow the presentation layer to skip the service layer for this simple operation:

```cpp
// BETTER: Let the presentation talk to the repository directly
// (or create a separate query service that is explicitly about retrieval)
class GetBooksQuery {  // Separate layer for read-only queries
    std::shared_ptr<BookRepository> repo;
    
    struct Page<T> {
        std::vector<T> items;
        int totalCount;
    };
    
    Page<Book> execute(const std::string& userId, int pageNum, int pageSize) {
        auto total = repo->countByUser(userId);
        auto books = repo->listByUser(userId, pageNum, pageSize);
        return {books, total};
    }
};
```

This is a **query handler** — separate from the service, explicitly about retrieving data with presentation-layer concerns (pagination), without polluting the domain service.

---

## 5.7 Anti-Patterns and How to Fix Them

### Anti-Pattern 1: Anemic Domain Model

**The symptom:** Domain classes have no methods, only properties. All logic is in service classes.

```cpp
// BAD
class User {
    std::string email;
    std::string passwordHash;
    // No methods
};

class UserService {
    bool validatePassword(const User& u, const std::string& pwd) {
        // Logic exiled to service
    }
};
```

**Why it fails:** Logic is separated from the data it operates on. Testing requires instantiating dummy objects. Refactoring is harder. The domain becomes a schema, not a model.

**Fix:** Move logic into the domain objects.

```cpp
// GOOD
class User {
    std::string email;
    std::string passwordHash;
    
public:
    bool validatePassword(const std::string& plaintext) const {
        return bcrypt::verify(plaintext, passwordHash);
    }
    
    void setPassword(const std::string& plaintext) {
        if (!meetsRequirements(plaintext)) {
            throw std::logic_error("Password too weak");
        }
        passwordHash = bcrypt::hash(plaintext);
    }
    
private:
    bool meetsRequirements(const std::string& pwd) const {
        return pwd.length() >= 8;
    }
};
```

### Anti-Pattern 2: Service-Only Architecture

**The symptom:** There are no domain classes, only services. `UserService.createUser`, `OrderService.placeOrder`. Entities are inert data bags passed between services.

**Why it fails:** The domain (the knowledge of the business) lives scattered across service classes instead of in cohesive domain objects. There is no clear sense of what a `User` *is*, only what services *do* to users.

**Fix:** Create rich domain objects. Services orchestrate, objects decide.

```cpp
// From service-only:
class OrderService {
    void placeOrder(const OrderData& data) {
        // 100 lines of business logic here
    }
};

// To domain-centric:
class Order {
    // The Order knows the rules for being placed
    void place() {
        if (items.empty()) {
            throw std::logic_error("Cannot place empty order");
        }
        if (totalPrice() > customer.creditLimit) {
            throw std::logic_error("Exceeds credit limit");
        }
        status = OrderStatus::PLACED;
        placedAt = clock->now();
    }
};

class OrderService {
    void placeOrder(const std::string& orderId) {
        auto order = repo->get(orderId);
        order->place();  // Delegate to domain
        repo->save(order);
    }
};
```

### Anti-Pattern 3: Layer-as-Pass-Through

**The symptom:** A service method does nothing but call a repository method.

```cpp
// BAD
class UserService {
    std::unique_ptr<UserRepository> repo;
    
    std::optional<User> getUser(const std::string& id) {
        return repo->getUser(id);  // No value added
    }
};
```

**Why it fails:** Indirection with no benefit. Adds cognitive overhead and call stack depth.

**Fix:** Delete the method. Let callers use the repository directly if there's no business logic.

```cpp
// GOOD: Remove the service entirely, or use it only for operations with logic
class UserService {
    std::unique_ptr<UserRepository> repo;
    
    // Keep this: has logic (validation, logging, side effects)
    User registerUser(const std::string& email, const std::string& pwd) {
        if (!isValidEmail(email)) throw std::logic_error("Invalid email");
        if (!isValidPassword(pwd)) throw std::logic_error("Weak password");
        return repo->createUser(email, hashPassword(pwd));
    }
    
    // Remove: no logic
    // std::optional<User> getUser(const std::string& id) { ... }
};
```

### Anti-Pattern 4: Leaking Persistence Concerns

**The symptom:** Domain classes reference database concepts directly. Lazy loading, eager/fetch strategies, ORM annotations on domain objects.

```cpp
// BAD: Database concern in domain
class User {
public:
    std::shared_ptr<std::vector<Order>> orders;  // Lazy-loaded?
    
    // Is this eager or lazy? Domain shouldn't care.
    void printOrders() {
        for (const auto& order : *orders) {  // May trigger a database query
            std::cout << order.id << "\n";
        }
    }
};
```

**Why it fails:** The domain becomes coupled to the persistence layer's choices. Testing is harder because executing a simple method might trigger database queries. Changing the loading strategy requires changing domain code.

**Fix:** Separate domain behavior from persistence strategy.

```cpp
// GOOD: Domain is clean
class User {
private:
    std::vector<Order> orders;  // Simple container, not lazy-loaded
    
public:
    void printOrders() const {
        for (const auto& order : orders) {
            std::cout << order.id << "\n";  // No database interaction
        }
    }
};

// Loading strategy lives in persistence/service layer
class UserService {
    std::unique_ptr<UserRepository> repo;
    std::unique_ptr<OrderRepository> orderRepo;
    
    User getUserWithOrders(const std::string& userId) {
        auto user = repo->getUser(userId);  // Fetch user
        auto orders = orderRepo->getOrdersByUser(userId);  // Fetch separately
        user.setOrders(orders);  // Populate after fetching
        return user;
    }
};
```

---

## 5.8 Tradeoffs

| Style | Pros | Cons | When to Use |
|---|---|---|---|
| **Strict Three-Layer** | Clear responsibility separation; easy to understand; enforces downward dependencies; testable layers | Call stack overhead; forces pass-through methods; difficulty handling pagination/filtering; can't skip layers even for trivial operations | Medium-to-large codebases where you can afford the overhead and value the clarity. |
| **Hexagonal** | Domain is central and independent; easy to swap implementations; clarifies what domain needs vs. what it provides; excellent for testing | More interfaces to design; can feel over-engineered for small systems; the inverting of dependency may be unfamiliar | Systems with multiple entry points (web + CLI + queue) or frequent framework changes. |
| **Clean Architecture (Rings)** | Generalizes the principle; very clear about what code goes where; scales well; philosophy fits large enterprise systems | More concentric rings = more interfaces; can feel heavy-handed for small apps; confusion about which ring something belongs in | Large codebases, teams, or systems where the policy/detail distinction really matters. |
| **Pragmatic Layers with Skipping** | Avoids useless traversal; requires judgment but results in cleaner code; still maintains separation of concerns | Requires discipline (must not skip upward); can become inconsistent if developers disagree on when skipping is allowed | Most production systems; a good default. |
| **No Layers (Single-file, small app)** | Simple, direct, no overhead | All concerns mixed; impossible to extend; brittle to test | Scripts, utilities, throw-away code, early prototypes. |

---

## 5.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Layers mean strict, always-downward call chains." | Layers are about *separation of concerns*. The call chain can skip layers if it makes sense; the rule is never go upward. |
| "A domain layer without any persistence code is always better." | A domain layer is better when there's enough domain logic to warrant it. A simple CRUD app with no complex rules doesn't need a rich domain; a query service is enough. |
| "If I'm using an ORM, I need a repository pattern." | The repository pattern is about *abstraction* — hiding whether you use SQL, NoSQL, or in-memory storage. A good ORM can be the repository; you don't need a layer on top. |
| "Every service method must call exactly one repository method." | Service methods should perform a *unit of work*. If that's one call, great; if it's five calls and two domain calculations, also fine. Indirection doesn't create value. |
| "Hexagonal architecture means six boundaries and lots of interfaces." | Hexagonal just means the domain is in the center and adapters plug into it. You implement as many or as few adapters as your system needs. |
| "Clean Architecture means every piece of code fits neatly into one ring." | No system is that clean. Exceptions exist. The principle is: dependencies should not flow from inner rings to outer rings. Where code *logically* belongs is a judgment call. |

---

## 5.10 Exercises

1. **Sketch your current system as layers.** Draw boxes for your presentation, domain, and persistence layers. Draw arrows showing how data flows. Do any arrows point upward? If yes, refactor one to point only downward.

2. **Find a pass-through layer.** In a service you work on, find a service method that does nothing but call a repository method. Propose how to remove it — either delete the method or add business logic to justify its existence.

3. **Reverse-engineer hexagonal architecture.** Take an existing codebase. Identify the domain core, the inbound ports (what the domain provides), and outbound ports (what it requires). Are the ports well-defined, or are they tangled with implementation details?

4. **Price the indirection.** Implement the same operation two ways: one with strict three-layer traversal, one skipping a layer. Write a benchmark that makes 10,000 calls to each. How much does the extra function call cost on your hardware?

5. **Find and fix an anemic domain.** Locate a domain class in your codebase that is mostly just data. Identify one method that belongs on it (currently in a service). Move the method. Did it improve the design?

6. **Design a BookShelf read model.** The BookShelf example in this chapter stores books. Design a separate read model for "top 10 most-read books this month" — what layer does it live in? Why?

---

## 5.11 Summary

Layered architecture is the most durable organizing principle for business applications. The three-layer model (Presentation, Domain, Persistence) separates concerns and makes code easier to reason about. Hexagonal architecture clarifies that the domain is central and independent; adapters plug in around it. Clean Architecture generalizes this principle across finer-grained levels of abstraction.

Strict enforcement of layers creates overhead. Pragmatic systems allow downward shortcuts while enforcing the core rule: never depend upward. The real value of layering is in *separation of concerns*, not in mechanical compliance. A domain layer is worth the complexity only when it contains real domain logic; a thin CRUD app doesn't need one. Anemic models, service-only architectures, and layer-as-pass-through are warning signs that the layering has become ceremony instead of clarity.

Layering earns its cost when it scales the team's ability to work independently on the presentation, domain, and persistence layers. When layers add indirection without clarity, they are a smell. Use your judgment about which layers matter in your system, but enforce the fundamental rule: dependencies must point downward, never up.

---

**[← Previous: Separation of Concerns](04-separation-of-concerns.md)** · **[↑ Part 4](README.md)** · **[Next: Controllers, Services, Repositories →](06-controllers-services-repositories.md)**
