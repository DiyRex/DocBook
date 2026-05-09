# Chapter 74 — Plugin Architecture

A plugin system lets users extend your application without recompiling or forking it. The trick is defining a *stable* extension point that survives core changes, while letting plugins evolve independently. The worst plugin API leaks the implementation details of the host and changes every release. The best looks like it was designed five years ago and hasn't changed since.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain the four major plugin loading mechanisms — dynamic library, embedded interpreter, subprocess, WebAssembly — and the tradeoff each makes on isolation vs performance.
2. Understand the ABI problem: why most plugin APIs are C-shaped even when the host is C++, and why name mangling and vtable layout matter.
3. Design a versioned plugin interface that survives ABI changes and detects version skew.
4. Implement a plugin host that discovers, loads, and manages plugin lifecycles safely.
5. Recognize when a plugin architecture is the right choice vs a library or a microservice.
6. Reason about sandboxing: what untrusted code needs to be confined, and which mechanism provides it.

---

## Why Plugins Exist

Software shipped by one team often needs to be extended by users — sometimes the user is another team, sometimes it's the customer, sometimes it's the open-source community.

Editors make this explicit: VSCode, Vim, and Emacs all let you write plugins in their extension language (TypeScript, Vimscript, Emacs Lisp respectively). A browser loads plugins: Flash, PDF readers, ad blockers. Games let modders write mods: a Minecraft mod extends the game's block types and recipes. Build tools are plugin factories: webpack, esbuild, Babel, and postcss all export plugin APIs; the core is a plugin host, and feature requests become plugins.

The alternative — users fork the code — does not scale. Forks diverge. You end up maintaining a hundred copies of your project, each with custom changes, none able to share bug fixes.

Plugin architectures ask: **what is the minimal interface that a user-provided extension needs to satisfy?** If you can answer that precisely, you can:

- Keep the host's evolution decoupled from the plugin's.
- Let multiple plugins coexist and be updated independently.
- Let plugins fail without crashing the host (ideally).
- Sandbox untrusted code.

The question is not "are plugins right for this?" (they usually are for reusable software). The question is "which loading mechanism is right, and how do I version the interface so it doesn't become a maintenance nightmare?"

---

## Loading Mechanisms

Four patterns are common. Each trades off isolation, performance, complexity, and binary compatibility.

### 1. Dynamic Library Loading

The simplest: your plugin is a `.so` (Linux), `.dll` (Windows), or `.dylib` (macOS) file. At runtime, you call `dlopen(path)` to load it, `dlsym(handle, "get_plugin")` to fetch a function pointer, and you get back a struct with function pointers to your plugin's implementation.

```cpp
// host.cpp
#include <dlfcn.h>
#include <cstdio>

// The plugin interface (C-shaped).
struct Plugin {
    const char* name;
    int (*initialize)(void);
    int (*process)(const char* input, char* output, size_t output_len);
    void (*shutdown)(void);
    int version;  // for ABI checking
};

typedef Plugin* (*GetPluginFn)(void);

int main(int argc, char** argv) {
    if (argc < 2) {
        std::fprintf(stderr, "usage: %s <plugin.so>\n", argv[0]);
        return 1;
    }

    // Load the plugin.
    void* handle = dlopen(argv[1], RTLD_LAZY);
    if (!handle) {
        std::fprintf(stderr, "dlopen failed: %s\n", dlerror());
        return 1;
    }

    // Get the plugin factory function.
    GetPluginFn get_plugin = (GetPluginFn)dlsym(handle, "get_plugin");
    if (!get_plugin) {
        std::fprintf(stderr, "dlsym failed: %s\n", dlerror());
        dlclose(handle);
        return 1;
    }

    // Instantiate the plugin.
    Plugin* plugin = get_plugin();
    if (!plugin) {
        std::fprintf(stderr, "plugin factory returned null\n");
        dlclose(handle);
        return 1;
    }

    // Check version.
    if (plugin->version != PLUGIN_VERSION) {
        std::fprintf(stderr, "plugin version mismatch: expected %d, got %d\n",
                     PLUGIN_VERSION, plugin->version);
        dlclose(handle);
        return 1;
    }

    // Use the plugin.
    if (plugin->initialize()) {
        std::fprintf(stderr, "plugin->initialize() failed\n");
        dlclose(handle);
        return 1;
    }

    char output[256];
    int ret = plugin->process("hello", output, sizeof(output));
    std::printf("result: %d, output: %s\n", ret, output);

    plugin->shutdown();
    dlclose(handle);
    return 0;
}
```

A plugin implements the interface:

```cpp
// plugin.cpp
extern "C" {

struct Plugin {
    const char* name;
    int (*initialize)(void);
    int (*process)(const char* input, char* output, size_t output_len);
    void (*shutdown)(void);
    int version;
};

#define PLUGIN_VERSION 1

int my_initialize(void) {
    std::printf("plugin initialized\n");
    return 0;
}

int my_process(const char* input, char* output, size_t output_len) {
    // Null-checking, bounds-checking omitted for brevity.
    std::snprintf(output, output_len, "processed: %s", input);
    return 0;
}

void my_shutdown(void) {
    std::printf("plugin shut down\n");
}

Plugin MY_PLUGIN = {
    .name = "my-plugin",
    .initialize = my_initialize,
    .process = my_process,
    .shutdown = my_shutdown,
    .version = PLUGIN_VERSION
};

Plugin* get_plugin(void) {
    return &MY_PLUGIN;
}

}  // extern "C"
```

Build the plugin:

```bash
g++ -fPIC -shared plugin.cpp -o plugin.so
g++ host.cpp -ldl -o host
./host ./plugin.so
```

**Pros:**
- Fast: native code, no interpretation, minimal indirection.
- Native: you can use C++ features in the plugin (careful: see ABI below).
- Familiar: how VSCode, Vim, game engines, and browsers do it.

**Cons:**
- ABI risk: if the host and plugin are compiled with different C++ versions, or optimizations, or struct layouts, the interface breaks.
- No isolation: a malicious plugin can read the host's memory, corrupt it, or crash the process.
- Platform-specific: a plugin for Linux won't run on Windows.

### 2. Embedded Interpreter

Your plugin is code in an embedded language — Lua, Python, or JavaScript (Node.js in a context, V8/SpiderMonkey as a library). The host runs the interpreter and gives it a set of C++ functions it can call.

```cpp
// host.cpp (lua example)
#include <lua.h>
#include <lauxlib.h>
#include <lualib.h>

// A C++ function we expose to Lua.
int cpp_process(lua_State* L) {
    const char* input = luaL_checkstring(L, 1);  // arg 1
    char output[256];
    std::snprintf(output, sizeof(output), "processed: %s", input);
    lua_pushstring(L, output);
    return 1;  // return one value
}

int main(int argc, char** argv) {
    if (argc < 2) {
        std::fprintf(stderr, "usage: %s <script.lua>\n", argv[0]);
        return 1;
    }

    // Create a Lua state.
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);  // standard library

    // Register our C++ function.
    lua_register(L, "process", cpp_process);

    // Load and execute the user's script.
    int ret = luaL_dofile(L, argv[1]);
    if (ret != LUA_OK) {
        std::fprintf(stderr, "lua error: %s\n", lua_tostring(L, -1));
        lua_close(L);
        return 1;
    }

    // Call a function defined in the Lua script.
    lua_getglobal(L, "on_message");
    lua_pushstring(L, "hello");
    if (lua_pcall(L, 1, 0, 0) != LUA_OK) {
        std::fprintf(stderr, "lua call failed: %s\n", lua_tostring(L, -1));
    }

    lua_close(L);
    return 0;
}
```

The plugin (in Lua):

```lua
-- plugin.lua
function on_message(msg)
    local result = process(msg)  -- calls our C++ function
    print("plugin received: " .. result)
end
```

Build:

```bash
g++ host.cpp -llua -o host
./host plugin.lua
```

**Pros:**
- ABI stable: the Lua interpreter is your boundary; changes to your C++ code don't break plugins.
- Easy: plugin authors don't need a C++ compiler, just the language.
- Sandboxable: Lua (and Python with restricted builtins) can be confined to a subset of functions.

**Cons:**
- Slower: layers of indirection, garbage collection pause, type checking at runtime.
- Type mismatch: you must manually marshal between C++ types and Lua types (what if a plugin passes a string where you need an integer?).
- Requires a runtime: you ship a Lua interpreter (or Python, or Node) alongside your binary, adding size and complexity.

### 3. Subprocess / IPC

The plugin runs in a separate process. The host communicates with it over stdin/stdout, a socket, or a message queue (LSP — the Language Server Protocol — does this).

```cpp
// host.cpp
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>
#include <cstring>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

int main(int argc, char** argv) {
    if (argc < 2) {
        std::fprintf(stderr, "usage: %s <plugin-executable>\n", argv[0]);
        return 1;
    }

    int to_plugin[2], from_plugin[2];
    if (pipe(to_plugin) == -1 || pipe(from_plugin) == -1) {
        perror("pipe");
        return 1;
    }

    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        // Child: plugin process
        dup2(to_plugin[0], 0);    // stdin from host
        dup2(from_plugin[1], 1);  // stdout to host
        close(to_plugin[1]);
        close(from_plugin[0]);
        execv(argv[1], argv + 1);  // exec the plugin
        perror("execv");
        _exit(1);
    }

    // Parent: host process
    close(to_plugin[0]);
    close(from_plugin[1]);

    // Send a request to the plugin.
    json req;
    req["method"] = "process";
    req["input"] = "hello";
    std::string req_str = req.dump() + "\n";
    write(to_plugin[1], req_str.c_str(), req_str.size());

    // Read the response.
    char buf[1024];
    ssize_t n = read(from_plugin[0], buf, sizeof(buf) - 1);
    if (n > 0) {
        buf[n] = '\0';
        json resp = json::parse(buf);
        std::printf("plugin returned: %s\n", resp["output"].get<std::string>().c_str());
    }

    // Clean up.
    close(to_plugin[1]);
    close(from_plugin[0]);
    waitpid(pid, nullptr, 0);
    return 0;
}
```

The plugin (a separate executable):

```cpp
// plugin.cpp
#include <cstdio>
#include <string>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

int main() {
    std::string line;
    while (std::getline(std::cin, line)) {
        if (line.empty()) continue;
        
        json req = json::parse(line);
        json resp;

        if (req["method"] == "process") {
            std::string input = req["input"];
            char output[256];
            std::snprintf(output, sizeof(output), "processed: %s", input.c_str());
            resp["output"] = output;
        }

        std::printf("%s\n", resp.dump().c_str());
        std::fflush(stdout);
    }
    return 0;
}
```

Build:

```bash
g++ plugin.cpp -o plugin
g++ host.cpp -o host
./host ./plugin
```

**Pros:**
- Isolation: a buggy or malicious plugin cannot corrupt the host.
- Language-agnostic: the plugin can be written in any language.
- Failure containment: if the plugin crashes, the host can restart it.

**Cons:**
- Slow: IPC overhead, context switches, serialization (JSON, Protocol Buffers).
- Complexity: you need to define and version your IPC protocol.
- Debugging is harder: state is split across two processes.

### 4. WebAssembly

Modern: your plugin is WASM bytecode, loaded by a runtime like Wasmtime or Wasmer.

```cpp
// host.cpp
#include <wasmtime.h>
#include <cstdio>
#include <cstring>

int main(int argc, char** argv) {
    if (argc < 2) {
        std::fprintf(stderr, "usage: %s <plugin.wasm>\n", argv[0]);
        return 1;
    }

    // Create a Wasmtime engine and store.
    wasm_engine_t* engine = wasm_engine_new();
    wasm_store_t* store = wasm_store_new(engine);

    // Read the WASM module.
    FILE* file = fopen(argv[1], "rb");
    if (!file) {
        perror("fopen");
        return 1;
    }
    fseek(file, 0, SEEK_END);
    size_t size = ftell(file);
    rewind(file);
    wasm_byte_vec_t binary;
    wasm_byte_vec_new_uninitialized(&binary, size);
    fread(binary.data, size, 1, file);
    fclose(file);

    // Instantiate the module.
    wasm_module_t* module = wasm_module_new(store, &binary);
    wasm_byte_vec_delete(&binary);
    if (!module) {
        std::fprintf(stderr, "wasm_module_new failed\n");
        return 1;
    }

    wasm_instance_t* instance = wasm_instance_new(store, module, nullptr);
    if (!instance) {
        std::fprintf(stderr, "wasm_instance_new failed\n");
        return 1;
    }

    // Export the "process" function.
    wasm_extern_vec_t externs;
    wasm_instance_exports(instance, &externs);
    wasm_func_t* process_fn = nullptr;
    for (size_t i = 0; i < externs.size; ++i) {
        wasm_name_t name;
        wasm_extern_name(externs.data[i], &name);
        if (strncmp((const char*)name.data, "process", name.size) == 0) {
            process_fn = wasm_extern_as_func(externs.data[i]);
            break;
        }
    }

    if (!process_fn) {
        std::fprintf(stderr, "process function not found\n");
        return 1;
    }

    // Call the process function. (Details: pass i32 args, receive i32 result.)
    wasm_val_t args[1];
    args[0].kind = WASM_I32;
    args[0].of.i32 = 42;

    wasm_val_t results[1];
    if (wasm_func_call(process_fn, args, results)) {
        std::fprintf(stderr, "wasm_func_call failed\n");
        return 1;
    }

    std::printf("plugin returned: %d\n", results[0].of.i32);

    // Cleanup.
    wasm_instance_delete(instance);
    wasm_module_delete(module);
    wasm_store_delete(store);
    wasm_engine_delete(engine);
    return 0;
}
```

The plugin (in Rust, compiled to WASM):

```rust
// plugin.rs (rustc --target wasm32-unknown-unknown plugin.rs -o plugin.wasm)
#[no_mangle]
pub extern "C" fn process(input: i32) -> i32 {
    input * 2
}
```

**Pros:**
- Sandboxed by design: WASM has no access to the host OS unless you explicitly give it.
- Language-agnostic: any language that compiles to WASM works.
- Portable: a plugin runs on any OS that has a WASM runtime.
- Emerging standard: good tooling, growing ecosystem.

**Cons:**
- Performance: slower than native code (though much faster than an interpreter).
- Immaturity: not yet as battle-tested as `.so` or subprocess approaches.
- Limited access: you must explicitly expose host functions; plugins cannot use libc directly.

---

## The ABI Problem

Name mangling, vtable layout, calling conventions, alignment, size of `size_t` — all of these are part of the Application Binary Interface (ABI). If the host and plugin disagree on any of them, the interface breaks silently.

### Why C-Shaped Interfaces Are Safer

C does not mangle names. C does not have virtual functions. C has a simple, stable calling convention. For these reasons, the safest plugin interface is a C interface — even if your host is C++.

Compare:

```cpp
// BAD: C++ interface
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual std::string process(std::string input) = 0;
};

typedef IPlugin* (*GetPluginFn)();
```

If the host is compiled with `-std=c++17` and the plugin with `-std=c++20`, or if one uses libstdc++ and the other uses libc++, the vtable layout might differ. `std::string` layout differs between versions. Name mangling is version-dependent. A plugin might call the wrong `process` override, or write past the end of a `std::string` in the host's memory.

```cpp
// GOOD: C-shaped interface
struct Plugin {
    void* (*create)(void);
    void (*destroy)(void*);
    int (*process)(void* self, const char* input, char* output, size_t output_len);
    int version;
};

typedef Plugin* (*GetPluginFn)(void);
```

Now the interface is pure C: no name mangling, no vtables, no standard library types. The plugin and host can disagree on everything except the struct layout and the function signatures.

### Name Mangling Example

```cpp
// host, compiled with g++ 9
namespace host {
void process(std::string s);  // mangled to _ZN4host7processESs
}

// plugin, compiled with g++ 10 or clang
// tries to call the same function, but the mangling is different
// so dlsym fails, or worse, calls the wrong function
```

With extern "C", there is no mangling:

```cpp
extern "C" {
void host_process(const char* s) { /* ... */ }
}
```

Both g++ 9 and g++ 10 use the same symbol: `host_process`.

---

## Versioning the Plugin API

Your plugin interface will change. You must handle skew.

### Semantic Versioning

Define a version constant in the interface:

```cpp
extern "C" {

#define PLUGIN_API_VERSION 3

struct Plugin {
    int api_version;
    const char* name;
    int (*initialize)(void);
    int (*process)(const char* input, char* output, size_t len);
    // ... more function pointers
};

}
```

Before using a plugin, check:

```cpp
Plugin* plugin = get_plugin();
if (plugin->api_version != PLUGIN_API_VERSION) {
    std::fprintf(stderr, "version mismatch: expected %d, got %d\n",
                 PLUGIN_API_VERSION, plugin->api_version);
    return false;
}
```

When you add a new required method, bump MAJOR:

```cpp
#define PLUGIN_API_VERSION 4

struct Plugin {
    int api_version;
    const char* name;
    int (*initialize)(void);
    int (*process)(const char* input, char* output, size_t len);
    int (*configure)(const char* json_config);  // NEW
};
```

When you add an optional method, provide a sentinel:

```cpp
#define PLUGIN_API_VERSION 4  // MINOR bump, not MAJOR

struct Plugin {
    int api_version;
    // ... existing methods ...
    int (*configure)(const char* json_config);  // new; may be NULL
};
```

Check at runtime:

```cpp
if (plugin->configure) {
    plugin->configure(config_str);
}
```

### Capability Negotiation

For more flexibility, let the plugin declare capabilities:

```cpp
struct Plugin {
    int api_version;
    const char* name;
    int (*initialize)(void);
    int (*process)(const char*, char*, size_t);
    
    // Capabilities bitmap
    int capabilities;  // bit 0: can_configure, bit 1: can_query, etc.
};

#define CAP_CONFIGURE (1 << 0)
#define CAP_QUERY     (1 << 1)

// In the host:
if (plugin->capabilities & CAP_CONFIGURE) {
    plugin->configure(...);
}
```

---

## Sandboxing

If you load untrusted plugins, you must confine them. Static binary loading (`.so`) offers no isolation. Subprocess and WebAssembly do.

### Subprocess Sandboxing

Run the plugin with restricted capabilities:

```cpp
pid_t pid = fork();
if (pid == 0) {
    // Child: restrict before exec.
    
    // Drop privileges.
    setuid(NOBODY_UID);
    
    // Restrict syscalls (Linux seccomp).
    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
    seccomp_load(ctx);
    
    // Now exec the plugin. It can only read, write, and exit.
    execv(plugin_path, argv);
    _exit(1);
}
```

The plugin process can only use whitelisted syscalls. Attempts to open files, fork, or use the network are killed by the kernel.

### WebAssembly Sandboxing

WASM sandboxing is built in. A WASM module cannot:

- Access the host's memory (unless you explicitly give it a region).
- Call functions (unless you expose them).
- Access the filesystem (unless you expose a virtual filesystem).
- Access the network (unless you expose a network API).

To give a plugin controlled access:

```cpp
// Expose a "read_file" function to the plugin.
wasm_func_t* read_file_fn = /* ... export from host ... */;

// The plugin calls it as needed; the host can audit and restrict.
```

---

## Discovery and Lifecycle

### Discovery

Plugins live in a directory. Scan it:

```cpp
#include <filesystem>

namespace fs = std::filesystem;

std::vector<std::string> discover_plugins(const std::string& plugin_dir) {
    std::vector<std::string> plugins;
    for (const auto& entry : fs::directory_iterator(plugin_dir)) {
        if (entry.path().extension() == ".so" ||
            entry.path().extension() == ".dll" ||
            entry.path().extension() == ".dylib") {
            plugins.push_back(entry.path().string());
        }
    }
    return plugins;
}
```

Or use a manifest:

```json
{
  "plugins": [
    { "name": "plugin_a", "path": "./plugins/a.so", "version": "1.0" },
    { "name": "plugin_b", "path": "./plugins/b.so", "version": "2.1" }
  ]
}
```

### Lifecycle

Plugins have a lifecycle:

1. **Load**: `dlopen` or subprocess spawn.
2. **Initialize**: call the plugin's init function; let it set up state.
3. **Run**: call the plugin's methods.
4. **Shutdown**: call the plugin's shutdown function; let it clean up.
5. **Unload**: `dlclose` or kill the subprocess.

```cpp
struct PluginInstance {
    void* handle;
    Plugin* plugin;
    bool initialized = false;
};

bool load_plugin(const std::string& path, PluginInstance& out) {
    void* handle = dlopen(path.c_str(), RTLD_LAZY);
    if (!handle) return false;

    GetPluginFn get_plugin = (GetPluginFn)dlsym(handle, "get_plugin");
    if (!get_plugin) {
        dlclose(handle);
        return false;
    }

    Plugin* plugin = get_plugin();
    if (!plugin || plugin->api_version != PLUGIN_API_VERSION) {
        dlclose(handle);
        return false;
    }

    if (plugin->initialize()) {
        dlclose(handle);
        return false;
    }

    out.handle = handle;
    out.plugin = plugin;
    out.initialized = true;
    return true;
}

void unload_plugin(PluginInstance& instance) {
    if (instance.initialized && instance.plugin) {
        instance.plugin->shutdown();
    }
    if (instance.handle) {
        dlclose(instance.handle);
    }
    instance.initialized = false;
    instance.handle = nullptr;
    instance.plugin = nullptr;
}
```

### Hot Reload

Some hosts (game engines, editors) let you reload plugins without restarting:

```cpp
bool reload_plugin(PluginInstance& instance, const std::string& path) {
    PluginInstance new_instance;
    if (!load_plugin(path, new_instance)) {
        return false;
    }

    // Migrate state from old to new (user-defined).
    migrate_state(instance.plugin, new_instance.plugin);

    unload_plugin(instance);
    instance = new_instance;
    return true;
}
```

This requires the plugin to support state serialization and the host to understand what that state is.

---

## Worked Example: Shape Calculator

A small host loads plugins that compute the area of shapes.

**Interface** (shared header):

```cpp
// plugin.h
#ifndef PLUGIN_H
#define PLUGIN_H

#include <cstddef>

#ifdef __cplusplus
extern "C" {
#endif

#define PLUGIN_API_VERSION 1

struct Shape {
    double* dimensions;  // varies per shape
    size_t num_dimensions;
};

struct Plugin {
    int api_version;
    const char* name;
    const char* shape_type;  // "circle", "square", "rectangle", ...
    
    int (*initialize)(void);
    double (*area)(struct Shape shape);
    void (*shutdown)(void);
};

typedef struct Plugin* (*GetPluginFn)(void);

#ifdef __cplusplus
}
#endif

#endif
```

**Host** (host.cpp):

```cpp
#include "plugin.h"
#include <dlfcn.h>
#include <cstdio>
#include <map>

struct LoadedPlugin {
    void* handle;
    Plugin* plugin;
};

std::map<std::string, LoadedPlugin> plugins;

bool load_plugin(const char* path) {
    void* handle = dlopen(path, RTLD_LAZY);
    if (!handle) {
        std::fprintf(stderr, "dlopen %s: %s\n", path, dlerror());
        return false;
    }

    GetPluginFn get_plugin = (GetPluginFn)dlsym(handle, "get_plugin");
    if (!get_plugin) {
        std::fprintf(stderr, "dlsym get_plugin: %s\n", dlerror());
        dlclose(handle);
        return false;
    }

    Plugin* plugin = get_plugin();
    if (!plugin || plugin->api_version != PLUGIN_API_VERSION) {
        std::fprintf(stderr, "plugin version mismatch\n");
        dlclose(handle);
        return false;
    }

    if (plugin->initialize()) {
        std::fprintf(stderr, "plugin->initialize() failed\n");
        dlclose(handle);
        return false;
    }

    plugins[plugin->shape_type] = {handle, plugin};
    std::printf("loaded plugin: %s (type: %s)\n", plugin->name, plugin->shape_type);
    return true;
}

double compute_area(const char* shape_type, double* dimensions, size_t num_dims) {
    auto it = plugins.find(shape_type);
    if (it == plugins.end()) {
        std::fprintf(stderr, "unknown shape: %s\n", shape_type);
        return -1.0;
    }

    Shape shape;
    shape.dimensions = dimensions;
    shape.num_dimensions = num_dims;

    return it->second.plugin->area(shape);
}

int main() {
    // Load plugins.
    load_plugin("./area_circle.so");
    load_plugin("./area_rectangle.so");

    // Use them.
    double circle_dims[] = {5.0};  // radius
    std::printf("circle area: %.2f\n", compute_area("circle", circle_dims, 1));

    double rect_dims[] = {4.0, 6.0};  // width, height
    std::printf("rectangle area: %.2f\n", compute_area("rectangle", rect_dims, 2));

    // Cleanup.
    for (auto& [name, loaded] : plugins) {
        loaded.plugin->shutdown();
        dlclose(loaded.handle);
    }

    return 0;
}
```

**Plugin 1** (area_circle.cpp):

```cpp
#include "plugin.h"
#include <cmath>

static int circle_initialize(void) {
    return 0;
}

static double circle_area(struct Shape shape) {
    if (shape.num_dimensions < 1) return 0.0;
    double radius = shape.dimensions[0];
    return M_PI * radius * radius;
}

static void circle_shutdown(void) {
}

extern "C" {

Plugin CIRCLE_PLUGIN = {
    .api_version = PLUGIN_API_VERSION,
    .name = "Circle Area",
    .shape_type = "circle",
    .initialize = circle_initialize,
    .area = circle_area,
    .shutdown = circle_shutdown
};

Plugin* get_plugin(void) {
    return &CIRCLE_PLUGIN;
}

}
```

**Plugin 2** (area_rectangle.cpp):

```cpp
#include "plugin.h"

static int rect_initialize(void) {
    return 0;
}

static double rect_area(struct Shape shape) {
    if (shape.num_dimensions < 2) return 0.0;
    return shape.dimensions[0] * shape.dimensions[1];
}

static void rect_shutdown(void) {
}

extern "C" {

Plugin RECT_PLUGIN = {
    .api_version = PLUGIN_API_VERSION,
    .name = "Rectangle Area",
    .shape_type = "rectangle",
    .initialize = rect_initialize,
    .area = rect_area,
    .shutdown = rect_shutdown
};

Plugin* get_plugin(void) {
    return &RECT_PLUGIN;
}

}
```

Build:

```bash
g++ -fPIC -shared area_circle.cpp -o area_circle.so
g++ -fPIC -shared area_rectangle.cpp -o area_rectangle.so
g++ host.cpp -ldl -o host
./host
```

Output:

```
loaded plugin: Circle Area (type: circle)
loaded plugin: Rectangle Area (type: rectangle)
circle area: 78.54
rectangle area: 24.00
```

---

## Plugins vs Libraries vs Microservices

Each is a different position on the boundary-crossing cost curve (Chapter 43):

| Aspect | Library | Plugin | Microservice |
|--------|---------|--------|--------------|
| Linkage | Compile time (static or dynamic) | Runtime (dlopen or subprocess) | Network (HTTP, gRPC) |
| Isolation | None — shared address space | Partial (subprocess) or full (WASM) | Full — separate process |
| Performance | Fastest — direct calls, inlined | Faster — function pointers, vtables | Slowest — IPC + serialization |
| Failure containment | None — crash kills host | Partial (subprocess) or full (WASM) | Full — host unaffected |
| Language mixing | Hard — same ABI required | Easier (subprocess) | Easy — any language |
| Deployment | Recompile and redeploy host | Add file to plugin directory | Deploy separate service |
| Versioning | Semantic versioning; often tight coupling | Versioned API; decoupled | Versioned API; fully decoupled |

**Choose a library** when you want high performance and tight integration, and you control both sides of the boundary.

**Choose a plugin** when you want extensibility by users or a community, and you need to evolve independently.

**Choose a microservice** when the plugin is large, you need strong isolation, or you want to scale it separately.

---

## Tradeoffs

| Mechanism | Performance | Isolation | ABI Stability | Ease of Use | Startup Time |
|-----------|-------------|-----------|---------------|-------------|--------------|
| Dynamic library | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Interpreter | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| Subprocess | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ |
| WebAssembly | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

Pick based on your constraints. A hot-path graphics plugin? Dynamic library. A user-provided data transformation? Subprocess or interpreter. A third-party untrusted plugin? WebAssembly or subprocess with seccomp.

---

## Common Misconceptions

1. **"Plugins must be C-shaped."** Not always. You can use C++ if both host and plugin are compiled with the same compiler, version, flags, and standard library. But it's fragile. C-shaped is safer.

2. **"Dynamic loading is always faster than subprocess."** True for single calls, but subprocess amortizes IPC costs over larger computations. A plugin that processes 10 MB of data might show no performance difference.

3. **"Subprocess plugins are 'slower' because fork() is expensive."** Fork() is fast on modern OSes. Spawning a process once and reusing it amortizes the cost. The bottleneck is IPC, not process creation.

4. **"WebAssembly plugins are slow."** Compared to native code, yes. Compared to interpreters, no. Compared to subprocess IPC, they're in the same ballpark or faster.

5. **"If I version my API, I can change anything without breaking plugins."** Versioning is necessary but not sufficient. You still need to maintain backwards compatibility, or your versioning system must *enforce* compatibility (like capability negotiation).

---

## Exercises

1. **Extend the shape calculator.** Add a `triangle` plugin that takes three dimensions (the three side lengths) and computes area via Heron's formula. Compile and load it alongside the existing plugins. Change one field order in the Plugin struct and observe the crash; then restore it and explain why ABI matters.

2. **Version skew.** Modify the host to expect `PLUGIN_API_VERSION 2` (add a new optional field, `const char* author`). Load an old plugin (version 1). Does it crash? Does versioning catch it? Now write a capability-based version negotiation system where the host and plugin exchange versions before using any function pointers.

3. **Subprocess plugin.** Rewrite the shape calculator with the plugin as a subprocess. Use JSON over stdin/stdout. Time it against the dynamic library version for 100,000 calls. How much slower is IPC?

4. **Interpreter plugin.** Rewrite the shape calculator in Lua. Expose the `Shape` struct as a Lua table. Let a user script (in Lua) compute areas by calling C++ functions. Observe the type mismatch problem: what happens if a plugin passes a string where you expect a number?

5. **Sandboxing.** Write a simple file-reading plugin that reads `/etc/passwd`. Load it in a subprocess with seccomp filtering that denies the `open` syscall. Observe the plugin's failure and the kernel's enforcement.

---

## Summary

Plugin architectures decouple a host from its extensions. The choice of loading mechanism — dynamic library, interpreter, subprocess, or WebAssembly — determines the tradeoff between performance and isolation. Stable plugin APIs are C-shaped and versioned; they survive ABI changes and allow independent evolution. Choose plugins when extensibility by users matters; choose libraries for high-performance integration; choose microservices for strong isolation and separate scaling.

---

> **[← Previous: An Event Bus](06-an-event-bus.md)** · **[↑ Part 7](README.md)** · **[Next: Hot Reloading →](08-hot-reloading.md)**
