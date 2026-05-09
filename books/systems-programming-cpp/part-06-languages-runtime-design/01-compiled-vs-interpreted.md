# Chapter 58 — Compiled vs Interpreted

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain why the **compiled/interpreted dichotomy is real but oversimplified** — most modern languages sit between the extremes.
2. Identify where a language sits on the **compilation spectrum**: pure interpretation, bytecode VM, AOT compilation, JIT, or hybrid.
3. Describe the **concrete costs and benefits** of each approach: startup time, peak performance, build time, binary size, memory overhead, and debuggability.
4. Read the actual spectrum of Python, Java, JavaScript, Go, Rust, and .NET — and understand why each made its choice.
5. Predict, given a language's position on this axis, what its runtime characteristics will be.

This chapter lays bare the false binary. You will see that the most successful modern languages do not pick "compiled" or "interpreted"; they pick a *point on a continuous spectrum*, and often move along that spectrum at runtime.

---

## 58.1 The Spectrum is Real, but It's Not Binary

Chapter 4 introduced the spectrum: tree-walking → bytecode → JIT → AOT. This chapter is not a repeat; it is a *deepening*. We now have the context to explain why the spectrum matters, where modern languages actually sit, and what each point costs you in practice.

The opening mistake is thinking "compiled" and "interpreted" are opposites. They are not. They are endpoints of a spectrum. And almost nothing of real importance sits at the endpoints.

```
Start of program execution                              Peak performance (warm code)
 |                                                       |
 v                                                       v
+---------+---------+----------+---------+--------+--------+--------+
|  Pure   |Bytecode | Bytecode | AOT     | JIT+   | AOT w/ | Hand-  |
|  tree-  |  VM     | + deopt  |compile  | fallback|profile | written|
| walking | (CPy,  | (PyPy,   |(C,Rust, |(Java   | -guided | asm    |
| (bash,  | Lua)   | LuaJIT)  | Go)     |HotSpot)|optim   |        |
| bash)   |        |          |         |        |(LLVM   |        |
|         |        |          |         |        | PGO)   |        |
+---------+---------+----------+---------+--------+--------+--------+
Startup:   Instant  Instant   Instant   Instant  Instant  Build    Instant
very fast  Load &   Load &    Compile   Compile  Compile  takes
           interp   interp    once      once     once     seconds
           Loop     Loop

Peak:      Slow     Fast      Very      Very     Fastest  Near-    Fastest
           1-10x    3-10x     fast      fast     possible impossible
                    slower    than JIT  with
                    than JIT  warmup
```

The key insight: **no language is purely at one point**. Most modern languages have moved or are moving toward the *middle-right* — combining ahead-of-time and just-in-time strategies, or offering multiple tiers.

---

## 58.2 Pure Interpretation: The Baseline

A **pure interpreter** reads source (or bytecode) and executes it directly, statement by statement, without producing a separate executable artifact. No build step between edit and run.

### How it works

The interpreter is a loop that reads and executes:

```c
// Pseudocode: tree-walking interpreter
void interpret(ASTNode *root, Environment *env) {
    switch (root->kind) {
        case LITERAL:
            return root->value;
        case VAR:
            return env->lookup(root->name);
        case BINOP:
            left = interpret(root->left, env);
            right = interpret(root->right, env);
            return apply(root->op, left, right);
        case IFSTMT:
            cond = interpret(root->cond, env);
            if (cond) return interpret(root->thenBranch, env);
            else return interpret(root->elseBranch, env);
    }
}
```

Each operation involves:
1. Examining the AST node's type.
2. Dispatching to the correct handler.
3. Recursing to evaluate sub-expressions.

This is simple, trivial to implement (200 lines of code for a toy language), and *slow*. A single tree-walked add operation in a real interpreter touches hundreds of CPU instructions: a virtual function call, a switch, pointer chasing, type checks.

### Examples

- **Bash, sh, zsh** — command shells.
- **Tcl** — tiny, tree-walking.
- **Early BASIC** — Commodore, Apple II.
- **SQL evaluators** in some databases.
- **Vimscript**, **Lua** (before LuaJIT).

### Costs

- **Startup**: instant (load source, start interpreting).
- **Runtime speed**: 1–10× slower than bytecode. 100–1000× slower than JIT'd or compiled code.
- **Memory overhead**: AST in memory, interpreter state, no compile-time optimization.

### Benefits

- **Immediate iteration**: edit, run, instantly.
- **Language design freedom**: easy to add features (just add a case to the switch).
- **Portable**: interprets on any platform with the interpreter binary.
- **Debuggable**: AST reflects source structure exactly; debugging is mechanical.

### When to use

Domain-specific languages, configuration files, embedded scripting, education. Anywhere "reasonably fast" is good enough and programmer productivity beats peak speed.

---

## 58.3 Bytecode Virtual Machines

A **bytecode interpreter** compiles source to a portable intermediate form — pseudo-instructions for a virtual machine — then interprets the bytecode. This is a sweet spot for many languages.

### How it works

1. **Compile phase** (at load time, once): Source → bytecode. The compiler produces a flat array of pseudo-instructions.

```python
# Source
a + b * 2

# Bytecode (Python-like)
LOAD_NAME    a
LOAD_NAME    b
LOAD_CONST   2
BINARY_MULTIPLY
BINARY_ADD
RETURN_VALUE
```

2. **Interpret phase** (at runtime): A tight loop runs the bytecode.

```c
// Pseudocode: bytecode VM loop
while (true) {
    Instruction inst = bytecode[pc++];
    switch (inst.opcode) {
        case LOAD_NAME:
            push(env->lookup(inst.arg));
            break;
        case BINARY_ADD:
            b = pop(); a = pop();
            push(a + b);
            break;
        // ... etc
    }
}
```

This is still interpretation, but *much* faster than tree-walking because:
- Instructions are in a flat array (cache-friendly).
- The dispatch loop is tight and predictable.
- No tree traversal, no repeated type checks.

### Examples

- **CPython** (`.pyc` bytecode).
- **Ruby (MRI)** (YARV bytecode).
- **Lua** (before LuaJIT).
- **Java's baseline interpreter** (before HotSpot JIT).
- **JVM** when run with `-Xint` (interpreter-only).
- **BEAM** (Erlang/Elixir's bytecode).
- **.NET's IL** (before JIT).

### Costs

- **Build time**: minimal (compile at load time).
- **Startup**: fast (load bytecode, interpret).
- **Runtime speed**: 3–10× slower than JIT'd code. Still slow for numerics.
- **Bytecode size**: compact; smaller than source or machine code.
- **Memory overhead**: VM runtime, bytecode, no aggressive optimization.

### Benefits

- **Portability**: bytecode is platform-independent. Compile once, ship everywhere.
- **Simplicity**: easier to implement than JIT; no platform-specific code generation.
- **Security**: sandboxing is easier with a VM boundary.
- **Iteration speed**: no build step; run bytecode immediately.

### When to use

Languages where portability matters (Java, Python, Lua), where the implementation must be simple, or where the workload is not performance-critical (scripting, configuration, light web servers).

---

## 58.4 Ahead-of-Time Compilation to Native

An **AOT compiler** translates source to machine code for a specific target CPU *once*, before the program is shipped. The compiled binary is then run directly by the CPU.

### How it works

```
Source code
    ↓
[Lexing, parsing, semantic analysis]
    ↓
Intermediate representation (IR)
    ↓
[Optimization passes: inlining, vectorization, strength reduction, etc.]
    ↓
Code generation (for x86-64, ARM, etc.)
    ↓
Object files (.o) + linking
    ↓
Final executable (ELF/Mach-O/PE)
    ↓
Run directly on CPU
```

Each stage is a chance for optimization:

- **Inlining**: small functions are merged into callers at compile time, eliminating call overhead.
- **Constant folding**: `2 + 3` is computed at compile time to `5`.
- **Loop unrolling**: a loop of 4 iterations is manually unrolled into 4 sequential blocks.
- **Vectorization**: scalar operations are rewritten using SIMD instructions.
- **Devirtualization**: if a virtual call's target is statically known, it becomes a direct call.
- **Dead code elimination**: unreachable code is deleted.

The optimizer is aggressive. Code like:

```cpp
int sum = 0;
for (int i = 0; i < 1000000; ++i) sum += i;
return sum;
```

becomes `return 499999500000` — no loop in the binary. The compiler computed it at compile time.

### Examples

- **C, C++**.
- **Rust**.
- **Go**.
- **Zig**.
- **Fortran**.
- **Swift** (to native; mostly AOT, though there is some JIT for REPL).

### Costs

- **Build time**: slow. Compiling a large C++ codebase takes minutes to hours.
- **Startup**: instant (binary is ready to run).
- **Runtime speed**: very fast; optimizations are aggressive.
- **Binary size**: large (code for all paths; no dead code in some cases).
- **Debugging**: harder. Optimizations can reorder code, inline functions, eliminate variables. Stepping through optimized code can be confusing.
- **Portability**: must recompile for each target CPU/OS.

### Benefits

- **Startup**: zero translation overhead; the binary runs immediately.
- **Peak performance**: no warmup. The binary is optimized at compile time for the average case (or profile-guided).
- **Predictability**: performance is stable from first execution.
- **Offline**: no runtime compilation complexity.

### When to use

- **Systems software**: kernels, databases, compilers.
- **Latency-sensitive code**: financial systems, embedded systems, real-time.
- **Code size matters**: embedded systems with limited memory.
- **Predictability is critical**: aerospace, automotive.

---

## 58.5 JIT Compilation: Compiling at Runtime

A **JIT** compiler observes which code runs hot (frequently), and compiles it to native machine code *on the fly*, using runtime information unavailable at AOT compile time.

### How it works

1. **Startup**: All code is interpreted (bytecode VM or AST walking).
2. **Profiling**: The runtime tracks which functions are called repeatedly, which loops run hot.
3. **Triggering**: When a function's call count or loop trip count hits a threshold, the JIT compiles it.
4. **Specialization**: The JIT uses *runtime data* to generate optimized code:
   - If a variable is always seen as an integer, the JIT emits integer code (no type checks).
   - If a polymorphic call always sees the same type, the JIT inlines it.
5. **Deoptimization**: If assumptions break (a polymorphic call sees a new type, a branch is taken unexpectedly), the JIT falls back to the interpreter and may recompile.

```
Program start
    ↓
Interpret all code (slow)
    ↓
Profile: which code is hot?
    ↓
JIT compile hot code using runtime types (fast path)
    ↓
Alternate between compiled + interpreted as assumptions hold
    ↓
If assumption breaks, deoptimize and recompile
```

### Examples

- **HotSpot JVM** (Java): two JITs (C1 fast compiler, C2 aggressive). Profiling at every call site. Inline caches and devirtualization.
- **V8** (JavaScript): multiple tiers (Ignition interpreter, Sparkplug simple JIT, Maglev mid-tier, TurboFan aggressive). Feedback vectors track observed types.
- **.NET CoreCLR**: Tiered JIT (quick first-pass, then aggressive optimization).
- **PyPy** (Python alternative): tracing JIT. Traces loops and compiles the trace.
- **LuaJIT**, **Julia**, modern **GraalVM**.

### Costs

- **Warmup**: until hot code is compiled, performance is interpreter-speed. A web service must handle requests slowly until it warms.
- **Complexity**: JIT is hard to write. Parsing, IR, code generation, GC-aware pointers, deoptimization — all at runtime.
- **Unpredictability**: first request is slow. Benchmarks must warm up before measuring.
- **Debugging**: JIT'd code is harder to step through; stepping may jump around.
- **Memory overhead**: JIT'd code lives in memory; code cache limits apply.

### Benefits

- **Specialization**: uses runtime data (types, values, branch patterns) unavailable at AOT compile time.
- **Adaptivity**: can optimize for the *actual* data the program sees.
- **Peak performance**: can match or beat AOT C++ on workloads the AOT compiler cannot predict.
- **Portability**: bytecode is portable; JIT compiles for the actual target at runtime.

### When to use

- Long-running services (web servers, app servers) where warm code runs.
- Dynamic languages (Python, JavaScript, Java) where static type information is unavailable.
- Workloads where the AOT compiler cannot know what data you'll see.

---

## 58.6 Hybrid Approaches: The Modern Mainstream

Nearly every successful modern language now uses *multiple* strategies at once. This is where the spectrum matters most in practice.

### Tiered compilation

Start slow, move to fast:

```
Tier 0: Interpreter (instant start)
  ↓ (after 100 calls)
Tier 1: Quick JIT (less aggressive; fast to compile)
  ↓ (if hot)
Tier 2: Aggressive JIT (interprocedural optimization, vectorization)
```

This is what **Java HotSpot**, **.NET CoreCLR**, and **V8** do. It balances startup (fast) and peak performance (very fast).

### Python's fragmented approach

- **CPython** (reference implementation): bytecode interpreter.
- **PyPy**: tracing JIT.
- **Cython**: AOT compile to C, then C compiler.
- **mypyc**: JIT'd type-checked Python.
- **Numba**: JIT numerical kernels.
- **Jython**: Python on JVM (gets Java's JIT for free).

There is no single "Python execution model"; programmers reach for whichever tool fits their bottleneck.

### GraalVM: multi-language JIT

GraalVM is a single JIT'd runtime for Java, JavaScript, Python, R, and more. Code written in any language compiles to an intermediate representation (Graal IR), then the JIT compiles it to native machine code. Polyglot code (Java calling Python, etc.) shares optimizations.

### Go: AOT with a light runtime

Go compiles to native code ahead of time. But the runtime includes:
- A garbage collector.
- A scheduler (goroutines).
- Channels and select.

It is neither pure AOT nor a heavy runtime like Java.

### Rust: AOT + LLVM

Rust compiles to LLVM IR, then LLVM compiles to native code. The compilation is AOT, but the LLVM optimizer is aggressive enough that some JIT effects appear naturally (it can see the whole program, unlike C).

Rust also supports:
- `wasm32-unknown-unknown`: compile to WebAssembly, run in the browser.
- WASI: WebAssembly System Interface; WebAssembly can run on servers.

This gives Rust a form of portability without bytecode.

### .NET Native AOT

Historically, .NET was bytecode (IL) + JIT. Recently, **.NET Native AOT** mode can compile C# to native code *without the JIT*:

```csharp
// dotnet publish -c Release -p:PublishAot=true
// Produces a native binary like C++
```

This gives C# the startup and binary-size benefits of AOT while keeping the language.

### Java's AOT (GraalVM native-image)

Similarly, you can use **GraalVM native-image** to compile Java to a native binary ahead of time, trading JIT flexibility for startup and memory:

```bash
native-image --class-path app.jar MyApp
# Produces a native binary
```

This is why Java is increasingly viable in serverless (AWS Lambda) — traditionally, JVM warmup was a killer. AOT compilation fixes it.

---

## 58.7 The Real Tradeoffs: Costs in Practice

This table captures the tradeoffs concretely. Each point assumes typical usage:

| Approach | Startup | Peak Perf | Build Time | Binary Size | Memory at Runtime | Warmup Unpredictability | Debuggability |
|---|---|---|---|---|---|---|---|
| **Tree-walking** | Instant | 1x (baseline) | Instant | Small | Moderate | None | Excellent |
| **Bytecode VM** | <1s | 3–10x | <1s | Small | Moderate | None | Good |
| **AOT compile** | Instant | 10–50x | Minutes+ | Large | Low | None | Hard (if `-O2`) |
| **JIT** | Slow (1–5s) | 10–50x | Variable | Moderate | Moderate-High | High | Hard |
| **JIT + interp** | Fast (100ms) | 10–50x | Instant | Moderate | Moderate-High | Moderate | Hard |
| **JIT + tier 1+2** | Medium (500ms) | 10–50x | Instant | Moderate | High | Low | Hard |
| **AOT + PGO** | Instant | 50–100x | Hours | Large | Low | None | Very hard |

Legend:
- **Startup**: time from launching the program to first user request served.
- **Peak perf**: operations per second on hot code, relative to tree-walking.
- **Build time**: time from code change to deployable binary.
- **Binary size**: relative size of the artifact.
- **Memory at runtime**: heap memory, code cache, runtime structures.
- **Warmup unpredictability**: how much does performance vary before warm?
- **Debuggability**: how easy to step through, set breakpoints, inspect variables.

The *Pareto frontier* is the middle-right: JIT with tiers. Start fast enough for interactive use, reach peak performance quickly, and avoid the build-time overhead of pure AOT.

---

## 58.8 Worked Example: Computing Primes

Let's run the same algorithm in multiple languages and compare startup, runtime, and memory.

**The algorithm**: Sieve of Eratosthenes, find all primes up to 1,000,000.

### C++ (AOT compiled)

```cpp
#include <iostream>
#include <vector>
#include <chrono>

int main() {
    auto t0 = std::chrono::high_resolution_clock::now();
    
    std::vector<bool> sieve(1000001, true);
    sieve[0] = sieve[1] = false;
    
    for (int i = 2; i * i <= 1000000; ++i) {
        if (sieve[i]) {
            for (int j = i * i; j <= 1000000; j += i) {
                sieve[j] = false;
            }
        }
    }
    
    int count = 0;
    for (bool p : sieve) if (p) count++;
    
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count();
    
    std::cout << "Count: " << count << " Time: " << ns / 1e6 << " ms\n";
}
```

Compile and run:

```bash
$ g++ -O2 primes.cpp -o primes
$ time ./primes
Count: 78498 Time: 2.35 ms
```

**Startup**: <1ms (overhead of C runtime). **Runtime**: 2.35ms. **Binary**: 17KB.

The optimizer recognizes the sieve and generates tight, cache-friendly code.

### Python (CPython bytecode interpreter)

```python
def sieve():
    sieve = [True] * 1000001
    sieve[0] = sieve[1] = False
    
    i = 2
    while i * i <= 1000000:
        if sieve[i]:
            j = i * i
            while j <= 1000000:
                sieve[j] = False
                j += i
        i += 1
    
    return sum(1 for x in sieve if x)

import time
t0 = time.time()
count = sieve()
t1 = time.time()
print(f"Count: {count} Time: {(t1 - t0) * 1000:.2f} ms")
```

Run:

```bash
$ python3 primes.py
Count: 78498 Time: 162.34 ms
```

**Startup**: ~50ms (Python initialization). **Runtime**: ~112ms (algorithm). **Memory**: ~20MB (Python object overhead, GC structures).

The CPython interpreter is ~70× slower than C++. Every `sieve[i]`, every `i * i`, every loop increment involves bytecode interpretation, method dispatch, type checking, allocation.

### PyPy (tracing JIT)

```python
# Same code as CPython
```

Run:

```bash
$ pypy3 primes.py
Count: 78498 Time: 4.12 ms
```

**Startup**: ~1s (PyPy JIT initialization and warmup). **Runtime**: ~4ms (after warmup). **Memory**: ~80MB (JIT code cache).

PyPy's tracing JIT sees the hot loop, specializes on the observed types (list of bools), and compiles the trace to machine code. Once warm, it is nearly as fast as C++. But the startup cost and memory overhead are significant.

### Go (AOT compile)

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	t0 := time.Now()
	sieve := make([]bool, 1000001)
	for i := range sieve {
		sieve[i] = true
	}
	sieve[0], sieve[1] = false, false
	
	for i := 2; i*i <= 1000000; i++ {
		if sieve[i] {
			for j := i * i; j <= 1000000; j += i {
				sieve[j] = false
			}
		}
	}
	
	count := 0
	for _, p := range sieve {
		if p {
			count++
		}
	}
	
	t1 := time.Now()
	fmt.Printf("Count: %d Time: %.2f ms\n", count, t1.Sub(t0).Seconds()*1000)
}
```

Compile and run:

```bash
$ go build -o primes primes.go
$ time ./primes
Count: 78498 Time: 1.85 ms
```

**Startup**: <1ms. **Runtime**: 1.85ms. **Binary**: 2.3MB (Go includes a light runtime for GC, scheduling, etc.).

Go is nearly as fast as C++, with simpler syntax and a built-in runtime for concurrency.

### Java (JIT'd bytecode)

```java
public class Primes {
    public static void main(String[] args) {
        long t0 = System.nanoTime();
        
        boolean[] sieve = new boolean[1000001];
        for (int i = 0; i < sieve.length; i++) sieve[i] = true;
        sieve[0] = sieve[1] = false;
        
        for (int i = 2; i * i <= 1000000; i++) {
            if (sieve[i]) {
                for (int j = i * i; j <= 1000000; j += i) {
                    sieve[j] = false;
                }
            }
        }
        
        int count = 0;
        for (boolean p : sieve) if (p) count++;
        
        long t1 = System.nanoTime();
        System.out.printf("Count: %d Time: %.2f ms%n", count, (t1 - t0) / 1e6);
    }
}
```

Compile and run:

```bash
$ javac Primes.java
$ time java Primes
Count: 78498 Time: 125.34 ms

$ time java Primes  # second run, after warmup
Count: 78498 Time: 2.41 ms
```

**First run (cold)**: 125ms (JVM startup ~100ms, bytecode interpretation).
**Second run (warm)**: 2.41ms (HotSpot JIT compiled).

The JIT warmup is clear. Cold startup is bad for one-off scripts, but in a long-running server, the code warms and becomes very fast.

### Summary

```
Language     | Startup | First Run | Warm | Memory | Binary
---|---|---|---|---|---
C++          | <1ms    | 2.35ms    | N/A  | Low    | 17KB
Python       | 50ms    | 162ms     | 162ms| 20MB   | Small
PyPy         | 1000ms  | 4.12ms    | 4.12ms| 80MB  | Moderate
Go           | <1ms    | 1.85ms    | N/A  | Low    | 2.3MB
Java (cold)  | 100ms   | 125ms     | N/A  | High   | N/A
Java (warm)  | 100ms   | 2.41ms    | N/A  | High   | N/A
```

**Key observations:**

- **C++ and Go**: instant startup, fast runtime. No warmup. Simple deployment.
- **Python**: reasonable for interactive scripts; terrible for latency-sensitive code.
- **PyPy**: after warmup, nearly matches C++. But startup and memory are high.
- **Java**: cold startup is slow (JVM overhead), but warm performance is excellent. Ideal for long-running servers.

**The right choice depends on the workload:**
- **One-off scripts**: C++, Go, bash.
- **Interactive development**: Python, PyPy, Node.
- **Long-running servers**: Java, Go, Rust, C++.
- **Serverless** (AWS Lambda, where instances are short-lived): Go, Rust, native C.

---

## 58.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Compiled is always faster than interpreted." | A warmed JIT can match or beat AOT C++ on dynamic code. The cost is warmup and memory. One-off scripts in C++, despite compilation, are slower than Python because compile time dominates. |
| "Bytecode is just a crutch for portability." | Bytecode is *also* a performance win over tree-walking interpretation. The VM loop is fast; the IR is optimizable. |
| "JIT is just-in-time; it adds delay." | For long-running code, JIT's *specialization* wins. It optimizes for the *actual* data, not the worst case. |
| "Pure AOT is always best." | AOT is best for predictability and startup, worst for adapting to runtime data. Dynamic languages need JIT or they stay slow. |
| "GC is the only cost of the JVM." | GC is one of many: JIT code cache, bytecode, runtime profiling infrastructure, GC pauses themselves. Total memory overhead can be 5–10× C++. |
| "A JIT interpreter is just interpretation." | No. Once hot code is JIT'd to machine code, it runs at compiled speed. The "interpreter" part is only for cold code. |
| "Startup time doesn't matter." | It matters enormously for serverless (where instances live seconds), command-line tools, and embedded systems. It is irrelevant for long-running servers. |
| "The spectrum is one dimension." | It is not. A language can be ahead-of-time *and* bytecode (Kotlin on JVM), or JIT *and* bytecode (Java), or pure AOT *and* portable (Go can cross-compile). The axes are independent. |

---

## 58.10 Exercises

1. **Measure startup + runtime.** Pick three languages you know (or learn quickly). Write a simple program (compute factorial of 5000, or find primes to 100,000). Measure:
   - Time from program start to printing the answer (including any interpreter/JVM initialization).
   - Actual computation time (by printing timestamps before and after the loop).
   - Binary/artifact size.
   
   Create a table. Which is best for one-off scripts? Which for long-running servers?

2. **Inspect bytecode.** For a language you use (Python, Java, .NET, Lua), examine the bytecode of a small function:
   - Python: `python3 -m dis` or `import dis; dis.dis(func)`.
   - Java: `javap -c ClassName`.
   - .NET: use `ildasm` or a decompiler.
   
   Write 2–3 sentences about what you see. Does the bytecode look like what you expected?

3. **Force interpreter mode.** For a JIT'd language (Java, JavaScript, Python), run a benchmark both with and without JIT:
   - Java: `java -Xint yourprogram` vs `java yourprogram`.
   - Node: `node --jitless script.js` vs `node script.js`.
   - PyPy: compare PyPy (JIT) vs CPython (no JIT).
   
   Measure the difference. Write a paragraph explaining why.

4. **Warmup observation.** In Java or C#, write a loop that runs a function 100 times and prints the time for each iteration. Watch the first few iterations be slow, then suddenly jump to fast. That is JIT compilation in real time.

5. **Hypothetical migration.** A coworker proposes rewriting a Python service in Java "for speed." The service processes 1000 requests/sec, averaging 10ms per request. Write three questions you'd ask before agreeing, based on what you learned in this chapter about compilation models. (Hint: startup, warmup, request profile.)

6. **Conceptual.** You are given control over a language's execution model. Your constraints:
   - Fast startup (must be <100ms for CLI tools).
   - Fast peak performance (must reach C++ speeds on hot code).
   - Small binary (deploy to embedded systems; <5MB preferred).
   
   Describe your execution strategy. What combination of compilation strategies would you use? What would you sacrifice?

---

## 58.11 Summary

The compiled/interpreted dichotomy is a false binary. Modern languages sit on a continuous spectrum from pure interpretation to aggressive AOT compilation, and many use multiple strategies at once.

**The spectrum**:
- **Pure interpretation** (tree-walking): instant startup, slow runtime. Rarely used for performance-critical code.
- **Bytecode VMs** (CPython, Ruby): fast startup, 3–10× slower than compiled. Portable.
- **JIT compilation** (Java, V8): slow startup, very fast peak after warmup. Specializes on runtime data.
- **Tiered JIT** (modern Java, .NET): balances startup and peak performance.
- **AOT compilation** (C++, Rust, Go): fast startup, fast runtime, slow build. No warmup.
- **Hybrid** (GraalVM, .NET Native AOT): combines portability with native-code performance.

**Practical costs**:
- **Startup matters** for one-off scripts, CLI tools, serverless.
- **Peak performance matters** for long-running servers, numerical compute.
- **Binary size matters** for embedded systems, containers.
- **Memory overhead matters** for resource-constrained environments.
- **Build time matters** for developer iteration.

The right choice is always contextual. A systems engineer should understand where their language sits on this spectrum, and be ready to reach for a different tool if it fits better.

---

> **[← Previous: Part 5 — Memory Ordering](../part-05-runtime-concurrency/12-memory-ordering.md)** · **[↑ Part 6](README.md)** · **[Next: JIT Compilation →](02-jit-compilation.md)**
