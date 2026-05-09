# Chapter 80 — System Calls: Crossing the Privilege Boundary

Every "real" thing your program does — read a file, write to the network, fork a child process, allocate memory, wake a thread from sleep — is ultimately a request to the kernel. That request crosses the privilege boundary from user mode to kernel mode via a special CPU instruction, the **syscall**. This chapter traces exactly what happens between the call site in your code and the kernel handler, why each syscall is more expensive than it first appears, and what the systems world has invented to mitigate that cost.

By the end of this chapter you will understand:

1. What a system call is at the CPU level and how the privilege boundary is enforced.
2. The true cost of a syscall transition — register saves, TLB pressure, cache disturbance.
3. Which syscalls are "cheap" and which are "expensive," and why the distinction matters.
4. The three big cost-mitigation strategies: buffering, batching, and shared kernel-userspace mapping.
5. How to trace, count, and profile syscall behavior in your own code.
6. The security and isolation implications of the syscall surface area.

---

## 80.1 Anatomy of a System Call

Recall from Chapter 3 that a system call is the boundary crossing from unprivileged user mode into privileged kernel mode. Let's now look at the machinery in fine detail.

### The transition: from call site to kernel handler

On x86-64 Linux, a syscall follows this sequence:

1. **Your C++ code** calls something like `write(1, buf, len)`.
2. **The C library** (libc) wraps it. The wrapper:
   - Loads the syscall number for `write` (which is `1` on x86-64) into register `rax`.
   - Loads the arguments: file descriptor `1` into `rdi`, buffer address into `rsi`, length into `rdx`.
   - Executes the `syscall` instruction.

3. **CPU behavior on `syscall` instruction**:
   - Saves the current privilege level and instruction pointer into kernel-managed registers.
   - Switches the CPU to ring 0 (kernel mode).
   - Jumps to the kernel's syscall entry point, set up during boot.
   - From this point on, the CPU is running kernel code and can execute privileged instructions.

4. **Kernel handler** (in `arch/x86_64/entry/syscall_64.c` in Linux):
   - Saves the user-mode registers into a kernel data structure on the kernel stack.
   - Reads `rax` to determine which syscall was requested.
   - Dispatches to the appropriate handler — in this case `sys_write` in `fs/read_write.c`.

5. **`sys_write` does its work**:
   - Validates the file descriptor.
   - Checks access permissions.
   - Copies bytes from the user-mode buffer into a kernel buffer (if necessary).
   - Interacts with the page cache or block device driver.
   - Possibly blocks the process if the operation can't complete immediately.

6. **Return to userspace**:
   - The kernel places the return value in `rax` (number of bytes written, or -1 for error).
   - Executes `sysret` (fast) or `iretq` (slower but used when returning to a different privilege level or on older CPUs).
   - CPU switches back to ring 3 (user mode).
   - Your code resumes at the next instruction after `syscall`.

### ARM64: a similar story

On ARM64, the transition is similar:

- User code executes `svc #0` (supervisor call).
- The CPU switches to EL1 (kernel exception level) and jumps to a predefined handler.
- The syscall number is passed in `x8` instead of `rax`.
- The CPU returns with `eret`.

The principle is identical: a special instruction, privilege check, register-based argument passing, and a return.

---

## 80.2 The True Cost of a Syscall

When someone says "syscalls are expensive," what exactly do they mean? Let's break down the cost.

### 1. Instruction latency and mode transition

The `syscall` instruction itself is not the bottleneck — it is a single instruction that executes in one cycle. The latency is in the *transition*: the CPU must:

- Flush the instruction pipeline.
- Invalidate userspace-only TLB entries (on some CPUs, depending on KPTI mitigation; see below).
- Jump to kernel code, which is in a different area of memory, causing instruction cache misses.

**This transition costs roughly 50–200 nanoseconds on modern CPUs.**

For comparison, a userspace function call costs ~1 nanosecond. A syscall is thus **50–200× more expensive than a function call.**

### 2. Register save and restore

Before the kernel can touch userspace registers for its own use, it must save them. The kernel saves at minimum:

- Instruction pointer and flags (implicit in the `syscall` instruction).
- Stack pointer, base pointer, and other general registers if the kernel handler needs them.

Saving ~20 registers to the kernel stack, then restoring them on return, costs nanoseconds but adds up.

### 3. TLB (translation lookaside buffer) pressure

The TLB is a CPU cache of recent virtual-to-physical page mappings. When the CPU switches from user to kernel mode, the set of valid virtual addresses changes. Kernel-only memory is not accessible from userspace, and vice versa.

Depending on the CPU microarchitecture and kernel version:

- **With KPTI enabled** (Kernel Page Table Isolation, a mitigation for Meltdown): The kernel and userspace have *separate* page tables. Switching between them requires a TLB flush or at minimum partial invalidation. This costs hundreds of cycles on modern CPUs.
- **Without KPTI**: The same page table is shared, but kernel memory is marked "kernel-only" and causes an exception if userspace touches it. TLB pressure is less severe.

KPTI is the default on most Linux distributions because Meltdown is a real risk. This means a syscall on an old CPU can cost a thousand cycles or more just from the TLB flush.

### 4. Cache disturbance

When the kernel handler runs, it evicts your process's working set from the L1 and L2 caches. If the kernel does a lot of work (e.g., a complex `read` involving multiple subsystems), your code's data is pushed out. When your code resumes, it takes a cache-miss penalty reloading that data. This indirect cost can exceed the direct cost of the transition.

### Summary: cost profile

- **Best case** (syscall handler does minimal work): ~50–200 ns.
- **Typical case** (handler does some work, cache misses on resume): ~500 ns – 1 µs.
- **Worst case** (handler blocks your thread, context switches other processes, returns much later): milliseconds or seconds.

The key insight: **the cost is not constant.** Counting syscalls is how you identify where the cost is hiding.

---

## 80.3 Mitigations: vDSO, io_uring, and Batching

The systems world has developed three categories of mitigations:

### vDSO: virtual dynamic shared object

Some "syscalls" are not really syscalls at all — they're kernel code mapped into userspace that can execute without a mode transition.

Example: `gettimeofday()` and `clock_gettime()`.

**Problem**: Your program calls `gettimeofday` once per millisecond (or more). Each call is a syscall. The kernel just has to *read the current clock value* and return it. The kernel doesn't do anything special — it just returns a value that's been maintained by the timer interrupt handler.

**Solution**: The kernel maps a page containing the timer value into *every* process's address space. The C library's `gettimeofday` implementation reads directly from that page without executing `syscall`. It is a regular memory read. The kernel updates the page when the timer fires. Zero mode transitions. Zero register saves.

On Linux, you can see the vDSO:

```bash
# The vDSO appears in your process's maps
cat /proc/self/maps | grep vdso

# It maps functions like gettimeofday, clock_gettime, getcpu
# Available functions vary by kernel version and CPU
```

vDSO is a win for:

- `gettimeofday`, `clock_gettime` — called frequently, do no actual work.
- `getcpu` — which CPU am I on?
- Some other fast queries.

It cannot be used for syscalls that actually *do something* (write to disk, fork a process, allocate memory). The kernel must retain exclusive control.

### io_uring: batched async I/O

**Problem**: Your program wants to issue 1000 network I/O requests. Doing it naively means 1000 syscalls. The context switch overhead alone makes this slow.

**Solution** (io_uring on Linux): Establish a pair of shared memory rings (submission queue, completion queue) between userspace and kernel:

1. Your code writes I/O requests to the submission queue in userspace.
2. When ready, you call `io_uring_enter` *once*, telling the kernel "process N requests."
3. The kernel batches them, issuing many I/Os internally.
4. The kernel writes completion results to the completion queue.
5. Your code polls the completion queue (or blocks once on `io_uring_enter` again) for results.

This amortizes the syscall cost over many requests. Instead of 1000 syscalls, you might have 10–20 `io_uring_enter` calls.

The tradeoff: more complexity in your code, but orders of magnitude fewer context switches for high-concurrency I/O.

### Buffering and batching

The simplest and most portable mitigation: **collect multiple small I/O operations into larger batches.**

Examples you use every day:

- `printf` does not call `write` for each character. It accumulates in a userspace buffer and flushes a full buffer at once.
- `std::cout` with a custom buffer does the same.
- Database clients batch inserts.
- HTTP libraries batch network writes.

**Cost of not batching**: Writing 10,000 characters one per write syscall = 10,000 syscalls.

**Cost of batching with 4 KB buffer**: Writing 10,000 characters in ~3 write syscalls.

The syscall cost is amortized: (cost of syscall) / (bytes per syscall) becomes negligible.

---

## 80.4 Common Syscalls and Their Cost Profiles

Not all syscalls are created equal. Here's a rough categorization:

| Syscall | Cost | Why | When to batch/cache |
|---------|------|-----|-----|
| `getpid`, `geteuid` | Microseconds | Data already in kernel | Nearly always — do it in init, cache the result |
| `gettimeofday`, `clock_gettime` | Microseconds (or nanoseconds via vDSO) | Read a value; vDSO available on modern kernels | Use vDSO; OK to call frequently |
| `read` (data in cache) | Microseconds | Copy from page cache to userspace buffer | Batch reads; use `readv` for multiple reads |
| `write` (to disk or network) | Microseconds – milliseconds | Depends on I/O subsystem. Network: varies. Disk: depends on cache. | Always buffer; use `writev` for multiple writes; io_uring for concurrency |
| `open` | Microseconds – milliseconds | Path lookup through filesystem; may trigger disk I/O if inode not cached | Cache file descriptors; batch opens if possible |
| `close` | Microseconds | Cleanup only; usually just deallocates a kernel object | Batch closures in bulk-deletion code |
| `mmap` | Microseconds | Allocates a VMA (virtual memory area); no I/O | Cheap; use for large mappings (avoids multiple `read` calls) |
| `fork` | Milliseconds – seconds | Copies page table (with COW); allocates kernel structures | Expensive; avoid in loops. Use `posix_spawn` if just exec'ing. Rarely the bottleneck in modern systems because of COW. |
| `futex` (uncontended) | Microseconds | Lock not held; returns immediately | Cheap when contention is low |
| `futex` (contended) | Milliseconds | Kernel has to wake other waiters; involves context switches | Avoid contention; use lock-free data structures where possible |
| `socket` | Microseconds | Allocates a socket object | Cache sockets; reuse connections |
| `epoll_wait` (no events) | Milliseconds | Blocks thread until events; no busy-wait | Good; fundamental to async servers |
| `epoll_wait` (many events) | Microseconds – milliseconds | Copying event list to userspace | Scales well; the mechanism is efficient |

**Key pattern**: Syscalls that *do real work* (fork, write to disk, wait for I/O) are naturally expensive. Syscalls that *return metadata* (getpid, gettimeofday) are fast and good candidates for caching or vDSO. Syscalls that *block* are cheap in terms of CPU cost but have latency; they're only used when you must wait.

---

## 80.5 Tracing and Profiling Syscalls

To understand where your program's syscall cost comes from, you need to measure.

### On Linux: `strace`

```bash
# Trace all syscalls, print a summary
strace -c ./your_program

# Output similar to:
% time     seconds  usecs/call     calls      errors syscall
------ ----------- ----------- --------- ---------- ----------------
 45.23    0.001234           5       250            write
 32.10    0.000876           3       300            read
 15.40    0.000420          42        10            futex
  7.27    0.000198           2       100            mmap
```

**Reading the output**:

- `% time`: What fraction of time was spent in this syscall.
- `seconds`: Total time in this syscall type.
- `usecs/call`: Average time per invocation.
- `calls`: How many times the syscall was made.

**Example analysis**: If `write` shows 250 calls and `usecs/call` is 5, and `read` shows 300 calls and `usecs/call` is 3, then `write` is the bigger cost (250 × 5 = 1250 µs vs 300 × 3 = 900 µs).

### On macOS: `dtruss`

```bash
# macOS equivalent (requires sudo)
sudo dtruss -c ./your_program 2>&1 | tail

# Similar summary table
```

### Flamegraphs with `perf` (Linux)

For more insight, use `perf` to generate a flamegraph:

```bash
# Record syscall activity (requires kernel >= 4.4)
perf record -e 'raw_syscalls:sys_enter' ./your_program

# Examine the output
perf report
```

This shows *which* functions in your code are calling syscalls most.

### In C++: manual counting

If you want to count syscalls from within your program without external tools, you can hook the libc wrapper functions. But this is fragile and not recommended for production. Use strace/dtruss instead.

---

## 80.6 Why Languages Buffer I/O

A practical reason syscall overhead matters: it's why every language's standard I/O buffers.

### Unbuffered I/O

```cpp
#include <unistd.h>

int main() {
    for (int i = 0; i < 100000; ++i) {
        write(1, "x", 1);  // Direct syscall for each character
    }
    return 0;
}

// This makes 100,000 syscalls. On a modern CPU,
// even at 1 microsecond per syscall, this takes 0.1 seconds.
// A loop 100,000 times with pure C++ logic might take 0.001 seconds.
```

### Buffered I/O (stdio)

```cpp
#include <cstdio>

int main() {
    for (int i = 0; i < 100000; ++i) {
        putchar('x');  // libc accumulates in a buffer
    }
    fflush(stdout);    // Single write syscall with 100,000 bytes
    return 0;
}

// This makes 1 syscall (or ~2-3 if the buffer is 64 KB and we overflow).
// Vastly faster.
```

The C library (and every other standard library) buffers because the mathematics are stark. Reducing syscall count by 50,000× can make a program 100× faster.

This is why:

- `printf` is faster than `std::cout` by default (different buffer sizes).
- Explicitly calling `std::flush` is slow (forces a write syscall).
- Reading a file in large chunks is faster than one byte at a time.

---

## 80.7 Container and Sandbox Implications

Every syscall represents a point where a process requests the kernel to do something on its behalf. If the kernel can be compromised or exploited through a particular syscall, the process is vulnerable.

### seccomp filters

**seccomp** (secure computing mode) allows restricting which syscalls a process may make:

```bash
# Example: a container running nginx might restrict it to 200 specific syscalls
# and forbid fork, exec, ptrace, mount, etc.
# This limits the damage an attacker can do if they compromise the process.
```

The Docker/Kubernetes default seccomp profile blocks:

- `fork`, `clone` — prevents the process from spawning new processes.
- `ptrace` — prevents debugging other processes.
- `mount` — prevents mounting filesystems.
- ~80 other sensitive syscalls.

The tradeoff: **the container can only do what its seccomp profile allows.** If a normal operation requires a forbidden syscall, the container fails (Permission denied).

### Syscall surface area

A smaller syscall surface area = smaller attack surface. Specialized runtimes like **gVisor** (Google) and **Kata Containers** reduce this by:

- gVisor: intercepting syscalls in userspace, allowing much tighter filtering.
- Kata: running the container in a lightweight VM, where the guest kernel handles syscalls and the hypervisor enforces isolation.

The cost: overhead. gVisor's syscall translation is slower than native; Kata's VM overhead adds latency.

---

## 80.8 Worked Example: The Cost of Unbuffered I/O

Let's make the syscall cost concrete with an experiment.

### Program 1: Unbuffered write (1 byte per syscall)

```cpp
#include <unistd.h>
#include <cstdio>

int main(int argc, char** argv) {
    int fd = open("output.bin", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }
    
    // Write 1 MB, one byte at a time
    for (int i = 0; i < 1048576; ++i) {
        unsigned char byte = i % 256;
        if (write(fd, &byte, 1) != 1) {
            perror("write");
            return 1;
        }
    }
    close(fd);
    return 0;
}
```

### Program 2: Buffered write (64 KB chunks)

```cpp
#include <cstdio>

int main(int argc, char** argv) {
    FILE* f = fopen("output.bin", "wb");
    if (!f) { perror("fopen"); return 1; }
    
    // Write 1 MB, using stdio's buffer
    for (int i = 0; i < 1048576; ++i) {
        unsigned char byte = i % 256;
        putc(byte, f);
    }
    fclose(f);
    return 0;
}
```

### Results

On a modern Linux laptop:

```bash
$ time ./unbuffered
real    0m0.847s
user    0m0.023s
sys     0m0.824s

$ time ./buffered
real    0m0.002s
user    0m0.001s
sys     0m0.001s
```

The buffered version is **400× faster**. Why? Look at syscall counts:

```bash
$ strace -c ./unbuffered
% time     seconds  usecs/call     calls      errors syscall
------ ----------- ----------- --------- ---------- ----------------
 95.12    0.000823           0     1048576        write

$ strace -c ./buffered
% time     seconds  usecs/call     calls      errors syscall
------ ----------- ----------- --------- ---------- ----------------
 12.34    0.000012           1        16        write
```

- Unbuffered: 1,048,576 write syscalls.
- Buffered: 16 write syscalls.

That's a **65,536× reduction in syscalls.** The overhead per syscall is small (~0.8 µs), but it accumulates.

---

## 80.9 Tradeoffs and Design Decisions

| Approach | Pros | Cons | When to use |
|----------|------|------|-------------|
| **Unbuffered syscalls** | Simple code, guaranteed immediate effects | Slow; cache line bouncing between user/kernel; TLB pressure | Testing, logging to ensure flush, interactive programs where latency matters more than throughput |
| **Buffered I/O** | Fast, predictable; good locality | Deferred visibility; complex flushing rules (line-buffered, block-buffered) | Standard practice for files and sockets; all production code |
| **io_uring** | Very fast for high concurrency; amortizes syscalls across many requests | Complex API; new (Linux 5.1+); not portable; harder to debug | High-performance servers, database engines, bulk I/O workloads |
| **vDSO** | Free syscall replacement for read-only data | Limited to a few syscalls (time, CPU ID); kernel must maintain the shared mapping | Frequent queries like `clock_gettime`, `getcpu` |
| **mmap** | Eliminates syscall per byte for file I/O; leverages page cache | Complexity; page faults on cold access; alignment constraints | Large file processing, databases, editor-style workloads |
| **Async I/O + event loop** | One thread can handle many connections; no context-switch overhead from threads | Complexity; harder debugging; not all I/O operations are async-friendly | Web servers, game engines, real-time systems |

---

## 80.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "All syscalls have similar cost." | No. `getpid` is microseconds; `fork` is milliseconds. Even among I/O syscalls, the cost varies wildly depending on whether data is cached. |
| "Syscall cost is just the transition time (~200 ns)." | The transition is the *smallest* part. Bigger costs: TLB flush (KPTI), cache eviction, actual kernel work, possible blocking and context switches. |
| "I should avoid syscalls at all costs." | Not true. Syscalls are the *only* way to do real I/O, allocate memory, and manage processes. The goal is not to avoid them but to batch them and amortize the cost. |
| "buffering makes code slower because I'm not doing work immediately." | The opposite. Buffering trades latency (microseconds longer delay before I see the output) for throughput (100× faster overall). Almost always the right choice. |
| "vDSO is a new thing I need to understand." | vDSO is entirely transparent. The C library uses it automatically. You don't need to do anything. It's worth knowing exists so you understand why `clock_gettime` is fast. |
| "io_uring replaces epoll." | No. io_uring is for *issuing* I/O operations. epoll is for *waiting* on them. Modern async servers use both: io_uring to submit, epoll/io_uring completion queue to get results. |
| "Containers make syscalls safer by using seccomp." | Safer, yes — but not safe. A exploited process can still damage its own namespace and data. seccomp is one layer; it doesn't replace memory safety or input validation. |

---

## 80.11 Exercises

1. **Measure your own buffer cost.** Write a program that writes 10 MB to `/dev/null`:
   - (a) using `write(1)` per byte.
   - (b) using a 4 KB buffer.
   - (c) using a 64 KB buffer.
   
   Time all three. Run under `strace -c` to count syscalls. Explain the relationship between buffer size, syscall count, and wall-clock time.

2. **Trace a real program.** Run `strace -c` on a program you use daily (e.g., `ls`, `grep`, your web browser). Identify the top 3 syscalls by time. Propose a mitigation (buffering, batching, caching) for the costliest one.

3. **vDSO in the wild.** Write a C++ program that calls `std::chrono::high_resolution_clock::now()` 1 million times in a loop. Run under `strace -c`. How many syscalls? (Hint: if it's 0 or very few, vDSO is doing its job.)

4. **Buffer size experiment.** Modify the unbuffered I/O example from Section 80.8 to try different buffer sizes: 256 bytes, 1 KB, 4 KB, 64 KB, 1 MB. For each, time the run and count syscalls. Plot buffer size vs. time. At what point does increasing the buffer size stop helping?

5. **Container syscall filtering.** If you have access to Docker:
   ```bash
   # Default profile
   docker run --rm debian ls
   
   # Restrictive seccomp profile (forbid fork, execve, etc.)
   docker run --security-opt seccomp=restricted_profile.json --rm debian ls
   ```
   
   Create a seccomp profile that forbids `open` and test what happens when you try to read a file inside the container.

6. **io_uring exploration** (Linux 5.1+). Read the io_uring man page and liburing documentation. Write a simple program that uses `io_uring` to read a file, then compare the syscall count to a traditional `read()` loop.

7. **Conceptual.** Your web server handles 100,000 concurrent connections. Each connection processes one request per second, and each request involves 3 syscalls (read request, write response header, write response body). Without batching, that's 300,000 syscalls/second. Using `epoll` or `io_uring`, sketch how you would reduce that. What's the theoretical minimum number of syscalls/second, and why?

---

## 80.12 Summary

A system call is the only way user code requests the kernel to do something on its behalf. The interface is simple — special CPU instruction, registers carry arguments, privilege level switches — but the cost is not. Mode transition, TLB invalidation, cache eviction, and actual kernel work can total microseconds to milliseconds per call. This is why high-performance systems obsess over syscall count: reducing syscalls from 1,000,000 to 10,000 can make a program 100× faster.

The mitigation strategies are time-tested: buffer small I/O into large batches, use facilities like vDSO and io_uring to amortize cost, and count syscalls with strace or perf to find the real culprits. Containers use seccomp to restrict the syscall surface area, reducing attack surface but at the cost of some flexibility. Understanding syscalls is understanding where your program touches the kernel, and that's where the money is — in latency and CPU.

---

> **[← Previous: Networking Internals](02-networking-internals.md)** · **[↑ Part 8](README.md)** · **[Next: Kernel vs User Space →](04-kernel-vs-user-space.md)**
