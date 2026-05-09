# Chapter 60 — How Python Executes Code

When you type `python script.py` and press Enter, a chain of events unfolds: lexing, parsing, AST construction, bytecode compilation, and then a giant dispatching loop that executes your code one bytecode instruction at a time. Along the way, the runtime manages memory through reference counting, holds a Global Interpreter Lock (GIL) to serialize thread execution, and maintains a call stack as heap-allocated frame objects. Each piece of this machinery has concrete implications for performance, debuggability, and design.

Unlike C++, which compiles to native machine code ahead of time, CPython (the reference implementation most of us use) is a **bytecode interpreter with a runtime**. This chapter opens that runtime and shows you exactly what happens when Python code runs.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the journey from Python source to bytecode, and why CPython caches `.pyc` files.
2. Read and interpret the output of `dis.dis()` — the bytecode disassembler — and understand what opcodes mean.
3. Describe how the CPython evaluation loop (the giant switch in `ceval.c`) dispatches opcodes and manages the value stack.
4. Explain why each function call creates a **frame object** on the heap, and how this enables generators and limits recursion.
5. Understand the Global Interpreter Lock (GIL) — what it is, when it releases, and why it prevents CPU-bound parallelism with threads.
6. Recognize that built-in types (int, list, dict, str) are implemented in C and bypass much of the bytecode overhead — explaining why NumPy is fast and pure Python loops are slow.
7. Predict where Python will be slow and when to reach for C extensions, Cython, or alternative runtimes.

---

## From Source to Bytecode

When you run a Python module, the interpreter does not directly execute your source code. Instead, it follows the three-layer model from Chapter 4:

1. **Source code** — your `.py` file.
2. **Bytecode** — a compact, language-agnostic intermediate form.
3. **Execution** — the interpreter reads bytecode and executes it.

The journey from 1 to 2 happens in stages: **lexing** → **parsing** → **AST construction** → **bytecode compilation**.

### Lexing and Parsing

Lexing splits the source text into tokens: `def`, `foo`, `(`, `)`, `:`, etc. Parsing assembles tokens into an **abstract syntax tree (AST)**, a tree representation of the program structure. If the parser fails, you get a `SyntaxError`.

```python
def square(x):
    return x * x
```

becomes an AST roughly like:

```
FunctionDef(
    name="square",
    args=[arg(name="x")],
    body=[
        Return(
            value=BinOp(
                left=Name(id="x"),
                op=Mult(),
                right=Name(id="x")
            )
        )
    ]
)
```

### From AST to Bytecode

CPython's compiler then walks the AST and emits **bytecode**, a sequence of 1- or 3-byte instructions. Each instruction has an opcode (the operation) and zero or two arguments.

Use the `dis` module to see bytecode:

```python
import dis

def square(x):
    return x * x

dis.dis(square)
```

Output (Python 3.11+):

```
  2           0 LOAD_FAST                0 (x)
              2 BINARY_MULTIPLY
              4 RETURN_VALUE
```

Read it as: at line 2 of the source, instruction 0 is `LOAD_FAST 0` (load local variable 0, which is `x`), instruction 2 is `BINARY_MULTIPLY` (pop two values, multiply, push result), instruction 4 is `RETURN_VALUE` (pop the top of stack, return it to the caller).

### Code Objects and the `.pyc` Cache

CPython packages bytecode into a **code object** — a C structure containing:

- The bytecode itself (a bytes blob).
- The constant pool (literals like `42`, `"hello"`, nested functions).
- Names and indices of local variables.
- The code object for each nested function.

Every function, class, and module gets its own code object.

When you `import module`, Python checks if a `.pyc` file exists in `__pycache__/`. If it does and it is newer than the source, Python unpickles the code object from the `.pyc` instead of recompiling. This is why repeated imports are instant — Python skips lexing, parsing, and compilation.

```bash
python -c "import my_module"    # Compiles, creates .pyc
python -c "import my_module"    # Uses .pyc, skips compilation
```

You can manually compile with `py_compile`:

```python
import py_compile
py_compile.compile('script.py', cfile='script.pyc')
```

---

## The CPython Evaluation Loop (Ceval)

The heart of CPython is the **evaluation loop** in `ceval.c`, a function called `_PyEval_EvalFrameDefault`. It is, conceptually, one giant switch statement on opcode.

### The Value Stack

The evaluation loop maintains a **value stack** — a dynamically-resized array of Python objects. Most opcodes pop arguments from the stack, perform an operation, and push a result.

For `square(5)`:

```
LOAD_FAST 0    # Push x (5 to the stack)
               # Stack: [5]
BINARY_MULTIPLY # Pop two values (both 5), push 5*5=25
               # Stack: [25]
RETURN_VALUE   # Pop 25, return from function
```

### A Sampling of Opcodes

CPython has about 120 opcodes. A few key ones:

- `LOAD_FAST(index)` — push local variable `index` onto the stack.
- `LOAD_GLOBAL(index)` — push global variable `index` onto the stack (slower; does a dict lookup).
- `LOAD_CONST(index)` — push constant `index` onto the stack (from the function's constant pool).
- `STORE_FAST(index)` — pop the stack and assign to local variable `index`.
- `BINARY_ADD`, `BINARY_MULTIPLY`, etc. — pop two values, perform operation, push result.
- `CALL_FUNCTION(argc)` — pop `argc` arguments and a callable, call it, push the return value.
- `RETURN_VALUE` — pop the stack and return from the function.
- `JUMP_ABSOLUTE(target)` — jump to bytecode offset `target` (no condition).
- `POP_JUMP_IF_FALSE(target)` — pop the stack; if false, jump to `target`; otherwise continue.

### Threaded Interpretation and Computed Gotos

Modern CPython uses **computed gotos** for speed. Instead of a single switch statement (which has branch prediction overhead), each opcode handler jumps directly to the next handler using a computed label. This is possible in C but not standard; it requires GNU extensions:

```c
#define DISPATCH() goto *opcode_dispatch[*next_instr]
void _PyEval_EvalFrameDefault(...) {
    static void *opcode_dispatch[] = {
        &&_LOAD_FAST, &&_BINARY_MULTIPLY, ...
    };
    #define LOAD_FAST(...)  { ... DISPATCH(); }
    #define BINARY_MULTIPLY { ... DISPATCH(); }
    ...
}
```

The benefit: each opcode handler's final instruction is a direct jump to the next handler, not a branch back to a switch statement. On modern CPUs with branch predictors, this is measurably faster.

---

## Frames and the Stack

Each function call creates a **frame object** on the heap. A frame is a Python object (a `PyFrameObject` in C) that holds:

- **Local variables** — a fast array of Python objects.
- **The instruction pointer** — which bytecode offset the interpreter is executing.
- **The value stack** — the operand stack for this function.
- **A link to the caller's frame** — so the interpreter knows where to return.

This is crucial: **the call stack is not the C stack**. When Python calls a function, it allocates a frame object on the Python heap and appends it to a linked list. When the function returns, the frame is (eventually) deallocated by the garbage collector.

### Why Frames on the Heap?

This design enables **generators**. A generator is a function that can pause and resume:

```python
def count():
    i = 0
    while True:
        yield i
        i += 1

g = count()
print(next(g))      # 0
print(next(g))      # 1
```

When `yield` is executed, the interpreter saves the entire frame (locals, instruction pointer, stack) and suspends. When `next()` is called, the interpreter restores the frame and resumes from where it left off. This is impossible if frames live on the C stack — the C stack is gone when the function returns.

### Recursion Limits

Because frames are heap-allocated objects, deep recursion eventually hits the **recursion limit**, set by default to about 1000:

```python
sys.getrecursionlimit()     # Usually 1000
```

This limit exists to prevent stack overflow — but not C stack overflow (there's no risk of that). Rather, it prevents unbounded memory usage. Each frame is ~300 bytes; a recursion depth of 1 million would allocate 300 MB just for frames. The limit is a safety valve.

You can raise it:

```python
sys.setrecursionlimit(10000)
```

But this is rarely a good idea. If you hit the recursion limit, the solution is usually to convert to iteration or use an explicit stack (a list), not to raise the limit.

---

## The Global Interpreter Lock (GIL)

The GIL is **one mutex** that the interpreter acquires before executing most Python bytecodes. At any moment, only one OS thread can hold the GIL and run Python bytecode.

This is a deliberate design choice, not a limitation of Python the language — it is a limitation of CPython the implementation.

### Why the GIL Exists

CPython uses **reference counting** (Chapter 12) to manage memory. Every object has a refcount field. When you assign `x = y`, the interpreter increments the refcount of the object `y` refers to. When a variable goes out of scope, the refcount is decremented.

Refcounting is not atomic. If thread A increments a refcount and thread B decrements it simultaneously on different CPU cores, the result is undefined (it may become corrupted, or the object may be freed while still in use).

To prevent this, CPython holds a single global lock during refcount operations. This is the GIL.

### The GIL's Release Cycle

The GIL is not held continuously. The interpreter releases it periodically:

1. **Every ~100 bytecodes** — by default, the interpreter releases and reacquires the GIL every 100 bytecode operations. This is controlled by `sys.getswitchinterval()` (in seconds, not bytecodes, as of Python 3.2):

```python
sys.getswitchinterval()     # Default: 0.005 (5 milliseconds)
sys.setswitchinterval(0.001)
```

2. **During system calls** — when a thread makes a system call (read, write, socket operations, sleep, etc.), it releases the GIL. This is why I/O-bound Python code can parallelize: while thread A is blocked in a read, thread B can run Python bytecode.

3. **During NumPy and C extension calls** — well-written C extensions release the GIL before doing long-running work.

### The Consequence: CPU-Bound Parallelism Is Serialized

If you spawn multiple threads to do CPU-bound work, only one thread runs Python bytecode at a time:

```python
import threading
import time

def cpu_work():
    """Compute-intensive loop"""
    total = 0
    for i in range(100_000_000):
        total += i

# Single-threaded: ~10 seconds
start = time.time()
cpu_work()
print(f"Single-threaded: {time.time() - start:.2f}s")

# Multi-threaded with GIL: also ~10 seconds (or slower due to context switches)
start = time.time()
t1 = threading.Thread(target=cpu_work)
t2 = threading.Thread(target=cpu_work)
t1.start()
t2.start()
t1.join()
t2.join()
print(f"Two threads: {time.time() - start:.2f}s")
```

Both take about the same time. The two threads do not run in parallel; they take turns holding the GIL, and the overhead of context switching makes the threaded version slightly slower.

**For I/O-bound work**, threads are fine:

```python
import threading
import time
import socket

def io_work():
    """Network I/O"""
    for _ in range(10):
        s = socket.socket()
        s.connect(("example.com", 80))  # Releases GIL
        s.close()

start = time.time()
t1 = threading.Thread(target=io_work)
t2 = threading.Thread(target=io_work)
t1.start()
t2.start()
t1.join()
t2.join()
print(f"Two threads: {time.time() - start:.2f}s")  # Much faster; threads run in parallel
```

### The Future: PEP 703 (No-GIL)

As of 2024, CPython is working toward a **no-GIL** implementation (PEP 703). The proposal is to replace the global lock with per-object locks and biased locking, enabling true parallelism. This is a major undertaking — it requires rewriting the interpreter's synchronization logic and potentially affects C extensions — but the goal is clear: allow CPU-bound code to use threads.

In the meantime, for CPU-bound work, use `multiprocessing` (separate processes, each with its own GIL) or alternative runtimes like PyPy (which uses generational GC instead of refcounting, and does not have a GIL).

---

## Built-In Types as C Implementations

`int`, `list`, `dict`, `str`, `tuple` — these are not implemented in Python. They are written in C and are tightly integrated with the runtime.

### Why This Matters

When you call a built-in method, you often skip the bytecode interpreter entirely:

```python
x = [1, 2, 3]
x.append(4)      # Calls C code, not Python bytecode
```

The `append` method is a C function that directly manipulates the internal array. There is no bytecode, no stack frame, no opcodes. It just runs, and returns.

In contrast, a pure-Python method:

```python
class MyList(list):
    def my_append(self, item):
        self.append(item)
        return self

x = MyList([1, 2, 3])
x.my_append(4)   # Python bytecode: LOAD_FAST, LOAD_METHOD, CALL_METHOD, etc.
```

The call to `my_append` creates a frame, executes bytecode, then returns.

### The Performance Implication

This is why **NumPy is fast**. NumPy arrays and operations are implemented in C. When you do:

```python
import numpy as np
x = np.array([1, 2, 3, 4, 5])
y = x * 2
```

There is no Python loop. The `*` operator calls C code that operates on the entire array in one go. The overhead of the Python interpreter is paid once, not per element.

In contrast, a pure-Python loop:

```python
x = [1, 2, 3, 4, 5]
y = [item * 2 for item in x]   # Pure Python; creates frame, executes bytecode for each iteration
```

is much slower because the interpreter overhead is paid for each element.

### Consequence: Not All Operations Are Equal

Some operations are "fast" because they are built-in; others are slow because they require bytecode:

```python
d = {'a': 1, 'b': 2, 'c': 3}

d['a']           # Fast: dict lookup is C code
for k in d:      # Slow: iterator protocol + bytecode loop
    print(k)

len(d)           # Fast: C code
d == {'a': 1}    # Fast: C code (dict comparison)
```

---

## How `__main__` and Modules Work

Every Python module is executed in its own namespace, creating a code object for the module body. The interpreter assigns each module a unique `__name__` attribute:

- If you run `python script.py`, the script's `__name__` is `"__main__"`.
- If you `import script`, the script's `__name__` is `"script"`.

This is why you see:

```python
if __name__ == "__main__":
    main()
```

This pattern detects whether the module was run directly or imported, and only executes `main()` if it was run directly. We will cover this in detail in the next chapter.

---

## Worked Example: From Source to Profiling

Let's trace a small function from source to profile.

### Source Code

```python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

result = fibonacci(30)
print(result)
```

### Bytecode

```python
import dis
dis.dis(fibonacci)
```

Output (simplified):

```
  2           0 LOAD_FAST                0 (n)
              2 LOAD_CONST               1 (1)
              4 POP_JUMP_IF_FALSE       10

  3           6 LOAD_FAST                0 (n)
              8 RETURN_VALUE

 4     >>    10 LOAD_GLOBAL              0 (fibonacci)
             12 LOAD_FAST                0 (n)
             14 LOAD_CONST               2 (1)
             16 BINARY_SUBTRACT
             18 CALL_FUNCTION            1
             20 LOAD_GLOBAL              0 (fibonacci)
             22 LOAD_FAST                0 (n)
             24 LOAD_CONST               3 (2)
             26 BINARY_SUBTRACT
             28 CALL_FUNCTION            1
             30 BINARY_ADD
             32 RETURN_VALUE
```

Read it: if `n <= 1` (line 2-4), return `n`. Otherwise, load the global `fibonacci`, compute `n-1`, call it, then repeat for `n-2`, add the results, and return.

### Profiling

```python
import cProfile

cProfile.run('fibonacci(30)')
```

Output (excerpt):

```
ncalls    tottime    cumtime
1         0.001     27.354  fibonacci(30)
2692537   26.903    26.903  fibonacci(...)
```

There are **2.6 million function calls** to compute `fibonacci(30)`. Each call:
1. Creates a frame object.
2. Loads and dispatches the function.
3. Executes the bytecode.
4. Deallocates the frame.

This is the overhead: 2.6 million times paying the cost of the interpreter.

### The Hot Path

The slow part is not the bytecode itself (arithmetic is fast). The slow part is the **function call overhead**: creating frames, dispatching, destroying frames. This is why recursive algorithms in Python are slow.

### Optimization: Memoization

Cache results to avoid recomputation:

```python
import functools

@functools.lru_cache(maxsize=None)
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

result = fibonacci(30)  # Now instant
```

With memoization, `fibonacci(n)` is called only once per unique value of `n`, so 31 calls instead of 2.6 million. The interpreter overhead is avoided for cached calls.

### Alternative: C Extension

Write the function in C:

```c
static PyObject *
fibonacci(PyObject *self, PyObject *args) {
    long n;
    if (!PyArg_ParseTuple(args, "l", &n)) return NULL;
    if (n <= 1) return PyLong_FromLong(n);
    return fibonacci(...);  // Recursive, but no Python interpreter overhead
}
```

A C extension can run 50-100x faster because there is no bytecode interpretation, no frame allocation, no refcounting overhead (though there is still some).

---

## Where CPython Is Slow

CPython's design has inherent bottlenecks:

1. **Method dispatch overhead** — every method call creates a frame and executes bytecode to look up the method, check its type, and call it.
2. **Global variable lookups** — `LOAD_GLOBAL` is a dict lookup, not an indexed access. Accessing a global variable is much slower than a local variable.
3. **No inlining** — the interpreter cannot inline function calls. A call to a small function pays the full overhead.
4. **GIL serialization** — CPU-bound code cannot parallelize across threads.
5. **Dynamic typing** — every operation (e.g., `a + b`) must check the types of `a` and `b` at runtime to decide which C function to call.

Most "make Python fast" projects address one of these:

- **PyPy** — uses a JIT to compile hot bytecode paths to machine code, eliminating interpreter overhead and enabling inlining.
- **Cython** — compiles Python-like code to C, removing the interpreter entirely for typed code.
- **mypyc** — compiles type-annotated Python to C, similar to Cython but integrated with mypy.
- **NumPy, Pandas** — use C/Fortran implementations for numeric work, avoiding the Python loop altogether.

---

## Tradeoffs: Choose the Right Tool

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **Pure CPython** | Easy to write, debug, profile. Standard library is mature. | Slow on CPU-bound work. GIL limits threads. | Prototyping, I/O-bound code, one-off scripts. |
| **NumPy/Pandas** | C implementations, fast vectorized operations. | Only useful for array-like data. Requires data rearrangement. | Numeric computing, data science. |
| **Cython** | Compiled to C, can match C speed. Optional typing. | Longer build time. C error messages. | Performance-critical inner loops. Gradual optimization. |
| **PyPy** | Faster than CPython on long-running code (JIT). Mostly compatible. | Smaller ecosystem. Startup slower. | Server-side code, long-running processes. |
| **multiprocessing** | True parallelism (no GIL). | Separate processes, high startup/communication cost. | CPU-bound work needing parallelism. |

---

## Common Misconceptions

**"Python is slow."** — Python *is* slow at certain workloads (single-threaded numeric loops, recursive algorithms). But most real-world Python code is I/O-bound (network requests, disk I/O, database queries), and I/O latency dominates. Python's overhead is negligible compared to network delays.

**"The GIL prevents all parallelism."** — False. I/O-bound code parallelizes fine across threads because threads release the GIL during I/O. CPU-bound code does not parallelize with threads, but that is a rare use case in Python (usually delegated to NumPy, multiprocessing, or PyPy).

**"All Python code is equally slow."** — False. Calling built-in operations (list append, dict lookup, NumPy operations) is fast because they are C code. Pure-Python loops are slower. The difference is 1-2 orders of magnitude.

**"You need to use type hints to go fast."** — Type hints do not make CPython faster. They help mypy and Cython (which can use them for optimization), but CPython ignores them. CPython pays the cost of dynamic typing regardless.

---

## Exercises

1. **Bytecode reading**: Write a function that uses a loop and a conditional. Run `dis.dis()` on it. Identify the `JUMP_ABSOLUTE` and `POP_JUMP_IF_FALSE` instructions. Trace the execution of one iteration.

2. **Frame introspection**: Write a function that calls `sys._getframe()` to inspect the current frame object. Print the frame's local variables, the code object, and the instruction pointer. Call your function and observe how the frame changes.

3. **GIL release timing**: Write two threads that increment a shared counter. Time single-threaded and multi-threaded versions. Observe that the multi-threaded version is not faster (or slower due to context switches). Then rewrite the counter with `multiprocessing` and observe true parallelism.

4. **Built-in vs. Python**: Write a function that sums a list using a Python loop (`total += item for item in lst`). Write another that uses `sum(lst)`. Profile both with `timeit.timeit()`. The built-in should be 5-10x faster.

5. **Memoization impact**: Write an expensive recursive function without memoization and profile it with `cProfile`. Then add `@functools.lru_cache()` and profile again. Observe the reduction in function calls and execution time.

---

## Summary

CPython executes Python code by compiling it to bytecode, caching the bytecode in `.pyc` files, and then dispatching opcodes in a giant evaluation loop. Each function call allocates a frame object on the heap, enabling generators but also limiting recursion depth and creating overhead. Memory is managed through reference counting, and the Global Interpreter Lock (GIL) serializes threads — making CPU-bound code single-threaded but allowing I/O-bound code to parallelize. Built-in types and operations are implemented in C and are much faster than pure-Python code, which is why NumPy-based code is fast and why pure-Python loops are slow. Understanding this machine — the bytecode, the frame stack, the GIL, the cost of interpretation — is the key to writing fast Python and knowing when to reach for C extensions, NumPy, Cython, or alternative runtimes.

---

> **[← Previous: JIT Compilation](02-jit-compilation.md)**  ·  **[↑ Part 6](README.md)**  ·  **[Next: Why Python Uses `__name__ == "__main__"` →](04-why-python-uses-name-main.md)**
