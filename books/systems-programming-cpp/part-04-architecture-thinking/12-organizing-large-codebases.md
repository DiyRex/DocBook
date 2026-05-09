# Chapter 44 — Organizing Large Codebases

At 10,000 lines of code, organization is a convenience. At 50,000 lines, it is a load-bearing wall. At 500,000 lines — which is not unusual for long-lived applications — the folder structure and dependency rules are the difference between a system a team can ship and a system that ships the team: constant merge conflicts, hidden dependencies, accidental cycles, modules that cannot be modified without breaking seven others. Organization stops being aesthetic and starts being essential.

This chapter is about how to organize code at scale so engineers can find what they need, change what they must, and avoid stepping on each other. We cover folder layouts, public and internal APIs, when to split into multiple repositories, and the tooling that prevents organizational rules from becoming suggestions.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Compare layer-first and feature-first folder structures, recognize the tradeoffs, and choose one that fits your system's growth pattern.
2. Distinguish between a module's public API and its internal implementation using headers, package visibility, and barrel files.
3. Evaluate the monorepo-vs-polyrepo decision: when one repository with many projects scales better than many repositories with one project each.
4. Apply dependency direction rules — Dependency Inversion Principle — and use tooling to enforce them in CI.
5. Read build times as a signal of architectural coupling and apply techniques to reduce them.
6. Establish code ownership without creating fiefdoms: use CODEOWNERS files and BUILD file visibility to clarify who can change what.
7. Reorganize a codebase from one structure to another without losing history.

---

## Folder Layouts: Layer-First vs Feature-First

The first major organizational choice is whether to group code by *layer* or by *feature*. Neither is universally correct; both have failure modes when applied to the wrong system.

### Layer-First Organization

In layer-first, all code of the same kind goes in one folder tree:

```
src/
├── controllers/
│   ├── user_controller.cpp
│   ├── order_controller.cpp
│   ├── payment_controller.cpp
├── services/
│   ├── user_service.cpp
│   ├── order_service.cpp
│   ├── payment_service.cpp
├── repositories/
│   ├── user_repository.cpp
│   ├── order_repository.cpp
│   ├── payment_repository.cpp
├── models/
│   ├── user.cpp
│   ├── order.cpp
│   ├── payment.cpp
```

This mirrors the three-layer architecture: presentation at the top, domain in the middle, persistence at the bottom. It has clear logical organization and makes it easy to understand where a particular *kind* of code lives. If you are working on data access patterns, you go to `repositories/`. If you are fixing a bug in request handling, you go to `controllers/`.

The cost becomes visible at scale. As the codebase grows, `services/` becomes a folder with 200 files. A new developer opening the project does not know which service to look at for user-related logic, which for payments, which for reporting. The folder has become a list, not a hierarchy. The layer obscures the features.

Layer-first also creates a natural bottleneck: if five teams are working on independent features, all of them must frequently touch `services/` at the same time. Merge conflicts are inevitable. The folder structure has high affinity with the layer boundary, not the team boundary.

### Feature-First Organization

In feature-first, all code for one feature lives together:

```
src/
├── users/
│   ├── controllers/
│   │   ├── user_controller.cpp
│   ├── services/
│   │   ├── user_service.cpp
│   ├── repositories/
│   │   ├── user_repository.cpp
│   ├── models/
│   │   ├── user.cpp
├── orders/
│   ├── controllers/
│   ├── services/
│   ├── repositories/
│   ├── models/
├── payments/
│   ├── controllers/
│   ├── services/
│   ├── repositories/
│   ├── models/
```

Now a developer working on user registration touches only code in `users/`. A team owning payments touches only `payments/`. The folder structure aligns with team ownership and feature boundaries. This scaling is nearly unlimited — you can add 50 new features with 50 new folders, and they do not interfere with each other.

The cost: *local* coupling. Inside `users/`, the controller, service, and repository are tightly bound. They know about each other's existence. Dependencies are short and direct, which is good. But if `users/` and `payments/` both need to emit events, you have a choice: duplicate event emission code in both, or create a shared events module outside both. The second choice is often right, but it is not obvious, and teams frequently choose the first, creating hidden duplication.

### Hybrid: Vertical Slices with Shared Utilities

In practice, the best organizations use feature-first for the 80% of code that is feature-specific, and a `shared/` or `core/` folder for the 20% of code that multiple features use:

```
src/
├── shared/
│   ├── events/
│   ├── logging/
│   ├── database/
│   ├── validation/
├── users/
│   ├── controllers/
│   ├── services/
│   ├── repositories/
├── payments/
│   ├── controllers/
│   ├── services/
│   ├── repositories/
├── orders/
│   ├── controllers/
│   ├── services/
│   ├── repositories/
```

The `shared/` folder contains only the truly generic infrastructure: database connection management, logging utilities, event bus infrastructure, validation helpers. Each feature module is self-contained. Shared code is explicitly named and clearly separated, making the dependency visible.

The key rule: code in `shared/` must not depend on any feature module. It must be purely generic. If you add a line to `shared/events/` that references `User` or `Order`, you have broken the abstraction. Move that code into a feature module.

### When to Choose Each

Layer-first works well for small to medium codebases (under 50,000 lines) where a single team owns multiple layers and they need a high-level organizational map. It works poorly when multiple teams work on independent features or when the codebase is so large that the `services/` folder has become unnavigable.

Feature-first works well for medium to large codebases with multiple teams, each owning features independently. It scales to hundreds of thousands of lines. It breaks down when you have so much shared code that you cannot fit it all in a `shared/` folder, or when the shared code becomes so complex that it needs its own internal layers.

The best practice: **start with feature-first if the codebase will live more than two years or be touched by more than one team**. Feature-first does not add overhead for small codebases, but it provides an escape hatch for growth.

---

## Internal vs Public API

Every module should expose a *small* API and hide everything else. This principle is true at every scale: in a single class, in a file, in a folder, in a service. The size of the API determines how much surface other code depends on, and thus how expensive changes become.

### C++ Headers: Public Interface, Private Implementation

In C++, the distinction is formalized in headers and source files:

```cpp
// user_service.h - PUBLIC API
#pragma once
#include <string>
#include <optional>

namespace users {
    struct User {
        std::string id;
        std::string email;
    };
    
    class UserService {
    public:
        std::optional<User> findById(const std::string& id);
        bool createUser(const std::string& email, const std::string& pwd);
    private:
        // Implementation details hidden
        class Impl;
        std::unique_ptr<Impl> impl_;
    };
}
```

```cpp
// user_service.cpp - PRIVATE IMPLEMENTATION
#include "user_service.h"
#include "user_repository.h"
#include "email_validator.h"

namespace users {
    class UserService::Impl {
    public:
        std::unique_ptr<UserRepository> repo_;
        std::unique_ptr<EmailValidator> validator_;
        
        // All internal helper methods here
        bool validatePassword(const std::string& pwd);
        std::string hashPassword(const std::string& pwd);
    };
    
    // Implementation
    std::optional<User> UserService::findById(const std::string& id) {
        return impl_->repo_->getUserById(id);
    }
}
```

The header declares what the module exposes; the source file contains everything else. Code outside the module sees only the header. It cannot see `Impl`, cannot see `repo_`, cannot see the helpers. This is enforceable: if you try to access `UserService::Impl` from another file, the compiler will reject it.

The cost is small: one extra indirection through the `Impl` pointer. The benefit is large: the implementation can change — you can replace `repo_` with a different implementation, add new helper methods, refactor the internal structure — without affecting callers.

### Java and C#: Package-Private and Internal

In Java, the distinction is formalized with access modifiers:

```java
// users/UserService.java - PUBLIC
public class UserService {
    public Optional<User> findById(String id) {
        // ...
    }
    
    // Package-private: visible only to other classes in the users package
    void sendWelcomeEmail(User user) {
        // ...
    }
}

// users/internal/UserRepository.java - INTERNAL
class UserRepository {
    // Default access: visible only to the users package
    User getUserById(String id) {
        // ...
    }
}
```

Classes that are only used internally go in an `internal/` sub-package or are marked package-private. Other modules can see the package-public classes, but not the implementation details.

### TypeScript/JavaScript: Barrel Files and Index Exports

In TypeScript and JavaScript, the convention is explicit: each module has an `index.ts` or `index.js` that re-exports only the public API:

```javascript
// users/user_service.ts
export class UserService {
    async findById(id: string): Promise<User | null> {
        // ...
    }
    
    // Internal method, not exported
    private async validateEmail(email: string): Promise<boolean> {
        // ...
    }
}

class UserRepository {
    async getUserById(id: string): Promise<User | null> {
        // ...
    }
}

// users/index.ts - PUBLIC API (barrel file)
export { UserService } from './user_service';
export type { User } from './types';
// UserRepository is NOT exported — it is internal
```

Other modules import from `users/` only, not from `users/user_service.ts` directly. The barrel file controls what is visible.

```javascript
// payment/payment_service.ts
import { UserService } from '../users';  // OK: UserService is exported
// import { UserRepository } from '../users/user_service';  // ERROR: cannot access internal
```

### The Discipline Matters More Than the Language Mechanism

The language provides the means, but discipline provides the effect. In C++, you can mark the `Impl` class `public` instead of private and undo all the protection. In Java, you can make helpers `public` when they should be package-private. In TypeScript, you can import directly from the internal file instead of through the barrel.

The rule: **if it is not in the public API file/header/barrel, do not depend on it**. Enforce this in code review, document it in CONTRIBUTING.md, and check it with tooling.

---

## Monorepo vs Polyrepo

A monorepo contains multiple projects in a single version-control repository. A polyrepo contains one project per repository. This choice determines how tightly code is coupled at the infrastructure level.

### Monorepo: One Repository, Many Projects

In a monorepo, you might have:

```
my-platform/
├── backend/
│   ├── user_service/
│   ├── order_service/
│   ├── payment_service/
├── frontend/
│   ├── web/
│   ├── admin/
├── shared/
│   ├── protos/
│   ├── types/
│   ├── test_fixtures/
```

A single commit can change code across services. A feature can include backend, frontend, and shared code in one PR. The version history is unified.

**Advantages:**
- Cross-project changes are easy. If you refactor a shared type, you update all consumers in one PR.
- Tooling is simpler. One CI system, one build system, one deployment system.
- Visibility is high. You can trace any usage of a piece of code across the entire platform.

**Disadvantages:**
- Build and test are slower. Every commit triggers a full build of all affected projects, which grows as the monorepo grows.
- Repository size grows over time. Cloning becomes slow; operations become slow.
- CI/CD becomes complex. You need sophisticated tooling (Bazel, Nx, Turborepo) to avoid rebuilding everything.
- Ownership is ambiguous. If there is one repository, does one team own it, or many? This must be made explicit with CODEOWNERS files.

### Polyrepo: One Repository Per Project

In a polyrepo:

```
user-service/              (separate repo)
├── src/
├── tests/
├── Dockerfile

order-service/             (separate repo)
├── src/
├── tests/
├── Dockerfile

shared-types/              (separate repo)
├── src/
├── package.json           (published as npm package or similar)
```

Each project is independent. A change to the user service does not trigger a build of the order service.

**Advantages:**
- Clear ownership. Each team owns a repository and controls its release cycle.
- Fast CI/CD. Changes to one service do not affect others. Build times are short.
- Independent deployment. Services can be deployed on different schedules.

**Disadvantages:**
- Cross-project changes are hard. If you change a shared type, you must bump the version in the shared-types repository, release it, and update the dependency in each consuming service. This takes multiple PRs and multiple releases.
- Visibility is poor. Usages of code across repositories are invisible unless you have a searchable artifact repository or manually grep all repositories.
- Tooling coordination is needed. Shared CI/CD configuration, shared testing standards, and shared deployment conventions must be documented and enforced across repositories.

### Hybrid: Monorepo with Weak Coupling

The best monorepos treat sub-projects as loosely coupled as polyrepos: each has its own build configuration, its own dependency graph, its own release cycle. Tools like Bazel and Nx enable this by tracking fine-grained dependencies and rebuilding only what has changed.

In Bazel, a monorepo looks like:

```
WORKSPACE
BUILD

backend/
├── user_service/
│   ├── BUILD
│   ├── src/
├── order_service/
│   ├── BUILD
│   ├── src/

shared/
├── types/
│   ├── BUILD
│   ├── src/
```

Each `BUILD` file declares dependencies explicitly. Bazel computes a fine-grained dependency graph and rebuilds only the targets affected by a change. A change to `user_service/` does not trigger a rebuild of `order_service/` unless it depends on it.

### When to Choose Each

**Choose monorepo if:**
- You have a small number of projects (under 10) that share code frequently.
- You have a single team that owns all projects.
- You can invest in build infrastructure (Bazel, Nx, Turborepo).
- Cross-project refactors happen frequently.

**Choose polyrepo if:**
- You have many independent projects (over 20).
- You have multiple teams, each owning one project.
- Projects have independent release cycles.
- You want to minimize CI/CD infrastructure.

The hybrid monorepo approach — one repository with weak coupling between projects — scales to hundreds of projects if you have the tooling.

---

## Dependency Direction Rules

A well-organized codebase has a *direction* to dependencies. Higher-level code depends on lower-level code; lower-level code never depends on higher-level code. This principle prevents cycles and makes the codebase navigable.

### The Dependency Inversion Principle

The Dependency Inversion Principle (DIP) states: **high-level modules should not depend on low-level modules. Both should depend on abstractions.**

In a payment system:

```
Level 0 (Highest): Business Logic (checkout flow)
  ↓ depends on
Level 1: Abstractions (PaymentProcessor interface)
  ↑ depends on (through the interface)
Level 2 (Lowest): Implementations (StripeProcessor, PaypalProcessor)
```

The checkout flow (high-level) does not depend on Stripe or Paypal (low-level). It depends on an abstraction: `PaymentProcessor`. The payment processors implement that interface.

If the checkout flow depended directly on `StripeProcessor`, then the system would be inverted: a low-level detail (Stripe's API) would control the high-level logic. Changing payment providers would require rewriting the checkout flow. That is not inversion; that is just bad architecture.

```cpp
// BAD: High-level depends on low-level
class CheckoutFlow {
public:
    bool checkout(const Order& order) {
        StripeProcessor stripe;  // Direct dependency on Stripe
        return stripe.chargeCard(order.total);
    }
};

// GOOD: Both depend on abstraction
class PaymentProcessor {
public:
    virtual ~PaymentProcessor() = default;
    virtual bool chargeCard(double amount) = 0;
};

class CheckoutFlow {
private:
    std::unique_ptr<PaymentProcessor> processor_;
public:
    bool checkout(const Order& order) {
        return processor_->chargeCard(order.total);  // Depends on abstraction
    }
};

// Implementations
class StripeProcessor : public PaymentProcessor {
    bool chargeCard(double amount) override {
        // Stripe-specific code
    }
};
```

### Detecting Bad Dependencies: Import Linters

As a codebase grows, maintaining dependency direction becomes difficult to enforce manually. Import-linting tools make it automatic:

**Python: `import-linter`**

```yaml
# .import-linter.ini
[importlinter:contract:1]
name = Layers should depend downward
type = layers
modules =
    checkout
    payments
    stripe

allowed_cycles =
    # none
```

Running `lint-imports` will check that `checkout` does not import from `stripe` or `payments`, and that `payments` does not import from `stripe`.

**TypeScript: `depcop` and `depcheck`**

```javascript
// depcheck.json
{
  "restrictions": [
    {
      "from": "src/checkout",
      "to": "src/stripe",
      "message": "Checkout should not depend on Stripe. Use PaymentProcessor abstraction."
    }
  ]
}
```

**Java: `arch-unit` (in tests)**

```java
@AnalyzeClasses(packages = "com.payment")
public class DependencyRulesTest {
    
    @ArchTest
    static final ArchRule checkout_should_not_depend_on_stripe =
        classes().that().resideInAPackage("..checkout..")
            .should().notDependOnClassesThat()
            .resideInAPackage("..stripe..");
}
```

Running the test suite will fail if the rule is violated. This catches bad dependencies before they are committed.

### Concrete Example: A Bad Dependency Caught in CI

Imagine a developer adds this line to the checkout service:

```cpp
// checkout_service.cpp
#include "stripe/stripe_api.h"

bool CheckoutService::processOrder(const Order& order) {
    StripeAPI api;  // Direct dependency on Stripe
    return api.charge(order.customer_id, order.total);
}
```

Without tooling, this might be committed and merge conflicts would ripple through the codebase for weeks. With an import-linter in the CI:

```
CI Output:
ERROR: checkout depends on stripe
  File: checkout_service.cpp, line 5: #include "stripe/stripe_api.h"
  Rule: checkout should not depend on stripe
  Allowed: checkout → payment_processor → stripe

Solution: Inject PaymentProcessor dependency into CheckoutService
instead of creating StripeAPI directly.
```

The PR fails until the developer uses dependency injection. The rule is enforced before the code lands.

---

## Build Times as a Signal

Build time is not just a developer convenience issue — it is an architectural signal. Long build times indicate too much coupling.

### Why Build Times Grow

In C++, when you change a `.h` file, everything that includes it (directly or transitively) must be recompiled. If your headers include too much, you trigger cascading recompiles:

```cpp
// user.h
#pragma once
#include "order.h"
#include "payment.h"
#include "inventory.h"
#include "shipping.h"

struct User {
    int id;
    std::string email;
};
```

Now, if `inventory.h` changes, it triggers a recompile of `user.h`, which triggers a recompile of every file that includes `user.h`. The compile time grows quadratically with the number of includes.

The fix: **minimize transitive includes**. Use forward declarations:

```cpp
// user.h
#pragma once
#include <string>

// Forward declare, don't include
class Order;
class Payment;
class Inventory;
class Shipping;

struct User {
    int id;
    std::string email;
    
    // Methods that use Order, Payment, etc. must be out-of-line
    void placeOrder(Order& order);
};
```

```cpp
// user.cpp
#include "user.h"
#include "order.h"

void User::placeOrder(Order& order) {
    // Now you can use Order
}
```

When `inventory.h` changes, `user.h` does not need recompilation. Only the files that *actually* use `inventory.h` are recompiled.

### Pimpl: Hiding Complexity Behind a Pointer

The Pimpl (Pointer to Implementation) idiom hides the internal structure behind a pointer:

```cpp
// user_service.h
#pragma once
#include <memory>

class UserRepository;
class EmailValidator;

class UserService {
public:
    void registerUser(const std::string& email, const std::string& pwd);
private:
    class Impl;
    std::unique_ptr<Impl> impl_;
};
```

```cpp
// user_service.cpp
#include "user_service.h"
#include "user_repository.h"
#include "email_validator.h"

class UserService::Impl {
public:
    std::unique_ptr<UserRepository> repo;
    std::unique_ptr<EmailValidator> validator;
};

void UserService::registerUser(const std::string& email, const std::string& pwd) {
    impl_->validator->validate(email);
    impl_->repo->create(email, pwd);
}
```

The header no longer needs to include `user_repository.h` or `email_validator.h`. The implementation details are hidden. When those files change, only `user_service.cpp` is recompiled, not every file that includes `user_service.h`.

### C++20 Modules and Code Splitting

C++20 modules promise to eliminate the header/source distinction and improve compile times dramatically. Until they are standard, the best approach is aggressive forward declaration and Pimpl.

In web development (JavaScript/TypeScript), the equivalent is code splitting: splitting your bundle into chunks that are loaded on-demand. A change to one module only invalidates the chunk that contains it, not the entire bundle.

### Reading the Build Graph

High-quality build systems (Bazel, Ninja) emit a dependency graph. Analyzing it reveals coupling:

```
files_that_include_user.h: 143
files_that_include_payment.h: 87
files_that_include_config.h: 412  <-- COUPLING ALERT

files_that_include_util.h: 230    <-- TOO GENERIC
```

If `config.h` is included by 412 files, a change to it triggers 412 recompiles. That is a sign that `config.h` is too broad. Split it into `config_database.h`, `config_logging.h`, etc., and include only what you need.

---

## Code Ownership and CODEOWNERS

As a codebase scales, you need clarity: who can approve changes to which parts? If anyone can change anything, then no one is responsible. If the responsibility is unclear, then important changes might not be reviewed at all.

### CODEOWNERS File

GitHub, GitLab, and Bitbucket support a `CODEOWNERS` file that specifies which people or teams must review changes:

```
# In root: CODEOWNERS (or .github/CODEOWNERS)

# All files
* @platform-team

# User service
src/users/ @users-team
src/users/admin.cpp @users-team @compliance-team  # Admin features need compliance sign-off

# Payment service
src/payments/ @payments-team
src/payments/pci.cpp @payments-team @security-team  # PCI compliance needs security review

# Shared code
src/shared/ @platform-team

# Build configuration
Dockerfile @devops-team
kubernetes/ @devops-team
```

When a PR modifies `src/payments/stripe.cpp`, GitHub requires approval from `@payments-team` before it can be merged. If the PR also touches `src/payments/pci.cpp`, it requires approval from both `@payments-team` and `@security-team`.

### Visibility in Build Files

In Bazel and similar systems, you can restrict which modules can depend on which:

```python
# src/payments/BUILD
cc_library(
    name = "payment_processor",
    srcs = ["payment_processor.cpp"],
    hdrs = ["payment_processor.h"],
    visibility = [
        "//src/checkout:__pkg__",
        "//src/admin:__pkg__",
        # Only checkout and admin can use this
    ]
)

cc_library(
    name = "pci_internal",
    srcs = ["pci.cpp"],
    visibility = ["//src/payments:__pkg__"],  # Internal to payments only
)
```

If a team tries to depend on `pci_internal` from `//src/admin`, the build fails before the code is even merged.

### Ownership Without Fiefdoms

Ownership must be scoped and explicit. A bad ownership model:

```
* @founder  # Founder reviews all changes
```

This is a bottleneck. The founder becomes the gatekeeper; the system cannot scale.

A better model:

```
src/users/ @users-team
src/payments/ @payments-team
src/orders/ @orders-team
src/shared/ @platform-team
```

Each team owns their module. The platform team owns shared code. A change to `users/` requires approval from the users team, not from a single person or the entire company.

Changes to shared code that affect multiple teams might require multiple approvals:

```
src/shared/events/ @platform-team @users-team @payments-team @orders-team
```

Or specify a escalation:

```
src/shared/ @platform-team  # Platform team approves by default
src/shared/critical/ @security-team @platform-team  # Critical paths require security
```

---

## Worked Example: Reorganizing from Layer-First to Feature-First

Imagine you have inherited a 200,000-line codebase that started layer-first and is now collapsing under its own disorganization:

```
src/
├── controllers/ (127 files, 45K lines)
├── services/ (156 files, 68K lines)
├── repositories/ (103 files, 42K lines)
├── models/ (87 files, 31K lines)
```

The `services/` folder is the worst offender: 156 files, developers frequently merge conflicts, changes ripple across the folder. You decide to reorganize to feature-first.

**Step 1: Identify Features**

Based on the domain, you identify: users, orders, payments, inventory, shipping, reporting, admin.

**Step 2: Create Feature Folders**

```
src/
├── users/
├── orders/
├── payments/
├── inventory/
├── shipping/
├── reporting/
├── admin/
├── shared/
```

**Step 3: Move Code by Feature**

Analyzing the imports, you determine which controllers/services/repositories belong to which feature. For each feature folder, you create the internal layer structure:

```
src/users/
├── controllers/
│   ├── user_controller.cpp
├── services/
│   ├── user_service.cpp
├── repositories/
│   ├── user_repository.cpp
├── models/
│   ├── user.cpp
```

**Step 4: Extract Shared Code**

Code that is used by multiple features goes into `shared/`:

```
src/shared/
├── database/
│   ├── connection.cpp
├── events/
│   ├── event_bus.cpp
├── validation/
├── logging/
```

**Step 5: Update Dependencies**

The biggest work: updating `#include` statements and import paths. A script can do 90% of this:

```python
# migrate_imports.py
old_path = "services/user_service.h"
new_path = "users/services/user_service.h"

# In all files that imported old_path, replace with new_path
```

**Step 6: Add CODEOWNERS and BUILD Visibility**

```
src/users/ @users-team
src/payments/ @payments-team
src/shared/ @platform-team
```

**Step 7: Test and Merge**

All tests pass (the structure change should not break functionality). The branch is merged, and the codebase is reorganized.

**Result:**

```
Before:
  services/user_service.h included by 47 files
  services/payment_service.h included by 31 files
  services/order_service.h included by 52 files
  Merge conflicts per week: 15-20

After:
  users/services/user_service.h included by 8 files (within users)
  payments/services/payment_service.h included by 5 files (within payments)
  orders/services/order_service.h included by 11 files (within orders)
  Merge conflicts per week: 3-4
```

The reduction is not because the code changed; it is because the structure now aligns with team boundaries and feature ownership. Conflicts drop by 75% because teams no longer collide in the same folders.

---

## Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|------------|
| **Layer-First** | Clear structural map, obvious where to find code | Scales poorly, merge conflicts, bottleneck at layer level | Small teams, single owner, codebase under 100K lines |
| **Feature-First** | Scales to many teams, aligned with ownership, few merge conflicts | Requires shared code discipline, less obvious structural map | Multiple teams, codebase growing beyond 100K lines |
| **Vertical Slices (hybrid)** | Combines benefits of both, scales indefinitely | Requires careful governance of shared folder | Medium to large codebases with multiple teams |
| **Monorepo** | Easy cross-project refactors, unified history, high visibility | Slow CI/CD, large repo size, complex tooling needed | Small number of projects, single team, invest in Bazel/Nx |
| **Polyrepo** | Fast CI/CD, independent release cycles, clear ownership | Hard cross-project changes, poor visibility, duplicate tooling | Many independent projects, multiple teams |

---

## Common Misconceptions

### "More Folders = Better Organized"

A folder with one file is not organized; it is scattered. A folder with 100 files is not disorganized; it depends on whether those 100 files are coherent. Organize by meaningful grouping, not by arbitrary subdivision. If `users/admin/` has only 2 files, move them to `users/`. If `users/` has 150 files, split by feature: `users/authentication/`, `users/profile/`, `users/admin/`.

### "If I Move Code, I'll Lose the Git History"

Git history follows content, not location. If you rename a file or move it to a different folder, `git log` and `git blame` still work. If you use `git mv` instead of deleting and recreating, the history is unbroken. Reorganizing is safe; git remembers.

### "Dependencies Are Only Imports"

A dependency is any assumption about another module's behavior. If module A directly accesses global state that module B manages, that is a dependency, even if there is no `#include`. If module A assumes that module B's data is in a specific format, that is a dependency. Focus on logical dependencies, not just syntactic ones. Tools like `depcop` catch the logical ones.

### "We Don't Have Ownership Yet, So We Don't Need CODEOWNERS"

You have implicit ownership. Someone owns the users module; they just don't know it yet. Making it explicit prevents surprises. CODEOWNERS is free enforcement of what should already be true.

---

## Exercises

1. **Analyze your codebase.** Create a dependency graph of your modules (or use Bazel/Nx to generate one). Identify cycles, long dependency chains, and heavily-included files. Are dependencies pointing consistently in one direction, or are they chaotic?

2. **Calculate merge conflict cost.** For the past month, count merge conflicts by folder. In which folders do most conflicts occur? What are the teams working on those folders? Does the folder structure align with team boundaries?

3. **Estimate build time reduction.** Pick one heavily-included header in your codebase. Measure compile time before and after converting direct includes to forward declarations and Pimpl. How much faster is the build?

4. **Write a CODEOWNERS file.** List your modules and assign ownership to teams or individuals. As an exercise, propose changes to the file that would reduce merge conflicts (e.g., splitting a module that multiple teams edit).

5. **Plan a reorganization.** If your codebase is layer-first and has more than 100K lines, sketch a reorganization to feature-first. What features would you identify? Which code is shared? What are the risks and benefits?

---

## Summary

Large codebases do not fail because developers are incompetent; they fail because structure breaks down. At 50,000 lines, organization stops being aesthetic and becomes structural. Choose a folder layout that aligns with team boundaries and feature ownership — typically feature-first with a shared utilities folder. Expose a minimal public API from each module and hide implementation details. Use tooling (import-linters, CODEOWNERS, BUILD file visibility) to enforce rules automatically. Measure build times as a signal of coupling; use forward declarations and Pimpl to reduce them. Ownership without authority creates chaos; make it explicit with CODEOWNERS files and scoped visibility. A well-organized large codebase is a competitive advantage: features are built faster, bugs are isolated more easily, and new team members onboard more quickly.

---

> **[← Previous: Boundaries — When To Split Logic](11-boundaries-when-to-split.md)**  ·  **[↑ Part 4](README.md)**  ·  **[Next: Architectural Decision Records →](13-architectural-decision-records.md)**
