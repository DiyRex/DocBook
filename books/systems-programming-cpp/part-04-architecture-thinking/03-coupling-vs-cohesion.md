# Chapter 35 — Coupling vs Cohesion

Every architecture textbook mentions these two words before concluding that "the solution" is "low coupling and high cohesion." That is technically correct and nearly useless. This chapter teaches you what they mean *precisely*, why they matter *in practice*, and how to recognize and measure them in C++ code.

**Coupling** measures how much two modules depend on each other. **Cohesion** measures how much the parts of a single module belong together. Good systems minimize unnecessary coupling while maximizing cohesion. Bad systems do the reverse — modules are tangled into each other, but each module's internal parts pull in different directions.

The key insight: coupling and cohesion predict *change cost*. Every feature request or bug fix is a change. A change in a tightly coupled system forces edits in many places. A change to a low-cohesion module means touching many unrelated pieces. Together, these metrics tell you how expensive the system is to maintain.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish between six types of coupling, from worst to best, and identify each in real C++ code.
2. Recognize seven levels of cohesion and diagnose when a module has low cohesion from its symptoms.
3. Estimate the change cost of a new feature by reasoning about coupling and cohesion.
4. Apply specific C++ techniques — forward declarations, the Pimpl idiom, dependency injection — to reduce unnecessary coupling.
5. Recognize anti-patterns: classes named `Manager`, `Helper`, `Util`, `Service` that often signal low cohesion.
6. Trade off coupling against other concerns: performance, compile time, testability, flexibility.

---

## The Six Levels of Coupling

Coupling describes how much one module depends on another. The scale ranges from "independent" to "intertwined." Larry Constantine's taxonomy, still the most useful, lists six types. We go from worst to best.

### Content Coupling (Worst)

One module directly modifies the internal state of another.

```cpp
// logger.h
class Logger {
public:
    std::string buffer;  // Public! Any caller can modify it.
};

// main.cpp
int main() {
    Logger log;
    log.buffer = "";  // Caller reaches inside and modifies internal state.
    log.buffer.append("debug: starting");
    // ...
}
```

Content coupling is rare in mature C++ (because most code uses `private:`), but it is insidious in legacy systems where `public` data is common. The problem: the caller is now responsible for invariants of the Logger. If the logger needs to track how many characters were added, the caller is violating that invariant by assigning. If the logger needed to validate the content, it cannot.

When you see `public` data, you are seeing content coupling. It is almost always a mistake.

### Common Coupling

Two modules both depend on a global or shared mutable state.

```cpp
// config.h
namespace config {
    int LOG_LEVEL = 0;  // Global mutable state.
}

// logger.cpp
void log(const std::string& msg) {
    if (config::LOG_LEVEL >= 1) {  // Common coupling to global.
        std::cerr << msg << "\n";
    }
}

// auth.cpp
void authenticate(const std::string& user) {
    config::LOG_LEVEL = 2;  // Module A modifies shared state...
    // ...
    log("auth attempt");  // ...which Module B uses. Coupling!
}
```

Both modules are coupled to the shared state. But here is the trap: they are also coupled to each other *indirectly*. If `auth` sets `LOG_LEVEL` to 2, then calls `log`, then `log` behaves differently. If some other module sets it to 0, `log` behaves differently. Debugging becomes detective work: which module modified the global? Who depends on what value? The dependencies are invisible.

Globals are sometimes unavoidable (stdin, stdout, errno are globals). Most of the time, they are a shortcut that costs more than it saves.

### External Coupling

Two modules depend on the same external data format or protocol, and cannot change independently.

```cpp
// user_service.cpp
struct User {
    int id;
    std::string name;
    uint32_t created_at;  // Unix timestamp. Hardcoded assumption.
};

void User::serialize(std::ostream& out) {
    out << id << "|" << name << "|" << created_at << "\n";
}

// analytics_service.cpp
struct UserEvent {
    int user_id;
    std::string name;
    uint32_t timestamp;  // Same assumption about timestamp format.
};

void parseUserEvent(std::istream& in, UserEvent& e) {
    in >> e.user_id >> e.name >> e.timestamp;
    // Assumes the same pipe-delimited format as User.
}
```

Here, `UserService` and `AnalyticsService` are coupled through the serialized format. They both assume "timestamps are Unix seconds." They both assume "fields are pipe-delimited." If one service decides to use milliseconds instead, the other breaks.

External coupling is common in distributed systems: two services talk via JSON, both assume a field is a string when it's really a number, and integration breaks. The fix: a shared schema (a `.proto` file, a JSON-Schema, an OpenAPI spec) that both services depend on, rather than both services having to guess.

### Control Coupling

One module passes control flow information to another, forcing the receiver to behave differently.

```cpp
// payment_processor.cpp
enum PaymentMode { DEBIT = 1, CREDIT = 2, DIRECT = 3 };

class PaymentService {
public:
    bool process(const Order& order, PaymentMode mode) {
        switch (mode) {
            case DEBIT:
                return processDebit(order);
            case CREDIT:
                return processCredit(order);
            case DIRECT:
                return processACH(order);
            default:
                return false;
        }
    }
};

// checkout.cpp
void checkout(const Order& order, bool useCredit) {
    PaymentService svc;
    bool ok = svc.process(order, useCredit ? CREDIT : DEBIT);
    // Caller is passing control flow flags to the payment service.
    // If a new payment mode is added, all callers must know about it.
}
```

The caller is tightly coupled to the payment service's control-flow options. Adding a new mode `CRYPTO` forces changes in every caller. The service is also tightly coupled to the caller's decision logic; it cannot change how modes are dispatched without affecting callers.

The fix: use polymorphism. Each payment mode is its own class, and the service takes a `PaymentMethod*` instead of an enum. New modes don't require changes to existing callers.

### Stamp Coupling

One module passes a complex data structure to another, even though the receiver only uses a subset of its fields.

```cpp
// order.h
struct Order {
    int id;
    std::string customer_name;
    std::vector<LineItem> items;
    Address shipping_address;
    Address billing_address;
    CreditCard payment_method;
    int warehouse_id;
    ShippingService shipping_svc;
};

// inventory.cpp
bool fulfillOrder(const Order& order) {
    // Only needs id and items.
    // But to call this function, I must know and construct
    // the full Order type, including payment_method, billing address, etc.
    // If Order gains a new field (a discount code), this function's
    // signature doesn't change, but callers must handle the new field.
    return warehouse.reserve(order.id, order.items);
}

// shipping.cpp
void shipOrder(const Order& order) {
    // Only needs id and shipping_address.
    // But to call this, I must construct the full Order with all fields.
    carrier.send(order.shipping_address, order.id);
}
```

The modules are coupled to the entire `Order` structure, not just the fields they use. If `Order` adds a new field, every function that takes `Order` must know about it, even if the function never touches it. Worse, in many languages (Java, Python), adding a field to `Order` requires a migration — all existing serialized instances are now invalid. The coupling is deep.

The fix: pass only what you need. `fulfillOrder` takes `(int orderId, const std::vector<LineItem>&)`. `shipOrder` takes `(int orderId, const Address&)`. Now each function is coupled only to what it uses. Adding fields to `Order` doesn't force changes to these signatures.

### Data Coupling (Best)

One module calls another and passes only the minimal data needed — no shared structures, no control flow information, no global state.

```cpp
// calculator.h
class Calculator {
public:
    int add(int a, int b) const;
    int multiply(int a, int b) const;
};

// main.cpp
int main() {
    Calculator calc;
    int result = calc.add(2, 3);
    // Coupling: Calculator exists, and I must call its methods.
    // That's it. Nothing else.
    // If Calculator adds new methods, add() is unchanged.
    // If add() changes its implementation, my code doesn't care.
}
```

This is nearly the weakest coupling possible in a connected system. `main` depends on `Calculator` existing and having an `add` method. That is the contract. Everything else is hidden.

To measure the six types, ask: **what assumptions about the other module must I make to call this function?** Content coupling assumes I know the module's internal structure. Common coupling assumes I know about shared global state. External coupling assumes I know the data format. Control coupling assumes I know the control-flow modes. Stamp coupling assumes I know all fields of a structure. Data coupling assumes only the types and semantics of the parameters.

---

## The Seven Levels of Cohesion

Cohesion describes how much the pieces of a single module belong together. A module with high cohesion has parts that are tightly related; a change to one feature affects only that module. Low cohesion means the parts are weakly related; a change might require touching many methods that have nothing in common.

### Coincidental Cohesion (Worst)

Methods in the same class happen to be grouped together, but they have no logical relationship.

```cpp
// Util.cpp
class Util {
public:
    static void logMessage(const std::string& msg) {
        std::cerr << msg << "\n";
    }
    
    static int computeTaxes(double income) {
        return income * 0.25;
    }
    
    static std::string formatDate(time_t t) {
        struct tm* info = localtime(&t);
        char buf[256];
        strftime(buf, sizeof(buf), "%Y-%m-%d", info);
        return buf;
    }
    
    static std::vector<int> quickSort(std::vector<int> nums) {
        // ...
    }
};
```

This is a classic `Util` or `Helper` class. The methods have no relationship: logging, taxes, dates, and sorting are four completely separate concepts. The class exists only as a garbage pail for "functions that don't fit elsewhere." Cohesion is nil.

When you encounter a class like this in a review, the fix is simple: delete it and move each method to the module that actually uses it. If a method is truly generic (quicksort), put it in a library designed for that purpose. If a method is used by three different features, create three separate modules, each with that method inlined or called from a shared utility.

### Logical Cohesion

Methods are related by category, but they don't actually work together.

```cpp
// UserOperations.cpp
class UserOperations {
public:
    void createUser(const UserData& data) { /* ... */ }
    void deleteUser(int id) { /* ... */ }
    void updateUser(int id, const UserData& data) { /* ... */ }
    void queryUser(int id) { /* ... */ }
    void exportUsersToCSV(const std::string& filename) { /* ... */ }
    void importUsersFromCSV(const std::string& filename) { /* ... */ }
};
```

All methods are "user-related," so they're grouped. But they are not truly cohesive. `createUser` and `deleteUser` work on a database. `exportUsersToCSV` works on a file format. `queryUser` works on a query engine. They're related by category, not by actual dependency.

The problem: when you call `createUser`, you don't call `updateUser`. When you fix a bug in `queryUser`, it doesn't affect `exportUsersToCSV`. They could be in the same class, or four different classes, and the code would work the same way. The class is organized by "what problem does it solve" rather than "what does it do."

The fix: split by actual usage patterns. A `UserRepository` handles create/delete/update/query. A `UserExporter` handles CSV export/import.

### Temporal Cohesion

Methods are called at the same time in the program's lifecycle, but they are otherwise unrelated.

```cpp
// Initializer.cpp
class Initializer {
public:
    void setupDatabaseConnection() { /* ... */ }
    void loadConfigurationFile() { /* ... */ }
    void initializeLogging() { /* ... */ }
    void warmCaches() { /* ... */ }
};

// main.cpp
int main() {
    Initializer init;
    init.setupDatabaseConnection();
    init.loadConfigurationFile();
    init.initializeLogging();
    init.warmCaches();
    // All called once, at startup. Nothing else calls them.
}
```

These methods happen at the same time (startup), but they have no real relationship. Database setup doesn't know about logging. Logging doesn't know about caches. They are in the same class only because they're all called together.

Temporal cohesion is not always wrong — sometimes it's the right level of granularity. But if the startup sequence gets complex, or if you need to re-initialize only part of the system, you'll find this design is inflexible. The fix: split by subsystem. Each subsystem owns its own initialization.

### Procedural Cohesion

Methods work together to produce a result, but they share no data.

```cpp
// EmailService.cpp
class EmailService {
private:
    std::string loadTemplate(const std::string& name) { /* ... */ }
    std::string renderTemplate(const std::string& tpl, const Data& d) { /* ... */ }
    bool validateEmail(const std::string& addr) { /* ... */ }
    void sendEmail(const std::string& to, const std::string& body) { /* ... */ }
    
public:
    void sendWelcomeEmail(const std::string& to, const UserData& user) {
        auto tpl = loadTemplate("welcome");
        auto body = renderTemplate(tpl, user);
        if (!validateEmail(to)) return;
        sendEmail(to, body);
    }
};
```

The methods work together in a sequence — load, render, validate, send — to accomplish a goal. But they don't share state. Each step is independent; you could call them from other places without the class falling apart.

Procedural cohesion is better than the lower levels. It has a *purpose* — to send emails. But it's not ideal. If you need to send a different kind of email (with a different validation rule or a different template engine), you end up touching multiple methods. The class is organized by steps in a process, not by data flow.

The fix depends on what changes together. If the rendering logic is stable, but the validation rule changes, split validation into its own module. If the template engine might change, encapsulate that in a separate class.

### Communicational Cohesion

Methods work on the same data, and that data is their only connection.

```cpp
// Order.cpp
class Order {
private:
    std::vector<LineItem> items;
    double subtotal;
    
public:
    void addItem(const LineItem& item) {
        items.push_back(item);
        recalculateSubtotal();
    }
    
    void removeItem(int index) {
        items.erase(items.begin() + index);
        recalculateSubtotal();
    }
    
    void recalculateSubtotal() {
        subtotal = 0;
        for (const auto& item : items) {
            subtotal += item.price * item.quantity;
        }
    }
    
    double getSubtotal() const { return subtotal; }
    
    void applyDiscount(double percent) {
        subtotal = subtotal * (1.0 - percent / 100.0);
    }
    
    // Unrelated method that just happens to work on Order's data.
    void serializeToJSON(std::ostream& out) {
        out << "{\"items\": [";
        // ...
        out << "], \"subtotal\": " << subtotal << "}";
    }
};
```

All methods touch `items` or `subtotal`. That's their only connection. They have nothing else in common. Some methods modify state (addItem), some read it (getSubtotal), some transform it (applyDiscount), some output it (serializeToJSON).

Communicational cohesion is moderate. The class is a data holder; the methods are a collection of operations on that data. This is common in domain models — a `User` class that holds user data and has methods like `getName`, `setEmail`, `toJSON`, etc. It often works, but it can become unwieldy when the number of operations grows.

### Sequential Cohesion

Methods are cohesive because the output of one is the input to another.

```cpp
// Pipeline.cpp
class MailPipeline {
private:
    std::vector<std::string> parseMailboxFile(const std::string& path);
    
    struct Mail {
        std::string from;
        std::string subject;
        std::string body;
    };
    
    std::vector<Mail> extractMailHeaders(const std::vector<std::string>& lines);
    std::vector<Mail> filterSpam(const std::vector<Mail>& mails);
    std::vector<Mail> sortByDate(const std::vector<Mail>& mails);
    
public:
    std::vector<Mail> processMail(const std::string& path) {
        auto lines = parseMailboxFile(path);
        auto mails = extractMailHeaders(lines);
        auto filtered = filterSpam(mails);
        auto sorted = sortByDate(filtered);
        return sorted;
    }
};
```

Each method produces output that becomes input to the next. There is a clear data flow: lines → Mail → Mail (filtered) → Mail (sorted). The methods are cohesive because they form a pipeline. If you need to change the pipeline — insert deduplication, remove date sorting — the change is localized to this class.

Sequential cohesion is strong. It matches the problem domain (processing an email mailbox has well-defined steps). Tests are easy to write (you can test each step independently). But be careful: if the pipeline is specific to one use case, you might end up with many similar classes. If the steps could be reused in other pipelines, consider extracting them into separate functions and composing a new pipeline.

### Functional Cohesion (Best)

All parts of the module work together to accomplish a single, well-defined goal. No part can be removed without breaking the whole.

```cpp
// Stack.h
template <typename T>
class Stack {
private:
    std::vector<T> data;
    
public:
    void push(T value) {
        data.push_back(value);
    }
    
    T pop() {
        if (empty()) throw std::underflow_error("Stack is empty");
        T result = data.back();
        data.pop_back();
        return result;
    }
    
    T top() const {
        if (empty()) throw std::underflow_error("Stack is empty");
        return data.back();
    }
    
    bool empty() const {
        return data.empty();
    }
    
    size_t size() const {
        return data.size();
    }
};
```

Every method is essential to the stack abstraction. You cannot remove `push` without breaking the contract. You cannot add an unrelated method like `serializeToJSON` without reducing cohesion. The class has a single reason to exist: to provide a stack data structure.

Functional cohesion is the ideal. Every part of the module contributes to one goal. Tests pass or fail based on one feature. Changes to one feature don't ripple into others.

---

## Why Both Matter

Here's the concrete impact on your work: **change cost is determined by coupling times cohesion.**

Suppose you get a feature request: "Users should be able to export their data as CSV."

In a system with **high coupling and low cohesion**:
- The `User` class has a method `exportAsCSV`. 
- But `User` is also tightly coupled to `Database`, which is coupled to `Authentication`, which is coupled to `Logger`.
- `exportAsCSV` depends on `User` having the right fields, but `User` also handles login, permissions, and email notifications.
- To add CSV export, you must touch `User`, `Database`, and the logging system.
- One feature request becomes three files changed, two deploys, two meetings, one regression.

In a system with **low coupling and high cohesion**:
- There is a `UserExporter` class whose only job is exporting user data.
- It depends on a `User` interface that provides only the data it needs.
- Adding a new export format (JSON, Parquet) means adding a new exporter class; nothing else changes.
- One feature, one class, one deploy.

The difference is not performance. It is *engineering velocity*. In the second system, you can move fast because changes are local. In the first, you move slowly because every change is global.

---

## How They Manifest in C++ Specifically

C++ has a unique angle on coupling and cohesion: the compilation model. Unlike Python or Java, C++ requires you to declare what you depend on in header files. This creates several C++-specific patterns.

### Coupling Through Includes

The most common form of coupling in C++ is a header dependency.

```cpp
// user.h
#include "database.h"  // Includes the entire Database implementation.
#include "logger.h"    // Includes the Logger implementation.
#include "crypto.h"    // Includes cryptographic utilities.

class User {
private:
    Database& db;
    Logger& logger;
    // ...
};
```

Every file that includes `user.h` now indirectly includes `database.h`, `logger.h`, and `crypto.h`. If `database.h` includes 10 other files, and `logger.h` includes 5, you've now got 16 files in the compilation graph. Changing any of them triggers a full rebuild of any code that uses `User`.

This is **compile-time coupling**, distinct from the runtime coupling of the previous section. You can have low runtime coupling but high compile-time coupling, and it will cost you in build times.

The fix: forward declarations.

```cpp
// user.h
class Database;  // Forward declaration: I promise this class exists, but I don't need its definition yet.
class Logger;

class User {
private:
    Database* db;      // I only need a pointer.
    Logger* logger;    // I don't need to see the full definition.
    // ...
};
```

Now `user.h` doesn't include any other headers. `user.cpp` can include them:

```cpp
// user.cpp
#include "user.h"
#include "database.h"  // Only included here, not in the header.
#include "logger.h"

User::User(Database& db, Logger& logger) : db(&db), logger(&logger) {}
```

The rule: **in headers, forward-declare; in implementation files, include.** This single rule cuts compile-time coupling drastically.

### The Pimpl Idiom

Sometimes forward declarations are not enough. Imagine `User` has a complex private member:

```cpp
// user.h
#include "complex_library.h"  // 50 headers transitively.

class User {
private:
    ComplexLibrary::ComplexType internal_state;
};
```

Every includer of `user.h` now includes all 50 headers. The fix is the Pimpl (Pointer to IMPLmentation) idiom:

```cpp
// user.h
class User {
private:
    class Impl;
    std::unique_ptr<Impl> pimpl;
};

// user.cpp
#include "complex_library.h"  // Included only here.

class User::Impl {
    ComplexLibrary::ComplexType internal_state;
};

User::User() : pimpl(std::make_unique<Impl>()) {}
User::~User() = default;  // Required for Impl forward declaration.
```

Now `user.h` doesn't include `complex_library.h`. The header is tiny; only `user.cpp` pays the cost. This is the technique used in Qt, Boost, and large C++ libraries to minimize header bloat.

The tradeoff: an extra indirection (pointer dereference) at runtime, and a bit more dynamic allocation. But for a library header that millions of lines of code include, the compile-time savings are worth it.

### Templates and Coupling

Templates introduce a different kind of coupling: definition coupling.

```cpp
// util.h
template <typename T>
T maximum(T a, T b) {
    return a > b ? a : b;
}

// main.cpp
#include "util.h"  // I need the full definition.

int main() {
    auto x = maximum(3, 5);  // OK: int is std::ordered, operator> exists.
    std::string s1 = "hello", s2 = "world";
    auto s = maximum(s1, s2);  // OK: string also has operator>.
}
```

Template definitions must be visible at every instantiation site. This is fine for small functions. But if `util.h` is 5,000 lines of template code, every includer pays the cost of parsing all of it. This is part of why C++ templates can slow compilation.

The Concept (C++20) helps here:

```cpp
// util.h
#include <concepts>

template <typename T>
requires std::totally_ordered<T>
T maximum(T a, T b) {
    return a > b ? a : b;
}
```

With concepts, the template's requirements are explicit. A type checker could in theory validate whether a type satisfies the concept without instantiating the template. (Current C++ tools don't do this yet, but the idea is there.)

### Dependency Injection

The cleanest way to reduce coupling in C++ is dependency injection: instead of a class creating or finding its dependencies, they are passed in.

```cpp
// Bad: high coupling.
class UserService {
    Database db{};  // Creates its own database.
    Logger log{};   // Creates its own logger.
    
public:
    void createUser(const std::string& name) {
        db.insert("users", {{"name", name}});
        log.info("User created: " + name);
    }
};

// Good: dependencies are injected.
class UserService {
    Database& db;
    Logger& log;
    
public:
    UserService(Database& db, Logger& log) : db(db), log(log) {}
    
    void createUser(const std::string& name) {
        db.insert("users", {{"name", name}});
        log.info("User created: " + name);
    }
};
```

In the bad version, `UserService` is tightly coupled to the specific implementations of `Database` and `Logger`. To test it, you must test the real database and logger. To swap loggers, you must rewrite `UserService`.

In the good version, `UserService` depends only on the interfaces. To test it, you inject a mock database and logger. To swap the real logger for a different one, you inject the new one at construction time. The coupling is to abstractions, not concretions.

---

## Worked Example: Refactoring a Low-Cohesion Mess

Suppose you inherit a `BillingService` class that handles four unrelated jobs. It is a real example from real code (names changed):

```cpp
// billing_service.h
class BillingService {
private:
    Database& db;
    Logger& log;
    EmailSender& email;
    
public:
    BillingService(Database& db, Logger& log, EmailSender& email) 
        : db(db), log(log), email(email) {}
    
    // Job 1: Process payments.
    bool processPayment(int user_id, double amount) {
        auto user = db.query("SELECT * FROM users WHERE id = ?", user_id);
        if (user.credit_limit < amount) {
            log.error("Insufficient credit for user " + std::to_string(user_id));
            return false;
        }
        db.execute("UPDATE users SET balance = balance - ? WHERE id = ?", amount, user_id);
        log.info("Payment processed: " + std::to_string(amount));
        return true;
    }
    
    // Job 2: Generate invoices.
    std::string generateInvoice(int order_id) {
        auto order = db.query("SELECT * FROM orders WHERE id = ?", order_id);
        std::string invoice = "INVOICE\n";
        invoice += "Order: " + std::to_string(order.id) + "\n";
        for (const auto& item : order.items) {
            invoice += item.name + " x " + std::to_string(item.qty) + " @ " + std::to_string(item.price) + "\n";
        }
        log.info("Invoice generated for order " + std::to_string(order_id));
        return invoice;
    }
    
    // Job 3: Send payment reminders.
    void sendReminder(int user_id) {
        auto user = db.query("SELECT * FROM users WHERE id = ?", user_id);
        if (user.balance > 0) {
            email.send(user.email, "Reminder", "Your balance is " + std::to_string(user.balance));
            log.info("Reminder sent to user " + std::to_string(user_id));
        }
    }
    
    // Job 4: Generate tax reports (completely different).
    std::string generateTaxReport(int year) {
        auto transactions = db.query("SELECT * FROM transactions WHERE year = ?", year);
        double total = 0;
        for (const auto& txn : transactions) {
            total += txn.amount;
        }
        log.info("Tax report generated for year " + std::to_string(year));
        return "Tax Report: " + std::to_string(total);
    }
};
```

This is **logical cohesion at best**. All methods touch billing-related tables, but they are four separate concerns that happen to be grouped. The problems:

1. Testing is slow: to test `generateInvoice`, you must set up payment reminders, payment processing, and tax reporting infrastructure.
2. Reusability is low: if you need to call `generateInvoice` from the admin UI but not from the API, you still have the whole class as a dependency.
3. Changes ripple: if you need to add multi-language support to reminders, you end up modifying the same class that handles payments.

Let's refactor into functionally cohesive classes:

```cpp
// payment_processor.h
class PaymentProcessor {
private:
    Database& db;
    Logger& log;
    
public:
    PaymentProcessor(Database& db, Logger& log) : db(db), log(log) {}
    
    bool process(int user_id, double amount) {
        auto user = db.query("SELECT * FROM users WHERE id = ?", user_id);
        if (user.credit_limit < amount) {
            log.error("Insufficient credit for user " + std::to_string(user_id));
            return false;
        }
        db.execute("UPDATE users SET balance = balance - ? WHERE id = ?", amount, user_id);
        log.info("Payment processed: " + std::to_string(amount));
        return true;
    }
};

// invoice_generator.h
class InvoiceGenerator {
private:
    Database& db;
    Logger& log;
    
public:
    InvoiceGenerator(Database& db, Logger& log) : db(db), log(log) {}
    
    std::string generate(int order_id) {
        auto order = db.query("SELECT * FROM orders WHERE id = ?", order_id);
        std::string invoice = "INVOICE\n";
        invoice += "Order: " + std::to_string(order.id) + "\n";
        for (const auto& item : order.items) {
            invoice += item.name + " x " + std::to_string(item.qty) + " @ " + std::to_string(item.price) + "\n";
        }
        log.info("Invoice generated for order " + std::to_string(order_id));
        return invoice;
    }
};

// payment_reminder.h
class PaymentReminder {
private:
    Database& db;
    Logger& log;
    EmailSender& email;
    
public:
    PaymentReminder(Database& db, Logger& log, EmailSender& email) 
        : db(db), log(log), email(email) {}
    
    void send(int user_id) {
        auto user = db.query("SELECT * FROM users WHERE id = ?", user_id);
        if (user.balance > 0) {
            email.send(user.email, "Reminder", "Your balance is " + std::to_string(user.balance));
            log.info("Reminder sent to user " + std::to_string(user_id));
        }
    }
};

// tax_reporter.h
class TaxReporter {
private:
    Database& db;
    Logger& log;
    
public:
    TaxReporter(Database& db, Logger& log) : db(db), log(log) {}
    
    std::string generate(int year) {
        auto transactions = db.query("SELECT * FROM transactions WHERE year = ?", year);
        double total = 0;
        for (const auto& txn : transactions) {
            total += txn.amount;
        }
        log.info("Tax report generated for year " + std::to_string(year));
        return "Tax Report: " + std::to_string(total);
    }
};
```

Now each class has **functional cohesion**. Each does one thing. The benefits:

1. **Testing is fast**: to test `InvoiceGenerator`, you only set up `Database` and `Logger` mocks. You don't instantiate payment or reminder logic.
2. **Reusability is high**: the admin UI can use `InvoiceGenerator` without importing the payment or reminder classes.
3. **Changes are local**: adding multi-language support to reminders touches only `PaymentReminder`.
4. **Coupling is reduced**: if you add a new reminder channel (SMS), `PaymentProcessor` doesn't care.

The single risk: more classes means more files to navigate. But in a well-organized codebase, more small focused classes beat fewer large classes. The tradeoff is almost always worth it.

---

## Diagnosing Low Coupling and Low Cohesion

You don't need metrics. You need symptoms.

### Symptoms of High Coupling

- **Shotgun surgery**: A single feature request requires changes to five files. A single bug fix touches code in three different modules that seem unrelated.
- **Circular dependencies**: Module A depends on Module B, which depends on Module C, which depends back on Module A. Refactoring becomes a puzzle.
- **Test tangles**: To test one class, you must instantiate five others. Mocks and stubs explode. Tests become slow and brittle.
- **Slow compiles**: C++ specifically: a change to one header forces recompilation of thousands of translation units.
- **Difficult refactors**: Extracting a method or moving a class becomes risky because so many call sites depend on it.

### Symptoms of Low Cohesion

- **Class names**: `Manager`, `Handler`, `Service`, `Helper`, `Util`. These are almost always garbage cans for unrelated code. (Not always — a `DatabaseService` is fine if it truly manages databases. But `GeneralService`? Red flag.)
- **Methods that don't use the class's data**: A method that takes all its arguments as parameters and never touches `this`. It's not cohesive; it shouldn't be there.
- **Wide constructors**: A class that requires 10 injected dependencies. It's doing too much, and it probably isn't cohesive.
- **Giant classes**: A class with 50+ methods. You can't hold it all in your head. It's almost certainly low cohesion; it's doing too many unrelated things.
- **Weak test names**: A test class with tests for `testCreateUser`, `testFormatDate`, and `testParseXML` in the same file. The class under test is low cohesion.

---

## Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| High coupling, many dependencies injected | Flexible, testable, decoupled from implementations | Many classes, many constructor arguments, harder to understand data flow | When you need to swap implementations (e.g., in-memory vs. database backend) or write many tests. |
| Low coupling, few dependencies, hard-coded implementations | Simple, fast, easy to understand control flow | Not testable, not flexible, hard to swap components | Early stages of a project, tools, one-off scripts. |
| High cohesion, many small classes | Each class does one thing, easy to test, easy to change | More files to navigate, more abstractions, higher cognitive overhead when understanding data flow | Large codebases, teams, projects that will live for years. |
| Low cohesion, few large classes | Fewer abstractions, fewer files, clear data ownership | Hard to test, hard to change, changes ripple, low reusability | Not recommended for production code. Acceptable only in very small scripts. |
| Using templates to reduce coupling | Avoids dynamic dispatch, no virtual call overhead, type-safe | Compilation is slower, binary size can balloon, template errors are cryptic | Performance-critical code, libraries with strict ABI requirements. |
| Using virtual functions for decoupling | Easy to swap implementations, less coupling, clear interfaces | Virtual call overhead, less type safety (some dispatch happens at runtime), larger binary | Application code, frameworks, when flexibility matters more than 1-2% performance. |
| Pimpl to reduce compilation coupling | Headers stay small, changes to implementation don't trigger recompilation | Extra indirection at runtime, dynamic allocation, more boilerplate | Library headers, code used by many translation units, slow builds. |
| Forward declarations | Zero runtime cost, minimal coupling | Can't use the full type in headers, more verbose signatures | Always, whenever you can afford a pointer instead of a reference. |

---

## Common Misconceptions

**"Loose coupling means no coupling."** 
No. Good systems have *intentional* coupling at module boundaries. A `User` class is intentionally coupled to its own data, to its own invariants, and to the classes that use it. The goal is zero *unnecessary* coupling, not zero coupling. A totally decoupled system is usually a set of independent scripts that don't talk to each other — not useful.

**"High cohesion means every method shares every field."**
No. High cohesion means every method contributes to the same purpose. A `Stack` class has a `data` field and five methods: `push`, `pop`, `top`, `empty`, `size`. Only one method reads `data` directly; the rest use `empty()`. They're cohesive because they work toward one goal: providing a stack abstraction. Not because they all access the same fields.

**"Coupling and cohesion are independent."**
They interact. A high-cohesion class that is tightly coupled to five other classes is still a mess. A loosely coupled system of low-cohesion classes is still a mess. You want both: low unnecessary coupling and high cohesion within each module.

**"Coupling is always bad."**
No. Necessary coupling is fine. Your `main()` function is coupled to your `UserService` — that's the whole point. The problem is *unnecessary* coupling, coupling that could be removed without changing the behavior. A global logger that every module uses is unnecessary coupling; a logger passed to a `PaymentProcessor` is necessary coupling.

**"If we just use interfaces, we solve coupling."**
Interfaces reduce the *kind* of coupling (now it's to an abstraction, not a concrete type), but they don't eliminate it. A class that depends on 20 interfaces is still highly coupled; it just has 20 abstract dependencies instead of 20 concrete ones. Reducing the number of dependencies is still the goal.

---

## Exercises

1. **Identify the cohesion level.** Take a class you wrote recently. What is its primary purpose? Can you articulate it in one sentence? Now list every method. How many methods don't contribute to that purpose? At what level of cohesion is it?

2. **Find the coupling type.** In a codebase you know, find a place where one module depends on another. Determine which of the six coupling types it is. Write down: what would it take to swap out the dependency? Is that feasible?

3. **Test the coupling.** Pick a class that takes five injected dependencies. Write a unit test that tests one behavior of that class. How many of the five dependencies do you actually need to mock? If you need fewer than three, the class is probably too coupled or doing too much.

4. **Refactor low cohesion.** Find a class named `Manager`, `Handler`, `Service`, or `Util` in a real codebase. Identify every method. Group them by purpose. Could they be split into separate classes? Sketch the refactoring. (You don't have to code it; just prove it's possible.)

5. **Compile coupling experiment.** In a C++ project, change a header file that many other files depend on. Measure how long a clean build takes. Now refactor to use forward declarations and Pimpl. Remeasure. How much faster?

6. **Circular dependency detective.** Graph the dependencies in your codebase (use your IDE's dependency viewer, or create one with a script). Find a cycle. What would it take to break it? Is it worth breaking?

---

## Summary

Coupling measures how much modules depend on each other; cohesion measures how unified a single module is. Together, they predict change cost. The six types of coupling range from terrible (content coupling, where one module rewrites another's internal state) to good (data coupling, passing only what you need). The seven levels of cohesion range from garbage cans (coincidental, many unrelated methods) to focused and purposeful (functional, every part serves the goal).

C++ exposes coupling in the compilation model: includes are coupling, forward declarations reduce it, and the Pimpl idiom hides it. Diagnosing coupling and cohesion is mostly pattern matching — watch for class names like `Util`, wide constructors, methods that don't use their own data, and tests that require elaborate setup. When you see these patterns, you can usually refactor: split low-cohesion classes, reduce dependencies through dependency injection, and use interfaces to depend on abstractions rather than concretions.

The payoff is not elegance. It is **engineering velocity**. Low coupling and high cohesion make changes local, tests fast, and comprehension easy. That is worth the extra care in design.

---

> **[← Previous: Why Large Programs Become Complex](02-why-large-programs-become-complex.md)** · **[↑ Part 4](README.md)** · **[Next: Separation of Concerns →](04-separation-of-concerns.md)**
