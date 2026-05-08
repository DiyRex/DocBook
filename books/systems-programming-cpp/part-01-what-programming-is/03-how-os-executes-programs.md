# Chapter 3 — How Operating Systems Execute Programs

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a **process** is, how it differs from "a running program," and what state the kernel maintains about every process.
2. Describe **virtual memory** — how each process gets the illusion of owning the entire address space, and how the MMU and page tables make this real.
3. Define **kernel mode vs user mode** and explain why the privilege boundary is the central security mechanism of a modern OS.
4. Trace what happens during a **system call**: the transition from your code into the kernel, what work the kernel does, and how it returns.
5. Describe the **scheduler**'s job and what a **context switch** actually costs.
6. Identify the four "everything is a..." abstractions modern OSes give you — files, processes, virtual memory, sockets — and recognize why they make systems thinking tractable.

This chapter is where most developers' confusion about "the system" gets resolved. After this, when somebody talks about "blocking I/O," "context switches," "fork-bomb," "shared memory," or "system call overhead," you will have a concrete picture of what is happening.

---

## 3.1 What an Operating System Actually Does

If you boil it down, an OS does three things:

1. **Multiplexes** scarce hardware resources (CPU cores, RAM, disks, network) across many programs that all believe they own the whole machine.
2. **Protects** programs from each other and from themselves, so a bug in one program does not corrupt another, and so user programs cannot accidentally talk to hardware in ways that crash the system.
3. **Abstracts** wildly different hardware (disks of all kinds, network cards, GPUs, keyboards) behind small, uniform interfaces (`read`, `write`, `open`, `close`, `mmap`, `socket`).

These three jobs explain almost every design choice in any kernel. The rest of this chapter expands them.

---

## 3.2 The Process: The Kernel's Bookkeeping for a Running Program

In Chapter 1 we said: "running a program" means the kernel constructs a *process*. Let's now look at what a process actually *is* from the kernel's point of view.

A process is a kernel data structure (on Linux: `struct task_struct`) — a record. Each process has:

- A unique **PID** (process ID).
- A **page table** describing its private virtual address space.
- A list of open **file descriptors** (small integers like `0`, `1`, `2` that map to kernel objects: open files, sockets, pipes, etc.).
- A set of **registers** that hold its CPU state when it's not currently running.
- A **parent process** (every process except `init`/`launchd`/`systemd` was forked by some parent).
- **Credentials**: which user owns it, which groups it belongs to, what capabilities it has.
- **Signal handlers** and a **pending signals** queue.
- Various scheduler state (priority, nice value, last time it ran, etc.).

When the kernel says "process P is running on CPU 0," what it means is: *the page table for process P is loaded into the MMU, P's saved registers are loaded into the CPU, and the CPU is executing instructions from P's address space.*

Switching from one process to another — a **context switch** — is exactly: save current registers into the old process's task_struct, change the page table, load the new process's registers, jump.

This is not free. A context switch on a modern Linux system costs roughly 1–10 microseconds, plus the indirect cost: the new process's working set is not in the CPU caches, so the first few thousand instructions after a switch will run slowly while the cache warms up. That indirect cost is often larger than the direct cost. **A high context-switch rate is one of the classic causes of systems that "feel slow" without obviously doing anything expensive.**

### Threads: Processes' Lighter Cousin

A **thread** is, at the kernel level on Linux, almost the same thing as a process — it's another `task_struct`. The difference: threads in the same program *share* a page table (and therefore all memory) and *share* the file descriptor table; processes do not.

This is why threads are cheaper than processes: less to set up, more to share. It is also why threads are dangerous: there is no memory protection between threads of one process. A wild pointer in one thread can corrupt another thread's data, and the OS will do nothing to stop it.

We will dedicate Chapter 19 to processes vs threads. For now, internalize: **a thread is just a process that shares memory with its siblings.** All the scheduling rules are the same.

---

## 3.3 Virtual Memory: The Most Important Lie the OS Tells

Every process believes it owns the address space from `0x0` to `0xFFFFFFFFFFFFFFFF`. Two processes can both store something at address `0x400000` and not collide. How?

The answer is **virtual memory**. Every address your code touches is a *virtual address*. Before reaching the RAM chip, it passes through the **MMU** (memory management unit), a piece of hardware that translates virtual addresses to physical addresses using a per-process **page table** maintained by the kernel.

### The mechanism

Memory is divided into **pages** (typically 4 KB, sometimes 16 KB on Apple Silicon, sometimes 2 MB or 1 GB "huge pages"). The page table is a tree indexed by parts of the virtual address; the leaves are **page table entries (PTEs)** that say:

- **Is this virtual page mapped?** (If not: a page fault — the kernel decides what to do.)
- **What physical page does it map to?**
- **What permissions?** (Read / write / execute, in any combination.)
- **Other flags** (was it accessed? was it modified? is it shared?)

Each memory access goes:

```
virtual address  ->  MMU  ->  physical address  ->  RAM
                       |
                       walks page table; uses TLB cache
```

The **TLB** (translation lookaside buffer) is a tiny CPU cache of recent virtual-to-physical mappings. A TLB hit costs nothing. A TLB miss costs a page-table walk, which can be a few extra memory accesses. A page fault — when the page is not mapped at all — costs *millions* of cycles, because the kernel has to step in.

### What virtual memory buys you

Once you have this machinery, you can do extraordinary things:

- **Process isolation.** Process A's page table simply does not contain mappings for process B's pages. A wild pointer in A cannot reach B's memory because A doesn't have a way to even *name* B's physical pages.
- **Demand paging.** When you `mmap` a 1 GB file, the kernel doesn't read 1 GB into RAM. It just sets up the page table to *say* it's mapped. The pages are loaded from disk lazily on first access. Your program "sees" a 1 GB array; the kernel transparently faults pages in as needed.
- **Copy-on-write.** When `fork()` clones a process, the kernel does not copy memory. It marks every page **read-only** in *both* parent and child. The first time either one tries to write, a page fault triggers; the kernel makes a private copy *just for that page* and unmaps the read-only mapping. Pages that are only read are never copied. This is why `fork` is fast even for huge processes.
- **Shared libraries.** `libc.so` is mapped into every process's address space. The *physical* memory holding it is one copy; each process's page table maps to that one copy. 100 processes use libc; libc takes one copy of RAM.
- **Mapped files.** `mmap` lets you treat a file as memory. The kernel keeps the page cache and your program's view consistent. It's how databases like SQLite and editors like Emacs work efficiently with large files.
- **Zero-fill on demand.** A new process's `.bss` and freshly-allocated heap pages are not actually zeroed up front. The kernel hands you a single shared "zero page," read-only, mapped wherever you "have" zeroes. The first time you write, it copies. Cheap.

Virtual memory is one of the great engineering achievements of computing. Almost every modern programming convenience — fast `fork`, shared libraries, `mmap`-based databases, GC heaps that grow lazily, JITs that allocate executable pages — depends on it.

### Experiment 3.1 — See your own page table

```bash
# Linux
cat /proc/self/maps

# macOS
vmmap $$ | less
```

You will see lines like:

```
55a4d5c00000-55a4d5c01000 r--p 00000000 08:01 12345  /usr/bin/cat
55a4d5c01000-55a4d5c02000 r-xp 00001000 08:01 12345  /usr/bin/cat
55a4d5c02000-55a4d5c03000 r--p 00002000 08:01 12345  /usr/bin/cat
55a4d5c03000-55a4d5c04000 rw-p 00003000 08:01 12345  /usr/bin/cat
55a4d5c04000-55a4d5c25000 rw-p 00000000 00:00 0      [heap]
...
7ffe...                rw-p 00000000 00:00 0          [stack]
```

Every range is a contiguous virtual region with uniform permissions. `r-xp` is "read-execute, private" — that's code. `rw-p` is "read-write, private" — that's data and the heap. The fields in order: virtual address range, permissions, file offset, device, inode, file name. Read it slowly. *This is your process's address space, exactly.*

---

## 3.4 Kernel Mode and User Mode

The CPU itself has a notion of **privilege level**. On x86 there are four "rings" (0–3) but only two are used in practice: ring 0 (kernel) and ring 3 (user). On ARM there are similar "exception levels" (EL0 = user, EL1 = kernel, EL2 = hypervisor, EL3 = secure firmware).

The CPU enforces:

- Certain instructions can *only* execute in ring 0. (Examples: load page table, halt CPU, talk directly to I/O hardware via `in`/`out` instructions, change interrupt mask.)
- Certain memory regions can be marked "kernel only." User-mode code that touches them traps.

This is the *only* mechanism that prevents your `./hello` program from, say, formatting your hard drive. The instructions to talk to the disk controller are privileged. Your code cannot execute them. The kernel can.

When user code needs the kernel to do something — read a file, allocate memory from the OS, send a network packet — it must transition into kernel mode. That transition is a **system call**.

---

## 3.5 The Anatomy of a System Call

A system call is the controlled doorway between user code and kernel code. Concretely, on x86-64 Linux:

1. Your code wants, say, `write(1, "hi\n", 3)`. The C library wraps this in a function that:
2. Loads `1`, `"hi\n"`'s address, and `3` into the registers `rdi`, `rsi`, `rdx` (per ABI).
3. Loads the system call number for `write` (`1`) into `rax`.
4. Executes the `syscall` instruction.
5. The CPU **transitions to ring 0** at a fixed entry point set up by the kernel (the syscall handler).
6. The kernel reads `rax` to learn which syscall is being requested, and dispatches into its `sys_write` implementation.
7. `sys_write` does its work — possibly involving the page cache, possibly the disk, possibly a network socket if FD 1 is a pipe — and writes a return value into `rax`.
8. The kernel executes `sysret` (or `iretq`), which transitions the CPU back to ring 3.
9. Your code receives the return value in `rax` and continues.

There are roughly 300+ syscalls on Linux (`man syscalls` lists them). Common ones: `read`, `write`, `open`, `close`, `mmap`, `brk`, `fork`, `execve`, `wait4`, `clone`, `nanosleep`, `socket`, `bind`, `accept`, `connect`, `sendto`, `recvfrom`, `epoll_*`, `futex`, `pipe`, `dup2`, `fstat`, `getpid`, `gettimeofday`. You don't memorize them, but recognizing categories matters: "I/O," "memory," "process control," "synchronization," "time," "networking."

### What a syscall costs

Best case: ~50–200 nanoseconds for the transition itself. That sounds small, but compared to in-process function calls (~1 ns), it's 50–200× slower. Plus, many syscalls do real work that takes much longer.

This is why high-performance code obsesses over syscall counts:

- **Buffered I/O.** Your `printf` does not call `write` for each character; it accumulates in a userspace buffer and flushes a kilobyte at a time. The `syscall` overhead is amortized.
- **Batch APIs.** `writev`, `sendmmsg`, `epoll_wait` exist precisely so you can hand the kernel a batch of work in one syscall instead of N.
- **Memory mapping over read.** Instead of calling `read` repeatedly to get a file's contents, `mmap` the file once and access it as memory. The kernel still moves bytes from disk, but you pay no per-access syscall.
- **io_uring (Linux), kqueue (BSD/macOS).** These allow you to submit many I/O requests and reap many completions with very few syscalls.

You will see this pattern recur throughout the book: **the boundary between layers is expensive; reduce trips across it.** Whether the layers are user/kernel, application/database, or service/service, the principle is the same.

### Experiment 3.2 — count your syscalls

```bash
# Linux: count which syscalls a program makes
strace -c ls

# macOS: dtruss with -c
sudo dtruss -c ls 2>&1 | tail
```

Run it on `ls`. You will see something like (Linux):

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 32.10    0.000223           7        29           getdents64
 19.29    0.000134           4        29           write
 ...
```

Now count `write`s. Notice how few there are despite `ls` printing many filenames — the C library is buffering.

Now write a program that does:

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

Time both. The second is dramatically slower — that's syscall overhead, fully exposed.

---

## 3.6 Scheduling: Many Programs, Few CPUs

There are usually hundreds of processes on a system, but only a few CPU cores. The **scheduler** decides which process runs on which core at any moment.

The classic scheduler model:

- Each process is in one of: **running**, **runnable** (ready, waiting for CPU), **sleeping** (blocked on I/O, a lock, a sleep), **stopped**, or **zombie** (exited but not yet reaped by parent).
- The scheduler maintains a list of runnable processes per CPU.
- When the running process's time slice expires (typically a few milliseconds), the scheduler interrupts it (a timer interrupt fires), saves its registers, picks another runnable process, restores its registers, and resumes it. This is a **preemption**.
- When a running process makes a blocking system call (e.g., `read` on a network socket with no data), it transitions to sleeping. The scheduler picks another runnable process.
- When the I/O completes, the kernel marks the sleeping process runnable. It will run again whenever the scheduler picks it.

Linux's modern scheduler (CFS — Completely Fair Scheduler) is more sophisticated, tracking accumulated runtime per process and trying to give each process a fair share weighted by priority. But the basic structure — runnable / sleeping / running, with preemption on a timer — is universal.

### What this means for you

- **Your code does not run uninterrupted.** Between any two instructions, the scheduler may have run another process for milliseconds. Your code's wall-clock duration is unrelated to its CPU duration on a busy system.
- **Sleeping is cheap; spinning is expensive.** A process blocked on `read` consumes no CPU. A process in `while(!ready);` consumes 100% of a core doing nothing. This is why production code virtually never busy-waits — it uses a syscall (`epoll_wait`, `futex_wait`, `select`) that puts the thread to sleep until something happens.
- **You compete with the rest of the system.** "Why is my program slow?" is sometimes "because the OS is also running antivirus, indexer, browser, and 200 other things, and my program got 17% of the CPU."

### Experiment 3.3 — see scheduler activity

```bash
# Linux
top -H        # show threads
htop          # nicer
vmstat 1      # context switches per second; look at "cs" column

# macOS
top -o cpu
sudo dtrace -n 'sched:::on-cpu { @[execname] = count(); }' # advanced
```

On a typical desktop you'll see thousands of context switches per second even when "nothing is happening." Those are timers, kernel threads, daemons, and your editor's heartbeat.

---

## 3.7 The Big Four Abstractions

Unix-derived OSes give you a small number of abstractions that are surprisingly universal. Recognizing them is half the battle when learning a new system.

### Abstraction 1: Files (and "everything is a file")

A **file descriptor** is a small integer. The kernel maintains a per-process table mapping that integer to an underlying object. Most things you can read or write are exposed as file descriptors:

- A regular file on disk → FD.
- A pipe → FD (actually two FDs, one read end, one write end).
- A network socket → FD.
- A terminal (stdin/stdout/stderr) → FDs 0/1/2.
- An event source (timer, signal, inotify) → FD on Linux.
- A character device (/dev/random, /dev/null) → FD.

The same syscalls — `read`, `write`, `close`, `select`/`poll`/`epoll` — work on all of them. This is the famous "everything is a file" design. It is a leaky abstraction (a network socket is *not* like a regular file in important ways), but it is a deeply useful one. It is the reason a Unix program that processes stdin can be plugged into pipes, files, sockets, terminals, and devices interchangeably.

### Abstraction 2: Processes and Their Lifecycle

The process API is small:

- `fork()` — duplicate this process.
- `execve()` — replace this process's image with a new program.
- `wait()`/`waitpid()` — wait for a child to exit, get its exit status.
- `exit()` — terminate this process.
- Signals — asynchronous notifications between processes (`SIGINT`, `SIGTERM`, `SIGCHLD`, `SIGKILL`, etc.).

That's almost all of it. The shell, init systems, supervisors, container runtimes — they're all built on these primitives. We will see how `fork` + `exec` lets a shell start a program, then redirect its FDs *before* the program starts (Chapter 7 of Part 7).

### Abstraction 3: Virtual Memory and `mmap`

The memory API is also small:

- `mmap()` — ask the kernel to map something into your address space (a file, anonymous zero-pages, shared memory).
- `munmap()` — unmap.
- `mprotect()` — change permissions of a mapped region.
- `brk()`/`sbrk()` — adjust the size of the data segment (legacy, but `malloc` may use it).

Every higher-level memory API — `malloc`, the JVM heap, Python's allocator, GC nurseries — sits on top of `mmap`/`brk`. Knowing this means you can always answer the question "where did this memory come from?" by following the chain down to the kernel.

### Abstraction 4: Sockets

Networking is exposed as a special kind of FD:

- `socket()` — create a socket FD.
- `bind()`, `listen()`, `accept()` — for servers.
- `connect()` — for clients.
- `read()`/`write()` (or `recv`/`send`) — once connected.
- `close()` — done.

On top of these primitives, every network library, every HTTP server, every database client is built. The protocols change; the syscall layer doesn't.

These four abstractions — files, processes, virtual memory, sockets — are *the* model of a Unix-style system. Once you have them in your head, every framework, language runtime, and tool starts looking like an arrangement of them.

---

## 3.8 Worked Example: What Happens When You `read` From a File

Let's tie everything together. Your program calls `read(fd, buf, 4096)`.

1. **In user space**: arguments are placed in registers. `syscall` instruction.
2. **CPU traps to ring 0** at the syscall entry point.
3. **Kernel** looks up `fd` in this process's file descriptor table. It finds a kernel object describing the file.
4. The kernel checks: are the bytes already in the **page cache** (RAM cached copy of file data)? If so:
   - Copy 4096 bytes from page cache into `buf`.
   - Return 4096.
5. If not:
   - The process is marked **sleeping** (state = uninterruptible sleep, `D` in `ps`).
   - The kernel issues a request to the disk driver for the appropriate sector(s).
   - The scheduler picks another process to run on this CPU.
   - Eventually the disk completes the I/O and raises an interrupt.
   - The kernel handles the interrupt, copies the data into the page cache, marks our process **runnable**.
   - Some time later the scheduler picks our process; it resumes inside `sys_read`, copies bytes into `buf`, returns.

This is **blocking I/O**. The thread can do nothing else while waiting.

If you don't want to block, you have options:

- **Non-blocking I/O.** Set the FD non-blocking; `read` returns `EAGAIN` if no data, you go do something else.
- **Multiplexed I/O.** Use `epoll`/`kqueue`/`select` to wait on many FDs at once in one syscall; when any becomes ready, do the corresponding work.
- **Async I/O.** `io_uring` (Linux) or POSIX AIO; submit a request, the kernel notifies you when it completes.
- **Threads.** Have many threads, each blocking on its own I/O. Cheap-ish on Linux because threads are cheap; the cost is concurrency complexity.

These are the four IO models. Every "high-performance server" — Nginx, Redis, Node.js, Tokio, Netty — picks among them, often combining several. We will return to this in Chapter 23 (Async Runtime Internals).

---

## 3.9 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| Process isolation via VM | TLB pressure, page table RAM, fork overhead | Memory safety between programs, copy-on-write, lazy paging |
| User/kernel split | Syscall transition cost | Privilege boundary; user code can't crash kernel |
| Preemptive scheduling | Context switch overhead, cache disturbance | Fairness, no process can hog CPU |
| Cooperative scheduling (e.g., goroutines, async) | One slow piece blocks everyone in same scheduler | Far less overhead per switch, fits more concurrency |
| Block I/O thread-per-connection | Lots of threads, lots of context switches | Simplest programming model |
| Async / event-loop I/O | Complex code, harder debugging | One thread can serve many connections |
| `fork` + `exec` | Two syscalls, transient COW pressure | Composable: shell can adjust env between fork and exec |
| `posix_spawn` / `CreateProcess` | Less flexible | Cheaper for the common case |

---

## 3.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "The OS executes my program." | The CPU executes your program. The OS *arranges* for your program to be the one currently running and provides services on demand. |
| "Memory is a single flat thing." | Each process has its own virtual address space, and the same virtual address means different physical bytes in different processes. |
| "Threads are independent of the OS." | On Linux/macOS/Windows, every thread is known to the kernel and is scheduled by the kernel. "Userspace threads" (goroutines, async tasks) are multiplexed *on top of* kernel threads. |
| "System calls are like function calls." | They cross a privilege boundary; they cost ~50–200× more; they may put your thread to sleep for milliseconds or seconds. |
| "Allocating memory is the kernel's job." | The kernel hands you big regions via `mmap`. Your *language's allocator* (`malloc`, GC) splits those into small pieces. The kernel is rarely involved in individual `new` calls. |
| "Fork is slow because it copies all memory." | Fork uses copy-on-write. The pages aren't copied; the table is, with read-only flags. Only modified pages are copied later. |
| "Async means multi-threaded." | They are unrelated. Async is about *not blocking* a thread on I/O. You can have single-threaded async (Node.js) or multi-threaded async (Tokio, Go). |

---

## 3.11 Exercises

1. **Map your shell.** Run `cat /proc/$$/maps` (Linux) or `vmmap $$` (macOS). Identify five distinct mapped regions and describe what each one is. Compare two terminals' shells: which mappings match (same physical bytes shared), and which differ?

2. **Watch a fork.** Write a small C++ program:

   ```cpp
   #include <unistd.h>
   #include <cstdio>
   int main() {
       printf("before fork, pid=%d\n", getpid());
       pid_t p = fork();
       printf("after fork, pid=%d, fork returned %d\n", getpid(), p);
       sleep(2);
       return 0;
   }
   ```

   Run it. Why do you see `before fork` printed once but `after fork` printed twice? What does `fork` return in the parent vs the child? Look up `man 2 fork` and compare your observation to the documentation.

3. **Counting syscalls.** Write a program that copies a file in two ways:
   (a) `read` 1 byte at a time, `write` 1 byte at a time.
   (b) `read` 65536 bytes at a time, `write` the chunk.

   Time both on a 10 MB file. Then run each under `strace -c` (Linux) and compare syscall counts. Explain the timing difference using the syscall count.

4. **Page fault tour.** On Linux, run a program and use:

   ```bash
   /usr/bin/time -v ./your-program
   ```

   Look at "minor (reclaiming a frame) page faults" and "major (requiring I/O) page faults." A major fault means the page had to be fetched from disk. Run a program that reads a large file via `mmap` once when the page cache is cold (after a fresh boot) vs warm. Watch the major-fault count change.

5. **Privilege boundary.** Write a program that tries to do something forbidden — for example, reading `/dev/mem` without permissions, or executing the `hlt` instruction (you'd need inline asm). What happens? Who delivers the error — the CPU, the kernel, libc?

6. **Scheduler observation.** On Linux, run:

   ```bash
   chrt -f 1 ./your-program
   ```

   to give your program a real-time scheduling class. What changes about its behavior under load? Why might you not want to do this in production?

7. **Conceptual.** A user runs `find /` in one terminal and a video game in another. Explain in one paragraph why the video game might suddenly stutter, in terms of: (a) memory pressure on the page cache, (b) scheduler fairness, (c) disk I/O contention. What kernel knobs (`nice`, `ionice`, cgroups) might mitigate this?

---

## 3.12 What's Next

You now know what the CPU does (Chapter 2) and how the OS turns it into a useful machine that runs many programs safely (Chapter 3). The next two chapters bring this all the way back up to the source code you write.

Chapter 4 looks at how source text becomes machine code: the **compiler**, the **interpreter**, and the **JIT** — the three answers to "how do we get from `int x = 1 + 2;` to bytes the CPU can execute?" Chapter 5 then re-examines memory layout, this time from inside a *running* C++ program, with examples and diagrams.

After those two chapters, Part 1 is complete and you have an unbroken mental model from the program counter all the way up to source code. Then Part 2 takes you deep into memory and execution mechanics — the foundation we will lean on for everything that follows.

---


**[← Previous: Chapter 2 — CPU, RAM, Stack, Heap, Registers](02-cpu-ram-stack-heap-registers.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 4 — Machine Code, Assembly, Compilers, Interpreters →](04-machine-code-assembly-compilers-interpreters.md)**
