# Chapter 91 — How To Learn Any Programming Language

## Opening

After your first language, none of them are really new. Not the concepts — loops, functions, memory, error handling — but the *arrangement* of those concepts. Every programming language is a rearrangement of the same handful of decisions: where to put the type information, how to manage memory, what happens when an error occurs, how code actually runs.

This chapter is the explicit recipe: the questions to ask, in order, when you sit down with a language you've never seen before. It is not a guide to mastering a language. Mastery takes years and is specific to each language's idioms, ecosystem, and community conventions. This chapter is about the *first week* — how to become productive enough to write small programs, read existing code, and know where to dig deeper.

If you've read Parts 1–6 of this book, you have the decision axes. You know where languages sit on typing, memory, execution, concurrency, abstraction, and mutability. This chapter teaches you how to *use* that knowledge as a learning tool.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Identify a language's position on the six decision axes from its documentation and "hello world" code.
2. Use that position to predict what the language will make easy and hard.
3. Follow a structured first-week learning plan that emphasizes writing code over reading books.
4. Recognize the difference between learning the syntax (layer 1) and learning the semantics (layer 2).
5. Spot common footguns that catch every newcomer to a language and know where to find answers.
6. Read and understand code in a new language without having to understand everything at once.

---

## 91.1 The Decision-Axis Recap

Before learning any language, you should know where it sits on the six decision axes from Part 6. This is the shortest possible sketch that tells you what the language will *feel* like.

**Typing**: Does the language check types ahead of time (static) or at runtime (dynamic), or somewhere in between?

**Memory management**: Who frees memory? The programmer manually, the garbage collector, the borrow checker, or something else?

**Execution model**: How does source code become running code? Interpretation, bytecode VM, JIT compilation, or ahead-of-time compilation?

**Concurrency model**: How do you write parallel or concurrent code? Threads with shared memory, async/await, goroutines, actors, or something else?

**Abstraction level**: How close to the machine are you? Very low (assembly-like), low with good abstractions (C++, Rust), mid-level (Go, Java), or very high (Python, Haskell)?

**Mutability by default**: Can you reassign variables and change data in place, or is data immutable by default?

You can sketch these from the official documentation or a quick tutorial. The sketch predicts everything: what will be easy, what will be hard, where performance surprises hide, what kinds of bugs are most likely.

**Example sketch for Go:**
- Static typing
- Garbage collection
- Compiled to native code
- Goroutines + channels for concurrency
- Mid-level abstraction
- Mutable by default

**Implication:** Go prioritizes simplicity and concurrency. The type system will catch errors at compile time. Memory management will be automatic (but GC pauses are possible). Concurrency is lightweight and expressible. You won't think much about machine details, but you can inspect them if needed. The hardest part will be breaking the habit of thinking about memory management manually.

Do this sketch *before* you try to write anything. It takes five minutes and saves hours of confusion.

---

## 91.2 The Six Questions To Ask First

Once you know where the language sits on the axes, ask these six questions in order. Each answers a specific part of "how does the computer actually run my code?"

### Question 1: What does a "hello world" look like with no imports?

This tells you: what is the absolute minimum code you need to run? How does the language handle the simplest possible program?

Write it, run it, watch what happens. Does it compile? Interpret instantly? Require a runtime? How long does it take?

**Go:**
```go
package main

func main() {
    println("hello, world")
}
```

**Python:**
```python
print("hello, world")
```

**Rust:**
```rust
fn main() {
    println!("hello, world");
}
```

The differences are already visible: Go requires a package and a main function; Python just runs statements top-to-bottom; Rust requires a main function and a macro (println!). These differences ripple outward. In Python, you can write code at module scope; in Go, you can't (statements must live inside functions). This affects how the language is used.

### Question 2: How do I declare a function and call it?

This tells you: how does the language spell *intent*? What information must you provide, and what does the language infer?

Try writing a function that takes an integer and returns an integer:

**Go:**
```go
func double(x int) int {
    return 2 * x
}
```

**Python:**
```python
def double(x):
    return 2 * x
```

**Rust:**
```rust
fn double(x: i32) -> i32 {
    2 * x
}
```

Notice: Go and Rust require you to declare types; Python infers them. Go's return type is at the end; Rust's is after an arrow. Rust uses `2 * x` (implicit return); Go and Python use `return`. These are surface details, but they matter for reading speed.

### Question 3: How do I represent a record, struct, or class?

This tells you: how does the language bundle data and methods? Is data separate from functions, or combined?

Try writing a struct with fields and a method:

**Go:**
```go
type Point struct {
    X int
    Y int
}

func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}
```

**Python:**
```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def distance(self):
        return math.sqrt(self.x**2 + self.y**2)
```

**Rust:**
```rust
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn distance(&self) -> f64 {
        ((self.x as f64).powi(2) + (self.y as f64).powi(2)).sqrt()
    }
}
```

Go uses a standalone struct and methods defined *outside* the struct (receiver in parentheses). Python uses a class with a special `__init__` method. Rust separates struct definition from implementation blocks (impl). These structural differences mean different tooling, different error messages, and different idioms for composing behavior.

### Question 4: How do errors propagate?

This tells you: when something goes wrong, how does the language signal it? Does it throw exceptions, return error values, panic, or something else?

Try writing code that might fail (e.g., parsing a string as an integer):

**Go:**
```go
n, err := strconv.Atoi("not a number")
if err != nil {
    log.Fatal(err)
}
```

**Python:**
```python
try:
    n = int("not a number")
except ValueError as e:
    raise e
```

**Rust:**
```rust
match "not a number".parse::<i32>() {
    Ok(n) => println!("{}", n),
    Err(e) => eprintln!("Error: {}", e),
}
```

Go returns errors as values; Python raises exceptions; Rust returns a Result enum that you must unwrap. This is *fundamental*. Go forces you to check errors at the call site; Python lets you check at a distance; Rust forces you to handle both the success and failure cases. This shapes how you write entire programs.

### Question 5: How do I do I/O?

This tells you: how does the language talk to the outside world? Synchronous blocking, async/await, callbacks, or message passing?

Try reading a file:

**Go:**
```go
data, err := ioutil.ReadFile("file.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))
```

**Python:**
```python
with open("file.txt") as f:
    data = f.read()
print(data)
```

**Rust:**
```rust
use std::fs;

let data = fs::read_to_string("file.txt")?;
println!("{}", data);
```

All three are synchronous here, but the syntax differs. The `?` operator in Rust is shorthand for error propagation. The `with` statement in Python is a context manager. Go's error-checking pattern is everywhere. Understanding these patterns immediately helps you read code written by natives of that language.

### Question 6: How do I run tests?

This tells you: how does the language organize verification? What is the standard way to write and run tests?

Try writing a simple test:

**Go (built-in):**
```go
func TestDouble(t *testing.T) {
    result := double(4)
    if result != 8 {
        t.Errorf("double(4) = %d, want 8", result)
    }
}
```

**Python (unittest or pytest):**
```python
def test_double():
    assert double(4) == 8
```

**Rust (built-in):**
```rust
#[test]
fn test_double() {
    assert_eq!(double(4), 8);
}
```

Go puts tests in `*_test.go` files; Python uses a separate test framework; Rust uses `#[test]` attributes. This is less "core language" and more "ecosystem," but knowing how tests are written tells you how the community validates code. It also affects where you put test files and how you run them.

---

## 91.3 After the Six Questions: What You Now Know

Answer these six questions and you can write and run toy programs. You can read most code and *mostly* understand it without getting stuck on syntax. You are still in layer 1 — syntax — but you have the outline.

What you do *not* yet know:
- How the language's idioms differ from other languages you know.
- What patterns are considered natural and elegant vs. awkward and forced.
- What the standard library offers and what you reach for external packages for.
- Where performance surprises hide.
- What the community's conventions are for naming, structure, and testing.

Layer 2 — semantics and idioms — is where you spend the next few days.

---

## 91.4 The Idiom Phase: Reading The Standard Library

Once you can write small programs, your job is to learn what's *idiomatic* in the language. Idiomatic code is code that reads naturally to natives of the language. It follows the language's grain.

The cheapest way to learn idioms is to read source code — specifically, the standard library.

**Why the standard library?**
1. It is written by the language's designers, who understand the idioms better than anyone.
2. It is heavily scrutinized, so it represents the consensus on what "good code" looks like.
3. It uses only the core language features, so it's not hiding behind obscure libraries.
4. It covers a wide range of problems (parsing, I/O, concurrency, data structures), so patterns repeat and click into place.

**How to read it:**
Start with one familiar problem. If you know how to write a linked list, find the linked list implementation in the standard library. If you know regular expressions, find the regex package. Read it. You're not trying to understand every line. You're trying to answer:

- How are types organized? Are they interfaces or concrete types?
- How are functions named? `DoThing`, `do_thing`, or `doThing`?
- How is error handling woven in?
- How is concurrency used (if at all)?
- What patterns repeat?

**Example: Reading Go's `fmt` package (printing)**

Go's `fmt` package is small and essential. Reading it teaches you:
- How to use interfaces (any type that has a `String()` method will be formatted nicely).
- How Go names functions (uppercase for exported, lowercase for private).
- How to use receivers (the method syntax).

After reading `fmt`, you see how to make your own types play nicely with print statements. This is a concrete win.

**Example: Reading Rust's `Result` type**

Rust's `std::result::Result` is an enum:
```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Reading the standard library's implementation of `map`, `and_then`, `unwrap_or`, etc., teaches you how Rust idiomatically chains operations on fallible code. You start understanding why `?` exists and when to use it.

**Linters are also teachers.** Run the language's linter (Go's `go vet`, Python's `pylint`, Rust's `clippy`) on your toy program. The warnings often point to idiomatic violations. Read the warning, understand it, and you've learned something.

---

## 91.5 How To Read New Code: Top-Down

You will encounter a large codebase (someone's open-source project, your company's service, a framework). You cannot understand the whole thing at once. Here's the pattern:

**1. Find the entry point.**
Most programs have a `main` function, an `__main__` guard, a script that kicks things off, or a configuration file. Find it. Don't try to understand the whole architecture yet; just find where execution starts.

**2. Trace one feature end-to-end.**
Pick the simplest user-visible behavior you can find. For a web server: "what happens when I GET /ping?" For a CLI tool: "what happens when I run `cli list`?" Trace that one path through the code from entry point to output. You will pass through multiple files and functions. You will not understand most of them. That's fine. Use the IDE's "go to definition" liberally. Click into functions, read their docstrings, understand what they do in isolation, and come back.

**3. Don't try to understand the whole codebase.**
After tracing one path, you might understand 5–10% of the codebase. That's the goal. You now have a mental model of "how does the high-level flow work?" You can ask questions about specific components without being lost.

**4. Expand gradually.**
Next time you need to fix something or add a feature, you'll touch another path. Gradually, the codebase becomes less mysterious.

**Concrete example:** You're reading a Python web framework's source code. You want to understand how routing works.

- Entry point: Find `__main__` or the example app startup.
- One feature: Pick a route like `GET /users/<id>`. Trace it from the HTTP handler to the route matching code to the actual view function.
- Don't understand the whole thing: You'll pass through middleware, decorators, and context managers that you don't fully grasp. Skip them for now.
- Expand: Now you understand "a GET request matches a route and calls a function." Next time you encounter middleware, you'll dig into that.

---

## 91.6 Common Footguns: What Every Language Has

Every language has 3–5 specific things that surprise newcomers. These are not bugs in the language; they are consequences of the language's design choices on the decision axes. They are predictable if you know what axis causes them.

### Default Mutability and Reassignment

**Languages where this bites:** Python, JavaScript, Go, Java.
**Why:** They are mutable by default.

In Python:
```python
x = [1, 2, 3]
y = x
y.append(4)
print(x)  # [1, 2, 3, 4] — x was changed too!
```

You intended to copy the list. Instead, you got a reference. This is obvious in retrospect, but it catches everyone.

**In Rust:**
```rust
let x = vec![1, 2, 3];
let y = x;
println!("{:?}", x);  // Compile error: x was moved, not copied
```

Rust makes this explicit. You can't accidentally share mutable state.

**Where to find answers:** Language-specific FAQ sections on references vs. copies, mutability semantics, and object identity.

### Equality Semantics

**Languages where this bites:** Python, JavaScript, Java.
**Why:** The language has both value equality and reference equality, and mixing them is easy.

In Python:
```python
a = [1, 2]
b = [1, 2]
print(a == b)   # True (value equality)
print(a is b)   # False (reference equality)
```

In JavaScript:
```javascript
[] == []        // false
{} == {}        // false
[] == false     // true (implicit coercion!)
```

Each language has different rules. You must know them.

**Where to find answers:** Search "[language] equality" or "[language] == vs ===". The community has written about this extensively.

### Integer Overflow

**Languages where this bites:** Rust, C++, Go (in different ways).
**Why:** They are close to the machine.

In Rust, integers do not silently overflow in debug builds:
```rust
let x: u8 = 255;
let y = x + 1;  // Panic in debug mode; wraps to 0 in release mode
```

In Go, overflow wraps silently:
```go
var x uint8 = 255
var y uint8 = x + 1  // y is 0
```

**Where to find answers:** Documentation on integer overflow behavior, often in a section on "safe arithmetic" or "unsafe code."

### Exception vs. Result Types

**Languages where this bites:** Rust, Go (learning from Python/Java).
**Why:** Different error handling philosophies.

Rust *forces* you to handle errors:
```rust
let n = "not a number".parse::<i32>()?;  // Must unwrap or propagate
```

Python *allows* you to ignore them:
```python
try:
    n = int("not a number")
except:
    pass  # Silently ignore the error
```

Go forces checking:
```go
n, err := strconv.Atoi("not a number")
if err != nil {
    // Must handle
}
```

**Where to find answers:** Language's error handling guide or comparison posts like "[language] error handling" or "[language] vs [other language] exceptions."

### Import Rules and Namespace Pollution

**Languages where this bites:** Go, Python, JavaScript.
**Why:** Different rules about what gets exported and how namespaces work.

In Go, anything starting with a capital letter is exported:
```go
func PublicFunction() { }      // Exported
func privateFunction() { }     // Private
```

In Python, you have to be explicit:
```python
def public_function():
    pass

def _private_function():
    pass  # Convention only
```

In JavaScript (modern modules), you must explicitly export:
```javascript
export function publicFunction() { }
// Other functions are private to the module
```

**Where to find answers:** Language's visibility/export documentation.

---

## 91.7 A One-Day Learning Plan

Here is a realistic schedule for your first day with a new language. Adjust times based on how much context you have.

**Hour 1: Install and Hello World (60 min)**
- Install the language (compiler, interpreter, package manager).
- Write "hello world."
- Write a slightly more complex program (read a file, do math, print output).
- Get comfortable with the edit-run cycle.
- Goal: Know that the language is installed and you can execute code.

**Hour 2: Small Program in Your Domain (60 min)**
- Write a ~100-line program in something you know well. If you're a web developer, write a simple HTTP client or server snippet. If you do data processing, write code that reads a CSV and filters it. If you do systems work, write a simple networking tool.
- Don't optimize; just make it work.
- Goal: Know that you can write something slightly non-trivial and it works.

**Hour 3: Tests (60 min)**
- Read the testing framework documentation (should be 10 minutes).
- Write 3–5 unit tests for functions in your 100-line program.
- Run them.
- Goal: Know how to verify that your code is correct, and that the test framework is not mysterious.

**Hour 4: Read Standard Library Code (60 min)**
- Pick one familiar component from the standard library (string formatting, list operations, date manipulation, HTTP client).
- Read 50–100 lines of its source.
- Do not memorize; just absorb patterns.
- Goal: See what idiomatic code in this language looks like.

**Hour 5: Read Open-Source Code (60 min)**
- Find a small open-source project (GitHub stars 100–1000 range, so it's not tiny but not enormous).
- Find a single function or method that does something you understand conceptually.
- Read it, trace it, use "go to definition" to understand the helpers it calls.
- Do not try to understand the whole project.
- Goal: Know that you can read code written by experts and mostly follow it.

**Hour 6–8: Build Something Small You'd Actually Use (180 min)**
- Write a program that solves a problem you actually have. A build script, a data processor, a CLI tool, a web service.
- Make it work.
- Add tests.
- Deploy or share it.
- Goal: Know that you can write code that delivers value, and that the language is not just for exercises.

**Outcome:** After eight hours, you have written code, read code, and know where to look for answers. You are not fluent, but you are functional.

---

## 91.8 Why Learning Stalls

Most people hit a wall around day 3 or 4. The wall is almost always the same: **vocabulary overload**.

They have learned syntax (layer 1) but are drowning in terminology they don't understand: generics, traits, lifetimes, metaclasses, protocols, interfaces, decorators, middleware, monads, type constructors, etc. Each term is attached to a feature of the language that is completely foreign.

**How to break through:**
1. **Skip the abstract concepts for now.** You don't need to understand *why* a feature exists, just how to use it. You can learn the why later.
2. **Focus on writing working code.** If you need to use a feature, look up three examples of it in use, copy the pattern, and move on.
3. **Resist the temptation to read the whole book.** You will not remember it, and it will slow you down.
4. **Idioms come later.** It is fine to write code that is *correct* but not idiomatic. After a few weeks, you'll naturally start writing more idiomatically as you read more code.

Example: Learning Rust generics.

You see code like:
```rust
fn max<T: Ord>(a: T, b: T) -> T { ... }
```

This looks like magic. What is `T`? What is `: Ord`? Why is there angle brackets?

You don't need to understand this deeply on day 1. You can learn that:
- `T` is a type variable (like `<T>` in Java or `T` in C++).
- `: Ord` means "`T` must be something that can be compared."
- The angle brackets are syntax for "here comes a generic parameter."

Once you use it correctly a few times, the deeper understanding comes naturally.

---

## 91.9 A Worked Example: Learning Go When You Know Python

You know Python well. You've heard that Go is "simple" and want to try it. Here's how to learn it in one sitting if you follow this chapter's recipe.

**Setup (5 minutes):**
Install Go. Run `go version`. Good.

**Questions 1–6 (30 minutes):**

Question 1 — Hello world:
```go
package main
import "fmt"

func main() {
    fmt.Println("hello, world")
}
```

Differences from Python:
- Every file belongs to a package.
- Imports must be explicit.
- Code must live inside functions; no module-level code (except in Python via `if __name__ == "__main__"`, but Go requires it everywhere).

Question 2 — Functions:
```go
func add(x int, y int) int {
    return x + y
}
```

Differences from Python:
- Types are required (Go is statically typed; Python is dynamic).
- Return type is after the parameters.

Question 3 — Structs:
```go
type Person struct {
    Name string
    Age  int
}

func (p Person) Greet() string {
    return "Hello, " + p.Name
}
```

Differences from Python:
- Methods are defined *outside* the struct.
- The receiver (p) is explicit.
- Struct fields must have types.
- String concatenation uses `+`.

Question 4 — Error handling:
```go
n, err := strconv.Atoi("not a number")
if err != nil {
    log.Fatal(err)
}
```

Differences from Python:
- Go returns errors as values, not exceptions.
- You must check errors explicitly.
- Multiple return values are normal in Go.

Question 5 — I/O:
```go
data, err := ioutil.ReadFile("file.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))
```

Differences from Python:
- No `with` statement; you get bytes and must convert to string.
- Error checking is explicit.

Question 6 — Tests:
```go
func TestAdd(t *testing.T) {
    result := add(2, 3)
    if result != 5 {
        t.Errorf("add(2, 3) = %d, want 5", result)
    }
}
```

Differences from Python:
- Test functions must be named `Test*`.
- Must live in `*_test.go` files.
- The test object has methods like `Errorf`.

**At this point, 30 minutes in, you know:**
- Static typing is mandatory; Python's dynamic typing is gone.
- Error handling is different (no exceptions by default).
- Method receivers are explicit.
- Imports and packages work differently.
- Tests are built-in.

You can now write toy programs. You're in layer 1 (syntax).

**Read the standard library (30 minutes):**

Read `fmt` to see how Go handles printing. Then read `strings` to see string operations. You notice:
- Exported functions start with capital letters.
- Methods often return multiple values (result, error).
- Simple is valued (no fancy abstractions).

**Read open-source code (20 minutes):**

Find a small Go project on GitHub. Look at its main function. Trace one feature. You're learning idioms:
- Error handling is pervasive.
- Functions return (value, error) pairs.
- Interfaces are structural (any type with the right methods satisfies them).

**Build something (remaining time):**

Write a small CLI tool: read a JSON file, filter it, print results. Use the patterns you've seen. Don't worry about being idiomatic; just make it work.

**Outcome:** After ~2 hours, you can read Go, write Go, and know where the surprises are. You're not fluent, but you're functional. Layer 2 (idioms) will come in the next few weeks as you write more code.

---

## 91.10 Tradeoffs In Learning Approaches

Different people learn differently. Here are the main approaches and their tradeoffs:

| Approach | Pros | Cons | Best For |
|---|---|---|---|
| **Book-first** | Comprehensive; explains the why; builds intuition | Slow; easy to get lost in details; doesn't feel productive early | Deep understanding; languages with weak online docs |
| **Project-first** | Immediate feedback; motivation; learn in context | Can miss important concepts; frustration when you don't know something; slower overall | Engineers who learn by doing; rapid iteration |
| **Tutorial-driven** | Fast feedback; guides you step-by-step; usually free | Shallow; doesn't teach you to read code; can feel rote | Absolute beginners; first language |
| **Source-reading** | See how experts do it; learn idioms; no fluff | Hard without context; easy to get lost; very slow initially | Engineers with 2+ languages; already know the concepts |

**Recommendation:** Use **project-first** + **source-reading** after the first week. Write code immediately (project-first), get stuck, look up answers (quick tutorials or docs), then read the standard library and open-source code to see the pattern (source-reading). Use books only when you have a specific question.

---

## 91.11 Common Misconceptions

**Misconception 1: "I must read a whole book before I write code."**

False. Books are reference tools, not learning tools for most people. You learn by doing and then checking the book when you get stuck. Even language designers don't read their own language manuals cover-to-cover.

**Misconception 2: "I should master the language in one week."**

False. Mastery takes years. A week gets you functional. A month gets you idiomatic. Mastery comes after writing real projects and reading other people's code for years.

**Misconception 3: "Learning a new language is like learning the first one."**

False. The first language is slow because you're learning both the concepts (loops, functions, memory) and the syntax. The second language is faster because the concepts are familiar; you're just learning a new costume. By the fourth language, it takes days.

**Misconception 4: "If I can't understand this design pattern/feature, I'm not smart enough."**

False. Some language features exist for edge cases, and you don't need to understand them immediately. You can write years of code without understanding variance in generics, or protocol buffers, or actor isolation. Learn what you need, when you need it.

**Misconception 5: "The language's official tutorial is the best way to learn."**

Sometimes. But official tutorials often are written by language designers, not educators, and they often skip the why in favor of the what. Find an explanation that clicks for you — it might be a blog post, a conference talk, or someone's personal notes.

---

## 91.12 Exercises

**Exercise 1: The Six Questions.**
Pick a language you've never used. Spend 30 minutes answering the six questions from §91.2. Write your answers as short code snippets. Do not look up the "right" answer; just make your best guess and then check it by running the code.

**Exercise 2: Place It On The Axes.**
For the same language, place it on the six decision axes (§91.1). Write one paragraph predicting what the language will make easy and hard based on its position on the axes. Then spend an hour writing code in the language. Did your prediction match reality?

**Exercise 3: Read Standard Library Code.**
Pick one function from the language's standard library that you understand conceptually (e.g., "split a string", "sort a list", "parse JSON"). Read its source code. Note down three idioms you see that are specific to that language.

**Exercise 4: Top-Down Reading.**
Find an open-source project in the language you're learning. Pick a simple user-facing feature (e.g., "print help text" or "read a configuration file"). Trace that feature through the codebase from entry point to completion, using the IDE's "go to definition" freely. Write down the five main functions/methods that were called.

**Exercise 5: The Footgun Hunt.**
Ask in the language's community (subreddit, Discord, forum) "What is the most common mistake newcomers to [language] make?" Collect five answers. For each, write a short code example showing the footgun, and write the correct version.

**Exercise 6: One-Day Learning Plan.**
Pick a language you've never used. Spend one day (8–10 hours, split across multiple sessions if needed) following the one-day learning plan from §91.7. At the end of the day, write a small program that solves a real problem (even a tiny one). Share it with someone.

---

## 91.13 Summary

Learning a new programming language after your first (or second, or third) is not about learning new *concepts* — those transfer. It is about learning a new *arrangement* of the same concepts.

Use the decision axes to sketch the language in five minutes. Then ask the six questions to understand the absolute fundamentals. Write toy code immediately. Read the standard library to learn idioms. Read open-source code to see patterns in context. Avoid books unless you have a specific question.

Most importantly: every language has 3–5 footguns that surprise newcomers. Those footguns are not random; they follow from the language's position on the decision axes. If you know the axes, you can predict and avoid them.

After a week of this process, you will write functional code. After a month, you will write idiomatic code. After a year, you will write the kind of code that makes a native of the language nod and say "yes, that's the right way to do it." That depth is worth reaching for, but it comes much faster than it did for your first language, because the foundations are already there.

---

## 91.14 What's Next

You now have a recipe for learning any language. Part 9 is about building mental models that transfer across languages, and this chapter is the first step: how to *absorb* a new language quickly enough to be useful.

The next chapter, "Recognizing Common Runtime Patterns," is about spotting the same patterns appearing in different languages. You'll see how Python's decorators, Java's annotations, and Rust's traits are solving the same problems with different syntax. Recognizing those patterns lets you transfer knowledge even faster.

After that, we move to "Reading Any Framework" — the same top-down approach, applied to a large codebase.

---

> **[← Previous: Part 8 — Advanced Systems Thinking](../part-08-advanced-systems-thinking/13-false-sharing.md)** · **[↑ Part 9](README.md)** · **[Next: Recognizing Common Runtime Patterns →](02-recognizing-runtime-patterns.md)**
