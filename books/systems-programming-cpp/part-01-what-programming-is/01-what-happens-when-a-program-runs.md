# Chapter 1 — What Happens When A Program Runs

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a "program" actually is on disk and in memory — they are not the same thing.
2. Trace, step by step, what happens between typing `./hello` and your code's `main()` being called.
3. Describe the role of the kernel, the loader, the dynamic linker, and the C runtime startup code.
4. Use real tools (`file`, `nm`, `objdump`, `readelf`/`otool`, `strace`/`dtruss`, `ldd`/`otool -L`) to inspect a binary and watch it start up.
5. Recognize that what we call "running a program" is actually the kernel constructing a *process* — a bookkeeping structure — and pointing the CPU at some bytes inside it.

This is the most foundational chapter in the book. Everything else — memory layouts, function calls, frameworks, runtimes, async, dependency injection — builds on the picture you form here. Take your time.

---

## 1.1 The Mental Model You Probably Have

If you ask most developers "what happens when you run a program?", you get something like:

> "The computer reads my code and does what it says."

This is wrong in almost every important way, but it's wrong in a specific way that is worth examining. It contains three implicit assumptions:

1. That there is a single thing called "the computer" doing the work.
2. That the code on disk is the thing being executed.
3. That execution is a continuous, top-to-bottom reading of your code.

None of these is true.

The reality is that:

1. There are at least four distinct actors involved in running your program: the **kernel**, the **loader**, the **dynamic linker**, and the **C runtime startup code** — and your `main()` function only runs *after* all four of them have done their jobs.
2. The bytes on disk are not the bytes that execute. They are a recipe for *constructing* an in-memory image, and that construction is non-trivial.
3. Execution is not a "reading" — it is the CPU's program counter (a single hardware register) being pointed at some memory and the CPU repeatedly fetching, decoding, and executing instructions until something asks it to stop.

We will rebuild your mental model from the bottom up. First, an analogy.

---

## 1.2 Analogy: The Theater

A program on disk is like a **script and a set of building instructions** delivered to an empty theater.

When you "run" the program, the following happens:

1. **A stage manager (the kernel)** receives a request: "stage this play."
2. The stage manager **builds an empty theater** (a process) — walls, lights, an empty stage, sections labeled "props on the left, costumes on the right, benches in the back."
3. A **set crew (the loader)** reads the building instructions from the script, and assembles the set on stage — putting the right props at the right positions, hanging the right backdrops.
4. A **costume crew (the dynamic linker)** notices the script says "the actor wears a doctor's coat" and goes off to find the *shared* doctor's coat from the theater's wardrobe (a shared library), bringing it to the stage.
5. A **director's assistant (the C runtime startup)** does final pre-show prep: turns on the spotlight on the right spot, hands the lead actor their first cue card, makes sure the curtain is in the correct position.
6. **Then, and only then**, the director says "action," and the lead actor (your `main()`) starts speaking their first line.

When developers say "I ran my program," what they mean is "the lead actor said their first line." But everything before that — the kernel, the loader, the dynamic linker, the runtime startup — is real, observable work, and understanding it dissolves an enormous amount of confusion later in your career.

We are now going to walk through each step concretely.

---

## 1.3 Setup: A Tiny Program

Save this as `hello.cpp`:

```cpp
#include <cstdio>

int main() {
    std::puts("hello, world");
    return 0;
}
```

Compile it:

```bash
clang++ -std=c++20 -O0 -g hello.cpp -o hello
```

(Or `g++` if you prefer.)

Run it:

```bash
./hello
```

You see `hello, world`. We are now going to spend the rest of this chapter understanding what happened in the gap between you pressing Enter and `hello, world` appearing on your terminal.

---

## 1.4 The Program on Disk Is Not the Program in Memory

Run this:

```bash
file ./hello
```

On Linux you will see something like:

```
./hello: ELF 64-bit LSB pie executable, x86-64, ...
```

On macOS:

```
./hello: Mach-O 64-bit executable arm64
```

ELF and Mach-O are **executable file formats**. They are not "code." They are *containers*: structured files describing how to construct a running process from this artifact.

An ELF file (we'll use ELF as the running example; Mach-O is conceptually similar) contains, roughly:

- A **header** at byte 0: "I am ELF, I am 64-bit, my entry point is at virtual address X, I have N program headers and M section headers."
- **Program headers**: a list of *segments*, each saying "load these bytes from the file at offset O, length L, into memory at virtual address V, with permissions P (read/write/execute)."
- **Section headers**: a finer-grained view of the bytes, used by linkers and debuggers (`.text` for code, `.rodata` for read-only data, `.data` for initialized globals, `.bss` for zero-initialized globals, `.symtab` for symbols, `.debug_*` for debug info, etc.).
- The **actual bytes**: the code, the constants, the initial values of globals.

This is critical: **the file on disk is not laid out the same way memory is.** The on-disk layout is optimized for storage and for the linker; the in-memory layout is optimized for execution. The transformation is the loader's job.

### Experiment 1.1 — See the segments

```bash
# Linux
readelf -l ./hello

# macOS
otool -l ./hello | head -80
```

On Linux you will see entries like:

```
Type           Offset     VirtAddr           FileSiz   MemSiz   Flg
LOAD           0x000000   0x0000000000000000 0x000648  0x000648 R
LOAD           0x001000   0x0000000000001000 0x0001a5  0x0001a5 R E
LOAD           0x002000   0x0000000000002000 0x000118  0x000118 R
LOAD           0x002db8   0x0000000000003db8 0x000260  0x000268 RW
```

Read this carefully:

- Each `LOAD` line is a **segment** — a contiguous chunk of bytes the loader will copy into memory.
- `Offset` is where the bytes live in the file. `VirtAddr` is where they will live in the process's memory. **They are different.** A read-only-data segment might live at file offset `0x2000` but be loaded at virtual address `0x402000`.
- `Flg` is the permission. `R E` means readable and executable — that's your code (`.text`). `R` alone means read-only (constants, your string `"hello, world"`). `RW` means read/write — globals.
- Notice the segment with `MemSiz` (`0x268`) larger than `FileSiz` (`0x260`). The extra 8 bytes are zero-initialized memory (`.bss`) — globals like `int x;` that don't need bytes stored on disk because they're just zeroes. The on-disk file saves space; the loader pads with zeroes when loading.

Right there you have learned something most programmers go years without realizing: **uninitialized globals do not take up space in your binary.** They take up space in memory, allocated by the loader. If you have `int big_buffer[1'000'000];` as a global, your binary does not grow by 4 MB. The loader sees "MemSiz 4 MB, FileSiz 0" and zero-fills the region at load time.

### Misconception: "The binary is the program"

The binary is a *recipe*. The program is the in-memory image the loader constructs from the recipe, plus the dynamic libraries it pulls in, plus the runtime state (stack, heap, registers, file descriptors). Two invocations of the same binary produce two different programs — different memory addresses (because of ASLR), different file descriptors, different state.

---

## 1.5 What Actually Happens When You Type `./hello`

We are now going to walk through every step. Some of the names below are Linux-specific; macOS uses different names but the concepts are identical.

### Step 1: The shell calls `execve` (or `posix_spawn` on macOS)

Your shell (`bash`, `zsh`) is itself a running program. When you type `./hello`, the shell does the following:

1. It calls `fork()`, which **duplicates** the shell process. There are now two near-identical processes running. The original is the *parent*; the new one is the *child*.
2. In the child only, the shell calls `execve("./hello", argv, envp)`. This is the **system call** that says to the kernel: "replace this process's memory image with the program at `./hello`."

(macOS prefers `posix_spawn`, which combines fork+exec into one call for efficiency, but the model is the same.)

`fork`/`exec` is one of those deep design choices that shapes everything above it on Unix. We will return to it. The point for now: **starting a program is a kernel-mediated operation.** Your shell does not "run" your program; it asks the kernel to run it.

### Step 2: The kernel constructs a fresh process image

Inside `execve`, the kernel does roughly this:

1. **Open** `./hello` and read its ELF header. Validate it: is this even an executable? Is it for this CPU? Does the user have permission?
2. **Tear down** the calling process's existing memory mappings. (This is why `exec` does not return on success — there's nothing to return to.)
3. **For each LOAD segment** in the ELF, create a memory mapping using `mmap`-style operations: "starting at virtual address V, map L bytes from this file with these permissions." For BSS-style segments, the kernel maps anonymous (file-backed-by-nothing) zero pages.
4. **Allocate a stack.** A fresh region of memory — typically 8 MB on Linux — mapped read/write, with a guard page at the bottom that will fault if you overflow.
5. **Set up `argv` and `envp` on the new stack.** The strings `./hello` and your environment variables are copied to the top of the new stack, and pointers to them are set up below.
6. **Determine the entry point.** Here's where it gets subtle. The ELF header says "entry point is at virtual address `0x...`." But for *dynamically linked* programs (which is most programs), this entry point is **not your `main`**. It is the entry point of the **dynamic linker** — typically `/lib64/ld-linux-x86-64.so.2` on Linux or `/usr/lib/dyld` on macOS.
7. **Set the CPU's program counter** to that entry point.
8. **Return to user space.** The CPU starts executing instructions at that entry point.

At this moment, your program is "running" — but `main()` is nowhere near being called yet.

### Step 3: The dynamic linker runs

The first code that runs in user space is the **dynamic linker** (also called the **interpreter**). Its job: pull in all the shared libraries your program depends on, resolve symbol references, and run library initializers.

For our `hello` program, that means at minimum loading **libc** — the C standard library — because `puts` lives there.

You can see this list:

```bash
# Linux
ldd ./hello

# macOS
otool -L ./hello
```

Linux output looks like:

```
linux-vdso.so.1 (0x00007ffd...)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

The dynamic linker:

1. Maps libc into the process's memory using the same ELF loading logic.
2. Walks through your binary's **relocations** — places where you reference a symbol like `puts` whose address wasn't known at compile time — and patches those references with the now-known addresses inside libc.
3. Runs every shared library's **initializers** (`.init_array` / `__attribute__((constructor))` functions, C++ global constructors that live in libraries, etc.).
4. Jumps to your binary's "real" entry point, called `_start`.

If you have ever gotten the error `error while loading shared libraries: libfoo.so.1: cannot open shared object file: No such file or directory`, this is the dynamic linker, before any of your code has run, refusing to continue because it could not find a library it was supposed to patch in.

### Step 4: `_start` runs (the C runtime entry)

`_start` is a tiny piece of assembly provided by your toolchain (specifically, by a file called `crt1.o` or similar — "C run-time, version 1, object file"). You did not write it. Your compiler links it into every executable.

`_start`'s job is roughly:

1. Pick `argc`, `argv`, and `envp` off the stack where the kernel placed them.
2. Call **`__libc_start_main`** (in glibc) or its equivalent, passing it the address of your `main` function.
3. `__libc_start_main` calls a chain of pre-main hooks: C++ global constructors (in `.init_array`), TLS (thread-local storage) setup, signal handler defaults, locale init, etc.
4. **Then** it calls your `main(argc, argv)`.
5. When your `main` returns, `__libc_start_main` does **post**-main cleanup: C++ global destructors, `atexit` handlers, flushing stdio buffers (this is why `printf` output appears even if you don't `\n`-terminate it — the buffer is flushed on exit).
6. Finally it calls the `exit` system call with your return value, which the kernel uses to tear down the process.

### Step 5: The program is gone

`exit` is one-way. The kernel:

1. Closes any file descriptors the program forgot to close.
2. Releases the memory mappings.
3. Sends a `SIGCHLD` to the parent (the shell) so it knows the child is done.
4. Stores the exit code in a small kernel structure until the parent calls `wait`.
5. Removes the process from the scheduler.

Your shell, in its `wait` call, receives the exit code and prints the next prompt.

---

## 1.6 The Visual Model

```
                     User types ./hello
                            |
                            v
          +-----------------------------------+
          |  shell: fork() -> child process    |
          |  shell: execve("./hello", ...)     |    [system call boundary]
          +-----------------+------------------+
                            |
                            v
        ============== KERNEL ===============
         - Read ELF header
         - mmap LOAD segments into memory
         - Allocate stack, set up argv/envp
         - Set PC = dynamic linker entry
         - Return to user space
        =====================================
                            |
                            v
        +--------------------------------------+
        |  /lib64/ld-linux-x86-64.so.2 (loader)|
        |  - Map libc.so, other deps           |
        |  - Resolve symbol relocations        |
        |  - Run library initializers          |
        |  - Jump to _start                    |
        +-----------------+--------------------+
                          |
                          v
        +---------------------------------------+
        |  _start (from crt1.o)                  |
        |  - Pull argc/argv/envp off stack       |
        |  - Call __libc_start_main(&main, ...)  |
        +-----------------+---------------------+
                          |
                          v
        +---------------------------------------+
        |  __libc_start_main                     |
        |  - Run C++ global constructors         |
        |  - Set up TLS, signals, locale         |
        |  - CALL YOUR main()  <-- finally        |
        |  - On return: run destructors, atexit  |
        |  - exit(retval)                        |
        +-----------------+---------------------+
                          |
                          v
        ============== KERNEL ===============
         - Close FDs, free memory
         - Notify parent (shell) via SIGCHLD
        =====================================
```

Stare at this diagram. Re-read it after each experiment below. It is the picture you want burned into your head.

---

## 1.7 Experiments

These are not optional. Each one teaches you to *see* a layer that was previously invisible.

### Experiment 1.2 — Watch the system calls

Run:

```bash
# Linux
strace -e trace=execve,openat,mmap,close,exit_group ./hello

# macOS
sudo dtruss ./hello 2>&1 | head -40
```

You will see something dramatic: dozens — possibly hundreds — of system calls, before `hello, world` appears. You'll see:

- The initial `execve("./hello", ...)`.
- `openat` calls opening `ld.so.cache` (a cache of where shared libraries live), then `libc.so.6`.
- `mmap` calls mapping libc segments into memory.
- Eventually a `write(1, "hello, world\n", 13)` — *that's* your `puts`.
- An `exit_group(0)`.

The point: **almost all the work of "running hello world" is the loader and dynamic linker doing setup.** Your one line of code generates exactly one syscall (`write`). Everything else is infrastructure.

### Experiment 1.3 — Find your symbols

```bash
nm ./hello | grep -E "main|_start|__libc"
```

You will see entries for `main`, `_start`, and references to `__libc_start_main`. Confirm with your eyes that `_start` and `__libc_start_main` exist in your "tiny" program — you didn't write them, but they're there.

### Experiment 1.4 — Watch a constructor run before main

Save as `ctor.cpp`:

```cpp
#include <cstdio>

struct Greeter {
    Greeter()  { std::puts("[ctor] before main"); }
    ~Greeter() { std::puts("[dtor] after main"); }
};

static Greeter g;

int main() {
    std::puts("[main] inside main");
    return 0;
}
```

Compile and run. The output is:

```
[ctor] before main
[main] inside main
[dtor] after main
```

This is empirical proof that there is code running *before* `main` and *after* `main`. The C++ global `g` is constructed by `__libc_start_main` (via the `.init_array` mechanism) and destructed by the at-exit handlers it registered.

This is also where many real bugs come from — if your constructor calls `std::cout`, you must trust that `std::cout` was already constructed by the standard library's own pre-main initialization. The order of "what got initialized when" is called the **static initialization order fiasco** in C++ folklore. We will revisit it.

### Experiment 1.5 — Static linking, no dynamic linker

```bash
clang++ -std=c++20 -static hello.cpp -o hello-static
ls -la hello hello-static
file hello-static
ldd hello-static       # "not a dynamic executable"
```

The static binary is much larger (typically 1-2 MB vs ~16 KB), because libc is now baked into the binary itself. There is no dynamic linker step at startup. The kernel sets the entry point directly to `_start`.

Compare startup times if you want:

```bash
time ./hello
time ./hello-static
```

The difference is small for hello-world, but the *mechanism* is meaningfully different. On systems with thousands of binaries, dynamic linking exists so that one copy of libc serves all of them; static linking trades disk and memory for startup simplicity. This is a tradeoff. Remember the principle: **every design choice in systems is a tradeoff. Always ask "what did this cost?"**

### Experiment 1.6 — See ASLR in action

Run this twice:

```bash
# Linux
./hello & cat /proc/$!/maps | head -5; wait

# macOS
./hello &
vmmap $! | head -20
wait
```

Run it again. The addresses shift each time. That's **ASLR** — Address Space Layout Randomization. The kernel deliberately picks different base addresses on each launch so that an attacker who finds a vulnerability cannot rely on hardcoded addresses. It also confirms our point: **the bytes on disk live at different addresses on different runs.** Memory layout is constructed at load time, not baked in.

---

## 1.8 What This Teaches Us About Higher-Level Languages

Everything you just learned applies, with cosmetic changes, to every language and runtime you will ever use.

- **Python.** When you run `python script.py`, the kernel loads the *Python interpreter* (which is itself an ELF/Mach-O binary, with its own loader/linker startup). The interpreter then opens `script.py`, parses it into bytecode, and walks that bytecode. Python's `if __name__ == "__main__":` exists precisely because Python doesn't have a `main()` — every file is executable code that runs top to bottom when imported, so the convention is needed to distinguish "running as a script" from "imported as a module." We will dissect this in Part 6.

- **Node.js.** Same picture — the kernel loads the Node binary, V8 starts up, V8's runtime initializers run, and *then* your `index.js` is parsed and executed. The startup flame graph of a Node program is dominated by V8 initialization, not your code.

- **JVM.** Even more so — the kernel loads `java`, which loads `libjvm.so`, which initializes the heap, the class loader, the JIT, runs `<clinit>` for boot classes, and *then* calls your `main`. JVM startup is famously slow precisely because all of this work happens up front.

- **Go.** Statically linked by default, no dynamic linker step. But the Go runtime — scheduler, garbage collector, goroutine bootstrap — runs before your `main` in exactly the same architectural slot that `__libc_start_main` occupies in C++. We will see this in detail in Part 6.

The pattern is universal: **before "your code" runs, a runtime constructs the conditions that make your code's assumptions true.** Most language confusion comes from being unaware of that runtime. This book teaches you to see it.

---

## 1.9 Tradeoffs and Why They Matter

Look back at the design choices we observed:

| Choice | Cost | Benefit |
|---|---|---|
| Dynamic linking | Slower startup, deployment complexity (must ship libs or rely on system libs) | Smaller binaries, shared library memory, hot-fix libs without re-linking |
| Static linking | Bigger binaries, no shared library memory, must rebuild for lib upgrades | Faster startup, deploy a single file, no "dependency hell" |
| ASLR | Tiny perf cost, debugger surprises | Security: makes exploits harder |
| `fork`+`exec` model | Slightly slower than spawn-style | Composability: you can fork and adjust the child's environment before exec |
| Pre-main initializers (C++ globals, JVM `<clinit>`, Python module top-level code) | Hidden execution, ordering hazards | Lets libraries set up state before user code touches them |

There is no "right answer" to any of these. There are *contexts* in which one tradeoff is better than another. Senior engineers think in tradeoffs, not in rules. This is the first chapter, and already we are doing it.

---

## 1.10 Common Misconceptions Recap

| Misconception | Reality |
|---|---|
| "The binary is the program." | The binary is a recipe; the program is a process — an in-memory image plus state. |
| "`main` is the start of execution." | `main` runs only after the kernel, dynamic linker, and C runtime have done substantial work. |
| "Compiled languages don't have a runtime." | They do. C++ has `__libc_start_main`. Go has the Go runtime. The runtime is *smaller* than Python's, but not zero. |
| "ASLR moves my code around at runtime." | No — ASLR picks the load address *once*, at process start. Once mapped, code stays put. |
| "Uninitialized globals waste disk space." | They live in `.bss` and take zero file bytes. The loader zero-fills them in memory. |
| "`fork` and `exec` are the same thing." | `fork` duplicates a process. `exec` *replaces* the current process's image. They are usually used together but they are independent operations. |

---

## 1.11 Exercises

1. **Trace your own toolchain.** Run Experiment 1.2 (`strace`/`dtruss`) on your `hello`. Count: how many `mmap` calls? How many `openat` calls? Which file is opened most often? Write a one-paragraph explanation of *why* — for the file opened most often — that file is being read.

2. **Find the real entry point.** Run `readelf -h ./hello` (Linux) or `otool -h ./hello` (macOS). Locate the entry point address. Now run `objdump -d ./hello | less` (or `otool -tV ./hello`) and search for that address. What instruction is there? What function name (if any) does it correspond to? Hint: it is not `main`.

3. **Break startup intentionally.** Modify the dynamic linker's library path so libc cannot be found:

   ```bash
   # Linux only — be careful, run only in a subshell
   ( LD_LIBRARY_PATH=/nonexistent LD_PRELOAD=/nonexistent ./hello )
   ```

   What error do you get? Who produced that error — the kernel, the dynamic linker, or your `main`? Prove your answer using `strace`.

4. **Two hellos, two memories.** Write a small program that prints the address of a global variable:

   ```cpp
   #include <cstdio>
   static int g = 42;
   int main() { std::printf("&g = %p\n", (void*)&g); return 0; }
   ```

   Run it 5 times. Are the addresses the same? Why or why not? Now disable ASLR for one run:

   ```bash
   # Linux
   setarch $(uname -m) -R ./a.out
   ```

   Are the addresses the same now? What does this tell you about the relationship between the file and memory?

5. **Constructor ordering.** Write a program with three global objects in three different `.cpp` files, each printing its name in its constructor. Compile and link. Are they constructed in a predictable order? Try changing the link order (`g++ a.o b.o c.o` vs `g++ c.o b.o a.o`). Does the output change? What does this tell you about portable code that relies on constructor order? (This is the "static initialization order fiasco" in miniature; we will fix it properly in Part 3.)

6. **Conceptual.** In one paragraph, explain to someone who has only ever written Python: when you run `python my_script.py`, what is the analog of (a) the kernel's `execve`, (b) the dynamic linker, (c) `_start`, (d) `__libc_start_main`, (e) your `main`? You will not know all the answers yet. Make your best guess. We will return to this in Chapter 31.

---

## 1.12 What's Next

You now know that "running a program" is a multi-stage construction process, not a single act. In the next chapter we go one level deeper: **what is the CPU actually doing during all this?** What are RAM, the stack, the heap, and registers? We will look at the CPU as a machine that does only one thing — fetch, decode, execute — and see how everything we discussed in this chapter is really just careful arrangement of bytes so that this dumb little machine produces the behavior we want.

The recurring theme of this book is going to become clear quickly: **everything in software, no matter how high-level, eventually grounds out in memory and a CPU.** You cannot reason about systems without holding that ground truth. That's what Chapter 2 builds.

---


**[Up: Part 1](README.md)** · **[Next: Chapter 2 — CPU, RAM, Stack, Heap, Registers →](02-cpu-ram-stack-heap-registers.md)**
