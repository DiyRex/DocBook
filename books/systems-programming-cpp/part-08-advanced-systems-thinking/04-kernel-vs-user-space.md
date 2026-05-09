# Chapter 80 — Kernel vs User Space

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the **privilege levels** that modern CPUs enforce (rings on x86, exception levels on ARM) and why the boundary between user and kernel space is the most important security mechanism in any OS.
2. Describe the three ways execution transitions from user to kernel: **system calls** (voluntary), **interrupts** (hardware-driven), and **exceptions** (faults like page faults).
3. Identify what lives in user space (your code, libraries, runtimes) vs kernel space (scheduler, memory manager, device drivers, networking stack).
4. Quantify the **cost of crossing the boundary** — mode switch overhead, cache disturbance, TLB flushes — and recognize when it matters.
5. Compare **monolithic kernels** (Linux, BSD: most services in the kernel) to **microkernels** (Mach, seL4, QNX: minimal kernel, services in user space) and understand the tradeoff.
6. Understand **eBPF** — how to run filtered kernel code to avoid boundary crossings for observability and networking.
7. Recognize privilege escalation attack patterns and why hardening the kernel boundary is a never-ending project.

This chapter completes the mental model started in Chapter 3 (How Operating Systems Execute Programs). You now know that the privilege boundary exists; here we explore *why it exists, what it costs, and how to think about it in real systems.*

---

## 4.1 The Privilege Boundary: Why It Exists

The modern CPU is not a generic machine that your code controls. It is a **privileged machine** — the CPU itself enforces different privilege levels. On x86-64, there are four "rings" (0, 1, 2, 3), though only rings 0 (kernel) and 3 (user) are used in practice. On ARM, there are exception levels: EL0 (user), EL1 (kernel), EL2 (hypervisor), EL3 (secure firmware).

The CPU enforces two critical rules:

1. **Certain instructions are privileged.** Only code running in ring 0 (kernel mode) can:
   - Load or modify the page table register.
   - Halt the CPU (`hlt`).
   - Mask or unmask interrupts.
   - Talk to I/O hardware via `in`/`out` instructions (x86) or memory-mapped I/O.
   - Switch between privilege levels.

2. **Certain memory regions are privileged.** The kernel can mark pages "kernel-only." If user code tries to read or write them, the CPU raises an exception.

This boundary exists for one reason: **isolation and stability**. A buggy user program cannot, through any sequence of instructions it can execute, crash the machine. The worst it can do is crash itself. The kernel is the only code that can crash the entire system — and the kernel is heavily tested and (in theory) carefully written.

### The tragedy of no privilege boundary

Before virtual memory and privilege rings, operating systems like early Unix ran everything in the same privilege level. Result: any buggy program could write to arbitrary memory, including the kernel's data structures. A single wild pointer could corrupt the scheduler, the file system, the device drivers — everything. The system was fragile.

Modern OSes inverted this: **assume user code is buggy, protect against it.**

---

## 4.2 Three Paths Into the Kernel

User code can enter the kernel in three ways. Each has different performance and semantics.

### Path 1: System Call (Voluntary)

A system call is a **deliberate transition** from user to kernel initiated by user code. Examples: `read(fd, buf, 4096)`, `write(fd, data, len)`, `fork()`, `execve()`, `mmap(addr, size, prot, flags, fd, offset)`.

Mechanism (x86-64):
1. User code loads syscall arguments into registers (rdi, rsi, rdx, rcx, r8, r9 per ABI).
2. Loads the syscall number into `rax`.
3. Executes the `syscall` instruction.
4. **CPU transitions to ring 0** at a fixed entry point set by the kernel (the syscall handler).
5. Kernel checks the syscall number, validates arguments, performs the work.
6. Kernel executes `sysret` (or `iretq` on older x86), transitioning back to ring 3.
7. User code resumes from the instruction after the `syscall`.

The transition itself is ~50–200 nanoseconds on modern hardware. But many syscalls do real work — a `read` might involve disk I/O (millions of cycles), a `fork` might copy page table structures (thousands of cycles).

### Path 2: Hardware Interrupt (Asynchronous)

When a peripheral (disk, network, timer) needs the CPU's attention, it raises an **interrupt**. The CPU:

1. Completes the current instruction.
2. Checks for pending interrupts.
3. If present and interrupts are enabled, the CPU (not the program) transitions to kernel mode at a fixed entry point set by the kernel (the interrupt handler vector).
4. The kernel saves the user context (registers, return address, mode) on the kernel stack.
5. The kernel handles the interrupt — e.g., reads data from the network card, or marks a sleeping process runnable.
6. The kernel restores the user context and transitions back to ring 3.

This is **asynchronous** — it can happen between any two instructions in user code. The user program doesn't request it; the hardware demands it.

Common interrupts: timer (preemption), network packet arrival, disk I/O completion.

### Path 3: Exception (Synchronous Fault)

An **exception** is a fault caused by an instruction in user code. Examples: page fault, illegal instruction, division by zero, segmentation fault.

When an exception occurs:

1. The CPU transitions to ring 0 at the appropriate handler (e.g., `#PF` for page fault).
2. The kernel examines the faulting address / instruction.
3. The kernel either **fixes it** (e.g., faults in a page from disk), **kills the process** (e.g., invalid instruction), or passes it to user code as a signal.

The difference from a syscall: the user code did *not intend* to trap. The CPU detected a fault and forced a transition.

### Summarizing the three paths

| Path | Initiator | Timing | Example |
|------|-----------|--------|---------|
| System call | User code (voluntary) | Synchronous | `read(fd, buf, 4096)` |
| Interrupt | Hardware peripheral | Asynchronous | Timer fires, network packet arrives |
| Exception | CPU (instruction fault) | Synchronous (but unintended) | Page fault, segfault |

All three save the user context, transition to ring 0, execute kernel code, and restore the context.

---

## 4.3 What Lives Where

The boundary between user and kernel space is not symmetrical. Different things live on each side.

### User Space

- **Your application code** (C++, Python, Rust, etc.).
- **Standard libraries** — libc (glibc, musl, etc.), C++ STL, language runtimes.
- **Third-party libraries** — everything you `#include` or `import`.
- **The heap** (malloc'd memory).
- **The stack** (local variables, return addresses).
- **Memory-mapped libraries** (e.g., `.so` files shared across processes).

User space code cannot:
- Access kernel memory without syscalls.
- Execute privileged instructions.
- Directly talk to hardware.

But user space can:
- Call syscalls to ask the kernel to do things.
- Use libc wrappers that hide syscall plumbing.
- Register signal handlers to react to asynchronous events.
- Use threads (which are kernel-scheduled).

### Kernel Space

- **The process scheduler** — decides which process runs on which CPU.
- **Memory manager** — manages page tables, page cache, swapping.
- **Interrupt handlers** — respond to hardware events.
- **Device drivers** — talk to disks, NICs, USB, etc.
- **Networking stack** — TCP/IP implementation, socket layer.
- **File systems** — ext4, NTFS, btrfs, etc.
- **Virtual file systems** (procfs, sysfs) — present kernel information as files.

Kernel code can:
- Execute privileged instructions.
- Access all memory (page tables, process data structures).
- Interrupt user code at any time.

Kernel code should *not* crash or hang, because if it does, the whole system goes down.

### The Boundary: libc and System Call Wrappers

The C standard library (libc) is the translator. When you call `read()` from C:

```cpp
#include <unistd.h>

ssize_t result = read(fd, buffer, 4096);
```

You are not directly executing the `syscall` instruction. Instead:

1. The libc wrapper function `read()` sets up the arguments.
2. Executes the `syscall` instruction.
3. The kernel runs `sys_read()`.
4. The kernel returns the result in `rax`.
5. libc unpacks it and returns to your code.

This is why knowing that libc is *on top of* syscalls is crucial: when debugging I/O performance, you must know whether the bottleneck is in libc buffering, in your code, or in the kernel. Chapter 23 (Async Runtime Internals) goes deeper.

---

## 4.4 The Cost of Crossing

A mode transition is not free. Let's quantify it.

### Direct cost: The transition itself

- **Best case**: ~50 nanoseconds on modern x86-64 (just the CPU instruction and entry into the kernel handler).
- **Syscall return**: another ~50 nanoseconds.
- **Total**: ~100–200 nanoseconds for the round-trip if the kernel does nothing.

By comparison, an in-process function call costs ~1 nanosecond. A syscall is thus **100–200× slower** than a function call, even if the kernel does no work.

### Indirect cost: Cache disturbance

When the kernel runs, it executes its own code and accesses its own data (page tables, process tables, device drivers). This:

- Evicts your application's code and data from the CPU's L1/L2/L3 caches.
- Pollutes the **instruction cache** with kernel code.
- Dirties the **data cache** with kernel data structures.

When your code resumes, the first few thousand instructions after the syscall run slower because the cache is cold. On a heavily-used system with many syscalls, this indirect cost can exceed the direct cost.

### TLB flush cost

On some architectures and with some kernel configurations, a mode transition invalidates the **TLB** (translation lookaside buffer — the CPU's cache of virtual-to-physical address mappings). This is because:

- User-space addresses are in the user part of the TLB.
- Kernel-space addresses are in the kernel part.
- Some CPUs segregate them; others don't.

A full TLB flush costs tens to hundreds of nanoseconds. If this happens per syscall, syscall-heavy workloads suffer badly.

Modern kernels use **KPTI** (Kernel Page Table Isolation) on x86 to mitigate Spectre/Meltdown vulnerabilities. KPTI separates user and kernel page tables more completely, which helps security but can increase TLB pressure.

### When it matters

Syscall cost is **amortized** across useful work:

- A `read(fd, buf, 65536)` syscall does 100–200 nanoseconds of transition + possibly milliseconds of disk I/O. The 100 ns is noise.
- A `write(fd, buf, 1)` syscall does 100–200 nanoseconds of transition + microseconds of buffering + possibly I/O. Still amortized.
- A tight loop doing `for(int i=0; i<1000000; ++i) sys_something()` where each syscall does no real work? The cost is **not** amortized. This is why production code avoids tight syscall loops.

**Practical example**: Chapter 3's exercise had you compare:

```cpp
for (int i = 0; i < 10000; ++i) std::printf("%d\n", i);
```

vs

```cpp
for (int i = 0; i < 10000; ++i) {
    std::printf("%d\n", i);
    std::fflush(stdout);
}
```

The second version does 10,000 `write` syscalls (one per line). Each syscall is ~100–200 ns of overhead, but the real cost is cache disturbance and thread preemption. It's dramatically slower.

The first version buffers in user space and flushes maybe 10 times. It amortizes the syscall cost.

---

## 4.5 Monolithic Kernels vs Microkernels

Operating systems differ in how much code runs in kernel space vs user space. This is a fundamental architectural choice.

### Monolithic Kernel

In a **monolithic kernel** (Linux, BSD, Windows), nearly all OS services run in kernel space:

- Scheduler, memory manager, page cache.
- File system (ext4, NTFS, btrfs).
- Networking stack (TCP/IP).
- Device drivers (disk, network, USB).
- Inter-process communication (pipes, sockets, shared memory).

**Pros:**
- **Performance.** One transition into the kernel; services communicate with function calls, not syscalls. File system reads another service; no mode switches.
- **Simplicity** (at the kernel level; overall the kernel is huge).
- **Maturity.** Linux, BSD kernels are battle-tested over 30+ years.

**Cons:**
- **Stability.** A bug in a device driver runs in ring 0. A badly-written file system can crash the entire kernel. One bad driver corrupts the whole system.
- **Size.** The Linux kernel is millions of lines of code. More code = more bugs.
- **Security.** A privilege escalation vulnerability in any driver or subsystem compromises the whole kernel.

### Microkernel

In a **microkernel** (Mach, seL4, QNX), the kernel does *only* the essentials:

- **Only** the scheduler, basic virtual memory, inter-process communication (message passing).
- Everything else (file system, drivers, networking) runs in **user-space processes**.

When a user process needs a file system operation, it:

1. Sends a message (syscall) to the file system process (another user process).
2. The file system process reads the message, does the work, sends a reply message.
3. The original process wakes up with the result.

**Pros:**
- **Stability.** A buggy file system or driver crashes that service, not the whole OS. Other processes keep running.
- **Security.** Privilege escalation is harder because fewer components run in ring 0. Services run with minimal privileges.
- **Modularity.** You can replace the file system, the network driver, etc., without restarting the kernel.

**Cons:**
- **Performance.** Every service call is 2–4 syscalls (message send, service execution, message receive). The cache disturbance is significant.
- **Complexity (user-facing).** You have more separate processes and services to manage.
- **Less mature.** Fewer production deployments; less battle-testing.

### In practice

- **Linux, BSD, Windows** are monolithic. They dominate.
- **QNX, seL4** are used in safety-critical systems (automotive, avionics) where stability and isolation matter more than performance.
- **macOS** uses a hybrid: mostly monolithic XNU kernel, but some services (audio, graphics) are in user space.

The monolithic/microkernel tradeoff is **performance vs isolation**. Modern CPU speeds have made monolithic kernels' performance advantage overwhelming. Microkernels are research artifacts and niche use cases.

---

## 4.6 eBPF — Programmable Kernel

In the last decade, a new approach has emerged: **eBPF** (extended Berkeley Packet Filter). Instead of putting code in the kernel or user space, eBPF lets you write *filtered code that runs in the kernel* without being part of the kernel.

### How eBPF works

An eBPF program is:

1. **Written in C or bytecode**, compiled to eBPF instructions.
2. **Verified by the kernel** to ensure it doesn't access invalid memory or run infinite loops.
3. **Attached to a kernel event** — a tracepoint (e.g., `open` syscall entry), a network packet filter, a kprobe, a user-defined event.
4. **JIT-compiled to native machine code** by the kernel.
5. **Executed in kernel space** when the event fires.

Unlike a driver, an eBPF program is:
- **Safe** (verified before execution; can't crash the kernel).
- **Dynamic** (loaded/unloaded without restarting the kernel).
- **Sandboxed** (can access only approved data structures).

### Use cases

**Observability:**
```bash
# Trace all open() syscalls system-wide
bpftrace -e 'tracepoint:syscalls:sys_enter_open { 
    printf("pid %d opened %s\n", pid, arg1); 
}'
```

Every `open` syscall triggers the eBPF program. The kernel collects the PID and filename and prints them, all in kernel space. No user-space overhead. This is how tools like `bcc`, `bpftrace`, and `Cilium` work.

**Networking:**
Cilium (a container networking system) uses eBPF to implement network policies at the kernel level, avoiding the overhead of user-space proxies.

**Security:**
Detect anomalous syscalls (e.g., a web server suddenly trying to read `/etc/shadow`). The eBPF program runs on every syscall entry; if it detects something bad, it can block the syscall or kill the process.

### Why it matters

eBPF **reduces boundary crossings**. Normally, observability tools:

1. User-space program (e.g., Prometheus exporter) wakes up.
2. Makes syscalls to read `/proc` or call system calls to gather data.
3. Aggregates data in user space.
4. Writes result to disk/network.

With eBPF:

1. Kernel event fires.
2. eBPF program runs in-kernel, aggregates data.
3. Only the final result (a few metrics) is copied to user space.

**Fewer boundary crossings = less cache disturbance, less context switching, lower latency.**

---

## 4.7 Privilege Escalation and the Hardening Game

The kernel/user boundary is the target of every attacker. Exploits aim to transition from user mode (which is what an attacker's code runs in) to kernel mode with malicious intent.

### Common privilege escalation paths

1. **Kernel bug.** A memory safety bug in the kernel (buffer overflow, use-after-free) allows user code to corrupt kernel structures and take control. Example: CVE-2022-0847 (Dirty Pipe in Linux).

2. **Privileged daemon.** A user runs a daemon as root (e.g., a web server, database). A bug in the daemon (e.g., SQL injection, path traversal) lets an attacker execute arbitrary code as root.

3. **Container escape.** A container (Docker, Kubernetes) runs unprivileged code. A bug in the kernel's container isolation (namespace/cgroup implementation) lets the container code escape and access the host kernel.

4. **Suid binary.** A binary marked `suid` runs as root even when executed by a normal user. A bug in that binary (e.g., unsanitized environment variable, unsafe `system()` call) lets an attacker execute code as root.

5. **Supply chain.** Malicious code in a library or build tool runs at build time or in CI/CD. It modifies the kernel, drivers, or privileged utilities.

### Hardening strategies

1. **Principle of least privilege.** Run services with the minimal required privileges. A web server doesn't need to be root.

2. **Seccomp.** Use `seccomp` (secure computing) to restrict which syscalls a process can make. A web server might deny `execve`, `fork`, direct disk access. If a web server is compromised, the attacker can't execute arbitrary code.

3. **AppArmor / SELinux.** Mandatory access control (MAC). Even if a root process is compromised, it can only access files and resources explicitly allowed by policy.

4. **Kernel hardening.** SMEP (Supervisor Mode Execution Prevention): user-space code can't be executed from kernel mode. SMAP (Supervisor Mode Access Prevention): kernel can't read/write user-space memory without explicit kernel-to-user copy functions.

5. **Memory safety in the kernel.** Rust in the Linux kernel (experimental) to avoid use-after-free and buffer overflows.

The game is endless. Each defense leads to new attack vectors. Modern OS security is a cat-and-mouse game played by kernel developers and exploit researchers.

---

## 4.8 Worked Example: Tracing syscalls with bpftrace

Let's trace every `open(2)` syscall on the system and see which processes are opening files.

**Setup:** Install `bpftrace` on Linux:
```bash
sudo apt install bpftrace  # Debian/Ubuntu
# or
sudo yum install bpftrace  # RHEL/CentOS
```

**Script:**
```bash
sudo bpftrace -e '
    tracepoint:syscalls:sys_enter_open {
        printf("%-16s %6d  %s\n", comm, pid, str(arg0));
    }
'
```

**What happens:**
1. When any process enters the `open()` syscall, the kernel fires the `sys_enter_open` tracepoint.
2. The eBPF program attached to that tracepoint runs in kernel space.
3. The program prints the command name (`comm`), PID, and the filename being opened (`arg0`).
4. The kernel copies the output to the user-space `bpftrace` process and displays it.

**Output:**
```
chrome          12345  /home/user/.config/chrome/history
bash            54321  /etc/passwd
nginx           20000  /var/log/access.log
sshd            15000  /etc/ssh/sshd_config
```

Run this on a busy system for 10 seconds. You'll see hundreds of `open` syscalls. Each one is a kernel/user boundary crossing. This demonstrates:

- How many boundary crossings a modern system makes (thousands per second).
- Which processes are most syscall-heavy (databases, web servers).
- The value of eBPF: you got this information with *one* user-space command, not by instrumenting each application.

---

## 4.9 Tradeoffs

| Design Choice | Pros | Cons | When to Choose |
|---|---|---|---|
| Monolithic kernel | Fast; mature | One bad driver crashes system | Desktop, server OSes (Linux, BSD) |
| Microkernel | Stable; modular; secure | Slower; message-passing overhead | Safety-critical (avionics, automotive) |
| eBPF observability | No performance cost; dynamic | Limited to kernel events; complex debugging | High-performance systems needing insight |
| Seccomp / AppArmor | Fine-grained isolation | Complex policy management; false positives | Containers, multi-tenant systems |
| User-space networking stack | Full control; visibility | Slower; reimplements TCP/IP | Research, niche applications |
| Kernel-space networking | Fast; proven | Hard to modify; one bug affects all | Production (Linux kernel's netdev) |

---

## 4.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "The kernel is a special program that runs on the CPU." | The kernel is code that runs at privilege level 0 (ring 0). The CPU itself enforces what instructions can run at each level. |
| "Syscalls are just function calls that happen to be slow." | Syscalls cross a privilege boundary, which involves CPU mode transitions, cache flushes, and context saves. Function calls do not. |
| "eBPF is code that runs on the kernel, so it must run as fast as native kernel code." | eBPF runs in the kernel (ring 0) but is JIT-compiled and verified, so there's some overhead. But it's faster than user-space syscalls. |
| "A microkernel is 'better' because it's modular." | Microkernels trade performance for modularity. They're useful only in domains where stability and isolation matter more than speed. |
| "I should minimize syscalls to improve performance." | Right idea, wrong metric. *Minimize boundary crossings per unit of work.* If a syscall does 1 MB of I/O, the overhead is noise. If it does 1 byte, it's wasteful. |
| "The kernel has full control of the CPU." | The CPU has rules that both kernel and user code must obey. The kernel runs at a higher privilege level, but even it can't violate the CPU's architecture (e.g., it can't execute from user-space memory if SMEP is enabled). |
| "eBPF is a security hole because it runs arbitrary code in the kernel." | eBPF is verified before execution. It can't dereference invalid pointers or run infinite loops. It's safer than a driver. |

---

## 4.11 Exercises

1. **Measure syscall cost.** Write a tight loop that does 1 million `getpid()` syscalls (which do no real work). Measure the wall-clock time. Now write a loop that caches the PID and calls `getpid()` once. Compare. The difference is pure syscall overhead.

2. **Trace your shell.** Run:
   ```bash
   sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[name] = count(); }' 
   ```
   while typing shell commands in another terminal. Which syscalls dominate? What does this tell you about what the shell does?

3. **Container syscalls.** Run a Docker container and:
   ```bash
   sudo strace -c docker run alpine /bin/sh -c "ls /tmp"
   ```
   Count syscalls. Now run the same command outside a container and compare. How many extra syscalls does the container runtime add?

4. **Privilege boundary tester.** Write a small C program that:
   - Tries to execute a privileged instruction (e.g., `hlt` via inline asm).
   - Tries to read kernel memory (e.g., `*(int*)0xffffffff00000000`).
   - Tries to change the page table directly.
   What error do you get? Is it from the CPU, the kernel, or libc?

5. **eBPF trace.** Install `bpftrace`. Trace the `execve` syscall and print the command and arguments every time a new process starts:
   ```bash
   sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { 
       printf("exec: %s\n", str(arg0)); 
   }'
   ```
   Run it for 10 seconds and observe. How many new processes are created in a typical work session?

6. **Scheduler activity.** On Linux, run:
   ```bash
   vmstat 1
   ```
   Look at the "cs" (context switches) and "in" (interrupts) columns. How many context switches per second is your system doing right now? What does that tell you about the syscall rate?

7. **Conceptual.** A web server serves 10,000 requests/sec. Each request involves ~50 syscalls (read, write, sendfile, etc.). Estimate the CPU time spent in syscall overhead alone (assume 100 ns per syscall transition). Is this a performance concern? Why or why not?

---

## 4.12 Summary

The privilege boundary between user space and kernel space is the central security mechanism of modern operating systems. User code runs in ring 3 (unprivileged); the kernel runs in ring 0 (privileged). Three mechanisms transition between them:

- **System calls** (user-initiated, voluntary).
- **Interrupts** (hardware-initiated, asynchronous).
- **Exceptions** (CPU-detected faults).

Each transition costs ~100–200 nanoseconds of direct overhead, plus indirect costs (cache disturbance, TLB pressure). For syscalls that do little work, this overhead is significant; for syscalls that do real work (disk I/O, memory allocation), the overhead is amortized.

Operating systems differ in where they draw the user/kernel boundary: **monolithic kernels** (Linux, BSD) put most services in ring 0 for speed; **microkernels** (QNX, seL4) minimize ring 0 for stability and security. **eBPF** offers a middle ground: dynamically loaded, verified code that runs in the kernel without being part of the kernel itself.

Hardening the boundary against privilege escalation is a never-ending game. The kernel is a high-value target; a single bug can compromise the entire system.

---

**[← Previous: Chapter 80 — Syscalls](03-syscalls.md)** · **[↑ Part 8](README.md)** · **[Next: Chapter 81 — Serialization Internals →](05-serialization-internals.md)**
