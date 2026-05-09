# Chapter 63 — How the JVM Works

The Java Virtual Machine is a 30-year-old bytecode VM that has had more engineering hours invested in it than perhaps any other piece of software outside operating systems. What started as a way to write code once and run it anywhere became a laboratory for dynamic language optimization. Today's HotSpot JVM can produce machine code that rivals or exceeds hand-written C++, thanks to tiered compilation, aggressive inlining, and escape analysis. Understanding what's inside the JVM changes how you reason about Java, Kotlin, Scala, Clojure, and every other language that runs on it.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Understand the class file format and bytecode representation — why the JVM was designed as a stack-based VM.
2. Explain the class loading hierarchy — bootstrap, platform, and application loaders — and why `Class.forName()` can trigger static initialization.
3. Describe the execution engine's three-tier compilation strategy: interpreter → baseline JIT (C1) → optimizing JIT (C2).
4. Map the JVM's runtime memory layout: heap (young and old generation), metaspace, per-thread stack, program counter, and native stacks.
5. Compare the major garbage collectors — Serial, Parallel, G1, ZGC, Shenandoah — and understand their pause-time vs. throughput tradeoffs.
6. Recognize the optimizations the JVM performs transparently: inlining, escape analysis, lock elision, null-check elimination.
7. Diagnose JVM startup costs and evaluate AppCDS, native-image, and CRaC/CRIU as solutions.

---

## 63.1 The Class File and Bytecode

### The `.class` File Structure

A compiled Java class is a binary file with a rigid structure. This structure was designed in the 1990s to be:

1. **Extensible**: new attributes can be added without breaking old loaders.
2. **Self-describing**: a classloader can inspect a `.class` file and understand the class's structure without running code.
3. **Stateless**: a single file contains everything needed to instantiate a class.

The structure is:

```
┌─────────────────────────────────────────┐
│ Magic number (0xCAFEBABE)               │ 4 bytes
├─────────────────────────────────────────┤
│ Minor version                           │ 2 bytes
├─────────────────────────────────────────┤
│ Major version                           │ 2 bytes
├─────────────────────────────────────────┤
│ Constant pool count                     │ 2 bytes
├─────────────────────────────────────────┤
│ Constant pool entries                   │ variable
│  (strings, class refs, field types, ...) │
├─────────────────────────────────────────┤
│ Access flags (public, final, abstract)  │ 2 bytes
├─────────────────────────────────────────┤
│ This class index (into constant pool)   │ 2 bytes
├─────────────────────────────────────────┤
│ Super class index                       │ 2 bytes
├─────────────────────────────────────────┤
│ Interfaces (count + indices)            │ variable
├─────────────────────────────────────────┤
│ Fields (name, type, attributes)         │ variable
├─────────────────────────────────────────┤
│ Methods (name, signature, attributes)   │ variable
│   ├─ Code attribute (bytecode)          │
│   ├─ Exceptions attribute               │
│   ├─ Line number table                  │
│   └─ Local variable table               │
├─────────────────────────────────────────┤
│ Class attributes                        │ variable
└─────────────────────────────────────────┘
```

The **constant pool** is the payoff: every reference — to a string, another class, a method, a field — is stored as an index into a table. This makes class metadata compact and allows the classloader to verify references without executing code.

### Bytecode: Stack-Based Operations

JVM bytecode is **stack-based**, not register-based. Every operation works by popping operands from a stack, computing, and pushing the result back. Here's a simple method:

```java
public int add(int a, int b) {
    return a + b;
}
```

Its bytecode (visible via `javap -c -p`):

```
public int add(int, int);
  Code:
     0: iload_1          // push a onto stack
     1: iload_2          // push b onto stack
     3: iadd             // pop b, pop a, push a+b
     4: ireturn          // pop result, return
```

This is deliberate. Stack-based bytecode is:

- **Compact**: no register allocation needed, bytecode is smaller.
- **Portable**: independent of CPU architecture.
- **Easy to JIT compile**: the stack discipline means the JIT knows what types are where.

The downside: interpreter dispatch is expensive (each bytecode is a switch case). Modern JVMs use **template-based interpretation** to mitigate this: at JVM startup, the interpreter generates a table of native code fragments, one per bytecode. When the interpreter runs `iload_1`, it jumps to that fragment, which pushes a value onto the stack and continues. Still slower than compiled code, but much faster than pure bytecode interpretation.

### Constant Pool Example

For a method call like `result = obj.method()`, the bytecode looks like:

```
invokevirtual #5  // call method at constant pool index 5
```

Constant pool entry 5 might be:

```
CONSTANT_Methodref {
    class_index: 7,      // points to the class that owns the method
    name_and_type_index: 12  // points to the method name and signature
}
```

At class loading time, the JVM validates that method 5 exists. At runtime, the compiled code does a method lookup (usually cached in an **inline cache**, see Chapter 59).

---

## 63.2 Class Loading

Classes are loaded lazily. The sequence is:

### 1. Bootstrap Loader

Loads core Java classes: `java.lang.Object`, `java.lang.String`, `java.util.*`, etc. from the JDK standard library. The bootstrap loader is part of the JVM itself (written in C++).

### 2. Platform Loader (Extension Loader)

Loads extension classes from `$JAVA_HOME/lib/ext` and modules from the JDK.

### 3. Application Loader

Loads user-defined classes from the classpath.

### Lazy Loading and Class.forName()

When a method references a class that hasn't been loaded yet, the JVM doesn't load it immediately. Loading happens when:

1. The class is first **instantiated** (`new MyClass()`).
2. A **static member** is accessed (`MyClass.staticField`).
3. A **static method** is invoked (`MyClass.staticMethod()`).
4. A **method reference** requires the class (`MyClass::method`).
5. **Explicit loading** via `Class.forName("com.example.MyClass")`.

Crucially, `Class.forName()` **triggers static initializers**:

```java
public class Expensive {
    static {
        System.out.println("Static init runs!");
        // ... load configuration, connect to database, etc.
    }
}

// This line prints "Static init runs!"
Class<?> clazz = Class.forName("Expensive");
```

Many frameworks use this to load plugins or register handlers. The cost can be significant.

### Verification

When a class loads, the JVM verifies it:

1. **Bytecode structuring**: is the bytecode well-formed?
2. **Type checking**: do all operations use types correctly (e.g., no popping an integer when a long is expected)?
3. **Control flow**: does every code path that doesn't throw an exception return a value?

This verification is done once, at load time. It buys correctness: if verification passes, the JVM knows the code won't have certain kinds of memory safety violations (no buffer overruns, no wild pointers).

---

## 63.3 The Execution Engine: Interpreter and Tiered JIT

When the JVM loads a class, methods are initially set to be executed by the **interpreter**. As methods warm up, the JIT compiler takes over.

### Tier 0: Interpreter (Execution Template)

The interpreter reads bytecode in a loop:

```cpp
// Pseudocode of the JVM interpreter dispatch loop
while (true) {
    bytecode = fetch_bytecode();
    switch (bytecode) {
        case ILOAD: 
            // push local variable (int) onto stack
            stack.push(locals[operand]);
            break;
        case IADD:
            // pop two ints, push sum
            int b = stack.pop();
            int a = stack.pop();
            stack.push(a + b);
            break;
        // ... 200+ more cases
    }
}
```

Modern JVMs use **template-based interpretation**: at startup, the JVM generates a table of machine code fragments, one per bytecode. Instead of a switch case, it jumps directly to the corresponding fragment. This is much faster than byte-at-a-time dispatch.

The interpreter also collects profiling data: what types flow through each operation, which branches are taken, which methods call which.

### Tier 1: Baseline JIT Compilation (C1 in HotSpot)

Once a method is called roughly 1,000–10,000 times (configurable), the JIT compiles it. The baseline JIT:

1. Converts bytecode to an intermediate representation (IR).
2. Performs simple optimizations (dead code elimination, constant propagation).
3. Generates machine code with **guard instructions**: checks that ensure assumptions hold.

Example: a method that's been called 1,000 times, always with integer arguments:

```java
public int compute(Object x) {
    if (x instanceof Integer) {
        return ((Integer) x).intValue() + 1;
    }
    return -1;
}
```

Baseline JIT might generate:

```asm
mov rax, [rdi + offset_of_x]       # load x
test rax, tag_integer_mask          # is it an integer?
jnz bailout                          # if not, bail to interpreter
mov eax, [rax + int_value_offset]   # extract integer value
inc eax
ret

bailout:
    jmp interpreter_or_slow_path
```

If the assumption holds (which it will for the next million calls), this executes very fast. If an unexpected type arrives, the code bails out to the interpreter, which handles it correctly.

Baseline compilation is **fast**: typically 1–10 milliseconds. It's worth the cost because even modest optimization beats interpretation for hot code.

### Tier 2: Optimizing JIT Compilation (C2 in HotSpot)

When a method is *really* hot (millions of invocations), it's recompiled by the **optimizing JIT**. This compilation is expensive (10–100ms) but produces much better code.

The optimizing JIT:

1. Uses the profiling data from the interpreter and baseline JIT to specialize code.
2. **Inlines aggressively**: if `obj.method()` has always called `A.method`, inline the entire method's body.
3. **Escape analysis**: if an object never escapes the method, allocate it on the stack instead of the heap.
4. **Lock elision**: if a lock is only ever held by one thread, remove it.
5. **Null-check elimination**: if a value is guaranteed non-null, remove the null check.
6. **Vectorization**: auto-SIMD for loops (available in newer JVMs).

Example: a hot loop that sums an array:

```java
public long sum(int[] arr) {
    long result = 0;
    for (int i = 0; i < arr.length; i++) {
        result += arr[i];
    }
    return result;
}
```

After optimization, this might be compiled to:

```asm
# Assume arr is not null (inline check, or use profiling data)
mov rcx, [rdi + array_length_offset]  # load array length
xor rax, rax                           # result = 0
xor r10, r10                           # i = 0

loop:
    cmp r10, rcx
    jge done
    
    # Bounds-check elimination: we know i < arr.length
    mov r8d, [rdi + r10*4 + array_data_offset]  # arr[i]
    add rax, r8
    inc r10
    jmp loop

done:
    ret
```

Notice what's *missing*:

- No null check on `arr` (the JIT knows it's safe).
- No bounds check in the loop (the JIT proved `i < arr.length` always).
- No type checks (the JIT knows it's an `int` array).

This code will run at near-C speed.

### Deoptimization

If code was specialized for integers, and suddenly a `Long` arrives, what happens?

The baseline JIT code has guard instructions. The guard fails, and the code **deopts**: jumps to a deoptimization routine that:

1. Reconstructs the interpreter's state (locals, stack, program counter).
2. Jumps back to the interpreter to continue execution.
3. If the pattern changes (e.g., Long is now common), the method is recompiled for both types.

Deoptimization is expensive (microseconds), so modern JITs bias toward monomorphic specialization: if a call site has seen 99 different types, don't specialize. But for 1–2 types, specialize aggressively.

---

## 63.4 Memory Areas: Heap, Metaspace, Stack

The JVM divides memory into several regions:

### Heap

The heap is where objects live. For a generational GC (the default), the heap is divided:

```
┌──────────────────────────────┐
│ Young generation (Nursery)   │  ~300 MB (typically 10% of heap)
│  [Eden] [Survivor0] [Survivor1]
├──────────────────────────────┤
│ Old generation (Tenured)     │  ~2.7 GB (90% of heap)
└──────────────────────────────┘
```

Objects are allocated in **Eden** (fast bump-pointer allocation). When Eden fills, a young generation GC runs, surviving objects are promoted to **old generation**, and Eden is reclaimed.

Generational GC is powerful because:

- **Young gen collections are very fast**: they only scan the young generation (~100 MB), not the whole heap (3 GB). A young GC pause is 5–50 milliseconds.
- **Most objects are short-lived**: the weak generational hypothesis holds for most workloads (web servers, batch processing). Long-lived objects are collected infrequently.

### Metaspace

Class metadata — method bytecode, field descriptors, method signatures, constant pools — lives in **metaspace** (separate from the heap in Java 8+). Metaspace can grow or shrink dynamically. If it grows unbounded (e.g., a custom classloader keeps loading new classes), the JVM can run out of metaspace and crash.

### Per-Thread Stack

Every thread has its own stack. The stack holds:

- **Local variables**: method parameters and local variables.
- **Operand stack**: for the stack-based bytecode operations.
- **Frame metadata**: method pointer, return address, exception handler table.

Stack space is limited (typically 1 MB per thread). Deep recursion or a thread explosion can exhaust it.

### Program Counter (PC) Register

Each thread has a PC register that points to the currently executing bytecode instruction. If the method is compiled to native code, the PC points to native instructions instead.

### Native Method Stack

When code calls a native method (via JNI, Java Native Interface), the native code runs on the native stack (not the Java stack). Memory safety violations in native code are not caught by the JVM's verification.

### Diagram: Memory Layout

```
     Thread A Stack        Heap              Metaspace
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │ Local vars  │  │ Young gen   │  │ Method code │
   │ (frame A)   │  │  [Object1]  │  │ Class info  │
   ├─────────────┤  │  [Object2]  │  ├─────────────┤
   │ Local vars  │  ├─────────────┤  │ Bytecode    │
   │ (frame B)   │  │ Old gen     │  │ Constants   │
   └─────────────┘  │  [Object3]  │  └─────────────┘
                    │  [Object4]  │
                    └─────────────┘
```

---

## 63.5 Garbage Collectors: Serial, Parallel, G1, ZGC, Shenandoah

The JVM ships with multiple garbage collectors, each with different pause time vs. throughput tradeoffs. Since Java 9, **G1GC** is the default.

### Serial GC (`-XX:+UseSerialGC`)

The simplest collector. Single thread, stop-the-world collection. Young gen and old gen are collected separately.

**Pause times**: 50–500 ms (for a 1 GB heap).  
**Throughput**: 98–99% (very little CPU spent on GC).  
**When to use**: single-threaded applications, small heaps, embedded systems.  
**Cost**: frequent pauses are unacceptable for latency-sensitive services.

### Parallel GC (`-XX:+UseParallelGC`)

Multi-threaded young gen collection. Scales well on multi-core machines. Young gen collections run in parallel; old gen is still stop-the-world.

**Pause times**: 20–100 ms (young), 500 ms – 1 sec (old).  
**Throughput**: 98–99% (parallel threads reduce pause time but don't eliminate it).  
**When to use**: batch jobs, data processing pipelines, workloads where occasional pauses are tolerable.

### G1GC: Garbage First (`-XX:+UseG1GC`)

Region-based, generational, concurrent marking. The heap is divided into regions (typically 1–32 MB). Young and old gen are interspersed. Concurrent marking runs in parallel with the mutator (application code).

**Pause times**: 10–100 ms (target is configurable, e.g., `-XX:MaxGCPauseMillis=200`).  
**Throughput**: 95–98%.  
**When to use**: large heaps (4 GB+), latency-sensitive applications, the default choice for production JVMs.  
**Mechanism** (from Chapter 11): region-based collection, generational with concurrent tri-color marking. Write barriers track cross-region pointers.

### ZGC: Z Garbage Collector (`-XX:+UseZGC`)

Ultra-concurrent, using **colored pointers**. The unused high-order bits of a pointer encode GC metadata (marked, remapped, etc.). Load barriers on every reference access let the GC run almost entirely concurrently with the mutator.

**Pause times**: <1 millisecond (often <100 microseconds).  
**Throughput**: 95–97% (load barriers add overhead).  
**When to use**: ultra-low-latency applications (fintech, online gaming), large heaps (100 GB+).  
**Tradeoff**: load barriers (on every reference read) add ~5–10% CPU overhead, but pauses are near-zero.

### Shenandoah (`-XX:+UseShenandoahGC`)

Similar to ZGC, using **load barriers** instead of colored pointers. Also ultra-concurrent.

**Pause times**: <10 milliseconds.  
**Throughput**: 95–97%.  
**When to use**: low-latency applications that can't tolerate pauses > 10 ms.  
**Difference from ZGC**: Shenandoah is simpler and older; ZGC is newer and tuned for extreme latency.

### Comparison Table

| Collector | Pause (Young) | Pause (Old) | Throughput | Best For |
|-----------|---------------|-------------|-----------|----------|
| Serial | 10–50 ms | 100–1000 ms | 99% | Single-threaded, small heap |
| Parallel | 10–50 ms | 500–2000 ms | 98–99% | Batch jobs, high throughput |
| G1 | 5–50 ms | 50–200 ms* | 95–98% | Large heaps, balanced latency |
| ZGC | <1 ms | <1 ms | 95–97% | Ultra-low latency, 100+ GB heaps |
| Shenandoah | <10 ms | <10 ms | 95–97% | Low-latency, responsive apps |

*G1 old gen collection is concurrent; pause is brief (remark phase only).

---

## 63.6 Optimizations You Get For Free

The JIT compiler performs several optimizations automatically, without programmer annotation:

### 1. Inlining

If a method is small and called frequently from a narrow set of call sites, the JIT inlines it (inserts its code into the caller). This eliminates method call overhead and enables further optimization.

```java
public class Point {
    public int getX() { return x; }
}

// Called millions of times:
for (Point p : points) {
    sum += p.getX();
}

// JIT inlines getX():
for (Point p : points) {
    sum += x;  // direct field access, no method call
}
```

### 2. Escape Analysis

If an object is allocated locally and never escapes the method (is never returned, never stored in a field, never passed to another thread), the JIT can allocate it on the stack instead of the heap.

```java
public int process(int[] data) {
    List<Integer> list = new ArrayList<>();  // allocated on stack?
    for (int x : data) {
        list.add(x);
    }
    return list.stream().mapToInt(i -> i).sum();
}

// If 'list' is never seen outside this method, allocate on stack.
// At method return, stack is popped; no heap allocation, no GC.
```

Escape analysis is powerful because it eliminates GC pressure and improves cache locality.

### 3. Lock Elision

If a lock is acquired and released within a single thread (never contended), the JIT removes the lock:

```java
public void append(StringBuilder sb) {
    synchronized (sb) {  // lock elided if sb is thread-local
        sb.append("hello");
    }
}
```

### 4. Null-Check Elision

If a value is guaranteed non-null (e.g., it's `this`, or it's checked immediately before use), the JIT removes the null check.

```java
public int process(Object obj) {
    if (obj == null) throw new NullPointerException();
    return obj.hashCode();  // null check already done
}

// JIT sees the check and removes redundant checks in called methods.
```

### 5. Bounds-Check Elimination

In tight loops, the JIT can prove that array accesses are always in bounds and remove the checks:

```java
for (int i = 0; i < arr.length; i++) {
    sum += arr[i];  // JIT proves i < arr.length always
}

// Compiled code has no bounds checks in the loop.
```

### 6. Vectorization (Auto-SIMD)

Recent JVM versions can auto-vectorize loops, using SIMD instructions:

```java
for (int i = 0; i < data.length; i++) {
    result[i] = data[i] * 2;
}

// Compiled to process 4–8 elements per iteration using SIMD.
```

These optimizations are **why the JVM can be fast**: the JIT doesn't just compile; it specializes and optimizes based on runtime information.

---

## 63.7 What the JVM Cannot Optimize

There are patterns the JVM struggles with:

### Reflection-Heavy Code

Every `invoke` call or field access via reflection requires a runtime lookup:

```java
Method m = clazz.getMethod("compute", int.class);
for (int i = 0; i < 1_000_000; i++) {
    result += (Integer) m.invoke(obj, i);
}
```

The JIT cannot inline the method because the target is unknown at compile time. Every iteration does a method lookup, unboxing, and boxing. This is orders of magnitude slower than direct calls.

### Boxing and Unboxing

Primitive values (int, long, double) are wrapped in objects (Integer, Long, Double). The wrapper object is allocated on the heap and requires GC.

```java
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // boxes: i -> new Integer(i), heap allocation
}

int sum = 0;
for (Integer x : list) {
    sum += x;  // unboxes: Integer -> int, heap access
}
```

Escape analysis mitigates this in some cases, but not all.

### High Allocation Rate

Even though the JVM can allocate very quickly (bump-pointer allocation in Eden is a single instruction), if you allocate millions of small objects, GC pressure increases.

```java
String result = "";
for (int i = 0; i < 100_000; i++) {
    result += i;  // creates new String each iteration
}
```

Each `+` operation allocates a new String. GC runs frequently. The solution: use `StringBuilder` instead of string concatenation.

### Long Monomorphic Call Chains

If code is deeply polymorphic (many different types flowing through), the JIT cannot specialize and must do full method lookups:

```java
interface Processor { int process(int x); }

class A implements Processor { 
    public int process(int x) { return x * 2; } 
}
class B implements Processor { 
    public int process(int x) { return x + 1; } 
}
class C implements Processor { 
    public int process(int x) { return x / 2; } 
}

// ... if this is called with A, B, and C randomly:
Processor p = getRandomProcessor();
for (int i = 0; i < 1_000_000; i++) {
    result += p.process(i);  // 3-way polymorphic call, no inlining
}
```

---

## 63.8 Startup Cost and Solutions

The JVM startup time includes:

1. **JVM initialization** (100–200 ms): load runtime, initialize GC, bootstrap class loader.
2. **Class loading** (100–500 ms): load application classes, verify bytecode.
3. **Static initialization** (0–1000+ ms): run static initializers.
4. **Warmup** (100–5000 ms): interpret code until methods are hot enough to JIT compile.

**Total startup time**: 500 ms to 10+ seconds, depending on the application size.

For serverless functions and CLI tools, this is a problem. Several solutions exist:

### AppCDS: Application Class Data Sharing

AppCDS pre-loads class metadata into a shared file. At startup, the JVM memory-maps the file instead of parsing `.class` files from disk.

**Startup improvement**: 10–30%.  
**How to use**:

```bash
# Step 1: Run the application with the JVM collecting class metadata
java -XX:ArchiveClassesAtExit=app.jsa MyApp

# Step 2: Run with the archive
java -XX:SharedArchiveFile=app.jsa MyApp
```

### GraalVM Native Image

Compile the entire application to native code ahead-of-time, without a JVM. This eliminates startup time (microseconds) and memory overhead.

**Startup improvement**: 100× (or more).  
**Tradeoff**: reflection and dynamic classloading must be declared at compile time. Loss of some JIT optimizations.  
**When to use**: CLI tools, serverless functions, microservices with strict startup requirements.

```bash
# Compile to native executable
native-image MyApp MyApp-native

# Run native executable (no JVM)
./MyApp-native
```

### CRaC and CRIU: Snapshot and Restore

CRaC (Coordinated Restore at Checkpoint) lets the JVM checkpoint its state (heap, loaded classes, compiled code) to disk. At startup, it restores from the checkpoint in microseconds.

**Startup improvement**: 100–1000×.  
**Tradeoff**: requires OS support (Linux with CRIU), works best with stateless workloads.  
**When to use**: containerized workloads where many instances are started and stopped frequently.

```bash
# After some warmup, checkpoint
jcmd PID JDK.checkpoint

# Restore checkpoint
java -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions \
    -XX:CRaCRestoreFrom=checkpoint.jcr
```

### Comparison

| Approach | Startup Gain | Memory Overhead | Complexity | Best For |
|----------|--------------|-----------------|-----------|----------|
| Default | 1× | Standard | Low | Desktop, servers with long uptime |
| AppCDS | 1.1–1.3× | Negligible | Low | Familiar JVM, modest improvement |
| Native Image | 50–100× | 10–50% less | Medium | CLI, serverless, strict latency |
| CRaC/CRIU | 100–1000× | Standard | Medium | Containerized, stateless services |

---

## 63.9 Worked Example: Watching Compilation

To see the JIT in action, run a program with compilation logging:

```java
public class HotLoop {
    public static void main(String[] args) {
        long sum = 0;
        for (int i = 0; i < 100_000_000; i++) {
            sum += fib(i % 20);
        }
        System.out.println("Result: " + sum);
    }
    
    static int fib(int n) {
        if (n <= 1) return n;
        return fib(n - 1) + fib(n - 2);
    }
}
```

Compile and run with logging:

```bash
javac HotLoop.java

java -XX:+PrintCompilation -XX:+PrintInlining HotLoop 2>&1 | head -50
```

Output (simplified):

```
    149   1       java.lang.System::identityHashCode (native)   (0 bytes)
    152   2       HotLoop::fib (15 bytes)
    156   3       HotLoop::main (25 bytes)
    180   3 %     HotLoop::main @ 14 (25 bytes)   made not entrant
    181   4       HotLoop::main (25 bytes)   made not entrant
    182   5       HotLoop::fib (15 bytes)
    ...
```

**What's happening:**

1. First, all code runs in the interpreter.
2. Around 150 ms, `fib` is called ~100,000 times. The baseline JIT (tier 1) compiles it (line: "2 HotLoop::fib").
3. `main` is also compiled.
4. At ~180 ms, `main` is recompiled by the optimizing JIT (tier 2) with more aggressive inlining ("5 HotLoop::fib" inlined into main).

The "made not entrant" lines mean the old compiled version is no longer used; a new version replaced it.

To see deoptimization, add a call that breaks the assumptions:

```java
public class HotLoopDeopt {
    static Object fib(int n) {
        if (n <= 1) return n;
        return (Integer) fib(n - 1) + (Integer) fib(n - 2);
    }
    
    public static void main(String[] args) {
        Object sum = 0;
        for (int i = 0; i < 100_000_000; i++) {
            sum = (Integer) sum + (Integer) fib(i % 20);
        }
        System.out.println("Result: " + sum);
    }
}
```

Run with:

```bash
java -XX:+PrintCompilation -XX:+PrintDeoptimization HotLoopDeopt 2>&1 | grep "deopt"
```

You'll see deoptimization events when types change unexpectedly.

---

## 63.10 Tradeoffs Across Languages and Runtimes

| Runtime | Strategy | Startup | Throughput | Latency | Best For |
|---------|----------|---------|-----------|---------|----------|
| **HotSpot JVM** | Interpreter + tiered JIT + G1 | 1–2 sec | 98–99% | 10–100 ms pauses | Server apps, data processing |
| **GraalVM Native Image** | AOT native | 10 ms | 98–99% | <1 ms pauses | CLI, serverless, strict startup |
| **CRaC/CRIU** | Interpreter + tiered JIT + checkpoint | 10 ms (after checkpoint) | 98–99% | 10–100 ms pauses | Containerized, ephemeral JVMs |
| **OpenJ9 (IBM)** | Interpreter + adaptive JIT | 1–2 sec | 99%+ | Similar to HotSpot | Alternative JVM, tuned for TPS |
| **Node.js (V8)** | Interpreter + Sparkplug + TurboFan | 100 ms | 95–98% | <50 ms pauses | Web servers, JavaScript |
| **Go runtime** | Direct execution + GC | 5 ms | 98–99% | <1 ms pauses | Services, network programs |

The JVM wins in **throughput and sustained performance**. Go wins in **startup and predictable latency**. Native Image wins where startup matters most.

---

## 63.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "The JVM is always slower than C++." | For compute-bound work, HotSpot JIT can match or exceed C++ due to profile-guided optimization and escape analysis. For latency-sensitive code, C++ is still faster because the JVM has pauses. |
| "Bytecode is the bottleneck." | Bytecode interpretation is fast in modern JVMs (template-based). The bottleneck is usually allocation rate or cache misses, not bytecode dispatch. |
| "Class loading is lazy." | Partially true. Classes are loaded lazily, but when loaded, static initializers run immediately. This can be expensive and unexpected. |
| "Escape analysis is a compile-time optimization." | It's a JIT-time optimization. The JIT has runtime information (what the object actually does). AOT compilers cannot do this. |
| "Write barriers in GC have negligible cost." | For generational GC with card marking, overhead is 5–10%. For concurrent collectors like ZGC, load barriers can add 5–10% CPU overhead. |
| "Deoptimization is rare." | It's not rare; it happens thousands of times per second in a busy application. But it's typically fast (microseconds) and doesn't disrupt user experience. |
| "You can tune the JVM to eliminate pauses." | No collector achieves zero pause. G1 targets 200ms. ZGC targets <1ms. Even the best collectors have a brief stop-the-world phase. |
| "More heap = faster JVM." | Not always. A larger heap means less frequent GC (good for throughput) but longer GC pauses when they happen. There's a tradeoff. |

---

## 63.12 Exercises

1. **Class file inspection**: Use `javap -c` to disassemble a small method (a loop, a method call). Compare the bytecode to a manually written assembly version. Why is the bytecode design stack-based?

2. **Profiling a hot loop**: Write a program that sums a million random integers in a tight loop. Run it with `-XX:+PrintCompilation`. How many times is the method recompiled? At what point does the JIT "kick in" (throughput stops improving)?

3. **Escape analysis**: Write a method that allocates a `List` and fills it, then returns a summary (not the list itself). Use `-XX:+PrintEscapeAnalysis` (diagnostic option). Is the list allocated on the stack or heap?

4. **GC tuning**: Run a memory-intensive application with different collectors (`-XX:+UseSerialGC`, `-XX:+UseParallelGC`, `-XX:+UseG1GC`). Measure wall-clock time, peak memory, and GC pause times. Which collector is fastest for your workload?

5. **Startup cost comparison**: Write a Java program that does some work on startup (load configuration, parse files). Time it under (a) regular JVM, (b) AppCDS, (c) GraalVM native-image. How much faster is native-image?

6. **Deoptimization**: Modify the `HotLoopDeopt` example to track deoptimization events. What types of code changes trigger deoptimization? Can you write code that deoptimizes in a pathological way (very frequent deoptimization)?

---

## 63.13 Summary

The JVM is a mature execution engine optimized for sustained performance, not startup. Classes are loaded lazily; bytecode is interpreted initially, then compiled to native code by a two-tier JIT. The baseline JIT (C1) is fast and safe; the optimizing JIT (C2) is aggressive and speculative, using runtime profiling data to inline, specialize, and eliminate checks.

Generational garbage collection exploits the weak generational hypothesis to keep young-generation pauses tiny. Modern collectors like G1 and ZGC layer concurrent marking and region-based collection to keep old-generation pauses reasonable (10–1000 ms).

For startup-critical workloads, AppCDS, GraalVM native-image, and CRaC offer exits from the warmup problem. For sustained throughput and latency-tolerant services, the JVM is a powerhouse.

Understanding the JVM means understanding the fundamental tradeoff: pay for startup and warmup, get predictable high throughput and sophisticated optimization that a human cannot write by hand.

---

**[← Previous: How Go Builds Binaries](05-how-go-builds-binaries.md)** · **[↑ Part 6](README.md)** · **[Next: Runtime GC Tradeoffs Across Languages →](07-runtime-gc-tradeoffs.md)**
