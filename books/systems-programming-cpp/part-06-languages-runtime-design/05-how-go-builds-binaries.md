# Chapter 62 — How Go Builds Binaries

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain why a Go binary is a *single static executable* with no runtime dependencies — and what that means for deployment.
2. Trace the compilation pipeline (`go build`: source → SSA IR → machine code → linker) and understand why the Go compiler is fast.
3. Describe what is *inside* a Go binary — the runtime, the garbage collector, all your dependencies — and why "hello world" is 2 MB.
4. Use `GOOS` and `GOARCH` to cross-compile binaries and understand why Go's approach is unique among modern languages.
5. Understand the build cache and module system, and produce reproducible builds.
6. Recognize the boundary where Go's smoothness ends: cgo, the FFI (foreign function interface) to C, and why it is expensive.
7. Read the symbol table of a Go binary with `go tool nm` and identify the runtime infrastructure.
8. Reason about the tradeoffs: static vs. dynamic linking, cgo vs. pure Go, binary size vs. startup time.

A Go binary feels fundamentally different from a C++ binary or a Java `.jar`. You run it. No `LD_LIBRARY_PATH` hunt, no DLL dependencies, no "runtime install." That property — and the design decisions that enable it — is the story of this chapter.

---

## 62.1 The Design Choice: Static Linking by Default

The first decision Go made was radical: **link everything into a single executable**.

When you run `go build` on a Go project, the Go compiler:

1. Compiles all your source files plus the entire standard library into machine code.
2. Links them together with the Go runtime, the garbage collector, the network poller, the scheduler — everything — into one ELF/Mach-O/PE file.
3. Ships that one file to production.

Result: the binary is self-contained. It has *no external dependencies* (except `libc` on Unix if you use cgo). Run it anywhere the OS/arch combo matches.

Compare to C++: statically linking everything (the standard library, all your dependencies) is possible but unusual — most C++ binaries are dynamically linked. Compare to Java: every JVM app depends on the JVM being installed. Compare to Python: every Python script needs Python + the right packages installed.

Go's "just works" reputation is largely because of this one choice. The operational story is dramatically simplified: one artifact, ship it, run it.

The cost: **binary size**. A "hello world" in Go is 2–3 MB on Linux. A "hello world" in C is 16 KB statically linked or 8 KB dynamically linked. Why? Because Go includes its entire runtime in every binary — the scheduler, the GC, the allocator, reflection metadata.

---

## 62.2 The Compilation Pipeline

Understanding the pipeline tells you *why the Go compiler is so fast* and *what that speediness costs elsewhere*.

```
Source (.go files)
    |
    v
PARSE & LEX
    (Lexing, parsing, AST construction; very fast)
    |
    v
SEMANTIC ANALYSIS & TYPE CHECKING
    (Resolve names, type check, overload resolution, interface satisfaction)
    |
    v
LOWER TO SSA IR
    (Static Single Assignment form; compiler-agnostic, target-agnostic)
    |
    v
OPTIMIZATION PASSES
    (Inlining, dead code elimination, escape analysis, bounds check elimination)
    |
    v
CODE GENERATION
    (SSA IR -> machine code for target CPU)
    |
    v
LINKING
    (Go linker: resolve symbols, embed runtime, produce final binary)
    |
    v
Single Binary Artifact
```

### Key characteristic: Single-pass compilation (for many phases)

The Go compiler is written in Go itself (the compiler binary is built from Go source). It is *designed to compile fast*. One technique it uses: **multiple passes over the program are avoided**. Where a C++ compiler might make three or more passes (once to gather info, once to optimize, once to generate code, and so on), the Go compiler tries to do work in fewer passes, sometimes a single pass.

Additionally, the Go compiler **does not do expensive whole-program optimizations**. It won't spend hours analyzing every possible code path. Instead:

- Inlining is conservative (small functions only; the threshold is around 80 nodes in the AST).
- Loop unrolling is minimal.
- Vectorization is not automatic (no auto-SIMD).
- It does not attempt profile-guided optimization (PGO) without explicit feedback (though recent Go versions do support PGO if you provide profiles).
- It does not inline across packages by default.

**Why?** Because fast compilation is a first-class goal. The Go design philosophy values "compile in a second or two, even a large codebase" over "spend a minute optimizing each function." This is good for developer iteration — write code, compile, test, repeat — and not good for latency-sensitive systems.

The optimizer *does* do some aggressive things within each function:

- **Escape analysis**: determines which allocations can stay on the stack vs. must go on the heap. Allocations that *escape* (are returned, stored in globals, or passed to other functions) get heapified and GC'd; those that don't escape stay on the stack and are automatically freed. This is why a local struct returned from a function doesn't need a destructor — it was allocated on the stack and implicitly freed.
- **Bounds check elimination**: for safe array access in loops, the compiler proves bounds are always safe and elides redundant checks.
- **Dead code elimination**: code with no observable effect is deleted.
- **Common subexpression elimination**: `a + b` computed twice becomes one computation.
- **Nil check removal**: if the compiler can prove a pointer is non-nil, the nil check is removed.

But these are *function-local* optimizations, not whole-program. The compiler does not use link-time code generation (LTCG) or look across the entire call graph to decide inlining.

### Example: Escape analysis at work

```go
func makeStruct() MyStruct {
    return MyStruct{a: 1, b: 2}  // Does not escape; stack-allocated
}

func makePointer() *MyStruct {
    s := MyStruct{a: 1, b: 2}
    return &s  // Escapes; Go compiler heap-allocates it
}
```

The first function allocates the struct on the stack; returning it copies the struct. The second function appears to return a pointer to a stack variable — which is undefined behavior in C++ — but Go's compiler *automatically* detects the escape and allocates the struct on the heap. You don't need to call `new`; the compiler handles it. This is why Go code is safe without manual memory management.

### The linker: Not the system linker

Go does not use the system's linker (`ld` on Linux, `ld64` on macOS, `link.exe` on Windows). It uses its own linker, written in Go, embedded in the `go` binary. This linker:

- Knows about the Go runtime's expectations and can statically link the runtime into the binary.
- Can generate the runtime's metadata tables (like `pclntab` for stack traces and `runtime.SymTab` for reflection).
- Resolves all symbols at link time.
- Strips symbols with `-ldflags="-s -w"` and other linker directives.
- Produces position-independent executables (PIE) by default on most platforms for ASLR support.

**Why a custom linker?** The system linker is designed for C/C++, with assumptions that don't hold for Go:

- The system linker assumes dynamic linking is common; Go wants static linking by default.
- The system linker doesn't know about goroutines, the GC, or the runtime scheduler.
- The system linker doesn't automatically generate the pclntab; Go needs to do it.
- Go's custom linker can emit special sections and metadata that only Go's runtime understands.

The linker also handles a key feature: **stripping**. When you use `-ldflags="-s"`, the linker removes the symbol table but *keeps* the pclntab. This is why Go stack traces work even on stripped binaries — the function PC-to-source mapping is not in the symbol table; it's in pclntab, which is considered part of the runtime data.

---

## 62.3 What's Inside a Go Binary

Open a Go binary and you find:

1. **Your code and all dependencies**, compiled to machine code.
2. **The Go runtime**: the goroutine scheduler, the memory allocator, the GC.
3. **The network poller**: the machinery for goroutines to sleep on I/O without threads.
4. **Reflection metadata**: type information for `reflect` and runtime type assertions.
5. **The syscall library and runtime stubs**: to talk to the OS.
6. **String tables, vtables, and tables for the scheduler and GC**.

This is why a `hello world` is 2 MB, even though the actual code to print "hello world" is tiny. You're shipping the entire execution substrate.

### Experiment 62.1 — inspect a Go binary

```bash
cat > hello.go << 'EOF'
package main
import "fmt"
func main() {
    fmt.Println("Hello, world!")
}
EOF

go build -o hello hello.go
ls -lh hello              # Check the size
go tool nm hello | head   # See the top-level symbols
```

You'll see something like:

```
T _rt0_amd64_linux
T main.main
T runtime.gcMark
T runtime.schedule
T ... (hundreds more)
```

The `T` means "text" (executable code). Notice all the `runtime.*` symbols — those are the scheduler, GC, allocator. They are *all in your binary*, even though you wrote just one tiny function.

Now try:

```bash
go build -ldflags="-s -w" -o hello hello.go
ls -lh hello
```

The `-s` strips the symbol table; the `-w` strips the DWARF debug info. The binary shrinks.

---

## 62.4 Static Linking: The Default and the Cost

When Go links a binary, it links *statically* by default. That means:

- All object files are combined into one.
- All symbols are resolved at link time.
- All dependencies are embedded in the final binary.

On Linux, the only exception is `libc` (via cgo). If you use `cgo`, the binary depends on `libc.so.6`. If you do *not* use `cgo`, the binary has no external dependencies — not even `libc`.

To see this:

```bash
# Pure Go binary
ldd ./hello
# Output: "not a dynamic executable"

# Or on macOS:
otool -L ./hello
# Output: (mostly just the Mach-O headers, not external .dylib dependencies)
```

**Cost of static linking:**

1. **Binary size**: You carry the entire runtime with you, even for trivial programs.
2. **Slow updates**: If a security patch is released for `libc`, you must rebuild and redeploy. With dynamic linking, you just update the system `libc`.
3. **Container bloat**: Go binaries are bigger than minimal C binaries; they don't benefit from shared libraries.

**Benefit:**

- Reproducible deployments.
- No "works on my machine" with different system libraries.
- Simpler operations: copy the binary, run it, done.

---

## 62.5 Cross-Compilation: One Toolchain, All Platforms

One of Go's smoothest features: **cross-compilation is built-in**.

```bash
# Compile for Linux ARM64 on macOS
GOOS=linux GOARCH=arm64 go build -o hello-arm64 hello.go

# Compile for Windows x86-64
GOOS=windows GOARCH=amd64 go build -o hello.exe hello.go

# Compile for older 32-bit systems
GOOS=linux GOARCH=386 go build -o hello-i386 hello.go
```

This works because:

1. **Go's compiler backends for every OS/arch combination are built in**. You don't download a separate compiler for ARM; your `go` binary *contains* the ARM backend.
2. **No external linker dependency**. Since Go uses its own linker, it doesn't depend on the system's `ld` or `cc` toolchain.
3. **No platform-specific startup code**. Go's runtime abstracts OS details.

No other major compiled language has this out of the box. C++ needs a separate cross-compiler toolchain (`arm64-linux-gcc`). Rust's approach is similar to Go, but less seamless in practice. Java and Python don't care because they're interpreted/JITted, but they have the dependency problem.

### When cross-compilation breaks: cgo

If you use `cgo` to call C code, cross-compilation breaks:

```bash
# This fails if the C code was compiled for a different arch
GOOS=linux GOARCH=arm64 go build  # and some .c files
# error: can't run host tools on target platform
```

**Why?** The C code has to be compiled for the target platform, and `cgo` can't do that without a C cross-compiler on your machine.

---

## 62.6 The Module System and Build Cache

### go.mod and Reproducible Builds

Go uses semantic versioning and a `go.mod` file to lock dependency versions:

```go
module github.com/myorg/myapp

go 1.21

require (
    github.com/some/lib v1.2.3
    github.com/other/lib v0.5.0
)
```

Additionally, Go maintains a `go.sum` file that records the cryptographic hash of each dependency:

```
github.com/some/lib v1.2.3 h1:abc123...
github.com/some/lib v1.2.3/go.mod h1:def456...
```

**Why both?** The `.sum` file verifies that a dependency's source hasn't been tampered with. If you fetch `v1.2.3` from the Go module repository and the hash doesn't match, `go build` fails. This is a supply-chain security feature.

Every `go get` records the version and hash. `go build` fetches that exact version. This ensures reproducibility: *the same source code and the same `go.mod`/`go.sum` always produce the same binary*, assuming you use the same Go compiler version.

To make the binary byte-for-byte reproducible across machines, use:

```bash
go build -trimpath -ldflags="-s -w"
```

The `-trimpath` flag removes absolute file paths from the binary (paths to your `/Users/you/projects` directory, for example). This means two builds on different machines will produce identical binaries — a critical property for verifiable supply chains and auditable deployments.

Without `-trimpath`, the binary embeds your file paths, and two builds on different machines will differ. This doesn't affect runtime behavior, but it breaks bit-for-bit reproducibility and can leak filesystem structure.

### Build Cache

`go build` caches compilation results per package in `$GOPATH/pkg/mod/cache` (or `$HOME/.cache/go-build` on Unix). If `mylib` hasn't changed since the last build, the cached object code is reused. This makes incremental rebuilds very fast — often just re-linking.

Cache key: hash of the package's source code, all its dependencies' hashes, the Go version, and the build flags. Change any `.go` file, change a dependency version, or change the Go toolchain version, and the cache automatically invalidates.

You can inspect the cache:

```bash
go clean -cache  # Clear it
```

The build cache is one reason Go compilation is fast — most of the time, you're re-linking, not recompiling.

---

## 62.7 Cgo: Where the Smoothness Ends

`cgo` is Go's FFI (foreign function interface) to C. It lets you call C functions from Go and vice versa:

```go
package main

import "C"

//export GoAdd
func GoAdd(a C.int, b C.int) C.int {
    return a + b
}

func main() {
    sum := C.int_add(5, 3)  // Call a C function
    // ...
}
```

Under the hood, `cgo` works by:

1. The Go compiler detects the `import "C"` directive.
2. It invokes the C compiler (gcc/clang) on the `.c` files in the package.
3. The object files are linked together by Go's linker at the end.

### The Cost of cgo

**1. Call overhead**

A cgo call (~200 nanoseconds) is much slower than a Go function call (~10 nanoseconds) — a 20× difference. The overhead comes from:

- Switching thread-local storage (disabling the GC).
- Saving and restoring registers that the C calling convention expects.
- Preventing the GC from scanning into the C function.
- Waiting for the C function to return.
- Re-enabling the GC and checking if any collections happened.

If your code calls a C function once per request in a web service, the overhead is negligible. If you call it 10,000 times per request, you've just cut your throughput dramatically.

**2. Prevents Static Linking (on Unix)**

With cgo, the binary *must* be dynamically linked against `libc`:

```bash
ldd ./program  # Output: libpthread.so, libc.so.6, etc.
GOOS=linux GOARCH=amd64 go build  # Works if no cgo
GOOS=linux GOARCH=amd64 go build  # Fails if cgo is present and no cross-compiler
```

On Linux, the binary no longer has the "all your code is in here" property. It now depends on the system's `libc` and `libpthread`, and if those are updated or missing, your binary breaks.

**3. Breaks Cross-Compilation**

You can't cross-compile a cgo binary without a C cross-compiler toolchain for the target. Go gives you ARM support automatically; cgo doesn't. You'd need `arm64-linux-gcc` installed on your Mac to build a cgo binary for ARM Linux. This eliminates Go's smoothest feature.

```bash
# This fails if the package uses cgo
GOOS=linux GOARCH=arm64 go build
# error: can't run host tools on target
```

**4. Memory Management Complexity**

C interop requires careful management:

- Go values passed to C must be pinned (cannot be GC'd while C code might reference them).
- C pointers into Go memory are allowed only in the current call; you can't return them.
- Type conversions between Go and C are manual.
- Buffer alignment and field packing must match.

```go
// Unsafe: the C function might save this pointer, and Go's GC might move it
var x int = 42
C.store_pointer(unsafe.Pointer(&x))  // BAD

// Safe: allocate on heap, C function uses it only during the call
p := C.malloc(C.sizeof_int)
C.store_int(p, 42)
C.free(p)  // OK, you control the lifetime
```

### When to use cgo

- You have a C library with no Go equivalent and you can't rewrite it.
- The C library is called infrequently (startup, or once per request).
- The rest of your code is pure Go (minimize cgo boundaries).
- You're willing to accept dynamic linking on Unix.

### When NOT to use cgo

- Your code calls C hot paths (10,000+ times per second).
- You want easy cross-compilation.
- You want a standalone, dependency-free binary.
- Your C dependency is frequently updated or not universally available.
- You're building a library (cgo makes it harder to use).

---

## 62.8 Symbols and Reflection Metadata

Go stores rich metadata in the binary to support reflection and stack traces.

### pclntab: Program Counter to Line Table

Every function has an entry in the `pclntab` (program counter to line number table):

```
PC range 0x4540a0 - 0x4540c0: main.Add (file: main.go line 5)
PC range 0x4540c0 - 0x4540f0: main.main (file: main.go line 8)
```

When a panic happens, the runtime walks the stack, looks up each PC in `pclntab`, and prints a stack trace. This is why Go stack traces are *so precise* — they're not guesses; they're looked up in a table built at compile time.

### runtime.SymTab and Type Metadata

Go also stores type information:

- The interface type hierarchy.
- Struct field names and types.
- Function signatures.
- Package and module information.

This is why `reflect.TypeOf` can tell you the exact type of any value at runtime, and why Go's error messages are so informative.

### Experiment 62.2 — inspect symbols and size

```bash
go build -o hello hello.go
go tool nm hello | grep -E "^[0-9a-f]+ [tT] " | wc -l
# Shows: hundreds of symbols

go tool objdump -s main.main hello | head -20
# Shows: the machine code of main.main

# See the binary's sections
size hello
# Output: text, data, bss sizes
```

### Stripping: Smaller Binaries, Lost Debugging

```bash
go build -ldflags="-s -w" -o hello hello.go
# -s: strip symbol table
# -w: strip DWARF debug info

# Typical size reduction: ~30–50%
# Cost: stack traces become hex PCs; cannot debug with `gdb`
```

---

## 62.9 Worked Example: Building, Analyzing, and Stripping

```bash
# Step 1: Build a simple program
cat > main.go << 'EOF'
package main

import "fmt"

func fibonacci(n int) int {
    if n <= 1 { return n }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    fmt.Println("fib(30) =", fibonacci(30))
}
EOF

# Step 2: Build and check size
go build -o fib main.go
ls -lh fib                           # ~2.0 MB
go tool nm fib | grep main.fibonacci # Locate the function
go tool objdump -s main.fibonacci fib | head -30  # See the assembly

# Step 3: Build stripped
go build -ldflags="-s -w" -o fib-stripped main.go
ls -lh fib-stripped                  # ~1.3 MB (35% smaller)

# Step 4: Inspect what was included
go tool nm fib | grep runtime | head -20
# You'll see: runtime.gcMark, runtime.schedule, runtime.mallocgc, etc.

# Step 5: See the binary's internal structure
readelf -S fib  (on Linux)
# or
otool -l fib    (on macOS)
# Shows the ELF/Mach-O sections and their sizes
```

The `main.fibonacci` symbol will be there, but its code size is tiny — just a handful of instructions. The 2 MB is dominated by the runtime.

---

## 62.10 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Static Go binary** | One artifact; no deps; trivial deployment; reproducible | Large size (~2 MB minimum); slow security updates | Microservices, CLIs, containerized workloads |
| **cgo-enabled binary** | Access to C libraries; leverage existing code | Slow cgo calls; dynamic linking; no cross-compilation; not standalone | Wrapping C libraries, gluing to existing C code |
| **Dynamically linked (rare)** | Smaller binaries; shared library memory; system library updates apply automatically | Runtime depends on system libraries; "works on my machine" issues | Not recommended for Go; use static linking |
| **Plugin .so files** | Extend binary without relinking; update plugins independently | Complexity; versioning; runtime loading costs | Rarely used; plugin ecosystem is small |

---

## 62.11 Common Misconceptions

**Misconception 1: "Go binaries are huge because Go is bloated."**

Reality: Go binaries include the entire runtime in every binary. For a simple CLI, you pay ~2 MB for the scheduler, GC, allocator, and metadata. But for a 50 MB web service, that 2 MB is noise. Compare to Java, where the JVM is hundreds of MB and you also have the `.jar`. Go's binary is actually *lean* for what it includes.

**Misconception 2: "Go is slow because the compiler doesn't optimize."**

Reality: Go's compiler is conservative to keep compilation fast. Peak performance is good (within 5–10% of C for CPU-bound code), but the compiler does not spend minutes chasing micro-optimizations. If you need 1% more performance, profile first; often the issue is algorithm or I/O, not compilation.

**Misconception 3: "Cross-compilation in Go is magic; it just works."**

Reality: Cross-compilation works for pure Go. The moment you add cgo, you need a C cross-compiler and it breaks. Always test cross-compiled binaries thoroughly.

**Misconception 4: "Go's static linking makes binaries safe."**

Reality: Static linking prevents *some* dependency issues but introduces *new* ones: if a security patch is released for `libc`, you must rebuild. With dynamic linking, the OS updates `libc` and all binaries benefit. Neither is universally "safer"; it's a tradeoff.

**Misconception 5: "You can strip symbols and break debugging."**

Reality: You can strip DWARF debug info (`-w`), but Go's stack traces still work because `pclntab` is separate. You lose the ability to set breakpoints in a debugger, but panics still print source locations. This is a Go-specific property, not true in C++.

---

## 62.12 Exercises

1. **Build and inspect.** Build a Go program, then:
   - Use `go tool nm` to list the top 20 largest symbols.
   - Use `go tool objdump -s runtime.schedule <binary>` to see assembly code from the runtime.
   - Use `readelf -S` (Linux) or `otool -l` (macOS) to list all sections and their sizes.

2. **Cross-compile and verify.** Write a small Go program, cross-compile it to three architectures:
   ```bash
   for arch in arm64 386 amd64; do
       GOOS=linux GOARCH=$arch go build -o app-$arch .
       file app-$arch  # Verify the architecture
   done
   ```
   Explain why this is feasible in Go but would require separate toolchains in C++.

3. **Measure cgo cost.** Write two versions of a compute-bound function: one in pure Go, one calling a C implementation via cgo. Benchmark both:
   ```bash
   go test -bench=. -benchmem
   ```
   Measure the cost per call and the call overhead. Discuss when this overhead is acceptable.

4. **Binary size comparison.** Write a trivial program ("hello world") in Go, C (static), and C (dynamic). Compare binary sizes:
   ```bash
   go build -o hello-go hello.go
   gcc -static -o hello-c hello.c
   gcc -o hello-c-dyn hello.c
   ls -lh hello-*
   ```
   Account for the differences. Why is the static C binary smaller? Why is the dynamic C binary much smaller?

5. **Reproducibility.** Build the same program twice, once with default flags and once with `-trimpath`. Compare the binaries:
   ```bash
   go build -o app1 .
   go build -trimpath -o app2 .
   sha256sum app1 app2
   ```
   They should differ. Then build both with `-trimpath` and verify they match. Explain what `-trimpath` does.

6. **Linker flags.** Experiment with `-ldflags`:
   ```bash
   go build -ldflags="-s" -o app-s .       # Strip symbols
   go build -ldflags="-w" -o app-w .       # Strip DWARF
   go build -ldflags="-s -w" -o app-sw .   # Strip both
   ls -lh app-*
   ```
   Measure the size reduction. Build a second version and intentionally cause a panic — can you still see the panic stack trace?

7. **Conceptual: Architecture decision.** You're building a microservice in Go that calls a C library 1,000 times per second. The C library is your company's proprietary code and cannot be rewritten. Discuss:
   - Why cgo's performance cost is problematic here.
   - What alternatives exist (wrap the C library in a separate C service; rewrite in Go; accept the cost).
   - What you'd measure to decide.

---

## 62.13 Summary

Go's approach to building binaries is philosophically different from C++, Java, and Python. By linking everything statically into a single executable, Go achieves frictionless deployment: one artifact, no dependencies, run anywhere. The cost is binary size and inability to benefit from shared libraries. The compiler's speed is a design goal, enabling fast iteration; the tradeoff is less aggressive optimization.

Understanding what's inside a Go binary — the runtime, the GC, the scheduler, the metadata tables — explains both why Go feels different to use and why binaries are larger than minimal C programs. Cross-compilation works smoothly for pure Go; the moment you use cgo, complexity and platform-specificity return. For systems programming and infrastructure, this tradeoff is a win; for tight memory or where C interop is essential, you pay a cost.

---

> **[← Previous: Chapter 61 — Why Python Uses `__name__ == "__main__"`](04-why-python-uses-name-main.md)** · **[↑ Part 6](README.md)** · **[Next: Chapter 63 — How the JVM Works →](06-how-the-jvm-works.md)**
