# Chapter 75 — Hot Reloading

Edit-save-see-result-in-100ms feels like magic the first time you experience it. A web developer changes a React component, hits save, and the browser updates in place, preserving the application state. A game developer edits shader code, recompiles it, and the next frame renders with the new effect without restarting the engine. A systems engineer patches a bug in a library without bringing down the production service.

Behind the magic is a careful choreography: file watchers that detect changes, module systems that track dependencies, state preservation mechanisms that keep the application alive while code underneath is swapped out, and error handling for the cases where that swap fails. This chapter walks through what each runtime does to make hot reload work — and more importantly, where it breaks.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Distinguish **hot reload** from **auto-restart**: hot reload replaces code while preserving state; auto-restart loses everything.
2. Explain the **module graph** and how it enables selective code reloading in JavaScript, Python, and other dynamic environments.
3. Implement a **file watcher** using OS primitives (`inotify`, `FSEvents`, `ReadDirectoryChangesW`) and understand the debouncing problem.
4. Build a **C++ hot reload harness** using dynamic loading (`dlopen`/`dlclose`) and know the constraints it imposes.
5. Identify **state preservation strategies**: serialization round-trips, immutable blessed structures, and the costs of each.
6. Recognize the **failure modes**: type-layout changes, captured closures, thread-local state, and static initialization order.
7. Distinguish **hot reload** from **live coding** (REPL-driven development) and know which tool fits which problem.

---

## What "Hot Reload" Actually Means

**Hot reload**: Replace running code without restarting the process. The application continues to execute, its state (open connections, in-memory caches, UI state, threads) is preserved, and the new code is immediately available for the next function call.

This is different from **auto-restart** (also called "auto-reload" or "reload on change"), which simply re-runs the program from the beginning when source changes. Auto-restart loses state; hot reload keeps it.

```
Auto-restart:          Hot reload:
-----------            -----------
1. Detect change       1. Detect change
2. Kill process        2. Compile new code
3. Recompile           3. Swap in-memory
4. Start new process   4. Continue execution
   (state lost)           (state preserved)
```

In practice, hot reload is useful when:

- **State is expensive to recreate.** A web application with open database connections, cached data, in-flight requests, or user sessions loses efficiency with auto-restart.
- **Iteration speed matters.** A game developer or UI designer wants to see feedback in milliseconds, not seconds.
- **Downtime is unacceptable.** A production service cannot afford to restart.

Hot reload is *not* useful when:

- **State changes are part of the bug fix.** If you're fixing a data structure layout, the old in-memory representation is now wrong; hot reload cannot safely run without migration.
- **The reload is more expensive than the restart.** If compilation takes 30 seconds and reload infrastructure takes 10 seconds, just restarting is faster.

---

## Approaches By Runtime

### JavaScript: Webpack/Vite HMR (Hot Module Replacement)

Webpack and Vite treat the application as a **module graph**: a directed acyclic graph (DAG) where each module is a node, and imports are edges. When a file changes, the bundler:

1. **Identifies the changed module** — e.g., `MyButton.jsx`.
2. **Re-parses and re-evaluates** only that module and modules that depend on it.
3. **Replaces the module in the runtime** — all other modules that imported it get the new version on their next function call.
4. **Re-runs the module's side effects** — imports, initializers, but not the entire application.

```javascript
// Module A: util.js
export function double(x) { return x * 2; }

// Module B: display.js
import { double } from "./util.js";
export function show(x) {
    console.log(double(x));
}

// Module C: main.js
import { show } from "./display.js";
show(5);  // Prints: 10
```

**Scenario 1: Edit `util.js` to `return x * 3`**

The bundler re-evaluates `util.js`, updates the module cache, and the next call to `show(5)` gets the new result (15) without restarting.

**Scenario 2: Edit `display.js` to add logging**

The bundler re-evaluates `display.js` and `main.js` (which depends on it). The previous call to `show(5)` is finished; the next one uses the new code.

#### React/Vue Integration

React and Vue provide **HMR plugins** that go further: they replace components without losing state.

```javascript
// Counter.jsx
import { useState } from "react";

export function Counter() {
    const [count, setCount] = useState(0);
    return (
        <div>
            {count}
            <button onClick={() => setCount(count + 1)}>Inc</button>
        </div>
    );
}
```

If you edit the button text, Webpack/Vite's HMR plugin:

1. Detects the component changed.
2. Re-evaluates `Counter.jsx`.
3. **Does not unmount the component.** (That would reset `count` to 0.)
4. Instead, **feeds the new component function** into React's reconciliation.
5. The DOM is updated with the new text; `count` remains as it was.

This is *not* free; it requires the framework to participate. A custom component or hook must be designed to survive re-evaluation (i.e., avoid generating new closures that capture stale state). A bare `export const MyVar = computeExpensiveValue()` in a re-evaluated module gets re-run and re-assigned — often exactly what you want, but sometimes a foot-gun.

**Limitations:**

- **Type changes.** If you change a component's prop interface, old code calling it with the old signature breaks until you reload those callers too.
- **Closure capture.** A closure capturing a module-level variable sees the old value until the module is re-evaluated again.
- **Third-party libraries.** Not all libraries are HMR-aware; some cache things or set up subscriptions that break if the code is swapped underneath.

### Erlang/Elixir: The Code Server

Erlang's **code server** is a supervisor process that manages which version of each module is loaded. Multiple versions of a module can be in memory simultaneously.

When a new version of a module is compiled and loaded:

1. The code server loads it as a new version (e.g., version N+1).
2. **All existing processes continue running on the old version (N).** The code they are executing is not invalidated.
3. **New processes use the new version.**
4. Old processes can explicitly call `code:get_module_info()` to discover a newer version and migrate if they choose.

The module can define a **`code_change/3` callback** that handles state migration:

```erlang
%% old version
-record(counter, {value = 0}).
counter(State = #counter{value = V}) ->
    receive
        inc -> counter(State#counter{value = V + 1});
        {code_change, _} -> counter_v2(State)
    end.

%% new version with extra field
-record(counter_v2, {value = 0, max = 100}).
counter_v2(State = #counter{value = V}) ->
    S2 = #counter_v2{value = V, max = 100},
    counter_v2(S2).

counter_v2(State = #counter_v2{value = V, max = Max}) ->
    receive
        inc -> 
            NewV = min(V + 1, Max),
            counter_v2(State#counter_v2{value = NewV});
        {code_change, _} -> counter_v2(State)
    end.
```

**Advantages:**

- **No forced state loss.** Old processes keep running; they migrate when ready.
- **Explicit state transformation.** The `code_change` callback is a deliberate transformation, not a guess.
- **Predictable behavior.** If a process hits undefined behavior (e.g., calling a function that no longer exists in the old module), the error is local, not cascading.

**Disadvantages:**

- **Multiple versions in memory.** Code duplication; version skew can arise if you have three versions loaded.
- **Limited to message passing.** Works well for actor-based systems; less well for shared mutable state (which Erlang avoids anyway).
- **Manual migration.** The programmer must write `code_change` and handle all migration logic.

### JVM: HotSpot (Limited in Standard JDK)

The HotSpot JVM can swap **method bodies** at runtime using instrumentation APIs. Certain IDEs and tools (notably JRebel) use this extensively for development-time hot reload.

Standard JDK limitations:

- **Cannot add/remove fields** to a class. The memory layout of instances is baked in.
- **Cannot add/remove methods.** Only existing methods can have their bytecode replaced.
- **Requires `-XX:+AllowEnhancedClassRedefinition`** and instrumentation agent.

```java
// Original
public class Service {
    public String greet(String name) {
        return "Hello, " + name + "!";
    }
}

// Edit the source: greet() now returns "Hi, " + name
// The JVM can swap the bytecode of greet() in place.
// Old code running greet() will use the new bytecode on next call.

// But this will NOT work:
public class Service {
    private int newField = 0;  // REJECTED: field added
    public String greet(String name) { ... }
}
```

JRebel and similar tools work around these restrictions using bytecode manipulation and a custom classloader, but it is fragile and tool-specific.

### Game Engines: Dynamic Library Reload

Many game engines (Unreal, custom C++ engines) implement hot reload by:

1. **Separating the engine core (stable) from game code (volatile).**
2. **Building game code as a dynamic library (`.dll` on Windows, `.so` on Linux).**
3. **The engine loads the library at startup and again after each recompile.**

On file change:

1. Compiler builds game code into a new library version.
2. Engine detects the new library.
3. Engine calls a **shutdown hook** in the old library to serialize state.
4. Engine calls `dlclose()` on the old library and `dlopen()` on the new one.
5. Engine calls an **init hook** in the new library to deserialize state.
6. Game continues executing.

```cpp
// Engine (stable)
typedef void (*OnUnload)(SerializedState* state);
typedef void (*OnLoad)(const SerializedState* state);

struct GameLibrary {
    void* handle;
    OnUnload onUnload;
    OnLoad onLoad;
};

void reloadGameLibrary(GameLibrary* lib) {
    SerializedState state;
    if (lib->onUnload) lib->onUnload(&state);
    
    if (lib->handle) dlclose(lib->handle);
    
    lib->handle = dlopen("./game.so", RTLD_NOW);
    lib->onLoad = (OnLoad)dlsym(lib->handle, "onLoad");
    lib->onUnload = (OnUnload)dlsym(lib->handle, "onUnload");
    
    if (lib->onLoad) lib->onLoad(&state);
}

// Game code (hot-reloadable)
SerializedState gGameState;

extern "C" void onUnload(SerializedState* state) {
    *state = gGameState;  // Save all game state to the struct
}

extern "C" void onLoad(const SerializedState* state) {
    gGameState = *state;  // Restore
    reinitializeStateThatCannotBeSerialized();
}

int main() {
    // ... initialize game ...
    while (running) {
        // Check for game.so modification
        if (gameLibraryChanged()) {
            reloadGameLibrary(&gameLib);
        }
        // ... game loop ...
    }
}
```

**Advantages:**

- **Full control.** Swap any code, add any fields, change any layout.
- **Explicit state handling.** The hooks make serialization visible.

**Disadvantages:**

- **Requires architecture discipline.** Game code must be separated from the engine.
- **Serialization overhead.** Every reload must save and restore state.
- **Thread safety.** If the game loop is multi-threaded, reloading while threads are running is unsafe (more on this below).

---

## What Breaks Hot Reload

### 1. Static Type Layout Changes

If you add a field to a struct and hot-reload code that creates instances of the old struct, the in-memory layout is wrong.

```cpp
// Old code running in memory
struct Point {
    int x, y;  // sizeof(Point) = 8 bytes
};
Point origin;  // Points to 8 bytes of memory

// You edit the source
struct Point {
    int x, y, z;  // sizeof(Point) = 12 bytes
};

// New code is compiled and loaded
// But 'origin' still points to 8 bytes; accessing z reads garbage or a neighbor's field.
```

**In JVM:** Adding a field is rejected by the JVM.
**In C++ with dlopen:** The new library has the new layout; the old instance is corrupted.
**In Erlang:** Old processes continue with the old record layout; new processes use the new one. An explicit `code_change` transforms instances.

### 2. Captured Closures and Hidden State

Languages with closures (JavaScript, Python, Rust) capture values from outer scopes. If the outer scope is re-evaluated, closures referencing old captures become stale.

```javascript
// Old code
let counter = 0;

function increment() {
    return ++counter;  // Closure captures 'counter'
}

export { increment };

// Hot reload: the module is re-evaluated
let counter = 0;  // NEW counter, separate memory location

// The old increment() function still references the OLD counter.
// New code calling increment() increments the NEW counter.
// This is usually correct (you want fresh state), but not always.
```

A more subtle issue:

```javascript
// Old code
const handlers = [];

function setup() {
    const localData = { count: 0 };
    handlers.push(() => localData.count++);  // Closure captures localData
}

export { setup, handlers };

// Hot reload occurs while the closure is running
// The handler is still alive, still referencing the old 'localData'
// New code cannot access or modify that state.
```

### 3. Thread-Local and Static Initialization Order

C++ has many implicit initializations:

- **Static variable initializers** run once, the first time the variable is accessed.
- **Thread-local initializers** run once per thread when the thread first accesses them.
- **Constructor side effects** (opening files, registering hooks) are often implicit.

When code is reloaded, these may or may not re-run, leading to inconsistency.

```cpp
// lib.cpp
static std::map<int, Handler> handlers;  // Initialized empty, once

void registerHandler(int id, Handler h) {
    handlers[id] = h;
}

// Hot reload: new lib.so is loaded
// The static 'handlers' in the NEW library is a fresh empty map.
// Handlers registered with the old library are lost.
```

### 4. Open File Handles and Resource Leaks

When a library is unloaded, OS resources it holds (file descriptors, memory from its allocator, thread handles) must be cleaned up. If the library has a `dlopen` but no matching resource cleanup, those resources leak.

```cpp
// lib.cpp (v1)
static FILE* logFile = fopen("app.log", "a");

void log(const char* msg) {
    fprintf(logFile, "%s\n", msg);
}

// Hot reload: onUnload() is called
// If onUnload() does NOT close logFile, the file descriptor leaks.
// The file is still open; the library is unloaded; no code can close it.
```

### 5. Virtual Function Tables and Inheritance

If a virtual function is defined in a base class that is NOT reloaded, but a derived class in the reloaded code overrides it, the virtual function table (vtable) is in an inconsistent state.

```cpp
// base.h (not reloaded; part of stable engine)
class Entity {
    virtual void update() = 0;
};

// game.cpp (reloaded)
class Player : public Entity {
    void update() override { /* ... */ }
};

// Scenario: Player instance was created with old code.
// Its vtable points to the old override.
// Hot reload loads new game.so with new Player class definition.
// The instance's vtable is stale; it still points to old code.
```

### 6. Module Dependencies and Recompilation

If a module B depends on module A, and both change, you must reload B after A for consistency. Get the order wrong and B references stale exports from A.

Many hot reload systems (Webpack, game engines) enforce a topological order or reload all dependents together.

---

## File Watching

Hot reload starts with **file watching**: detecting when source files change.

### OS Primitives

**Linux: `inotify`**

```cpp
#include <sys/inotify.h>

int wd = inotify_add_watch(ifd, "/path/to/file", IN_MODIFY | IN_CLOSE_WRITE);

// Later, in a loop:
char buf[4096];
int len = read(ifd, buf, sizeof(buf));
for (char* p = buf; p < buf + len; ) {
    struct inotify_event* e = (struct inotify_event*)p;
    if (e->mask & IN_CLOSE_WRITE) {
        printf("File modified: %s\n", e->name);
    }
    p += sizeof(*e) + e->len;
}
```

**macOS: `FSEvents`**

```cpp
#include <CoreServices/CoreServices.h>

FSEventStreamRef stream = FSEventStreamCreate(
    nullptr,
    callback,           // Called on changes
    nullptr,
    CFArrayCreate(..., "/path/to/dir", ...),
    kFSEventStreamEventIdSinceNow,
    1.0,  // Latency: coalesce changes within 1 second
    kFSEventStreamCreateFlagNoDefer | kFSEventStreamCreateFlagFileEvents
);

FSEventStreamScheduleWithRunLoop(stream, CFRunLoopGetCurrent(), kCFRunLoopDefaultMode);
FSEventStreamStart(stream);
```

**Windows: `ReadDirectoryChangesW`**

```cpp
#include <windows.h>

HANDLE hDir = CreateFileA(
    "C:\\path\\to\\dir",
    FILE_LIST_DIRECTORY,
    FILE_SHARE_READ | FILE_SHARE_WRITE,
    nullptr,
    OPEN_EXISTING,
    FILE_FLAG_BACKUP_SEMANTICS,
    nullptr
);

char buf[4096];
DWORD bytesReturned;
while (ReadDirectoryChangesW(
    hDir, buf, sizeof(buf), TRUE,
    FILE_NOTIFY_CHANGE_LAST_WRITE,
    &bytesReturned, nullptr, nullptr
)) {
    FILE_NOTIFY_INFORMATION* info = (FILE_NOTIFY_INFORMATION*)buf;
    printf("Changed: %.*ls\n", info->FileNameLength / 2, info->FileName);
    info = (FILE_NOTIFY_INFORMATION*)((char*)info + info->NextEntryOffset);
}
```

### Cross-Platform Libraries

Most developers use a library:

- **chokidar** (Node.js) — watches files, debounces, handles OS quirks.
- **watchman** (Facebook) — scalable file watcher for large codebases, caches state.
- **Boost.Asio + Boost.Filesystem** (C++) — portable but not as polished.

### Debouncing

A single file edit can trigger multiple `inotify` events (one for the write, one for `fsync`, etc.). Also, editors often write the file in chunks (backup, then move). A naive watcher triggers a rebuild for each event.

**Solution: debounce.** Collect events over a short window (50–200 ms), coalesce them, and trigger the reload once.

```cpp
#include <chrono>
#include <set>

class DebouncedWatcher {
    std::set<std::string> pending;
    std::chrono::steady_clock::time_point lastFire;
    static constexpr auto DEBOUNCE = std::chrono::milliseconds(100);
    
public:
    void onFileChanged(const std::string& path) {
        pending.insert(path);
        
        auto now = std::chrono::steady_clock::now();
        if (now - lastFire >= DEBOUNCE) {
            onReloadDue();
            lastFire = now;
            pending.clear();
        }
    }
    
    void onReloadDue() {
        // Rebuild, reload, etc.
    }
};
```

---

## State Preservation Strategies

### 1. Serialization Round-Trip

The application serializes its state before unload, and deserializes after load.

**Pros:**
- Explicit and visible.
- Works for any state that is serializable.
- Solves the "stale object layout" problem: you serialize the old layout, deserialize into the new layout.

**Cons:**
- Serialization overhead. If state is large, this is slow.
- Serialization format must be stable. If you change the serialization, old saves are unreadable.
- Some state is hard to serialize (OS file handles, thread IDs, locks).

```cpp
// In game engine
struct GameState {
    std::vector<EntityData> entities;
    std::map<std::string, int> globals;
    float elapsedTime;
};

void serializeState(const GameState& state, FILE* f) {
    fprintf(f, "%zu\n", state.entities.size());
    for (const auto& e : state.entities) {
        fprintf(f, "%d %d %d\n", e.id, e.x, e.y);
    }
    fprintf(f, "%zu\n", state.globals.size());
    for (const auto& [k, v] : state.globals) {
        fprintf(f, "%s %d\n", k.c_str(), v);
    }
    fprintf(f, "%f\n", state.elapsedTime);
}

void deserializeState(FILE* f, GameState& state) {
    size_t n;
    fscanf(f, "%zu\n", &n);
    state.entities.resize(n);
    for (auto& e : state.entities) {
        fscanf(f, "%d %d %d\n", &e.id, &e.x, &e.y);
    }
    fscanf(f, "%zu\n", &n);
    state.globals.clear();
    for (size_t i = 0; i < n; ++i) {
        char k[256];
        int v;
        fscanf(f, "%s %d\n", k, &v);
        state.globals[k] = v;
    }
    fscanf(f, "%f\n", &state.elapsedTime);
}

// In reload loop
void reloadGame() {
    GameState tempState;
    serializeState(globalGameState, stdout);  // Save
    dlclose(gameHandle);
    gameHandle = dlopen("./game.so", RTLD_NOW);
    auto onLoad = (void(*)(const GameState&))dlsym(gameHandle, "onLoad");
    // Pass temp state to new library; it deserializes
    onLoad(tempState);
}
```

### 2. Immutable Blessed Structures

Some state is designated as "blessed" — it must remain valid across reloads. It is allocated in stable memory (not in the reloaded library), and the new code knows to read from it.

```cpp
// stable.h (part of the engine, not reloaded)
struct BlessedState {
    int* entityCount;
    EntityData* entities;
    FILE* logFile;
};

extern BlessedState gBlessedState;  // Lives in engine, not game.so

// game.cpp (reloaded)
#include "stable.h"

void gameUpdate() {
    for (int i = 0; i < *gBlessedState.entityCount; ++i) {
        // Read/modify entity
        gBlessedState.entities[i].x += 1;
    }
}

// Reload is safe: gameUpdate() still sees the same pointers, same data
```

**Pros:**
- No serialization overhead.
- Simple; data is always in sync.
- Pointer identity is preserved.

**Cons:**
- Layout is fixed. Cannot add/remove fields to blessed structures.
- Brittle. New code must know to use the blessed structures.
- Mixed responsibilities. The engine manages some state; the game code manages other state.

### 3. Accepting State Loss

The simplest approach: **don't preserve state.** Re-run initialization after a reload.

```cpp
void onReload() {
    globalCache.clear();
    globalConnections.closeAll();
    reinitialize();
}
```

**Pros:**
- Trivial to implement.
- No bugs from stale state.

**Cons:**
- Expensive if initialization is slow (e.g., loading assets, connecting to databases).
- Breaks user experience in interactive applications (UI state is lost).

---

## Worked Example: C++ Edit-Recompile-Reload Loop

Let us build a minimal hot-reload harness in C++. The setup:

- **engine**: the stable main program, stays resident.
- **game.so**: the reloadable plugin, contains game logic.
- **watcher**: detects `game.cpp` changes, recompiles, signals engine to reload.

### Directory Structure

```
hot-reload-demo/
├── engine.cpp          (main, stable, does the reloading)
├── game.h              (interface; known to both engine and game)
├── game.cpp            (game logic; reloadable)
├── build.sh            (compile script)
└── Makefile
```

### game.h (Stable Interface)

```cpp
#ifndef GAME_H
#define GAME_H

struct GameState {
    int frame;
    int health;
};

// Each reloaded library exports these symbols
extern "C" {
    void gameInit(GameState* state);
    void gameUpdate(GameState* state);
    void gameShutdown(GameState* state);
};

#endif
```

### game.cpp (Reloadable)

```cpp
#include "game.h"
#include <cstdio>

// Static module state (NOT preserved across reload)
static int ticksPerSecond = 60;

extern "C" void gameInit(GameState* state) {
    state->frame = 0;
    state->health = 100;
    std::printf("[GAME] Init complete\n");
}

extern "C" void gameUpdate(GameState* state) {
    state->frame++;
    // Damage the player over time
    if (state->frame % ticksPerSecond == 0) {
        state->health--;
        std::printf("[GAME] Frame %d, Health %d\n", state->frame, state->health);
    }
}

extern "C" void gameShutdown(GameState* state) {
    std::printf("[GAME] Shutdown, final health: %d\n", state->health);
}
```

### engine.cpp (Stable Host)

```cpp
#include "game.h"
#include <dlfcn.h>
#include <csignal>
#include <cstdio>
#include <cstring>
#include <sys/inotify.h>
#include <unistd.h>
#include <chrono>
#include <thread>

// Function pointers to the reloaded library
typedef void (*GameInitFn)(GameState*);
typedef void (*GameUpdateFn)(GameState*);
typedef void (*GameShutdownFn)(GameState*);

struct GameLib {
    void* handle;
    GameInitFn init;
    GameUpdateFn update;
    GameShutdownFn shutdown;
};

GameLib gameLib = {};
GameState gameState = {};
bool shouldReload = false;
bool shouldExit = false;

void signalHandler(int sig) {
    if (sig == SIGUSR1) {
        shouldReload = true;
    } else if (sig == SIGINT) {
        shouldExit = true;
    }
}

bool loadGameLib(GameLib* lib) {
    // Close old library if it exists
    if (lib->handle) {
        if (lib->shutdown) {
            lib->shutdown(&gameState);
        }
        dlclose(lib->handle);
    }
    
    // Open new library
    lib->handle = dlopen("./game.so", RTLD_NOW);
    if (!lib->handle) {
        std::fprintf(stderr, "dlopen failed: %s\n", dlerror());
        return false;
    }
    
    // Load symbols
    lib->init = (GameInitFn)dlsym(lib->handle, "gameInit");
    lib->update = (GameUpdateFn)dlsym(lib->handle, "gameUpdate");
    lib->shutdown = (GameShutdownFn)dlsym(lib->handle, "gameShutdown");
    
    if (!lib->init || !lib->update) {
        std::fprintf(stderr, "dlsym failed\n");
        dlclose(lib->handle);
        lib->handle = nullptr;
        return false;
    }
    
    // Initialize (or reinitialize, depending on design)
    // For this example, we skip init on reload to preserve state
    if (gameState.frame == 0) {
        lib->init(&gameState);
    }
    
    std::printf("[ENGINE] Game library loaded\n");
    return true;
}

void fileWatcherThread() {
    int ifd = inotify_init1(IN_NONBLOCK);
    if (ifd < 0) {
        perror("inotify_init1");
        return;
    }
    
    int wd = inotify_add_watch(ifd, "game.cpp", IN_CLOSE_WRITE);
    if (wd < 0) {
        perror("inotify_add_watch");
        close(ifd);
        return;
    }
    
    std::printf("[WATCHER] Watching game.cpp\n");
    
    char buf[4096];
    std::chrono::steady_clock::time_point lastTrigger;
    const auto DEBOUNCE = std::chrono::milliseconds(200);
    
    while (!shouldExit) {
        int len = read(ifd, buf, sizeof(buf));
        
        if (len > 0) {
            auto now = std::chrono::steady_clock::now();
            if (now - lastTrigger >= DEBOUNCE) {
                std::printf("[WATCHER] game.cpp changed, rebuilding...\n");
                // Trigger recompile
                int result = system("make game.so");
                if (result == 0) {
                    shouldReload = true;
                    lastTrigger = now;
                } else {
                    std::fprintf(stderr, "[WATCHER] Build failed\n");
                }
            }
        } else if (len < 0 && errno != EAGAIN) {
            perror("read");
        }
        
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    
    inotify_rm_watch(ifd, wd);
    close(ifd);
}

int main() {
    std::signal(SIGUSR1, signalHandler);
    std::signal(SIGINT, signalHandler);
    
    // Start file watcher in background
    std::thread watcher(fileWatcherThread);
    watcher.detach();
    
    // Load initial game library
    if (!loadGameLib(&gameLib)) {
        std::fprintf(stderr, "Failed to load game library\n");
        return 1;
    }
    
    // Main loop
    std::printf("[ENGINE] Starting main loop\n");
    int frameCount = 0;
    
    while (!shouldExit) {
        if (shouldReload) {
            std::printf("[ENGINE] Reloading...\n");
            if (!loadGameLib(&gameLib)) {
                std::fprintf(stderr, "[ENGINE] Reload failed, continuing with old code\n");
            }
            shouldReload = false;
        }
        
        gameLib.update(&gameState);
        frameCount++;
        
        std::this_thread::sleep_for(std::chrono::milliseconds(1000 / 60));  // 60 FPS
    }
    
    if (gameLib.shutdown) {
        gameLib.shutdown(&gameState);
    }
    if (gameLib.handle) {
        dlclose(gameLib.handle);
    }
    
    std::printf("[ENGINE] Exiting\n");
    return 0;
}
```

### Makefile

```makefile
CXXFLAGS = -std=c++17 -fPIC -Wall -Wextra

all: engine game.so

engine: engine.cpp game.h
	g++ $(CXXFLAGS) -o engine engine.cpp -ldl -lpthread

game.so: game.cpp game.h
	g++ $(CXXFLAGS) -shared -o game.so game.cpp

clean:
	rm -f engine game.so

run: all
	./engine

.PHONY: all clean run
```

### Running the Example

```bash
make
./engine
# In another terminal:
# Edit game.cpp, change ticksPerSecond to 30 or the damage formula
# Save and watch the reload happen; state (frame, health) is preserved
```

### What Breaks This Example

**Scenario 1: You add a field to `GameState`**

```cpp
// Edit game.h
struct GameState {
    int frame;
    int health;
    int mana;  // NEW
};
```

Rebuild game.so. But the `GameState` instance in engine was created with the old struct size. The new library reads/writes `mana` at the wrong offset. Corruption.

**Fix:** Use serialization. Before reload, engine calls a `gameSerialize()` function that returns a byte buffer. After reload, engine calls `gameDeserialize()` to restore into the new layout.

**Scenario 2: You spawn a thread in the new game.so**

```cpp
// game.cpp, new version
#include <thread>
void gameUpdate(GameState* state) {
    std::thread t([] { /* ... */ });
    t.detach();
}
```

You reload. The old thread is still running, executing code from the old `.so`. The new `.so` is unloaded while the thread is mid-execution. Segfault or undefined behavior.

**Fix:** Before unload, wait for all threads spawned by the game code to exit. This requires explicit lifecycle tracking, not just dlclose().

**Scenario 3: The game code opens a file**

```cpp
// game.cpp
static FILE* logFile = fopen("game.log", "a");

extern "C" void gameUpdate(GameState* state) {
    fprintf(logFile, "Frame %d\n", state->frame);
}
```

On reload, the new `.so` opens a NEW file handle. The old one is leaked. After a few reloads, you run out of file descriptors.

**Fix:** Use the `gameShutdown()` hook to close resources. Ensure it is called before dlclose().

---

## Hot Reload vs Live Coding

**Live coding** (Smalltalk, Lisp REPL, Clojure REPL) treats the running program as the source of truth. You edit a function in the REPL, and the next call uses the new code. State is never restarted.

**Hot reload** starts from source files. You edit the file, rebuild, then the running process loads the new binary.

| Aspect | Live Coding | Hot Reload |
|--------|-------------|-----------|
| Source of truth | Running program | Source files |
| Feedback loop | Instant (milliseconds) | Slower (recompile time) |
| Architecture | Interpreted or REPL-based | Compiled; dynamic loading |
| State preservation | Automatic; program never stops | Explicit; requires hooks or serialization |
| Iteration style | Function-by-function experimentation | Edit-save-reload workflow |
| Tooling | Smalltalk image, REPL | File watcher, build system |

**When to use each:**

- **Live coding:** Exploratory programming, prototyping, data analysis (Python REPL, Julia REPL). The program is the artifact; source files are secondary.
- **Hot reload:** Development of compiled systems, game engines, web apps. Source files are the artifact; the running program is transient.

Both have their place. A discipline using hot reload can borrow live-coding ideas: a REPL connected to the running engine can evaluate expressions and call functions interactively, combining the speed of live coding with the structure of hot reload.

---

## Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|------------|
| **JavaScript HMR (module graph)** | Instant feedback; preserves React state; fine-grained reloads | Requires framework participation; module layout may break; third-party libs don't cooperate | Web development with Node/Webpack/Vite |
| **Erlang code server** | No forced state loss; multiple versions coexist; supervisor pattern is clean | Manual code_change callbacks; version skew; complexity | Erlang/Elixir services with high uptime demands |
| **JVM instrumentation (JRebel)** | Swap method bodies; preserves heap state; no restart | Limited to method bodies, not fields; fragile; requires agent | Development-time hot reload of Java apps |
| **C++ dlopen/dlclose** | Full control; any code can be swapped | Unsafe if not careful (threads, state layout, resource leaks); requires discipline | Game engines, plugin systems |
| **Serialization round-trip** | Explicit and visible; handles layout changes | Overhead; serialization format brittleness | When state is large or complex |
| **Blessed structures** | No serialization overhead; pointers stay valid | Fixed layout; brittle; mixed responsibilities | When blessed data is truly immutable or grows-only |
| **Accept state loss** | Simplest | Breaks UX if initialization is slow | Stateless services, rapid iteration |

---

## Common Misconceptions

1. **"Hot reload is the same as auto-restart."** No. Auto-restart kills the process and loses state. Hot reload keeps the process and state alive.

2. **"If I just use dlopen/dlclose, hot reload is free."** No. Dynamic loading is the mechanism, but you must handle state migration, thread safety, and resource cleanup. The mechanism does not solve these problems.

3. **"Hot reload means I don't need to worry about backwards compatibility."** Wrong. If you change a data structure, the old running instance is corrupt. You still need a migration strategy (serialization, schema versioning, etc.).

4. **"Hot reload works for any code."** No. It breaks on type-layout changes, captured closures, threads, and static initialization. Different runtimes have different failure modes.

5. **"Live coding is just hot reload with a REPL."** Close, but not quite. Live coding treats the image as the source of truth; hot reload treats source files as authoritative. The mental models differ.

---

## Exercises

1. **Implement a file watcher.** Write a small C++ program using `inotify` (Linux) or `FSEvents` (macOS) that monitors a directory and prints file changes with proper debouncing. Aim for < 100 ms latency between edit and detection.

2. **Extend the game example.** Add a `gameSerialize()` / `gameDeserialize()` pair to the worked example. Edit `game.h` to add a new field (e.g., `int score`). Verify that the state is correctly preserved across reloads, including the new field.

3. **Thread safety.** In the game example, add a worker thread that reads `gameState` in a loop. Then implement a safe reload mechanism that waits for the thread to reach a safe point before swapping libraries. What synchronization primitive do you use?

4. **Track resource leaks.** Modify the game example to open and close a file in the game library. Reload multiple times and check using `lsof` or `/proc/<pid>/fd` that no file descriptors are leaked. What happens if you forget the shutdown hook?

5. **Comparison: serialization vs blessed structures.** Implement two versions of a small cache system: one using serialization (save on unload, load after reload), one using a blessed structure (cache lives in engine, game code accesses it). Which is simpler? Which is faster? Which is safer?

6. **Break it intentionally.** Modify the game example to violate assumptions: add a field to `GameState` without updating the engine, spawn a thread and reload while it is running, open a file and forget to close it. Document what breaks and why.

---

## Summary

Hot reload is the illusion of editing code and seeing results in real time while keeping the application alive. Behind the scenes, file watchers detect changes, a build system recompiles, and dynamic loading (or framework-level swapping) exchanges the old code for the new. State is preserved via serialization, blessed structures, or explicit hooks; what breaks is type layout changes, captured closures, threads, and resource leaks.

JavaScript and game engines excel at hot reload because they are already architected for dynamism; statically typed systems like C++ require more discipline. The tradeoff is always the same: flexibility in exchange for explicit state management. Understand the runtime you are using, architect for reload from the start, and hot reload stops being magic and becomes a reliable tool.

---

> **[← Previous: Plugin Architecture](07-plugin-architecture.md)** · **[↑ Part 7](README.md)** · **[Next: Configuration Systems →](09-configuration-systems.md)**
