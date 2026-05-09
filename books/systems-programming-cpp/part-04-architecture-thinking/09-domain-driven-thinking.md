# Chapter 41 — Domain-Driven Thinking

Domain-Driven Design (DDD) has a reputation for being heavyweight. Books run 500+ pages; frameworks offer "DDD tooling"; consultants charge five figures to teach it. Most of that complexity is misread. The core idea is one sentence: **structure the code around the language that the domain experts use.** Everything else—aggregates, repositories, bounded contexts—is technique to make that idea workable at scale. This chapter separates the substance from the ceremony.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Define a ubiquitous language and explain why code and conversation must use the same words.
2. Identify bounded contexts and understand why a single model cannot be consistent across the entire system.
3. Recognize entities (identity matters), value objects (identity doesn't), and aggregates (clusters with a root).
4. Use repositories to load and save aggregates, and domain events to let aggregates announce what happened.
5. Distinguish strategic DDD (context boundaries) from tactical DDD (rich models) and know which gives more leverage.
6. Decide whether your codebase needs DDD thinking or whether simpler patterns suffice.
7. Apply DDD to a small domain and recognize the cost-benefit tradeoff.

---

## 41.1 Why the Model Matters: Ubiquitous Language

Here is a dangerous scenario: a team has a "Customer" class in the code. The customer service team calls them "accounts." The billing team calls them "subscription holders." The support team calls them "contacts."

No one realizes they are talking about the same thing. Bugs appear: the customer service team's system tries to do something with an "account" that the billing system's "subscription holder" forbids. Two integrations, the same domain object, two names, three integration points where confusion lives.

This is not a naming problem. It is a model problem.

Domain-Driven Design starts with a deceptively simple rule: **the team and the code must speak the same language.** That language is the Ubiquitous Language. It is called ubiquitous because it appears everywhere: in conversations, in code, in documentation, in tests, in architecture diagrams.

When your code says `processRefund()` but the business says "we issue a reversal," those are two languages. A developer hears "reversal" and looks at the code: is it a reversal? It looks like a refund. Are they the same? A bug is waiting. Especially if the developer never asks.

The Ubiquitous Language is an ongoing conversation, not a glossary you write once. It evolves as the team and domain experts collaborate. When the language shifts—when the business realizes that what they called a "refund" is sometimes a credit, sometimes a chargeback—the code reflects that shift immediately, not after a quarterly refactor.

Building the Ubiquitous Language is the first responsibility of any software team working on a domain. Not writing interfaces, not choosing a framework. Language.

If your codebase has no coherent domain language—if identifiers drift, if the same concept has five names, if conversations and code diverge—DDD is not the answer. A glossary is not the answer either. The answer is a decision: will we speak one language, or five? If one, that language lives in the code. And the code must reflect the domain accurately.

---

## 41.2 Bounded Contexts: Why One Model Cannot Be Everywhere

Here is another dangerous belief: "we will have one data model, shared across the whole company, and all services will use it."

This fails.

A model is only consistent within its boundary. The "Customer" in the billing context is not the same shape as the "Customer" in the support context.

**In the billing context:**
- Customer has: id, billing address, payment methods, subscription status, balance owed.
- Customer's operations: change payment method, add address, update card.
- Customer's rules: cannot have negative balance; cannot change address during charge period.

**In the support context:**
- Customer has: id, contact name, contact email, contact phone, ticket history, satisfaction score.
- Customer's operations: update contact info, assign to support team, escalate ticket.
- Customer's rules: email is required; cannot change email if open tickets exist.

These are not the same thing. Forcing them into one model leads to a bloated object that satisfies neither context. A shared "Customer" model would require fields that billing does not care about (satisfaction_score) and rules that support does not enforce (cannot change address during charge). Over time, changes in one context break assumptions in another.

The solution is bounded contexts. Each context has its own model, uses its own language, maintains its own invariants. The billing context has a `Customer` aggregate (id, billing address, payment methods). The support context has a `Contact` aggregate (id, name, email, phone, ticket history). They are separate models.

At the boundary—where billing needs to know "is this customer active" for support tickets, or support needs to process a refund—there is a translator. It is explicit. Both sides understand the translation is happening. A change in one model does not surprise the other.

```cpp
// billing/customer.h — Billing context
class Customer {
private:
    uint64_t id_;
    Address billing_address_;
    std::vector<PaymentMethod> payment_methods_;
    Money balance_owed_;
    
public:
    // Billing-specific operations
    bool canChargeNow() const { return balance_owed_ > Money::zero(); }
};

// support/contact.h — Support context
class Contact {
private:
    uint64_t id_;
    std::string email_;
    std::string phone_;
    std::vector<Ticket> tickets_;
    SatisfactionScore score_;
    
public:
    // Support-specific operations
    bool hasOpenTickets() const { return !tickets_.empty(); }
};

// integration/billing_support_integration.h — Translator at boundary
struct BillingToSupportAdapter {
    static Contact toBillingModel(const billing::Customer& biller) {
        // Load the support Contact for the same person
        // They share an id, but otherwise are separate
        Contact contact;
        contact.setCustomerId(biller.id());
        // Do not copy billing_address_ to contact; they are different concerns
        return contact;
    }
};
```

This is more code than sharing one "Customer" everywhere. That is the point. The extra code is the safety cost. The shared model is the liability cost.

Common DDD mistake: trying to model the whole company at once. Mature DDD systems have 5–20 bounded contexts, not one. Each context has autonomy. Alignment happens at boundaries, through explicit translation, not through force-fitting everything into one model.

---

## 41.3 Entities, Value Objects, and Aggregates

Inside a bounded context, models take three forms: entities, value objects, and aggregates that compose them.

### Entities

An entity is an object whose identity matters. If you have two `Order` objects with the same id, they are the same order—even if their contents differ (one has shipped, one hasn't). The id is immutable; it defines what the object is.

```cpp
class Order {
private:
    const uint64_t id_;  // Identity—immutable
    std::vector<LineItem> items_;
    OrderStatus status_;
    
public:
    uint64_t id() const { return id_; }
    // If two Orders have the same id_, they are the same order
};
```

### Value Objects

A value object is the opposite. Identity doesn't matter. Two `Money` objects with the same amount and currency are interchangeable; you do not care which one you get.

```cpp
class Money {
private:
    double amount_;
    std::string currency_;  // "USD", "EUR"
    
public:
    Money(double amt, std::string_view curr)
        : amount_(amt), currency_(curr) {}
    
    bool operator==(const Money& other) const {
        return amount_ == other.amount_ && currency_ == other.currency_;
    }
    
    Money operator+(const Money& other) const {
        // No id; just combine the values
        assert(currency_ == other.currency_);
        return Money(amount_ + other.amount_, currency_);
    }
};
```

Value objects have no side effects. They are immutable (or act as though they are). You pass them by value, copy them freely, compare them by content. No one needs to say "give me the Money object with id 42"; they just say "give me 100 dollars."

Entities have identity and state. Value objects have no identity. Both matter.

### Aggregates

An aggregate is a cluster of entities and value objects that is treated as a single unit. The aggregate has one **root entity**—the entry point through which all changes flow. Changes to other entities in the aggregate go through the root.

Why? Because aggregates enforce invariants. An invariant that spans multiple objects cannot be enforced unless someone coordinates.

Example: an `Order` contains multiple `LineItem`s. The rule is "total order value cannot exceed $1 million." This rule involves both the order and its line items. A single `LineItem` cannot enforce this—it does not know the total. Only the `Order` (the root) can enforce it.

```cpp
class Order {
private:
    const uint64_t id_;  // Root entity
    std::vector<LineItem> line_items_;  // Entities
    Money total_;  // Value object
    OrderStatus status_;
    
    void checkInvariant() const {
        // Invariant: total cannot exceed limit
        const Money limit(1000000, "USD");
        assert(total_ <= limit);
    }
    
public:
    // Changes to line items go through the root
    bool addLineItem(const LineItem& item) {
        // Only the root can add; no one can add directly to line_items_
        
        // Check invariant before modifying
        Money new_total = total_ + item.price();
        if (new_total > Money(1000000, "USD")) {
            return false;  // Reject the change; invariant would break
        }
        
        line_items_.push_back(item);
        total_ = new_total;
        checkInvariant();
        return true;
    }
    
    const std::vector<LineItem>& lineItems() const {
        return line_items_;
    }
};
```

The caller does not say "add this line item to the order's line_items_ array." They say `order.addLineItem(item)`. The order enforces the rule. A new person reading the code has only one place to look to understand the rule: the `Order` class.

Aggregates are sized to enforce invariants, not to optimize queries. A mistake: "let me make this a giant aggregate to avoid joins." That is a query concern, not an invariant concern. Separate them.

---

## 41.4 Repositories and Domain Events

### Repositories

A repository saves and loads aggregates. It is the bridge between the domain (which thinks about aggregates) and persistence (which thinks about data).

The repository hides the storage mechanism. It does not expose SQL, does not expose the schema, does not expose ORM entities. It just says: "I will save your aggregate, and later I will load it back, unchanged."

```cpp
// Domain-facing interface
class OrderRepository {
public:
    virtual ~OrderRepository() = default;
    
    // Save an aggregate
    virtual void save(const Order& order) = 0;
    
    // Load an aggregate
    virtual std::shared_ptr<Order> findById(uint64_t id) = 0;
    
    // Find multiple aggregates
    virtual std::vector<std::shared_ptr<Order>> findByCustomer(uint64_t customer_id) = 0;
};

// Implementation (hides persistence details)
class PostgresOrderRepository : public OrderRepository {
private:
    DatabaseConnection& db_;
    
public:
    void save(const Order& order) override {
        // Translate Order aggregate to ORM entities (OrderEntity, LineItemEntity)
        // INSERT or UPDATE in database
        // The caller does not know or care about this translation
    }
    
    std::shared_ptr<Order> findById(uint64_t id) override {
        // Query database, reconstruct Order aggregate from rows
        // Return the aggregate, not the raw data
    }
};
```

The key insight: the repository returns aggregates, not rows. If the caller loads an Order, they get a rich Order object with methods and invariants, not a DTO.

### Domain Events

When something important happens to an aggregate—a Customer places an Order, an Order is shipped, a Payment is processed—the aggregate emits a domain event. Domain events announce what happened, without the aggregate needing to know who cares.

```cpp
// Domain event
struct OrderPlaced {
    uint64_t order_id;
    uint64_t customer_id;
    Money total;
    std::chrono::system_clock::time_point timestamp;
};

// Aggregate emits the event
class Order {
private:
    std::vector<std::function<void(const OrderPlaced&)>> order_placed_listeners_;
    
public:
    void place() {
        // Validate and transition to "placed" state
        status_ = OrderStatus::Placed;
        
        // Emit the event
        OrderPlaced evt{
            id_,
            customer_id_,
            total_,
            std::chrono::system_clock::now()
        };
        
        // Notify listeners (they are registered by the framework, not by Order)
        for (auto& listener : order_placed_listeners_) {
            listener(evt);
        }
    }
};
```

Domain events decouple behavior. The Order does not know that:
- A billing service needs to invoice the customer
- A warehouse service needs to pick and ship items
- An analytics service needs to record the order for reporting

But when the order is placed, all three of these happen. They happen because they listen to the `OrderPlaced` event. If a new service needs to do something when an order is placed, you add a listener. The Order aggregate doesn't change.

This is vastly better than the Order calling out to a BillingService, WarehouseService, and AnalyticsService directly. That creates tight coupling. With events, the Order is isolated.

---

## 41.5 Strategic vs. Tactical DDD

DDD has two levels.

**Tactical DDD** is about building the model correctly: rich models, value objects, aggregates, repositories. It is how you write the code inside one bounded context. Chapter 39 (Why Models Exist) covers tactical DDD in depth.

**Strategic DDD** is about defining boundaries correctly: which concepts belong in which context, how contexts relate, how to integrate across boundaries. It is about the big picture.

Here is which one matters more: strategic DDD.

Tactical DDD—rich models, aggregates—is good engineering, but it is local. If you have a 10-person team in one domain, tactical DDD is elegant. If you have 10 teams, each with their own domain, and they don't communicate, tactical DDD in each team does not help. The system is still chaos.

Strategic DDD—bounded contexts, context maps, integration patterns—addresses the chaos. It says: which teams own which concepts? What happens when Team A (billing) needs data from Team B (customer)? Is there a shared database? An API? An event stream?

The highest-leverage DDD work happens at the strategic level. Spend time getting the boundaries right, and the implementation (tactical) often follows naturally. Get the boundaries wrong, and even perfect tactical DDD cannot save you.

**Context Map** is a strategic DDD tool. It is a diagram showing all bounded contexts and their relationships. Example:

```
┌────────────────┐
│    Billing     │
│   (payment)    │
└────────────────┘
         │
         │ API call (when customer balance is due)
         ▼
┌────────────────┐        ┌──────────────┐
│   Customer     │◄──────►│  Support     │
│                │  events│  (contacts)  │
│                │        │              │
└────────────────┘        └──────────────┘
         │
         │ event stream (Customer.placed_order)
         ▼
┌────────────────┐
│   Warehouse    │
│   (inventory)  │
└────────────────┘
```

Each box is a bounded context. Each arrow is an integration: Billing calls the Customer API to fetch balance; Customer publishes events that Warehouse and Support consume. With this map, everyone understands the system architecture without reading code.

Most DDD mistakes stem from not doing this strategic work. Teams end up with context boundaries that cut through the problem space badly, or with hidden dependencies that create integration nightmares. Strategic DDD prevents that.

---

## 41.6 When DDD Pays Off

DDD is heavy. It requires investment: time to identify contexts, time to build repositories, time to define invariants. When does that investment pay off?

DDD shines when:

1. **The domain has rich rules.** If your system is CRUD (create, read, update, delete), DDD is overkill. If your system has business logic—subscription rules, billing cycles, approval workflows, risk calculations—DDD captures that logic and keeps it maintainable.

2. **The system lives for years.** If you are building a three-month prototype, don't do DDD. If you are building a system that your company will rely on for a decade, DDD's investment in clear boundaries and enforced invariants pays off huge.

3. **Multiple teams will work on it.** One person can keep everything in their head. Ten people cannot. DDD gives them a language and boundaries to work within independently.

4. **The domain evolves.** If the rules are fixed forever, document them and move on. If the rules change—the subscription rules become more complex, a new product line is added, a new market segment opens—DDD's model-first approach lets you evolve the code without starting over.

DDD does not pay off when:

1. **No real domain.** A CRUD app that just stores and fetches data is not a domain. It is a database with a UI. Don't use DDD for this; use a simple ORM and services.

2. **One person, one time.** If you are the only engineer and the system will be replaced in two years, spend your time on features, not architecture.

3. **The rules are vague.** If domain experts cannot articulate the rules clearly, DDD won't help you discover them. You will spend time modeling fog.

4. **External constraints dominate.** If the problem is "we need to call this API and show the result," the domain is the API, not your code. DDD is not relevant.

The honest assessment: DDD is for systems where the domain is complex and valuable enough to justify the investment. Start without it. When you feel pain—when rules scatter, when contexts collide, when people talk at cross-purposes—DDD is the answer. Not before.

---

## 41.7 Worked Example: A Small Order Domain

Let us build a small `Order` domain with three rules:

1. An order can have at most 100 line items.
2. An order must have a shipping address.
3. An order cannot be canceled once it has shipped.

### The Ubiquitous Language

In conversation with the business, we establish:
- An **order** starts in "draft" state. You can add items, remove items.
- When you **place** an order, it transitions to "placed" state. No more items can be added.
- The warehouse **ships** the order. It transitions to "shipped" state.
- A **placed** order can be canceled if the warehouse hasn't picked it yet. A **shipped** order cannot be canceled.

This language lives in the code.

### The Model

```cpp
// domain/order_status.h
enum class OrderStatus { Draft, Placed, Shipped, Canceled };

// domain/line_item.h
class LineItem {
private:
    const uint64_t product_id_;
    uint32_t quantity_;
    Money price_;
    
public:
    LineItem(uint64_t product_id, uint32_t qty, const Money& price)
        : product_id_(product_id), quantity_(qty), price_(price) {}
    
    uint64_t productId() const { return product_id_; }
    uint32_t quantity() const { return quantity_; }
    Money price() const { return price_; }
    
    Money total() const {
        return price_ * static_cast<double>(quantity_);
    }
};

// domain/order.h
class Order {
private:
    const uint64_t id_;
    const uint64_t customer_id_;
    OrderStatus status_;
    std::vector<LineItem> line_items_;
    std::optional<Address> shipping_address_;
    std::chrono::system_clock::time_point placed_at_;
    std::chrono::system_clock::time_point shipped_at_;
    
    static constexpr size_t MAX_LINE_ITEMS = 100;
    
    void checkInvariant() const {
        // After "placed", shipping address must be set
        if (status_ == OrderStatus::Placed || status_ == OrderStatus::Shipped) {
            assert(shipping_address_.has_value());
        }
        
        // Line items count never exceeds limit
        assert(line_items_.size() <= MAX_LINE_ITEMS);
    }
    
public:
    Order(uint64_t id, uint64_t customer_id)
        : id_(id), customer_id_(customer_id),
          status_(OrderStatus::Draft),
          placed_at_(std::nullopt),
          shipped_at_(std::nullopt) {}
    
    uint64_t id() const { return id_; }
    uint64_t customerId() const { return customer_id_; }
    OrderStatus status() const { return status_; }
    const std::vector<LineItem>& lineItems() const { return line_items_; }
    
    // Rule 1: add item (only in draft; at most 100 items)
    struct AddResult { bool success; std::string reason; };
    
    AddResult addItem(const LineItem& item) {
        if (status_ != OrderStatus::Draft) {
            return { false, "can only add items in draft state" };
        }
        
        if (line_items_.size() >= MAX_LINE_ITEMS) {
            return { false, "maximum 100 items per order" };
        }
        
        line_items_.push_back(item);
        checkInvariant();
        return { true, "" };
    }
    
    // Rule 2: place order (requires shipping address)
    struct PlaceResult { bool success; std::string reason; };
    
    PlaceResult place() {
        if (status_ != OrderStatus::Draft) {
            return { false, "only draft orders can be placed" };
        }
        
        if (!shipping_address_.has_value()) {
            return { false, "shipping address required" };
        }
        
        if (line_items_.empty()) {
            return { false, "order must have at least one item" };
        }
        
        status_ = OrderStatus::Placed;
        placed_at_ = std::chrono::system_clock::now();
        checkInvariant();
        return { true, "" };
    }
    
    // Rule 3: cancel order (only if placed; not if shipped)
    struct CancelResult { bool success; std::string reason; };
    
    CancelResult cancel() {
        if (status_ == OrderStatus::Draft) {
            return { false, "draft orders are already canceled implicitly" };
        }
        
        if (status_ == OrderStatus::Shipped) {
            return { false, "cannot cancel a shipped order" };
        }
        
        if (status_ == OrderStatus::Canceled) {
            return { false, "already canceled" };
        }
        
        status_ = OrderStatus::Canceled;
        checkInvariant();
        return { true, "" };
    }
    
    // Warehouse ships the order
    struct ShipResult { bool success; std::string reason; };
    
    ShipResult ship() {
        if (status_ != OrderStatus::Placed) {
            return { false, "only placed orders can be shipped" };
        }
        
        status_ = OrderStatus::Shipped;
        shipped_at_ = std::chrono::system_clock::now();
        checkInvariant();
        return { true, "" };
    }
    
    // Set shipping address (only in draft)
    bool setShippingAddress(const Address& addr) {
        if (status_ != OrderStatus::Draft) {
            return false;
        }
        shipping_address_ = addr;
        checkInvariant();
        return true;
    }
};
```

### Using the Model

```cpp
// Create an order
Order order(42, 100);  // id=42, customer_id=100

// Add items
auto r1 = order.addItem(LineItem(1, 2, Money(50, "USD")));
auto r2 = order.addItem(LineItem(2, 1, Money(100, "USD")));
assert(r1.success && r2.success);

// Set address
Address addr{"123 Main St", "Springfield", "US"};
assert(order.setShippingAddress(addr));

// Try to place
auto place_result = order.place();
assert(place_result.success);

// Cannot add items anymore
auto r3 = order.addItem(LineItem(3, 1, Money(25, "USD")));
assert(!r3.success);  // "can only add items in draft state"

// Warehouse ships
auto ship_result = order.ship();
assert(ship_result.success);

// Try to cancel (should fail)
auto cancel_result = order.cancel();
assert(!cancel_result.success);  // "cannot cancel a shipped order"
```

The rules are not scattered in a service; they are enforced by the Order aggregate itself. If a new operation is added later (e.g., "pause" an order), the rules are checked in one place.

---

## 41.8 Tradeoffs

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Full DDD (strategic + tactical)** | Clear bounded contexts; rich models with enforced invariants; rules localized; scales to many teams; evolvable | Heavy upfront modeling; more code; requires domain expertise; overkill for small systems | Large systems with complex domains and multiple teams. Domains that change over time. |
| **Tactical DDD only (rich models, no context map)** | Rules enforced locally; clear model semantics; easy to test | Context boundaries unclear; teams may duplicate work; integration points fuzzy; doesn't solve the big-picture problem | Medium systems with complex rules but single context or loose coupling. |
| **Simple data services (anemic model + services)** | Less code; easy to map to database; familiar pattern | Rules scatter; invariants not enforced; hard to refactor; couples model to persistence | Small CRUD systems. Thin frontends. No complex domain logic. |
| **Minimal modeling (DTOs + functions)** | Lowest overhead; straightforward; stateless | No invariants; no domain language; highly procedural; hard to onboard new engineers | Tiny systems. Microservices that are just pipes. Throwaway code. |

---

## 41.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "DDD requires a heavy framework or library." | DDD is a way of thinking about domain boundaries and models. You can apply it with plain C++ classes. Frameworks help at scale, but DDD is not about frameworks. |
| "DDD means building a giant shared model." | DDD means the opposite: drawing clear boundaries so each context has its own model. A shared model is an anti-pattern. |
| "You need domain experts to do DDD." | You need to *talk to* domain experts and understand what they care about. The resulting language lives in code; engineers design the model. |
| "DDD is only for startups / enterprises." | DDD scales from small teams to large ones. It is about clarity, not size. A tiny system with fuzzy rules benefits from DDD thinking; a large CRUD system might not. |
| "DDD means no databases." | DDD means the domain model is separate from the persistence model. You still use databases. The repository pattern bridges them. |
| "Aggregates must be small." | Aggregates are sized to enforce invariants, not for performance or query efficiency. Some aggregates are large. Use bounded contexts and queries for optimization, not by shrinking aggregates. |
| "DDD is done; we don't need architectural work." | DDD is continuous. As the domain evolves, boundaries may shift. As teams grow, contexts may split. As integrations multiply, maps must be updated. |

---

## 41.10 Exercises

1. **Identify a ubiquitous language gap.** Pick a codebase you know. Find a concept that has two different names in conversation and code. Describe how this gap has caused confusion. Now write the correct name in the code; how much would change?

2. **Draw a context map.** Take a system you work on (or a realistic example). Identify all the bounded contexts. Draw them as boxes. For each relationship, label the integration: API call, event stream, shared database, etc. Does the map make sense?

3. **Separate entities and value objects.** Take a domain (orders, users, payments, anything). List the concepts. For each, decide: is identity important? If yes, it is an entity; if no, a value object. Sketch the classes. What happens if you try to treat an entity like a value object?

4. **Design an aggregate and its invariants.** You are modeling a `Library`. A library has books. The invariant: no book can be checked out by two patrons simultaneously. Sketch the `Library` aggregate. How is the invariant enforced? What are the public methods? How would you test that the invariant holds?

5. **Implement a repository.** Take the Order example. Write a `OrderRepository` interface that hides the persistence layer. Then implement it with a simple in-memory map (no database). Show that the caller does not care how orders are stored.

6. **Domain events.** The Order aggregate now needs to emit an event when it is placed: `OrderPlaced`. Show: (a) the event structure, (b) how the Order emits it, (c) how a listener (e.g., a BillingService) would respond. What does the Order not need to know about the listener?

7. **When not to use DDD.** Describe a system where DDD would be wrong. Why would it be overkill? What simpler approach would work better?

---

## 41.11 Summary

Domain-Driven Design is not a framework or a ceremony; it is a commitment to structure code around the problem domain. The core practice is ubiquitous language: the team and code speak the same words, in the same way. Bounded contexts prevent the mistake of forcing one model everywhere; each context has its own language, models, and invariants. Inside a context, rich models (entities, value objects, aggregates) enforce invariants and localize behavior. Repositories hide persistence; domain events decouple actions. Strategic DDD (context boundaries and maps) matters more than tactical DDD (rich models); get the boundaries right, and the implementation follows. DDD pays off in systems with complex rules, multiple teams, and long lifespans. For simple CRUD systems with few rules, it is overhead. The tradeoff is not "DDD or simplicity"—it is "enforced domain rules in isolated contexts or scattered rules across services and teams."

---

**[← Previous: MVC Internals](08-mvc-internals.md)** · **[↑ Part 4](README.md)** · **[Next: Why Frameworks Use DI →](10-why-frameworks-use-di.md)**
