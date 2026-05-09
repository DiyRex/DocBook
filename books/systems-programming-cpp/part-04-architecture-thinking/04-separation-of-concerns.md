# Chapter 36 — Separation of Concerns

A class has a reason to change whenever a business requirement changes. A class with three reasons to change has three concerns tangled together. Almost every best practice in software architecture — from the Single Responsibility Principle, to layered design, to microservices — reduces to one thing: finding the seams where concerns separate, and pulling them apart before they become expensive to untangle.

This chapter teaches you to *see* those seams.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define a "concern" precisely as a reason for change, and recognize why the unit of change matters more than the unit of code.
2. Understand why the Single Responsibility Principle is about stakeholders, not function counts.
3. Identify cross-cutting concerns (logging, auth, tracing, metrics) and explain why ordinary encapsulation fails to contain them.
4. Apply the separation principle at different scales — functions, classes, modules, services — and recognize that the shape changes with scale, but the principle does not.
5. Refactor tangled code by identifying concerns and introducing thin orchestrators.
6. Judge when separation is worth its cost, and when it is premature division.
7. Recognize the most common misconception: that separation is always virtuous.

---

## 36.1 What Is a Concern?

Start with this working definition:

> **A concern is something that a business stakeholder, or a development team, or a system operator, cares about and wants to change independently of other things.**

Some examples of concerns:

- **Validation:** the rules for whether data is well-formed. Business owns this. When the rules change, the validator changes.
- **Persistence:** how objects get stored and retrieved. The database team owns this. When you need to switch from PostgreSQL to DynamoDB, the persistence layer changes; the business logic should not.
- **Presentation:** how data is formatted and displayed. The frontend team owns this. When you change the API response format, the presentation layer changes; the business logic should not.
- **Authorization:** who is allowed to do what. Security and product own this. When the permission model changes, auth logic changes; the billing logic should not.
- **Logging:** what events get recorded for debugging and monitoring. Operations owns this. When you want JSON logs instead of plaintext, or when you add a new diagnostic field, logging changes; the business logic should not.
- **Payment processing:** how money moves. Accounting and finance own this. When the payment provider changes, payment code changes; order logic should not.

Each concern has a **stakeholder** — the person or team who asks for changes to that concern. The principle of separation is simple: if two concerns have different stakeholders, they should be in different pieces of code.

Why? Because **change latency**. If validation and payment processing are tangled in a single class, then every time payment processing changes, the validation code gets re-deployed, re-tested, and put at risk of regression. If they are separate, payment changes do not touch validation.

At scale, this compounds. A 100-person organization with tangled code pays a tax every time any concern changes: the other concerns must be re-tested, integration tests must run, the entire service must be re-deployed. A 100-person organization with separated concerns can have five teams working on five concerns in parallel, with minimal blocking.

---

## 36.2 The Single Responsibility Principle, Reconsidered

The Single Responsibility Principle (SRP) is often misread as "a class should do one thing." That is surface-level. The real meaning:

> **A class should have only one reason to change.**

A reason to change is not "do one thing." A reason to change is "one stakeholder asked for it."

This distinction matters. Consider two examples.

**Example 1: A class that does one thing, but has two reasons to change.**

```cpp
// BAD: two reasons to change (validation rule, data format)
class User {
public:
    User(std::string_view name, std::string_view email) {
        if (name.empty()) {
            throw std::invalid_argument("name required");
        }
        if (email.find('@') == std::string::npos) {
            throw std::invalid_argument("email invalid");
        }
        this->name = name;
        this->email = email;
    }

    std::string name;
    std::string email;

    std::string toJSON() const {
        return R"({"name":")" + name + R"(","email":")" + email + R"("})";
    }
};
```

This class "does" one simple thing: represent a user. But it has **two** reasons to change:

1. The product team says "we now require a middle name." That touches the fields and the JSON.
2. The frontend team says "we want XML instead of JSON." That touches `toJSON()`.

These are separate stakeholders. Tangling them violates SRP.

**Example 2: A class that does many things, but has one reason to change.**

```cpp
// GOOD: one reason to change (HTTP routing)
class UserHandler {
public:
    std::string handleGetUser(int id) {
        // construct SQL query
        // execute it
        // extract the row
        // convert to JSON
        // return
        return database->queryOne(
            R"(SELECT name, email FROM users WHERE id=?)", id
        ).toJSON();
    }
};
```

This looks like it "does" too much: query building, database access, format conversion. But there is **one** reason to change: the HTTP API changed. The front-to-back implementation details are all internal; they are not separate concerns, they are steps in the same workflow.

This is not a violation of SRP. This is a single orchestration of multiple concerns.

**The disambiguator:** ask "who asks for this change?" If multiple stakeholders ask for different changes at different times, separate the code. If one stakeholder owns the entire workflow, keep it together.

---

## 36.3 Cross-Cutting Concerns

Most concerns live in one place: validation in the validator, persistence in the repository, payment in the payment service. But some concerns are **cross-cutting** — they touch many pieces of code, because they are needed everywhere.

Examples:

- **Logging:** every function might need to log. You cannot put all logging in one class.
- **Authorization:** every API endpoint needs to check permissions. You cannot put all auth checks in one place.
- **Metrics:** every operation needs to record latency, error rates, etc. Scattered everywhere.
- **Tracing:** to debug a request across microservices, every service needs to propagate a trace ID. Scattered everywhere.
- **Error handling and recovery:** every network call might fail and need retry logic. Scattered everywhere.

If you tried to keep these concerns separate using ordinary encapsulation, you would need one of these approaches:

**Approach 1: Pass them as parameters.**

```cpp
// BAD: boilerplate explosion
class OrderService {
public:
    void processOrder(
        const Order& order,
        Logger& logger,
        AuthCheck& auth,
        MetricsCollector& metrics,
        TraceContext& trace,
        RetryPolicy& retry
    ) {
        // 6 parameters, and that's before the real parameters
        // every function in the call chain needs all 6
        // impossible to scale
    }
};
```

**Approach 2: Global singletons.**

```cpp
// WORKS, but fragile
class OrderService {
public:
    void processOrder(const Order& order) {
        Logger::instance().info("processing order");
        if (!AuthCheck::instance().allowed(order)) {
            MetricsCollector::instance().record("auth_denied");
            throw std::runtime_error("unauthorized");
        }
        // ... logic ...
    }
};
```

This works but is fragile: global state is testable only by resetting the singleton between tests, which is error-prone and fragile.

**Approach 3: Middleware / Decorators / Aspects / Interceptors.**

```cpp
// GOOD: separates concerns without boilerplate
class OrderService {
public:
    void processOrder(const Order& order) {
        // just the business logic
    }
};

class LoggingDecorator {
public:
    void processOrder(const Order& order) {
        Logger::info("starting");
        try {
            inner->processOrder(order);
            Logger::info("success");
        } catch (...) {
            Logger::error("failed");
            throw;
        }
    }
private:
    std::shared_ptr<OrderService> inner;
};

class AuthDecorator {
public:
    void processOrder(const Order& order) {
        if (!auth->allowed(order)) {
            MetricsCollector::record("auth_denied");
            throw std::runtime_error("unauthorized");
        }
        inner->processOrder(order);
    }
private:
    std::shared_ptr<OrderService> inner;
};
```

Each decorator wraps the service and handles one cross-cutting concern. The decorators compose: you can build `AuthDecorator(LoggingDecorator(MetricsDecorator(OrderService())))`, and each decorator sees the exact same interface.

This is the pattern behind:
- HTTP middleware (Express, FastAPI, Go's `http.Handler` chain)
- Aspect-Oriented Programming (AspectJ, Spring AOP)
- Python decorators
- Go's `http.Handler` wrapper pattern
- C++ template mixins

The point: **when a concern touches many pieces of code, do not try to encapsulate it in one place. Instead, use a pattern that lets you inject it at the boundary where it applies.**

---

## 36.4 Concerns at Different Scales

The shape of separation changes with scale, but the principle stays the same.

### Function scale

A function should do one thing: read input, do one logical operation, write output. If a function has nested loops processing different data structures, or if-else trees checking unrelated conditions, it probably has multiple concerns.

```cpp
// BAD: two concerns (user lookup, permission check)
std::optional<User> getUserIfAllowed(int id, const Role& role) {
    auto result = database->query("SELECT * FROM users WHERE id = ?", id);
    if (!result) return std::nullopt;
    
    User u = result.value();
    
    if (role == Role::Admin) return u;
    if (role == Role::Manager && u.department == getCurrentDept()) return u;
    return std::nullopt;  // tangled!
}

// GOOD: separated
std::optional<User> getUser(int id) {
    return database->query("SELECT * FROM users WHERE id = ?", id);
}

bool isAllowed(const User& u, const Role& role) {
    if (role == Role::Admin) return true;
    if (role == Role::Manager && u.department == getCurrentDept()) return true;
    return false;
}

// caller:
auto user = getUser(id);
if (user && isAllowed(user.value(), role)) {
    return user;
}
```

### Class scale

A class should represent one entity or one coherent piece of responsibility. If it has methods that operate on unrelated parts of its internal state, it probably has multiple concerns.

```cpp
// BAD: three concerns (data storage, validation, serialization)
class Order {
public:
    Order(int id, const std::string& items, double total)
        : id_(id), items_(items), total_(total) {}

    bool validate() {
        if (items_.empty()) return false;
        if (total_ < 0) return false;
        return true;
    }

    std::string toJSON() const {
        return R"({"id":)" + std::to_string(id_) + R"(,...})";
    }

    std::string toXML() const {
        return "<Order><id>" + std::to_string(id_) + "</id>...</Order>";
    }

    void save(Database& db) {
        if (!validate()) throw std::runtime_error("invalid");
        db.insert("orders", /* ... */);
    }

private:
    int id_;
    std::string items_;
    double total_;
};
```

This has **four** concerns: representation, validation, serialization (two formats!), persistence. Different stakeholders care about each. Separate them:

```cpp
// Core entity: just data
class Order {
public:
    Order(int id, const std::string& items, double total)
        : id_(id), items_(items), total_(total) {}

    int id() const { return id_; }
    const std::string& items() const { return items_; }
    double total() const { return total_; }

private:
    int id_;
    std::string items_;
    double total_;
};

// Validation: one concern
class OrderValidator {
public:
    bool isValid(const Order& o) const {
        return !o.items().empty() && o.total() >= 0;
    }
};

// Serialization: one concern per format
class OrderJSONSerializer {
public:
    std::string serialize(const Order& o) const {
        return R"({"id":)" + std::to_string(o.id()) + R"(,...})";
    }
};

// Persistence: one concern
class OrderRepository {
public:
    void save(const Order& o, Database& db) const {
        if (!validator.isValid(o)) {
            throw std::runtime_error("invalid");
        }
        db.insert("orders", /* ... */);
    }

private:
    OrderValidator validator;
};
```

Now, when the frontend asks for XML, only the serializer changes. When the database team switches databases, only the repository changes. When validation rules change, only the validator changes.

### Module scale

A module (a collection of files or a namespace) should provide a coherent capability. If it has sub-concerns that evolve independently, split it.

A "User Management" module might naturally contain:
- User entity definition
- User repository (CRUD)
- User validator (password rules, email rules)
- User notifier (send emails)

But if **notification** changes independently of **CRUD**, they should be separate modules: `user` and `notification`. The `user` module uses the `notification` module; they are not tangled.

### Service scale

A microservice should own one business capability. "Order Service" should handle everything about orders: creation, validation, persistence, shipping integration. But if **payment** is a separate capability with separate stakeholders (accounting, finance), it should be a separate service.

The principle is the same at every scale: **separate code that changes for different reasons.**

---

## 36.5 Worked Example: Refactoring an OrderProcessor

Here is a realistic class that violates SRP badly:

```cpp
class OrderProcessor {
public:
    void process(const Order& order) {
        // Concern 1: Validation
        if (order.items.empty()) {
            throw std::invalid_argument("order cannot be empty");
        }
        if (order.total <= 0) {
            throw std::invalid_argument("total must be positive");
        }

        // Concern 2: Persistence
        database->insertOrder(order);
        
        // Concern 3: Payment processing
        try {
            paymentGateway->charge(order.customerId, order.total);
        } catch (const PaymentException& e) {
            database->updateOrderStatus(order.id, "payment_failed");
            throw;
        }

        // Concern 4: Notification
        emailService->send(
            order.customerEmail,
            "Your order has been processed",
            "Total: $" + std::to_string(order.total)
        );

        // Concern 5: Logging
        logger->info("Order " + std::to_string(order.id) + " processed");
    }
};
```

This has **five** stakeholders:
1. Product (validation rules)
2. Database team (persistence)
3. Finance team (payment)
4. Customer relations (notification)
5. Operations (logging)

The class has five reasons to change. When the payment provider changes, the validation code gets re-deployed. When the email template changes, the payment code is affected. This is fragile.

**Step 1: Extract the stakeholder concerns into separate classes.**

```cpp
class OrderValidator {
public:
    void validate(const Order& order) const {
        if (order.items.empty()) {
            throw std::invalid_argument("order cannot be empty");
        }
        if (order.total <= 0) {
            throw std::invalid_argument("total must be positive");
        }
    }
};

class OrderRepository {
public:
    void save(const Order& order) const {
        database->insertOrder(order);
    }

    void updateStatus(int orderId, const std::string& status) const {
        database->updateOrderStatus(orderId, status);
    }
};

class PaymentProcessor {
public:
    void charge(int customerId, double amount) const {
        paymentGateway->charge(customerId, amount);
    }
};

class OrderNotifier {
public:
    void notifyOrderProcessed(const Order& order) const {
        emailService->send(
            order.customerEmail,
            "Your order has been processed",
            "Total: $" + std::to_string(order.total)
        );
    }
};
```

**Step 2: Create a thin orchestrator.**

```cpp
class OrderProcessor {
public:
    OrderProcessor(
        std::shared_ptr<OrderValidator> validator,
        std::shared_ptr<OrderRepository> repository,
        std::shared_ptr<PaymentProcessor> payment,
        std::shared_ptr<OrderNotifier> notifier,
        std::shared_ptr<Logger> logger
    )
        : validator_(validator),
          repository_(repository),
          payment_(payment),
          notifier_(notifier),
          logger_(logger) {}

    void process(const Order& order) {
        // Orchestrate the concerns
        validator_->validate(order);
        
        repository_->save(order);
        logger_->info("Order saved");
        
        try {
            payment_->charge(order.customerId, order.total);
            logger_->info("Payment successful");
        } catch (const std::exception& e) {
            repository_->updateStatus(order.id, "payment_failed");
            logger_->error("Payment failed: " + std::string(e.what()));
            throw;
        }

        notifier_->notifyOrderProcessed(order);
        logger_->info("Notification sent");
    }

private:
    std::shared_ptr<OrderValidator> validator_;
    std::shared_ptr<OrderRepository> repository_;
    std::shared_ptr<PaymentProcessor> payment_;
    std::shared_ptr<OrderNotifier> notifier_;
    std::shared_ptr<Logger> logger_;
};
```

Now each concern lives in its own place. When the payment provider changes, only `PaymentProcessor` changes. When email templates change, only `OrderNotifier` changes. The `OrderProcessor` is a thin coordinator that knows the workflow: validate, persist, charge, notify. Nothing more.

**Testing cost comparison:**

*Before:*
```cpp
// Testing just validation requires mocking database, payment, email, logging
OrderProcessor processor(
    realValidator, mockDatabase, mockPayment, mockEmail, mockLogger
);
processor.process(order);
ASSERT_EQ(mockDatabase.callCount(), 1);
ASSERT_EQ(mockPayment.callCount(), 0);  // failed before charge
```

*After:*
```cpp
// Testing just validation: real validator, no mocks needed
OrderValidator validator;
EXPECT_THROW(validator.validate(badOrder), std::invalid_argument);

// Testing payment only: real payment processor, real dependencies
PaymentProcessor payment(realGateway);
EXPECT_THROW(payment.charge(customerId, -100), InvalidAmount);

// Testing orchestration: real components, focused test
OrderProcessor processor(
    validator, repository, payment, notifier, logger
);
processor.process(validOrder);
```

The separation is not just cleaner; it is **testable**. Each concern can be tested in isolation. Integration tests are smaller and more focused.

---

## 36.6 When To Stop

Separation of concerns is not a virtue by itself. Too much separation costs as much as too little.

**Cost of over-separation:**

- **Boilerplate:** more classes means more constructors, more dependency injection, more factories.
- **Indirection:** to get something done, you call through multiple layers.
- **Coordination complexity:** coordinating five services is harder than one monolith.
- **Subtle bugs at boundaries:** bugs that span layers are harder to diagnose than bugs in one place.

**Cost of under-separation:**

- **Long change cycles:** change one concern, re-test everything.
- **Tangled tests:** tests must mock out many unrelated concerns.
- **Team friction:** different teams step on each other's changes.

**The heuristic: separate a concern when:**

1. **It has a different stakeholder.** If Product, Database Team, and Finance all have input, separate them.
2. **It evolves independently.** If validation rules change every sprint but persistence never changes, keep them separate.
3. **It is complex enough to be worth isolating.** A three-line concern does not deserve its own class.

**The rule of three:** if you see the same concern appear in three different places, and the places are under different ownership, extract it. The rule keeps you from premature extraction (one occurrence) and from leaving obvious duplication (four occurrences).

```cpp
// One occurrence: leave it inline
void handleUserCreation(const User& user) {
    if (user.email.find('@') == std::string::npos) {
        throw std::invalid_argument("invalid email");
    }
}

// Two occurrences: still inline
void handleUserUpdate(const User& user) {
    if (user.email.find('@') == std::string::npos) {
        throw std::invalid_argument("invalid email");
    }
}

// Three occurrences + different files / teams: extract
void handlePasswordReset(const User& user) {
    if (user.email.find('@') == std::string::npos) {
        throw std::invalid_argument("invalid email");
    }
}

// NOW extract:
class EmailValidator {
    void validate(std::string_view email) const {
        if (email.find('@') == std::string::npos) {
            throw std::invalid_argument("invalid email");
        }
    }
};
```

---

## 36.7 Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **Single monolithic class** | Simple to understand; low indirection; easy to debug | Hard to test; tangled concerns; long change cycles; team contention | Small domains; single stakeholder; code that rarely changes |
| **Separated by concern** | Testable in isolation; independent change cycles; team independence | More boilerplate; more files; more indirection; bugs at boundaries | Domains with multiple stakeholders; code that evolves frequently |
| **Thin orchestrator + collaborators** | Separation with clarity; easy to see the workflow | Requires discipline to keep orchestrator thin; can hide complexity | Most business logic; especially where concerns interact in sequence |
| **Middleware / decorators** | Clean separation of cross-cutting concerns; no parameter drilling | Harder to debug (stack of wrappers); silent failures from wrapper composition | Logging, auth, tracing, metrics; anything that wraps a core operation |
| **Global singletons** | Simple API; no boilerplate | Hard to test; implicit dependencies; fragile state sharing | Only for truly global things (config, logging) and with extreme caution |
| **Dependency injection** | Testable; swappable; explicit dependencies | Verbose; requires a container or factory; setup cost | Production systems; systems that need testing; systems with multiple implementations |

---

## 36.8 Common Misconceptions

**Misconception 1: "Separation means smaller classes."**

Reality: Separation means **fewer reasons to change**, not fewer lines. A class with 500 lines that handles one concern is better designed than a class with 50 lines that handles five concerns. (Though 500 lines is a red flag for other reasons.)

**Misconception 2: "I should separate every concern immediately."**

Reality: Premature separation adds cost without benefit. **Let concerns coexist until you have a reason to split them** — different stakeholders, independent evolution, test complexity. The rule of three exists for a reason.

**Misconception 3: "Separation of concerns is the same as layered architecture."**

Reality: Layered architecture (presentation, business, data) is *one way* to separate concerns *by function*. But horizontal separation (validation, persistence, payment) also separates concerns *by business capability*. Use the structure that matches your stakeholders.

**Misconception 4: "A well-separated codebase never needs refactoring."**

Reality: Good separation reduces refactoring burden, but does not eliminate it. Code still accumulates cruft. Stakeholder interests shift. Today's clean boundary becomes tomorrow's awkward seam. Refactoring is ongoing.

**Misconception 5: "Dependencies on cross-cutting concerns (logging, metrics) break separation."**

Reality: Cross-cutting concerns are exempt. Every class can depend on `Logger` or `MetricsCollector` without violating SRP. The key is that these concerns are **infra**, not **domain logic**. If domain logic in the validator depends on domain logic in the payment processor, that is bad. If both depend on logging, that is fine.

**Misconception 6: "Separation is about reusability."**

Reality: Separation is primarily about *independent change*. Reusability is a side effect. A repository might be used in multiple services, sure. But it is separated from the validator primarily because they have different stakeholders, not because it is reusable.

---

## 36.9 Exercises

1. **Identify the stakeholders.** Take a class from a codebase you know. For each method, ask: "who would ask for changes to this method?" If different people answer with different titles, you probably have multiple concerns. Sketch how you would separate them.

2. **The three-concern refactor.** Write a class that handles user registration: validation, persistence, and notification. All three tangled together (like the OrderProcessor example). Then refactor it by extracting collaborators. Compare the test cost before and after. Which is easier to test?

3. **Cross-cutting in your code.** Find a cross-cutting concern in your codebase — logging, metrics, error handling, or auth. Measure how many places it appears. Estimate the boilerplate if you had to pass it as a parameter everywhere. Argue for or against extracting it into a decorator.

4. **The false boundary.** Find a class that is separated into two pieces (e.g., `UserService` and `UserRepository`). Trace a typical workflow through both. Do they have genuinely different stakeholders, or would they be clearer if merged? Justify your answer with a business example.

5. **Premature separation.** Find a piece of code that is separated into multiple classes or modules, but appears in only one place. Ask: does each piece have a different stakeholder? Is there evidence that they will evolve independently? If not, make the case for merging them back.

6. **Scale the principle.** In a microservices codebase, identify two services. For each, list its concerns. Do the concerns have clear boundaries? Would splitting one service into two improve team independence? What would break?

---

## 36.10 Summary

A concern is a reason to change. When a class has multiple reasons to change, it has multiple concerns, and they should be separated. The Single Responsibility Principle is not about "do one thing"; it is about "have one reason to change." Separate when concerns have different stakeholders, evolve independently, or make tests expensive. Do not separate when the concern is too small, or when concerns naturally form a single workflow. Cross-cutting concerns (logging, auth, metrics) require special techniques — decorators, middleware, aspects — because they touch many places. The skill is **matching the separation strategy to the stakeholder structure and change frequency of your codebase.**

---

> **[← Previous: Chapter 35 — Coupling vs Cohesion](03-coupling-vs-cohesion.md)** · **[↑ Part 4](README.md)** · **[Next: Chapter 37 — Layered Architecture →](05-layered-architecture.md)**
