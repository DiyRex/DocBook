# Part 8 — Advanced Systems Thinking

> **[← Back to book home](../README.md)**  ·  **[← Back to DocBook home](../../../README.md)**

Part 8 is the deep-systems chapter set. The first half is OS-adjacent — memory-mapped files, networking, syscalls, kernel/userspace, serialization, protocol design. The second half is the CPU-adjacent — performance engineering, profiling, pipelines, branch prediction, SIMD, cache misses, false sharing.

By the end you can read a perf flame graph and know where to look, recognize a misaligned cache line at a glance, and reason about why a "fast" service is bottlenecked on a syscall storm.

---

## Chapters

78. **[Memory Mapped Files](01-memory-mapped-files.md)** — `mmap` mechanics; when it beats read/write; databases and IPC.
79. **[Networking Internals](02-networking-internals.md)** — sockets, TCP buffers, backlog, Nagle, zero-copy, QUIC.
80. **[Syscalls](03-syscalls.md)** — what a syscall costs; vDSO, io_uring; tracing with strace.
81. **[Kernel vs User Space](04-kernel-vs-user-space.md)** — the privilege boundary, eBPF, microkernel vs monolithic.
82. **[Serialization Internals](05-serialization-internals.md)** — text vs binary, schema'd, zero-copy, evolution.
83. **[Protocol Design](06-protocol-design.md)** — framing, versioning, idempotency, security from day one.
84. **[Performance Engineering](07-performance-engineering.md)** — measure first; the bottleneck hierarchy; tools.
85. **[Profiling](08-profiling.md)** — sampling vs instrumentation; flame graphs; off-CPU profiling.
86. **[CPU Pipelines](09-cpu-pipelines.md)** — pipelining, ILP, hazards, speculation, what compilers do.
87. **[Branch Prediction](10-branch-prediction.md)** — predictors, branch-free code, the famous sorted-array example.
88. **[SIMD](11-simd.md)** — auto-vectorization, intrinsics, std::simd; data layout for vector instructions.
89. **[Cache Misses (Deeper)](12-cache-misses.md)** — three C's; tiling; software prefetching; TLB.
90. **[False Sharing](13-false-sharing.md)** — adjacent variables, padding, `std::hardware_destructive_interference_size`.

---

## How to Use Part 8

- **Run every benchmark.** This part is the "I read it and didn't believe it until I measured" part.
- **Don't apply low-level tricks before measuring.** The hierarchy in Ch 84 is real: algorithm and cache misses dominate; SIMD and branch tricks are the last 5%.

> **Next: Part 9 — Building A Mental Model For Any Language** *(coming soon)*
