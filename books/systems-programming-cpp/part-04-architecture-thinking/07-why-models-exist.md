# Chapter 39 — Why Models Exist

A model is the stable name your code agrees on for a real-world thing. It is not a row in a database. It is not a JSON response from an API. It is the concept, with identity, state, behavior, and rules, that your code speaks about. When the business says "a customer can cancel their subscription," the model is what decides what happens next. When a database stores that subscription across three tables because of normalization, the model abstracts that complexity away. A model is how your domain thinks; a data model is how your persistence layer stores. They are not the same thing, and confusing them leads to architectures that break under change.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish a domain model from a data model and understand why they differ.
2. Identify what a good model must capture: identity, state, behavior, and invariants.
3. Recognize anemic models (data with getters/setters) and rich models (behavior on the object).
4. Explain why ORMs blur the line between domain and data models, and what problems result.
5. Use DTOs correctly as boundary-crossing shapes, distinct from domain models.
6. Redesign a service-based system by moving behavior into rich models.
7. Decide whether to build rich models or use simpler data holders, given your constraints.

---

## 39.1 Domain Models vs. Data Models

Two concepts live under the name "model," and they are not interchangeable.

**A data model describes how state is stored.** It is the shape of the persistence layer: tables, columns, relationships, indexes. It answers: "What does a database row look like? How are entities normalized? What can be queried efficiently?"

**A domain model describes what concepts the business cares about.** It is the shape of the problem: entities, value objects, aggregates, and the rules that govern them. It answers: "What is a subscription? What makes a subscription valid? What can you do with a subscription?"

The data model is shaped by concerns of persistence: normalization to reduce redundancy, indexes for performance, foreign keys for referential integrity, schema migration. The domain model is shaped by concerns of correctness: rules that must always be true, operations that make semantic sense, boundaries that prevent invalid state.

**They often differ in shape.**

A `User` in the domain model might be a single object with a name, email, profile. The data model might split this across three tables: `users` (id, email), `user_profiles` (user_id, name, bio), and `user_settings` (user_id, notifications_enabled, theme). Normalization demands this split. But in your code, you want to think of a single `User` with all these properties.

A `Subscription` in the domain model has rules: cannot cancel during the first month, auto-renews unless paused, only one subscription per customer at a time. The data model stores this as a row in `subscriptions` (id, customer_id, status, created_at, renewal_date) and a row in `subscription_rules` (subscription_id, rule_name, rule_data). The rules are implicit constraints checked by application logic, not enforced by schema.

**The mismatch is not a bug; it is necessary.** The data model optimizes for storage and querying. The domain model optimizes for reasoning about correctness.

The translation between them happens at a boundary: the repository, the ORM, the service that reads from the database and constructs domain objects.

---

## 39.2 What a Good Model Captures

A domain model is not merely data. It is a contract with the code that uses it. The contract includes four things: identity, state, behavior, and invariants.

### Identity

An object has identity if you can say "this is the same one as before" even if its state changed. A `Subscription` with id 42 is always that subscription, even after its status changes from "active" to "canceled."

```cpp
class Subscription {
private:
    const uint64_t id_;  // immutable; defines identity
    
public:
    uint64_t id() const { return id_; }
    
    Subscription(uint64_t id) : id_(id) {}
};
```

Identity is usually immutable. A subscription's id does not change. A customer's id does not change. The object's state changes; its identity does not.

### State

State is the set of values that describe the object at a moment in time: active, paused, canceled; created at this date; renews on that date.

```cpp
enum class SubscriptionStatus { Active, Paused, Canceled };

class Subscription {
private:
    const uint64_t id_;
    SubscriptionStatus status_;
    std::chrono::system_clock::time_point created_at_;
    std::chrono::system_clock::time_point renewal_date_;
    
public:
    SubscriptionStatus status() const { return status_; }
    std::chrono::system_clock::time_point createdAt() const { return created_at_; }
    std::chrono::system_clock::time_point renewalDate() const { return renewal_date_; }
};
```

State is mutable; it changes as the object is used. But not all changes are valid. A subscription cannot have a renewal date in the past. A subscription's status must be one of the enum values, never garbage.

### Behavior

Behavior is the set of operations you can perform on the object. These operations often change state, and they enforce the rules of the domain.

```cpp
class Subscription {
public:
    // Try to pause this subscription.
    // Returns true if it succeeds, false if it fails (e.g., already canceled).
    bool pause() {
        if (status_ == SubscriptionStatus::Canceled) {
            return false;  // cannot pause a canceled subscription
        }
        status_ = SubscriptionStatus::Paused;
        return true;
    }
    
    // Try to resume a paused subscription.
    bool resume() {
        if (status_ != SubscriptionStatus::Paused) {
            return false;
        }
        status_ = SubscriptionStatus::Active;
        return true;
    }
    
    // Cancel the subscription permanently.
    void cancel() {
        status_ = SubscriptionStatus::Canceled;
        // Optionally: record the cancellation date, send event, etc.
    }
};
```

Behavior is not just state mutation. It is the translation of domain operations into code. When you `pause()` a subscription, you are not just setting a field; you are asserting that this transition is valid at this moment.

### Invariants

An invariant is a predicate that must be true after every public operation. For a subscription: the status is always valid; the renewal date is never in the past (unless the subscription is canceled); only one subscription per customer is active at a time.

The last invariant is interesting: it cannot be enforced by a single `Subscription` object. It is an invariant of the aggregate: the `Customer` and all their `Subscription`s together must satisfy it. This is why we group models into aggregates.

```cpp
class Customer {
private:
    const uint64_t id_;
    std::vector<std::shared_ptr<Subscription>> subscriptions_;
    
public:
    // Invariant: at most one active subscription
    bool addSubscription(std::shared_ptr<Subscription> sub) {
        // Check invariant
        for (const auto& existing : subscriptions_) {
            if (existing->status() == SubscriptionStatus::Active) {
                return false;  // already have an active subscription
            }
        }
        subscriptions_.push_back(sub);
        return true;
    }
};
```

---

## 39.3 Anemic Models and Why They Fail

An anemic model is an object that holds data but has no behavior. It is just getters, setters, and fields.

```cpp
class Anemic_Subscription {
public:
    uint64_t id;
    std::string customer_id;
    std::string status;  // "active", "paused", "canceled" — no enum, no validation
    std::chrono::system_clock::time_point created_at;
    std::chrono::system_clock::time_point renewal_date;
    
    std::string getStatus() const { return status; }
    void setStatus(const std::string& s) { status = s; }
    // ... getters and setters for every field
};
```

The rules live outside the model, in a service:

```cpp
class SubscriptionService {
public:
    bool cancel(Anemic_Subscription& sub) {
        if (sub.getStatus() == "canceled") {
            return false;  // already canceled
        }
        sub.setStatus("canceled");
        return true;
    }
    
    bool pause(Anemic_Subscription& sub) {
        if (sub.getStatus() == "canceled") {
            return false;  // cannot pause a canceled subscription
        }
        sub.setStatus("paused");
        return true;
    }
};
```

This looks clean at first: the service owns the logic, the model is simple data. In reality, it is a disaster.

**Problem 1: Rules scatter.** The rule "cannot pause a canceled subscription" lives in `SubscriptionService::pause()`. But there are other places that might transition the subscription's status: an admin endpoint, a cron job that processes renewals, an event handler from another service. Each place must duplicate the rule. A new rule gets added later; it is forgotten in one of the places; a bug appears.

**Problem 2: The invariant is not enforced.** A caller can do this:

```cpp
Anemic_Subscription sub;
sub.status = "invalid_status";  // No compiler error; no runtime error. The invariant is broken.
```

The model does not protect itself. Correctness depends on everyone using the service. If someone writes code that directly modifies the model (or if the service has a bug), the invariant breaks and no one will find out until a downstream system breaks.

**Problem 3: Refactoring breaks everything.** Suppose the rule changes: "you can cancel a subscription anytime, but if you cancel in the first month, you pay a penalty." Now the `cancel()` method needs to return more information: whether a penalty was charged, how much. You change the signature of `SubscriptionService::cancel()`. Every caller must be updated. With an anemic model, this ripples through the whole codebase.

**Problem 4: Behavior and state drift apart.** The model is a class in the codebase, but the rules are scattered across services. A month later, someone adds a field to the model: `refund_date`. The services do not know about it. The rule "refunds are calculated from refund_date" lives in the old notes, not in code. Technical debt accumulates.

Anemic models are common in Java and C# codebases that use ORMs without careful design, or in systems where DTOs are mistaken for domain models. They appear to work at first, then become a maintenance nightmare.

---

## 39.4 Rich Models and How They Scale

A rich model puts behavior on the object itself. The rules are enforced by the model's methods, not by external services.

```cpp
class Subscription {
private:
    const uint64_t id_;
    const uint64_t customer_id_;
    SubscriptionStatus status_;
    std::chrono::system_clock::time_point created_at_;
    std::chrono::system_clock::time_point renewal_date_;
    std::optional<std::chrono::system_clock::time_point> canceled_at_;
    
    // Invariant: status is always valid; renewal_date is in the future unless canceled
    void validate() const {
        assert(status_ != SubscriptionStatus::Invalid);
        if (status_ != SubscriptionStatus::Canceled) {
            assert(renewal_date_ > now());
        }
    }
    
public:
    Subscription(uint64_t id, uint64_t customer_id, 
                 std::chrono::system_clock::time_point created_at)
        : id_(id), customer_id_(customer_id), 
          status_(SubscriptionStatus::Active),
          created_at_(created_at),
          renewal_date_(created_at + std::chrono::days(30)) {
        validate();
    }
    
    // Operations return a result: success or failure reason
    struct CancelResult { bool success; std::string reason; };
    
    CancelResult cancel() {
        if (status_ == SubscriptionStatus::Canceled) {
            return { false, "already canceled" };
        }
        
        // Rule: cannot cancel in the first month
        auto now = std::chrono::system_clock::now();
        if (now < created_at_ + std::chrono::days(30)) {
            return { false, "cannot cancel in first month" };
        }
        
        status_ = SubscriptionStatus::Canceled;
        canceled_at_ = now;
        validate();
        return { true, "" };
    }
    
    struct PauseResult { bool success; std::string reason; };
    
    PauseResult pause() {
        if (status_ == SubscriptionStatus::Canceled) {
            return { false, "cannot pause a canceled subscription" };
        }
        if (status_ == SubscriptionStatus::Paused) {
            return { false, "already paused" };
        }
        
        status_ = SubscriptionStatus::Paused;
        validate();
        return { true, "" };
    }
    
    struct ResumeResult { bool success; std::string reason; };
    
    ResumeResult resume() {
        if (status_ != SubscriptionStatus::Paused) {
            return { false, "only paused subscriptions can be resumed" };
        }
        
        status_ = SubscriptionStatus::Active;
        validate();
        return { true, "" };
    }
    
    // Accessors: read-only, no setters
    uint64_t id() const { return id_; }
    uint64_t customerId() const { return customer_id_; }
    SubscriptionStatus status() const { return status_; }
    std::chrono::system_clock::time_point createdAt() const { return created_at_; }
    std::chrono::system_clock::time_point renewalDate() const { return renewal_date_; }
    bool isCanceled() const { return status_ == SubscriptionStatus::Canceled; }
};
```

Now the caller uses the model like this:

```cpp
Subscription sub(42, 100, now);

auto result = sub.pause();
if (result.success) {
    std::cout << "Paused\n";
} else {
    std::cout << "Failed: " << result.reason << "\n";
}
```

The advantages:

1. **Rules are enforced in one place.** The logic for "cannot cancel in the first month" is in `Subscription::cancel()`. Every code path that cancels a subscription goes through this method. If the rule changes, you change it once.

2. **The invariant is guaranteed.** A caller cannot write `sub.status_ = "invalid";` because `status_` is private. The only way to change the status is through the public methods, which validate before changing.

3. **Behavior matches the domain.** Callers think in terms of actions: `sub.pause()`, `sub.cancel()`, `sub.resume()`. Not in terms of fields: `setStatus("paused")`. The code reads like the domain problem.

4. **Refactoring is local.** If the cancel rule changes, you change the `cancel()` method. All callers benefit automatically. No scatter-gun refactoring across the codebase.

5. **Errors are explicit.** The method returns a result that says whether it succeeded and why. The caller knows what went wrong without guessing.

A rich model is not harder to write. It is just more careful. You think about what the object can do (its public methods) and what rules it must enforce (its invariants and validation).

---

## 39.5 ORMs and the Leaky Boundary

Object-Relational Mapping tools (Hibernate, SQLAlchemy, Entity Framework) promise to make the database feel like objects. You write:

```cpp
auto sub = db.subscriptions().findById(42);
sub.status = "canceled";  // Looks like setting a field
db.subscriptions().save(sub);  // Looks like saving an object
```

This is convenient. It is also a dangerous illusion. The ORM has created a **data model** that looks like a **domain model**, but it is not.

The ORM is fundamentally about persistence: how to map objects to tables, how to load related objects (lazy loading, eager loading, joins), how to detect changes and write them back, how to handle transactions.

These concerns *leak into your domain model*. You start making decisions based on the ORM:

- "I'll make this a separate object so it lazy-loads separately." (Persistence concern, not domain concern.)
- "I'll embed this object in the parent to avoid a join." (Performance concern leaks into design.)
- "I can't have this relationship because it would create a cycle in the schema." (Schema concern, not domain concern.)
- "I'll delete these objects automatically when the parent is deleted." (Cascade delete is about schema, not about whether deletion makes sense.)

Over time, your domain model becomes a **data model**. It has the shape of the database, not the shape of the problem. And because ORMs provide getters and setters (or lazy-loading magic), it is easy to fall into the anemic trap: the ORM manages persistence, so the model just holds data.

The boundary should be clear:

- **Domain model:** rich, with behavior and invariants. Lives in your `domain/` or `models/` package.
- **ORM model (entity):** dumb, with getters and setters, lazy-loading proxies, change tracking. Lives next to the database, in your `persistence/` package.
- **Boundary (repository):** translates between them. Loads ORM entities, constructs domain models, saves domain models as ORM entities.

```cpp
// domain/subscription.h — domain model
class Subscription {
private:
    uint64_t id_;
    SubscriptionStatus status_;
    // ... behavior
public:
    CancelResult cancel() { /* ... */ }
};

// persistence/subscription_entity.h — ORM model
class SubscriptionEntity {
public:
    uint64_t id;
    std::string status;  // "active", "paused", "canceled"
    // ... just data; getters and setters
};

// persistence/subscription_repository.h — boundary
class SubscriptionRepository {
public:
    std::shared_ptr<Subscription> findById(uint64_t id) {
        auto entity = db_.query("SELECT * FROM subscriptions WHERE id = ?", id);
        if (!entity) return nullptr;
        
        // Construct domain model from ORM entity
        auto sub = std::make_shared<Subscription>(
            entity.id,
            entity.customer_id,
            entity.created_at
        );
        
        // Restore state from entity
        if (entity.status == "paused") sub.pause();
        if (entity.status == "canceled") sub.cancel();
        
        return sub;
    }
    
    void save(const std::shared_ptr<Subscription>& sub) {
        // Translate domain model to ORM entity
        SubscriptionEntity entity;
        entity.id = sub.id();
        entity.status = (sub.status() == SubscriptionStatus::Active) ? "active" : "paused";
        // ... more translations
        
        db_.save(entity);
    }
};
```

This is more work than letting the ORM handle everything. But it keeps the persistence boundary clear. The domain model does not leak; it stays focused on rules and behavior, not on lazy loading and change tracking.

---

## 39.6 DTOs and Boundary Translation

A DTO (Data Transfer Object) is a dumb shape for crossing process boundaries: HTTP, RPC, message queues.

```cpp
// HTTP request JSON might be:
// { "subscription_id": 42, "action": "cancel" }

struct CancelSubscriptionRequest {
    uint64_t subscription_id;
    std::string action;
};

// HTTP response JSON might be:
// { "success": true, "message": "Subscription canceled" }

struct CancelSubscriptionResponse {
    bool success;
    std::string message;
};
```

DTOs are **not** domain models. They are the opposite: minimal shapes designed to cross boundaries efficiently. They have no behavior; they are just fields. They are serializable (to JSON, Protocol Buffers, etc.) without tricks.

The controller is responsible for translating between DTOs and domain models:

```cpp
class SubscriptionController {
private:
    SubscriptionRepository repo_;
    SubscriptionService service_;
    
public:
    CancelSubscriptionResponse cancel(const CancelSubscriptionRequest& req) {
        // Boundary entry: translate DTO to domain concepts
        auto sub = repo_.findById(req.subscription_id);
        if (!sub) {
            return { false, "subscription not found" };
        }
        
        // Core logic: operate on domain model
        auto result = sub->cancel();
        
        // Boundary exit: translate domain result to DTO
        return { result.success, result.reason };
    }
};
```

The mistake is treating DTOs as if they were domain models, passing them deep into the system:

```cpp
// WRONG: DTO leaks into domain logic
class BadSubscriptionService {
public:
    CancelSubscriptionResponse cancel(const CancelSubscriptionRequest& req) {
        // DTO should not be here; this couples domain to HTTP
    }
};
```

DTOs belong at boundaries (HTTP handlers, message queue listeners). The domain model stays inside the system.

---

## 39.7 Worked Example: Subscription with Rules

Let us design a subscription system with domain rules, then show how to implement it both ways: anemic (wrong) and rich (right).

### The Rules

1. A subscription starts in the active state.
2. A subscription cannot be canceled in the first month (free trial period).
3. A subscription can be paused and resumed.
4. A paused subscription cannot be canceled; it must be resumed first.
5. A canceled subscription cannot be resumed or modified.
6. When canceled, record the cancellation date.

### Anemic Implementation

```cpp
class AnemiC_Subscription {
public:
    uint64_t id;
    uint64_t customer_id;
    std::string status;  // "active", "paused", "canceled"
    std::chrono::system_clock::time_point created_at;
    std::optional<std::chrono::system_clock::time_point> canceled_at;
};

class SubscriptionService {
private:
    SubscriptionRepository repo_;
    
public:
    struct OperationResult { bool success; std::string reason; };
    
    OperationResult cancel(AnemiC_Subscription& sub) {
        // Rule 1: check if already canceled
        if (sub.status == "canceled") {
            return { false, "already canceled" };
        }
        
        // Rule 2: check if in trial period
        auto now = std::chrono::system_clock::now();
        if (now < sub.created_at + std::chrono::days(30)) {
            return { false, "cannot cancel during trial period" };
        }
        
        // Rule 4: check if paused
        if (sub.status == "paused") {
            return { false, "cannot cancel paused subscription" };
        }
        
        // Apply changes
        sub.status = "canceled";
        sub.canceled_at = now;
        repo_.save(sub);
        
        return { true, "" };
    }
    
    OperationResult pause(AnemiC_Subscription& sub) {
        if (sub.status == "canceled") {
            return { false, "cannot pause canceled subscription" };
        }
        if (sub.status == "paused") {
            return { false, "already paused" };
        }
        
        sub.status = "paused";
        repo_.save(sub);
        
        return { true, "" };
    }
    
    OperationResult resume(AnemiC_Subscription& sub) {
        if (sub.status != "paused") {
            return { false, "only paused subscriptions can be resumed" };
        }
        
        sub.status = "active";
        repo_.save(sub);
        
        return { true, "" };
    }
};
```

Problems:

- Rules are scattered in service methods.
- A new operation (e.g., `autoRenew()`) must duplicate the checks.
- Nothing enforces that the status field is valid.
- The repository must save the model, coupling the model to persistence.

### Rich Implementation

```cpp
enum class SubscriptionStatus { Active, Paused, Canceled };

class Subscription {
private:
    const uint64_t id_;
    const uint64_t customer_id_;
    SubscriptionStatus status_;
    const std::chrono::system_clock::time_point created_at_;
    std::optional<std::chrono::system_clock::time_point> canceled_at_;
    
    void checkInvariant() const {
        assert(status_ != SubscriptionStatus::Invalid);
        // Canceled subscriptions can have a cancellation date; others cannot
        if (status_ == SubscriptionStatus::Canceled) {
            assert(canceled_at_.has_value());
        }
    }
    
public:
    Subscription(uint64_t id, uint64_t customer_id,
                 std::chrono::system_clock::time_point created_at)
        : id_(id), customer_id_(customer_id),
          status_(SubscriptionStatus::Active),
          created_at_(created_at),
          canceled_at_(std::nullopt) {
        checkInvariant();
    }
    
    struct Result { bool success; std::string reason; };
    
    Result cancel() {
        // Rule: cannot cancel a canceled subscription
        if (status_ == SubscriptionStatus::Canceled) {
            return { false, "already canceled" };
        }
        
        // Rule: cannot cancel a paused subscription
        if (status_ == SubscriptionStatus::Paused) {
            return { false, "cannot cancel paused subscription" };
        }
        
        // Rule: cannot cancel during trial (first month)
        auto now = std::chrono::system_clock::now();
        if (now < created_at_ + std::chrono::days(30)) {
            return { false, "cannot cancel during trial period" };
        }
        
        status_ = SubscriptionStatus::Canceled;
        canceled_at_ = now;
        checkInvariant();
        return { true, "" };
    }
    
    Result pause() {
        if (status_ == SubscriptionStatus::Canceled) {
            return { false, "cannot pause canceled subscription" };
        }
        if (status_ == SubscriptionStatus::Paused) {
            return { false, "already paused" };
        }
        
        status_ = SubscriptionStatus::Paused;
        checkInvariant();
        return { true, "" };
    }
    
    Result resume() {
        if (status_ != SubscriptionStatus::Paused) {
            return { false, "only paused subscriptions can be resumed" };
        }
        
        status_ = SubscriptionStatus::Active;
        checkInvariant();
        return { true, "" };
    }
    
    // Read-only accessors
    uint64_t id() const { return id_; }
    uint64_t customerId() const { return customer_id_; }
    SubscriptionStatus status() const { return status_; }
    std::chrono::system_clock::time_point createdAt() const { return created_at_; }
    std::optional<std::chrono::system_clock::time_point> canceledAt() const { return canceled_at_; }
};
```

Usage is clean:

```cpp
Subscription sub(42, 100, now);

auto result = sub.cancel();
if (!result.success) {
    std::cerr << "Cannot cancel: " << result.reason << "\n";
    return;
}

// Subscription is now canceled; invariant is guaranteed
assert(sub.status() == SubscriptionStatus::Canceled);
```

If a new rule appears ("cannot cancel within 30 days of renewal"), you change `cancel()`. All code benefits. If a rule disappears, you remove it from one method.

---

## 39.8 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Anemic model + services** | Models are simple; easy to map to database; works with code generation | Rules scatter; hard to refactor; invariants not enforced; couples model to persistence | Small CRUD systems where there are few rules. Not recommended for domain-heavy systems. |
| **Rich model** | Rules in one place; invariants enforced; refactoring is local; public interface matches domain | More thought required in design; harder to map to database directly; cannot use naive ORM generation | Default choice for systems with business logic. Anything where rules matter more than simplicity. |
| **Hybrid: rich domain model + anemic ORM entity** | Rules stay in domain; persistence is decoupled; ORM can be simple | Translation layer (repository) adds code | Best practice. Clean boundary between domain and persistence. |
| **DTO-only** | No model; just data shapes. Serialization is trivial | No rules; no invariants; every operation is an external service call. Works for thin frontends, not for domain logic | Web frontends, mobile apps, microservices that are just pipes. Not for systems with business logic. |

---

## 39.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "A model is a database row." | A model is a concept with identity, state, behavior, and rules. A database row is how that model is stored. They are different. |
| "If the ORM generates the model, it is a domain model." | ORM-generated classes are data models shaped by the database schema. They are not domain models unless you add behavior. Most ORM-generated classes are anemic. |
| "Rich models are unmaintainable because they have too much logic." | Rich models are maintainable because the logic is localized. It is easier to change one method than to find and change rules scattered across services. |
| "You must choose: either rich models or ORMs." | False. Use ORMs for persistence (anemic entities), then translate to rich domain models at the boundary. The repository pattern does this. |
| "DTOs are the same as domain models." | DTOs cross boundaries (HTTP, RPC). Domain models live inside the system. They serve different purposes. |
| "Models should have no behavior, only data." | Models are where invariants are enforced. If a model has no behavior, the invariants are not enforced anywhere. This is the anemic-model trap. |
| "Adding a method to a model is bloat." | A method that enforces an invariant is not bloat; it is the whole point. The bloat is the duplicate logic in five services. |

---

## 39.10 Exercises

1. **Identify an anemic model in your codebase.** Find a class that is mostly getters and setters. Describe what rules about that class live in other services. List three places where the same rule is checked. Now rewrite the class as a rich model and consolidate the rules.

2. **Separate data model from domain model.** Take a domain model in your codebase and an ORM entity. Write a repository that translates between them. Show that the domain model and ORM entity can have different shapes. Verify that changing the ORM entity does not require changing the domain model.

3. **Design invariants.** You are building a `BankAccount` model. The invariant: balance never goes negative (no overdrafts). Write a `BankAccount` class with methods `deposit(amount)` and `withdraw(amount)`. Ensure the invariant is enforced in the code, not left to chance.

4. **Refactoring exercise: rules to model.** You have a `Task` service with methods like `startTask(task)`, `completeTask(task)`, `archiveTask(task)`, each of which checks a few business rules. Rewrite the `Task` model as a rich model with `start()`, `complete()`, `archive()` methods that enforce those rules. Show that the service now becomes a thin wrapper around the model (or disappears entirely).

5. **Test a rich model.** Write unit tests for the `Subscription` example. Verify that: (a) all valid state transitions are possible, (b) all invalid transitions are rejected with appropriate error messages, (c) the invariant holds after every operation. Does the test pass? (It should.)

6. **DTO translation.** You have a REST endpoint that takes a JSON request to cancel a subscription and returns a JSON response. Write a controller that accepts a DTO, translates to a domain model, calls the model's `cancel()` method, and translates the result back to a DTO. Show how this keeps DTOs and domain models separate.

7. **Conceptual: when not to use rich models.** Describe a scenario (a codebase, a domain) where rich models would be overkill. What would you use instead? Why?

---

## 39.11 Summary

A model is how your code talks about the problem domain, not how it talks about the database. A good model captures identity (what makes it unique), state (what is true now), behavior (what it can do), and invariants (what must always be true). Anemic models—data with getters and setters—scatter rules across services and fail to enforce invariants; they appear simple but create maintenance disasters. Rich models put behavior and invariants on the object, making rules localized and unbreakable. ORMs blur the line between domain models and data models, creating the illusion that database rows are objects; the solution is a clear boundary: use anemic ORM entities for persistence, translate to rich domain models in the application, and keep DTOs strictly at process boundaries. The tradeoff is not "rich models or simplicity"—it is "enforced invariants in one place or scattered rules in ten places."

---

**[← Previous: Controllers, Services, Repositories](06-controllers-services-repositories.md)** · **[↑ Part 4](README.md)** · **[Next: MVC Internals →](08-mvc-internals.md)**
