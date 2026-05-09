# Chapter 94 — Recognizing Architectural Patterns

When you open a codebase for the first time, or read about a team's "novel architecture," you will almost always recognize it immediately. Not because you have seen *that exact codebase* before, but because the underlying pattern recurs. A lending library service and a food delivery platform look nothing alike on the surface, but both organize code as layers, repositories, and services. A chat application and a financial data processor both emit and consume events. A monolithic Rails application and a Node.js microservice deployed separately are both variations of the same architectural family.

This chapter is a field guide. By the end, you will be able to walk into any codebase, recognize its pattern from its directory structure and code organization, understand what that pattern was designed to solve, and see what it trades away. You will also understand how patterns compose—most real systems are not pure implementations of one pattern but layers of several, each solving a different problem.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Name and recognize a dozen foundational architectural patterns from code structure and folder layout.
2. Understand what problem each pattern solves and what it costs.
3. Identify when a pattern is appropriate for the problem and when it is cargo-culted overhead.
4. Recognize how patterns compose: layered + event-driven, hexagonal + CQRS, modular monolith + feature flags.
5. Spot anti-patterns that masquerade as architecture—big ball of mud, distributed monolith, god classes.
6. Make conscious decisions about which patterns fit your specific constraints.

---

## The Pattern Catalog

### Layered Architecture

**Structure**: Code organized as horizontal layers—Presentation, Domain, Persistence. Each layer depends on the layer below it.

```
┌──────────────────────┐
│   Presentation       │  (HTTP handlers, CLI)
├──────────────────────┤
│   Domain/Services    │  (Business logic)
├──────────────────────┤
│   Persistence        │  (Repositories, queries)
└──────────────────────┘
```

**Real-world example**: A Django application with `views.py`, `models.py`, and `serializers.py`. An Express.js app with `controllers/`, `services/`, and `repositories/`. A Rails monolith with app models, controllers, and helpers organized by feature.

**Problem it solves**: Separates infrastructure concerns (HTTP, database) from business logic. Allows the domain layer to be tested without a web server or database. Makes it clear which code runs where and why.

**When to recognize it**: Look at the folder structure. Does it have a clear distinction between web/API handlers and business logic? Are there repository or data access layer files? Can you trace a request from handler → service → repository?

**Cost**: Extra indirection. A simple `GET /user/:id` might traverse Handler → Service → Repository → ORM → Database instead of querying directly. For trivial CRUD, this is overhead. For systems with complex rules, this is clarity.

**Variations**: Strict three-layer (never skip a layer) vs. pragmatic (skip the service for simple queries). Hexagonal architecture inverts dependencies (Chapter 37) so the domain defines what it needs, not what it provides.

---

### Hexagonal Architecture (Ports and Adapters)

**Structure**: The domain is at the center. Ports are the interfaces the domain requires. Adapters implement those ports, connecting the domain to the outside world (HTTP, databases, queues, etc.).

```
External Systems
    |  |  |
    ↓  ↓  ↓
  Adapters (HTTP, DB, Queue)
    ↑  ↑  ↑
    |  |  |
    Ports (interfaces)
        |
        ↓
    DOMAIN CORE
    (Business Logic)
```

**Real-world example**: A Java Spring service where repositories are interfaces defined in the domain, implemented in the infrastructure layer. A Go application where business logic depends on interfaces that are mocked in tests and implemented with HTTP or database adapters in production.

**Problem it solves**: Makes the domain the center of the system. Infrastructure is pluggable. Tests mock adapters without touching the domain. The same domain logic works behind HTTP, gRPC, message queues, or CLIs.

**When to recognize it**: Look for folders named `ports/`, `adapters/`, or `interfaces/`. Check if repositories are interfaces that are implemented separately. See if the domain layer imports nothing from the framework. Can you test domain logic without any infrastructure?

**Cost**: More interfaces. Feels over-engineered for a simple CRUD app, essential for systems with multiple entry points or frequent framework changes.

---

### CQRS + Event Sourcing

**Structure**: Command Query Responsibility Segregation separates read and write models. Event Sourcing persists the immutable history of events, reconstructing current state from that history.

```
Write Path:
  Command → Command Handler → Domain Logic → Emit Events → Event Store

Read Path:
  Query → Read Model (materialized view) → Return data

Event Sourcing:
  All writes produce events.
  Current state = event1 + event2 + event3 + ...
  No UPDATE statements. Only INSERT events.
```

**Real-world example**: A trading system where every trade is an immutable event (you never edit a trade, you record a correction or reversal). An e-commerce platform with one write model (OrderService handling commands like PlaceOrder, CancelOrder) and separate read models for "customer orders," "inventory status," and "analytics."

**Problem it solves**: Writes and reads have different shapes. For writes, you need strong consistency and validation. For reads, you need fast queries. CQRS lets you optimize each path separately. Event Sourcing gives you a complete audit trail and the ability to rebuild state or analyze historical patterns.

**When to recognize it**: Look for event tables. Do inserts outnumber updates dramatically? Are there separate "query" or "reporting" tables that look different from the write models? Are events versioned? Is there an event bus or message broker? Can you replay the history and reconstruct past state?

**Cost**: Eventual consistency. The read models lag behind writes. Debugging is harder (a bug might be in the event generation, the event store, or the projection). More infrastructure (event store, event bus, projections).

**When to use**: Financial systems (audit trail matters), event-rich domains (the history is valuable), systems with complex read patterns that don't match the write model.

---

### Microservices

**Structure**: Multiple independently deployable services, each owning its own data, communicating over network boundaries (HTTP, gRPC, message queues).

```
Service A          Service B          Service C
(User Management)  (Orders)           (Inventory)
  |                  |                  |
  ├─ Database     ├─ Database     ├─ Database
  |                  |                  |
  └──────────────────────────────────────
       (Network, HTTP/gRPC/Queues)
```

**Real-world example**: Amazon's architecture (supposedly 100+ microservices). Netflix (hundreds of services). A team with separate Payments, Notifications, and Shipping services, each deployed independently, communicating via REST or message queues.

**Problem it solves**: Teams can own, deploy, and scale services independently. A bug in Notifications doesn't bring down Orders. You can use different languages for different services (Python for ML, Go for high-throughput, Java for stability).

**When to recognize it**: Multiple Git repositories or folders with independent deployable units. Each service has its own database. Communication is network-based. There is a service registry, load balancer, or API gateway. Deployment is separate for each service.

**Cost**: Distributed systems problems. Network calls are not function calls—they fail, they timeout, they are slow. Debugging spans multiple logs and services. Data consistency becomes eventual. Testing is harder (you need to stub or mock services). Operational overhead (deployment, monitoring, tracing).

**When to use**: Teams large enough to own services independently (Amazon's two-pizza rule). High-scale systems where independent scaling matters. Organizations that need services to deploy on different cadences.

**Common misconception**: Microservices are not about REST or small services. They are about independent deployability and ownership. A monolithic domain can be split across multiple deployed artifacts.

---

### Modular Monolith

**Structure**: One deployable artifact, but strong module boundaries. Modules do not reach across boundaries without going through public APIs. Inter-module calls are controlled, visible, and often enforced via linting.

```
One Deployment Package
├─ Module: Users
│  └─ users_service.cpp
├─ Module: Orders
│  └─ orders_service.cpp
├─ Module: Inventory
│  └─ inventory_service.cpp
└─ Orchestration (calls modules' APIs)

Rules:
  - Users module exports users_service.cpp
  - Orders calls users_service, not users_database directly
  - Dependencies between modules are explicit and limited
```

**Real-world example**: A Ruby on Rails application with enforced boundaries (Packwerk gems or similar). A Java application using Maven modules with strict inter-module visibility rules. A Go application where each `internal/users/`, `internal/orders/`, `internal/inventory/` is a package with a clean API.

**Problem it solves**: Captures many benefits of microservices (clear module boundaries, independent testing, controlled dependencies) without the distributed systems overhead. You can refactor across modules within one transaction. Debugging is simpler (all code in one place).

**When to recognize it**: One deployment artifact, but modules have clear boundaries. There is linting or tooling that prevents crossing module boundaries. Folder structure shows distinct modules. Dependencies between modules go through explicit APIs.

**Cost**: You still have the complexity of module boundaries, but without the benefit of independent scaling or deployment. Modules can become cargo-cult if they are not truly independent.

**When to use**: Teams that want module structure but cannot yet justify the operational overhead of microservices. Pre-microservices architecture as a stepping stone.

---

### Pipeline / ETL (Extract, Transform, Load)

**Structure**: Data flows through stages. Each stage takes input, transforms it, produces output that feeds the next stage.

```
Input → Extract → Transform → Load → Output
         (Parse)  (Enrich)    (Store) (Result)

Or with parallelism:
├─ Source A ──┐
├─ Source B ──┼→ Transform ──→ Load ──→ Output
├─ Source C ──┘
```

**Real-world example**: A data pipeline that extracts raw events from a log, transforms them into customer behavior data, loads into a data warehouse. A build system that compiles → links → packages. An image processing pipeline: upload → resize → compress → store.

**Problem it solves**: Organizes work as distinct, serializable stages. Each stage can be tested independently. Failures are localized (if loading fails, you know the data was already transformed correctly). Parallelism is explicit (multiple sources feeding one transform).

**When to recognize it**: Look for queue-like structure. Is data being queued between stages? Are there intermediate files or tables? Does configuration describe the pipeline as a sequence? Are there monitor jobs that pick up where a failed job left off?

**Cost**: Intermediate storage. If stages are not co-located, network overhead. Debugging a multi-stage failure is harder (did the bug happen in extract, transform, or load?).

**When to use**: Data processing (batch jobs, stream processing), build systems, workflow orchestration.

---

### Actor Model

**Structure**: Concurrent entities (actors) that encapsulate state and communicate only via asynchronous messages. No shared memory.

```
Actor A ───→ send(msg) ───→ Actor B
Actor B ───→ send(msg) ───→ Actor C
```

**Real-world example**: Akka actors in Scala/Java. Erlang/Elixir processes. A distributed system where each entity is an actor with its own queue, processing one message at a time.

**Problem it solves**: Avoids shared-memory concurrency problems (locks, atomics, data races). Each actor is single-threaded (at least logically); only messages cross thread boundaries. Scales to thousands of concurrent entities.

**When to recognize it**: Look for event queues or message passing. Do actors/entities have state that is not shared? Is there an actor framework (Akka, Orleans)? Are updates happening via messages, not method calls?

**Cost**: Different mental model. Debugging distributed actor systems is hard (message arrival order matters, timing is subtle). Testing async behavior is trickier than synchronous tests.

**When to use**: Highly concurrent systems where traditional threading is too fine-grained. Systems that need to distribute across multiple machines (Akka cluster).

---

### Plugin Host / Plugin Architecture

**Structure**: A small core system with a plugin interface. Plugins extend the core without modifying it.

```
Core (minimal)
├─ PluginHost
├─ PluginInterface (abstract)
└─ PluginRegistry

Plugins (loaded at startup or runtime)
├─ Plugin A (implements PluginInterface)
├─ Plugin B (implements PluginInterface)
├─ Plugin C (implements PluginInterface)
```

**Real-world example**: Visual Studio Code (core editor, thousands of extensions). Kubernetes (core API, hundreds of plugins for storage, networking, etc.). A game engine where game logic is a plugin, but the rendering engine is the core.

**Problem it solves**: Allows extensibility without modifying the core. New plugins can be added without recompiling or redeploying the core. Plugins can be third-party or even user-written.

**When to recognize it**: Look for an interface or registry that plugins implement. Plugins are loaded dynamically. The core is stable; extensions come and go. There is a plugin marketplace or plugin directory.

**Cost**: The core must be stable and well-designed (bad core design makes plugins awkward). Plugin discovery and loading adds complexity.

**When to use**: Systems designed to be extended. IDEs, browsers, game engines, infrastructure platforms.

---

### Client-Server / N-Tier Architecture

**Structure**: Clear separation between client (requests) and server (responds). Servers are stateless. N-tier extends this with load balancers, caches, multiple app servers, etc.

```
┌─────────────┐
│  Clients    │  (Web browsers, mobile apps, CLIs)
└──────┬──────┘
       │ HTTP/HTTPS
┌──────▼──────────┐
│  Load Balancer  │
└──────┬──────────┘
       │
   ┌───┼────┐
   │   │    │
┌──▼─┐│  ┌─▼──┐
│App │├─→│App │  (Stateless servers)
│ 1  │   │ 2  │
└────┘   └────┘
   │   (Cache, Sessions)
   ↓
┌──────────────┐
│  Database    │
└──────────────┘
```

**Real-world example**: Every web application deployed on the internet. AWS Lambda behind an API Gateway. A REST API with multiple app server instances behind a load balancer.

**Problem it solves**: Scales horizontally. Clients do not know about server internals. Servers can fail and be replaced. New app servers can be added for more capacity.

**When to recognize it**: Multiple app server instances behind a load balancer. Sessions might be stored in a shared cache. Servers are stateless (each request could go to any server). External HTTP/REST interface.

**Cost**: Session management is tricky (if session is stored only on Server A, but next request goes to Server B, the session is lost). Caching adds complexity. Consistency is harder.

**When to use**: Web applications, REST APIs, anything that needs to scale horizontally.

---

### Event-Driven Architecture

**Structure**: Components communicate by emitting and consuming events. No direct coupling between event producers and consumers.

```
┌────────────┐
│  Producer  │
└──────┬─────┘
       │ emit(UserCreated)
       ↓
┌────────────────────┐
│   Event Bus/Topic  │
└────────────────────┘
    │            │
    │ subscribe  │ subscribe
    ↓            ↓
┌──────────┐  ┌─────────────┐
│Consumer 1│  │ Consumer 2   │
│(Email)   │  │ (Analytics)  │
└──────────┘  └─────────────┘
```

**Real-world example**: A user registration system where "UserRegistered" event is published. An email service subscribes and sends a welcome email. An analytics service subscribes and logs the registration. An inventory system subscribes and adjusts stock.

**Problem it solves**: Producers and consumers are decoupled. New consumers can be added without modifying producers. Failures are isolated (email service crashes, but registration completes). Asynchronous, so fast operations don't wait for slow ones.

**When to recognize it**: Look for event topics, message brokers (RabbitMQ, Kafka, AWS SNS/SQS). Is there a pub-sub pattern? Do changes result in events being emitted? Are there separate service modules that "listen" for events?

**Cost**: Eventual consistency. Debugging causality is hard (which event caused which change?). Testing event flow is complex (you need to set up the event broker).

**When to use**: Asynchronous systems, systems with many stakeholders that need to react to changes independently, systems that need to decouple producer and consumer.

---

## How to Recognize Each Pattern From Code

### Layered
- **Folder structure**: `controllers/`, `services/`, `repositories/`, or `app/`, `domain/`, `persistence/`.
- **File names**: `user_service.cpp`, `user_repository.cpp`, `user_controller.cpp`.
- **Dependency direction**: Handler imports Service imports Repository.
- **Absence of**: Database code in HTTP handlers. Business logic in repository implementations.

### Hexagonal
- **Folder structure**: `domain/`, `ports/`, `adapters/`.
- **Key files**: `repository.hpp` (interface, in domain), `postgres_repository.cpp` (implementation, in adapters).
- **Dependency direction**: Adapters depend on domain, not reverse.
- **Test setup**: Domain tests mock the ports; no adapters are instantiated.

### CQRS
- **Table structure**: Many INSERT-only event tables. Separate read tables.
- **File names**: `*_command.cpp`, `*_query.cpp`, `*_event.cpp`.
- **Absence of**: UPDATE statements in write paths.
- **Presence of**: Event handlers that update read models.

### Microservices
- **Repository structure**: Separate repos or separate deploy units.
- **Communication**: HTTP/gRPC/queues between services.
- **Database ownership**: Each service owns its database.
- **Deployment**: Each service has its own deployment pipeline.

### Modular Monolith
- **One deployment**: Single binary or JAR.
- **Module boundaries**: `internal/users/`, `internal/orders/`, each with clear APIs.
- **Linting**: Tools that enforce module boundaries.
- **Imports**: Explicit, controlled cross-module dependencies.

### Pipeline/ETL
- **Stages**: Extract, Transform, Load, or custom stages.
- **Queues/files**: Intermediate data between stages.
- **Configuration**: Describes the pipeline as a sequence.
- **Recovery**: Restart from a failed stage.

### Actor
- **Framework**: Akka, Erlang, Orleans, or hand-rolled.
- **Message queues**: Each actor has a mailbox.
- **Immutable communication**: State changes via messages, not shared memory.

### Plugin Host
- **Interface**: PluginInterface or Extension base class.
- **Registry**: Central place where plugins register.
- **Loading**: Plugins loaded at startup, possibly dynamically.

### Client-Server
- **Entry point**: HTTP/REST endpoints.
- **Load balancer**: Multiple app server instances.
- **Statelessness**: Each request could go to any server.

### Event-Driven
- **Event broker**: RabbitMQ, Kafka, Redis Pub/Sub, AWS SNS/SQS.
- **Publishers and subscribers**: Clear separation.
- **No direct calls**: Components don't call each other directly; they emit events.

---

## Recognizing Anti-Patterns

Anti-patterns are patterns that look like they solve a problem but actually create more problems.

### Big Ball of Mud
**What it looks like**: No clear structure. Code is organized by accident. A request might touch any file, in any order.

**Why it fails**: Impossible to understand. Changes ripple everywhere. Testing is a nightmare because everything depends on everything else.

**Fix**: Introduce layers. Or if the system is small enough, move it to a new language or framework where you enforce structure from the start.

### God Class
**What it looks like**: One class that has hundreds of methods and knows about everything. It is the User, the Order, the Payment, the Notification—all in one.

**Why it fails**: It becomes impossible to change without breaking everything. Testing requires instantiating the entire class and mocking half the internet.

**Fix**: Split the class. User should know about being a user. OrderService should handle orders. Notification should handle notifications.

### Stovepipe System
**What it looks like**: Multiple implementations of the same thing scattered across the codebase or the organization. Three different "user" systems in three different parts of the app, three different ways to persist data.

**Why it fails**: Inconsistency, bugs that exist in one stovepipe but not another, duplicate work.

**Fix**: Unify. Create one User system that everyone uses. Create one persistence layer. It is harder short-term but saves enormous time long-term.

### Distributed Monolith
**What it looks like**: Microservices architecture (multiple services) but they all share a database. Changes to the database schema require coordinating multiple teams.

**Why it fails**: You have the overhead of distributed systems (network calls, eventual consistency) without the benefit (independence). A database schema change breaks all services.

**Fix**: Give each service its own database. If they need to share data, they must communicate via APIs, not shared tables.

### Layer-as-Pass-Through
**What it looks like**: A service method that does nothing but call a repository method. A repository that does nothing but wrap an ORM call.

**Why it fails**: Indirection without value. You traverse multiple function calls for no benefit.

**Fix**: Delete the pass-through. Let callers use the repository directly if there is no logic in the service.

---

## Why Patterns Compose

Most real systems are not pure implementations of one pattern. Instead, patterns layer on top of each other.

**Layered + Event-Driven**: A system with clear layers (presentation, domain, persistence) where the domain emits events that are consumed by other services or listeners. The layers provide organization; events provide decoupling.

**Hexagonal + CQRS**: A domain at the center with clear ports for what it needs (repositories, external services). Reads and writes are split. Commands mutate domain state and emit events. Queries hit read models.

**Modular Monolith + Pipeline**: One deployed artifact with strong module boundaries. But the data flow through the system follows a pipeline: extract from source → transform in module A → enrich in module B → load into warehouse.

**Microservices + Event-Driven**: Multiple services with their own databases, communicating via events. A user registration service emits "UserRegistered," and payment, email, and analytics services subscribe.

**Client-Server + Caching + Async**: Web clients talk to a load-balanced server. The server uses caching to reduce database load. Background jobs handle async work. All three patterns are necessary to make the system work.

The art is recognizing which patterns your system needs and implementing them consistently. A system with layers but no event support is brittle. A system with events but no layers is a ball of mud. Both matter.

---

## When a Pattern Is Wrong for the Problem

**Microservices for a three-person team**: You gain distributed complexity, lose the ability to refactor across service boundaries within one transaction. Not worth it.

**CQRS for a simple CRUD app**: Separate read and write models adds complexity. If reads and writes are simple and similar, stick with one model.

**Hexagonal for a 1000-line CLI tool**: You don't need ports and adapters. A simple main function with no layers is fine.

**Event Sourcing for a straightforward operational database**: Store the current state. If you need the history for auditing, add a separate audit log. Event sourcing adds complexity; use it when the history is as valuable as the current state.

**Plugin architecture for a closed system**: If no one will ever extend it, don't build for extensibility.

**The test**: Does the pattern solve a real problem in your system, or does it add overhead without benefit?

Real architectural decisions are about recognizing the problem and choosing the pattern that solves it. Cargo-culting patterns because "that's how Netflix does it" or "I read this was best practice" is how systems become over-engineered. Understand the tradeoff. Make it consciously. Write it down (Chapter 13 — ADRs).

---

## Worked Example: A Rails Monolith Evaluated

Let's trace through a real codebase: a medium-sized Rails application for a booking system.

### Structure
```
app/
  models/          (Domain: Booking, Property, Host, Availability)
  controllers/     (Presentation: bookings_controller, properties_controller)
  services/        (Domain orchestration: BookingService, ReviewService)
  repositories/    (Persistence: None—Rails uses ActiveRecord directly)
  jobs/            (Background: SendConfirmationEmail, UpdateAvailability)
```

### Patterns Identified

1. **Layered**: Controllers → Services → Models. Dependencies point downward.

2. **Models with Rules**: Models are not anemic. A `Booking` knows whether it can be cancelled, computes the total price, validates dates. This is rich domain modeling.

3. **Service Layer**: Services orchestrate bookings, reviews, notifications. They call models and repositories (via ActiveRecord).

4. **Event-like behavior via Jobs**: Background jobs are triggered (when a booking is created, send confirmation email). Not a true event bus, but the spirit is there.

5. **No hexagonal/ports**: Rails tightly couples models to the database (ActiveRecord knows about the database schema). This is not hexagonal. It is pragmatic—ActiveRecord *is* the abstraction.

### If You Migrated to Microservices

You would:

1. **Split by feature**: Bookings service, Properties service, Reviews service. Each owns its database.

2. **Replace jobs with events**: Instead of a job that runs async, emit events (BookingCreated, BookingCancelled) and have separate services listen.

3. **Add API boundaries**: Services talk via REST or gRPC, not in-process method calls.

4. **Lose transactions**: A booking that triggers an email and an availability update now spans three services. If the email fails, the booking still exists. You must handle eventual consistency.

5. **Gain independent scaling**: Bookings service can run 10 instances; reviews service can run 2.

### The Tradeoff

The monolith is simpler, easier to debug, allows transactions. Microservices are more complex but allow independent scaling and deployment.

The decision depends on scale and team size. For a three-person team processing 100 bookings/day, the monolith is obviously better. For a 30-person team processing 100,000 bookings/day with multiple teams working independently, microservices make sense.

---

## Tradeoffs

| Pattern | Pros | Cons | When to Use |
|---------|------|------|-------------|
| **Layered** | Clear responsibility; easy to test layers independently; dependencies are predictable | Extra indirection; forces pass-through methods; traversal overhead | Most applications where domain logic matters |
| **Hexagonal** | Domain is independent; tests don't need infrastructure; easy to swap adapters | More interfaces to design; feels over-engineered for small systems | Multiple entry points (HTTP + CLI + queue); framework changes anticipated |
| **CQRS + Event Sourcing** | Complete audit trail; reads and writes optimized separately; history is first-class | Eventual consistency; more infrastructure; debugging is harder | Financial systems; event-rich domains; complex read patterns |
| **Microservices** | Independent scaling, deployment, team ownership; technology diversity | Distributed systems complexity; harder debugging; operational overhead; eventual consistency | Large teams, high scale, independent deployment needs |
| **Modular Monolith** | Module boundaries with lower distributed systems cost | Not truly independent; still need strong discipline | Medium-sized teams; pre-microservices stepping stone |
| **Pipeline/ETL** | Clear stages; parallelizable; failures are localized | Intermediate storage; harder end-to-end debugging | Data processing; build systems; batch jobs |
| **Actor Model** | Avoids shared-memory concurrency; scales to many concurrent entities | Different mental model; debugging timing-sensitive issues is hard | Highly concurrent systems; distributed actor systems |
| **Plugin Host** | Extensible without modifying core; third-party extensions | Core design must be stable; complexity of plugin loading | IDEs, game engines, infrastructure platforms |
| **Client-Server** | Horizontal scalability; stateless; simple mental model | Session management; consistency is harder | Most web applications |
| **Event-Driven** | Loose coupling; asynchronous; new consumers can be added easily | Eventual consistency; causality debugging is hard; more infrastructure | Asynchronous systems; many independent stakeholders |

---

## Common Misconceptions

1. **"Layered architecture is the only real architecture."** No. It is the most common and usually the right default. But hexagonal, event-driven, and actor-based are equally valid. Choose based on the problem.

2. **"Microservices are the future. Monoliths are dead."** Microservices are necessary at scale with large teams. For small teams and constrained budgets, a well-structured monolith beats a poorly-managed set of microservices.

3. **"Design patterns are timeless rules."** They are guidelines, not laws. A pattern is appropriate if it solves a real problem in your system. If it adds overhead without solving a problem, skip it.

4. **"You should design for the architecture you *might* need."** No. Design for the architecture you *do* need. Premature generalization is as bad as premature optimization. Add flexibility when you have a real reason.

5. **"Event Sourcing means you never delete data."** It means you model data as an append-only log of changes. You can still delete data if legally required (GDPR). It is just a design choice, not a constraint.

6. **"CQRS requires eventual consistency."** The read model can be synchronously updated (same transaction as the write). It is more complex, but possible. The point of CQRS is that read and write models *can* be different, not that they *must* be.

---

## Exercises

1. **Identify the patterns in a codebase you know.** Open a project you work on or know well. Map its structure. What patterns do you see? Are they pure (one pattern throughout) or mixed (layers in some parts, events in others)? Write a brief analysis.

2. **Sketch what microservices would look like.** Take a monolithic codebase. Draw lines showing where services would split. What would each service own (data, logic, endpoints)? What messages would pass between them? Don't implement it; just design it.

3. **Event audit trail.** Take a business process you know (user registration, order placement, loan approval). Model it as events. What events occur? What listeners react to them? How would the system change if a new requirement came in (send SMS on stage 2, for example)?

4. **Reverse-engineer ADRs.** For a codebase you know, infer what architectural decisions were made. Why are there repositories? Why are there services? Write imaginary ADRs explaining those decisions—context, rationale, consequences.

5. **Convert a pattern.** Take a simple monolithic system (a Rails app or Flask app). Design how it would look as an event-driven set of services. What becomes easier? What becomes harder? What new infrastructure is needed?

6. **Spot the anti-pattern.** Read a codebase. Find one instance of a big ball of mud, god class, stovepipe, or pass-through layer. Propose a refactoring. What pattern would fix it?

---

## Summary

Architectural patterns recur. A lending library service and a food delivery system both use layers and repositories. A chat app and a financial data processor both emit and consume events. When you walk into a new codebase, you will recognize the pattern from its structure.

Each pattern solves specific problems: layers separate concerns, hexagonal inverts dependencies, CQRS splits read and write, microservices enable independent deployment, events decouple producers and consumers.

Most real systems compose multiple patterns. A microservices architecture (pattern) uses event-driven communication (pattern) and each service is organized in layers (pattern).

The art is choosing which patterns solve real problems in your system and which add overhead. A three-person team does not need microservices. A simple CRUD app does not need CQRS. A 1000-line script does not need hexagonal architecture. Use your judgment. Understand the tradeoff. Write it down.

---

> **[← Previous: Mapping Framework Concepts Across Languages](03-mapping-framework-concepts.md)**  ·  **[↑ Part 9](README.md)**  ·  **[Next: Understanding Any Codebase Quickly →](05-understanding-any-codebase.md)**
