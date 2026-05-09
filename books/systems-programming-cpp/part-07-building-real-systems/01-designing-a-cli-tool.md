# Chapter 68 — Designing a CLI Tool

A command-line interface is the most direct application of everything in Parts 1–6. You parse arguments (string processing and memory layout). You route to business logic (abstraction, layering). You stream input and output (I/O buffering, signals). You exit with a code that other programs read (process contracts). You handle signals that the OS sends (kernel interaction). You read environment variables and config files (file I/O, encoding). You format error messages that humans read.

Most engineers treat the CLI as secondary — a quick wrapper around business logic, built with a library and never revisited. The engineers who do it well know that a CLI is a microcosm of all the techniques this book teaches. A good CLI is the difference between a tool people reach for and one they tolerate.

This chapter builds that difference.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Design a CLI that follows the Unix philosophy: small, composable, pipe-friendly, text-based.
2. Implement robust argument parsing using libraries like `cxxopts` and `CLI11`, and know when to write your own.
3. Layer configuration sources — defaults, files, environment variables, flags — and apply them in the right order.
4. Implement proper stdin/stdout/stderr discipline: what goes where, and when to detect terminals.
5. Handle signals correctly: SIGINT, SIGTERM, and SIGPIPE, and clean up properly on exit.
6. Report errors clearly: what went wrong, what was tried, how to fix it — without dumping stack traces on end users.
7. Test CLI behavior: parsing, business logic, output formatting, and end-to-end behavior.
8. Recognize the tradeoffs between monolithic, layered, and framework-driven CLI architectures.

---

## 68.1 The Anatomy of a Good CLI

A CLI is a function from command line to exit code. The job is to:

1. **Parse arguments** — understand what the user asked.
2. **Read configuration** — gather defaults, env vars, files.
3. **Route to logic** — dispatch to the right operation.
4. **Execute** — do the work, stream output, handle errors.
5. **Exit** — return a code that scripts can read.

The format is almost always:

```
command [subcommand] [args] [--flags]
```

Examples:

```bash
git commit -m "message"
docker ps --filter status=running
curl https://example.com --output file.txt
```

### Exit Codes Are a Contract

The exit code is part of your tool's API. Scripts compose tools based on exit codes. The convention:

- **0** — success. The operation completed as intended.
- **1** — generic error. Something went wrong; the reason is in stderr.
- **2** — usage error. The user called the command incorrectly (e.g., missing required argument, invalid flag).
- **3+** — specific errors. Your tool defines what each code means.

Example:

```bash
$ wcx nonexistent.txt
wcx: cannot open 'nonexistent.txt': No such file or directory
$ echo $?
3
```

A script can react:

```bash
wcx "$file" > counts.txt
if [ $? -eq 3 ]; then
    echo "File not found, creating new one"
    touch "$file"
fi
```

**Never use exit code 1 for everything.** That discards information. When your tool returns distinct codes for distinct failures, scripts compose cleanly.

Some standard codes have conventions:

- **64** — command-line usage error (matches EX_USAGE from `<sysexits.h>`).
- **65** — data format error.
- **69** — unavailable service.
- **70** — internal software error.
- **74** — I/O error.
- **130** — program terminated by Ctrl+C (SIGINT).
- **143** — program terminated by SIGTERM.

Choose codes that are meaningful for your tool's failure modes. Document them in your help or man page so scripts can rely on them.

### Streams Have Distinct Purposes

- **stdout** — the result, the output the user asked for. If the operation succeeds, this is what the user wanted.
- **stderr** — diagnostics: progress, warnings, errors, debug output. Intended for humans (or logs).
- **stdin** — input, often piped from another tool.

Example:

```bash
# stdout is the result; stderr is progress
$ wcx large-file.txt 2>/dev/null
1000 2500 500000 large-file.txt
```

Most engineers mix these. A tool that writes errors to stdout is broken:

```cpp
// BAD: Errors go to stdout
std::cout << "error: file not found\n";
std::cout << "1000 2500 500000 file.txt\n";
```

A script piping the output will get corrupt data:

```bash
wcx file.txt | awk '{print $1}'  # Includes error line; now broken
```

**The rule:** if a user redirects stdout to a file, the file should contain only the tool's result, with no diagnostic output. All diagnostics go to stderr.

### Pipe-Friendly Output By Default

Unix tools compose. A tool that only produces pretty output is not composable:

```bash
$ wcx *.txt | awk '{total += $1} END {print total}'
# Fails if wcx colors the output or adds blank lines
```

The convention: **output plain text by default; add pretty formatting (colors, alignment) only when connected to a terminal.**

Detect a terminal with `isatty()`:

```cpp
#include <unistd.h>

bool isTerminal(int fd) {
    return isatty(fd) != 0;
}

// Usage
if (isTerminal(STDOUT_FILENO)) {
    // Output to terminal: add colors, alignment
    std::cout << "\033[32msuccess\033[0m\n";
} else {
    // Output to pipe or file: plain text
    std::cout << "success\n";
}
```

Also honor the `NO_COLOR` environment variable:

```cpp
bool shouldUseColor() {
    return isTerminal(STDOUT_FILENO) && !std::getenv("NO_COLOR");
}
```

---

## 68.2 Argument Parsing

Argument parsing is the boundary between the outside world and your program. Do it wrong, and users either can't use the tool or fall into traps.

### The Anatomy of Arguments

```
command subcommand [args] [--flags] [--flag=value] [-s] [-s=value]
```

- **Positional arguments** — arguments without a dash. Order matters. Examples: filenames, commands.
- **Long flags** — `--flag`, `--flag=value`. Readable, used for clarity.
- **Short flags** — `-f`, `-f value`. Compact, used for speed.
- **Flag terminator** — `--`. Everything after this is treated as a positional argument, even if it starts with `-`. Example: `wcx -- -weirdly-named-file.txt`.

### Common Parsing Mistakes

**Mistake 1: Not supporting `--`**

```bash
wcx -- -file.txt  # Should work: treat "-file.txt" as positional argument
```

If your parser doesn't recognize `--`, users with unusual filenames are stuck.

**Mistake 2: Ambiguous short flags**

```bash
tool -abc
# Does this mean: -a -b -c, or -ab c, or -abc (a single flag)?
```

Establish a convention. Most Unix tools allow `-abc` to mean `-a -b -c`. But be explicit in help text.

**Mistake 3: Not validating required arguments**

```cpp
// BAD: Lets the program run with missing required argument
if (argc < 2) {
    // Silently does nothing or crashes later
}

// GOOD: Explicit validation
if (argc < 2) {
    std::cerr << "usage: " << argv[0] << " <input_file>\n";
    return 2;  // Exit code 2 = usage error
}
```

**Mistake 4: Inconsistent behavior across invocations**

```bash
tool --output file.txt --input input.txt    # Works
tool --input input.txt --output file.txt    # Different behavior? Broken!
```

Flags should be order-independent unless there's a reason they shouldn't be.

### Using a Library: cxxopts and CLI11

Most C++ projects use a library. Two popular choices:

**cxxopts** — minimal, single-header library. Useful for simple tools with flags and options:

```cpp
#include "cxxopts.hpp"

cxxopts::Options options("wcx", "count words, lines, bytes");

options.add_options()
    ("h,help", "Show this help message")
    ("i,input", "Input file", cxxopts::value<std::string>())
    ("o,output", "Output file", cxxopts::value<std::string>()->default_value(""))
    ("l,lines", "Count lines only", cxxopts::value<bool>()->default_value("false"));

auto result = options.parse(argc, argv);

if (result.count("help")) {
    std::cout << options.help() << "\n";
    return 0;
}

std::string input = result["input"].as<std::string>();
bool countLines = result["lines"].as<bool>();
```

cxxopts handles basic validation, type conversion, and help generation. It stays out of your way for tools that don't need subcommands.

**CLI11** — more feature-rich, better for complex tools and subcommands:

```cpp
#include "CLI/CLI.hpp"

CLI::App app("A CLI tool");

std::string input;
std::string output;
bool verbose = false;

app.add_option("--input,-i", input, "Input file")->required();
app.add_flag("--verbose,-v", verbose, "Verbose output");
app.add_option("--output,-o", output, "Output file");

CLI11_PARSE(app, argc, argv);

// Use input, output, verbose...
```

CLI11 adds features like option groups, constraint validation, and better error messages. For subcommands (like `git commit`, `git push`):

```cpp
CLI::App app("git clone of our tool");

// Subcommand: commit
auto commit = app.add_subcommand("commit", "Commit changes");
std::string message;
commit->add_option("-m", message, "Commit message")->required();
commit->callback([&]() {
    // Handle commit
});

// Subcommand: push
auto push = app.add_subcommand("push", "Push changes");
push->callback([&]() {
    // Handle push
});

CLI11_PARSE(app, argc, argv);
```

CLI11 automatically generates help for subcommands and enforces that exactly one is selected. This is the right choice for tools with multiple operations.

### When to Write Your Own Parser

A custom parser is rarely needed. Libraries handle edge cases (nested subcommands, flag parsing across `--`, type conversion). **Use a library unless your tool is trivial** (single command, few flags).

When custom parsing might be justified:

- Your tool is a shell-like script with custom syntax.
- You need to preserve the exact format of arguments (rare).
- You're building an embedded tool where dependencies are forbidden.

Even then, consider a library first.

---

## 68.3 Configuration Layering

A good CLI accepts configuration from multiple sources, in a predictable order:

```
1. Built-in defaults
2. Config file (~/.config/toolname/config)
3. Environment variables (TOOLNAME_*)
4. Command-line flags
```

Later layers override earlier layers. This is sometimes called "last writer wins." This order is intuitive: people expect command-line flags to override everything, because they are the most explicit.

Example:

```cpp
struct Config {
    std::string input;
    std::string output;
    int threads = 1;  // Built-in default
    bool verbose = false;
};

Config config;

// Layer 1: Built-in defaults (already set above)

// Layer 2: Load from file
std::string configFile = std::getenv("HOME") + "/.config/wcx/config";
if (std::filesystem::exists(configFile)) {
    loadConfigFile(configFile, &config);
}

// Layer 3: Environment variables
if (const char* threads = std::getenv("WCX_THREADS")) {
    config.threads = std::stoi(threads);
}

// Layer 4: Command-line flags (parsed last, override everything)
if (result.count("threads")) {
    config.threads = result["threads"].as<int>();
}

// Now use config...
```

This layering is powerful because it respects the user's intent at each level: defaults are conservative, files are declarative, environment variables are scripted, flags are interactive and explicit.

### Config File Format

Use a standard format: TOML, YAML, or INI. Don't invent your own.

TOML is simple and unambiguous:

```toml
# ~/.config/wcx/config
input = "/tmp/input.txt"
threads = 4
verbose = false
```

Load it with a library like `toml++`:

```cpp
#include "toml.hpp"

void loadConfigFile(const std::string& path, Config* out) {
    auto tbl = toml::parse_file(path);
    
    if (auto v = tbl["input"]) {
        out->input = v.value_or(out->input);
    }
    if (auto v = tbl["threads"]) {
        out->threads = v.value_or(out->threads);
    }
    if (auto v = tbl["verbose"]) {
        out->verbose = v.value_or(out->verbose);
    }
}
```

### XDG Base Directories

Most Unix tools follow the XDG Base Directory specification. Use `$XDG_CONFIG_HOME` for config:

```cpp
std::string getConfigDir() {
    if (const char* xdg = std::getenv("XDG_CONFIG_HOME")) {
        return xdg;
    }
    return std::string(std::getenv("HOME")) + "/.config";
}

std::string configPath = getConfigDir() + "/wcx/config";
```

This way, users can customize where config goes.

---

## 68.4 Stdin / Stdout Discipline

Many CLIs read from stdin when no file is specified. This makes them composable:

```bash
cat input.txt | wcx
wcx < input.txt
wcx           # Reads from stdin
wcx file.txt  # Reads from file
```

The Unix convention: if the filename is `-` or missing, read from stdin. This is so fundamental that it's expected by tools that don't even explicitly document it.

### Implementing Stdin Reading

```cpp
#include <iostream>
#include <sstream>

std::string readInput(const std::string& filename = "") {
    std::string content;
    
    if (filename.empty() || filename == "-") {
        // Read from stdin
        std::string line;
        while (std::getline(std::cin, line)) {
            content += line + "\n";
        }
    } else {
        // Read from file
        std::ifstream file(filename);
        if (!file) {
            std::cerr << "error: cannot open '" << filename << "'\n";
            return "";
        }
        content = std::string((std::istreambuf_iterator<char>(file)),
                               std::istreambuf_iterator<char>());
    }
    
    return content;
}
```

### Streaming vs. Loading

Never load entire files into memory if you can stream. This is not premature optimization; it is correctness:

```cpp
// BAD: Loads entire file into memory
std::string content = readFile(filename);
process(content);
// Out-of-memory if file is too large

// GOOD: Streams line by line
std::string line;
while (std::getline(std::cin, line)) {
    processLine(line);
}
// Works for files of any size
```

Streaming means your tool handles gigabyte-sized files without blowing memory. It also means interactive use is possible — the tool can output results incrementally instead of waiting for all input.

### Buffering

When output is piped, the stdio buffer is often larger (4KB) and line-buffering is disabled. This can delay output if the buffer fills slowly, breaking interactive behavior:

```cpp
// Ensure output reaches the pipe promptly
std::cout << result << std::flush;
```

Or disable buffering entirely:

```cpp
std::cout.setf(std::ios::unitbuf);  // Unbuffered; slower but immediate
```

Use flushing sparingly — it's a performance cost. Only flush when necessary (after a line that should appear immediately, or when waiting for user input).

### Color and Formatting

Only colorize when writing to a terminal. Most people don't want ANSI color codes in redirected output:

```cpp
if (isTerminal(STDOUT_FILENO)) {
    std::cout << "\033[32m✓ success\033[0m\n";
} else {
    std::cout << "success\n";
}
```

Also honor the `NO_COLOR` environment variable — some users have visual processing differences and hate colored output even on terminals:

```cpp
bool shouldColor() {
    return isTerminal(STDOUT_FILENO) && !std::getenv("NO_COLOR");
}
```

---

## 68.5 Signals and Cleanup

Your CLI runs in an environment where signals arrive at any time. Handle them correctly.

### SIGINT (Ctrl+C)

When the user presses Ctrl+C, the OS sends SIGINT. Your tool should:

1. Stop what it's doing.
2. Clean up (flush buffers, close files).
3. Exit cleanly with a specific code.

```cpp
#include <signal.h>
#include <atomic>

static std::atomic<bool> interrupted{false};

void sigintHandler(int sig) {
    interrupted = true;
    // Note: Only async-signal-safe operations here!
    // Don't call std::cout, malloc, etc.
}

int main(int argc, char** argv) {
    signal(SIGINT, sigintHandler);
    
    for (int i = 0; i < 1000000; ++i) {
        if (interrupted) {
            std::cerr << "\nInterrupted.\n";
            return 130;  // Standard exit code for SIGINT (128 + 2)
        }
        
        doWork();
    }
    
    return 0;
}
```

### SIGTERM (Graceful Shutdown)

Unlike SIGINT (from user), SIGTERM is sent by the OS or a process manager asking for graceful shutdown. Handle it similarly:

```cpp
static bool termRequested = false;

void sigtermHandler(int sig) {
    termRequested = true;
}

signal(SIGTERM, sigtermHandler);

// In main loop
if (termRequested) {
    cleanup();
    return 143;  // 128 + 15
}
```

### SIGPIPE (Downstream Closed)

When a pipe reader closes (e.g., `wcx | head -n 1`), the writer gets SIGPIPE. By default, this terminates the process. Often that's fine — your tool exits when nobody's listening. But you can handle it explicitly:

```cpp
signal(SIGPIPE, SIG_IGN);  // Ignore SIGPIPE; let write() return -1 instead

// Or:
signal(SIGPIPE, [](int) { exit(EXIT_SUCCESS); });  // Clean exit
```

### Cleanup on Exit

Use `atexit()` or RAII to ensure cleanup happens:

```cpp
void cleanup() {
    // Flush buffers
    std::cout.flush();
    std::cerr.flush();
    
    // Close files
    if (outputFile.is_open()) {
        outputFile.close();
    }
}

int main() {
    std::atexit(cleanup);
    
    // Work...
}
```

Or use a RAII guard:

```cpp
class Cleanup {
public:
    ~Cleanup() {
        // Flush, close, etc.
    }
};

int main() {
    Cleanup guard;
    // Work; cleanup happens when guard is destroyed
}
```

---

## 68.6 Error Reporting

Users hate cryptic error messages. Your job is to make errors actionable. Engineers writing scripts hate unclear errors because they have to debug why the automation broke.

### The Three Parts of an Error Message

1. **What went wrong** — a one-line summary.
2. **What was tried** — the context (which file, which operation).
3. **How to fix it** — a suggestion, if possible.

Example — bad:

```
Error: std::filesystem::filesystem_error: operation not permitted
Backtrace:
  ... 20 lines of stack trace ...
```

Example — good:

```
error: cannot read '/root/.private': Permission denied
hint: check file permissions or run with appropriate privileges
```

The bad example tells a developer what went wrong in the code; the good example tells a user what went wrong in their task and how to fix it.

Implementation:

```cpp
void reportError(const std::string& operation, const std::string& target, const std::string& reason) {
    std::cerr << "error: " << operation << " '" << target << "': " << reason << "\n";
}

// Usage
if (!std::filesystem::exists(inputFile)) {
    reportError("read", inputFile, "No such file or directory");
    return 3;
}
```

Consider context in error messages. A tool that processes files should identify which file failed:

```cpp
// BAD: Vague
std::cerr << "error: parsing failed\n";

// GOOD: Specific
std::cerr << "error: parsing failed in '" << filename << "' at line " 
          << lineNumber << "\n";
```

### When to Show Verbose Output

Reserve `--verbose` for details users don't need by default:

```cpp
if (verbose) {
    std::cerr << "[DEBUG] Loading config from " << configPath << "\n";
    std::cerr << "[DEBUG] Parsed " << lineCount << " lines\n";
}
```

Verbose output should help debugging, not replace clear error messages. Even with `--verbose`, error messages should be human-readable.

**Never dump a stack trace to the user.** Log it for debugging, but don't print it to stderr. Most users don't know how to read a stack trace:

```cpp
// BAD
try {
    doWork();
} catch (const std::exception& e) {
    std::cerr << e.what() << "\n";
    throw;  // Stack trace goes to console
}

// GOOD
try {
    doWork();
} catch (const std::exception& e) {
    std::cerr << "error: " << humanReadableError(e) << "\n";
    if (verbose) {
        std::cerr << "[DEBUG] Internal error: " << e.what() << "\n";
    }
    return 1;
}
```

The verbose flag is for developers debugging the tool, not for end users trying to use it.

---

## 68.7 Testability

A well-layered CLI is testable. Pull I/O behind interfaces.

### The Layering

```
┌─────────────────────────────────┐
│   CLI Interface (main)          │
│   Parses args, handles I/O      │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│   Business Logic                │
│   (no knowledge of stdin/stdout)│
└─────────────────────────────────┘
```

Example:

```cpp
// Business logic (testable, no I/O)
class WordCounter {
public:
    struct Result {
        int lines = 0;
        int words = 0;
        int bytes = 0;
    };
    
    Result count(const std::string& content) const {
        Result r;
        // Count lines, words, bytes
        return r;
    }
};

// Tests
TEST(WordCounter, CountsEmpty) {
    WordCounter counter;
    auto result = counter.count("");
    EXPECT_EQ(result.lines, 0);
    EXPECT_EQ(result.words, 0);
}

// CLI layer
int main(int argc, char** argv) {
    std::string input = readInput(inputFile);
    
    WordCounter counter;
    auto result = counter.count(input);
    
    std::cout << result.lines << " " << result.words << " " << result.bytes << "\n";
}
```

### Integration Tests

Test the full CLI with subprocess calls:

```cpp
TEST(CliIntegration, ProcessesFile) {
    std::system("echo 'hello world' > /tmp/test.txt");
    
    int exitCode = std::system("wcx /tmp/test.txt > /tmp/output.txt");
    EXPECT_EQ(exitCode, 0);
    
    std::string output = readFile("/tmp/output.txt");
    EXPECT_THAT(output, HasSubstr("1 2 12"));
}
```

### Snapshot Tests

For output-heavy tools, snapshot tests prevent regressions:

```cpp
TEST(CliOutput, MatchesBaseline) {
    std::string output = runCommand("wcx test.txt");
    
    std::string baseline = readFile("testdata/baseline.txt");
    EXPECT_EQ(output, baseline);
}
```

If output changes (intentionally), update the snapshot.

---

## 68.8 Worked Example: wcx — Word/Line/Byte Counter

Let's build a simple `wcx` tool in C++20. It counts words, lines, and bytes — like Unix `wc`, but simpler and more extensible.

### The Domain Layer

```cpp
// counter.h
#pragma once
#include <string>

class WordCounter {
public:
    struct Result {
        int lines = 0;
        int words = 0;
        int bytes = 0;
    };
    
    // Count the given content
    Result count(const std::string& content) const;
};
```

```cpp
// counter.cpp
#include "counter.h"
#include <sstream>

WordCounter::Result WordCounter::count(const std::string& content) const {
    Result r;
    
    // Bytes
    r.bytes = content.size();
    
    // Lines (count newlines)
    for (char c : content) {
        if (c == '\n') r.lines++;
    }
    
    // Words (count whitespace-separated tokens)
    std::istringstream iss(content);
    std::string word;
    while (iss >> word) {
        r.words++;
    }
    
    return r;
}
```

### The CLI Layer

```cpp
// main.cpp
#include "counter.h"
#include <iostream>
#include <fstream>
#include <filesystem>
#include <cstring>
#include <cstdlib>
#include <signal.h>
#include <unistd.h>
#include <atomic>

static std::atomic<bool> interrupted{false};

void sigintHandler(int sig) {
    interrupted = true;
}

bool isTerminal(int fd) {
    return isatty(fd) != 0;
}

std::string readInput(const std::string& filename) {
    if (filename.empty() || filename == "-") {
        // Read from stdin
        return std::string((std::istreambuf_iterator<char>(std::cin)),
                           std::istreambuf_iterator<char>());
    }
    
    // Read from file
    std::ifstream file(filename);
    if (!file) {
        std::cerr << "wcx: cannot open '" << filename << "': "
                  << std::strerror(errno) << "\n";
        return "";
    }
    
    return std::string((std::istreambuf_iterator<char>(file)),
                       std::istreambuf_iterator<char>());
}

void showHelp(const char* progName) {
    std::cout << "usage: " << progName << " [--lines | --words | --bytes] [file]\n"
              << "\n"
              << "Count lines, words, and bytes in input.\n"
              << "\n"
              << "Options:\n"
              << "  --lines, -l    Count lines only\n"
              << "  --words, -w    Count words only\n"
              << "  --bytes, -b    Count bytes only\n"
              << "  --help, -h     Show this help\n";
}

int main(int argc, char** argv) {
    signal(SIGINT, sigintHandler);
    
    // Parse arguments (simple version, without a library)
    bool countLines = false, countWords = false, countBytes = false;
    std::string inputFile;
    
    for (int i = 1; i < argc; ++i) {
        std::string arg = argv[i];
        
        if (arg == "--help" || arg == "-h") {
            showHelp(argv[0]);
            return 0;
        } else if (arg == "--lines" || arg == "-l") {
            countLines = true;
        } else if (arg == "--words" || arg == "-w") {
            countWords = true;
        } else if (arg == "--bytes" || arg == "-b") {
            countBytes = true;
        } else if (arg == "--") {
            // Rest are positional
            if (i + 1 < argc) inputFile = argv[i + 1];
            break;
        } else if (arg[0] == '-') {
            std::cerr << "wcx: unknown option: " << arg << "\n";
            return 2;
        } else {
            inputFile = arg;
        }
    }
    
    // Default: show all
    if (!countLines && !countWords && !countBytes) {
        countLines = countWords = countBytes = true;
    }
    
    // Read input
    std::string content = readInput(inputFile);
    if (content.empty() && !inputFile.empty()) {
        // readInput already printed the error
        return 3;
    }
    
    if (interrupted) {
        std::cerr << "\nInterrupted.\n";
        return 130;
    }
    
    // Count
    WordCounter counter;
    auto result = counter.count(content);
    
    // Output
    bool first = true;
    if (countLines) {
        if (!first) std::cout << " ";
        std::cout << result.lines;
        first = false;
    }
    if (countWords) {
        if (!first) std::cout << " ";
        std::cout << result.words;
        first = false;
    }
    if (countBytes) {
        if (!first) std::cout << " ";
        std::cout << result.bytes;
        first = false;
    }
    
    // Append filename if provided
    if (!inputFile.empty() && inputFile != "-") {
        std::cout << " " << inputFile;
    }
    
    std::cout << "\n";
    
    return 0;
}
```

### Usage

```bash
$ echo "hello world" | wcx
1 2 11

$ wcx --lines /etc/passwd
42

$ wcx < /etc/passwd
42 83 2100

$ wcx /etc/passwd
42 83 2100 /etc/passwd
```

---

## 68.9 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Monolithic `main`** | Simple, direct, no overhead | All logic in one function; hard to test | Scripts, prototypes, <100 lines |
| **Layered** (parse → logic → format) | Testable, reusable logic, clean separation | More files, more abstraction | Most production CLIs |
| **Framework-driven** (Cobra, CLI11) | Handles subcommands, help, validation automatically | Dependency added; less control; can feel over-engineered for simple tools | Complex CLIs with many subcommands |
| **Unix-style** (stdio pipes, exit codes) | Composable, simple contract | User must understand exit codes; error messages harder to format | Systems tools, filters |
| **Interactive CLI** (readline, prompts) | Better UX for complex tools | Harder to automate, test; heavier dependencies | Interactive tools (shells, databases) |

---

## 68.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Exit code 1 is fine for any error." | Exit codes are part of your tool's API. Distinct codes let scripts handle different failures differently. Use them. |
| "Printing errors to stdout is acceptable." | stdout is the contract with the caller. Errors to stdout corrupt the data the user asked for. Always use stderr for diagnostics. |
| "Adding color is always good." | Color breaks pipes and scripts. Check if you're writing to a terminal; skip color if you're not. |
| "My tool should work exactly like `tool --help`." | Only if your tool is a clone of that tool. Otherwise, follow conventions but be consistent with yourself. |
| "Configuration files are nice-to-have." | For tools users run repeatedly, config files are essential. They reduce flag repetition and make intent clear. |
| "Signals are the OS's problem." | Your tool runs in an environment with signals arriving at any time. Handle SIGINT, SIGTERM, SIGPIPE correctly or your tool will leave garbage behind. |
| "Streaming is premature optimization." | Streaming is not optimization; it's necessary for large inputs. A tool that loads entire files into memory is broken. |

---

## 68.11 Exercises

1. **Design a CLI for a task you do regularly.** Write a one-page specification: what it does, what arguments it takes, what it outputs, what exit codes it returns. Show a few usage examples.

2. **Implement the layering.** Build a simple tool (20 lines of logic). Separate the business logic from the CLI layer. Write three tests for the logic; write one integration test for the full tool.

3. **Signal handling.** Write a tool that counts down from 100, printing each number. Add signal handling so Ctrl+C stops cleanly. Test it: does the output buffer flush? Does the exit code reflect the signal?

4. **Configuration layering.** Create a tool that accepts a config file and environment variables. Load them in order (defaults, file, env, flags). Verify that later layers override earlier ones.

5. **Pipe-friendly output.** Build a tool that outputs formatted results (aligned columns, colors). Add terminal detection: colorize only when connected to a terminal. Test: `tool | cat` should produce plain text.

6. **Error messages.** Find a tool you use whose error messages you dislike. Redesign the error messages: include what went wrong, what was tried, and how to fix it. Would a user understand it?

---

## 68.12 Summary

A CLI is the most direct application of everything in Parts 1–6: parsing and buffering (Part 2), abstractions and layering (Parts 3–4), signals and I/O (Part 5), and design decisions about the tool's interface. A good CLI is pipe-friendly, composable, and clear about its contract (exit codes, output format, error messages).

The key techniques:

- **Layer your code** — separate parsing, business logic, and output formatting so each is independently testable.
- **Respect the streams** — stdout is the result; stderr is diagnostics; stdin is input.
- **Use distinct exit codes** — let scripts compose based on success/failure granularity.
- **Handle signals** — SIGINT, SIGTERM, SIGPIPE arrive at any time; be ready.
- **Detect terminals** — colorize output only when writing to a terminal; honor NO_COLOR.
- **Make configuration layerable** — defaults, files, environment, flags, in that order.
- **Report errors clearly** — what went wrong, what was tried, how to fix it.

A tool that does these things well becomes part of workflows, gets extended, gets maintained. A tool that doesn't becomes a footnote on an internal wiki, replaced the moment a better alternative appears.

---

**[← Previous: Part 6 — Language Design Tradeoffs](../part-06-language-design-tradeoffs/10-language-design-tradeoffs.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: Designing a Web Server →](02-designing-a-web-server.md)**
