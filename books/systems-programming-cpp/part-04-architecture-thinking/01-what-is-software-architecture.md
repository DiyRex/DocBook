# Chapter 33 — What Is Software Architecture

## Learning Objectives

By the end of this chapter you will be able to:

1. Define software architecture precisely: the set of decisions that are expensive to change, everything else is implementation.
2. Distinguish between architecture, design, and implementation — three scales of decision, each with a different cost of reversal.
3. Apply the Cost-of-Change Test to judge whether a decision is architectural.
4. Identify quality attributes (non-functional requirements) and explain why they trade against each other — you cannot maximize all simultaneously.
5. Recognize what architecture is not: drawing diagrams, following a named pattern, adopting a particular style.
6. Write and maintain Architecture Decision Records (ADRs) to document significant choices and their rationale.
7. Evaluate architectural tradeoffs concretely by analyzing a worked example with three different styles.
8. Spot common misconceptions about architecture and understand the reality behind them.

---

## 33.1 The One-Sentence Definition

**Software architecture is the set of decisions that are expensive to change. Everything else is implementation.**

This sentence contains nearly all you need to know. Let's unpack it.

A decision is architectural if reversing it costs much more than getting it right the first time. The cost can be measured in engineering hours, calendar time, risk to the system, or some combination. The cost includes not just the technical reversal but the coordinated effort across teams, the migration of live systems, the retesting, the communication overhead.

What makes a decision expensive to change?

1. **Scope** — the decision affects many modules, services, or teams. Changing the database engine affects every part of the system that talks to persistence. Changing a variable name affects maybe three call sites.

2. **Ripple** — reversing the decision requires reversing downstream decisions. Switching from single-process to microservices forces you to add service discovery, distributed tracing, message queues. You can't just revert one piece.

3. **Coordination** — the decision couples teams. Moving from one database to another requires coordination between backend, devops, data, analytics. Renaming a local variable does not.

4. **Risk** — getting the decision wrong has high consequences. A choice of database engine that does not scale can tank a product. Misalignment with an edge case in a utility function is a bug fix.

Decisions low in all four dimensions are *not* architectural — they are implementation details. You can change them freely; when you do, you learn immediately if the change was good.

---

## 33.2 Architecture vs Design vs Implementation

When we talk about decisions in software, there are three scales. The boundaries are fuzzy, but the distinction is useful.

### Architecture: What Gets Split Into What

Architecture is about system boundaries. "What module, service, process, or machine does what?"

Examples of architectural decisions:

- Monolith vs distributed services
- Synchronous request/response vs event-driven
- Single database vs database per service
- Push-based (publishing events) vs pull-based (polling)
- Frontend in the same process as backend vs separate
- Shared code repository vs separate repos
- Centralized logging vs application-managed logs

These decisions define the *shape* of the system. Once made, they ripple everywhere. Changing from monolith to services at year three is possible (companies do it) but expensive. The earlier you make the decision, the cheaper it is to reverse — but you also know less about the problem.

### Design: Interface and Behavior of One Module

Design is about how a single module — a service, a component, a class — looks from the outside and how it works from the inside.

Examples of design decisions:

- What public methods does a service expose?
- Is the API synchronous, async, streaming?
- What are the inputs, outputs, failure modes?
- How does the module achieve its invariants?
- What are the main branches and data flows?

Design decisions are local to a module. They cost less to change than architectural ones (no coordination across service boundaries) but more than implementation (you may need to update clients).

### Implementation: The Code Itself

Implementation is everything else. How a data structure is laid out, what a function's local variable is called, whether you use a for-loop or recursion, the exact order of statements.

Examples of implementation decisions:

- Should we use `std::vector` or `std::deque`?
- How is the state machine encoded?
- What's the exact sequence of validation steps?
- Which helper functions exist?

These decisions are cheap to change. You can refactor freely. The cost is the time to write good tests and the review cycle, not system-wide coordination.

### The Boundary Is Fuzzy, But The Principle Is Clear

A database choice affects the entire application and costs a year of engineering time to reverse. It is architectural.

A choice of authentication library affects many modules but costs a few weeks to swap in a different library. It is closer to design.

The name of a local variable affects one module and costs minutes to change. It is implementation.

**The skill is developing an intuition for where the boundary is.** This comes from experience, but the Cost-of-Change Test (§33.3) is the tool that sharpens that intuition.

---

## 33.3 The Cost-of-Change Test

Ask this question about any decision: **How much would it cost to reverse this decision after it is deployed and the system is live?**

Measure cost in engineering-weeks, not lines of code. Include:

- **Technical work**: how many files, how many modules, how many services would you need to touch?
- **Testing and validation**: how much test coverage would you need to verify the change is correct?
- **Coordination**: how many teams would need to align?
- **Data migration**: if the decision involves how data is stored or transmitted, what migration complexity is there?
- **Risk and safety**: can you roll this change out gradually, or is it all-or-nothing?
- **Learning curve**: how much would the team need to re-learn?

Examples:

**Database choice**: You pick PostgreSQL on day one. Three years later, you realize Redis would be better for your access pattern. Cost: months. You need to design a migration strategy, copy all data, verify correctness on live data, roll out gradually, update every service that queries the database, rewrite queries (which may have different semantics), test disaster recovery. This is architectural.

**API response format**: You pick JSON. Later you want to switch to protobuf. Cost: weeks. You need to update clients and servers, add a version negotiation, test on live traffic. This is design.

**Variable name**: You call it `user_list`. Later you want to call it `active_users`. Cost: minutes. Rename and run tests. Implementation.

**Microservices vs monolith**: You start monolithic. Three years later you want to split into services. Cost: many months to over a year. You need to identify service boundaries (hard), set up inter-service communication, design data consistency strategy, add distributed logging/tracing, set up deployment pipelines, coordinate team splits. Architectural.

**Function signature**: A public function takes `(int user_id, bool include_deleted)`. You want to change it to `(GetUserRequest req)` for better extensibility. Cost: days to a week. You update all call sites, test, coordinate with consuming services if it's a public API. Design.

Notice: a decision becomes more expensive to change the later you realize you should change it, *and* the more dependent code you have written. This is why getting architectural decisions right early matters, and why premature architecture is also a trap — you make a high-cost decision before you have information.

---

## 33.4 Quality Attributes (Non-Functional Requirements)

A system has *functional requirements* — "it must calculate a user's balance" — and *non-functional requirements*, also called quality attributes.

Quality attributes are properties the system must exhibit. The main ones:

**Performance**: How fast does it respond? What is the latency percentile (p50, p99)? How much throughput (requests per second)?

**Scalability**: Can it handle growth? Horizontal (add more machines) or vertical (bigger machines)? What axis — users, events, storage?

**Availability**: Does it stay up? What is the target uptime (99.9%, 99.99%)? What is the recovery time if it fails?

**Reliability**: How often does it fail? How does it fail (gracefully or catastrophically)? Can it survive one component breaking?

**Security**: Can it defend against attacks? Confidentiality (can only the right people read data)? Integrity (can you trust the data is not modified)? Authenticity (can you trust who sent this)?

**Modifiability**: How easy is it to add a feature? How many places do you need to change? How isolated are changes?

**Testability**: How easy is it to verify the system works? Can you test in isolation?

**Deployability**: How easy is it to release a change? Instant, or must you coordinate? One data center or many?

**Understandability**: Can a new engineer understand it? How long to onboard? Where is the complexity?

### The Tradeoff Core

Here is the hard truth: **you cannot maximize all quality attributes simultaneously.** Every architectural choice trades off one attribute against another.

Examples:

- **Performance vs Modifiability**: To make the system faster, you might tightly couple components or hardcode behavior. To make it modifiable, you add abstraction layers, which add indirection and performance cost.

- **Scalability vs Simplicity**: To scale horizontally, you add distributed caches, message queues, and consistency protocols. Each adds complexity. A single process is simpler but does not scale.

- **Availability vs Cost**: High availability (multi-region, instant failover) costs much more than single-region. You trade money for uptime.

- **Security vs Usability**: Perfect security is friction (multi-factor auth, rate limits, request signing). More usable systems allow shortcuts (weaker authentication, fewer audit logs). Healthcare and fintech go high-security; consumer apps go high-usability.

- **Deployability vs Data Consistency**: Fast deploys (canary, blue-green) are easy in stateless services. Services with long-lived state are harder to deploy without risking inconsistency.

This is why "what is a good architecture?" is unanswerable in the abstract. Good architecture is one that **prioritizes the right quality attributes for your problem**.

If your business logic is "we must never lose a transaction," you optimize for reliability and possibly accept performance cost (synchronous writes, consensus protocols).

If your business logic is "fresh data is nice, but loss is acceptable," you optimize for performance and accept higher failure rates (async writes, eventual consistency, graceful degradation).

If you are a startup with limited resources, you optimize for deployability and modifiability (get features out fast). If you are Google with unlimited resources, you can afford to optimize for performance (spend a year optimizing caches).

---

## 33.5 What Architecture Is Not

### Myth 1: Architecture is the diagram

Many engineers think "architecture" is a box-and-arrow diagram, maybe in C4, UML, or ArchiMate notation. The diagram is a *communication artifact*. It is useful for explaining architecture to others. But the diagram is not the architecture.

The architecture is the decisions themselves: what goes in each box, why it's there, what the boxes can and cannot do, what happens when one fails, how data flows between them.

A beautiful diagram of a non-existent architecture is worse than useless. A scribbled sketch of an architecture that actually works is valuable.

### Myth 2: Architecture is applying a named pattern

"We are using Clean Architecture" or "we are using microservices" or "we are using the repository pattern." These are styles or patterns — collections of decisions that other people have documented.

Real architecture is making *your* decisions. You might use parts of Clean Architecture or microservices, but cherry-picking the pieces that make sense for your problem is architecture. Applying a pattern wholesale because it's famous is not.

The worst code is code written to "follow the pattern," not to solve the problem. Read the pattern; understand the tradeoffs; make your own choices.

### Myth 3: Architecture is using microservices

Some engineers believe that "architecture" means "break into lots of services." Microservices is one architectural choice. It solves real problems (independent scaling, team isolation, deployment independence) and introduces real ones (distributed tracing, consensus complexity, network latency, distributed debugging).

A well-architected monolith outscales a poorly-architected microservices mess. A monolith that is modular in code (good internal boundaries) is easier to split later than a monolith where everything is tangled.

Microservices is a tool, not a goal. Use it when the problem demands it, not because it is fashionable.

### Myth 4: Architecture is what architects do

In some large organizations, there is a role called "architect" and an assumption that only architects think about architecture. This is a organizational failure, not a definition.

Architecture happens at every scale. A junior engineer choosing a data structure is making a micro-architectural decision. A tech lead choosing which services exist is making a mid-level decision. A principal engineer choosing whether to use a distributed cache is making a system-level decision.

Good organizations push architectural thinking down. Every engineer should be able to explain the tradeoffs in their local corner.

---

## 33.6 Architecture Decision Records (ADRs)

An Architecture Decision Record is a short, structured document that captures one architectural decision.

An ADR has this shape:

```
# ADR-NNN: Title

## Status
Accepted / Proposed / Deprecated / Superseded by ADR-NNN

## Context
Why does this decision matter? What is the problem?
What are the constraints or assumptions?

## Decision
What are we choosing and why?

## Rationale
What tradeoffs are we making?
What alternatives did we consider?
Why is this better than the alternatives?

## Consequences
What becomes easier?
What becomes harder?
What will we need to revisit?
```

An example:

```
# ADR-003: Use PostgreSQL for primary data store

## Status
Accepted

## Context
We are building a SaaS product with structured user, account, and transaction data.
We need ACID guarantees (no lost payments).
We need to scale to millions of rows.
Our team has PostgreSQL expertise.
We need to run on commodity cloud infrastructure.

## Decision
We will use PostgreSQL as the primary data store.

## Rationale
PostgreSQL provides ACID guarantees, which eliminates a class of data-loss bugs.
It has a rich query language (SQL) that lets us express complex access patterns.
It has good tooling and managed services (RDS, Cloud SQL).
It is mature and well-understood.

We considered MongoDB but rejected it because:
- Lack of ACID transactions in early versions (improved now)
- Weak consistency guarantees for our use case
- Less mature at scale for financial transactions

We considered Cassandra but rejected it because:
- Eventual consistency is not suitable for money
- Higher operational complexity for our team size

## Consequences
We are bound to relational schemas (hard to change structure).
We accept that horizontal scaling is harder than with NoSQL.
We must handle postgres-specific things: connection pooling, query optimization, backup/restore.
We can now use complex queries and transactions, making application logic simpler.
```

Why ADRs matter:

1. **Searchable future**: when someone asks "why do we use Postgres," you have a document that answers it.
2. **Preserves context**: "we chose this because of X" is lost if only one person remembers. ADRs keep context.
3. **Forces clarity**: writing an ADR forces you to articulate why you made a choice. Half-baked reasoning becomes obvious.
4. **Tracks reversals**: if you later decide to switch databases, you create a new ADR and reference the old one. This is the record of the system's evolution.
5. **Onboarding**: new engineers can read the ADR and understand not just what was chosen but why.

**Best practice**: One ADR per significant decision. Not every decision (that is overhead), but decisions that are architectural by the Cost-of-Change Test. Update them as they evolve. Keep them in version control near the code.

---

## 33.7 Worked Example: Logging System, Three Architectural Styles

Let's see architecture in action. The problem is simple: "Log incoming events and summarize them every minute."

Here are three architectural choices and their quality attribute tradeoffs.

### Style 1: Single Process, Synchronous

The application runs as one process. When an event arrives, it is processed and logged immediately. Every minute, it summarizes and outputs the summary.

```cpp
#include <iostream>
#include <deque>
#include <chrono>
#include <thread>

struct Event {
    std::string type;
    int value;
    std::chrono::system_clock::time_point time;
};

int main() {
    std::deque<Event> events;

    // Simulate incoming events
    for (int i = 0; i < 100; ++i) {
        Event e{"user_action", i, std::chrono::system_clock::now()};
        events.push_back(e);
        std::cout << "logged: " << e.type << " " << e.value << "\n";

        // Every 100 events, summarize
        if (events.size() >= 100) {
            int sum = 0;
            for (const auto& ev : events) sum += ev.value;
            std::cout << "SUMMARY: " << sum << " total from "
                      << events.size() << " events\n";
            events.clear();
        }

        // Simulate time passing
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }

    return 0;
}
```

**Quality attributes:**

- **Performance**: Excellent. No network, no IPC, no queues. Direct memory access.
- **Simplicity**: Excellent. One process, one thread (mostly). Easy to test.
- **Modifiability**: Good. To change logging, you modify the main loop.
- **Scalability**: Poor. One process handles everything. Cannot scale to multiple machines.
- **Availability**: Poor. If the process crashes, all logging stops.
- **Testability**: Good. Pass in test events, check output.

**When to use**: Small systems, single-machine deployments, prototypes, systems where the entire event stream fits in one box.

### Style 2: Single Process with Internal Queue

The process has two threads: one accepts events and puts them in a queue, one reads the queue and logs/summarizes. This decouples ingestion from processing.

```cpp
#include <iostream>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>

struct Event {
    std::string type;
    int value;
};

int main() {
    std::queue<Event> event_queue;
    std::mutex queue_lock;
    std::condition_variable event_ready;
    bool done = false;

    // Thread 1: accept events
    std::thread ingester([&]() {
        for (int i = 0; i < 100; ++i) {
            Event e{"action", i};
            {
                std::lock_guard<std::mutex> lock(queue_lock);
                event_queue.push(e);
            }
            event_ready.notify_one();
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
        {
            std::lock_guard<std::mutex> lock(queue_lock);
            done = true;
        }
        event_ready.notify_one();
    });

    // Thread 2: process events
    std::thread processor([&]() {
        int batch_sum = 0;
        int batch_count = 0;
        while (true) {
            {
                std::unique_lock<std::mutex> lock(queue_lock);
                event_ready.wait(lock, [&]() { return !event_queue.empty() || done; });

                while (!event_queue.empty()) {
                    Event e = event_queue.front();
                    event_queue.pop();
                    batch_sum += e.value;
                    batch_count++;
                    std::cout << "processed: " << e.type << " " << e.value << "\n";

                    if (batch_count == 25) {
                        std::cout << "SUMMARY: " << batch_sum << " from " 
                                  << batch_count << " events\n";
                        batch_sum = 0;
                        batch_count = 0;
                    }
                }
            }
            if (done) break;
        }
    });

    ingester.join();
    processor.join();
    return 0;
}
```

**Quality attributes:**

- **Performance**: Good. Ingestion and processing overlap. Latency is higher (queue overhead) but throughput is better.
- **Availability**: Better. If processing is slow, ingestion doesn't block. If one thread crashes, the other might recover (depending on signals).
- **Modifiability**: Good. Ingestion and processing are separate threads; you can change one without touching the other.
- **Scalability**: Still poor. Still one process. If you have multiple ingestion sources or multiple summaries, you need more coordination.
- **Concurrency safety**: Poor. Mutexes, condition variables, potential deadlocks, data races if you forget a lock.
- **Testability**: Harder. You need to test synchronization; race conditions are hard to reproduce.

**When to use**: Single-machine systems where throughput or responsiveness matters more than simplicity. When decoupling ingestion from processing helps.

### Style 3: Two Services with Message Broker

The ingestion service receives events and puts them in a message broker (e.g., Kafka or RabbitMQ). A separate processing service consumes from the broker and logs/summarizes.

```
┌──────────────────┐        ┌────────────────┐        ┌──────────────────┐
│ Ingestion        │        │ Message Broker │        │ Processing       │
│ Service          │───────▶│ (Kafka)        │───────▶│ Service          │
│                  │        │                │        │                  │
│ HTTP endpoint    │        │ Event topic    │        │ Read events      │
│ Accept events    │        │                │        │ Summarize        │
└──────────────────┘        └────────────────┘        └──────────────────┘
```

```cpp
// Pseudo-code; actual implementation would use librdkafka or similar

// Ingestion service:
void handle_event(const Event& e) {
    producer.send("events", e.to_bytes());  // Async send
}

// Processing service:
void process() {
    auto consumer = kafka::Consumer("events");
    int batch_sum = 0, batch_count = 0;
    for (const auto& msg : consumer) {
        Event e = Event::from_bytes(msg.value());
        batch_sum += e.value;
        batch_count++;

        if (batch_count == 25) {
            std::cout << "SUMMARY: " << batch_sum << "\n";
            batch_sum = 0;
            batch_count = 0;
        }
    }
}
```

**Quality attributes:**

- **Performance**: Acceptable. Message broker adds latency (network round-trip) but better throughput if many events.
- **Scalability**: Excellent. Each service scales independently. Add more ingestion replicas, add more processing replicas. Broker handles distribution.
- **Availability**: Excellent. If ingestion crashes, the broker retains messages. If processing crashes, another replica picks up where it left off.
- **Modifiability**: Good. The two services are independent; you can change one without touching the other. However, they are now coordinated through the broker contract.
- **Operational complexity**: High. You now have a broker to manage, monitoring to set up, distributed tracing to understand message flow.
- **Testability**: Moderate. You can test each service in isolation by mocking the broker. Testing the integration requires a broker.
- **Consistency and ordering**: Depends on the broker. Kafka guarantees message order per partition but complexity grows if you have many partitions.

**When to use**: Systems that need to scale components independently, handle high event rates, or survive component failure. When you have multiple teams building different parts.

### Summary of Tradeoffs

| Attribute | Style 1 (Monolithic) | Style 2 (Internal Queue) | Style 3 (Services + Broker) |
|-----------|---|---|---|
| Performance | Excellent | Good | Acceptable |
| Simplicity | Excellent | Good | Poor |
| Scalability | Poor | Poor | Excellent |
| Availability | Poor | Fair | Excellent |
| Modifiability (internal) | Good | Good | Good |
| Deployability | Excellent | Excellent | Good |
| Testability | Excellent | Good | Moderate |
| Operational complexity | None | Low | High |

**Each style is right for a different problem:**

- **Style 1**: You are a startup with five events per second and one engineer. Time-to-market matters. Use the simplest thing.
- **Style 2**: You are at 10,000 events per second on one box. Simplicity matters but throughput requires decoupling. One machine can still handle the load.
- **Style 3**: You are at 100,000 events per second across multiple data centers. You need fault tolerance and independent scaling. Operational complexity is worth the benefit.

---

## 33.8 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Architecture is the diagram." | The diagram is a communication artifact. Architecture is the decisions. Plenty of poor diagrams of good systems; plenty of pretty diagrams of non-existent systems. |
| "There is one best architecture." | There is no best — only better or worse for a specific problem. A monolith is best for a startup. Microservices is best for a 500-person organization. Both claims are true. |
| "We should design the architecture upfront before writing code." | You need to make some big choices early (persistence, services, deployment), but too much design before code is wasted effort. Start with a rough shape, build, learn, adapt. |
| "Microservices means good architecture." | Microservices is a tool, not a goal. A monolith with clean boundaries is better than a mess of microservices. Conversely, microservices is the only way to solve some scaling problems. |
| "Architecture is for big systems." | Architecture thinking applies at all scales. A script with no architecture will become unmaintainable. An app with bad architecture will waste money. Architecture is about making good tradeoffs, not about size. |
| "The architect decides the architecture." | In healthy organizations, architectural thinking is distributed. The architect facilitates, documents, and enforces consistency, but good engineers at all levels make architectural decisions in their corner. |
| "Good architecture requires the most advanced technology." | No. Good architecture matches the problem. Sometimes the best choice is boring (PostgreSQL, REST, single monolith). Technology is a tool, not a virtue. |
| "Architecture is done once and then it doesn't change." | Architecture evolves. Successful systems' architectures change as they scale, as business needs shift, as the team grows. Evolution happens through ADRs and versioning, but it is constant. |

---

## 33.9 Exercises

1. **The Cost-of-Change Test**: Pick a decision in a system you know — database, language, deployment platform, authentication mechanism, something. Estimate how much it would cost to reverse it now (hours, days, weeks). What would happen if you reversed it in production? Identify which of the four cost dimensions (scope, ripple, coordination, risk) contributes most to the cost.

2. **Quality Attributes in Practice**: Look at a system you know (work codebase, open-source, anything). List five quality attributes it optimizes for and five it de-prioritizes. For each, explain what you see in the architecture that reveals this choice. (Hint: you can see priorities by what the system handles well and what it handles poorly.)

3. **Tradeoff Analysis**: Design a system for one of these scenarios:
   - A mobile app that works offline and syncs when online
   - A payment processor that must never lose a transaction
   - A social media feed that must load in under 1 second
   
   For your system, identify three architectural choices you would make. For each, list the quality attributes it optimizes for and the ones it sacrifices.

4. **Write an ADR**: Revisit a significant decision in a codebase you maintain or know well. Write an ADR for it, including context, decision, alternatives, and consequences. If the decision is no longer current, mark it as "Superseded" and explain what replaced it and why.

5. **Compare Styles**: For a problem you are solving, design two different architectural styles (like the logging example). For each, list the quality attributes with scores: excellent, good, acceptable, poor. When would you use each style?

6. **Identify Leaking Abstraction**: Some architectural choices leak in practice. Find one example (e.g., a "service-oriented" system where services are so tightly coupled that changing one requires changing five others). What does this reveal about whether the architecture matched the problem? What would you change?

---

## 33.10 Summary

Software architecture is the set of decisions that are expensive to change. These decisions define system boundaries, how components interact, which quality attributes are prioritized, and what tradeoffs are accepted.

The Cost-of-Change Test — estimating the cost to reverse a decision after deployment — separates architectural decisions from design and implementation. It is a useful heuristic that sharpens over time.

You cannot maximize all quality attributes. Every architecture trades off performance for simplicity, or scalability for understandability, or availability for cost. Good architecture recognizes these tradeoffs explicitly and makes choices that fit the problem.

Architecture is not a diagram, a pattern, or a role. It is thinking about how the system is organized and why. This thinking happens at all scales and in all roles. ADRs are lightweight tools to document these choices and preserve the reasoning.

The worked example showed that there is no universal "best" architecture — only better or worse for a specific problem and specific quality attributes. A monolith is right for a startup. Microservices is right for a 500-person organization. Both are architecture, and both are right given the context.

As you move into Part 4, each chapter will deepen these ideas. We will see how layers, services, repositories, and controllers are answers to the question "how should this system be organized?" and what tradeoffs each implies.

---

## 33.11 What's Next

You now have the conceptual frame for architectural thinking. Architecture is not a monolith of decisions; it is a web of decisions at different scales, made at different times, with different costs and tradeoffs.

Chapter 34 explores why large programs inevitably become complex, and what architecture can do to manage that complexity rather than pretend it doesn't exist. You will learn the patterns that emerge when teams build large systems — not because they chose a named pattern, but because the problem itself demands certain shapes.

---

**[← Previous: Dependency Injection From Scratch](../part-03-understanding-abstractions/10-dependency-injection-from-scratch.md)** · **[↑ Part 4](README.md)** · **[Next: Why Large Programs Become Complex →](02-why-large-programs-become-complex.md)**
