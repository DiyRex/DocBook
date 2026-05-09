# Chapter 59 — JIT Compilation

You have a method running 10 million times per second. The first 100 calls, the interpreter's dispatch loop executes it. The interpreter doesn't optimize it; it just reads bytecode instructions and shuffles data around. But on call 101, the runtime has watched the method enough. It hands the bytecode to a compiler, which produces machine code, and the next call jumps straight to that machine code. By the time the method has run a million times, it runs at speeds that can match or exceed ahead-of-time compiled C++.

This is JIT — just-in-time compilation. The trick is doing it without making the program slower than pure interpretation would have been.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what a JIT compiler does and why languages like Java, C#, JavaScript, and modern Python use one.
2. Describe the **tiered strategy** — why production JITs don't compile once, but instead compile multiple times with increasing aggressiveness.
3. Understand **profile-guided optimization** — how runtime type information, branch frequencies, and call-site shapes let the JIT specialize code in ways an AOT compiler cannot.
4. Recognize the cost of **speculation and deoptimization** — why the JIT can assume something is true, and what happens when that assumption breaks.
5. Explain **inline caching** and **hidden classes** — how V8 accelerates polymorphic property access.
6. Know when JIT wins and when AOT is the better choice.

---

## 59.1 The Problem JIT Solves

Recall from Chapter 4:

- **Pure interpretation** is simple and portable, but slow. Every operation is a switch case, a type check, a method-table lookup.
- **Ahead-of-time compilation** to machine code is fast at runtime, but inflexible. The compiler can't adapt to runtime data — it must generate code that works *for all possible inputs*.

JIT sits between. The idea:

1. Start with interpretation (slow, but program starts instantly).
2. Identify the methods or loops that actually run hot (consume 90% of the time).
3. Compile just those to machine code, using runtime information the interpreter gathered.
4. Execute the compiled code. It's specialized for the types, branches, and call targets the program actually encounters.

This requires the runtime to do something AOT compilers cannot: **look at the data the program is working on, make a bet about what will come next, and optimize based on that bet.**

The catch: bets can fail. If code was specialized for integers, and suddenly a float arrives, the JIT must **deoptimize** — abandon the compiled code and fall back to the interpreter — then recompile if the pattern changes.

---

## 59.2 The Tiered Strategy

No production JIT compiles only once. Instead, code is compiled multiple times, each time more aggressively.

### Tier 0: The Interpreter

When the program starts, all code runs in the bytecode interpreter. The interpreter is slow, but it:

- Starts instantly (no compilation latency).
- Collects profiling data: what types flow through each operation, which branches are taken, which methods call which.

The interpreter is also a safety net: if something goes wrong in the compiled code, the JIT can deoptimize back to it.

### Tier 1: Baseline JIT

Once a method is called a few hundred times, the JIT compiles it. This compilation is *fast* — it takes milliseconds, not seconds. The baseline JIT:

- Does minimal optimization.
- Emits straightforward code without speculative assumptions.
- Includes safety guards: type checks, branch guards, polymorphism guards.

The result is better than interpretation, but not as good as it could be.

### Tier 2+: Optimizing JIT

If a method runs *really* hot (millions of invocations), the optimizing JIT recompiles it. This compilation is *expensive* — it might take hundreds of milliseconds. But the resulting code is highly optimized:

- Inlines aggressively based on observed call targets.
- Specializes on observed types.
- Removes redundant checks.
- Vectorizes loops.
- Reorders instructions for the CPU pipeline.

### Real Examples

**HotSpot JVM** (Java/Kotlin):
- Tier 0: Bytecode interpreter.
- Tier 1 (C1): Fast, simple JIT. Compilation in ~1ms. No speculation on types.
- Tier 2 (C2): Aggressive JIT. Speculates heavily. Inlines recursively. Takes 10–100ms to compile.

**V8** (JavaScript):
- Ignition: Bytecode interpreter.
- Sparkplug: Fast JIT. Baseline compilation.
- Maglev: Medium optimization.
- TurboFan: Aggressive optimization. Speculates on types, shapes, and branch predictions.

**.NET CoreCLR** (C#, F#):
- Tier 0: Interpreter (or fast startup compiled JIT).
- Tier 1: Simple JIT, compiled early.
- Tier 2: Optimizing JIT, runs later with profiling data.

The strategy is defensive: "most code is never hot, so don't waste time optimizing it. For the code that is hot, we'll invest in increasingly aggressive optimization."

---

## 59.3 Profile-Guided Optimization at Runtime

The interpreter hands the JIT a gift: profiling data. **What types have flowed through each operation? Which branches are taken? Which call targets have been seen?**

The JIT uses this to specialize code in ways an AOT compiler cannot.

### Type Specialization

Consider a JavaScript function that adds two numbers:

```javascript
function add(a, b) { return a + b; }
```

An AOT compiler sees `a` and `b` are unknown types. The compiled code must handle them generically: type checks, method tables, boxing/unboxing. It's slow.

The interpreter watches calls to `add`. Over 100,000 calls, it sees:

```
add(5, 3)     // integer
add(7, 2)     // integer
add(100, 50)  // integer
```

The JIT specializes the compiled code for integers. The compiled version does:

```
mov eax, [rdi + offset_to_a]     # unbox a
mov ecx, [rsi + offset_to_b]     # unbox b
add eax, ecx
mov rax, [heap + offset_for_result]
mov [rax], eax                    # reboxed integer
ret
```

No type checks. No method-table lookup. Just arithmetic. For integer-heavy code, this can be 10–50× faster.

### Call-Site Inlining

A polymorphic call site is one where multiple different methods can run:

```javascript
obj.method()  // could be A.method or B.method depending on what obj is
```

An AOT compiler typically does not inline polymorphic calls — it doesn't know which method will run. The compiled code does a method-table lookup (vtable in C++ terms), then calls.

The interpreter watches the call site. Over 10,000 executions, it sees:

```
obj.method()  -> calls A.method (9,900 times)
obj.method()  -> calls B.method (100 times)
```

The JIT inlines A.method. The code becomes:

```
// check: is obj an instance of A?
test [obj + offset_type], type_A
jne bail_to_slow_path
// inline A.method's body here
mov eax, [obj + field_x]
add eax, 1
mov [obj + field_x], eax
// ... rest of A.method
jmp continue
bail_to_slow_path:
call method_lookup(obj)
```

If the check passes (99% of the time), the method is inlined — *massive* speedup. If the check fails (1% of the time), the code bails out to a slow path that does the full method lookup.

This is called **polymorphic inline caching** (PIC) and it's one of the JIT's most powerful tools.

### Branch Prediction

The interpreter records which branches are taken:

```javascript
if (x > 10) {
    // hot path: taken 99% of the time
} else {
    // cold path: taken 1% of the time
}
```

The JIT can:
- Lay out the hot branch first in the code (better CPU branch prediction).
- Even speculate: assume the branch will always be taken, and only generate code for the hot case. If the branch is ever taken the other way, deoptimize and recompile.

---

## 59.4 Speculation and Deoptimization

JIT code is built on assumptions. If assumptions hold, code is fast. If they break, the JIT must deoptimize.

### How Speculation Works

The JIT compiles code assuming "x will always be an integer." The compiled code does not type-check x. But before the assumption is baked in, the JIT adds a **check instruction**:

```asm
mov rax, [rsi + offset_x]           # load x
test rax, tag_integer               # is it tagged as integer?
jnz deopt_path                      # if not, bail to deoptimization
; ... use x as integer (no further checks)
```

Most of the time, the check passes, and the code runs fast. But if `x` stops being an integer, the `jnz` is taken, and execution jumps to **deoptimization**.

### What Deoptimization Does

Deoptimization is expensive. The JIT:

1. Abandons the compiled code.
2. Reconstructs the interpreter's frame from the compiled frame's state (unpacking inlined calls, extracting intermediate values).
3. Jumps back to the interpreter.
4. The interpreter resumes, now with code that handles the unexpected type.

This costs microseconds to milliseconds — far slower than a single type check would have been. But it's rare, so on average the code is still faster.

If the new type becomes hot, the JIT will recompile later with code that handles both the original type and the new type. This is called **polymorphic specialization** — the code becomes less optimized but more correct.

### When Deoptimization Is Cheap vs. Expensive

Deoptimization is cheap if it happens rarely. Cheap: "a call site is 99.9% of one type, and 0.1% of another."

Deoptimization is expensive if checks fail often. Expensive: "code is specialized for integers, but the inputs are random types." In this case, the JIT will deoptimize thousands of times per second, and the program is slower than pure interpretation.

A good JIT watches deoptimization rates and adjusts: if a code path deoptimizes too often, it's recompiled without the speculation, or with broader speculation that covers more types.

---

## 59.5 Inline Caching and Hidden Classes (V8)

V8's approach to speeding up object property access is worth understanding in detail, because it's a concrete example of how JIT turns slow operations into fast ones.

### The Problem: Property Access Is Polymorphic

In JavaScript:

```javascript
obj.field
```

This could mean:
- `obj` is an object with property `field` at offset 16.
- `obj` is a different kind of object with property `field` at offset 32.
- `obj` has no property `field`, so fall back to prototype lookup.

Without optimization, every property access does:

1. Check the object's type/shape.
2. Look up the property name in the shape's dictionary.
3. Extract the offset.
4. Load from that offset.

This is expensive — several instructions, possible cache misses.

### Inline Caches

V8's solution: **inline caching**. The first time `obj.field` is executed:

1. The interpreter or baseline JIT sees `obj` has a particular shape (let's call it `Shape_A`).
2. It records: "when the object has shape `Shape_A`, the field is at offset 16."
3. The code is compiled with an inline cache entry:

```asm
mov rax, [rsi]                   # load obj
cmp [rax + offset_shape], Shape_A  # check shape
jne slow_path
mov rax, [rsi + 16]              # load field from offset 16 (cached)
jmp done
slow_path:
call lookup_property(obj, "field")
done:
```

If `obj` always has the same shape, the check always succeeds, and property access is one load after a cached comparison. Much faster.

### Polymorphic Inline Caches

If two different shapes are seen:

```asm
mov rax, [rsi]                      # load obj
cmp [rax + offset_shape], Shape_A
je shape_a
cmp [rax + offset_shape], Shape_B
je shape_b
jmp slow_path
shape_a:
mov rax, [rsi + 16]
jmp done
shape_b:
mov rax, [rsi + 24]
jmp done
slow_path:
call lookup_property(obj, "field")
done:
```

Now two shapes are inlined. If a third shape appears, the slow path is taken, and V8 decides: recompile with all three shapes, or abandon specialization entirely.

### Hidden Classes

The key insight: **if objects of the same "type" are created in the same order, they will have the same shape, and V8 can track that shape.** V8 calls these shapes "hidden classes." They're not user-visible, but they're the mechanism by which property access becomes nearly as fast as struct field access.

Consider:

```javascript
function Point(x, y) {
    this.x = x;
    this.y = y;
}
let p1 = new Point(3, 4);
let p2 = new Point(5, 6);
```

Both `p1` and `p2` will have the same hidden class (same properties, same offsets), so inline caches will hit immediately.

If you write:

```javascript
let p3 = new Point(7, 8);
p3.z = 9;  // add a new property — p3 now has a different shape
```

`p3` has a different hidden class. Future property accesses will either hit a polymorphic cache (if the shape is common) or fall back to the slow path.

---

## 59.6 Garbage Collection Interaction

JIT-compiled code must cooperate with the garbage collector.

### GC Barriers

When compiled code writes a pointer to the heap:

```cpp
obj->child = other_obj;
```

The GC needs to know: "obj now points to other_obj." If the GC is generational (younger objects collected more often), it needs to know which old objects point to young objects.

So the JIT emits a **GC barrier** — a check that records the write:

```asm
mov [rdi + offset_child], rsi      # obj->child = other_obj
cmp [rsi], young_generation_mask   # is other_obj in young gen?
jz barrier_not_needed
call record_barrier_slow            # record the pointer
barrier_not_needed:
```

This barrier is part of every reference write in JIT code. It adds some cost, but the GC's faster collection of young objects makes up for it.

### Stack Maps

The GC needs to know which slots on the stack hold pointers. In AOT code, this is encoded in static metadata (DWARF tables). In JIT code, the JIT must emit **stack maps** — metadata that tells the GC "at this instruction, these stack slots contain pointers."

JIT compilers build these maps as they generate code:

```
[instruction_offset, stack_map_id]
[stack_map_id] -> [slot_0 is_pointer, slot_1 is_pointer, ...]
```

When the GC pauses all threads, it reads the instruction pointer of each thread, looks up the stack map, and scans the pointers found in that map.

This is harder than it sounds: if the JIT inlines methods, it must track which inlined frame corresponds to which instruction, so the GC can properly identify pointers. If speculative assumptions cause deoptimization, the stack map must account for the reconstructed interpreter frame.

---

## 59.7 When JIT Wins, When AOT Wins

### JIT Excels At:

1. **Long-running processes.** A server running for days has time to warm up. By the time 99% of the requests arrive, the JIT has compiled the hot paths. Peak performance matters more than startup time.

2. **Dynamic types and polymorphism.** If the program's behavior changes over time (different input types, different call targets), the JIT can recompile. An AOT compiler would have to generate code for all possibilities upfront.

3. **Adaptive optimization.** If certain paths are always cold, the JIT doesn't waste code size or compile time on them. The interpreter handles them just fine.

4. **Profile-driven optimization without programmer effort.** The programmer doesn't need to annotate hot paths or provide profiles. The JIT learns automatically.

### AOT Excels At:

1. **Startup time.** A command-line tool that runs for 100ms cannot wait for JIT compilation. The code must be ready to execute immediately.

2. **Predictability.** AOT code's performance is consistent — no deoptimization surprises, no warmup, no GC pauses. This matters in real-time systems, financial trading, robotics.

3. **Embedded and resource-constrained environments.** A JIT's memory overhead (interpreter, compiler, profiling data, compiled code cache) is a luxury many embedded systems cannot afford.

4. **Single-artifact deployment.** Ship one binary. It works everywhere. No runtime needed. No version incompatibility between JIT and runtime.

5. **Security and verification.** In some security models, you want the final code to be deterministic and verifiable. A JIT introduces non-determinism.

### Hybrid: JIT + AOT Caching

Modern systems sometimes use both:

- **App startup:** Compiled code is cached (or AOT precompiled and shipped with the app).
- **On first run:** The JIT compiles and caches code profiles.
- **On subsequent runs:** The cached profiles are used to warm the JIT faster.

Java's AppCDS (Application Class Data Sharing), .NET's ReadyToRun, and V8's code caching all follow this pattern.

---

## 59.8 Worked Example: Java Method Gets Warm

Let's compile and run a Java program with JIT profiling visible.

```java
public class Fibonacci {
    static long fib(int n) {
        if (n <= 1) return n;
        return fib(n - 1) + fib(n - 2);
    }
    
    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            long x = fib(30);  // Deliberately compute fib(30) 100 times
        }
    }
}
```

Compile:

```bash
javac Fibonacci.java
```

Run with JIT compilation visible:

```bash
java -XX:+PrintCompilation Fibonacci
```

You'll see output like:

```
    128   1       java.lang.String::hashCode (56 bytes)
    129   2       java.lang.Object::<init> (1 bytes)
    129   3       java.lang.String::length (6 bytes)
   ...
    142   4       Fibonacci::fib (18 bytes)
   142   5    %  Fibonacci::main (27 bytes)
```

Column meanings:

- First number: timestamp (milliseconds since JVM started).
- `1, 2, ...`: tier level (1 = C1 fast JIT, 2 = C2 aggressive JIT, % = OSR — on-stack replacement, meaning a loop was recompiled while running).
- Method name and bytecode size.

Notice: `Fibonacci::fib` gets compiled (tier 4) early because it's called 100 times. Later, if it gets *really* hot, the C2 JIT would recompile it more aggressively (you'd see tier 2 or higher).

For a deeper look:

```bash
java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions \
     -XX:PrintAssemblyOptions=intel -XX:+PrintInlining Fibonacci
```

This prints inlining decisions and assembly code (requires `hsdis` library on most systems), showing you *exactly* what the JIT decided about `fib`.

---

## 59.9 Tradeoffs

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| **Pure Interpreter** | Instant startup; simple; portable | 10–100× slower than native; high memory | Embedded, small scripts, quick testing |
| **Baseline-Only JIT** | Simple JIT; reasonable peak perf; fast compilation | Limited optimization; still slower than aggressive JIT | Long-running apps where compilation speed matters |
| **Tiered JIT** | Best peak performance; adapts to data; good startup tradeoff | Complex implementation; harder to debug; warmup latency | General-purpose interpreters (V8, JVM) |
| **JIT + AOT Cache** | Fast startup + peak performance on later runs | Requires offline profiling or cache management | Shipping applications with bundled VMs |
| **Ahead-of-Time (AOT)** | Deterministic startup; no warmup; small runtime | Less optimization for polymorphism; binaries per platform | Systems software, real-time, embedded |

---

## 59.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "JIT is always faster than interpretation." | No. JIT is faster once warmed up, but only for code that actually runs hot. Cold code and short programs are faster as pure interpretation. |
| "JIT is always faster than AOT." | No. AOT can match or exceed JIT for code with predictable structure and known types. JIT's advantage is *adapting to runtime data*. |
| "Deoptimization is rare." | Depends. In well-behaved code with consistent types, yes. In code with many polymorphic call sites or changing assumptions, deoptimization can be a serious cost. |
| "Inline caching is a Java/JavaScript thing." | No. Many languages use it: PyPy, Lua, Ruby, even compiled languages via virtual method optimization. |
| "JIT can always infer types." | No. If types are truly dynamic and unpredictable (random object types every call), the JIT's type specialization gains nothing. It's the *pattern* of types that JIT exploits. |
| "Garbage collection is orthogonal to JIT." | False. JIT code must emit GC barriers and stack maps, adding overhead and complexity. Poor GC pauses can destroy peak performance. |

---

## 59.11 Exercises

1. **Observe JIT warmup in your language.** Pick a language with a JIT (Java, JavaScript/Node, .NET, PyPy). Write a loop that calls the same method 10 million times. Measure the time per iteration at the start, middle, and end. Note how the first million iterations are slower, then performance plateaus. Explain what you're observing in terms of tier 0, 1, and 2.

2. **Speculate and fail.** In JavaScript (V8), write a function that is specialized for integers but occasionally receives a string:
   ```javascript
   function compute(x, y) { return x + y; }
   for (let i = 0; i < 1000000; i++) {
       compute(i, i+1);
   }
   compute("oops", "wrong");  // type mismatch
   for (let i = 0; i < 1000000; i++) {
       compute(i, i+1);  // back to numbers
   }
   ```
   Measure the time cost of the type mismatch and recompilation. Then modify the code to never trigger the mismatch and measure again. What's the cost of a failed assumption?

3. **Inline caching in action.** In JavaScript, create two objects with different shapes and access the same property:
   ```javascript
   let obj1 = {x: 1, y: 2};
   let obj2 = {a: 1, b: 2, c: 3};  // different shape
   for (let i = 0; i < 1000000; i++) {
       obj1.x;
       obj2.a;
   }
   ```
   Compare the time to:
   ```javascript
   let objs = [obj1, obj2];
   for (let i = 0; i < 1000000; i++) {
       objs[i % 2].x;  // or .a
   }
   ```
   Explain why the first is faster (monomorphic property access) and the second slower (polymorphic).

4. **Deoptimization cost.** In Java, write a method that is initially monomorphic but becomes polymorphic:
   ```java
   class Base { int compute() { return 1; } }
   class Derived extends Base { int compute() { return 2; } }
   
   static void loop(Base obj) {
       for (int i = 0; i < 10_000_000; i++) {
           obj.compute();
       }
   }
   
   public static void main(String[] args) {
       loop(new Base());     // Base only
       loop(new Derived());  // Now Derived — deoptimization triggered
       loop(new Base());     // Back to Base
   }
   ```
   Run with `-XX:+PrintCompilation` to see deoptimization events. Then measure the time cost of the polymorphic change.

5. **AOT vs. JIT startup.** Implement a simple CLI tool that:
   - Reads a file.
   - Computes a checksum.
   - Prints the result.
   
   Implement it in three languages: C++ (compiled), Java (JIT), and Python. Measure the end-to-end time on a 1MB file. Note how C++ is fastest, Java has startup overhead, and Python is slowest. Then time it on a 100MB file and see how Java's and Python's per-byte throughput eventually dominates once the JIT warms up.

---

## 59.12 Summary

JIT compilation is the runtime betting on what code will be hot, compiling that code with specializations based on observed runtime data, and falling back to interpretation if assumptions break. The tiered strategy — interpreter, then fast JIT, then aggressive JIT — balances startup time and peak performance. Profile-guided optimization, inline caching, and speculative assumptions let JIT code match or exceed AOT-compiled code in peak performance on dynamic workloads. The cost is complexity: the runtime must maintain an interpreter, compiler, profiler, and deoptimization machinery. But for long-running applications where adaptability and peak performance matter, the cost is worth it.

---

> **[← Previous: Compiled vs Interpreted](01-compiled-vs-interpreted.md)** · **[↑ Part 6](README.md)** · **[Next: How Python Executes Code →](03-how-python-executes-code.md)**
