# Part 5 — Runtime & Concurrency

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Part 5 covers what your code looks like *while it's running*. Processes vs threads at the OS level. Concurrency vs parallelism (different problems, often conflated). Schedulers — kernel and userland. The state machine inside `async/await`. The cancellation problem and how every modern runtime solves it. Event loops, coroutines, Python's asyncio specifically. Then thread safety: the three hazards, locks vs atomics, and the memory ordering that breaks intuition.

After Part 5, you can read concurrent code in any modern language and reason about what's executing where, what's contending with what, and what can go wrong.

---

## Chapters

46. **[Processes vs Threads](01-processes-vs-threads.md)** — anatomy, cost, communication, isolation, when to use which.
47. **[Concurrency vs Parallelism](02-concurrency-vs-parallelism.md)** — Pike's distinction; Amdahl's law; patterns.
48. **[Schedulers](03-schedulers.md)** — OS preemptive, cooperative, M:N, work-stealing.
49. **[Async Runtime Internals](04-async-runtime-internals.md)** — state machines, event loops, wakers.
50. **[Why Go Uses Context](05-why-go-uses-context.md)** — cancellation propagation; deadlines; how other languages solve it.
51. **[Cancellation Propagation](06-cancellation-propagation.md)** — four models; structured concurrency; HTTP server cancellation.
52. **[Event Loops](07-event-loops.md)** — the loop stripped down; phases; microtasks; epoll/kqueue underneath.
53. **[Coroutines](08-coroutines.md)** — stackful vs stackless; C++20, JS, Python, Rust; why suspension points matter.
54. **[How Python AsyncIO Works](09-how-python-asyncio-works.md)** — tasks, futures, the GIL, common footguns.
55. **[Thread Safety](10-thread-safety.md)** — data race vs race condition vs deadlock; what atomic and volatile actually do.
56. **[Locks and Atomics](11-locks-and-atomics.md)** — futex internals; lock_guard; CAS loops; lock-free briefly.
57. **[Memory Ordering](12-memory-ordering.md)** — relaxed, acquire/release, seq_cst; x86 vs ARM; double-checked locking done right.

---

## How to Use Part 5

- **Run the benchmarks.** Concurrency intuition fails almost everyone — measuring is the only way to build a reliable mental model.
- **Read Ch 55–57 in order.** They build a single argument; skipping the middle one (Ch 56) leaves the last (Ch 57) confusing.
- **Forward to Part 8** if you want even more on cache effects under contention (false sharing).

> **Next: Part 6 — Languages & Runtime Design** *(coming soon)*
