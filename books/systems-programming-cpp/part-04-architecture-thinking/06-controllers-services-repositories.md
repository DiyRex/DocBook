# Chapter 38 — Controllers, Services, Repositories

## Learning Objectives

By the end of this chapter you will be able to:

1. Define the three responsibilities — Controller (protocol), Service (workflow), Repository (persistence) — and recognize why they must be separate.
2. Explain what each layer should do and, more importantly, what each should not do.
3. Identify violations of this pattern and predict what bugs they enable.
4. Recognize when the pattern is overkill (CRUD-heavy systems with no domain logic) and honestly assess the cost.
5. Design the boundary between layers using concrete examples from real frameworks.
6. Trace a use case flow through all three layers and explain the data transformations at each boundary.

---

## 38.1 The Three Names, The Three Boundaries

If you have written a web application in the last 15 years, you have seen these folder names:

```
src/
  controllers/
  services/
  repositories/
```

Or these annotations:

```typescript
@Controller
class OrderController { }

@Service
class OrderService { }

@Repository
class OrderRepository { }
```

Or these conceptual layers, explicit or implicit, in nearly every Rails app, Django project, Spring backend, ASP.NET application, or TypeScript service.

These three names appear everywhere for the same reason: they solve the same problem, which is to separate concerns that change for different reasons.

> **A Controller handles protocol translation. A Service holds business workflow. A Repository hides data storage.**

These are not names that matter for their own sake. They matter because the responsibilities they describe are genuinely different, and mixing them creates bugs.

---

## 38.2 The Controller: Translate Protocol to Method Call

A Controller's job is narrow and mechanical. It translates an incoming request (HTTP, gRPC, CLI command, event) into a method call to a service, and translates the result back into a response format the protocol demands.

### What a Controller Does

```cpp
// Pseudocode: HTTP-based controller in C++
class OrderController {
public:
    HttpResponse createOrder(const HttpRequest& req) {
        // 1. Parse the HTTP protocol
        auto json = parseJson(req.body);
        
        // 2. Extract parameters
        string userId = json["user_id"];
        string productId = json["product_id"];
        int quantity = json["quantity"];
        
        // 3. Validate those parameters exist and are the right type
        if (userId.empty() || productId.empty() || quantity <= 0) {
            return HttpResponse(400, "Missing or invalid fields");
        }
        
        // 4. Call the service (delegate to business logic)
        auto result = orderService->createOrder(userId, productId, quantity);
        
        // 5. Handle the result
        if (!result.isSuccess()) {
            return HttpResponse(400, result.errorMessage());
        }
        
        // 6. Format the response in the protocol
        auto responseJson = Json::object();
        responseJson["order_id"] = result.orderId();
        responseJson["total"] = result.total();
        return HttpResponse(200, responseJson.dump());
    }
};
```

Notice what happened:

- Lines 1–3: Protocol-specific knowledge (HTTP, JSON).
- Lines 5–9: Parameter validation — checking that inputs exist and have the right shape.
- Line 13: Delegation to the service — this is where business logic lives.
- Lines 15–21: Response formatting — translating the result back to JSON and HTTP.

The controller knows nothing about how orders are created, what makes an order valid from a business perspective, or how data is persisted. The controller's world is: request in, method call out, method result in, response out.

### What a Controller Should NOT Do

A controller should not contain business logic. Here are the common violations:

**Violation 1: Business rule checking**

```cpp
// WRONG
HttpResponse createOrder(const HttpRequest& req) {
    auto json = parseJson(req.body);
    string userId = json["user_id"];
    
    // This is business logic, not protocol translation
    auto user = database->findUser(userId);
    if (!user.hasValidPaymentMethod()) {
        return HttpResponse(403, "No valid payment method");
    }
    if (user.accountBalance() < order.estimatedTotal()) {
        return HttpResponse(400, "Insufficient funds");
    }
    // ... more business checks
}
```

Why is this wrong? Because the same business checks must run whether the order is created via HTTP, via a batch import, or via an API call from a partner. If the logic lives in the controller, you will duplicate it. If a rule changes, you will update it in multiple places and miss one. The bugs follow immediately.

**Violation 2: Direct data access**

```cpp
// WRONG
HttpResponse listOrders(const HttpRequest& req) {
    auto userId = req.parameters()["user_id"];
    
    // Controller talking directly to the database
    auto orders = database->query(
        "SELECT * FROM orders WHERE user_id = ? AND status = ?",
        userId, "completed"
    );
    
    // Now manually serialize to JSON
    auto result = Json::array();
    for (auto& order : orders) {
        auto obj = Json::object();
        obj["id"] = order.id;
        obj["total"] = order.total;
        result.push_back(obj);
    }
    return HttpResponse(200, result.dump());
}
```

Why is this wrong? Because the service layer is supposed to be your boundary between the application and data storage. If the controller bypasses it, you have leaked the persistence mechanism into the protocol layer. Later, when you want to add caching, add logging, or change which fields are returned, you must hunt through every controller action. The service layer cannot do its job.

**Violation 3: Orchestrating workflows**

```cpp
// WRONG
HttpResponse processRefund(const HttpRequest& req) {
    auto orderId = req.parameters()["order_id"];
    
    // This is a workflow (sequence of coordinated operations)
    auto order = orderRepo->findById(orderId);
    auto payment = paymentRepo->findByOrderId(orderId);
    
    payment.setStatus("refunded");
    paymentRepo->save(payment);
    
    order.setStatus("refunded");
    orderRepo->save(order);
    
    var emailService = new EmailService();
    emailService.sendRefundEmail(order.userId());
    
    // What if the email service fails? We already saved state.
    // Who is responsible for rolling this back?
}
```

This is a workflow orchestration. It belongs in the service layer, where it can be tested, retried, and wrapped in a transaction. A controller that orchestrates workflows becomes hard to test (it has HTTP mocking, database mocking, and email service mocking all tangled together) and impossible to reuse (the workflow can only run from HTTP).

### The Controller Boundary

A healthy controller:

- Knows the protocol (HTTP status codes, JSON serialization, request/response format).
- Translates request objects into domain objects (or primitives).
- Validates shape ("is this field present?"), not rules ("is this user allowed to buy?").
- Delegates to a service.
- Translates the result back to the protocol.
- Never touches the database directly.
- Never contains business logic.

A controller is a *thin, mechanical adapter*. If your controller is more than 10–15 lines, something probably belongs in a service.

---

## 38.3 The Service: Orchestrate the Use Case

A Service's job is to implement a business use case. One service method should correspond to one meaningful action from the system's perspective: create an order, refund a payment, activate a subscription, reset a password.

The service coordinates the work that a use case requires: validating rules, loading data, mutating state, calling external systems, and ensuring consistency.

### What a Service Does

```cpp
// Pseudocode: Service layer orchestrating a use case
class OrderService {
private:
    OrderRepository& orderRepo;
    UserRepository& userRepo;
    PaymentService& paymentService;
    Transaction& transaction;
    
public:
    // One method per use case
    Result<Order> createOrder(string userId, string productId, int quantity) {
        // 1. Load the user (check that the user exists)
        auto user = userRepo->findById(userId);
        if (!user) {
            return Error("User not found");
        }
        
        // 2. Apply business rules
        if (!user.hasActiveSubscription()) {
            return Error("Subscription required");
        }
        if (quantity > user.monthlyLimit()) {
            return Error("Quantity exceeds monthly allowance");
        }
        
        // 3. Load or create the aggregate
        auto product = lookupProduct(productId);
        auto order = Order::create(user, product, quantity);
        
        // 4. Persist the state change (within a transaction)
        auto price = calculatePrice(order, user);
        order.setTotal(price);
        
        try {
            transaction.begin();
            orderRepo->save(order);
            
            // 5. Side effects (can fail, so they're in the transaction)
            paymentService->charge(user.paymentMethod(), price);
            
            transaction.commit();
        } catch (Exception& e) {
            transaction.rollback();
            return Error("Transaction failed: " + e.message());
        }
        
        return Ok(order);
    }
};
```

Notice the structure:

- **Load**: Fetch the aggregates and prerequisites the use case needs.
- **Validate**: Apply business rules (subscription required, quota checks, etc.).
- **Compute**: Calculate what should change.
- **Persist**: Save the changes in a transaction.
- **Side effects**: Call external systems (payment, email, notifications) within the same transaction boundary.

The service method is self-contained and testable. You can mock the repositories and external services and verify that the service orchestrates them correctly.

### What a Service Should NOT Do

**Violation 1: HTTP details in the service**

```cpp
// WRONG
class OrderService {
    Result<Order> createOrder(HttpRequest& req) {
        // Service should not know about HTTP
        auto userId = req.parameters()["user_id"];
        
        // Should not return HTTP responses
        if (quantity < 0) {
            return HttpResponse(400, "Invalid quantity");
        }
        // ...
    }
};
```

The service takes domain objects (userId: string, productId: string, quantity: int), not HTTP requests. It returns domain results (Order, errors), not HTTP responses. This separation means the service can be called from anywhere: CLI, scheduled job, event handler, API gateway, partner integration. The controller translates HTTP to domain and back.

**Violation 2: Database-specific queries in the service**

```cpp
// WRONG
class OrderService {
    Result<Order> createOrder(...) {
        // Service should not know SQL
        auto user = database->query(
            "SELECT * FROM users WHERE id = ? AND status = ?",
            userId, "active"
        );
        
        // Should not build SQL strings
        orders = database->query(
            "UPDATE orders SET status = ? WHERE user_id = ? AND created > ?",
            "completed", userId, oneWeekAgo()
        );
    }
};
```

Queries belong in the repository. The service asks the repository for a user with certain properties; the repository decides how to fetch it. This separation means you can swap the repository implementation (PostgreSQL to MongoDB, cached to non-cached) without touching the service.

**Violation 3: Multi-aggregate orchestration without explicit transaction**

```cpp
// WRONG
class OrderService {
    Result<Order> refund(string orderId) {
        auto order = orderRepo->findById(orderId);
        auto payment = paymentRepo->findByOrderId(orderId);
        
        // If this fails, order is marked refunded but payment wasn't processed
        order.setStatus("refunded");
        orderRepo->save(order);
        
        payment.setStatus("refunded");
        paymentRepo->save(payment);
        
        // No transaction boundary. What happens if the second save fails?
    }
};
```

When a use case touches multiple aggregates (Order and Payment, here), consistency becomes fragile. If the first save succeeds and the second fails, the system is in an inconsistent state. The service must either ensure both succeed (transaction) or design idempotence (retry-safe). This is a service responsibility, not a repository one.

### The Service Boundary

A healthy service:

- Takes domain objects and primitives as input, returns domain objects.
- Knows nothing about HTTP, gRPC, or any protocol.
- Knows nothing about SQL, nor does it build queries.
- Implements one business use case per method.
- Coordinates repositories and external services.
- Wraps multi-step operations in a transaction.
- Applies business rules (validation, authorization, constraints).
- Can be called from any protocol (HTTP, events, jobs, CLI).

A service is where the actual business logic lives. If your service is thin or absent, either you don't have much business logic (possible for simple CRUD), or it's leaking into the controller or repository (a bug).

---

## 38.4 The Repository: Translate Persistence Boundary

A Repository is a *domain-shaped interface to storage*. It gives the application a collection-like interface (`findById`, `save`, `findAll`) while hiding the storage mechanism (SQL, NoSQL, cache, multiple sources).

### What a Repository Does

```cpp
// Pseudocode: Repository interface
class OrderRepository {
public:
    // Queries named after domain intent, not SQL
    optional<Order> findById(string id);
    vector<Order> findByUserId(string userId);
    vector<Order> findCompletedAfter(DateTime cutoff);
    
    // Mutations
    void save(Order& order);
    void delete(string orderId);
};

// Implementation: Hides SQL from callers
class PostgreSQLOrderRepository : public OrderRepository {
private:
    Database& db;
    
public:
    optional<Order> findById(string id) override {
        auto result = db.query(
            "SELECT id, user_id, total, status FROM orders WHERE id = ?",
            id
        );
        
        if (result.empty()) {
            return nullopt;
        }
        
        // Translate row to domain object
        return Order(
            result[0]["id"],
            result[0]["user_id"],
            result[0]["total"],
            result[0]["status"]
        );
    }
    
    void save(Order& order) override {
        // Decide: insert or update?
        auto existing = findById(order.id());
        
        if (existing) {
            db.execute(
                "UPDATE orders SET user_id = ?, total = ?, status = ? "
                "WHERE id = ?",
                order.userId(), order.total(), order.status(), order.id()
            );
        } else {
            db.execute(
                "INSERT INTO orders (id, user_id, total, status) "
                "VALUES (?, ?, ?, ?)",
                order.id(), order.userId(), order.total(), order.status()
            );
        }
    }
};
```

Notice what happened:

- The public interface uses domain language (`findByUserId`, `findCompletedAfter`), not SQL.
- The implementation hides SQL behind that interface.
- The service calls `orderRepo->findById(id)` and gets back an Order object, not a database row.
- If you later swap PostgreSQL for a different database, only the repository implementation changes.

### The Inverse of the Violation: What a Repository Should NOT Do

**Violation 1: Business logic in the repository**

```cpp
// WRONG
class OrderRepository {
    vector<Order> findByUserId(string userId) {
        auto orders = db.query(
            "SELECT * FROM orders WHERE user_id = ? AND status = ?",
            userId, "completed"
        );
        
        // Business rule: filter by user's tier
        auto user = db.query("SELECT tier FROM users WHERE id = ?", userId);
        vector<Order> filtered;
        for (auto& order : orders) {
            if (order.total > user.tier * 1000) {
                filtered.push_back(order);
            }
        }
        return filtered;
    }
};
```

The repository should return what the database has, not apply business rules. If "only show orders above a certain total based on user tier" is a business rule, it belongs in the service. Otherwise, two different services that need the same rule will each reimplement it in their own repositories, and when the rule changes, both go out of sync.

The rule: **Repository filters by data shape; Service filters by business rule.**

**Violation 2: Multiple repositories in one class**

```cpp
// WRONG
class OrderRepository {
    vector<Order> findCompleteOrdersForUser(string userId) {
        auto orders = db.query(
            "SELECT o.* FROM orders o JOIN users u ON o.user_id = u.id "
            "WHERE u.id = ? AND o.status = ?",
            userId, "complete"
        );
        
        // Now load payment info from a different table
        for (auto& order : orders) {
            auto payment = db.query(
                "SELECT * FROM payments WHERE order_id = ?",
                order.id()
            );
            order.setPayment(payment);
        }
        
        return orders;
    }
};
```

This repository method does the work of two repositories: loading Order and Payment aggregates. It creates a hidden dependency: if Payment is updated, callers don't know they should re-query. The service should explicitly coordinate: load Order, then load Payment, then combine. That way the service controls the workflow and is testable.

**Violation 3: Generic all-purpose queries**

```cpp
// WRONG
class OrderRepository {
    vector<Row> query(string sql, vector<any> params) {
        return db.query(sql, params);
    }
};
```

A repository that just forwards SQL queries is not a repository; it's a database wrapper. It provides no abstraction. You have not separated concerns; you have added a thin, useless layer. Every caller must know SQL, and there is no isolation point when the database changes.

### The Repository Boundary

A healthy repository:

- Exposes methods named after domain intent (`findActiveUsersInRegion`, `findOrdersCreatedBy`, `save`).
- Hides the persistence mechanism (SQL, API calls, cache queries).
- Translates storage rows to domain objects and vice versa.
- Never contains business logic.
- Can be swapped with another implementation (in-memory for tests, different database for production) without changing callers.
- May coordinate with other repositories at the *interface* level (if you need Order + Payment, you ask for both explicitly, not by loading Order which loads Payment).

A repository is where you make the system *replaceable*. If you want to test the service without a real database, you mock the repositories. If you want to change databases, you write a new repository implementation. The service code never changes.

---

## 38.5 A Clean Flow: Tracing Order Creation Through Three Layers

Let's trace a single request through the three layers and see how responsibilities divide.

**HTTP Request arrives:**
```
POST /orders HTTP/1.1
Content-Type: application/json

{
  "user_id": "u123",
  "product_id": "p456",
  "quantity": 2
}
```

**Controller layer** (protocol → domain):
```cpp
auto createOrder(const HttpRequest& req) {
    // Parse HTTP
    auto body = parseJson(req.body);
    
    // Extract and validate shape
    string userId = body.value("user_id", "");
    string productId = body.value("product_id", "");
    int quantity = body.value("quantity", 0);
    
    if (userId.empty() || productId.empty() || quantity <= 0) {
        return HttpResponse(400, "Missing or invalid fields");
    }
    
    // Delegate to service
    auto result = orderService->createOrder(userId, productId, quantity);
    
    // Translate result to HTTP
    if (!result.isSuccess()) {
        return HttpResponse(400, result.error());
    }
    
    auto response = Json::object();
    response["order_id"] = result.order().id();
    response["total"] = result.order().total();
    response["status"] = "pending";
    
    return HttpResponse(201, response.dump());
}
```

**Service layer** (domain → use case):
```cpp
auto createOrder(string userId, string productId, int quantity) {
    // Load prerequisites
    auto user = userRepo->findById(userId);
    if (!user) {
        return Error("User not found");
    }
    
    // Validate business rules
    if (!user.hasActiveSubscription()) {
        return Error("Subscription required to create orders");
    }
    
    auto product = productRepo->findById(productId);
    if (!product) {
        return Error("Product not found");
    }
    
    // Compute the order
    auto order = Order::create(user.id(), product.id(), quantity);
    auto price = calculatePrice(product, quantity, user.discount());
    order.setTotal(price);
    
    // Persist atomically
    try {
        transaction.begin();
        
        // Store the order
        orderRepo->save(order);
        
        // Update inventory
        auto inventory = inventoryRepo->findByProductId(productId);
        inventory.decrementBy(quantity);
        inventoryRepo->save(inventory);
        
        transaction.commit();
        
        return Ok(order);
    } catch (Exception& e) {
        transaction.rollback();
        return Error("Failed to create order: " + e.message());
    }
}
```

**Repository layer** (domain → storage):
```cpp
void save(Order& order) {
    // Decide: insert or update?
    auto existing = db.query(
        "SELECT id FROM orders WHERE id = ?",
        order.id()
    );
    
    if (existing.empty()) {
        // Insert
        db.execute(
            "INSERT INTO orders (id, user_id, product_id, quantity, total, status, created_at) "
            "VALUES (?, ?, ?, ?, ?, ?, NOW())",
            order.id(), order.userId(), order.productId(),
            order.quantity(), order.total(), order.status()
        );
    } else {
        // Update
        db.execute(
            "UPDATE orders SET product_id = ?, quantity = ?, total = ?, status = ? "
            "WHERE id = ?",
            order.productId(), order.quantity(), order.total(),
            order.status(), order.id()
        );
    }
}
```

### What Each Layer Knows

| Layer | Knows | Doesn't Know |
|-------|-------|--------------|
| **Controller** | HTTP request/response, JSON, status codes | Business rules, database schema, how orders are calculated |
| **Service** | Business rules, use cases, workflows, transactions | HTTP, SQL, storage implementation, protocol details |
| **Repository** | Storage mechanism (SQL, API), data mapping | Business rules, orchestration, use cases |

Data flows:

- **Inbound**: HTTP → JSON → domain objects (userId, productId, quantity) → Order + Inventory changes.
- **Outbound**: Order + status → JSON → HTTP response.

Each layer speaks a different language, and the boundaries are translation points.

---

## 38.6 Real-World Naming Across Frameworks

These three layers have different names in different frameworks, but the pattern is consistent.

**Spring (Java)**

```java
// Controller
@RestController
@RequestMapping("/orders")
public class OrderController {
    @PostMapping
    public ResponseEntity<OrderResponse> create(@RequestBody CreateOrderRequest req) {
        Order order = orderService.createOrder(req.userId, req.productId, req.quantity);
        return ResponseEntity.status(201).body(new OrderResponse(order));
    }
}

// Service
@Service
public class OrderService {
    @Transactional
    public Order createOrder(String userId, String productId, int quantity) {
        // ... orchestration
    }
}

// Repository
@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    // JPA generates the SQL
}
```

**Laravel (PHP)**

```php
// Controller
class OrderController {
    public function create(Request $request) {
        $validated = $request->validate([
            'user_id' => 'required|uuid',
            'product_id' => 'required|uuid',
            'quantity' => 'required|int|min:1'
        ]);
        
        $order = $this->orderService->createOrder(
            $validated['user_id'],
            $validated['product_id'],
            $validated['quantity']
        );
        
        return response()->json(['order_id' => $order->id], 201);
    }
}

// Service (often manual, not a framework annotation)
class OrderService {
    public function createOrder($userId, $productId, $quantity) {
        // ... orchestration with transaction
    }
}

// Repository (often an Eloquent Model with custom methods)
class OrderRepository {
    public function findById($id) {
        return Order::find($id);
    }
    
    public function save(Order $order) {
        $order->save();
    }
}
```

**ASP.NET Core (C#)**

```csharp
// Controller
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase {
    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderRequest req) {
        var order = await _orderService.CreateOrderAsync(
            req.UserId, req.ProductId, req.Quantity
        );
        return CreatedAtAction(nameof(CreateOrder), new { id = order.Id }, order);
    }
}

// Service
[Service]
public class OrderService {
    public async Task<Order> CreateOrderAsync(string userId, string productId, int quantity) {
        // ... async orchestration
    }
}

// Repository
public interface IOrderRepository {
    Task<Order> FindByIdAsync(string id);
    Task SaveAsync(Order order);
}
```

**Node.js/Express (TypeScript)**

```typescript
// Controller
router.post("/orders", async (req, res) => {
    const { user_id, product_id, quantity } = req.body;
    
    if (!user_id || !product_id || quantity <= 0) {
        return res.status(400).json({ error: "Invalid input" });
    }
    
    try {
        const order = await orderService.createOrder(
            user_id, product_id, quantity
        );
        res.status(201).json({ order_id: order.id, total: order.total });
    } catch (err) {
        res.status(400).json({ error: err.message });
    }
});

// Service
class OrderService {
    async createOrder(userId, productId, quantity) {
        // ... orchestration with transaction
    }
}

// Repository
class OrderRepository {
    async findById(id) {
        return db.query("SELECT * FROM orders WHERE id = ?", [id]);
    }
    
    async save(order) {
        // ... insert or update
    }
}
```

The names differ, but the pattern is identical: **separation of protocol, workflow, and storage**.

---

## 38.7 When the Pattern Fails: CRUD Systems with No Domain Logic

Here is an honest assessment: the three-layer pattern is overkill for some systems.

If your application is purely CRUD (Create, Read, Update, Delete) with no business rules, no workflows, and no invariants to maintain, then a service layer becomes a passthrough:

```cpp
// CRUD service that adds nothing
class UserService {
    User getUser(string id) {
        return userRepo->findById(id);
    }
    
    void updateUser(User& user) {
        userRepo->save(user);
    }
};
```

The service does no validation, no coordination, no transaction wrapping. It is just forwarding calls. In this case, you might legitimately skip the service layer and have the controller call the repository directly. Or you might use Rails-style Models that blend repository and validation logic.

However, most systems have *some* domain logic, and it tends to grow. A system that starts as pure CRUD often accumulates:

- Validation rules ("discount must be between 0 and 100").
- Workflows ("cannot delete a user if they have active orders").
- Side effects ("when an order is created, email the merchant").
- Transactions ("update inventory and order atomically").

Once these appear, the lack of a service layer becomes painful. Every controller action becomes a minigame of "which repositories do I call in what order?" Bugs appear because one controller orchestrates a workflow one way and another controller orchestrates it differently. You end up refactoring into a service layer anyway, and by then the damage is done.

**Better approach:** Start with the three-layer pattern. It is not overkill if it prevents bugs. A service method that just delegates to a repository costs almost nothing. The cost accrues if you skip the service and later regret it.

For small applications (under 10,000 lines, no workflows), you can honestly skip the service. For anything larger or complex, the pattern earns its cost.

---

## 38.8 Common Misconceptions

**Myth 1: "You must always have a service layer."**

No. If your application is pure CRUD with no business rules, a service layer is overhead. But most applications accumulate business logic, and skipping the service layer makes it hard to locate that logic later.

**Myth 2: "A repository is just a database wrapper."**

A repository is a *domain-shaped database wrapper*. The distinction is crucial. A wrapper that just forwards SQL queries provides no abstraction. A repository that exposes domain operations (`findActiveUsers`, `findOrdersForMonth`) is replaceable. If you can only call SQL queries through your "repository," it is not a repository.

**Myth 3: "Controllers can have some business logic for simple cases."**

No. If the business logic is simple, it is still business logic, and it still needs to be testable and reusable. The simplicity is not an excuse for the violation. Controllers become complicated fast when logic leaks into them.

**Myth 4: "Services should be thin; most logic belongs in models/entities."**

There is a pattern called "Domain-Driven Design" (Fowler) that emphasizes pushing logic into domain models themselves. That is legitimate, but it is different from what is described here. In DDL, the model (Order) has methods like `refund()`, `cancel()`, `setDiscountedPrice()`. The service coordinates *calls* to those methods. Here, I am focusing on the simpler three-layer structure, where the service holds logic. Both are correct; DDD is more sophisticated and requires more design upfront.

**Myth 5: "The repository interface should match the database schema."**

No. The repository interface should match your access patterns. If you often need "orders completed in the last 30 days by user X," that should be a method name (`findCompletedByUserIn30Days`), not a generic query builder where every caller must construct the same WHERE clause.

---

## 38.9 Tradeoffs: When to Use This Pattern

| Aspect | Three-Layer Pattern | Direct Controller-to-DB | Monolithic Model |
|--------|---------------------|------------------------|------------------|
| **Testability** | High — mock repos, test service logic | Moderate — need HTTP mocking | Depends on model design |
| **Reusability** | High — service can be called from any protocol | Low — logic is tangled with HTTP | High — models are self-contained |
| **Changeability** | High — swap repo implementation, change service, swap protocol | Low — changing DB or protocol touches many files | Moderate — models are stable but can grow large |
| **Complexity** | Moderate — three layers to understand | Low — less structure | Moderate — models become complex |
| **Cognitive Load** | Moderate — clear separation of concerns | High — everything mixed | High — large models |
| **Code Duplication** | Low — logic is centralized | High — same checks in multiple controllers | Low — encapsulated in models |

Use the three-layer pattern when:

- You have multiple protocols (HTTP, gRPC, CLI, events) that need to trigger the same logic.
- You have business rules that must run consistently.
- You need to test business logic without a database.
- You plan to change how data is stored.

Use a simpler approach when:

- Your application is genuinely CRUD with no logic.
- Your team is tiny and can maintain discipline.
- You are prototyping or in discovery mode (but be ready to refactor).

---

## 38.10 Exercises

**Exercise 1: Identify the Violations**

Read the following code and identify which layer is violating its responsibility. Explain what bug the violation enables.

```cpp
class PaymentController {
    HttpResponse processRefund(const HttpRequest& req) {
        auto orderId = req.parameter("order_id");
        
        auto user = userDatabase->query(
            "SELECT * FROM users WHERE id = ?",
            req.session.userId()
        );
        
        if (user.tier != "premium") {
            return HttpResponse(403, "Refunds only for premium users");
        }
        
        auto order = orderDatabase->query(
            "SELECT * FROM orders WHERE id = ?",
            orderId
        );
        
        auto payment = paymentDatabase->query(
            "SELECT * FROM payments WHERE order_id = ?",
            orderId
        );
        
        // If this fails, payment is marked refunded but order isn't
        paymentDatabase->execute("UPDATE payments SET status = ? WHERE id = ?", "refunded", payment.id);
        orderDatabase->execute("UPDATE orders SET status = ? WHERE id = ?", "refunded", order.id);
        
        return HttpResponse(200, "Refunded");
    }
};
```

What bugs are present? How would you restructure it?

**Exercise 2: Design a Repository Interface**

You are building an e-commerce system. Design the repository interfaces for:

- `UserRepository`: Methods for finding users, saving users.
- `OrderRepository`: Methods for finding orders by user, by status, by date range, and saving orders.

Write the method signatures. Do not write implementations. Focus on what *domain operations* should be available.

**Exercise 3: Trace a Workflow**

You have a "cancel subscription" use case. It must:

1. Load the subscription.
2. Check that the user can cancel (not already cancelled, not in a contract period).
3. Mark it as cancelled.
4. Refund any overages.
5. Email the user.

Write pseudocode for the service method that orchestrates this. Assume you have a `subscription`, `payment`, and `email` repository/service.

**Exercise 4: Identify Over-Abstraction**

Is the following code over-engineered? Justify your answer.

```cpp
class HelloController {
    HttpResponse hello() {
        auto message = helloService->generateHello();
        return HttpResponse(200, message);
    }
};

class HelloService {
    string generateHello() {
        return helloRepository->findGreeting();
    }
};

class HelloRepository {
    string findGreeting() {
        return "Hello, World!";
    }
};
```

---

## 38.11 Summary

Controllers translate protocol to method calls. Services orchestrate use cases. Repositories hide storage.

Each layer has a clear responsibility and should not violate it. Controllers should not contain business logic, services should not know about HTTP, and repositories should not apply business rules. Violating these boundaries enables bugs: logic leaks into the wrong place, becomes hard to test, becomes hard to change.

The pattern is not a law, but a heuristic. CRUD-heavy systems with no domain logic can skip the service layer. But most systems have business logic, and that logic has nowhere to live unless you provide the layers.

Think of the three layers as a way to ask: "Where should this decision live?" Protocol decisions in controllers. Business decisions in services. Storage decisions in repositories. When you can answer that question clearly, your architecture is sound.

---

> **[← Previous: Layered Architecture](05-layered-architecture.md)**  ·  **[↑ Part 4](README.md)**  ·  **[Next: Why Models Exist →](07-why-models-exist.md)**
