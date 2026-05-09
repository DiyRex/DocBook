# Chapter 43 — Boundaries: When to Split Logic

Every architecture decision is, at heart, a question about boundaries. Where do you draw the line? What's on each side? The wrong line creates accidental complexity that no amount of clean code can erase. A boundary drawn too late forces you to refactor code that has become too tangled. A boundary drawn too early costs in operational complexity and debugging difficulty. This chapter teaches you to recognize the signals that a boundary should exist, and to understand the true cost of each kind of boundary you might draw.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Recognize the seven types of boundaries and the cost spectrum from function-level to organizational boundaries.
2. Identify the signals that suggest a split is needed: independent change cadence, different scaling profiles, different failure modes.
3. Distinguish between genuine reasons to split (Conway's law, operational maturity) and false reasons (symmetry, premature optimization).
4. Understand the microservice trap: why distributed systems are vastly harder than monoliths, and when they're worth the cost.
5. Recognize the modular monolith as a middle ground: strong module boundaries in a single process.
6. Evaluate split decisions by their true costs and benefits, not by rules of thumb.
7. Know how to reverse a split decision if the premises change.

---

## The Boundary Types in Increasing Cost

Imagine an order-processing system. As the codebase grows, you face a sequence of choices about where to draw boundaries.

```
Function boundary (cheapest, lowest isolation)
    ↓
Class boundary
    ↓
Module/package boundary
    ↓
Process boundary
    ↓
Network boundary (microservice)
    ↓
Trust boundary
    ↓
Organizational boundary (most expensive, highest isolation)
```

Each boundary buys isolation. Each boundary costs in latency, debuggability, and operational complexity.

### Function Boundary

Extracting logic into a function.

```cpp
// Before: inline
void processOrder(Order& order) {
    // 20 lines of validation
    // 15 lines of calculation
    // 10 lines of persistence
}

// After: extracted function
void processOrder(Order& order) {
    validateOrder(order);
    calculateTotal(order);
    persistOrder(order);
}

void validateOrder(Order& order);
void calculateTotal(Order& order);
void persistOrder(Order& order);
```

**Cost:** None. One stack frame. Debugger can step into it. Refactoring is free (rename, move the function, extract again).

**Isolation:** Minimal. The caller and function share memory, so state changes are visible immediately.

**When to use:** Always. The function boundary is free. There is no such thing as "too many functions."

### Class Boundary

Grouping related functions into a class with shared state.

```cpp
class OrderProcessor {
private:
    Database& db;
    Logger& log;
    Validator& validator;
    
public:
    void process(Order& order) {
        validate(order);
        calculate(order);
        persist(order);
    }
};
```

**Cost:** Negligible at runtime. One vtable lookup if the class uses virtual functions. Slightly more complex to navigate in the IDE.

**Isolation:** Moderate. The class encapsulates state, but callers still see all public methods and can access public data.

**When to use:** Whenever multiple functions operate on the same data, or when you want to enforce encapsulation. This is the most fundamental boundary in object-oriented design.

### Module / Package Boundary

Grouping related classes into a directory or namespace, with explicit public APIs.

```cpp
// billing/src/order_processor.h
namespace billing {
    class OrderProcessor {
        // ...
    };
}

// main.cpp
#include "billing/src/order_processor.h"
using billing::OrderProcessor;
```

**Cost:** Compile-time cost (more headers to include, longer build times if not careful). Conceptual cost (developers must understand the module's API).

**Isolation:** Strong. The module defines what it exports. Callers depend only on the public API, not internal implementation.

**When to use:** When a collection of classes serves a cohesive purpose and can be understood as a unit. Typically when the module has 500+ lines of code or is reused by multiple other parts of the system.

### Process Boundary

Running logic in a separate OS process on the same machine.

```cpp
// main.cpp
Process billingWorker = fork_process("./billing_worker");
billingWorker.send_message({order_id: 123});

// billing_worker.cpp
int main() {
    while (true) {
        auto msg = receive_message();
        processOrder(msg.order_id);
    }
}
```

**Cost:** Process creation (milliseconds on startup, negligible per-request after that). Memory overhead (each process has its own heap, globals, etc.). IPC cost (serialization, socket communication, a few milliseconds per round-trip). Debugging is harder (separate processes in the debugger, harder to trace across processes).

**Isolation:** Very strong. Processes have separate memory spaces. A crash in one doesn't affect the other. You can restart the billing worker without restarting the main app.

**When to use:** When two components have radically different reliability or scaling requirements. The main app needs to respond to web requests in 100ms; the billing worker can afford to batch and run every 10 minutes. One should fail without bringing down the other.

### Network Boundary (Microservice)

Running logic in a different machine, accessed via HTTP or gRPC.

```cpp
// Frontend calls billing service over the network
auto response = http_client.post("http://billing-service/order", order_json);

// Billing service on a different machine
void POST /order(const json& order_json) {
    auto order = parse(order_json);
    processOrder(order);
    return {success: true};
}
```

**Cost:** Network latency (10-100ms per round-trip over LAN, 100ms-1000ms over WAN). Serialization/deserialization cost (JSON parsing, encoding, allocation). Network failures are now possible (timeouts, connection refused, DNS failure). Debugging is much harder (distributed tracing required). Operational complexity (multiple deployments, multiple databases, data consistency becomes hard).

**Isolation:** Extreme. Microservices are independent. You can deploy one without deploying the others. You can scale the billing service without scaling the frontend.

**When to use:** Almost never, until you have experienced all the following pain:
- You cannot scale the components independently with a monolith.
- You need to deploy components on different schedules.
- You have a team large enough to sustain multiple deployments (often called Conway's law: architecture mirrors team structure).
- You have operational maturity (monitoring, distributed tracing, automated deployments, automated rollbacks).

### Trust Boundary

Splitting code to separate trusted from untrusted logic.

```cpp
// Untrusted user input
std::string name = get_user_input();

// Trust boundary: validate at the edge
if (!is_valid_name(name)) {
    return error("Invalid name");
}

// Trusted: inside the system, assume valid
class User {
    std::string name;  // Assumed valid
public:
    User(const std::string& n) : name(n) {}  // No validation here
};
```

**Cost:** Design overhead (must be explicit about what's trusted).

**Isolation:** Prevents invalid state from propagating.

**When to use:** Always, at the boundaries between user input and the system. Once data has passed validation, it can be treated as trusted.

### Organizational Boundary

Splitting code between teams so each team owns a codebase.

```
Team A (Frontend):
    - Web UI
    - Frontend API client

Team B (Backend):
    - HTTP API
    - Database queries

Team C (Data):
    - Analytics ETL
    - Reporting queries
```

**Cost:** Communication overhead (meetings, code reviews across teams). Merge conflict management. Deployment coordination (if teams depend on each other). Potential for divergent code standards.

**Isolation:** Very strong. Each team can work independently, deploy independently (if their boundaries are clean).

**When to use:** When you have more than one team working on the same system, or when you anticipate multiple teams. This is described by Conway's law: the structure of a system reflects the structure of the organization that built it. If your team structure mirrors the system boundaries, you can work efficiently. If they don't, you'll spend time in coordination meetings instead of writing code.

### Cost-Isolation Tradeoff

The following chart shows the relationship:

```
Operational Cost (debugging, latency, deployment)
^
|               Network (microservice)
|              /
|            Trust / Org boundary
|           /
|        Process boundary
|       /
|    Module boundary
|   /
| Class boundary
|/
Function boundary ----+-----> Isolation (independence, scalability)
```

A function boundary is free in cost but offers no real isolation. A network boundary offers extreme isolation but costs significantly in latency, debugging, and operational complexity. The best boundary for your system is the one where the cost of drawing it is less than the cost of *not* drawing it.

---

## When To Split: The Signals

You don't split because the codebase *might* grow, or because a tutorial recommended it. You split when the cost of not splitting exceeds the cost of splitting.

### Signal 1: Independent Change Cadence

The strongest signal is when two pieces of code change for unrelated reasons at different rates.

**Example:** An order-processing system has a Notification service (sends emails, SMS) and a Billing service (charges credit cards).

- The Notification service changes when you add a new channel (Slack, Telegram) or adjust templates.
- The Billing service changes when payment processors change, fraud rules change, or tax laws change.
- These changes are genuinely independent. A new email template doesn't touch billing logic, and a new payment processor doesn't touch notification templates.

If both live in the same codebase and the same deployment, every notification change requires a billing test, and every billing change requires a notification review. The cost accumulates.

If they are split (either as separate modules or services), changes are local. A notification developer can commit without involving billing.

**Decision:** If you observe that commits to component A and component B come from different developers with different motivations, the split is probably justified.

### Signal 2: Different Scaling Profiles

Your system receives 10,000 requests per second. 9,999 are reads (viewing the catalog). 1 is a checkout (creating an order).

The read path needs to be horizontal-scaled — add servers to handle load. The checkout path needs to be rock-solid and transactional, but doesn't need to handle the same volume.

If both live in the same process, scaling is difficult. You either scale the entire process (reading huge overhead) or you build a complex routing layer. If they are split, you scale the read servers and the write servers independently.

**Decision:** If two components would benefit from different scaling strategies (one needs many replicas, one needs none), consider a split.

### Signal 3: Different Failure Modes

A component's reliability requirements differ.

**Example:** A web app serving user requests needs 99.9% uptime. A background job that sends daily digest emails can afford to fail occasionally and be retried.

If both run in the same process, a bug in the email job can crash the entire app and lose user traffic. If they are separate, the email job can fail without affecting the web server.

**Decision:** If one component's failure would make the system unusable while another can degrade gracefully, consider splitting.

### Signal 4: Different Stakeholders

Multiple people or teams depend on the component, with different needs.

**Example:** A Company's data platform is used by three groups:
- Data analysts (need fast query results)
- Data engineers (need reliable ETL pipelines)
- Financial teams (need auditable, correct numbers)

All three groups depend on the same data, but they have different SLAs and concerns. Analysts want to experiment with new queries quickly. Engineers want reliable, debuggable pipelines. Finance wants no surprises, ever.

A monolithic "data platform" service forces all three to move together. If analysts demand a schema change, engineers must vet it, finance must approve it. Friction increases.

If the system is split into separate services (query service, ETL service, audit service), each team can move at its own pace.

**Decision:** If more than one organizational group depends on a component and their needs conflict, that's a signal to split.

### Signal 5: Conway's Law Alignment

If your team structure already mirrors the desired system boundaries, the split becomes easier.

You have a frontend team (5 people), a backend team (8 people), and a data team (3 people). Each team would naturally own a separate service. In this case, the organizational structure already suggests the boundaries. The split enables the teams to work independently.

Conversely, if you have one monolithic team today but split the code into microservices, the team structure won't align with the code structure. This creates friction: developers from the monolithic team end up touching multiple microservices, debugging becomes cross-team, deployments require coordination.

**Decision:** Draw code boundaries where your team boundaries already exist. Splitting the code too far ahead of the team structure creates friction, not benefit.

### Signal 6: The Rule of Three

If the same logic is needed in three separate places, it's a candidate for extraction into its own module.

**Example:** Order validation logic is needed in:
1. The order API (when a customer submits an order)
2. The admin UI (when an admin manually creates an order)
3. The batch importer (when orders are bulk-imported from a legacy system)

The first time you duplicate logic, it might be a coincidence. The second time, you see the pattern and extract into a shared function. By the third time, you have strong evidence that the logic belongs in its own cohesive module.

**Decision:** If a piece of logic is genuinely needed in three or more places, factor it into its own module.

---

## When NOT To Split: The Anti-Signals

Splitting for the wrong reasons creates complexity without benefit.

### Anti-Signal 1: Symmetry for Its Own Sake

"We have a billing module, so we should have a shipping module and a reporting module, even though they're tiny."

Symmetry looks elegant in diagrams. But if a module is 200 lines of code and doesn't change independently, it's unnecessary. You are paying in navigation overhead (more files to open) and conceptual overhead (more modules to understand) without gaining anything.

**Decision:** Don't split until the natural growth of the codebase demands it. A single 3,000-line module that changes as a unit is better than three 1,000-line modules that don't.

### Anti-Signal 2: "It Might Grow Someday"

"The notification service might be reused in other products, so let's split it now."

This is YAGNI (You Aren't Gonna Need It) applied to architecture. You are adding complexity for a hypothetical future that may never arrive. If the notification service is split, you immediately pay the cost of IPC, deployment, and debugging. If it turns out to be reused, great — you paid for something you needed. If it isn't, you paid for speculation.

**Decision:** Build it in the same module. If it genuinely gets reused, extract it later. Extracting later is more work (you must ensure clean interfaces), but extracting early on false premises wastes more time.

### Anti-Signal 3: Following a Tutorial

"The microservices tutorial showed a user service, an order service, and a billing service, so we'll implement those three."

Tutorials teach architecture patterns in isolation. A real system has context: team size, deployment maturity, budget, expected growth. A tutorial doesn't know your context. A pattern that works for Netflix might be overkill for a startup.

**Decision:** Design the boundaries based on your actual constraints, not a template.

### Anti-Signal 4: Splitting Because "Single Responsibility Principle"

"Single Responsibility means one class should do one thing, so let's make separate services for each thing."

Single Responsibility is about cohesion *within* a class, not about splitting services. A service with one responsibility is brittle — most real operations involve multiple responsibilities. Billing requires validation, persistence, and notification. If each is a separate service, a billing operation becomes a distributed transaction, which is much harder.

**Decision:** Keep related logic together. SRP applies to classes and functions, not to service architecture.

### Anti-Signal 5: Splitting to Avoid Refactoring

"The codebase is a mess, so let's split it into microservices."

This is a form of avoidance. The real problem is that the codebase is a mess. Splitting it doesn't fix the mess; it just distributes the mess across multiple processes. You still have tangled dependencies, low cohesion, and high coupling — now with added network latency.

**Decision:** Fix the architecture first (better modules, clearer boundaries). Only split if the problems persist after refactoring.

---

## The Microservice Trap

Microservices are not an architecture; they are an operational model. They are worth considering only after you have mastered monolithic architecture and hit its real limitations.

### The Hidden Costs

The microservices industry marketing emphasizes independence and scalability. The reality:

**Distributed systems are fundamentally harder than monoliths.** Here are the costs:

1. **Network calls fail.** A function call always succeeds or throws an exception, which you can catch. A network call can time out, connection refused, DNS fail, or hang indefinitely. You must handle all these cases, with retries, circuit breakers, and timeouts.

2. **Data consistency is hard.** In a monolith, a transaction ensures consistency across multiple entities. In microservices, you cannot use transactions across services. You must use eventual consistency, sagas, or distributed transactions (which are even harder). Managing inconsistent state is much more complex.

3. **Debugging is an order of magnitude harder.** In a monolith, you stack trace from the entry point through all calls. In microservices, you must reconstruct the call path from logs across multiple services, requiring distributed tracing infrastructure.

4. **Deployments are more complex.** Each service has its own deployment pipeline. Coordinating releases becomes a project. Backwards compatibility becomes critical (you cannot deploy both services simultaneously; one will be on the old version for a moment). Rollbacks are harder (if you deploy service A, then service B, and B fails, you must coordinate both rollbacks).

5. **Operational overhead is immense.** Microservices require:
   - Multiple databases (and consistency between them)
   - Service discovery (how does service A find service B?)
   - Load balancing (distributing requests across replicas)
   - Monitoring (each service has its own logs, metrics, health checks)
   - Distributed tracing (reconstructing request flows)
   - Container orchestration (Kubernetes or similar)
   - Secrets management (each service has credentials)

   A small startup team cannot manage this. You need DevOps engineers, SREs, and a whole operational infrastructure. The total cost of ownership is 3-5x higher than a monolith.

6. **Latency is higher.** Network calls add 10-100ms each. A flow that involved 5 function calls (instant) now involves 5 network calls (50-500ms). Users notice. Your response times degrade.

### When Microservices Are Worth It

Microservices become justified when:

1. **You have experienced monolith pain at scale.** You tried to scale the entire monolith and hit a wall. Specific components need to scale independently, and splitting them into separate services is the only way.

2. **You have the operational maturity.** You have built and run distributed systems. You have monitoring, tracing, and automated deployments. You understand failure modes and have built systems to handle them.

3. **Your team is large enough.** If you have fewer than 3-4 teams, a monolith with clear module boundaries is simpler. Microservices shine with 10+ teams, where code ownership is clear and coordination overhead justifies the operational cost.

4. **Your components truly decouple at the network level.** Not everything decouples. A billing system and an inventory system are tightly coupled by business logic (you can't ship an order without billing). Splitting them helps nothing and hurts everything. But a real-time chat system and a batch reporting system are truly independent — splitting helps.

### The Honest Assessment

If you are considering microservices, ask yourself:

- Can we scale different components independently in a monolith (with separate processes or load balancers)?
- Do we have monitoring, tracing, and automated deployments in place?
- Do we have on-call engineers who understand distributed systems?
- Have we actually hit scaling limits in the monolith?

If you answer "no" to any of these, microservices will cost more than they save.

---

## The Modular Monolith Alternative

There is a middle ground that is often overlooked: **the modular monolith**. Strong module boundaries within a single process.

```cpp
// Each module has a clear public API (the header)
// module_a/public.h
namespace module_a {
    class OrderProcessor {
        void process(const Order& order);
    };
}

// module_a/internal/* (implementation details, not exposed)

// main.cpp
#include "module_a/public.h"
#include "module_b/public.h"

int main() {
    module_a::OrderProcessor proc;
    module_b::Notifier notifier;
    // ...
}
```

### Benefits

- **Cheap to refactor.** If module A needs to call module B, the refactoring is one #include and one function call. No network calls, no serialization.
- **Single deployment.** All modules deploy together. No coordination needed.
- **No distributed data consistency.** A transaction can span multiple modules. Invariants are enforced atomically.
- **Easy debugging.** Stack traces show the full flow. Print debugging works everywhere.
- **Scalable.** Deploy multiple replicas of the entire monolith behind a load balancer. This scales you to 100k+ requests per second.

### Constraints

- **One language.** All modules are in the same language (C++ in this case). You can't have a module in Python. This is fine for most systems.
- **Shared process failure.** A bug in one module crashes the entire process. This is actually fine in practice — if your app is crashed, everything should fail anyway. Use monitoring to detect and restart.

### When It's the Right Choice

A modular monolith is the right choice for:
- Startups and small teams
- Systems that don't yet have scaling bottlenecks
- Systems with tight data consistency requirements
- Teams without strong operational maturity

Many successful systems never outgrow a modular monolith. Monoliths at scale (Shopify, Etsy) often use this model, not microservices.

---

## Worked Example: The Order System

Let's design an order-processing system with three split decisions: extracting `Notifier`, extracting `Billing`, and splitting `Reporting`.

### The Basic Monolith

```cpp
// order_service.h
class OrderService {
private:
    Database& db;
    EmailService& email;
    
public:
    void processOrder(const Order& order) {
        validateOrder(order);
        db.saveOrder(order);
        email.sendConfirmation(order);
        chargeCard(order);
    }
    
private:
    void chargeCard(const Order& order) {
        // 50 lines of billing logic
        // Integration with payment processor
        // Retry logic, error handling
    }
};
```

### Decision 1: Extract Notifier as a Module (Not a Service)

**Should we split notifications into a separate service?**

- **Change cadence:** Notification templates change often (marketing owns this). Billing changes less frequently (finance owns it).
- **Scaling:** Notifications can batch; billing must be synchronous.
- **Failure mode:** Failed notification is retryable; failed billing is critical.
- **Stakeholders:** Multiple features might need notifications.

**Conclusion:** Extract as a module, not a service.

```cpp
// notifications/public.h
namespace notifications {
    class Notifier {
        void sendOrderConfirmation(const Order& order);
        void sendShippingUpdate(const std::string& order_id);
        void sendCancellationNotice(const std::string& order_id);
    };
}

// order_service.h
#include "notifications/public.h"

class OrderService {
private:
    Database& db;
    notifications::Notifier& notifier;
    
public:
    void processOrder(const Order& order) {
        validateOrder(order);
        db.saveOrder(order);
        notifier.sendOrderConfirmation(order);
        chargeCard(order);
    }
};
```

**Cost:** One extra #include. One indirection in the function call (negligible).

**Benefit:** Notification templates can be updated without touching order processing. Notification developers own this module. Testing notifications doesn't require setting up the full order system.

### Decision 2: Keep Billing in the Same Module (For Now)

**Should we split billing into a separate service?**

- **Change cadence:** Billing and orders change together (each order operation involves billing).
- **Scaling:** Both need to handle the same load (every order has a billing operation).
- **Failure mode:** If billing fails, the entire order fails. They're not independent.
- **Stakeholders:** Billing is internal; only order processing uses it.

**Conclusion:** Keep in the monolith as a separate module, not a service.

```cpp
// billing/public.h
namespace billing {
    class BillingEngine {
        struct ChargeResult {
            bool success;
            std::string transaction_id;
            std::string error;
        };
        
        ChargeResult chargeOrder(const Order& order);
    };
}

// order_service.h
#include "billing/public.h"

class OrderService {
private:
    Database& db;
    notifications::Notifier& notifier;
    billing::BillingEngine& billing;
    
public:
    void processOrder(const Order& order) {
        validateOrder(order);
        
        auto charge = billing.chargeOrder(order);
        if (!charge.success) {
            throw std::runtime_error("Billing failed: " + charge.error);
        }
        
        db.saveOrder(order);
        notifier.sendOrderConfirmation(order);
    }
};
```

**Cost:** One extra module interface to understand.

**Benefit:** Billing logic is isolated. We can mock it for testing order processing. If we later need billing from another service (admin panel, API), we can use the same interface. If billing complexity grows, we can refactor it internally without touching order processing.

### Decision 3: Split Reporting to a Separate Service (or Repository)

**Should we split reporting into a separate service?**

- **Change cadence:** Reports are generated on demand; order processing is real-time. They change independently.
- **Scaling:** Reporting can be slow (queries take seconds). Order processing must be fast (ms). They have different scaling needs.
- **Failure mode:** Failed report is not critical. Failed order processing is critical.
- **Stakeholders:** Finance team owns reporting, product team owns orders.

**Conclusion:** Extract as a separate read-only service or repository.

```cpp
// Option A: Module within monolith (if reporting is simple)
// reports/public.h
namespace reports {
    class ReportGenerator {
        struct OrderReport {
            int total_orders;
            double total_revenue;
            int orders_last_24h;
        };
        
        OrderReport generateOrderReport(const std::string& date);
    };
}

// Option B: Separate service (if reporting becomes complex)
// Reporting service (separate process/container)
class ReportGenerator {
    Database& db;  // Read replica of the main database
    
public:
    OrderReport generateOrderReport(const std::string& date) {
        // Query database
        // Build complex aggregations
        // Return report
    }
};

// main service calls it
auto report = http_client.get("http://reports-service/report?date=2025-01-01");
```

**Cost (as module):** Minimal. One extra include.

**Cost (as service):** Significant. Separate database, separate deployment, network call, eventual consistency (the reporting database might lag behind the main database).

**Benefit (as module):** Simple to implement, no operational overhead. Works well if reporting is simple.

**Benefit (as service):** Reports don't slow down the main service. Reporting database can be optimized for analytics (different schema, indices). Reporting team can deploy independently.

**Recommendation:** Start as a module. If reporting queries start blocking order processing, or if the reporting team grows large, extract to a service.

---

## How To Reverse a Boundary Decision

Splitting later is harder in some ways and easier in others.

### Extracting Later (Monolith → Module)

**Easier because:**
- You learned what the boundaries should be by living with the monolith.
- The API is clear; you know what to expose.
- No network calls means the refactoring is mostly mechanical.

**Harder because:**
- The code may have tangled dependencies (order processing calls billing calls notifications). Untangling requires work.
- You must audit all callers to ensure they use the public API, not internal details.

**Process:**
1. Identify the module's responsibilities.
2. Create a `public.h` header with the module's API.
3. Move implementation details into an `internal/` subdirectory.
4. Update callers to import only `public.h`.
5. Refactor internal calls to use the public API.

### Extracting Later (Module → Service)

**Easier because:**
- You know the exact interface the service should expose (it was already the module's public API).
- You can migrate gradually: keep the module, add a service wrapper, gradually move callers to the service.

**Harder because:**
- You now have two deployments to coordinate.
- Network calls are slower than function calls.
- You must handle distributed failures (timeouts, retries, circuit breakers).

**Process:**
1. Create a separate service with the same interface as the module.
2. Have both the monolith and the service read from the same database.
3. Route new callers to the service.
4. Gradually migrate old callers.
5. Once all callers point to the service, delete the module.

### Reversing a Split (Service → Monolith)

If you split too early and realize it's not worth the cost, merging back is possible but awkward.

**What's hard:**
- The services have separate codebases and deployment pipelines.
- They might have different dependencies (one uses library A, the other uses library B).
- They might have drifted in their interfaces and assumptions.

**Process:**
1. Identify which service logic is actually coupled to another service.
2. Plan the merge: where will the code live in the monolith?
3. Create feature flags so you can deploy the merged code without breaking the split code.
4. Migrate database writes to use the unified schema.
5. Once all traffic flows through the merged code, delete the separate services.

**Lesson:** Start with a monolith (or modular monolith). Extract to services only after you understand the boundaries. Reversing a premature split is expensive.

---

## Conway's Law in Practice

Conway's law states: "Any organization that designs a system will produce a design whose structure is a mirror image of the organization's communication structure."

In practice:

- **If you have one team:** One monolith, possibly with internal modules.
- **If you have two teams (frontend and backend):** Two services: frontend and backend.
- **If you have three teams (frontend, backend, data):** Three services: frontend, backend, data.

The key insight: if your team structure and your code structure are misaligned, you'll spend time in coordination meetings. If they're aligned, teams can move independently.

**Example:** You split billing into a separate service because it seemed like a good idea. But the only team member who understands it is Alice from the backend team. Now Alice is on call for billing issues, attending meetings with two teams, and responsible for coordinating deployments. The split created friction.

If instead you had split billing when a dedicated billing team was hired, the split would have aligned with the organization. Alice would be part of the billing team. The split would reduce coordination overhead, not increase it.

---

## Tradeoffs Summary

| Boundary Type | Cost | Isolation | When to Use |
|---|---|---|---|
| Function | None | None (shared memory) | Always. Extract freely. |
| Class | Negligible (one vtable lookup) | Moderate (encapsulation) | Whenever multiple methods operate on shared state. |
| Module | Compile-time (more headers); Navigation | Strong (clear API) | When a coherent group of classes serves a purpose; 500+ lines of code. |
| Process | Memory (separate heap); IPC (10-100ms) | Very strong (separate memory space) | Different scaling or reliability needs; need to isolate failures. |
| Network (Microservice) | Latency (100ms+); Serialization; Deployment complexity; Operational overhead (3-5x cost) | Extreme (independent deployment, scaling) | Only after hitting monolith limits. Requires operational maturity and large teams. |
| Trust | Design overhead | Prevents invalid state | Always, at input boundaries. |
| Organizational | Communication overhead; Coordination | Strong (independent teams) | When team structure already aligns with code structure. |

---

## Common Misconceptions

**"Smaller services are always better."**
No. There is an inflection point past which the operational cost of services exceeds the benefits of independence. The optimal service size is "large enough that the cost of changing it is higher than the cost of coordinating with other teams." For a startup, this might be the entire monolith. For a large company, it might be a few services. For a tech giant, it might be hundreds of tiny services (because they have the infrastructure to manage them).

**"Microservices mean you can use different languages."**
You can use different languages in a monolith too (if you have IPC). And you probably shouldn't. Mixed-language codebases are harder to maintain. People specialize in languages; you force developers to learn new ones. Stick to one language unless you have a very good reason (and different languages are a smell that you need the specialization, not the language).

**"Monoliths don't scale."**
Monoliths scale fine. Shopify, Etsy, and GitHub are monoliths (or mostly monoliths) handling billions of requests per second. What doesn't scale is a monolith with poor architecture — low cohesion, high coupling, tangled dependencies. Fix the architecture first; split second.

**"Splitting makes testing easier."**
It can, but not for the reasons people think. Splitting makes testing easier if the split *improves the architecture* — reduces coupling, increases cohesion. Splitting for its own sake doesn't help testing; distributed testing is harder (mocking network calls, handling timeouts). The benefit is isolation, not testing ease.

**"Once you've split, you can never merge back."**
False. You can merge back, though it's awkward. Use feature flags to migrate traffic gradually. It takes time and care, but it's reversible. The lesson: don't optimize for a future you're not certain about.

---

## Exercises

1. **Map your system's boundaries.** Draw a box for each module or service in your system. Draw arrows showing dependencies. Are there circular dependencies? Are there modules that depend on many others? Could any be merged?

2. **Identify independent change cadences.** In the last 10 commits to your codebase, how many touched different modules? For each pair of modules, estimate how often they change together. Modules that rarely change together are candidates for splitting.

3. **Calculate the team alignment cost.** Does your team structure match your code structure? If not, estimate how many hours per week are spent in cross-team coordination. If it's significant, that's a signal to realign code boundaries.

4. **Simulate a split.** Pick a module you might want to extract. Estimate the cost: new tests, IPC latency, deployment overhead. Estimate the benefit: faster iteration for that team, independent scaling. Does the benefit outweigh the cost?

5. **Find a forced split.** Locate a place where your code calls another module synchronously and would benefit from being asynchronous (a slow database query, a call to an external API). Design how you would split this into separate components. Do you need a service, or would a background job suffice?

6. **Reverse a split (design exercise).** If you had to merge one of your services back into the monolith, what would that look like? What would be the hardest parts? What would be easy?

---

## Summary

Every boundary is a choice about where to draw a line and what it costs. The simplest boundary is a function (free, no isolation). The most expensive is a network boundary (high cost, extreme isolation). The right boundary is the one where not drawing it costs more than drawing it.

The strongest signals for a boundary are independent change cadence, different scaling profiles, different failure modes, and alignment with team structure. False signals — symmetry, premature optimization, following tutorials, misapplying SRP — create complexity without benefit.

Microservices are powerful but expensive. They are worth considering only after you have hit scaling limits in a monolith and have the operational maturity to manage them. A modular monolith — strong boundaries within a single process — is often the right balance.

Design boundaries based on your actual constraints and experience, not on templates. Start with a monolith. Extract modules when the codebase demands it. Extract services only when you understand the problem deeply and have the operational maturity to sustain them. If you split too early, you can merge back; it's awkward, but it's reversible.

The system that will last is the one whose boundaries match the organization's structure and the problem's nature, not the system that followed the latest architectural trends.

---

> **[← Previous: Why Frameworks Use DI](10-why-frameworks-use-di.md)**  ·  **[↑ Part 4](README.md)**  ·  **[Next: Organizing Large Codebases →](12-organizing-large-codebases.md)**
