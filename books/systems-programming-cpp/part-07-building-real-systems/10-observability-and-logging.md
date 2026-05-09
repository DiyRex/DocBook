# Chapter 77 — Observability and Logging

When production breaks at 3 AM, you don't get to attach a debugger. The system has already moved on, processing the next 10,000 requests. Your debugger sits unused. What you have instead is data — a record of what the system was doing when things went wrong. That data is observability. The system has to *tell you* what it was doing, in enough detail that you can reconstruct the failure without reproducing it.

Observability is not logging. It is not metrics. It is not traces. It is all three, working together, creating a picture of the system's behavior. One signal tells you what happened (logs). One tells you how often and at what scale (metrics). One connects the dots across services (traces). Separately, each is useful. Together, they are powerful enough to resolve incidents that defy reproduction.

This chapter teaches the three signals, how to emit them from C++ code, how to correlate them, and the tradeoffs involved. By the end, you will understand why observability matters more in production systems than in development, and how to design your system to *tell* you what it is doing.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish between logs, metrics, and traces — what each measures and when to use each signal.
2. Implement structured logging (JSON over free-text) and understand why it scales better than printf-style logs.
3. Apply log levels correctly: DEBUG, INFO, WARN, ERROR, FATAL — and know why "log everything at DEBUG and grep" is an anti-pattern in production.
4. Design and emit Prometheus metrics (counters, gauges, histograms) and understand cardinality cost — why user_id as a label blows up your time series database.
5. Implement OpenTelemetry tracing: spans, parent-child relationships, sampling strategies.
6. Correlate logs, metrics, and traces through trace IDs — the practice that turns three independent signals into a coherent story.
7. Apply SLI/SLO thinking to observability — define what "good" means before production breaks.
8. Use C++ logging libraries (spdlog, glog) correctly: async logging, sinks, performance.
9. Recognize the tradeoffs in observability design — cost vs. detail, latency vs. accuracy, retention vs. storage.
10. Spot common misconceptions about observability and understand the reality.

---

## 77.1 The Three Signals

Observability rests on three pillars. Each answers a different question. Together, they let you understand nearly any failure.

### Logs: "What Happened?"

A log is a discrete event. When something notable occurs — a request arrived, a calculation succeeded, an error was thrown — you emit a log line. The log captures context: timestamps, identifiers, error messages, relevant state.

Logs are *discrete* — each log line represents one event. Logs are *contextual* — they include the information needed to understand that event in isolation.

Example:

```json
{
  "timestamp": "2025-05-09T14:23:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "request_id": "req-12345",
  "message": "Payment processing failed",
  "event": "payment_failed",
  "reason": "insufficient_funds",
  "user_id": "usr-789",
  "amount_cents": 5000,
  "error": "InsufficientFunds"
}
```

This log line tells you:
- When it happened (timestamp).
- How severe (ERROR level).
- Where (which service).
- How to trace it (trace_id, request_id).
- What happened (payment processing failed, insufficient funds).
- The data involved (user, amount).

Logs are human-readable and machine-queryable. A human can read the message; a machine can parse JSON and search by trace_id or user_id.

### Metrics: "How Often? At What Scale?"

A metric is a numeric time series. Instead of "what happened once," it answers "how many times did this happen per second, and what is the distribution?"

Four metric types cover most needs:

**Counter** — a number that only increases. Example: total requests processed. You plot it over time and see throughput.

```
requests_total{service="api"} = 1,000,000
```

One hour later:

```
requests_total{service="api"} = 1,001,500
```

You processed 1,500 requests in that hour. Counters measure rate (requests per second), rate of change (is throughput growing or shrinking?).

**Gauge** — a number that can go up or down. Example: active connections, queue depth, memory usage. A gauge tells you the *current state* at a moment in time.

```
active_connections{service="api"} = 245
memory_usage_bytes{service="api"} = 512000000
```

**Histogram** — a distribution of values across buckets. Example: request latency. Instead of "average latency was 50ms," a histogram tells you "50% of requests were under 20ms, 99% under 100ms, 99.9% under 500ms."

```
request_duration_seconds_bucket{le="0.01"}  = 5000   # 5000 requests under 10ms
request_duration_seconds_bucket{le="0.05"}  = 45000  # 45000 requests under 50ms
request_duration_seconds_bucket{le="0.1"}   = 49500  # 49500 requests under 100ms
request_duration_seconds_bucket{le="+Inf"}  = 50000  # All requests
```

From this, you compute percentiles. P99 latency is the value where 99% of requests fall below. Histograms are powerful because they capture distribution, not just average.

**Summary** — similar to histogram but computed client-side. Useful when you cannot afford the bucket overhead of histograms. (Histograms add one time series per bucket; summaries add one per quantile.)

Metrics are numbers. They compress — a counter that goes from 1 million to 1,000,000 can be represented as "grew by 1.5K per hour" in aggregated storage. Logs, by contrast, are raw events; they don't compress as easily.

Metrics answer questions like "is latency getting worse?" and "how many errors per minute?" Logs answer "what caused that error?"

### Traces: "How Did This Request Flow Through Services?"

A trace is a causally linked sequence of spans across services. A span represents one operation — a database query, an HTTP call, a cache lookup. Spans have a parent-child relationship, forming a tree.

Example trace for one user request hitting an e-commerce site:

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

Span 1 (API Service): handle_request           [0ms - 145ms]
  └─ Span 2 (Auth Service): validate_token    [10ms - 25ms]
  └─ Span 3 (Inventory Service): check_stock  [26ms - 95ms]
     └─ Span 3.1 (Database): SELECT inventory [35ms - 92ms]
  └─ Span 4 (Payment Service): charge         [96ms - 140ms]
     └─ Span 4.1 (External API): stripe_api   [105ms - 138ms]
```

Each span has:
- **Name** — what operation (validate_token, check_stock).
- **Timestamps** — when it started and ended.
- **Duration** — how long it took (calculated from start/end).
- **Status** — success or failure.
- **Attributes** — key-value context (user_id, status_code, cache_hit).
- **Events** — noteworthy occurrences during the span (cache miss, retry).
- **Links** — references to other traces (async operations, cross-process calls).

A trace tells you the complete path a request took. You can see that the total request took 145ms, but the inventory check took 70ms of that. If you optimize the inventory check, the whole request gets faster. This is information logs alone cannot give you — logs tell you what happened at each service, but only traces show you the causality and timing across services.

---

## 77.2 Structured Logging

Text logs are easy to generate but hard to query at scale. Free-text logs like:

```
2025-05-09 14:23:45 ERROR Payment processing failed for user 789: insufficient funds
```

are human-readable, but a log query tool (like grep or ELK) must parse them with heuristics. What if the message format varies slightly? What if you want to find all failed payments for a specific user?

Structured logs (JSON) are queryable:

```json
{
  "timestamp": "2025-05-09T14:23:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "event": "payment_failed",
  "user_id": "usr-789",
  "amount_cents": 5000,
  "reason": "insufficient_funds"
}
```

A log aggregation system (like Datadog, Splunk, or Prometheus Loki) can parse this once and create indexed fields. You can then query: "all ERROR logs where event=payment_failed and user_id=usr-789" in milliseconds.

### Log Destinations

The 12-factor app principle: logs go to stdout. The infrastructure (container orchestrator, log aggregator) decides what to do with them — write to a file, send to a remote service, buffer in memory.

This decouples the application from the logging infrastructure:

```cpp
// BAD: Application decides where logs go
std::ofstream logFile("app.log", std::ios::app);
logFile << "something happened\n";
logFile.close();

// GOOD: Application writes to stdout; infrastructure redirects
std::cout << jsonLogLine << "\n";
std::cout << std::flush;  // Ensure it reaches the pipe
```

Reasons:

1. **Scalability** — if logs go to a local file, you need a separate file collector to send them to aggregation. If they go to stdout, the container orchestrator handles it.
2. **Simplicity** — the application doesn't need to know about syslog, file rotation, remote sinks. It writes; others read.
3. **Testability** — you can capture stdout in tests; you cannot easily test file I/O in containers.

In production, the infrastructure redirects stdout to:
- **Syslog** — for systems following Unix conventions.
- **Files with rotation** — for high-volume logs (Logrotate or similar).
- **Remote sinks** — log aggregation services (Datadog, Splunk, ELK).
- **Queues** — message brokers (Kafka) for asynchronous processing.

### Log Levels

Five levels cover almost everything:

| Level | When to Use | Example |
|-------|-------------|---------|
| DEBUG | Detailed information for troubleshooting | "User object created: {user_id: 123, email: 'a@b.com'}" |
| INFO | Important events that happen in normal operation | "Request processed successfully in 45ms" |
| WARN | Something unexpected but non-fatal | "Cache miss; falling back to database" |
| ERROR | An operation failed; error recovery is possible | "Payment failed: insufficient funds" |
| FATAL | The system cannot continue; it must exit | "Cannot bind to port 8080; exiting" |

**Use levels sparingly.** The default is INFO. DEBUG should be disabled in production (or at least not default to logging everything — see the anti-pattern below).

### The "Log Everything at DEBUG and Grep" Anti-Pattern

Some engineers believe the solution to missing observability is "log everything at DEBUG level, and when there's a problem, grep the logs."

This doesn't scale:

1. **Cost** — logging everything is expensive. You may log 100MB of DEBUG data per minute. Storage and ingestion costs grow quickly.
2. **Performance** — generating that many log lines uses CPU and I/O bandwidth that could serve user requests.
3. **Signal-to-noise** — 99.9% of logs are irrelevant to the failure you're investigating. Finding the signal requires manual analysis.
4. **Retention** — you cannot afford to keep every DEBUG log forever. You must delete old logs, but the one you needed was deleted last week.

The right approach: **Emit INFO-level logs for normal operation. When something is anomalous (high latency, error), automatically emit DEBUG-level context.** This is called *adaptive logging* or *debug on error.* Tools like Datadog and Splunk can do this automatically based on error triggers.

---

## 77.3 Metrics: Cardinality and Cost

Metrics are cheap to store compared to logs, but only if you control cardinality — the number of unique label combinations.

A metric has labels (dimensions):

```
http_requests_total{method="GET", status="200", endpoint="/api/users"}
http_requests_total{method="POST", status="400", endpoint="/api/users"}
```

Each unique combination of labels creates a new time series. If you have:
- 10 HTTP methods (GET, POST, PUT, DELETE, PATCH, etc.)
- 20 status codes (200, 201, 400, 401, 403, 404, 500, etc.)
- 50 endpoints

You get 10 × 20 × 50 = 10,000 time series from one metric. That's manageable.

But if you add a label like `user_id`:

```
http_requests_total{method="GET", status="200", endpoint="/api/users", user_id="usr-123"}
```

And you have 1 million users, you get 10 × 20 × 50 × 1,000,000 = 10 billion time series. Your time series database explodes. Storage, ingestion, and query latency all degrade.

**Rule: Never use high-cardinality fields (user_id, request_id, session_id) as metric labels.** Use them in logs instead.

Safe label values:

- **Status codes** — a fixed set.
- **Endpoints** — a fixed set (or sanitized — /api/users/:id becomes /api/users/{id}).
- **Methods** — GET, POST, PUT, DELETE (fixed set).
- **Service names** — api, database, cache (fixed set).
- **Environments** — prod, staging, dev (fixed set).
- **Error types** — "timeout", "rate_limited", "not_found" (fixed set).

Unsafe label values:

- **User ID** — unbounded.
- **Request ID** — unique per request.
- **Session ID** — unique per session.
- **IP address** — could be many.
- **Query parameter values** — unbounded.

When you must track per-user behavior, use logs: "payment processed for user_id=123." Aggregate in Prometheus or your time series database by counting how many logs match a filter.

---

## 77.4 Traces: OpenTelemetry

OpenTelemetry is the standard for distributed tracing. It defines how to emit spans (the building blocks of traces) and how to propagate trace context across services.

### Emit a Span

```cpp
#include <opentelemetry/api/trace/tracer.h>
#include <opentelemetry/api/trace/span.h>

using opentelemetry::trace::Tracer;
using opentelemetry::trace::SpanStartOptions;
using opentelemetry::trace::StatusCode;

auto tracer = opentelemetry::trace::global::GetTracerProvider()
    ->GetTracer("my-service", "1.0.0");

// Start a span
auto span = tracer->StartSpan("process_payment");
{
    // Do work
    span->SetAttribute("user_id", user_id);
    span->SetAttribute("amount_cents", 5000);
    
    // If something goes wrong
    if (error) {
        span->SetStatus(StatusCode::kError, error_message);
    }
}
// Span ends automatically when it goes out of scope
```

A span has:
- **Name** — what operation (process_payment, validate_token).
- **Attributes** — key-value pairs (user_id, amount_cents). Structured data about the operation.
- **Events** — discrete occurrences ("cache_miss", "retry_attempt"). Different from attributes; attributes are facts about the span, events are things that happen during it.
- **Links** — references to other spans, useful for async operations or cross-process calls.
- **Status** — whether the operation succeeded or failed.

### Parent-Child Relationships

When a span creates another span, the new span becomes a child:

```cpp
auto parent_span = tracer->StartSpan("handle_request");

// Inside handle_request, call another service
auto child_span = tracer->StartSpan("call_inventory_service");
child_span->SetStatus(inventory_status);
```

The trace collector builds a tree:

```
handle_request (0-145ms)
  ├─ call_auth_service (10-25ms)
  ├─ call_inventory_service (26-95ms)
  │   └─ database_query (35-92ms)
  └─ call_payment_service (96-140ms)
      └─ stripe_api_call (105-138ms)
```

From this tree, you can see that inventory was the bottleneck (70ms out of 145ms). If you optimize that, the request gets faster.

### Trace Context Propagation

For traces to work across services, you must propagate the trace ID. When Service A calls Service B, A includes the trace ID in the HTTP header:

```cpp
// Service A calls Service B
auto span = tracer->StartSpan("call_service_b");

// Inject trace context into HTTP headers
opentelemetry::trace::propagation::HTTPTextPropagator propagator;
std::map<std::string, std::string> headers;
propagator.Inject(
    opentelemetry::context::Context::GetCurrent(),
    headers
);

// Send HTTP request with headers
httpClient.post("http://service-b.local/endpoint", 
                {
                    {"traceparent", headers["traceparent"]},
                    {"tracestate", headers["tracestate"]}
                });
```

Service B receives the headers and continues the trace:

```cpp
// Service B receives request
std::map<std::string, std::string> received_headers = request.headers();

// Extract trace context
opentelemetry::trace::propagation::HTTPTextPropagator propagator;
auto context = propagator.Extract(received_headers);

// New span becomes a child of the parent
auto span = tracer->StartSpan("handle_request", SpanStartOptions{.parent = context});
```

The trace system (Jaeger, Zipkin, Datadog) receives spans from all services, groups them by trace ID, and builds the tree.

### Sampling Strategies

Sampling decides which traces to keep. Sampling is necessary because tracing every request at high throughput is expensive.

**Head-based sampling** — decide at the start of the request whether to trace it.

```cpp
if (random(0, 1) < 0.1) {  // 10% sampling
    // Trace this request
}
```

Simple but problematic: if you sample 10% of requests, and a request that was sampled calls a service with a 5% sampling rate, the child span might be dropped even though the parent is kept. You lose the complete trace.

**Tail-based sampling** — decide at the end, after seeing the request's outcome. "Trace failed requests; sample successful ones."

```cpp
auto span = tracer->StartSpan("handle_request");
bool has_error = true;
try {
    // Do work
    has_error = false;
} catch (...) {
    span->SetStatus(StatusCode::kError);
}

// Send span to sampler
if (has_error || random(0, 1) < 0.01) {  // Always trace errors; sample 1% of successes
    send_to_collector(span);
}
```

Tail-based sampling gives you complete traces of failures (which you care about) while sampling successes (which you don't), reducing cost.

Both approaches can be combined: head-based sampling for high-volume normal traffic, tail-based for errors and anomalies.

---

## 77.5 Correlating Across Signals

Three signals are powerful only if you can connect them. The link is the trace ID.

### Inject Trace ID Into Logs

Every log line should include the trace ID:

```json
{
  "timestamp": "2025-05-09T14:23:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "message": "Payment processing failed",
  "reason": "insufficient_funds"
}
```

With this, you can:
1. Find the trace in your trace system by clicking the trace_id.
2. See all logs from that request across all services.
3. Understand the timeline (which log came first, second, etc.).

Implementation:

```cpp
// Get current trace context
auto trace_context = opentelemetry::context::RuntimeContext::GetCurrent();
auto span = opentelemetry::trace::GetSpan(trace_context);
auto trace_id = span->GetContext().trace_id();

// Include in log
logJson["trace_id"] = trace_id.ToLowerBase16();
std::cout << logJson.dump() << "\n";
```

### Exemplars: Link Metrics to Traces

An exemplar is a trace ID attached to a metric point. When a histogram bucket increments, it can include an exemplar — "this request is in the p99 bucket, and its trace is 4bf92f3577b34da6a3ce929d0e0e4736."

```
request_duration_seconds_bucket{le="0.5"} = 49500  # exemplar: trace_id=4bf...
```

A Prometheus-compatible system can click the exemplar and jump to the trace. You can see not just "p99 latency was 500ms" but *why* — which span was slow, which service was the bottleneck.

---

## 77.6 SLI / SLO Briefly

An SLI (Service Level Indicator) is a measurable property of the service. An SLO (Service Level Objective) is a target for that SLI.

Example:

- **SLI**: "percentage of requests that succeed and complete in under 200ms."
- **SLO**: "we target 99.5% of requests to meet that SLI."

SLIs are observable — you measure them from logs or metrics. SLOs are goals — you track whether you're hitting them.

Why this beats "uptime in nines" (99.9%, 99.99%):

1. **Uptime is the wrong metric.** A system can be "up" (responds to pings) but slow (serving every request in 5 seconds). Users care about usability, not uptime.
2. **SLI captures what users experience.** If 99.5% of requests succeed in under 200ms, users are happy.
3. **Error budget** — if your SLO is 99.5%, you have a 0.5% error budget. You can spend it on risky deploys, experiments, or let it accumulate.

Observability enables SLI/SLO: you cannot define or track SLIs without logs (to measure success rate), metrics (to measure latency), and traces (to understand why latencies vary).

---

## 77.7 Logging in C++

### spdlog: Simple, Fast, Flexible

spdlog is a header-only logging library. It is simple to use, async by default, and extensible.

Basic usage:

```cpp
#include "spdlog/spdlog.h"
#include "spdlog/sinks/stdout_color_sinks.h"

int main() {
    // Create a logger that writes to stdout
    auto console = spdlog::stdout_color_mt("console");
    console->set_level(spdlog::level::info);
    
    console->info("Application started");
    console->warn("This is a warning");
    console->error("An error occurred");
    
    return 0;
}
```

Formatted output:

```
[14:23:45.123] [console] [info] Application started
[14:23:45.124] [console] [warn] This is a warning
[14:23:45.125] [console] [error] An error occurred
```

### Structured Logging with spdlog

For JSON output, create a custom sink:

```cpp
#include "spdlog/spdlog.h"
#include "spdlog/sinks/base_sink.h"
#include <nlohmann/json.hpp>

using json = nlohmann::json;

class JsonSink : public spdlog::sinks::base_sink<std::mutex> {
protected:
    void sink_it_(const spdlog::details::log_msg& msg) override {
        json j;
        j["timestamp"] = fmt::format("{:%Y-%m-%dT%H:%M:%S.%fZ}", 
                                     std::chrono::system_clock::now());
        j["level"] = spdlog::level::to_string_view(msg.level);
        j["message"] = msg.payload;
        
        // Add custom fields
        if (msg.metadata) {
            // Assuming metadata contains trace_id, user_id, etc.
            j["trace_id"] = msg.metadata.trace_id;
        }
        
        std::cout << j.dump() << "\n";
    }
    
    void flush_() override {
        std::cout.flush();
    }
};

int main() {
    auto json_sink = std::make_shared<JsonSink>();
    spdlog::spdlog::logger logger("json_logger", json_sink);
    
    logger.info("Payment processed", {{"user_id", "usr-123"}, {"amount", 5000}});
    
    return 0;
}
```

Output:

```json
{"timestamp": "2025-05-09T14:23:45.123Z", "level": "info", "message": "Payment processed", "user_id": "usr-123", "amount": 5000}
```

### Async Logging

Logging can block your hot path. Async logging decouples the application from I/O:

```cpp
#include "spdlog/async.h"

spdlog::init_thread_pool(8192, 1);  // 8KB buffer, 1 worker thread

auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
std::vector<spdlog::sink_ptr> sinks{console_sink};
auto logger = std::make_shared<spdlog::async_logger>(
    "async_logger",
    sinks.begin(),
    sinks.end(),
    spdlog::thread_pool(),
    spdlog::async_overflow_policy::overrun_oldest  // Drop old logs if buffer is full
);

logger->info("Non-blocking log");  // Returns immediately; logged asynchronously
```

The worker thread batches logs and flushes them to the sink. Your hot path doesn't wait for I/O.

---

## 77.8 Worked Example: An HTTP Service with Three Signals

Let's build a small HTTP service that emits logs, metrics, and traces. A single curl request will appear in all three signals.

### Setup

```cpp
// server.h
#pragma once
#include <string>
#include <memory>
#include <opentelemetry/trace/tracer.h>

class PaymentService {
public:
    struct PaymentRequest {
        std::string user_id;
        int amount_cents;
    };
    
    struct PaymentResponse {
        bool success;
        std::string message;
        std::string trace_id;
    };
    
    PaymentResponse process_payment(const PaymentRequest& req);
};
```

### Implementation with Observability

```cpp
// server.cpp
#include "server.h"
#include "spdlog/spdlog.h"
#include <opentelemetry/api/trace/tracer.h>
#include <opentelemetry/api/metrics/meter.h>
#include <nlohmann/json.hpp>

using json = nlohmann::json;
using opentelemetry::trace::Tracer;

// Global logger and tracer (in real code, use dependency injection)
std::shared_ptr<spdlog::logger> g_logger;
std::shared_ptr<Tracer> g_tracer;
opentelemetry::metrics::Counter* g_payment_attempts = nullptr;
opentelemetry::metrics::Histogram* g_payment_latency = nullptr;

PaymentService::PaymentResponse PaymentService::process_payment(
    const PaymentRequest& req) {
    
    // Start a trace span
    auto span = g_tracer->StartSpan("process_payment");
    span->SetAttribute("user_id", req.user_id);
    span->SetAttribute("amount_cents", req.amount_cents);
    
    // Get trace ID for correlation
    auto trace_context = opentelemetry::context::RuntimeContext::GetCurrent();
    auto trace_id = opentelemetry::trace::GetSpan(trace_context)
        ->GetContext().trace_id().ToLowerBase16();
    
    // Measure latency
    auto start = std::chrono::high_resolution_clock::now();
    
    // Emit a log
    json log_json;
    log_json["timestamp"] = "2025-05-09T14:23:45.123Z";
    log_json["level"] = "INFO";
    log_json["service"] = "payment-service";
    log_json["trace_id"] = trace_id;
    log_json["event"] = "payment_started";
    log_json["user_id"] = req.user_id;
    log_json["amount_cents"] = req.amount_cents;
    g_logger->info(log_json.dump());
    
    // Simulate processing
    bool success = (req.amount_cents < 1000000);  // Reject large amounts
    if (!success) {
        // Emit error log
        json error_log;
        error_log["timestamp"] = "2025-05-09T14:23:45.124Z";
        error_log["level"] = "ERROR";
        error_log["service"] = "payment-service";
        error_log["trace_id"] = trace_id;
        error_log["event"] = "payment_failed";
        error_log["reason"] = "amount_too_large";
        error_log["user_id"] = req.user_id;
        error_log["amount_cents"] = req.amount_cents;
        g_logger->error(error_log.dump());
        
        span->SetStatus(opentelemetry::trace::StatusCode::kError, 
                       "amount_too_large");
    }
    
    // Measure latency
    auto end = std::chrono::high_resolution_clock::now();
    auto latency_ms = std::chrono::duration<double, std::milli>(end - start).count();
    
    // Emit metrics
    g_payment_attempts->Add(1, {{"status", success ? "success" : "failure"}});
    g_payment_latency->Record(latency_ms, {{"status", success ? "success" : "failure"}});
    
    // Emit success log
    json success_log;
    success_log["timestamp"] = "2025-05-09T14:23:45.125Z";
    success_log["level"] = "INFO";
    success_log["service"] = "payment-service";
    success_log["trace_id"] = trace_id;
    success_log["event"] = "payment_completed";
    success_log["user_id"] = req.user_id;
    success_log["amount_cents"] = req.amount_cents;
    success_log["latency_ms"] = latency_ms;
    g_logger->info(success_log.dump());
    
    return {success, success ? "Payment processed" : "Payment failed", trace_id};
}
```

### Usage

One curl request:

```bash
curl -X POST http://localhost:8080/pay \
  -H "Content-Type: application/json" \
  -d '{"user_id": "usr-123", "amount_cents": 5000}'
```

**In logs (stdout), you see:**

```json
{"timestamp": "2025-05-09T14:23:45.123Z", "level": "INFO", "service": "payment-service", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "event": "payment_started", "user_id": "usr-123", "amount_cents": 5000}
{"timestamp": "2025-05-09T14:23:45.124Z", "level": "INFO", "service": "payment-service", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "event": "payment_completed", "user_id": "usr-123", "amount_cents": 5000, "latency_ms": 1.23}
```

**In Prometheus metrics, you see:**

```
payment_attempts_total{status="success"} = 1
payment_latency_seconds{status="success"} = [1.23ms bucket]
```

**In Jaeger traces, you see:**

```
Trace 4bf92f3577b34da6a3ce929d0e0e4736
  └─ Span: process_payment (0-1.23ms)
      Attributes: user_id=usr-123, amount_cents=5000
```

All three signals are correlated by trace_id. Click the trace ID in Datadog and you jump from the log to the trace. Click the trace in Jaeger and you can see all logs from that request.

---

## 77.9 Tradeoffs

| Signal | Pros | Cons | When to Use |
|--------|------|------|-------------|
| **Logs** | Rich context; human-readable; machine-queryable | High cardinality; expensive to retain; requires aggregation | Every event that humans need to understand |
| **Metrics** | Cheap; queryable; aggregable; enables alerting | Low cardinality (or explodes in cost); no context | Observing trends, rates, distribution of behavior |
| **Traces** | Shows causality; reveals bottlenecks; cross-service | Expensive if not sampled; requires coordination | Understanding latency and request flow |

The right approach: **emit all three.** Logs capture context and enable investigation. Metrics enable alerting ("latency is above threshold"). Traces show you why ("service B was slow"). Together they resolve incidents.

---

## 77.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "More logs = better observability." | High cardinality, unstructured logs hide the signal. Targeted logs + metrics + traces beat volumes of text. Quality over quantity. |
| "We should log everything at DEBUG and disable DEBUG in production." | Disabling a level doesn't reduce code, only output. Structured INFO logs + adaptive logging on errors is better. |
| "Metrics are for dashboards." | Metrics enable alerting. "Alert when request latency > 1s" is only possible with metrics. Logs cannot trigger alerts. |
| "Traces are only for slow requests." | Traces show normal latency distribution. You need traces of fast requests to know what "fast" looks like. Tail-based sampling helps. |
| "We need to trace everything." | Tracing everything is prohibitively expensive. Sample intelligently: all errors, all slow requests, small percentage of fast requests. |
| "SLO is just uptime." | SLO is a measurable target (99.5% of requests under 200ms). Uptime is binary (up or down). Uptime doesn't capture user experience. |
| "Observability doesn't matter; testing does." | Testing catches bugs. Observability catches failures you didn't predict. Both are essential. |

---

## 77.11 Exercises

1. **Define SLIs for a system you know.** Pick an application (web service, database, tool). Define three SLIs: success rate, latency (p99), and error rate. What would reasonable SLOs be?

2. **Emit structured logs.** Write a C++ program that logs (in JSON) when it opens a file, reads a line, and closes the file. Include timestamp, level, filename, and line number. Parse the JSON to verify it's valid.

3. **Measure cardinality.** In a hypothetical HTTP metrics system with 100 endpoints and 10 status codes, add a label for "environment" (3 values). How many time series? Now add "user_id" and assume 1 million users. What happens?

4. **Create a trace.** Write a simple C++ program that makes two HTTP calls in sequence. Using OpenTelemetry, emit spans for each call. Structure the spans so one is a child of the other (use trace context propagation, even if both are local). Print the trace as JSON.

5. **Correlate signals.** In the worked example, add a metric that counts failures by reason ("amount_too_large", "network_error", etc.). Query: "all ERROR logs for this trace_id" and "count of failures by reason in the same time window." Which signal answers which question?

6. **Design a logging strategy.** You are building a payment processor. What would you log at each level (DEBUG, INFO, WARN, ERROR)? When would you add DEBUG context on error?

---

## 77.12 Summary

Observability is not logging, metrics, or tracing alone — it is all three, connected by a trace ID. Logs capture rich context and enable investigation. Metrics enable alerting and reveal trends. Traces show causality and latency distribution across services.

Structured (JSON) logs are queryable and scalable. Free-text logs are cheap to generate but expensive to analyze at scale. Log levels (DEBUG, INFO, WARN, ERROR) should be used judiciously; targeted INFO logging beats "log everything at DEBUG and grep."

Metrics compress well but require careful label design to avoid cardinality explosion. Histograms reveal distributions (p99 latency); counters reveal rates (requests per second).

Traces connect the dots across services. Parent-child spans show which operation was slow. Sampling strategies (head-based, tail-based) make tracing affordable at scale.

Correlating logs, metrics, and traces through trace IDs transforms three independent signals into a coherent story. SLI/SLO thinking (define success, measure it, track against target) gives structure to observability.

The cost of observability is storage, ingestion bandwidth, and query latency. The benefit is the ability to resolve production incidents without reproducing them — the engineer at 3 AM who can see exactly what the system was doing when it broke.

---

> **[← Previous: Configuration Systems](09-configuration-systems.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: Part 8 — Advanced Systems Thinking](../part-08-advanced-systems-thinking/README.md)** *(coming soon)*
