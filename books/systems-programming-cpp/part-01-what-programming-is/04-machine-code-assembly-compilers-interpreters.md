# Chapter 4 — Machine Code, Assembly, Compilers, Interpreters

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the relationship between **source code, assembly, machine code, and bytecode** — and why each layer exists.
2. Describe what a **compiler** does in a small number of stages: lexing, parsing, semantic analysis, IR, optimization, code generation, linking.
3. Describe what an **interpreter** does and when interpretation is the right choice.
4. Define **JIT compilation** and explain why languages like Java, C#, JavaScript, and modern Python use it.
5. Read short pieces of x86-64 (or ARM64) assembly comfortably and recognize *what your high-level code became*.
6. Recognize why some languages "have a runtime" and others appear not to — and why every language has a runtime if you look hard enough.

This chapter is the bridge between human-readable code and machine-executable bytes. After it, you will never again look at a `.cpp` or `.py` file as a black box.

---

## 4.1 Three Layers of Code

There are essentially three forms code can take:

1. **Source code** — what you type. `int x = a + b;`. Designed for humans.
2. **An intermediate form** — designed for tools. Could be assembly (text) or bytecode (binary, for a virtual machine) or compiler IR (LLVM IR, GCC's GIMPLE).
3. **Machine code** — the binary bytes the CPU's decoder accepts. Designed for the silicon.

The journey from level 1 to level 3 can be long or short, eager or lazy, run once or run continuously. The choice of journey is *the defining choice* a language makes.

- **C, C++, Rust, Go, Zig** — *Ahead-of-time compiled.* Source → machine code, once, before you ship. Runtime cost: zero translation overhead.
- **Java, C#, Kotlin** — *Compile to bytecode for a virtual machine, JIT to machine code at runtime.* Bytecode is portable; the JIT specializes for the actual CPU and the actual data the program sees.
- **Python (CPython), Ruby, classic JavaScript** — *Compile to bytecode, then interpret it.* (Modern V8, PyPy, LuaJIT add a JIT on top.)
- **Bash, Tcl, classic shell scripts** — *Tree-walking interpretation.* Each statement is parsed (or pre-parsed to AST) and walked at runtime.

There is no "right" answer. Each choice trades different things: startup time, peak performance, portability, dynamism, complexity of implementation, debuggability.

---

## 4.2 What an Assembler Does

The first translation we will inspect is the most concrete: **assembly to machine code**.

Assembly is not a language so much as a textual notation for the CPU's instruction set. Each instruction in assembly corresponds, with rare exceptions, to one machine instruction. The assembler's job is mechanical: turn the text into bytes.

Take this single line of x86-64 assembly:

```asm
add rax, rbx
```

This means: "add the contents of register `rbx` to register `rax`, store the result in `rax`." The assembler emits 3 bytes for it: `48 01 D8`. Let's decode:

- `48` — the **REX prefix**, a byte that says "the operands are 64-bit."
- `01` — the **opcode** for "add r/m64, r64" form.
- `D8` — the **ModR/M byte**, a packed encoding of "the destination is `rax`, the source is `rbx`."

This level of detail does not need to be memorized. The point is: **machine code is a byte stream with a structured format that the CPU's decoder hardware can parse.** The assembler is a transparent text-to-bytes converter. It is not making decisions; it is encoding.

### Experiment 4.1 — see the bytes

Save as `tiny.s`:

```asm
.intel_syntax noprefix
.global _start
.text
_start:
    mov rax, 60        # syscall number for exit
    mov rdi, 42        # exit code
    syscall
```

Assemble it and inspect:

```bash
# Linux
as tiny.s -o tiny.o
ld tiny.o -o tiny
objdump -d tiny

# Then run it
./tiny
echo $?    # prints 42
```

`objdump -d` shows you *both* the bytes and the disassembled instructions side by side. Read it slowly. You wrote three lines; the assembler emitted ~15 bytes; those bytes are exactly what the CPU executes when you run `./tiny`.

This is the most direct possible piece of computing: source-to-byte-to-CPU, no runtime, no library, no allocator, just three instructions and a syscall. Understanding this is the foundation for understanding everything else.

---

## 4.3 What a Compiler Does

A compiler is much more than an assembler. Its job is to take *high-level source code* and produce *low-level code* that:

1. Means the same thing semantically (the language standard says what `int x = a + b;` means; the compiler must implement that meaning).
2. Runs as efficiently as possible on the target hardware.
3. Cooperates with all the other compiled units (separate `.cpp` files, libraries) to form a single program.

Modern compilers are organized in stages. Understanding the stages tells you *why* compiler errors look the way they do, *why* some "optimizations" exist, and *why* certain kinds of bugs only appear in optimized builds.

### The stages, briefly

Take a single C++ source file:

```cpp
int square(int x) { return x * x; }
```

#### 1. Lexing / Tokenization

The source text is split into **tokens**: `int`, `square`, `(`, `int`, `x`, `)`, `{`, `return`, `x`, `*`, `x`, `;`, `}`. Lexing is dumb — it doesn't know that `square` is a function name; it just knows it's an identifier.

#### 2. Parsing

Tokens are assembled into an **abstract syntax tree (AST)**: a tree representation of the program structure.

```
FunctionDecl(name="square", returnType=int, params=[Param(int, "x")], body=
  CompoundStmt(
    ReturnStmt(
      BinaryOp("*", Reference("x"), Reference("x"))
    )
  )
)
```

If the parser fails, you get a syntax error. (`expected ';' before '}'` and so on.)

#### 3. Semantic analysis

The AST is checked against the language's rules. Are types compatible? Are names in scope? Is `x` declared before use? Is `square` overloaded? Can the return type be deduced? *This* is where most C++ errors happen — `error: no match for 'operator+' (operand types are 'int' and 'std::string')` is the semantic-analysis stage saying "you wrote a syntactically valid expression that doesn't make sense in the type system."

#### 4. Lowering to intermediate representation (IR)

The semantically-checked AST is lowered into an **IR**. LLVM IR is the most famous; GCC has GIMPLE/RTL; MSVC has its own. IR is *language-agnostic but target-agnostic* — it has notions of "function," "basic block," "branch," "load," "store," "add," but no notion of "C++ class" or "x86 register."

Our function in LLVM IR (roughly):

```llvm
define i32 @_Z6squarei(i32 %x) {
entry:
  %mul = mul nsw i32 %x, %x
  ret i32 %mul
}
```

#### 5. Optimization

This is where the compiler earns its keep. Hundreds of small passes run over the IR:

- **Constant folding**: `2 + 3` becomes `5` at compile time.
- **Dead code elimination**: code that has no observable effect is deleted.
- **Inlining**: small function calls are replaced by the function's body.
- **Loop unrolling**: a loop of 4 iterations is rewritten as 4 sequential operations.
- **Vectorization**: scalar loops are rewritten using SIMD instructions to process 4–16 values at once.
- **Strength reduction**: `x * 2` becomes `x << 1`; `x * 8` becomes `x << 3`.
- **Common subexpression elimination**: `(a + b) * (a + b)` computes `a + b` once.
- **Devirtualization**: virtual calls whose target is statically knowable become direct calls.

Compilers can be aggressive. Code like:

```cpp
int sum = 0;
for (int i = 0; i < 1000000; ++i) sum += i;
return sum;
```

with `-O2` typically becomes a single `mov eax, 499999500000` because the compiler computed the result at compile time. There is no loop in the binary. **This is one of the most important practical facts about C++:** the source is a *specification of intent*, and the optimizer is permitted to produce *any* equivalent code, no matter how different from what you wrote.

#### 6. Code generation

The optimized IR is converted to actual machine code for the target CPU. This includes:

- **Register allocation**: deciding which IR-level "virtual registers" go in which physical registers, and which spill to the stack.
- **Instruction selection**: turning IR operations into the target's actual instructions (`mul` vs `imul`, `lea` for addition with shifts, etc.).
- **Scheduling**: reordering instructions to keep the CPU's pipeline full.

The output is an **object file** (`.o` on Linux/macOS, `.obj` on Windows): a file in the same family as ELF/Mach-O containing compiled code with *unresolved symbols*. Our `square.o` says "I provide `_Z6squarei`. I don't reference any external symbols."

#### 7. Linking

Multiple object files plus libraries are combined into a final executable or library. The linker:

- Concatenates `.text` from all `.o` files into the executable's `.text`.
- Resolves cross-file references: if `main.o` calls `_Z6squarei`, the linker fills in the actual address.
- Pulls in needed object files from static libraries (`.a` archives).
- Records dynamic library dependencies for the dynamic linker (Chapter 1).
- Produces the final ELF/Mach-O/PE executable.

### Experiment 4.2 — see the stages

```bash
# 1. Preprocessor only
clang++ -E hello.cpp | head -50

# 2. Source -> assembly (no assembling)
clang++ -S -O2 -masm=intel hello.cpp -o hello.s
cat hello.s

# 3. Source -> object file (no linking)
clang++ -c hello.cpp -o hello.o
nm hello.o          # see symbols this object file provides/needs

# 4. Object files -> executable (linking)
clang++ hello.o -o hello

# 5. Bonus: dump LLVM IR
clang++ -S -emit-llvm -O0 hello.cpp -o hello.ll
cat hello.ll
```

Each command performs a different *stage*. Understanding that you can stop at any stage demystifies what `clang++ hello.cpp -o hello` is actually doing.

---

## 4.4 What an Interpreter Does

An interpreter takes source code (or bytecode) and *executes it directly*, one piece at a time, without producing a separately stored machine-code artifact.

The simplest model is a **tree-walking interpreter**: the interpreter parses the source into an AST, then *walks* the AST, executing each node by looking at its kind:

```python
# pseudocode of a tree-walking interpreter
def eval(node, env):
    if node.kind == "literal":     return node.value
    if node.kind == "var":         return env[node.name]
    if node.kind == "binop":
        l = eval(node.left, env)
        r = eval(node.right, env)
        return apply(node.op, l, r)
    if node.kind == "if":
        if eval(node.cond, env): return eval(node.then, env)
        else:                    return eval(node.otherwise, env)
    ...
```

Tree-walking is conceptually clean and easy to implement. It is also slow: every operation involves examining the AST node's type, dispatching on it, recursing — with pointer chasing through a tree. A "real" CPU instruction takes 1 cycle; a tree-walked operation takes hundreds.

Most production interpreters use a **bytecode interpreter** instead. The source is compiled (once, at load time) to a flat array of pseudo-instructions for a virtual machine — an instruction set the language designer invented. The interpreter is a tight loop:

```c
// pseudocode of a bytecode VM
while (true) {
    Instr i = bytecode[pc++];
    switch (i.op) {
        case PUSH: stack[sp++] = i.arg; break;
        case ADD:  { int b = stack[--sp]; stack[sp-1] += b; break; }
        case JMP:  pc = i.arg; break;
        case CALL: ...
        case RET:  ...
    }
}
```

Compared to tree-walking, this is much faster: each instruction is a tiny, predictable operation in a tight loop, and modern CPUs can pipeline through it well. CPython, Ruby (MRI), Lua, and the JVM (in its baseline mode) all work this way.

### Experiment 4.3 — see Python's bytecode

```bash
python3 -c "import dis; dis.dis(compile('a + b * 2', '<x>', 'eval'))"
```

You'll see something like:

```
  0 LOAD_NAME                a
  2 LOAD_NAME                b
  4 LOAD_CONST               2
  6 BINARY_MULTIPLY
  8 BINARY_ADD
 10 RETURN_VALUE
```

This is exactly what the CPython interpreter executes when it runs `a + b * 2`. Now think about the cost: each one of these is a switch case, an unpacking of operands, a dispatch — *plus* every operand is a Python object (a heap-allocated structure), so `BINARY_MULTIPLY` does method-table lookups, type checks, allocator calls, and a real multiply. *That's* why Python is slow at numerics: not because Python is "interpreted," but because every operation pays a thick layer of dispatch and allocation.

---

## 4.5 JIT Compilation: Best of Both Worlds

A **JIT** (just-in-time) compiler is a runtime component that takes hot code (bytecode of frequently-run methods, or even hot interpreter loops) and compiles it to machine code on the fly.

The model:

1. The program starts up. All code is interpreted.
2. The runtime profiles execution: which methods run a lot? Which loops are hot?
3. When a method becomes hot, the JIT compiles it to machine code, optimized using *runtime information* (types observed, branches taken, polymorphic call targets seen).
4. The next call to that method jumps to the compiled code.
5. If runtime assumptions stop holding (a polymorphic call site sees a new type, for example), the JIT may **deoptimize** — fall back to the interpreter — and recompile later.

This is the architecture of:

- **HotSpot JVM** (Java): two JITs (C1 fast, C2 aggressive); aggressive inlining; based on observed runtime types.
- **V8** (JavaScript in Chrome/Node): tiers of compilation (Ignition interpreter, Sparkplug, Maglev, TurboFan); inline caches; hidden classes.
- **.NET CoreCLR**: tiered JIT.
- **PyPy** (alternative Python): tracing JIT; compiles loop traces.
- **LuaJIT**, **Julia**, modern **GraalVM**.

JITs are hard to write but produce code that is, in some workloads, *faster than ahead-of-time compiled C* — because they can specialize on runtime data that an AOT compiler cannot see. (E.g., if a JS function is always called with integers, the JIT can compile a version specialized for ints, with no boxing or type checks.)

The cost is **warmup**: until hot methods are compiled, you pay interpreter speed. This is why JVM benchmarks must be warmed up before measuring "real" performance, and why "cold-start" matters so much in serverless platforms — your code never gets warm enough to be JITted before it's killed.

### Experiment 4.4 — observe JIT warmup

In Java:

```java
public class Warmup {
    static long sum(int n) {
        long s = 0;
        for (int i = 0; i < n; i++) s += i;
        return s;
    }
    public static void main(String[] args) {
        for (int run = 0; run < 5; run++) {
            long t0 = System.nanoTime();
            long s = sum(100_000_000);
            long t1 = System.nanoTime();
            System.out.printf("run %d: %d ns, sum=%d%n", run, t1 - t0, s);
        }
    }
}
```

Run it. The first iteration is dramatically slower than later ones — the JIT compiles `sum` after the first few calls. This is JIT warmup in real time.

---

## 4.6 The Spectrum

Putting it together:

```
Slowest                                                   Fastest at peak
+---------+---------+---------+---------+----------+----------+
| Tree-   |Bytecode | JIT     | AOT     | AOT +     | Hand-     |
| walking |interp.  |  +      |compile  |  Profile  | written   |
| interp. |         | interp. |  (C/C++)|  -guided  | assembly  |
|         |         |  (JVM,  |         |  optim.   |           |
|         |         |   V8)   |         |  (LLVM    |           |
|         |         |         |         |   PGO)    |           |
+---------+---------+---------+---------+----------+----------+
   |          |         |         |          |           |
   v          v         v         v          v           v
Easy      Common      Complex,   Stable,   Stable,    Maximum
to write  for         huge       small     better     control
& change  scripting   runtime    runtime   peak       no help
                      Adapts to                       from tools
                      data
```

Each technology is the right answer for some context:

- **Tree-walking** for tiny domain-specific languages, configuration parsers.
- **Bytecode interpretation** for scripting languages where simplicity matters more than peak speed.
- **JIT** for long-running applications where adapting to runtime data wins.
- **AOT** for systems software, predictability, ship-once-run-anywhere binaries.
- **PGO/LTO AOT** for applications you ship with profiles (browsers, kernels, databases).
- **Hand assembly** for crypto inner loops, vectorized hot paths in libraries (BLAS, ffmpeg).

---

## 4.7 The Universal Truth: Every Language Has a Runtime

A common confusion: "C++ has no runtime, but Java does." This is wrong; the right statement is "C++ has a *small* runtime, and Java has a *large* runtime."

C++'s runtime includes:

- The C runtime (`crt1.o`, `__libc_start_main`) that runs before `main`.
- The `std::` library when used (allocators, container internals).
- Exception handling support (unwind tables and the unwinder).
- Static-storage initializers and at-exit handlers.
- TLS infrastructure.

Java's runtime is just *bigger*:

- The garbage collector.
- The JIT.
- The classloader and verifier.
- The bytecode interpreter.
- The reflection machinery.

Go's runtime:

- The scheduler (goroutines, M:N).
- The garbage collector.
- The channel and select implementation.
- The memory allocator.

Python's runtime:

- The interpreter loop.
- The garbage collector (reference counting + cycle collector).
- The object system (every value is a heap-allocated object).

The pattern is universal. **Languages provide a runtime to make their abstractions real.** When Java says "you can throw an exception across a thread boundary," the runtime catches it. When Go says "this goroutine sleeps until the channel has data," the runtime schedules it. When C++ says "this destructor runs at the end of scope," the compiler emits the call and the runtime's exception handler ensures it runs even when an exception unwinds through.

> **The right question is never "does this language have a runtime?" The right question is "what does this language's runtime do for me, and what does it cost?"**

---

## 4.8 Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| AOT compile | Must rebuild for new platforms; slow build times; less runtime adaptivity | Fast startup, no runtime warmup, predictable performance, smaller deploy footprint |
| Bytecode interpret | Slow per operation; higher memory use | Portable bytecode; quick iteration; simple to implement |
| JIT compile | Warmup latency; complex implementation; harder to debug | Adapts to runtime data; can outperform AOT on dynamic code |
| Aggressive optimization | Surprising debugger behavior; longer build times; harder to reason about | Sometimes 10× speed; vectorization; inlining |
| Static linking | Big binaries; can't share library memory across processes | Single artifact; immune to "dependency hell"; faster startup |
| Dynamic linking | Shared library memory; per-process startup work | Smaller binaries; shared upgrades; plugin systems |

---

## 4.9 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Compiled languages are always faster than interpreted." | A well-warmed JIT can match or beat AOT C++ on dynamic workloads. The cost is warmup. |
| "Python is slow because it's interpreted." | Python is slow primarily because every value is a heap-allocated object. The interpreter is a contributor, not the only one. |
| "C++ has no runtime." | It has a small one. `__libc_start_main`, the unwinder for exceptions, static initializers, TLS, allocator internals. |
| "Assembly is hard to read." | Different from your source, but most x86-64 assembly is readable with a register-name cheat sheet and an hour of practice. |
| "The compiler does what I tell it to." | The compiler does what *the language standard says is equivalent to* what you wrote. The actual instructions can be wildly different — that's the optimizer's job. |
| "Optimization just makes things faster." | Optimization can also expose latent bugs (UB), change observable behavior in ways that violate the standard if you wrote UB, and reorder operations across thread boundaries if you didn't use atomics. |

---

## 4.10 Exercises

1. **Look at what your code becomes.** Pick five small functions you wrote recently in your day-job language. Compile or transpile them to a lower form (C++ → assembly with `-S`; Java → bytecode with `javap -c`; Python → bytecode with `dis.dis`). For each, write 1–2 sentences describing one thing you did not expect.

2. **Watch the optimizer work.** Compile this with `-O0`, `-O1`, `-O2`, `-O3` and inspect the assembly each time:

   ```cpp
   int triangular(int n) {
       int s = 0;
       for (int i = 1; i <= n; ++i) s += i;
       return s;
   }
   ```

   At what optimization level does the loop disappear and become a closed-form expression? Read the resulting instructions; can you match them to the formula `n*(n+1)/2`?

3. **The cost of dispatch.** Write the same arithmetic-heavy benchmark in C, Python (CPython), and one of: Java, Go, or JavaScript (Node). Measure all three. Then disable the JIT in your JIT'd language (e.g., `java -Xint`, `node --jitless`) and re-measure. Explain what each measurement tells you.

4. **Lex your own language.** Write a 50-line tokenizer for a tiny language with integers, +, -, *, parens, and identifiers. Then write a parser that produces an AST. Then write a tree-walking evaluator. Compare its speed (operations per second) to a hand-rolled C version doing the same arithmetic. Now you have *felt* why bytecode interpreters and JITs exist.

5. **Read your binary.** Take any C++ program. Use `objdump -d -C` (Linux) or `otool -tV` (macOS) to disassemble it. Find `main`. Read its assembly. How many instructions is your `main`? Locate the call to `__libc_start_main` (or trace from `_start`).

6. **Inspect linker behavior.** Compile two `.cpp` files separately:

   ```bash
   clang++ -c a.cpp -o a.o
   clang++ -c b.cpp -o b.o
   ```

   Use `nm a.o` to see what symbols `a.o` defines (`T`) and references but doesn't define (`U`). Now link: `clang++ a.o b.o -o app`. The linker resolved every `U`. Try it with a missing symbol and observe the error. *That error is the linker's primary failure mode.*

7. **Conceptual.** A coworker says "let's rewrite this Python service in Rust for speed." Write three questions you'd ask before agreeing — drawing on what you learned about why Python is slow. (Hint: is the bottleneck CPU work, IO, allocation, or something else? Have you measured?)

---

## 4.11 What's Next

You now have the entire vertical picture: from your source code, through compilation or interpretation, into a binary, loaded by the kernel, executing on a CPU with a layered memory system. That's the substrate.

Chapter 5 zooms back in and looks at the **memory layout of a process** in detail — armed now with the language to talk about it. We'll dissect a real process map, figure out what every region is for, and build the mental model you'll use every time you debug a memory issue, design a data structure, or wonder where your variable lives.

After Chapter 5, we move into the *mechanics* — function calls, the call stack, pointers — that turn the layout into actual computation.

---


**[← Previous: Chapter 3 — How Operating Systems Execute Programs](03-how-os-executes-programs.md)** · **[Up: Part 1](README.md)** · **[Next: Chapter 5 — Memory Layout Of A Process →](05-memory-layout-of-a-process.md)**
