# Part 7 — Building Real Systems

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Parts 1–6 built the substrate, the abstractions, and the runtime models. Part 7 puts them to work. Each chapter walks through a real subsystem you'll build (or replace) at some point: a CLI tool, a web server, a database layer, a mini framework, a DI container, an event bus, a plugin loader, hot reloading, configuration, observability.

The point is not to teach the libraries — they change. The point is to show what's underneath the libraries so you can pick, replace, or build them with judgment.

---

## Chapters

68. **[Designing a CLI Tool](01-designing-a-cli-tool.md)** — argument parsing, exit codes, signals, stdin/stdout discipline, testability.
69. **[Designing a Web Server](02-designing-a-web-server.md)** — connection lifecycle, concurrency model, routing, middleware, graceful shutdown.
70. **[A Database Layer](03-a-database-layer.md)** — pools, repositories, transactions, migrations, ORMs vs hand-rolled.
71. **[A Mini Framework](04-a-mini-framework.md)** — library vs framework; building router + middleware + DI in 150 lines.
72. **[A DI Container](05-a-di-container.md)** — registrations, lifetimes, type erasure in C++, captive dependency pitfalls.
73. **[An Event Bus](06-an-event-bus.md)** — in-process vs cross-process; delivery semantics; ordering.
74. **[Plugin Architecture](07-plugin-architecture.md)** — dynamic libraries, embedded interpreters, subprocesses, WebAssembly.
75. **[Hot Reloading](08-hot-reloading.md)** — module swap, file watching, state preservation, what breaks.
76. **[Configuration Systems](09-configuration-systems.md)** — layering, typed config, secrets, feature flags.
77. **[Observability and Logging](10-observability-and-logging.md)** — structured logs, metrics, traces, SLI/SLO.

---

## How to Use Part 7

- **Build at least one of these.** Reading is one mode of learning; building is another. The chapters give you enough to write a working version of each subsystem.
- **Compare to the library you use.** When you read your framework's source after this part, you'll recognize what's underneath instead of seeing magic.

> **Next: Part 8 — Advanced Systems Thinking** *(coming soon)*
