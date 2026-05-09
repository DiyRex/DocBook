# Chapter 73 — An Event Bus

An event bus is a decoupling mechanism. Instead of saying "A calls B," you say "A announces something happened; anyone interested in that something can listen." The emitter doesn't know the listeners, the listeners don't know each other, and if a listener is slow or fails, the emitter doesn't care. Done well, this is one of the cleanest patterns in software. Done poorly, it becomes a black box where nobody knows what triggered what.

This chapter builds event buses—starting simple, then facing the design choices that make the difference between "this works" and "this works at scale."

---

## Learning Objectives

By the end of this chapter you will:

1. Understand what an event is: a fact, in the past tense, that something happened.
2. Design and implement an in-process event bus with type erasure.
3. Recognize the tradeoff between synchronous (simple) and asynchronous (decoupled timing) event handling.
4. Explain delivery semantics and why exactly-once is mostly impossible.
5. Handle ordering guarantees and subscription patterns.
6. Know when an event bus is the right tool and when it is not.

---

## 73.1 What An Event Is

An event is a statement that something happened. Past tense. Immutable. Owned by the bounded context that emitted it.

Good events:
- **OrderPlaced** — a fact that an order moved from "draft" to "placed" state.
- **PaymentProcessed** — a fact that a payment succeeded and cleared.
- **CustomerEmailChanged** — a fact that a customer's email was updated.

Bad events (usually):
- **ShouldProcessOrder** — a command, not an event. Commands are requests; events are facts.
- **GetOrderStatus** — a query, not an event.
- **OrderPlacedBecauseCustomerAskedUs** — too much narrative. The event is the fact; the reasons are implicit.

An event is a data structure. It has:
- A **type** that names what happened.
- **Data** relevant to the fact (order ID, customer ID, timestamp).
- A **timestamp** (when it was emitted).
- Optionally, a **correlation ID** (to link related events across contexts).

```cpp
// An event: immutable, past-tense, self-contained
struct OrderPlaced {
    uint64_t order_id;
    uint64_t customer_id;
    Money total;
    std::chrono::system_clock::time_point emitted_at;
    std::string correlation_id;  // link to originating request
};

// Not an event—it's a command
struct PlaceOrder {
    uint64_t order_id;
    uint64_t customer_id;
    Money total;
};
```

Events are emitted by the domain model (the aggregate). The model doesn't know who listens; it only knows to emit the event.

---

## 73.2 Events as Domain Language

Before diving into implementation, understand that events are part of the domain language (from Chapter 41: Domain-Driven Thinking). They represent concepts the business cares about. When a domain expert says "an order was placed," that phrase lives in the code as an event.

This has implications:
- Event names use the domain's past tense. Not "PlaceOrder" (a command) but "OrderPlaced" (a fact).
- Events are owned by the bounded context that creates them. If the Order context emits OrderPlaced, the Billing context has no right to emit it.
- Events are immutable. Once emitted, they cannot change. (Immutability prevents handlers from disagreeing about what happened.)
- Events are rich with intent. A good event name and its fields tell a story: what happened, to whom, when, why (via correlation ID).

```cpp
// This is a domain event: past tense, self-contained, immutable
class OrderPlaced final {
private:
    const uint64_t order_id_;
    const uint64_t customer_id_;
    const std::vector<LineItem> items_;  // Snapshot at time of placement
    const Money total_;
    const std::chrono::system_clock::time_point emitted_at_;
    const std::string correlation_id_;  // Link to originating request
    
public:
    OrderPlaced(uint64_t oid, uint64_t cid,
                const std::vector<LineItem>& items, const Money& total,
                const std::string& corr_id)
        : order_id_(oid), customer_id_(cid), items_(items), total_(total),
          emitted_at_(std::chrono::system_clock::now()), correlation_id_(corr_id) {}
    
    // Only accessors; no setters
    uint64_t orderId() const { return order_id_; }
    uint64_t customerId() const { return customer_id_; }
    const std::vector<LineItem>& items() const { return items_; }
    Money total() const { return total_; }
    std::string correlationId() const { return correlation_id_; }
};
```

The event captures the moment of truth. If handlers process this event days later (in an asynchronous system), they're working from a snapshot of the facts at the time OrderPlaced was emitted, not the current state.

---

## 73.3 In-Process Synchronous Bus

The simplest event bus runs in one process, on one thread, and handlers run inline.

### How It Works

```
Emitter (calls publish)
    ↓
Event bus (receives event)
    ↓
Handler 1 runs (blocks)
    ↓
Handler 2 runs (blocks)
    ↓
Handler 3 runs (blocks)
    ↓
Control returns to emitter
```

The emitter publishes an event. The bus iterates its handlers, calls each one, and waits for all to finish. Then the emitter continues.

### Implementation

The tricky part is **type erasure**. Events can be any type; the bus doesn't know them all at compile time. We need to:

1. Store handlers for different event types.
2. Call the right handler when an event of that type is published.

Here's a type-erased in-process bus in ~80 lines of C++20:

```cpp
// event_bus.h
#pragma once

#include <map>
#include <vector>
#include <functional>
#include <typeindex>
#include <memory>

class EventBus {
private:
    using HandlerFunc = std::function<void(const void*)>;
    using HandlerList = std::vector<HandlerFunc>;
    
    // Map from type_index to list of handlers for that type
    std::map<std::type_index, HandlerList> handlers_;
    
public:
    // Register a handler for events of type T
    template <typename T>
    void subscribe(std::function<void(const T&)> handler) {
        auto type_key = std::type_index(typeid(T));
        
        // Wrap the handler: convert void* back to T&, then call
        HandlerFunc erased = [handler](const void* event_ptr) {
            const T& event = *static_cast<const T*>(event_ptr);
            handler(event);
        };
        
        handlers_[type_key].push_back(erased);
    }
    
    // Publish an event of type T
    template <typename T>
    void publish(const T& event) {
        auto type_key = std::type_index(typeid(T));
        
        auto it = handlers_.find(type_key);
        if (it == handlers_.end()) {
            return;  // No handlers for this event type
        }
        
        // Call each handler in order
        for (auto& handler : it->second) {
            handler(&event);
        }
    }
};
```

### Using It

```cpp
// Define events
struct OrderPlaced {
    uint64_t order_id;
    std::string customer_name;
};

struct PaymentProcessed {
    uint64_t order_id;
    double amount;
};

// Define handlers
void on_order_placed(const OrderPlaced& evt) {
    std::cout << "Billing: invoice customer " << evt.customer_name << "\n";
}

void on_order_placed_2(const OrderPlaced& evt) {
    std::cout << "Warehouse: pick items for order " << evt.order_id << "\n";
}

void on_payment_processed(const PaymentProcessed& evt) {
    std::cout << "Analytics: logged payment of " << evt.amount << "\n";
}

// Wire them up
int main() {
    EventBus bus;
    
    // Two handlers for OrderPlaced
    bus.subscribe<OrderPlaced>(on_order_placed);
    bus.subscribe<OrderPlaced>(on_order_placed_2);
    
    // One handler for PaymentProcessed
    bus.subscribe<PaymentProcessed>(on_payment_processed);
    
    // Emit an event
    OrderPlaced evt{42, "Alice"};
    bus.publish(evt);
    
    // Output (in order):
    // Billing: invoice customer Alice
    // Warehouse: pick items for order 42
    
    return 0;
}
```

### Pros and Cons

**Pros:**
- Simple. One call to publish; handlers run immediately.
- Transactional. If the emitter and handlers are in the same transaction (database), all-or-nothing semantics work: if any handler fails, the whole transaction rolls back.
- Debugging is easy. You can trace the call stack from emitter to all handlers.

**Cons:**
- Emitter is blocked by slowest handler. If one handler takes 5 seconds, the emitter waits 5 seconds.
- Failure cascades. If handler 1 throws, handlers 2 and 3 don't run (unless you wrap each in try-catch).
- No ordering guarantees across handlers. If handler 1 calls `publish()`, and that triggers a handler that also publishes, the interleaving is complex.

### Failure Handling

You need to decide: do all handlers run even if one fails?

```cpp
template <typename T>
void publish(const T& event) {
    auto type_key = std::type_index(typeid(T));
    auto it = handlers_.find(type_key);
    if (it == handlers_.end()) return;
    
    std::vector<std::exception_ptr> errors;
    
    for (auto& handler : it->second) {
        try {
            handler(&event);
        } catch (...) {
            // Collect error; continue to next handler
            errors.push_back(std::current_exception());
        }
    }
    
    // Decide: log errors? re-throw the first? re-throw all?
    if (!errors.empty()) {
        // Option 1: log and continue (fire-and-forget semantics)
        for (auto& err : errors) {
            // log(err)
        }
        
        // Option 2: re-throw first error (fail-fast)
        // std::rethrow_exception(errors.front());
    }
}
```

---

## 73.3 In-Process Asynchronous Bus

The synchronous bus blocks the emitter. In a concurrent system, that's often too slow. An asynchronous bus decouples timing: the emitter publishes, the event goes on a queue, and handlers pull and process it later.

### How It Works

```
Emitter (calls publish)
    ↓
Event goes on a queue
    ↓
Emitter returns immediately
    ↓
[separate thread/task]
Handler 1 processes event
    ↓
Handler 2 processes event
```

### Implementation

```cpp
// async_event_bus.h
#pragma once

#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <memory>
#include <typeindex>
#include <map>

class AsyncEventBus {
private:
    using HandlerFunc = std::function<void(const void*)>;
    using HandlerList = std::vector<HandlerFunc>;
    
    // Event wrapper: type info + pointer
    struct EnqueuedEvent {
        std::type_index type;
        std::shared_ptr<void> data;  // Heap-allocated event data
    };
    
    std::queue<EnqueuedEvent> event_queue_;
    std::map<std::type_index, HandlerList> handlers_;
    
    std::thread worker_thread_;
    std::mutex queue_mutex_;
    std::condition_variable queue_cv_;
    bool running_ = false;
    
    void worker_loop() {
        while (running_) {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            
            // Wait for event or shutdown signal
            queue_cv_.wait(lock, [this] {
                return !event_queue_.empty() || !running_;
            });
            
            if (!running_ && event_queue_.empty()) {
                break;
            }
            
            if (!event_queue_.empty()) {
                EnqueuedEvent enqueued = event_queue_.front();
                event_queue_.pop();
                lock.unlock();  // Release lock while processing
                
                // Process the event
                auto it = handlers_.find(enqueued.type);
                if (it != handlers_.end()) {
                    for (auto& handler : it->second) {
                        try {
                            handler(enqueued.data.get());
                        } catch (...) {
                            // Log error, continue
                        }
                    }
                }
            }
        }
    }
    
public:
    AsyncEventBus() : running_(true) {
        worker_thread_ = std::thread([this] { worker_loop(); });
    }
    
    ~AsyncEventBus() {
        shutdown();
    }
    
    void shutdown() {
        if (!running_) return;
        
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            running_ = false;
        }
        queue_cv_.notify_one();
        
        if (worker_thread_.joinable()) {
            worker_thread_.join();
        }
    }
    
    template <typename T>
    void subscribe(std::function<void(const T&)> handler) {
        auto type_key = std::type_index(typeid(T));
        
        HandlerFunc erased = [handler](const void* event_ptr) {
            const T& event = *static_cast<const T*>(event_ptr);
            handler(event);
        };
        
        handlers_[type_key].push_back(erased);
    }
    
    template <typename T>
    void publish(const T& event) {
        // Heap-allocate the event
        auto event_copy = std::make_shared<T>(event);
        
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            event_queue_.push({
                std::type_index(typeid(T)),
                std::static_pointer_cast<void>(event_copy)
            });
        }
        queue_cv_.notify_one();
    }
};
```

### Usage

```cpp
int main() {
    AsyncEventBus bus;
    
    bus.subscribe<OrderPlaced>([](const OrderPlaced& evt) {
        std::cout << "Billing (async): processing order " << evt.order_id << "\n";
        // Simulate slow processing
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    });
    
    bus.subscribe<OrderPlaced>([](const OrderPlaced& evt) {
        std::cout << "Warehouse (async): preparing shipment " << evt.order_id << "\n";
    });
    
    // Emit events
    OrderPlaced evt1{42, "Alice"};
    bus.publish(evt1);
    
    OrderPlaced evt2{43, "Bob"};
    bus.publish(evt2);
    
    // Main thread continues immediately (doesn't wait for handlers)
    std::cout << "Events published. Main thread continuing...\n";
    
    // Wait for handlers to finish before exiting
    std::this_thread::sleep_for(std::chrono::milliseconds(300));
    bus.shutdown();
    
    return 0;
}
```

### Pros and Cons

**Pros:**
- Emitter is never blocked. `publish()` returns immediately.
- Decoupled timing. Handlers run when the worker thread is ready, not when the emitter calls `publish()`.
- Can handle bursts. Events queue up; the worker processes them at its own pace.

**Cons:**
- No ordering guarantees across event types. If emitter publishes OrderPlaced, then PaymentProcessed, the PaymentProcessed handler might run before the OrderPlaced handler finishes.
- Handlers must be thread-safe if they access shared state.
- Failure handling is opaque. If a handler fails, the emitter doesn't know (and can't recover).
- At-least-once or at-most-once delivery? If the worker thread crashes mid-handler, the event is lost. If it crashes after dequeuing but before finishing, the same event might run twice on restart.

---

## 73.4 Cross-Process Event Brokers

In distributed systems, a single-process bus isn't enough. Services need to exchange events over the network. This is where brokers come in: Kafka, RabbitMQ, NATS, Redis Streams, and others.

### How a Broker Works

```
Service A                    Broker                  Service B
(emitter)                   (durable queue)         (listener)

publish(OrderPlaced) -------→ [OrderPlaced] -------→ subscribe(OrderPlaced)
                                  queue               process event
                            (persisted to disk)       ack
```

The broker is a message queue. Service A publishes; the broker stores the message durably (on disk). Service B subscribes; the broker delivers the message. When B acknowledges, the broker can delete the message.

### Why It Matters

A broker introduces new capabilities and new problems:

**Capabilities:**
- **Durability.** If Service A crashes after publishing, the message isn't lost; it's in the broker's queue.
- **Replay.** If Service B wants to reprocess all OrderPlaced events, the broker can replay them from the log.
- **Decoupling.** Service A doesn't need to know where Service B runs. Both just talk to the broker.
- **Scaling.** Multiple instances of Service B can consume from the same topic, load-balanced by the broker.

**Problems:**
- **Network latency.** In-process calls are microseconds; network calls are milliseconds. 100x slower.
- **Partial failure.** Service A publishes, but the network drops, and the broker never receives it. How do you retry? How do you detect it?
- **Operational complexity.** The broker must be deployed, monitored, scaled. It's another system that can fail.

### Kafka's Guarantees

Kafka partitions topics by key. Events with the same key always go to the same partition, so they are processed in order. This solves the ordering problem:

```
Topic: OrderEvents
Partition 0 (customer_id % 2 == 0): [OrderPlaced(cid=2), OrderShipped(cid=2), ...]
Partition 1 (customer_id % 2 == 1): [OrderPlaced(cid=1), OrderCanceled(cid=1), ...]

Consumers:
- Consumer A subscribes to Partition 0
- Consumer B subscribes to Partition 1
- Both process events in order, independently
```

This per-key ordering is powerful for domain events. All events about customer 1's orders arrive in the sequence they were emitted.

---

## 73.5 Delivery Semantics

When an event is emitted and a handler processes it, what guarantees does the system make?

### At-Most-Once

"The handler runs 0 or 1 times. It might not run at all."

**When:** The handler is best-effort. If the system crashes, the event is gone.

**Example:** Logging a user's action to analytics. If we lose one event, it doesn't matter.

**Implementation:** Dequeue, run handler, if it crashes the event is lost.

```cpp
// At-most-once: dequeue → execute → done (no retry)
EnqueuedEvent evt = event_queue_.front();
event_queue_.pop();
handlers[evt.type](evt.data.get());  // If this crashes, event is lost
```

### At-Least-Once

"The handler runs 1 or more times. It will eventually run, even if the system crashes."

**When:** The event is important. Losing it would corrupt state.

**Example:** A payment must be processed. If the system crashes after dequeuing but before completing, the payment must run again on restart.

**Implementation:** Keep the event on durable storage (database or log) until the handler acknowledges completion.

```cpp
// At-least-once: log → dequeue → execute → ack
database.log_event(evt);  // Persist to durable storage
handlers[evt.type](evt.data.get());
database.mark_event_processed(evt.id);  // Only after handler succeeds
// If we crash between execute and ack, the event will be replayed on restart
```

**The cost:** The handler must be **idempotent**. If the same event is processed twice, the result is the same as if it were processed once.

```cpp
// Idempotent handler: safe to run multiple times
void process_payment(const PaymentProcessed& evt) {
    // Check if already processed
    if (database.payment_processed(evt.payment_id)) {
        return;  // Already done; idempotency
    }
    
    // Mark as processed (before doing anything else)
    database.mark_payment_processed(evt.payment_id);
    
    // Then do the work
    update_customer_balance(evt.customer_id, evt.amount);
}
```

### Exactly-Once

"The handler runs exactly once, no more, no less, even if the system crashes."

**Reality:** Exactly-once is a fairy tale in distributed systems.

Why? Because there are always two failure modes:

1. **Handler runs, then system crashes before acknowledging.** On restart, did the handler run or not? You don't know. You replay the event. Handler runs twice. You now have at-least-once, not exactly-once.

2. **Handler runs, system crashes while acknowledging.** Same problem.

The only way to guarantee exactly-once is to have a system-wide failure detector and strong consistency guarantees. This is expensive, and most applications don't need it.

**Practical answer:** Use at-least-once delivery and idempotent handlers. This gives you the same effect as exactly-once with much less complexity.

---

## 73.5a The Outbox Pattern: Having Your Cake

One more option: the **outbox pattern** or **transactional outbox**. Instead of publishing to a broker, you write events to a local database table (the outbox). A separate process (a polling task) reads from the outbox and publishes to the broker.

```cpp
// In the same transaction as the domain change
void place_order() {
    Order order = create_order(...);
    order.place();
    
    // Save order to orders table
    database.save(order);
    
    // Save event to outbox table (same transaction)
    database.save_to_outbox(OrderPlaced{
        order.id(), order.customer_id(), order.total(), ...
    });
    
    // Both commits or both rolls back
    transaction.commit();
}

// Separate process, polling the outbox
class OutboxPoller {
public:
    void poll() {
        while (true) {
            auto events = database.read_unpublished_outbox_events(limit=100);
            
            for (const auto& evt : events) {
                try {
                    broker.publish(evt);  // Publish to Kafka, RabbitMQ, etc.
                    database.mark_outbox_published(evt.id());
                } catch (...) {
                    // Retry on next poll; don't mark as published
                    continue;
                }
            }
            
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }
};
```

**Advantages:**
- Events are transactional with the domain change. No risk of saving the order but losing the event.
- At-least-once semantics are guaranteed (the outbox table is durable).
- No event loss; if the poller crashes mid-publish, the event is still in the outbox.

**Disadvantages:**
- Polling latency. Events aren't published instantly; they wait for the next poll.
- Requires database schema changes (add outbox table).
- The poller is another service to run and monitor.

This pattern is popular in microservices because it combines transactional consistency with eventual consistency.

---

## 73.6 Ordering Guarantees

When multiple events are published, in what order do handlers see them?

### Per-Key Ordering (Kafka Partition Model)

"Events with the same key always arrive in the order they were emitted. Events with different keys have no guaranteed order."

**Example:**

```
Emit: OrderPlaced(order_id=1, key="customer_1")
Emit: OrderShipped(order_id=1, key="customer_1")
Emit: PaymentFailed(order_id=2, key="customer_2")

Handler sees (guaranteed order):
  [customer_1]: OrderPlaced, OrderShipped
  [customer_2]: PaymentFailed (might run before or after customer_1 events)
```

This is useful for domain events. All events for the same aggregate (same customer, same order) stay in order. Events for different aggregates can run in parallel.

**Implementation:** Use a queue per key.

```cpp
class KeyPartitionedBus {
private:
    std::map<std::string, std::queue<EnqueuedEvent>> key_to_queue;
    std::map<std::string, std::thread> key_to_worker;
    
    // Derive key from event (event type must have a key() method)
    template <typename T>
    std::string key_for(const T& evt) {
        return evt.key();  // or evt.aggregate_id, or whatever
    }
};
```

### Per-Emitter Ordering

"Events from the same emitter (source) are ordered. Events from different emitters have no guaranteed order."

Less common. Useful if you care about the timeline of actions from one actor.

### No Ordering

"Events can arrive in any order."

This is the default for many distributed systems. Handlers must be designed to handle arrival in any order.

When there is no ordering guarantee, your handlers must be **commutative** for concurrent events (or at least safe). Example: if PaymentProcessed and OrderCanceled can arrive in either order, both must leave the system in a consistent state.

```cpp
// Problematic: order matters
void on_payment_processed(const PaymentProcessed& evt) {
    order.mark_paid();
    if (order.is_shipped()) {
        // Invoice the customer (they paid, order shipped, so they're done)
    }
}

void on_order_canceled(const OrderCanceled& evt) {
    order.cancel();
    if (order.is_paid()) {
        // Issue refund
    }
}

// If PaymentProcessed arrives, then OrderCanceled: refund issued. Correct.
// If OrderCanceled arrives, then PaymentProcessed: invoice issued. Wrong! Order canceled.

// Fix: add a version number or timestamp to resolve conflicts
void on_payment_processed(const PaymentProcessed& evt) {
    if (order.status() == OrderStatus::Canceled) {
        // Order was already canceled; refund the payment
        refund(evt.amount);
        return;
    }
    order.mark_paid();
}
```

---

## 73.7 Subscriptions

How are handlers registered? Static or dynamic? Do they persist across restarts?

### Static Subscriptions (Compile-Time)

Handlers are wired up in code at startup.

```cpp
int main() {
    EventBus bus;
    
    // Handlers are registered before events are emitted
    bus.subscribe<OrderPlaced>(on_order_placed);
    bus.subscribe<OrderPlaced>(on_inventory_reserved);
    
    // ... process events ...
}
```

**Pros:** Simple. Fast (no dynamic lookup). Clear dependency graph (readers can see all handlers).

**Cons:** Adding a new handler requires code changes.

### Dynamic Subscriptions (Runtime)

Handlers are registered while the system is running.

```cpp
class EventBus {
public:
    uint64_t subscribe(std::type_index type, HandlerFunc handler) {
        uint64_t handler_id = next_id++;
        handlers_[type].push_back({handler_id, handler});
        return handler_id;
    }
    
    void unsubscribe(std::type_index type, uint64_t handler_id) {
        auto& list = handlers_[type];
        // Remove handler with this ID
    }
};

// Usage
uint64_t handle_id = bus.subscribe(typeid(OrderPlaced), on_order_placed);
// ... later ...
bus.unsubscribe(typeid(OrderPlaced), handle_id);
```

**Pros:** Can add handlers without restarting.

**Cons:** Harder to trace dependencies (where is the handler registered?). Can cause memory leaks if unsubscribe is forgotten.

### Persistent Subscriptions (Stored)

Subscriptions are stored in a database and loaded at startup. Useful for workflows or plugins.

```cpp
// At startup, load subscriptions from database
auto subscriptions = database.load_subscriptions();
for (const auto& sub : subscriptions) {
    EventBus::subscribe(sub.event_type, load_handler(sub.handler_class));
}
```

**Pros:** Subscriptions survive restarts. Can be configured without code changes.

**Cons:** More complex to implement. Must handle versioning if handler code changes.

---

## 73.8 Worked Example: A Small Event Bus

Let's build a complete in-process synchronous event bus with good error handling:

```cpp
// event_bus.h
#pragma once

#include <map>
#include <vector>
#include <functional>
#include <typeindex>
#include <stdexcept>

class EventBusException : public std::runtime_error {
public:
    explicit EventBusException(const std::string& msg)
        : std::runtime_error(msg) {}
};

class EventBus {
private:
    using HandlerFunc = std::function<void(const void*)>;
    
    struct HandlerEntry {
        uint64_t id;
        HandlerFunc func;
    };
    
    std::map<std::type_index, std::vector<HandlerEntry>> handlers_;
    uint64_t next_handler_id_ = 0;
    
public:
    // Subscribe: returns an ID to unsubscribe later
    template <typename T>
    uint64_t subscribe(std::function<void(const T&)> handler) {
        if (!handler) {
            throw EventBusException("Handler cannot be null");
        }
        
        auto type_key = std::type_index(typeid(T));
        uint64_t handler_id = next_handler_id_++;
        
        HandlerFunc erased = [handler](const void* event_ptr) {
            const T& event = *static_cast<const T*>(event_ptr);
            handler(event);
        };
        
        handlers_[type_key].push_back({handler_id, erased});
        return handler_id;
    }
    
    // Unsubscribe
    template <typename T>
    bool unsubscribe(uint64_t handler_id) {
        auto type_key = std::type_index(typeid(T));
        auto it = handlers_.find(type_key);
        if (it == handlers_.end()) {
            return false;
        }
        
        auto& list = it->second;
        auto pos = std::find_if(list.begin(), list.end(),
            [handler_id](const HandlerEntry& e) { return e.id == handler_id; });
        
        if (pos == list.end()) {
            return false;
        }
        
        list.erase(pos);
        return true;
    }
    
    // Publish with error collection
    template <typename T>
    std::vector<std::exception_ptr> publish(const T& event) {
        auto type_key = std::type_index(typeid(T));
        auto it = handlers_.find(type_key);
        
        std::vector<std::exception_ptr> errors;
        
        if (it == handlers_.end()) {
            return errors;  // No handlers; no errors
        }
        
        for (auto& entry : it->second) {
            try {
                entry.func(&event);
            } catch (...) {
                errors.push_back(std::current_exception());
            }
        }
        
        return errors;
    }
    
    // Publish with fail-fast
    template <typename T>
    void publish_strict(const T& event) {
        auto errors = publish(event);
        if (!errors.empty()) {
            std::rethrow_exception(errors.front());
        }
    }
    
    // Handler count (for testing)
    template <typename T>
    size_t handler_count() const {
        auto type_key = std::type_index(typeid(T));
        auto it = handlers_.find(type_key);
        return it == handlers_.end() ? 0 : it->second.size();
    }
};
```

### Usage

```cpp
#include "event_bus.h"
#include <iostream>

struct OrderPlaced {
    uint64_t order_id;
    std::string customer_name;
};

int main() {
    EventBus bus;
    
    // Subscribe multiple handlers
    auto h1 = bus.subscribe<OrderPlaced>([](const OrderPlaced& evt) {
        std::cout << "Billing: invoice " << evt.customer_name << "\n";
    });
    
    auto h2 = bus.subscribe<OrderPlaced>([](const OrderPlaced& evt) {
        std::cout << "Warehouse: ship order " << evt.order_id << "\n";
    });
    
    auto h3 = bus.subscribe<OrderPlaced>([](const OrderPlaced& evt) {
        throw std::runtime_error("Warehouse is down!");
    });
    
    // Publish with error collection
    OrderPlaced evt{42, "Alice"};
    auto errors = bus.publish(evt);
    
    std::cout << "Errors: " << errors.size() << "\n";
    // Output:
    // Billing: invoice Alice
    // Warehouse: ship order 42
    // Errors: 1
    
    // Unsubscribe the failing handler
    bus.unsubscribe<OrderPlaced>(h3);
    
    // Now all handlers run
    errors = bus.publish(evt);
    std::cout << "After fix - Errors: " << errors.size() << "\n";
    // Output:
    // Billing: invoice Alice
    // Warehouse: ship order 42
    // After fix - Errors: 0
    
    return 0;
}
```

---

## 73.8a Testing Events and Handlers

Events and handlers need testing, but they can be tricky to test in isolation.

**Testing a handler directly:**

```cpp
#include <gtest/gtest.h>

class BillingServiceTest : public ::testing::Test {
protected:
    MockDatabase db;
    BillingService billing{db};
};

TEST_F(BillingServiceTest, HandleOrderPlaced) {
    OrderPlaced evt{
        .order_id = 42,
        .customer_id = 100,
        .total = Money(99.99, "USD"),
        .emitted_at = std::chrono::system_clock::now(),
        .correlation_id = "req-123"
    };
    
    EXPECT_CALL(db, invoice)
        .With(customer_id = 100, amount = 99.99)
        .Times(1);
    
    billing.on_order_placed(evt);
    
    EXPECT_THAT(billing.last_correlation_id(), Eq("req-123"));
}
```

**Testing the bus itself:**

```cpp
TEST(EventBusTest, PublishesToAllHandlers) {
    EventBus bus;
    
    int handler1_called = 0, handler2_called = 0;
    
    bus.subscribe<OrderPlaced>([&](const OrderPlaced& evt) {
        handler1_called++;
    });
    
    bus.subscribe<OrderPlaced>([&](const OrderPlaced& evt) {
        handler2_called++;
    });
    
    OrderPlaced evt{42, 100, Money(99.99, "USD"), {}, "req-123"};
    bus.publish(evt);
    
    EXPECT_EQ(handler1_called, 1);
    EXPECT_EQ(handler2_called, 1);
}

TEST(EventBusTest, UnsubscribeStopsHandling) {
    EventBus bus;
    
    int called = 0;
    auto handler_id = bus.subscribe<OrderPlaced>([&](const OrderPlaced&) {
        called++;
    });
    
    OrderPlaced evt{42, 100, Money(99.99, "USD"), {}, "req-123"};
    bus.publish(evt);
    EXPECT_EQ(called, 1);
    
    bus.unsubscribe<OrderPlaced>(handler_id);
    bus.publish(evt);
    EXPECT_EQ(called, 1);  // Not incremented
}
```

**Testing handler order and failures:**

```cpp
TEST(EventBusTest, HandlersRunInSubscriptionOrder) {
    EventBus bus;
    std::vector<std::string> execution_order;
    
    bus.subscribe<OrderPlaced>([&](const OrderPlaced&) {
        execution_order.push_back("first");
    });
    
    bus.subscribe<OrderPlaced>([&](const OrderPlaced&) {
        execution_order.push_back("second");
    });
    
    OrderPlaced evt{42, 100, Money(99.99, "USD"), {}, "req-123"};
    auto errors = bus.publish(evt);
    
    EXPECT_THAT(execution_order, ElementsAre("first", "second"));
    EXPECT_EQ(errors.size(), 0);
}

TEST(EventBusTest, ErrorsAreCollected) {
    EventBus bus;
    int handler2_called = 0;
    
    bus.subscribe<OrderPlaced>([](const OrderPlaced&) {
        throw std::runtime_error("Handler 1 failed");
    });
    
    bus.subscribe<OrderPlaced>([&](const OrderPlaced&) {
        handler2_called++;  // Should still run
    });
    
    OrderPlaced evt{42, 100, Money(99.99, "USD"), {}, "req-123"};
    auto errors = bus.publish(evt);
    
    EXPECT_EQ(errors.size(), 1);
    EXPECT_EQ(handler2_called, 1);  // Second handler still ran
}
```

The key principle: **handlers are side-effect functions, so mock or spy on the side effects.** Don't test the bus's internals; test that when you publish, the right thing happens.

---

## 73.9 When NOT To Use An Event Bus

Event buses are flexible, but they are not always the right tool.

**Don't use an event bus when:**

1. **The relationship is logically synchronous.** "Validate the order, then save it." These are two steps of one operation, not two independent reactions. Use a function call.

   ```cpp
   // Right: synchronous function
   bool save_order(const Order& order) {
       if (!validate(order)) {
           return false;
       }
       database.save(order);
       return true;
   }
   
   // Wrong: async event bus
   Order order = create_order(customer_id);
   bus.publish(OrderCreated{order});  // Validation happens asynchronously? Confusing.
   ```

2. **You need a return value.** If the caller needs to know the result immediately, use a function call, not an event.

   ```cpp
   // Right
   ProcessResult result = process_payment(payment_info);
   if (result.success) { ... }
   
   // Wrong
   bus.publish(PaymentRequested{payment_info});
   // How do we know if it succeeded?
   ```

3. **The listeners are tightly coupled to the emitter.** If there's only one listener, and you wrote both, you're adding complexity for no gain. Call it directly.

   ```cpp
   // If there's only one place that cares about OrderPlaced,
   // and you control both the Order and that place:
   
   // Right: direct call
   order.place();
   billing_service.invoice(order);
   
   // Overkill: event bus
   bus.publish(OrderPlaced{...});
   ```

4. **The relationship is one-to-one and the caller needs to know it happened.** An event bus is for one-to-many or zero-to-many relationships.

---

## 73.10 Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **Synchronous bus** | Simple; transactional; debuggable; fast. | Emitter blocked by slowest handler; failure cascades; no concurrency. | Small systems; bounded contexts in the same codebase; handlers are fast. |
| **Asynchronous (in-process)** | Emitter not blocked; handles bursts; scales to more handlers. | Complex failure handling; at-least-once or at-most-once semantics; threads needed. | Medium systems; many handlers; handlers are slow. |
| **Broker (Kafka/RabbitMQ)** | Durable; distributes load; scales to many services; replayable. | Network latency; operational complexity; potential for cascading failures if broker goes down. | Large systems; cross-service communication; event replay needed. |
| **Outbox + polling** | Transactional; no message loss; can be replayed. | Polling overhead; not real-time; requires database changes. | Strict consistency needed; legacy system integration. |

---

## 73.11 Common Misconceptions

**"Event buses always make code more decoupled."**

False. An event bus can hide tight coupling instead of removing it. If you emit OrderPlaced and three handlers form an invisible dependency chain (billing must run before inventory, inventory before shipping), you have coupling—just harder to see. Document the dependencies.

**"All events should be published."**

No. Publish events that are part of the domain model and that other bounded contexts might care about. Don't publish every internal state change. OrderPlaced is an event. "Running validation" is not.

**"Events must be persisted."**

Not always. In-process synchronous buses don't persist; events exist only during handling. This is fine for single-process systems or tightly coupled services. Persistence is useful for durability and replay, but it adds complexity.

**"Event handlers should never fail."**

They will. The question is: what do you do when they do? Decide on a policy: fail-fast, collect errors, retry, log and continue. Don't ignore it.

**"Event buses are for eventual consistency only."**

Not true. A synchronous event bus is transactional and supports immediate consistency. It's only when you go async or cross-process that eventual consistency becomes relevant.

---

## 73.12 Exercises

1. **Implement a simple synchronous bus** (from scratch, not copying). Publish three different event types. Write two handlers for one type, one for another. Verify all handlers run in order.

2. **Add error handling.** Modify the bus so that if one handler throws, the others still run. Collect the errors and return them to the caller. How would you log them?

3. **Design idempotent handlers.** Write a handler that processes a PaymentProcessed event. Make it idempotent: if the same payment is processed twice, the state is correct. (Hint: check if already processed before updating.)

4. **Per-key ordering.** Sketch a bus that guarantees per-key ordering. How would you extract the key from an event? How would you partition handlers?

5. **Compare sync vs async.** Build both a synchronous and asynchronous bus. Emit 100 events to each. Time both. Which is faster for small events? Large events? Slow handlers?

6. **When not to use events.** Write a domain model that emits too many events (internal state changes, validation checks, etc.). Then redesign it to emit only domain events. What changed?

7. **Handling network failures in a broker.** Sketch a service that publishes OrderPlaced to Kafka. The publish succeeds (broker confirms), but the service crashes before returning to the caller. What happens to: (a) the order (in the database), (b) the event (in Kafka)? Is this the desired outcome?

8. **Idempotent handlers in detail.** Write a payment handler that is idempotent. It should:
   - Check if the payment was already processed (using payment ID).
   - If yes, return early (don't double-charge).
   - If no, charge the customer, then mark as processed.
   - Ensure the check and mark are atomic (hint: database with a unique constraint on payment ID).

---

## 73.13 Summary

An event bus decouples emitters from listeners by turning direct function calls into message passing. The simplest form—a synchronous in-process bus—is straightforward to implement using type erasure and a map from type to handlers. Asynchronous buses add a queue and a worker thread, giving better concurrency at the cost of more complex failure semantics. Delivery semantics matter: at-most-once (fire-and-forget), at-least-once (idempotent handlers), or exactly-once (rare and expensive). Event buses shine for one-to-many relationships and decoupled bounded contexts. They are wrong when the relationship is logically synchronous, when a return value is needed, or when a direct call is clearer. Know the tradeoff: the cleanliness of loose coupling against the opacity of hidden dependencies.

---

> **[← Previous: A DI Container](05-a-di-container.md)** · **[↑ Part 7](README.md)** · **[Next: Plugin Architecture →](07-plugin-architecture.md)**
