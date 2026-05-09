# Chapter 76 — Configuration Systems

Nine of ten production incidents involve a configuration mistake. Someone misconfigured a database connection string. A flag got flipped in the wrong environment. A secret leaked into a config file. The engineer who wrote the code was not at fault; they wrote the right logic. The system that held the configuration was at fault.

Configuration is the part of software you wish you didn't have to think about. It sits between deployment-time decisions (which database server?) and runtime behavior (how many worker threads?). It bridges intent (the system should do this) and reality (this environment is constrained that way). It is invisible when it works, catastrophic when it doesn't.

This chapter teaches how to make configuration boring — typed, layered, validated, observable. The goal is simple: configuration mistakes should be caught at load time, not at 3 AM in production.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Understand the layering order of configuration sources and why last-writer-wins is the right mental model.
2. Distinguish between static configuration (read once at startup) and dynamic configuration (re-read at runtime) and know the tradeoffs.
3. Build a typed configuration system that validates on load and fails fast instead of crashing at runtime.
4. Recognize when configuration should be data versus code, and avoid the common mistake of baking policy into code.
5. Separate secrets from configuration and know how to load them safely.
6. Apply the 12-Factor App configuration principle and understand where it breaks for legacy systems.
7. Understand feature flags as a specialized configuration system and when they are worth the complexity.
8. Implement and test a configuration system that handles multiple file formats, environment variables, and overrides.

---

## 76.1 The Layering Order

A realistic application reads configuration from multiple sources. The order matters. Here is the standard hierarchy:

```
Layer 1: Built-in defaults (baked into the binary)
Layer 2: Configuration file (TOML, YAML, JSON)
Layer 3: Environment variables (MYAPP_SETTING=value)
Layer 4: Command-line flags (--setting=value)
Layer 5: Runtime overrides (if applicable)
```

Later layers override earlier layers. This is called "last writer wins." It is intuitive because it reflects how engineers think: defaults are conservative; files are declarative; environment variables are scripted (for containers, CI/CD); flags are immediate and explicit.

Example: A database URL.

```cpp
struct Config {
    std::string database_url = "sqlite:///:memory:";  // Layer 1: default
};

// Layer 2: Load from file
auto cfg_file = toml::parse_file("config.toml");
if (cfg_file.contains("database_url")) {
    config.database_url = cfg_file["database_url"].value_or(config.database_url);
}

// Layer 3: Environment variable
if (const char* env_url = std::getenv("MYAPP_DATABASE_URL")) {
    config.database_url = env_url;
}

// Layer 4: Command-line flag
if (result.count("database-url")) {
    config.database_url = result["database-url"].as<std::string>();
}
```

Now, a user can do any of these:

```bash
# Use default (in-memory SQLite)
./myapp

# Use config file
./myapp --config=production.toml

# Override with env var (typical in containers)
MYAPP_DATABASE_URL=postgres://prod.db ./myapp

# Override with flag (for quick testing)
./myapp --database-url=postgres://test.db
```

Each layer is respected. A flag overrides an env var, which overrides a config file, which overrides the default. This is the right order because it aligns with how operators think: "I want the default behavior, except for this one thing right now."

**Why this order, not some other?**

- Defaults are baked; they never change without rebuilding.
- Config files are deployment-time; they change between environments (dev, staging, prod).
- Environment variables are container-time; they change as the container runs in different pods or machines.
- Flags are run-time and explicit; they are for immediate overrides.

This is a contract. If you implement it, operators learn it once and it works everywhere. If you make up a different order (or mix strategies), they have to learn your tool's idiosyncrasies.

---

## 76.2 Static vs. Dynamic Configuration

There are two models: read the configuration once at startup, or re-read it at runtime when it changes.

### Static Configuration

Static configuration is read once and never changes. Most systems work this way.

Pros:
- Simple to reason about. Config is set once; the system is predictable.
- No surprises. A part of the system doesn't suddenly change mid-operation.
- Easy to test. Mock the config once; it stays mocked.

Cons:
- To change config, you must restart the application.
- Restarts are disruptive. For a service handling traffic, a restart means downtime (or graceful shutdown + startup latency).

```cpp
// Static configuration: read once at startup
class Application {
private:
    Config config_;
    
public:
    Application(const Config& cfg) : config_(cfg) {}
    
    void run() {
        // config_ never changes; it is safe to read from any thread
        auto db = Database::connect(config_.database_url);
        // ...
    }
};

int main() {
    Config cfg = loadConfig();
    Application app(cfg);
    app.run();
}
```

### Dynamic Configuration

Dynamic configuration can change at runtime. A service like LaunchDarkly, Consul, or etcd provides the source of truth. The application polls or subscribes to changes.

Pros:
- No restart required. You can change configuration and see it take effect immediately.
- Gradual rollouts. A feature flag can be enabled for 10% of users, then 50%, then 100%, all without restarting.
- Experimentation. A/B tests can flip between configurations without stopping the system.

Cons:
- Much more complex. You must handle the case where a config value changed while the system was using the old value.
- Consistency questions arise. If config X changed but config Y didn't, and they are supposed to be coordinated, what happens? Eventual consistency is painful.
- Failure modes are subtle. A broken config push can silently corrupt behavior; a static config error catches at startup.

```cpp
// Dynamic configuration: watch for changes
class Application {
private:
    std::mutex config_mutex_;
    Config config_;
    
public:
    Application(const Config& initial_cfg) : config_(initial_cfg) {}
    
    void updateConfig(const Config& new_cfg) {
        std::lock_guard<std::mutex> lock(config_mutex_);
        config_ = new_cfg;
    }
    
    void run() {
        // Each read must lock; expensive if read frequently
        std::lock_guard<std::mutex> lock(config_mutex_);
        auto db = Database::connect(config_.database_url);
        // ...
    }
};

// Separate thread polls for config changes
void configWatcher(Application& app, ConfigServer& server) {
    while (true) {
        auto new_cfg = server.fetch();
        app.updateConfig(new_cfg);
        std::this_thread::sleep_for(std::chrono::seconds(5));
    }
}
```

**When to use dynamic configuration:**

- Feature flags (per-user, per-percentage rollouts). Waiting for a restart to test a flag is untenable.
- High-churn configuration. If you change config values weekly, dynamic is worth the cost.
- Blue-green deployments. You want to switch versions without restarts.

**When to stick with static:**

- Configuration that is rarely changed. Most systems fit this.
- Safety-critical systems where accidental config changes are unacceptable.
- Simple services where restart latency is acceptable.

The default choice is static. Move to dynamic only when you feel concrete pain.

---

## 76.3 Typed Configuration

Configuration is data, but data without a schema is error-prone. An untyped config system is like a map:

```cpp
// BAD: Untyped configuration
std::map<std::string, std::string> config;
config["max_workers"] = "4";
config["database_url"] = "postgres://...";

int workers = std::stoi(config["max_workers"]);  // Crashes if missing or invalid
```

Problems:
- If "max_workers" is missing, `std::stoi` throws. You don't know until runtime.
- If "max_workers" is "abc", the error is unintelligible.
- If "max_workers_" (typo) is set, the code silently uses the default.

Typed configuration validates on load:

```cpp
// GOOD: Typed configuration
struct Config {
    uint32_t max_workers = 4;
    std::string database_url;
    bool enable_cache = true;
    
    // Validation happens here
    void validate() const {
        if (database_url.empty()) {
            throw std::invalid_argument("database_url is required");
        }
        if (max_workers == 0 || max_workers > 1024) {
            throw std::invalid_argument("max_workers must be in [1, 1024]");
        }
    }
};

Config loadConfig(const std::string& path) {
    auto toml_data = toml::parse_file(path);
    
    Config cfg;
    cfg.max_workers = toml_data["max_workers"].value_or(cfg.max_workers);
    cfg.database_url = toml_data["database_url"].value_or(cfg.database_url);
    cfg.enable_cache = toml_data["enable_cache"].value_or(cfg.enable_cache);
    
    cfg.validate();  // Fail early
    return cfg;
}
```

Now, if "max_workers" is missing, the default (4) is used. If it is "abc" or 9999, validation catches it. If "database_url" is missing entirely, `validate()` rejects the config.

**The principle: fail fast.** A config error should be caught at load time, not when the code tries to use it hours later.

Languages with strong type systems (Rust, TypeScript) make this natural. Python has Pydantic. In C++, you can build it with structs and validation functions, or use a third-party library like Cpptoml or JSON for Modern C++.

---

## 76.4 Schema and Validation

Validation is not just type-checking. It is checking constraints: "max_workers must be positive and not exceed 1024," "database_url must be a valid URI," "timeout must be between 1 second and 1 hour."

Tools like JSON Schema, Cue, and Pkl let you define a schema separately from the data:

```toml
# config.toml
max_workers = 4
database_url = "postgres://localhost/mydb"
timeout_seconds = 30
```

```json
{
  "type": "object",
  "properties": {
    "max_workers": {
      "type": "integer",
      "minimum": 1,
      "maximum": 1024,
      "default": 4
    },
    "database_url": {
      "type": "string",
      "format": "uri"
    },
    "timeout_seconds": {
      "type": "integer",
      "minimum": 1,
      "maximum": 3600
    }
  },
  "required": ["database_url"]
}
```

A validator checks the config against this schema. If "max_workers" is 5000, the schema catches it. If "database_url" is not a URI, the schema catches it.

In C++, you can achieve this with a simple validation function in the Config struct:

```cpp
struct Config {
    uint32_t max_workers = 4;
    std::string database_url;
    uint32_t timeout_seconds = 30;
    
    void validate() const {
        if (database_url.empty()) {
            throw std::invalid_argument("database_url is required");
        }
        
        // Check URI format (simplified; a real implementation would use a URI parser)
        if (database_url.find("://") == std::string::npos) {
            throw std::invalid_argument("database_url must be a valid URI");
        }
        
        if (max_workers < 1 || max_workers > 1024) {
            throw std::invalid_argument("max_workers must be in [1, 1024]");
        }
        
        if (timeout_seconds < 1 || timeout_seconds > 3600) {
            throw std::invalid_argument("timeout_seconds must be in [1, 3600]");
        }
    }
};
```

**Why validate at load time?**

Because a config error at 3 AM in production, after the system has been running for hours, is worse than a config error at startup. If validation is deferred—if the system only checks "is max_workers in range?" when it first uses max_workers—the system might crash mid-operation. Typos and out-of-range values must be caught before the main event loop starts.

---

## 76.5 Secrets vs. Configuration

Configuration is metadata about the system: how many workers, what database, what timeout. Secrets are credentials: database passwords, API keys, private encryption keys.

The rules:
1. **Never put secrets in config files checked into version control.** Ever.
2. **Load secrets from a secret store, not a file.** Use AWS Secrets Manager, Vault, Kubernetes secrets, or encrypted env vars.
3. **Audit secret access.** Know who accessed which secret and when.
4. **Rotate secrets regularly.** A compromised secret should expire.

Bad:

```toml
# config.toml (committed to git)
database_url = "postgres://user:password@localhost/mydb"
api_key = "sk-1234567890abcdef"
```

Everyone with access to the repo has the secrets. If the repo is ever open-source, or if an engineer's laptop is stolen, the secrets are compromised.

Good:

```cpp
struct Secrets {
    std::string database_password;
    std::string api_key;
};

Secrets loadSecrets() {
    // Load from Vault, AWS Secrets Manager, or env vars
    Secrets secrets;
    
    if (const char* pwd = std::getenv("DB_PASSWORD")) {
        secrets.database_password = pwd;
    } else {
        throw std::runtime_error("DB_PASSWORD env var not set");
    }
    
    if (const char* key = std::getenv("API_KEY")) {
        secrets.api_key = key;
    } else {
        throw std::runtime_error("API_KEY env var not set");
    }
    
    return secrets;
}
```

In a containerized environment, secrets are provided at runtime:

```bash
docker run -e DB_PASSWORD=secret123 -e API_KEY=key456 myapp
```

Or in Kubernetes:

```yaml
spec:
  containers:
  - name: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

The secret is never stored in the source code or in a file. It is injected at deployment time. If the secret is compromised, you rotate it; the application uses the new secret the next time it is deployed or the next time it reads the env var.

---

## 76.6 The 12-Factor App Principle

The 12-Factor App methodology says: **All configuration comes from environment variables.** No config files, no command-line flags, just env vars.

The rationale: environment variables are the simplest way to vary behavior across environments (dev, staging, prod). Each environment sets different env vars; the binary is identical everywhere.

```bash
# Development
DATABASE_URL=sqlite:///:memory: WORKERS=1 ./myapp

# Production
DATABASE_URL=postgres://prod.db WORKERS=16 ./myapp
```

Strictly following 12-Factor means:

```cpp
// Everything from environment variables
struct Config {
    std::string database_url = std::getenv("DATABASE_URL") ?: "";
    uint32_t workers = std::stoi(std::getenv("WORKERS") ?: "1");
    bool debug = std::getenv("DEBUG") != nullptr;
};
```

**Pros:**
- Trivially different between environments. No config files to manage.
- Clear separation: code in git, secrets in env vars injected at runtime.
- Container-native. Kubernetes, Docker, and most orchestrators use env vars.

**Cons:**
- Hard to express complex config. A large config is dozens of env vars; setting them is tedious.
- Hard to version config. If you need to roll back config changes, env vars don't have history.
- No schema enforcement at a glance. "Is WORKERS required? What is the range?"

**Where 12-Factor works:**
- Microservices with 5-20 configuration values.
- Containers, Kubernetes, serverless. The deployment infrastructure manages env vars.
- Stateless services where all config is external.

**Where 12-Factor strains:**
- Legacy monoliths with 100+ configuration values.
- Systems where config must be versioned alongside code.
- Applications where configuration is hierarchical or has conditional sections.

The pragmatic approach: use env vars for secrets and deployment-specific values (database URL, port, feature flags). Use config files for application-level policies (timeouts, cache sizes, logging levels). Layer them: env vars override files, both override defaults.

---

## 76.7 Feature Flags

Feature flags are a specialized configuration system. A flag controls whether a feature is on or off, often per-user or per-percentage.

```cpp
if (featureFlags.isEnabled("new_checkout_flow", user_id)) {
    // New behavior
    return newCheckoutFlow(order);
} else {
    // Old behavior
    return legacyCheckoutFlow(order);
}
```

Purpose:
- **Gradual rollout.** Enable the feature for 10% of users, observe, then 50%, then 100%.
- **A/B testing.** Half the users see version A, half see version B; measure outcomes.
- **Kill switch.** If a feature is breaking production, flip it off without restarting.
- **Separation of deploy and release.** You can deploy code that includes a disabled feature, then enable it later.

Dynamic feature flags (LaunchDarkly, Statsig, Flagsmith) are worth it if:
- You deploy frequently (daily or more).
- You do A/B tests or gradual rollouts regularly.
- Restarting for a flag change is disruptive (24/7 services).

Simple feature flags (baked into config) are fine if:
- You deploy infrequently (weekly or monthly).
- You don't need per-user granularity.
- Restarting is acceptable.

Example: Static feature flags in a config file.

```toml
[features]
new_checkout_enabled = true
experimental_recommendation_engine = false
```

```cpp
struct FeatureFlags {
    bool new_checkout_enabled = false;
    bool experimental_recommendation_engine = false;
};

FeatureFlags loadFeatureFlags(const std::string& path) {
    auto toml_data = toml::parse_file(path);
    FeatureFlags flags;
    
    auto features = toml_data["features"];
    if (features) {
        flags.new_checkout_enabled = features["new_checkout_enabled"].value_or(false);
        flags.experimental_recommendation_engine = features["experimental_recommendation_engine"].value_or(false);
    }
    
    return flags;
}

// Usage
if (flags.new_checkout_enabled) {
    return newCheckout();
}
```

For per-user flags, you need a service:

```cpp
class FeatureFlagService {
private:
    std::string service_url_;
    
public:
    bool isEnabled(const std::string& flag_name, uint64_t user_id) {
        // Call LaunchDarkly, Statsig, or similar
        // GET /api/flags/new_checkout?user=<user_id>
        // Returns { "enabled": true }
        // (Simplified; real implementation would cache results)
    }
};
```

Feature flags are valuable but not free. They add complexity: more config to manage, more branches in code (dead code paths when flags are old), more testing (test both on and off). Use them when the benefit (faster rollout, easier experimentation) is clear.

---

## 76.8 Worked Example: A Typed Configuration Loader

Let us build a small configuration system in C++17 that:
1. Reads from a TOML file.
2. Layers in environment variables.
3. Validates.
4. Is testable.

### The Configuration Struct

```cpp
// config.h
#pragma once

#include <string>
#include <cstdint>
#include <stdexcept>

struct Config {
    // Database
    std::string database_url;
    uint32_t max_connections = 10;
    
    // Server
    uint16_t port = 8080;
    uint32_t worker_threads = 4;
    uint32_t request_timeout_seconds = 30;
    
    // Logging
    bool debug_mode = false;
    std::string log_level = "info";
    
    // Validation
    void validate() const {
        if (database_url.empty()) {
            throw std::invalid_argument("database_url is required");
        }
        
        if (max_connections < 1 || max_connections > 1000) {
            throw std::invalid_argument(
                "max_connections must be in [1, 1000]");
        }
        
        if (port == 0 || port > 65535) {
            throw std::invalid_argument(
                "port must be in [1, 65535]");
        }
        
        if (worker_threads < 1 || worker_threads > 512) {
            throw std::invalid_argument(
                "worker_threads must be in [1, 512]");
        }
        
        if (request_timeout_seconds < 1 || request_timeout_seconds > 3600) {
            throw std::invalid_argument(
                "request_timeout_seconds must be in [1, 3600]");
        }
        
        if (log_level != "debug" && log_level != "info" && 
            log_level != "warn" && log_level != "error") {
            throw std::invalid_argument(
                "log_level must be debug, info, warn, or error");
        }
    }
};
```

### The Loader

```cpp
// config_loader.h
#pragma once

#include "config.h"
#include <optional>
#include <map>

class ConfigLoader {
public:
    // Load config: defaults → file → env vars → flags
    static Config load(
        const std::string& config_file,
        const std::map<std::string, std::string>& flags = {});
    
private:
    static Config loadFromFile(const std::string& path);
    static void applyEnvironmentVariables(Config& cfg);
    static void applyFlags(Config& cfg, 
        const std::map<std::string, std::string>& flags);
    
    static std::optional<std::string> getEnvVar(const std::string& name);
    static std::optional<uint32_t> parseUint32(const std::string& value);
    static std::optional<uint16_t> parseUint16(const std::string& value);
};
```

### The Implementation

```cpp
// config_loader.cpp
#include "config_loader.h"
#include <fstream>
#include <sstream>
#include <cstdlib>
#include <filesystem>

std::optional<std::string> ConfigLoader::getEnvVar(const std::string& name) {
    if (const char* value = std::getenv(name.c_str())) {
        return std::string(value);
    }
    return std::nullopt;
}

std::optional<uint32_t> ConfigLoader::parseUint32(const std::string& value) {
    try {
        return std::stoul(value);
    } catch (...) {
        return std::nullopt;
    }
}

std::optional<uint16_t> ConfigLoader::parseUint16(const std::string& value) {
    auto parsed = parseUint32(value);
    if (parsed && *parsed <= 65535) {
        return static_cast<uint16_t>(*parsed);
    }
    return std::nullopt;
}

Config ConfigLoader::loadFromFile(const std::string& path) {
    if (!std::filesystem::exists(path)) {
        throw std::runtime_error("config file not found: " + path);
    }
    
    Config cfg;  // Start with defaults
    
    // In a real implementation, use a TOML library (toml++ or cpptoml)
    // For this example, we parse a simple key=value format
    std::ifstream file(path);
    std::string line;
    
    while (std::getline(file, line)) {
        // Skip comments and empty lines
        if (line.empty() || line[0] == '#') continue;
        
        auto eq = line.find('=');
        if (eq == std::string::npos) continue;
        
        std::string key = line.substr(0, eq);
        std::string value = line.substr(eq + 1);
        
        // Trim whitespace (simplified)
        key.erase(0, key.find_first_not_of(" \t"));
        key.erase(key.find_last_not_of(" \t") + 1);
        value.erase(0, value.find_first_not_of(" \t"));
        value.erase(value.find_last_not_of(" \t") + 1);
        
        if (key == "database_url") {
            cfg.database_url = value;
        } else if (key == "max_connections") {
            if (auto v = parseUint32(value)) {
                cfg.max_connections = *v;
            }
        } else if (key == "port") {
            if (auto v = parseUint16(value)) {
                cfg.port = *v;
            }
        } else if (key == "worker_threads") {
            if (auto v = parseUint32(value)) {
                cfg.worker_threads = *v;
            }
        } else if (key == "request_timeout_seconds") {
            if (auto v = parseUint32(value)) {
                cfg.request_timeout_seconds = *v;
            }
        } else if (key == "debug_mode") {
            cfg.debug_mode = (value == "true");
        } else if (key == "log_level") {
            cfg.log_level = value;
        }
    }
    
    return cfg;
}

void ConfigLoader::applyEnvironmentVariables(Config& cfg) {
    // Layer 3: Environment variables override file settings
    if (auto url = getEnvVar("MYAPP_DATABASE_URL")) {
        cfg.database_url = *url;
    }
    
    if (auto conns = getEnvVar("MYAPP_MAX_CONNECTIONS")) {
        if (auto v = parseUint32(*conns)) {
            cfg.max_connections = *v;
        }
    }
    
    if (auto p = getEnvVar("MYAPP_PORT")) {
        if (auto v = parseUint16(*p)) {
            cfg.port = *v;
        }
    }
    
    if (auto t = getEnvVar("MYAPP_WORKER_THREADS")) {
        if (auto v = parseUint32(*t)) {
            cfg.worker_threads = *v;
        }
    }
    
    if (auto timeout = getEnvVar("MYAPP_REQUEST_TIMEOUT_SECONDS")) {
        if (auto v = parseUint32(*timeout)) {
            cfg.request_timeout_seconds = *v;
        }
    }
    
    if (auto debug = getEnvVar("MYAPP_DEBUG_MODE")) {
        cfg.debug_mode = (*debug == "true" || *debug == "1");
    }
    
    if (auto level = getEnvVar("MYAPP_LOG_LEVEL")) {
        cfg.log_level = *level;
    }
}

void ConfigLoader::applyFlags(Config& cfg,
    const std::map<std::string, std::string>& flags) {
    // Layer 4: Command-line flags override everything
    for (const auto& [key, value] : flags) {
        if (key == "database-url") {
            cfg.database_url = value;
        } else if (key == "max-connections") {
            if (auto v = parseUint32(value)) {
                cfg.max_connections = *v;
            }
        } else if (key == "port") {
            if (auto v = parseUint16(value)) {
                cfg.port = *v;
            }
        } else if (key == "worker-threads") {
            if (auto v = parseUint32(value)) {
                cfg.worker_threads = *v;
            }
        } else if (key == "request-timeout-seconds") {
            if (auto v = parseUint32(value)) {
                cfg.request_timeout_seconds = *v;
            }
        } else if (key == "debug-mode") {
            cfg.debug_mode = (value == "true" || value == "1");
        } else if (key == "log-level") {
            cfg.log_level = value;
        }
    }
}

Config ConfigLoader::load(
    const std::string& config_file,
    const std::map<std::string, std::string>& flags) {
    
    // Layer 1: Defaults (Config constructor)
    // Layer 2: Config file
    auto cfg = loadFromFile(config_file);
    
    // Layer 3: Environment variables
    applyEnvironmentVariables(cfg);
    
    // Layer 4: Command-line flags
    applyFlags(cfg, flags);
    
    // Validate the final config
    cfg.validate();
    
    return cfg;
}
```

### Usage

```cpp
// main.cpp
#include "config_loader.h"
#include <iostream>

int main(int argc, char** argv) {
    try {
        // Simulate flag parsing (in reality, use cxxopts or CLI11)
        std::map<std::string, std::string> flags;
        if (argc > 1) {
            flags["port"] = argv[1];
        }
        
        // Load config: defaults → file → env vars → flags
        auto cfg = ConfigLoader::load("config.toml", flags);
        
        std::cout << "Configuration loaded:\n";
        std::cout << "  database_url: " << cfg.database_url << "\n";
        std::cout << "  port: " << cfg.port << "\n";
        std::cout << "  worker_threads: " << cfg.worker_threads << "\n";
        std::cout << "  debug_mode: " << (cfg.debug_mode ? "true" : "false") << "\n";
        
        // Start the application with this config
        // ...
        
    } catch (const std::exception& e) {
        std::cerr << "Configuration error: " << e.what() << "\n";
        return 1;
    }
    
    return 0;
}
```

### Config File

```
# config.toml
database_url = postgres://localhost/mydb
max_connections = 20
port = 8080
worker_threads = 4
request_timeout_seconds = 30
debug_mode = false
log_level = info
```

### Testing

```cpp
// test_config.cpp
#include <cassert>
#include "config_loader.h"

void testDefaultConfig() {
    Config cfg;
    cfg.database_url = "postgres://localhost/mydb";
    cfg.validate();
    assert(cfg.port == 8080);
    assert(cfg.worker_threads == 4);
}

void testValidationFailure() {
    Config cfg;
    cfg.database_url = "";  // Required
    try {
        cfg.validate();
        assert(false);  // Should throw
    } catch (const std::invalid_argument&) {
        // Expected
    }
}

void testValidationRangeCheck() {
    Config cfg;
    cfg.database_url = "postgres://localhost/mydb";
    cfg.max_connections = 2000;  // Out of range
    try {
        cfg.validate();
        assert(false);  // Should throw
    } catch (const std::invalid_argument& e) {
        assert(std::string(e.what()).find("max_connections") != std::string::npos);
    }
}

int main() {
    testDefaultConfig();
    testValidationFailure();
    testValidationRangeCheck();
    std::cout << "All tests passed\n";
    return 0;
}
```

---

## 76.9 Tradeoffs

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Code constants** | Simple, versioned with code, type-safe | Can't change without recompiling; not for deployment-specific values | Development defaults, never-changing policies |
| **Config file only** | Separate from code, human-readable, versionable | No environment-specific override; restart required | Small services, infrequent config changes |
| **Environment variables only** (12-Factor) | Container-native, no file management, clear separation | Hard to express complex config; no schema; env var sprawl | Microservices, containers, cloud-native systems |
| **Layered** (defaults + file + env + flags) | Flexible, respects operator intent, testable, clear contract | More code; mental model takes time to learn | Production systems, tools, services with varied deployments |
| **Dynamic config service** (LaunchDarkly, Consul) | No restart required, gradual rollouts, experiments | Significant complexity, eventual consistency, new failure modes | Frequent rollouts, A/B tests, high-uptime services |
| **Feature flags** (static) | Easy to understand, versioned with code | Dead code paths, requires code deployments to change flags | Infrequent rollouts, simple on/off behavior |
| **Feature flags** (dynamic) | Gradual rollouts, kill switch, no deployment required | External dependency, caching complexity, per-user logic | Frequent releases, A/B testing, feature experiments |

---

## 76.10 Common Misconceptions

| Misconception | Reality |
|---|---|
| "Configuration should be flexible; anything can be config." | No. Configuration is for deployment-time values (database, port). Policy (business rules, algorithms) belongs in code. Mixing them leads to unmaintainable sprawl. |
| "If I make everything configurable, the system is flexible." | Flexible systems are also fragile. Too many config knobs means operators don't understand the effects of changing them. Constraint is a feature. |
| "Environment variables are the only scalable approach." | Environment variables work for microservices but struggle with complex hierarchical config. Combine them with config files. |
| "All config should be secrets." | No. Secrets are passwords, keys, tokens—things that must not be revealed. Configuration is metadata. Don't confuse them. |
| "Feature flags are just for startups / tech companies." | Feature flags are for any system that deploys frequently and values rapid iteration. They are worth the cost if you deploy weekly or more. |
| "Validation is nice-to-have." | Validation is essential. A config error caught at startup is 100x cheaper than one discovered in production. Never defer validation. |
| "My config file format doesn't need a schema." | Every config format needs a schema, even if it is implicit. Make it explicit. It forces you to think about requirements and catches mistakes. |

---

## 76.11 Exercises

1. **Design a configuration system.** Take a service you know (or a realistic example: a web server, a worker, a scheduler). List all the configuration values it needs. Organize them by category (database, server, logging, etc.). Now design the loading strategy: which values come from defaults, which from files, which from env vars? Write the Config struct.

2. **Layering order experiment.** Create a small program with a Config struct. Implement the four layers: defaults, file, env vars, flags. Set different values in each layer. Verify that the layering order is correct (later layers override earlier ones). What happens if you reverse the order?

3. **Validation and error messages.** Write a Config validator with at least five constraints (required fields, range checks, format checks). Test it with invalid configs. Are the error messages clear enough for an operator to fix the problem?

4. **Static vs. dynamic.** Build a small service with static config. Now imagine you need to enable a feature for 10% of requests without restarting. How would you do it? What would the code look like? Would it be worth the complexity?

5. **Secrets handling.** Write a Secrets struct that loads from environment variables. Make it fail loudly if a secret is missing. Test that the secret is never logged or printed in error messages.

6. **Config file formats.** Try three formats: TOML, YAML, and JSON. Load the same config from each. Which format is easiest to read? Which is easiest to parse? Which would you choose for a new system?

7. **Feature flags as config.** Add feature flags to a simple program. Implement them as a config file. Now add logic to make them dynamic: poll a file for changes and update the flags without restarting. What breaks? What becomes easier?

---

## 76.12 Summary

Configuration is the bridge between intent (what the system should do) and reality (what the environment allows). A good configuration system is typed, layered, validated, and observable. Start with defaults baked into code. Layer in config files (for deployment-specific values), environment variables (for container-time overrides), and flags (for immediate testing). Validate the entire config at load time, not when it is used. Separate secrets from configuration; load secrets from a secret store, never from version-controlled files. For most systems, static configuration (read once at startup) is sufficient; move to dynamic configuration only when you feel pain. Feature flags are powerful for gradual rollouts and experimentation, but introduce complexity; use them when the benefit is clear. The highest-leverage work is getting the layering order right: operators learn it once, and it works everywhere. Configuration mistakes will happen, but they should be caught at load time, not at 3 AM in production.

---

> **[← Previous: Hot Reloading](08-hot-reloading.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: Observability and Logging →](10-observability-and-logging.md)**
