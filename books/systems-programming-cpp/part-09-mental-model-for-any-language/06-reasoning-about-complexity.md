# Chapter 96 — Reasoning About Complexity

Every complaint that "this codebase is complex" refers to one of two specific problems, even if the speaker does not realize it. The first: you must hold too many concepts in your head at once (cognitive load). The second: your change ripples farther than it should, affecting code you did not expect to touch (coupling depth). This chapter teaches you the vocabulary to be precise about which problem you are facing, and more importantly, how to reason about it systematically.

Most engineers confuse "this system has a lot of code" with "this system is hard to understand." They are not the same. A 100,000-line compiler can be easier to reason about than a 5,000-line monolith where everything depends on everything else. The difference is not size; it is *structure*. This chapter gives you the tools to evaluate structure and to make it visible to yourself and others.

---

## 96.1 The Two Dimensions of Complexity

When you say a system is "complex," you are actually identifying one or both of these problems.

**Cognitive Load: Holding Concepts in Your Head**

Cognitive load is the number of concepts, invariants, and relationships you must simultaneously understand to make a safe change. A function with 200 lines of nested conditionals, each depending on the state of six variables, has high cognitive load. You must understand all six variables, all twelve branches, all their interactions. A single mistake in one branch can silently break another. The "safe" size of a function is not arbitrary; it is the size you can understand and trace through in 5–10 minutes without a second read.

Examples of high cognitive load:
- A function with 15+ parameters (you cannot remember them all).
- A class with mutable state in five different places that interact indirectly.
- A data structure where invariants are not documented ("the count must equal the sum of all sizes, except when a recompute is in progress...").
- A business logic function where the order of statements matters because of hidden dependencies.
- A system where the same concept is named differently in different places (is it a "user," an "actor," or an "entity"?).

**Coupling Depth: How Far Your Change Ripples**

Coupling depth is how many other modules or systems your change affects. A change to Module A that requires changes in Modules B, C, D, and F has deeper coupling than a change that affects only Module A and B. More importantly, if changing B forces you to understand E and F (because B depends on them), the coupling depth is the longest chain, not the sum of dependencies.

Examples of deep coupling:
- Changing a data structure in one service breaks queries in three others that depend on its schema.
- Modifying validation logic in the authentication module requires changes in six call sites across four different services.
- A "global" caching layer that many modules depend on; invalidating it correctly requires understanding all those modules.
- A shared database schema where removing a column breaks code in five different services, each with subtle assumptions about that column.

### Why These Matter Separately

A system can have high cognitive load but low coupling depth. A deeply nested state machine within a single, well-isolated module has high cognitive load (understanding the state machine is hard) but low coupling depth (changing it does not affect other modules).

Conversely, a system can have low cognitive load but deep coupling. A REST API with clean, simple endpoints can be trivially easy to understand (cognitive load is low), but changing the response format might break ten clients that parse it (coupling depth is high).

The worst case is high cognitive load *and* deep coupling: a tangled module that is hard to understand internally *and* coupled to many other modules. This is the "Big Ball of Mud" from Part 4. The best case is low in both: clear concepts, well-isolated boundaries.

---

## 96.2 Tools to Quantify Complexity

You can measure complexity objectively. These measurements are not perfect, but they are useful smoke detectors. When one signals high, it is worth investigating.

### Cyclomatic Complexity (Control-Flow Paths)

Cyclomatic complexity counts the number of linearly independent paths through a piece of code. It is calculated as: 1 + (number of decision points).

```cpp
// Cyclomatic complexity = 1
int max(int a, int b) {
    return a > b ? a : b;
}

// Cyclomatic complexity = 2 (if and else)
int classify(int x) {
    if (x > 0)
        return 1;
    else
        return -1;
}

// Cyclomatic complexity = 4 (three if statements, each adds 1)
int validate(int age, int income, bool hasLicense, bool hasInsurance) {
    if (age < 18) return false;
    if (income < 10000) return false;
    if (!hasLicense) return false;
    if (!hasInsurance) return false;
    return true;
}
```

The rule of thumb: cyclomatic complexity above 10 is a warning sign. Above 20 is almost certainly a design problem.

**Why it matters**: Each path is a potential bug. With 20 paths, you have 20 things that can go wrong. Testing all of them requires exponential test cases. The number of paths can double with each added condition.

**Why it is incomplete**: A function with low cyclomatic complexity can still have high cognitive load if each branch is subtle or if variables interact in non-obvious ways. Conversely, a function with high cyclomatic complexity (like a switch statement with 30 cases) might be straightforward to understand if each case is independent and simple.

### Cognitive Complexity (Sonar)

Cognitive complexity is a more nuanced measure that counts how hard it is to *follow* the code, not just how many paths exist.

It increases by:
- Each nesting level (nested loops count more than sequential ones).
- Certain structural elements (recursion, exception handling, complex boolean expressions).
- "Cognitive friction" — elements that require mental overhead to understand.

Example: a switch statement with 30 cases has low cognitive complexity (one case per branch, no nesting). A deeply nested if-else ladder with the same number of branches has higher cognitive complexity (you must hold the entire nesting context in your head).

**Why it matters**: It correlates better with defect density than cyclomatic complexity alone. A function with high cognitive complexity is more likely to have bugs.

**Where to find it**: Sonar (sonarqube.org) calculates it automatically. Many IDEs have plugins.

### Halstead Metrics (Code Volume)

Halstead metrics measure the "vocabulary" and "volume" of code. The vocabulary is the number of unique operators and operands. The volume is related to the total length.

The key metric: **Volume = Code Length × log2(Vocabulary)**

A function that uses few distinct operators and operands but is written verbosely has high volume relative to its functionality. This suggests redundancy or verbosity.

**Why it matters**: High volume can indicate code that could be simplified. It is not a strict rule — sometimes verbosity is good for clarity — but it is a signal.

**Why it is incomplete**: It does not distinguish between necessary complexity (the problem domain is hard) and accidental complexity (the code is poorly structured).

### Coupling Metrics (Dependency Count)

For the second dimension (coupling depth), count the number of modules or services that depend on a given module.

More precisely:
- **Incoming coupling**: How many modules depend on this one? (Also called "fan-in")
- **Outgoing coupling**: How many modules does this one depend on? (Also called "fan-out")

**The rule**: A module should have low fan-out (few outgoing dependencies) and moderate fan-in (multiple things can use it, but not everything).

High fan-out is a design problem: the module is a "hub" that must be updated whenever any of its dependencies change. High fan-in in a badly designed module is a problem: many things depend on it, so changing it is risky. (High fan-in in a *well-designed* module is fine; it means the module is useful.)

**Why it matters**: High fan-out indicates tight coupling. If Module A depends on B, C, D, E, and F, then changing any of those will affect A. The cost of change grows multiplicatively.

### A Checklist: Using These Together

None of these metrics is sufficient alone. Use them as a checklist:

- **High cyclomatic complexity (>10)?** The function probably has too many paths. Consider extracting cases into separate functions or simplifying the control flow.
- **High cognitive complexity?** The code is hard to follow. Look for deeply nested structures or complex boolean expressions. Refactor for clarity.
- **High volume relative to vocabulary?** The code is verbose. Look for duplication or unnecessarily long expressions.
- **High fan-out (>3–4 dependencies)?** The module is tightly coupled. Consider breaking it into smaller pieces or consolidating its dependencies.
- **High fan-in in a poorly designed module?** Many things depend on it, and it is hard to change. This is a design smell. Consider whether this module is too broad or whether its interface is poorly designed.

---

## 96.3 Complex vs Complicated

There is an important distinction between *complex* and *complicated*.

**Complicated** means having many parts, all of which are individually understandable. A watch is complicated: it has many gears, springs, and levers, but each one behaves in a predictable way. If you understand gears, you can understand how the watch works. The path is clear: start at the mainspring, follow the gear train, observe the escapement, see how it moves the hands. Complicated systems are hard to build but relatively easy to understand once built.

**Complex** means the behavior of the system cannot be understood by studying its parts in isolation. An ecosystem is complex: knowing all about trees does not tell you how the system will respond to climate change. A social network is complex: understanding individual decision-making does not predict viral spread. Complex systems require simulation or empirical observation to understand.

In software:

**A complicated codebase** has many modules, classes, and functions, but each one can be understood independently. The ordering system is complicated: it has modules for payments, shipping, inventory, tax calculation, and notifications. But each module is understandable in isolation. To understand the system, you trace one path through it (an order being created, confirmed, and shipped). The path is long, but each step is clear.

**A complex codebase** has modules whose behavior depends on the state of other modules in ways that are not obvious from reading the code. The user model interacts with the cache in a way that creates a subtle race condition. The scheduling service depends on the queue, which depends on the database, which affects the cache, which affects the scheduling service. You cannot understand any one part without simulating the system in your head.

**You should strive for complicated, not complex.**

Complicated is manageable. You can document it ("here is the path a payment takes"). You can test it systematically (each module can be tested in isolation, then the integration tested). You can refactor it (replace one module with another, so long as the interface is the same).

Complex systems are inherently harder to manage. You cannot fully test them. Refactoring is risky because the effects are non-obvious. The only way to truly understand them is to run them and observe the behavior.

### How Codebases Become Complex

Complexity creeps in through:

- **Shared mutable state**: Two modules both read and write the same data structure, creating dependencies on the order of operations.
- **Circular dependencies**: Module A imports Module B imports Module C imports Module A. The modules cannot be reasoned about separately.
- **Undocumented invariants**: The code assumes "the count is always even" or "this map is never empty," but these assumptions are not written down. When someone changes the code, they break the invariant silently.
- **Non-local side effects**: A function that seems pure but has hidden side effects (writes to a global, updates a cache, triggers a network request).
- **Loose coupling with tight behavior**: The modules are "decoupled" (they do not import each other), but they are tightly coupled through shared infrastructure (database, message queue) or implicit assumptions about data format.

Once these are in place, the system behaves in ways that are hard to predict without running it.

---

## 96.4 Heuristics for Reducing Complexity

Given the two dimensions (cognitive load and coupling depth), here are specific moves to reduce each.

### For Cognitive Load

**Make the implicit explicit**

If the code assumes something, write it down. If a function must be called with `count >= 0`, say so. If a data structure has invariants (all keys are in the set), document them.

```cpp
// Before: implicit assumption
void process(const std::vector<Item>& items) {
    // Assumes items are sorted by price
    // Assumes no duplicates
    // ...
}

// After: explicit assumption
void process(const std::vector<Item>& items) {
    // PRECONDITION: items are sorted by ascending price
    // PRECONDITION: no duplicate item IDs
    assert(!items.empty());
    assert(std::is_sorted(items.begin(), items.end(),
        [](const Item& a, const Item& b) { return a.price < b.price; }));
    // ...
}
```

This reduces cognitive load because you do not have to figure out the assumption from the code.

**Reduce non-obvious interactions**

If a function's behavior depends on the state of a global or an object's field, make that dependency explicit in the signature.

```cpp
// Before: non-obvious dependency
void logTransaction(const Transaction& t) {
    if (g_debug_mode) {  // Depends on a global
        // ...
    }
}

// After: explicit dependency
void logTransaction(const Transaction& t, bool debug) {
    if (debug) {
        // ...
    }
}
```

**Prefer pure functions**

A function that depends only on its inputs and has no side effects is easier to understand. It can be reasoned about in isolation. When you see a pure function, you know that calling it three times in different parts of your code will have the same effect.

```cpp
// High cognitive load: depends on state of the object
class Cache {
    bool has(const Key& k) {
        if (!is_valid()) {  // Depends on internal state
            rebuild();      // Side effect: modifies the cache
        }
        return data.count(k) > 0;
    }
};

// Lower cognitive load: pure function
bool has(const Key& k, const std::unordered_map<Key, Value>& data) {
    return data.count(k) > 0;
}
```

**Push side effects to the edges**

If you must have side effects (and you must; the whole point of software is to affect the world), confine them to specific layers. A function that does 80% computation and 20% side effects is harder to understand than one that does 100% computation, called by a function that does the side effect.

```cpp
// Before: mixed computation and side effects
void processOrderAndSave(const Order& o) {
    double tax = calculateTax(o);
    double total = o.subtotal + tax;
    if (total > 1000) {
        notifyApprover();  // Side effect in the middle
    }
    o.total = total;
    db.save(o);  // Side effect at the end
    return;
}

// After: computation separated from side effects
double computeTotal(const Order& o) {
    double tax = calculateTax(o);
    return o.subtotal + tax;
}

void processOrderAndSave(const Order& o) {
    o.total = computeTotal(o);
    if (o.total > 1000) {
        notifyApprover();
    }
    db.save(o);
}
```

**Document invariants**

An invariant is a condition that must always be true. Document them at class or module boundaries.

```cpp
// Before: invariants are implicit
class OrderService {
    std::vector<Order> orders;
    int total_count;
    // ...
};

// After: invariants are explicit
class OrderService {
    // INVARIANT: total_count == orders.size() at all times
    // INVARIANT: no order with status == CANCELLED is in orders
    std::vector<Order> orders;
    int total_count;
    // ...
};
```

### For Coupling Depth

**Identify the "unit of reasoning"**

A unit of reasoning is the smallest piece of code that can be understood independently. It might be a function, a class, a module, or a service. Make it clear what the boundaries are.

```
Unit 1: OrderService
  - Responsibility: Create, confirm, ship orders
  - Dependencies: PaymentGateway, ShippingService, Database
  - Cannot change: OrderService's public interface (other teams depend on it)

Unit 2: PaymentGateway
  - Responsibility: Communicate with Stripe/Square
  - Dependencies: Database (to log transactions)
  - Cannot change: The PricingEngine (it depends on this)
```

**Reduce outgoing dependencies**

If a module depends on many others, it has high fan-out. Each dependency is a potential coupling point. Reduce it.

```cpp
// Before: high fan-out (depends on Logger, Cache, Database, EmailService)
class OrderProcessor {
    void process(const Order& o) {
        logger.log("processing order " + o.id);
        if (cache.has(o.customerId)) {
            // ...
        }
        auto c = db.find(o.customerId);
        if (c.status == "premium") {
            emailService.send(c.email, "order confirmed");
        }
    }
};

// After: lower fan-out (depends only on Database and PricingEngine)
class OrderProcessor {
    void process(const Order& o, const IPricingEngine& pricing) {
        auto total = pricing.calculate(o);
        db.update(o.customerId, total);
    }
};
```

The logging and email are now external concerns, handled by the caller, not by OrderProcessor.

**Invert the dependency when possible**

If Module A imports Module B, then A depends on B. If B changes, A might break. Inverting this means: B exports an interface, and A implements it. Now B depends on the interface, not on A.

```cpp
// Before: OrderService depends on Database
class OrderService {
    Database db;
    void save(const Order& o) {
        db.insert(o);  // OrderService depends on concrete Database
    }
};

// After: Database (or its caller) depends on OrderService
class OrderService {
    IOrderRepository repo;  // Injected interface
    void save(const Order& o) {
        repo.persist(o);
    }
};
```

**Define clear boundaries**

If two modules communicate, define exactly what they can exchange. This prevents one module from reaching into another's internals.

```cpp
// Before: loose boundary (A reaches into B's internals)
class ModuleA {
    void process() {
        moduleB.internal_cache.clear();  // Reaching into B's internals
    }
};

// After: clear boundary (A calls B's public method)
class ModuleB {
public:
    void invalidateCache() {
        internal_cache.clear();
    }
};

class ModuleA {
    void process() {
        moduleB.invalidateCache();  // Using B's public interface
    }
};
```

---

## 96.5 A Reasoning Toolkit

When you encounter a complex system, use this toolkit to understand it.

**"What is the unit of reasoning here?"**

Is it a function? A class? A module? A service? Identify the boundary of the thing you are trying to understand. If the boundary is unclear, that is a smell. Good systems have clear units of reasoning.

**"What invariants hold at this boundary?"**

What preconditions must be true before you can use this unit? What postconditions will be true after? What properties are guaranteed to not change?

```
Unit: PaymentService
  Precondition: amount > 0, customer has valid payment method
  Postcondition: transaction is persisted to database, confirmation email sent
  Invariant: every transaction has a unique ID, status is one of {pending, success, failed}
```

**"If I change this, what depends on it?"**

Trace the dependencies. If you change the return type of a method, who else is affected? If you change a data structure, which modules must update?

**"Could this state be passed in instead of being global?"**

If a function or class depends on global state (a global variable, a class field, a service singleton), ask: could this dependency be made explicit by passing it as a parameter?

This is the dependency injection principle: prefer passing in what you need rather than reaching for it globally.

**"What would it look like if I had to explain this to someone in 10 minutes?"**

If you cannot explain a component in 10 minutes without referring to other components, the component is too large or too coupled. Break it down.

**"Is there a simpler way to achieve this without the coupling?"**

Sometimes you realize that you are coupling to something for a reason that is no longer valid. Ask whether the coupling is necessary or historical.

---

## 96.6 Worked Example: Refactoring for Reduced Complexity

Consider this function, taken from a real order-processing system:

```cpp
void processOrder(const Order& order, const std::string& coupon_code,
                 bool is_bulk_purchase, const std::string& region,
                 double& out_total, int& out_discount_percent) {
    // Calculate subtotal
    double subtotal = 0;
    for (const auto& item : order.items) {
        subtotal += item.price * item.quantity;
    }
    
    // Apply coupon (but not if bulk purchase)
    double discount = 0;
    if (!coupon_code.empty() && !is_bulk_purchase) {
        auto coupon = db.getCoupon(coupon_code);
        if (coupon.type == "percentage") {
            discount = subtotal * coupon.value / 100;
        } else if (coupon.type == "fixed") {
            discount = coupon.value;
        } else if (coupon.type == "bogo") {
            // Buy one get one: discount = price of cheapest item
            double min_price = 1e9;
            for (const auto& item : order.items) {
                if (item.price < min_price) min_price = item.price;
            }
            discount = min_price;
        }
    }
    
    // Apply bulk discount (but only if subtotal > 500)
    if (is_bulk_purchase && subtotal > 500) {
        discount += subtotal * 0.1;  // 10% additional bulk discount
    }
    
    // Apply regional tax
    double tax = 0;
    if (region == "CA") {
        tax = (subtotal - discount) * 0.0825;
    } else if (region == "NY") {
        tax = (subtotal - discount) * 0.08;
    } else if (region == "TX") {
        tax = (subtotal - discount) * 0.0625;
    } else {
        tax = (subtotal - discount) * 0.07;  // Default tax
    }
    
    // Final total
    out_total = subtotal - discount + tax;
    out_discount_percent = static_cast<int>((discount / subtotal) * 100);
}
```

**Metrics of this function:**

- **Lines of code**: 60
- **Cyclomatic complexity**: 6 (four if statements)
- **Cognitive complexity**: ~12 (nested conditions, multiple concerns)
- **Fan-out**: 1 (depends only on `db`)
- **Clarity**: Poor. You must hold in your head: the distinction between coupon discounts and bulk discounts, the special rules for bulk vs coupon, the order of operations (discount then tax, not tax then discount), the three coupon types.

**The problem**: This function violates the single-responsibility principle. It handles pricing, discounting, and tax calculation. It is tightly coupled to the database and to regional tax logic.

**The refactored version:**

```cpp
// Unit 1: Price calculation (no side effects, no dependencies)
double calculateSubtotal(const Order& order) {
    double sum = 0;
    for (const auto& item : order.items) {
        sum += item.price * item.quantity;
    }
    return sum;
}

// Unit 2: Discount calculation (pure function)
struct DiscountBreakdown {
    double discount_amount;
    int discount_percent;
};

DiscountBreakdown calculateDiscount(double subtotal, bool is_bulk,
                                   const ICouponProvider& coupon_provider,
                                   const std::string& coupon_code) {
    double discount = 0;
    
    // Coupon discount (not applied to bulk orders)
    if (!coupon_code.empty() && !is_bulk) {
        discount = applyCoupon(subtotal, coupon_code, coupon_provider);
    }
    
    // Bulk discount (only if subtotal > 500)
    if (is_bulk && subtotal > 500) {
        discount += subtotal * 0.1;
    }
    
    int percent = static_cast<int>((discount / subtotal) * 100);
    return {discount, percent};
}

// Unit 3: Tax calculation (pure function)
double calculateTax(double subtotal, double discount,
                   const ITaxCalculator& tax_calc,
                   const std::string& region) {
    return tax_calc.getTax(subtotal - discount, region);
}

// Unit 4: Orchestration (the main function, now simple)
void processOrder(const Order& order, const std::string& coupon_code,
                 bool is_bulk_purchase, const std::string& region,
                 ICouponProvider& coupon_provider, ITaxCalculator& tax_calc,
                 double& out_total, int& out_discount_percent) {
    double subtotal = calculateSubtotal(order);
    auto discount = calculateDiscount(subtotal, is_bulk_purchase,
                                     coupon_provider, coupon_code);
    double tax = calculateTax(subtotal, discount.discount_amount,
                            tax_calc, region);
    
    out_total = subtotal - discount.discount_amount + tax;
    out_discount_percent = discount.discount_percent;
}

// Unit 5: Coupon logic (extracted)
double applyCoupon(double subtotal, const std::string& code,
                  const ICouponProvider& provider) {
    auto coupon = provider.getCoupon(code);
    if (coupon.type == "percentage") {
        return subtotal * coupon.value / 100;
    } else if (coupon.type == "fixed") {
        return coupon.value;
    } else if (coupon.type == "bogo") {
        // ... BOGO logic in its own function
    }
    return 0;
}
```

**Metrics of the refactored version:**

- **Main function (processOrder)**: 8 lines, cyclomatic complexity = 1. It reads like a checklist: calculate subtotal, calculate discount, calculate tax, combine. Each step is a black box you can understand separately.
- **Helper functions**: Each does one thing. `calculateSubtotal` is pure. `calculateDiscount` is pure (dependencies are explicit). `calculateTax` is pure.
- **Cognitive complexity of each function**: 1–3 (very low).
- **Fan-out of processOrder**: Now 2 (depends on `ICouponProvider` and `ITaxCalculator`), explicit and injected.
- **Clarity**: Excellent. Each function can be understood in isolation. The flow is clear: input -> subtotal -> discount -> tax -> total.

**What happened to the complexity?** It did not disappear. It redistributed. The tax logic is still the same; it just moved to `calculateTax`. The coupon logic is still the same; it is now in `applyCoupon`. But now each piece is understandable independently, and the overall orchestration is trivial.

---

## 96.7 When Complexity Is Essential

Not all complexity can be eliminated. Some domains are inherently complex. If you are building a compiler, a database query optimizer, or a distributed consensus algorithm, the complexity is in the problem, not the code.

For these domains:

**Do not pretend the complexity does not exist.** Acknowledge it. Document it. Build tools to manage it.

**Separate essential complexity from accidental complexity ruthlessly.** A compiler's essential complexity (parsing, type checking, code generation) is unavoidable. Its accidental complexity (five different intermediate representations when one would suffice) is not.

**Build testable abstractions around the complexity.** A database query optimizer is complex, but you can test it by comparing its output (an execution plan) against known correct plans. You do not have to test the entire system end-to-end.

**Invest in documentation.** For essential complexity, documentation is not optional. New engineers must understand *why* the complexity is there, not just *what* it does.

**Use visualization.** For domains like distributed systems or concurrent algorithms, diagrams and state charts can make complexity visible and manageable.

---

## 96.8 Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|------------|
| **Simplify through extraction** | Low risk, improves clarity, does not lose functionality | Takes time, may expose design flaws | When a function exceeds 50 lines or cyclomatic complexity > 10 |
| **Invert dependencies** | Decouples modules, makes testing easier | Requires more abstraction layers, slightly more code | For modules with high fan-out or that are hard to test |
| **Separate pure from impure** | Pure functions are easier to test and reason about | Requires more function signatures, more composition | For functions that do computation + side effects |
| **Document invariants** | Prevents subtle bugs, makes intent clear | Requires discipline, can become outdated | For any module with non-trivial state |
| **Accept the complexity** | Avoids over-engineering, ships faster | Risk of poor maintainability later | For short-lived code or prototypes where complexity is essential |

---

## 96.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "More code means more complexity." | No. A 300-line function is more complex than 300 lines distributed across 10 functions. Complexity is about cognitive load and coupling, not line count. |
| "We should minimize the number of functions." | No. You should minimize the complexity of each function. More smaller functions are usually better than fewer larger ones. |
| "Complex code is just hard; there is no fix." | No. Complexity is usually a symptom of poor structure. Refactoring to separate concerns reduces it measurably. |
| "You should design for maximum flexibility upfront." | No. Over-abstraction adds complexity. Start simple; refactor when you see real coupling. |
| "Pure functions are impractical; real software must have side effects." | True, software must have side effects. But that does not mean every function must have them. Separate pure logic from side-effect-performing logic. |
| "If it works, do not touch it." | Dangerous. Code that works but is poorly structured accumulates bugs as it is modified. "It works now" does not mean "it will work tomorrow." |

---

## 96.10 Exercises

1. **Measure a function in your codebase.** Pick a function you find hard to understand. Calculate its cyclomatic complexity. Estimate its cognitive complexity. Count its fan-out. For each metric, identify what is driving the complexity. Is it the problem domain (essential) or the code structure (accidental)?

2. **Trace dependencies in a system.** Draw the dependency graph of a small service or module you know (who imports whom, or which tables are queried together). Count the number of edges. Identify one edge that creates tight coupling. Design a refactoring to break that coupling. Estimate the cost.

3. **Refactor for clarity.** Take a function with cyclomatic complexity > 5. Refactor it to reduce complexity by at least 50% (aim for complexity < 3 per function). Do not change behavior. Measure the improvement.

4. **Extract invariants.** Pick a class with mutable state. List all invariants that must hold (even the ones that are implicit in the code). Document them as comments. Identify one invariant that is frequently violated (a bug waiting to happen). Add an assertion to catch violations.

5. **Separate pure from impure.** Take a function that mixes computation and side effects. Refactor it into a pure computation function and a separate function that handles side effects. Verify the pure function is testable in isolation.

6. **Identify essential vs accidental complexity.** Pick a complex module in your system. List ten things that make it complex. For each, ask: is this complexity in the problem domain (essential) or in the code structure (accidental)? Can any accidental complexity be removed without changing behavior?

---

## 96.11 Summary

Complexity is not size. It is cognitive load (how much you must hold in your head) and coupling depth (how far your changes ripple). These are measurable. Cyclomatic complexity, cognitive complexity, and coupling metrics are useful smoke detectors.

Strive to be *complicated* (many parts, each understandable) rather than *complex* (parts that interact unpredictably). Make implicit assumptions explicit. Reduce non-obvious interactions. Prefer pure functions. Push side effects to edges. Document invariants.

When you encounter complexity, use the reasoning toolkit: identify units of reasoning, invariants, dependencies, and state. Ask whether state could be passed in instead of being global. If you cannot explain something in 10 minutes, it is too big or too coupled.

Complex systems are sometimes unavoidable (compilers, databases, distributed systems). When they are, separate essential complexity from accidental complexity ruthlessly, build testable abstractions, and invest in documentation.

The choice at every refactoring is the same: spend time now simplifying the structure, or spend more time later debugging and changing it. The math almost always favors now.

---

**[← Previous: Understanding Any Codebase Quickly](05-understanding-any-codebase.md)** · **[↑ Part 9](README.md)** · **[Next: How Senior Engineers Think →](07-how-senior-engineers-think.md)**
