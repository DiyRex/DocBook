# Chapter 6 — Function Calls Internally

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what happens, step by step, when a function is called and when it returns — at the level of registers and memory.
2. Describe the **calling convention** for x86-64 SysV (Linux/macOS) and ARM64: where arguments go, where return values go, which registers are caller-saved vs callee-saved.
3. Read and write a stack frame: prologue, body, epilogue.
4. Explain why **inlining**, **tail calls**, and **leaf functions** are special, and what they mean for performance.
5. Trace, by hand, a small chain of function calls and predict what's on the stack at each point.
6. Recognize how exceptions, longjmp, and coroutines bend the rules of "normal" function call/return.

This chapter is the *mechanism* underneath every function call you've ever made. Once you've internalized it, debuggers, profilers, stack traces, FFI, and ABI compatibility all stop being mysterious.

---

## 6.1 What "Function Call" Has to Mean

To call a function, the caller and the callee must agree on:

1. **Where the arguments are.** The callee has to know where to look for its parameters.
2. **Where to return to.** The callee has to know where to jump back to when done.
3. **Where the return value goes.** The caller has to know where to look for it.
4. **Who saves which registers.** The CPU has limited registers; both sides use them; a contract is needed for who is responsible for preserving each.
5. **What the stack looks like.** Both sides must agree on the stack layout for spilled arguments, locals, return address.

The agreement is called a **calling convention** or, more broadly, an **ABI** (Application Binary Interface). Different OS-CPU pairs have different ABIs. Your code does not care about this *unless* it crosses a boundary — into a library written elsewhere, into a different language, into the kernel. At those boundaries, the ABI is everything.

---

## 6.2 The x86-64 SysV Convention (Linux, macOS, BSD)

This is what you'll see most often. Memorize at most this much:

### Argument passing

| Argument # | Integer/pointer reg | Floating-point reg |
|---|---|---|
| 1 | `rdi` | `xmm0` |
| 2 | `rsi` | `xmm1` |
| 3 | `rdx` | `xmm2` |
| 4 | `rcx` | `xmm3` |
| 5 | `r8`  | `xmm4` |
| 6 | `r9`  | `xmm5` |
| 7+ | on the stack | on the stack |

So `add(int a, int b)` is called with `a` in `edi`, `b` in `esi`. Eight arguments? First six in registers, last two pushed on the stack.

### Return values

- Integer / pointer return → `rax` (and `rdx` for 128-bit returns).
- Floating-point return → `xmm0`.
- Return value larger than 16 bytes → caller passes a hidden first argument: a pointer to where to write the result. (This is what happens when a function returns a struct by value.)

### Caller-saved vs callee-saved

When a function calls another, who is responsible for preserving each register?

- **Caller-saved** (volatile): the callee can clobber these. If the caller cares, *it* must save them before the call and restore after.
  - `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`, all `xmm`.
- **Callee-saved** (non-volatile): the callee must save them on entry and restore on exit if it uses them.
  - `rbx`, `rbp`, `r12`–`r15`, plus `rsp` (always preserved as part of the return discipline).

This split exists because not every register is "alive" across every call. Some registers (the caller-saved ones) are convenient scratch space — the caller doesn't expect them to survive a call, so the callee can use them freely. Other registers (callee-saved) are "long-lived" in the caller — values kept across many calls — so the callee must preserve them or save and restore.

### The stack

- The stack must be **16-byte aligned** at the moment of a `call` instruction (this becomes 16-byte alignment minus 8 *inside* the called function, because `call` pushed 8 bytes — the return address). Modern code maintains alignment by allocating multiples of 16; misalignment is a real source of crashes when SSE instructions, which require alignment, are used by the compiler.
- The stack grows down (toward smaller addresses).

### The famous **red zone**

SysV reserves the **128 bytes below `rsp`** as a "red zone" the function can use for temporaries without explicitly allocating, *as long as it doesn't make any function calls*. Leaf functions (functions that call nothing) can take advantage of this for free temporary space. The kernel doesn't use a red zone — kernel code must always explicitly allocate.

---

## 6.3 ARM64 (AArch64) Convention

If you're on Apple Silicon or modern ARM Linux, here's the equivalent:

| Argument # | Reg |
|---|---|
| 1–8 (int/ptr) | `x0`–`x7` |
| 1–8 (float) | `v0`–`v7` |
| 9+ | on the stack |

- Return value: `x0` (int) or `v0` (float).
- Caller-saved: `x0`–`x18`.
- Callee-saved: `x19`–`x30` (where `x29` = frame pointer, `x30` = link register).
- The **link register** `x30` (`lr`) is special: it holds the *return address*. A `bl` (branch-and-link) instruction sets `lr = next-instruction; pc = target`. The callee's return is `ret`, which is essentially `pc = lr`.

The biggest conceptual difference from x86-64: ARM uses a *register* for the return address rather than pushing it on the stack at call time. Non-leaf functions still must save `lr` to the stack before they call something else; otherwise their own `lr` would be overwritten by the inner call. Leaf functions can return without touching the stack at all.

---

## 6.4 The Anatomy of a Stack Frame

We will draw the x86-64 picture; ARM64 is almost identical conceptually. Consider:

```cpp
int add(int a, int b) {
    int result = a + b;
    return result;
}

int main() {
    return add(2, 3);
}
```

When `main` calls `add(2, 3)`, the sequence is:

1. **Caller (`main`) sets up arguments.**
   ```asm
   mov  edi, 2          ; first arg
   mov  esi, 3          ; second arg
   ```

2. **Caller executes `call add`.** This single instruction:
   - Pushes the return address (the address of the next instruction in `main`) onto the stack.
   - Sets `rip` (program counter) to `add`'s entry point.

3. **`add`'s prologue.** Most non-trivial functions begin with:
   ```asm
   push rbp           ; save caller's frame pointer
   mov  rbp, rsp      ; new frame pointer = current stack top
   sub  rsp, 16       ; allocate space for locals (rounded up for alignment)
   ```

   At this point the stack looks like (high to low):
   ```
   high addr  | ... main's locals ...        |
              | return address               | <- pushed by 'call'
              | saved rbp (caller's rbp)     | <- pushed by 'push rbp'
              | local: result (int, 4 bytes) |
              | padding                      |
   low addr   | <- rsp                       |
   ```

   `rbp` now points to "saved rbp" location; the local `result` is at `[rbp - 4]`; the saved return address is at `[rbp + 8]`.

4. **`add`'s body.**
   ```asm
   mov  dword ptr [rbp - 4], edi     ; spill 'a' to local stack
   mov  dword ptr [rbp - 8], esi     ; spill 'b' to local stack
   mov  eax, dword ptr [rbp - 4]
   add  eax, dword ptr [rbp - 8]
   mov  dword ptr [rbp - 12], eax    ; result = a + b
   mov  eax, dword ptr [rbp - 12]    ; load return value
   ```
   (This is `-O0` code. With `-O2`, all of this collapses to `lea eax, [rdi + rsi]`.)

5. **`add`'s epilogue.**
   ```asm
   mov  rsp, rbp      ; deallocate locals
   pop  rbp           ; restore caller's frame pointer
   ret                ; pop return address into rip
   ```

   The `ret` instruction pops 8 bytes from the stack into `rip`, jumping back to `main`.

6. **Caller (`main`) reads return value** from `eax`.

That is the complete mechanism. Every C++ function call you have ever written follows this dance, possibly heavily optimized.

### Visual summary

```
Time           Action                        Stack (top is low addr, growing down)
-----          ------                        -------------------------------------
main running   ...                           |  main's locals          |
                                             |                         | <- rsp
call add       push retaddr; jmp add         |  main's locals          |
                                             |  retaddr                | <- rsp
add prologue   push rbp; rbp=rsp; sub rsp,16 |  main's locals          |
                                             |  retaddr                |
                                             |  saved rbp              | <- rbp
                                             |  add's locals (16 B)    | <- rsp
add body       computes                       (same)
add epilogue   mov rsp,rbp; pop rbp; ret     |  main's locals          | <- rsp
                                             (retaddr popped into rip)
back in main   continues                     |  main's locals          | <- rsp
```

---

## 6.5 Frame Pointers: Optional, But Useful

We used `rbp` as a "frame pointer" — a stable reference to the current frame, regardless of how `rsp` moves around within the function. Using `rbp` this way makes:

- **Stack walking easy.** A debugger or profiler can chase the linked list of saved-`rbp` values to walk the call stack.
- **Variable addressing predictable.** Each local has a fixed offset from `rbp`, even if the function dynamically allocates more stack.

But it costs:

- One extra register (`rbp` is unavailable for general use).
- Two instructions per function (`push rbp; mov rbp, rsp` in the prologue; `pop rbp` in the epilogue).

So compilers often **omit the frame pointer** with `-fomit-frame-pointer` (which is on by default at `-O1` and above on most platforms). Local addressing then becomes `[rsp + offset]` with offsets computed by the compiler.

The cost is paid by tooling: profilers and stack walkers must use **DWARF unwinding tables** (a debug-info side table that says "at this PC, the previous frame is at this offset") instead of walking `rbp` chains. This is slower and more complex.

This is a real tradeoff in production: many companies (Meta, Google) re-enable frame pointers for production binaries because cheap profiling matters more than the small per-call cost. Others do not.

---

## 6.6 Inlining

When the compiler **inlines** a function call, it pastes the function's body into the caller's body, *replacing the call*. There is no `call`, no prologue, no epilogue, no register-saving — the work just happens.

```cpp
inline int square(int x) { return x * x; }

int f(int a) { return square(a) + 1; }
```

After inlining:

```cpp
int f(int a) { return a * a + 1; }
```

The benefits:

- Eliminates call overhead (10–30 cycles).
- Enables further optimization. After inlining, the compiler sees the *combined* code and can constant-fold, eliminate dead code, etc. — optimizations that are blocked by the function-call abstraction.
- Better register allocation across the call boundary.

The costs:

- **Code size growth.** Inlining a function 100 times duplicates its code 100 times. Bigger binaries hurt instruction cache.
- **Compile time.** The optimizer has more to think about.

Modern compilers inline aggressively at `-O2` and above, with or without the `inline` keyword (the keyword is more about ODR/linkage than actual inlining nowadays). They use heuristics: small functions, hot call sites, called once, etc. You can encourage it (`__attribute__((always_inline))`, `[[gnu::always_inline]]`) or discourage it (`__attribute__((noinline))`).

> **Mental model:** if a function is small and called a lot, assume it's inlined. Look at the assembly to confirm. The "call overhead" you imagine usually doesn't exist in optimized builds.

---

## 6.7 Tail Calls and Why They Matter

A **tail call** is a function call that is the last action in the calling function:

```cpp
int g(int x);
int f(int x) {
    return g(x + 1);   // the call to g is the last thing f does
}
```

A naive compilation:
1. Push `f`'s return address.
2. Call `g`.
3. `g` returns to `f`.
4. `f` returns to its caller.

But notice: when `g` returns, `f` has nothing to do but immediately return. So the compiler can transform this into a **tail call**: instead of `call g`, just `jmp g`. `g` then returns directly to `f`'s caller, skipping the `f` frame entirely.

The benefit:
- No stack growth per recursive call. Tail-recursive functions can recurse indefinitely without stack overflow.
- One less frame on the stack means slightly faster.

Languages that *guarantee* tail-call optimization — Scheme, Scala (in some forms), Erlang, Lua — let you use recursion as a control-flow primitive without worry. C++ does not guarantee TCO; compilers usually do it at `-O2` for direct recursive tail calls but won't promise across all cases. Hand-written tail-recursive loops in C++ can blow the stack at `-O0` and not at `-O2` — a notoriously confusing experience.

---

## 6.8 Leaf Functions

A **leaf function** calls nothing. The compiler can optimize aggressively:

- No need to save callee-saved registers it doesn't use.
- Can use the **red zone** without allocating.
- On ARM64, doesn't need to save `lr` to the stack — `lr` is intact for the return.

```cpp
int leaf(int x) { return x + 1; }
```

The entire compiled function might be:
```asm
lea  eax, [rdi + 1]
ret
```

Two instructions, no stack manipulation. Compare to a non-leaf function which must allocate a frame, possibly save registers, etc.

This is one reason "small functions are cheap" — leaf small functions are essentially free; the optimizer will inline them and they evaporate completely.

---

## 6.9 Variadic Functions

`printf("%d %s\n", x, name)` — how does `printf` know how many arguments are passed and of what types?

In C/C++, the answer is: by the **format string**. The ABI says: arguments beyond the named ones are placed in the standard argument registers and on the stack just like normal arguments; the function uses `va_list` macros to walk through them.

This is a leaky and fragile design:
- The format string and the argument list must match. `printf("%d", "hello")` is undefined behavior.
- Compilers add special checks (`-Wformat`) to catch mismatches at compile time.
- Different ABIs handle variadic differently from named arguments. On x86-64 SysV, `rax` is set to the number of `xmm` registers used so `va_arg` knows where to look.

This is why varargs is hard for FFI: a Rust binding to a C variadic function, or a Python-to-C call with variable arguments, requires careful per-platform glue code.

---

## 6.10 Exception Handling: A Different Way Off the Stack

Normal function return follows the call stack down. **Exception handling** *unwinds* the stack — it walks back through frames, running destructors at each level, until it finds a `catch` block.

For this to work:
- Each frame must have **unwind information** (DWARF tables in `.eh_frame` on Linux).
- Throwing constructs an exception object on a special heap-like region (the exception heap), then begins the unwind.
- Each frame's entry in the unwind table says: "if an exception passes through here, run this cleanup code (destructors), then continue unwinding."

This is dramatically slower than a normal return — exception throwing on x86-64 typically costs microseconds to tens of microseconds, dominated by table lookup. The "zero cost when not thrown" mantra is real (no overhead in the success path), but the exceptional path is expensive. This is why exceptions are appropriate for *exceptional* conditions, not control flow.

We will dedicate a chapter to exception handling in Part 3.

---

## 6.11 Crossing Languages: FFI

When a C++ function calls a function written in C, the calling convention must match. `extern "C"` in C++ tells the compiler "use C linkage" — which means: don't mangle the function name, and use the C calling convention (which on most platforms is identical to C++ for non-class types).

When C++ calls Python, it's different — Python has its own runtime, its own stack of "frames" (PyFrameObjects), its own argument passing model. The C-Python API (`Py_BuildValue`, `PyArg_ParseTuple`) is the bridge.

When C++ calls Java via JNI, again — JNI exposes C-style functions that take a `JNIEnv*` and operate on Java objects through opaque handles. The mismatch in calling conventions (C++ stack frames vs JVM frames) is hidden by the JNI layer.

The general rule: **languages that share an ABI** (C, C++ with `extern "C"`, Rust with `extern "C"`, Zig with `export`) can call each other directly with little overhead. Languages with their own runtimes (Python, Java, Go, Node) require glue layers, and the glue is usually slow. This is why a hot loop calling from Python into a C extension is fast, but a hot loop calling from Python into native code via FFI repeatedly is not — the overhead is at the *boundary*, not in the C code itself.

---

## 6.12 Worked Example — A Hand Trace

```cpp
int multiply(int x, int y) { return x * y; }
int compute(int a, int b)  { return multiply(a + 1, b - 1); }
int main()                 { return compute(5, 3); }
```

Trace, x86-64 SysV, `-O0`:

```
Initial state:    rsp = 0x7fff_0000 (some high address)
                  rip = main's entry

1. main's prologue
   push rbp            ; rsp -= 8;  *rsp = old rbp
   mov  rbp, rsp

2. main calls compute(5, 3)
   mov  edi, 5         ; first arg
   mov  esi, 3         ; second arg
   call compute        ; rsp -= 8; *rsp = retaddr; jmp compute

3. compute's prologue
   push rbp            ; save main's rbp
   mov  rbp, rsp
   sub  rsp, 16        ; locals

4. compute calls multiply(a+1, b-1)
   mov  edi, [rbp+...] ; load a, add 1; first arg
   mov  esi, [rbp+...] ; load b, sub 1; second arg
   call multiply

5. multiply's prologue
   push rbp
   mov  rbp, rsp

6. multiply body
   mov  eax, edi       ; eax = x
   imul eax, esi       ; eax *= y

7. multiply epilogue
   pop  rbp
   ret                 ; pop retaddr -> rip; rsp += 8

8. back in compute, return value in eax
   ; compute uses it as its return; epilogue follows

9. compute epilogue
   leave               ; mov rsp, rbp; pop rbp
   ret                 ; back to main

10. back in main, return value in eax
    ; main returns eax to its caller (__libc_start_main)
```

At the deepest point (step 6, inside `multiply`), the stack contains, from low to high:
```
[rsp] -> multiply's saved rbp
         retaddr (back to compute)
         compute's locals
         compute's saved rbp
         retaddr (back to main)
         main's saved rbp
         (main's caller's frame, etc.)
```

This is exactly what a debugger shows when you ask for a backtrace.

---

## 6.13 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| Function call overhead (10-30 cycles) | Slows tight loops if not inlined | Modular code, separate compilation, debuggability |
| Aggressive inlining | Code bloat, instruction cache pressure | Eliminates overhead, enables further optimization |
| Frame pointer (`-fno-omit-frame-pointer`) | One register, two instructions/call | Cheap, accurate stack traces |
| Tail call optimization | Loses one stack frame in debugger | Unbounded recursion in tail position; cheaper calls |
| Calling conventions in registers (modern) | Limits how many fit before stack | Faster than memory-based arg passing |
| Exception handling | Unwind-table size, slow exception path | Decoupled error handling, automatic cleanup |
| Variadic arguments | ABI complexity, no static type checking | Flexible APIs (printf, log functions) |

---

## 6.14 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Function calls are slow." | A non-inlined call is ~20 cycles. Most calls are inlined in optimized builds. The overhead in real programs is rarely the bottleneck. |
| "Recursion is always expensive." | Tail-recursive calls are converted to jumps in optimized builds. Even non-tail recursion is rarely the bottleneck unless very deep. |
| "Local variables are stored on the stack." | They're stored *wherever the compiler likes* — often in registers, only spilled to the stack when needed. `&local` forces spilling. |
| "Calling conventions are an OS feature." | They're a CPU+OS+language combination. The same CPU has different conventions on Windows vs Linux, and within Linux differs by architecture. |
| "Stack traces always show every function." | Inlined functions may not appear (unless DWARF inlining info is preserved). Tail-called functions skip a frame. Optimizations make traces less obvious. |
| "Throwing an exception is free if there's no catch." | Throwing is expensive (microseconds). The "zero cost" applies to the *non-throwing* path. |

---

## 6.15 Exercises

1. **Disassemble a small call chain.**
   ```cpp
   int g(int x) { return x + 1; }
   int f(int x) { return g(x) * 2; }
   int main()   { return f(7); }
   ```
   Compile with `-O0 -S -masm=intel`. Identify, in the assembly: each prologue, each epilogue, each `call`, and where the return values are passed. Now compile with `-O2`. What's left?

2. **Frame pointer experiment.** Compile a moderately complex program with `-fno-omit-frame-pointer` and with `-fomit-frame-pointer`. Run it under a profiler (`perf`, Instruments). Which one gives more accurate stack traces? Run a microbenchmark: how much slower is the `-fno-omit-frame-pointer` version?

3. **Force a stack overflow.** Write:
   ```cpp
   int rec(int n) { return rec(n + 1); }
   int main()     { return rec(0); }
   ```
   Compile with `-O0` and `-O2`. What happens with each? Why? (Hint: the optimized compile will likely turn this into an infinite tail-call loop with no stack growth.)

4. **Trace by hand.** Pick any function from a real codebase that calls 2-3 other functions. Without running it, write down what the stack looks like at the deepest point. Then run it under `gdb`/`lldb`, set a breakpoint at the deepest point, and use `bt` to compare. How accurate was your prediction?

5. **Calling convention smoke test.** Write a function in C with 8 integer arguments. Compile, disassemble. Identify which argument went where. Confirm it matches the table in §6.2.

6. **Inlining experiment.** Write:
   ```cpp
   inline int hot(int x) { return x * x + 1; }
   int main() {
       int s = 0;
       for (int i = 0; i < 1'000'000; ++i) s += hot(i);
       return s;
   }
   ```
   Compile with `-O0`, `-O2`. Look for the call to `hot` in the disassembly of `-O2`'s `main`. Is it there? If not, what happened?

7. **Conceptual.** A coworker says, "Let's pass this 200-byte struct by value to all the helper functions." Explain (a) what the ABI does on a struct return larger than 16 bytes (caller passes hidden pointer), (b) why passing a 200-byte struct by value to many helpers is wasteful, (c) what `const&` does instead and why it's cheaper. Connect each answer to specific things from this chapter.

---

## 6.16 What's Next

You can now mentally execute a function call instruction-by-instruction. In Chapter 7 we zoom out and look at the call stack as a *whole structure*: how it composes, how stack traces work, what stack overflows look like, how coroutines and fibers stash and resume separate "stacks of frames," and why understanding this single structure unlocks debuggers, profilers, async runtimes, and exception handling all at once.

After Chapter 7 we'll spend Chapter 8 on pointers — the most-misunderstood feature in C++ and the one that ties Chapter 6's "where is this in memory" to Chapter 5's "what region does it live in." Then Chapters 9 and 10 close out Part 1 with the philosophical foundation for the rest of the book: why abstractions exist, and what languages actually do.

---


**[← Previous: Chapter 5 — Memory Layout Of A Process](05-memory-layout-of-a-process.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 7 — Call Stack Deep Dive →](07-call-stack-deep-dive.md)**
