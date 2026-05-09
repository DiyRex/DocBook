# Chapter 69 — Designing a Web Server

## Opening

Every HTTP request that reaches your service travels a predictable path: a client initiates a TCP connection, sends bytes, your server parses them, routes to a handler, executes logic, generates a response, and decides whether to keep the connection alive or close it. This chapter walks that entire pipeline — what each piece does, what can go wrong, and which decisions are architectural.

A web server is not magic. It is a state machine for accepting connections and transforming HTTP requests into responses. Unlike a CLI tool (Chapter 68), which processes input sequentially in one invocation, or a database (Chapter 70), which manages persistent state, a web server must handle hundreds or thousands of concurrent connections, each at a different stage of request/response, while remaining responsive. The core challenge is: **how do you organize the code and the concurrency model so that this is both possible and maintainable?**

By the end of this chapter you will understand:

1. The **connection lifecycle** — from TCP accept through HTTP parse, route, handle, response, keep-alive decision.
2. **Concurrency model choices** — thread-per-connection, event-loop, thread pool, async runtime, and their tradeoffs.
3. **HTTP parsing** — request structure, chunked bodies, security pitfalls, and why you should use a library.
4. **Routing** — static dispatch, trie structures, regex matching, and the performance/flexibility tradeoff.
5. **Middleware** — cross-cutting concerns as a pipeline, the onion model, and composition.
6. **Backpressure and limits** — why connection caps, request body limits, and timeouts are architectural decisions, not afterthoughts.
7. **Graceful shutdown** — draining in-flight work without losing requests or causing hangs.
8. A **worked example** — a 150-line C++ web server showing route, handler, middleware, and graceful shutdown.

---

## 69.1 The Connection Lifecycle

A web server processes each connection through a sequence of states:

```
TCP Listen
    ↓
TCP Accept (SYN-ACK)
    ↓ (connection established)
TLS Handshake (optional)
    ↓ (encrypted tunnel, if HTTPS)
Read HTTP Request Line + Headers
    ↓
Validate & Parse Headers
    ↓
Read Request Body (if present)
    ↓
Route to Handler
    ↓
Handler Executes (may access database, call external APIs, compute)
    ↓
Generate Response (status line, headers, body)
    ↓
Write Response to Socket
    ↓
Keep-Alive Decision
    ├─ If Keep-Alive: loop back to "Read HTTP Request Line"
    └─ If Close: TCP FIN, close connection
```

This is a loop. The same connection may carry multiple HTTP requests (HTTP/1.1 keep-alive), or just one (HTTP/1.0, or explicit `Connection: close`).

### State at Each Stage

At each point, the server holds state:

- **TCP Listen:** listening socket, waiting for `accept()` to return a new connection.
- **TCP Accept:** new file descriptor (connection), client address, OS buffers.
- **TLS Handshake:** TLS state machine, certificate, cipher negotiation.
- **Read Request:** byte buffer, partial request, parser state.
- **Parse Headers:** parsed headers table, content-length, transfer encoding.
- **Read Body:** accumulating body bytes, checking against declared size.
- **Route:** matched handler, request parameters, path variables.
- **Execute Handler:** request/response pair, any context the handler needs.
- **Write Response:** response queue, bytes sent, remaining to send.
- **Keep-Alive:** decision flag, connection state, buffer management.

### Failure Modes at Each Stage

Understanding the lifecycle is crucial because failures are not uniform:

| Stage | Failure | Effect | Mitigation |
|-------|---------|--------|------------|
| TCP Accept | Accept fails (OS limit reached) | New connections dropped silently | Bounded connection pool, connection limits |
| TLS Handshake | Handshake timeout | Client hangs or times out; connection stuck | Timeout (typically 30s) |
| Read Request | Partial request, stalled client | Buffer accumulation; memory leak | Read timeout, max request size |
| Parse Headers | Invalid syntax (colon in wrong place) | Malformed request; invalid state | Strict parsing, early rejection |
| Read Body | Content-Length mismatch | Client/server disagreement; connection corruption | Validate length before reading; abort on overflow |
| Route | No handler found | 404; handler must exist | Fallback handler (catch-all), 404 responder |
| Handler Execute | Handler timeout or hangs | Server unavailable for other clients | Handler timeout, cancellation, worker thread isolation |
| Write Response | Socket buffer full | Partial write; incomplete response | Non-blocking writes, buffer draining, backpressure |
| Keep-Alive | Lingering connection | Resource exhaustion | Keep-alive timeout, connection reaping |

---

## 69.2 Concurrency Model Choice

The biggest architectural decision in a web server is **how to handle multiple concurrent connections**. There are four dominant patterns:

### Pattern 1: Thread-Per-Connection (Apache mpm_worker)

One OS thread per active connection.

```
Accept Connection 1  →  spawn Thread 1
Accept Connection 2  →  spawn Thread 2
Accept Connection 3  →  spawn Thread 3

Thread 1: Read → Parse → Handler → Write → Keep-Alive/Close
Thread 2: Read → Parse → Handler → Write → Keep-Alive/Close
Thread 3: Read → Parse → Handler → Write → Keep-Alive/Close
```

**Pros:**
- Simple mental model: one thread, one connection, synchronous code.
- Easy to debug (no state machine, no reactor pattern complexity).
- Handler can block without starving other connections.

**Cons:**
- High memory per connection (thread stack is typically 1-8 MB; 10,000 connections = 10-80 GB).
- Context switch overhead (kernel must schedule threads; latency tail increases).
- Limited concurrency (2000-5000 connections before saturation on a single machine).

**When to use:** Smaller deployments (< 1000 concurrent connections), legacy code, or when handlers routinely block and you cannot refactor to async.

### Pattern 2: Event-Loop (nginx, Node.js, Tokio with single core)

One OS thread, one event loop (see Chapter 52), driven by I/O multiplexing.

```
Event Loop:
  while (true) {
    Poll ready connections (epoll_wait)
    for each ready connection {
      Read available bytes → Parse → Route
      If handler ready → Execute → Write response
      If keep-alive → re-register for read
      Else → close
    }
  }
```

**Pros:**
- Low memory per connection (one entry in epoll/kqueue, a small buffer).
- No context switch overhead (one thread running at a time).
- Scales to 10,000+ concurrent connections on a single core.

**Cons:**
- CPU-bound handlers block the entire loop (no parallelism).
- Must write non-blocking code (no synchronous `read()` or `write()`; use async/await or state machines).
- Single-core only; must run multiple loops for multi-core.

**When to use:** I/O-bound servers (web APIs, proxies, real-time services), when handlers are quick (< 10 ms), and you can't afford context switch overhead.

### Pattern 3: Thread Pool + Non-Blocking I/O (Tomcat, Undertow)

Fixed pool of threads; each thread runs an event loop for a subset of connections.

```
Accept → Assign to Thread Pool Slot 1

Thread Pool:
  [Thread 1, Thread 2, Thread 3, ..., Thread N]
  Each thread runs an event loop:
    Poll assigned connections (epoll_wait on subset of FDs)
    Read → Parse → Route → Execute handler (blocking is OK; other threads are not starved)
    Write response
```

**Pros:**
- Scales to 10,000+ connections (bounded thread pool; fewer threads than connections).
- Handlers can block (one thread blocks; others keep running).
- Multi-core aware (thread pool naturally distributes across cores).

**Cons:**
- Complexity: thread pool management, work-stealing, load balancing.
- Context switch overhead (less than thread-per-connection, but more than single loop).
- Thread pool tuning (too few threads = starvation; too many = thrashing).

**When to use:** Servers with mixed I/O and CPU-bound handlers, moderate concurrency (1000-10,000 connections), Java/JVM ecosystems.

### Pattern 4: Async Runtime (Tokio, Go, Python asyncio on multi-loop)

Language-level coroutines scheduled by a runtime; multiple event loops (one per core).

```
Tokio (Rust):
  [Event Loop 1, Event Loop 2, ..., Event Loop N]
  Each loop runs thousands of async tasks.
  
  Task structure:
    accept() → async_handler() → write_response()
  
  When task hits .await, it yields to the loop.
  If I/O is not ready, task parks; other tasks run.
  When I/O completes, task resumes.
```

**Pros:**
- Very low memory per connection (coroutine stack is ~50 bytes; a few MB for thousands).
- Multi-core by default (runtime spawns one loop per core).
- Language abstracts the state machine (you write "async" code, not event handlers).

**Cons:**
- Language-dependent (Rust, Go, Node.js; not portable to C++).
- Blocking syscalls still block the loop (must use async libraries; no blocking I/O).
- Tooling and debugging can be opaque (stack traces span multiple event loops).

**When to use:** Greenfield projects in languages with good async support (Rust, Go), high concurrency required (100,000+ connections), performance-critical services.

### Tradeoff Summary

| Model | Memory per Conn | Latency (avg) | Max Connections | Blocking OK | Multi-Core | Complexity |
|-------|-----------------|---------------|-----------------|-------------|-----------|-----------|
| Thread-per-conn | 1-8 MB | High (context switch) | 2K-5K | Yes | Yes | Low |
| Event loop | <10 KB | Low (no switch) | 10K+ | No | No (1 loop) | Medium |
| Thread pool | Low (shared) | Medium | 10K-100K | Yes | Yes | High |
| Async runtime | <10 KB | Low | 100K+ | No | Yes | Medium |

---

## 69.3 HTTP Parsing

An HTTP request is a sequence of bytes: a request line, headers, an optional body.

```
GET /api/users?id=42 HTTP/1.1\r\n
Host: example.com\r\n
Content-Length: 10\r\n
\r\n
request-body (10 bytes)
```

### Parsing Phases

1. **Request Line:** `METHOD PATH VERSION`
   - Must extract method (GET, POST, etc.), request-target (path + query), HTTP version.
   - Edge case: request-target can be absolute URI, authority (CONNECT), asterisk (OPTIONS *), or relative path.
   
2. **Headers:** Key-value pairs, terminated by `\r\n\r\n`.
   - Duplicate headers are allowed (multiple `Set-Cookie` values, for example).
   - Header names are case-insensitive; values are case-sensitive.
   - Whitespace handling is strict (leading/trailing spaces must be trimmed).
   
3. **Body:** Bytes following the blank line, sized by:
   - `Content-Length` header (fixed size).
   - `Transfer-Encoding: chunked` (variable-size chunks, prefixed by size).
   - End-of-stream (for responses; reading until connection closes).

### Security Pitfalls

**Header Injection:** A header value with `\r\n` can inject fake headers.

```
// Attacker sends:
GET / HTTP/1.1
Host: example.com
X-Custom: value\r\nX-Injected: hacker

// Parser might extract:
Host: example.com
X-Custom: value
X-Injected: hacker  ← attacker-controlled
```

**Request Smuggling:** Discrepancy between proxy and origin server on where request ends (e.g., `Content-Length` vs `Transfer-Encoding`). A request could be parsed as two requests, or vice versa, allowing cache poisoning.

**Slow Loris:** Attacker sends request headers very slowly (one byte per second). If the server waits for the full request, it ties up a connection and memory. With enough slow clients, the server exhausts its connection pool.

**Large Body:** Attacker sends `Content-Length: 999999999` but no actual body. Server allocates huge buffer or waits forever.

### Why Not Write Your Own Parser

HTTP parsing looks simple but has dozens of edge cases:
- Whitespace rules (space before/after colon?).
- Header folding (multi-line headers with continuation spaces).
- Chunked encoding (chunk size in hex, chunk extensions, trailer headers).
- Pipelining (multiple requests in one packet).
- Authority forms (CONNECT for tunneling).

**Rule: Always use a library.** Popular choices:
- **http_parser** / **llhttp** (C, used by Node.js): small, audited, fast.
- **Boost.Beast** (C++): well-integrated with Asio, HTTP/1.1 + WebSocket support.
- **cpp-httplib** (C++, single-header): minimal, easy to embed.
- **POCO Net** (C++): heavier, full-featured.

A typical parser invocation:

```cpp
#include <llhttp.h>

llhttp_t parser;
llhttp_settings_t settings;

// Set callbacks
settings.on_url = [](llhttp_t* parser, const char* at, size_t len) {
    auto* req = (HttpRequest*)parser->data;
    req->path = std::string(at, len);
    return 0;
};

llhttp_init(&parser, HTTP_REQUEST, &settings);
parser.data = &request;

size_t parsed = llhttp_execute(&parser, buffer, buffer_len);
if (parser.http_errno) {
    // Parse error
}
```

---

## 69.4 Routing

Once a request is parsed, the path must be mapped to a handler. Three strategies:

### Strategy 1: Static Dispatch (Switch Statement)

```cpp
void routeRequest(const HttpRequest& req, HttpResponse& resp) {
    if (req.path == "/") {
        handleRoot(req, resp);
    } else if (req.path == "/api/users") {
        handleListUsers(req, resp);
    } else if (req.path.startsWith("/api/users/")) {
        std::string userId = req.path.substr(11);
        handleGetUser(userId, req, resp);
    } else {
        resp.status = 404;
        resp.body = "Not Found";
    }
}
```

**Pros:** Zero allocation, instant dispatch.
**Cons:** Not scalable (100 routes = 100 if statements); path parameters are error-prone.

### Strategy 2: Trie Structure (Prefix Tree)

Routes are stored in a trie; path components are traversed to find the handler.

```
Route tree:
  /
  ├─ api
  │  └─ users
  │     ├─ [static] → GET handler, POST handler
  │     └─ [id param] → GET handler, PUT handler, DELETE handler
  ├─ [static] → GET handler (root)
  └─ static
     └─ [path...] → file server

Lookup "/api/users/42":
  / → api → users → (param) → found; extract param "42"
```

**Pros:** O(path components) lookup; scales to thousands of routes; efficient parameter extraction.
**Cons:** Moderate complexity; allocation overhead.

Example (simplified):

```cpp
struct TrieNode {
    std::string path_segment;
    bool is_param = false;  // {id} or literal?
    std::vector<std::shared_ptr<TrieNode>> children;
    std::map<std::string, Handler> handlers;  // GET, POST, etc.
};

std::optional<Handler> route(const TrieNode& root, const std::vector<std::string>& segments) {
    auto current = &root;
    for (const auto& seg : segments) {
        // Try exact match
        auto it = std::find_if(current->children.begin(), current->children.end(),
            [&](const auto& child) { return !child->is_param && child->path_segment == seg; });
        if (it != current->children.end()) {
            current = it->get();
            continue;
        }
        
        // Try param match
        it = std::find_if(current->children.begin(), current->children.end(),
            [&](const auto& child) { return child->is_param; });
        if (it != current->children.end()) {
            current = it->get();
            continue;
        }
        
        return std::nullopt;  // No route
    }
    
    return current->handlers.count("GET") ? current->handlers.at("GET") : std::nullopt;
}
```

### Strategy 3: Regex Matching (Express.js, Flask)

Routes are regular expressions; incoming paths are matched against each in order.

```cpp
std::vector<Route> routes = {
    {"GET", "^/$", rootHandler},
    {"GET", "^/api/users$", listUsersHandler},
    {"GET", "^/api/users/([0-9]+)$", getUserHandler},
    {"POST", "^/api/files/.*", uploadHandler},
};

void routeRequest(const HttpRequest& req, HttpResponse& resp) {
    for (const auto& route : routes) {
        if (route.method != req.method) continue;
        
        std::smatch match;
        if (std::regex_match(req.path, match, std::regex(route.pattern))) {
            // Extract captures from match.str(1), match.str(2), ...
            route.handler(req, resp, match);
            return;
        }
    }
    
    resp.status = 404;
}
```

**Pros:** Flexible; compact; matches the mental model (Express developers expect this).
**Cons:** Slower (regex matching per request); regex DoS vulnerability (malicious regex starves server).

### Performance vs Flexibility

- **Performance-critical (API gateway, reverse proxy):** Use a trie. Lookup is O(path length), not O(routes).
- **Flexibility-critical (web framework, CMS):** Use regex, with a performance budget (regex timeout).
- **Simple systems:** Static dispatch is fine for < 20 routes.

---

## 69.5 Middleware and the Onion Model

A middleware is a function that wraps a handler and intercepts the request/response.

```cpp
using Handler = std::function<void(HttpRequest&, HttpResponse&)>;
using Middleware = std::function<void(const Handler&, HttpRequest&, HttpResponse&)>;
```

**Examples:**
- **Auth:** Check credentials; if invalid, return 401.
- **Logging:** Log request (method, path), call handler, log response (status, duration).
- **Compression:** Call handler, compress response body if `Accept-Encoding: gzip`.
- **CORS:** Check origin, add `Access-Control-Allow-Origin` header.
- **Rate Limiting:** Check rate limit; if exceeded, return 429.

### Composing Middleware

The "onion model": middleware layers wrap each other, with the handler at the core.

```
Request → Logging → Auth → CORS → Rate Limit → Handler → Rate Limit → CORS → Auth → Logging → Response

Request enters outer layer first, peels layers inward to reach handler, then response peels back out.
```

Example:

```cpp
auto loggingMiddleware = [](const Handler& next, HttpRequest& req, HttpResponse& resp) {
    auto start = std::chrono::high_resolution_clock::now();
    std::cout << "→ " << req.method << " " << req.path << "\n";
    
    next(req, resp);  // Call the wrapped handler
    
    auto duration = std::chrono::high_resolution_clock::now() - start;
    std::cout << "← " << resp.status << " (" << duration.count() / 1e6 << " ms)\n";
};

auto authMiddleware = [](const Handler& next, HttpRequest& req, HttpResponse& resp) {
    if (req.headers.count("Authorization") == 0) {
        resp.status = 401;
        resp.body = "Unauthorized";
        return;
    }
    next(req, resp);
};

// Compose: auth wraps handler; logging wraps auth
Handler handler = [](HttpRequest& req, HttpResponse& resp) {
    resp.status = 200;
    resp.body = "OK";
};

Handler wrapped = [&](HttpRequest& req, HttpResponse& resp) {
    authMiddleware(handler, req, resp);
};

Handler fully_wrapped = [&](HttpRequest& req, HttpResponse& resp) {
    loggingMiddleware(wrapped, req, resp);
};
```

Request flow: logging enters → auth checks → handler runs → response bubbles up through auth → logging logs → response sent.

### Middleware Order Matters

Order determines behavior:

```
Logging → Auth → Handler:  Logs all requests, including rejected ones (visibility)
Auth → Logging → Handler:   Logs only authorized requests (security)

CORS → Auth:               CORS headers always sent (even if 401)
Auth → CORS:               CORS headers sent only on success (security by obscurity, bad)
```

---

## 69.6 Backpressure and Connection Limits

A web server must limit resource consumption. Without limits, an attacker can exhaust memory, file descriptors, or CPU.

### Limit 1: Maximum Connections

Every TCP connection consumes kernel memory (socket buffer, connection state). The OS enforces a limit (typically thousands per process). Beyond that, `accept()` fails.

**Strategy:** Cap connections explicitly.

```cpp
class Server {
    int max_connections = 1000;
    int current_connections = 0;
    std::mutex conn_lock;
    
    void acceptLoop() {
        while (true) {
            {
                std::lock_guard<std::mutex> lock(conn_lock);
                if (current_connections >= max_connections) {
                    // Sleep; wait for a connection to close
                    continue;
                }
            }
            
            int client_fd = accept(listen_fd, ...);
            if (client_fd < 0) continue;
            
            {
                std::lock_guard<std::mutex> lock(conn_lock);
                current_connections++;
            }
            
            handleConnection(client_fd);  // Decrements current_connections when done
        }
    }
};
```

### Limit 2: Request Body Size

Prevent `Content-Length: 1000000000` attacks.

```cpp
const size_t MAX_BODY_SIZE = 10 * 1024 * 1024;  // 10 MB

void readRequestBody(int fd, HttpRequest& req) {
    size_t content_length = std::stoul(req.headers["Content-Length"]);
    
    if (content_length > MAX_BODY_SIZE) {
        // Reject immediately
        HttpResponse error;
        error.status = 413;
        error.body = "Payload Too Large";
        sendResponse(fd, error);
        return;
    }
    
    // Read safely
    req.body.reserve(content_length);
    while (req.body.size() < content_length) {
        size_t remaining = content_length - req.body.size();
        size_t chunk_size = std::min(remaining, size_t(4096));
        // ...
    }
}
```

### Limit 3: Request Timeout

If a client doesn't send the full request within T seconds, close the connection.

```cpp
void readRequestWithTimeout(int fd, HttpRequest& req, int timeout_sec) {
    auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(timeout_sec);
    
    while (!request_complete) {
        auto now = std::chrono::steady_clock::now();
        if (now > deadline) {
            // Timeout; close connection
            close(fd);
            return;
        }
        
        int remaining_ms = std::chrono::duration_cast<std::chrono::milliseconds>(deadline - now).count();
        
        struct timeval tv = {
            .tv_sec = remaining_ms / 1000,
            .tv_usec = (remaining_ms % 1000) * 1000
        };
        setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
        
        ssize_t n = recv(fd, buffer, sizeof(buffer), 0);
        // ... accumulate in parser
    }
}
```

### Limit 4: Handler Timeout

If a handler takes longer than T seconds, cancel it.

**In a thread-based server:**

```cpp
void handleConnectionWithTimeout(int client_fd) {
    HttpRequest req = parseRequest(client_fd);
    HttpResponse resp;
    
    auto handler_thread = std::thread([&](){ router.handle(req, resp); });
    
    bool completed = handler_thread.wait_for(std::chrono::seconds(30));
    if (!completed) {
        // Handler timed out; can't stop it (C++ has no thread cancellation)
        // but we can:
        resp.status = 504;
        resp.body = "Gateway Timeout";
        sendResponse(client_fd, resp);
        // Handler thread keeps running in background (leak)
    } else {
        sendResponse(client_fd, resp);
    }
}
```

**In an event-loop server (Rust/Go):**

```rust
// Tokio
let result = tokio::time::timeout(
    Duration::from_secs(30),
    handler(req)
).await;

match result {
    Ok(resp) => send_response(resp).await,
    Err(_) => send_error(504, "Timeout").await,
}
```

---

## 69.7 Graceful Shutdown

When the server receives a shutdown signal (SIGTERM, Ctrl+C), it must:

1. **Stop accepting new connections.**
2. **Wait for in-flight requests to complete.**
3. **Close existing connections.**
4. **Exit cleanly.**

Without graceful shutdown, in-flight requests are aborted, clients get 503 errors, and data may be corrupted.

### Naive Approach (Bad)

```cpp
void main() {
    Server server;
    server.start();
    
    // Blocks indefinitely
    // Ctrl+C → exit immediately (in-flight requests are terminated)
}
```

### Proper Shutdown

```cpp
class Server {
    std::atomic<bool> shutdown_requested = false;
    std::unordered_set<int> active_connections;
    std::mutex conn_lock;
    
    void acceptLoop() {
        while (!shutdown_requested) {
            int client_fd = accept(listen_fd, ...);
            if (client_fd < 0) continue;
            
            {
                std::lock_guard<std::mutex> lock(conn_lock);
                active_connections.insert(client_fd);
            }
            
            // Spawn handler (thread or async task)
            spawn([this, client_fd] { handleConnection(client_fd); });
        }
    }
    
    void handleConnection(int client_fd) {
        try {
            // Read, parse, route, handle, write
            HttpRequest req = parseRequest(client_fd);
            HttpResponse resp;
            router.handle(req, resp);
            sendResponse(client_fd, resp);
        } catch (const std::exception& e) {
            // Log error
        }
        
        close(client_fd);
        {
            std::lock_guard<std::mutex> lock(conn_lock);
            active_connections.erase(client_fd);
        }
    }
    
    void gracefulShutdown(int timeout_sec) {
        // 1. Stop accepting new connections
        shutdown_requested = true;
        close(listen_fd);  // Make accept() fail, exit loop
        
        // 2. Wait for in-flight to complete (with timeout)
        auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(timeout_sec);
        while (true) {
            {
                std::lock_guard<std::mutex> lock(conn_lock);
                if (active_connections.empty()) {
                    break;  // All connections drained
                }
            }
            
            if (std::chrono::steady_clock::now() > deadline) {
                std::cerr << "Shutdown timeout; forcing close\n";
                {
                    std::lock_guard<std::mutex> lock(conn_lock);
                    for (int fd : active_connections) {
                        close(fd);
                    }
                }
                break;
            }
            
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
};

// Main
int main() {
    Server server;
    
    std::thread accept_thread([&]{ server.acceptLoop(); });
    
    // Wait for signal
    std::signal(SIGTERM, [](int) { /* global flag */ });
    
    // In signal handler or main loop:
    server.gracefulShutdown(30);  // 30-second timeout
    accept_thread.join();
    
    std::cout << "Server shut down cleanly\n";
    return 0;
}
```

**Key insight:** Graceful shutdown requires tracking in-flight work and waiting (with a timeout) for it to complete.

---

## 69.8 Worked Example: A 150-Line C++ Web Server

Here is a minimal but complete web server using Boost.Beast (HTTP parsing) and Boost.Asio (event-driven I/O):

```cpp
// minimal_server.cpp
#include <iostream>
#include <string>
#include <memory>
#include <functional>
#include <map>
#include <boost/asio.hpp>
#include <boost/enable_shared_from_this.hpp>

using boost::asio::ip::tcp;
using Handler = std::function<void(const std::string&, std::string&)>;

// Route table
struct Router {
    std::map<std::string, Handler> routes;
    
    void add(const std::string& path, Handler h) {
        routes[path] = h;
    }
    
    bool handle(const std::string& path, std::string& body) {
        auto it = routes.find(path);
        if (it != routes.end()) {
            it->second(path, body);
            return true;
        }
        return false;
    }
};

// Simple middleware
Handler loggingMiddleware(Handler next) {
    return [next](const std::string& path, std::string& body) {
        std::cout << "→ GET " << path << "\n";
        next(path, body);
        std::cout << "← 200\n";
    };
}

class Connection : public std::enable_shared_from_this<Connection> {
    tcp::socket socket_;
    enum { MAX_LENGTH = 4096 };
    char data_[MAX_LENGTH];
    Router& router_;
    
public:
    typedef std::shared_ptr<Connection> pointer;
    
    static pointer create(boost::asio::io_context& io_context, Router& router) {
        return pointer(new Connection(io_context, router));
    }
    
    tcp::socket::lowest_layer_type& socket() {
        return socket_.lowest_layer();
    }
    
    void start() {
        socket_.async_read_some(
            boost::asio::buffer(data_, MAX_LENGTH),
            boost::bind(&Connection::handle_read, shared_from_this(),
                boost::asio::placeholders::error,
                boost::asio::placeholders::bytes_transferred)
        );
    }
    
private:
    Connection(boost::asio::io_context& io_context, Router& router)
        : socket_(io_context), router_(router) {}
    
    void handle_read(const boost::system::error_code& error, size_t bytes_transferred) {
        if (!error) {
            std::string request(data_, bytes_transferred);
            
            // Minimal parsing: extract path from "GET /path HTTP/1.1"
            size_t space1 = request.find(' ');
            size_t space2 = request.find(' ', space1 + 1);
            std::string path = request.substr(space1 + 1, space2 - space1 - 1);
            
            // Route
            std::string body = "Not Found";
            int status = 404;
            if (router_.handle(path, body)) {
                status = 200;
            }
            
            // Build response
            std::string response =
                "HTTP/1.1 " + std::to_string(status) + " OK\r\n"
                "Content-Type: text/plain\r\n"
                "Content-Length: " + std::to_string(body.size()) + "\r\n"
                "\r\n" + body;
            
            // Write
            boost::asio::async_write(
                socket_,
                boost::asio::buffer(response),
                boost::bind(&Connection::handle_write, shared_from_this(),
                    boost::asio::placeholders::error)
            );
        }
    }
    
    void handle_write(const boost::system::error_code& error) {
        if (!error) {
            // Keep-alive: read next request
            start();
        }
    }
};

class Server {
    boost::asio::io_context io_context_;
    tcp::acceptor acceptor_;
    Router router_;
    
public:
    Server(int port) : acceptor_(io_context_, tcp::endpoint(tcp::v4(), port)) {
        // Register handlers
        router_.add("/", [](const std::string& p, std::string& b) {
            b = "Hello, World!";
        });
        
        router_.add("/api/status", [](const std::string& p, std::string& b) {
            b = "OK";
        });
        
        start_accept();
    }
    
    void run() {
        io_context_.run();
    }
    
private:
    void start_accept() {
        Connection::pointer new_connection = Connection::create(io_context_, router_);
        
        acceptor_.async_accept(
            new_connection->socket(),
            boost::bind(&Server::handle_accept, this, new_connection,
                boost::asio::placeholders::error)
        );
    }
    
    void handle_accept(Connection::pointer new_connection,
                       const boost::system::error_code& error) {
        if (!error) {
            new_connection->start();
        }
        start_accept();
    }
};

int main() {
    try {
        Server server(8080);
        std::cout << "Listening on port 8080...\n";
        server.run();
    } catch (std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
    }
    return 0;
}
```

**To build and run:**

```bash
g++ -std=c++11 -o server minimal_server.cpp -lboost_system -pthread
./server &
curl http://localhost:8080/
# Output: Hello, World!
curl http://localhost:8080/api/status
# Output: OK
```

**What this shows:**
1. **Async I/O:** Each connection is handled without blocking others (event-loop pattern via Asio).
2. **Routing:** Simple map-based dispatcher.
3. **Keep-alive:** After writing a response, the handler re-enters `start()` to read the next request.
4. **Middleware:** Trivial to add (wrap the handler function).
5. **Error handling:** 404 response for unregistered paths.

**What it lacks:**
- Proper HTTP parsing (we manually split on spaces).
- Request body handling.
- Header parsing.
- Timeouts.
- Graceful shutdown (Ctrl+C will exit abruptly).
- TLS/HTTPS.

A production server would add all of these.

---

## 69.9 What You Don't Build Yourself

The surface area of a web server is small. The surface area of the protocols it speaks is enormous.

### Don't Build: TLS

TLS (HTTPS) is cryptographic protocol requiring years of security review. Use OpenSSL or Rustls.

```cpp
// With Boost.Asio + OpenSSL
typedef boost::asio::ssl::stream<tcp::socket> ssl_socket;
ssl_socket socket(io_context, ssl_context);
// ... TLS handshake is automatic
```

### Don't Build: HTTP/2 Framing

HTTP/2 is a binary protocol with complex framing: frames, streams, priority, flow control. Use a library (nghttp2 for C++).

```cpp
#include <nghttp2/nghttp2.h>
// ... frame parsing and priority negotiation are delegated
```

### Don't Build: WebSockets

WebSocket is a protocol upgrade on top of HTTP. Use a library (Beast for C++, ws for Node.js).

### Don't Build: Cookie Parsing and Validation

Cookies have complex rules: domain, path, secure flag, SameSite, expiration. Use a library or framework.

### You *Do* Build

- **Routing:** Framework-specific; tailored to your application.
- **Handlers:** Business logic; core of your application.
- **Middleware chain:** Application concerns (auth, logging, rate limiting).
- **Graceful shutdown:** Application-specific resource cleanup.
- **Error handling:** Application-specific error responses.

---

## 69.10 Common Misconceptions

**"A web server must use an event loop."**

No. Thread-per-connection (Apache) works fine for < 1000 concurrent connections and is simpler. Event loops are an optimization for high concurrency. Use the right tool for your scale.

**"Async/await is always faster than threads."**

No. Async has less overhead per task; threads have higher overhead but fewer moving parts. At small scales (< 100 concurrent tasks), threads are often faster in practice. Async shines at 10,000+.

**"Keep-alive means reusing the same handler instance."**

No. Keep-alive reuses the TCP connection; each request gets a fresh handler invocation. The connection state is transport-level, not application-level.

**"Middleware can be inserted or removed at runtime without consequences."**

Only if the middleware is non-ordering. If middleware A depends on middleware B running first (e.g., auth before logging), reordering breaks it. Document middleware ordering.

**"Request timeouts prevent Slow Loris attacks entirely."**

Partial mitigation. A request timeout of 30 seconds still allows an attacker to send one byte every second. Better: aggressively close connections that aren't making progress (e.g., no byte received for 5 seconds).

**"Graceful shutdown means waiting forever."**

No. Always set a timeout (typically 30 seconds). After the timeout, force-close remaining connections. Infinite waits are as bad as immediate shutdown — they cause deployment hangs.

---

## 69.11 Exercises

1. **Measure concurrency limits.** Write a test that opens 1000 concurrent connections to a thread-per-connection server and to an event-loop server. Plot response latency vs number of concurrent connections. At what point does each server's latency degrade?

2. **Parse a request.**Write a minimal HTTP request parser that extracts method, path, and headers. Feed it malformed input (missing colons, invalid chunks, header injection attempts). Verify it rejects correctly.

3. **Implement a route trie.** Implement a trie-based router supporting:
   - Literal paths: `/api/users`
   - Path parameters: `/api/users/{id}`
   - Catch-all: `/static/{path...}`
   Benchmark against regex-based routing for 1000 routes.

4. **Middleware chain.** Write a logging middleware, an auth middleware, and a rate-limiting middleware. Compose them in different orders and observe how behavior changes. Document the required order.

5. **Graceful shutdown simulation.** Implement a server that tracks in-flight requests. Simulate shutdown: stop accepting connections, wait for in-flight to complete, with a configurable timeout. Inject slow handlers (100ms) and verify the server waits for them.

6. **Slow Loris simulation.** Write a client that sends HTTP request headers one byte per second. Connect 100 such clients simultaneously. If the server has a per-connection timeout of 10 seconds, how many connections does it hold before exhausting the limit? How long does it take?

---

## 69.12 Summary

A web server is a state machine that accepts connections, parses HTTP requests, routes to handlers, executes them, and returns responses. The biggest architectural choice is the concurrency model: thread-per-connection is simple but doesn't scale; event-loops scale but require non-blocking code; thread pools offer a middle ground; async runtimes give the best of both if your language supports them.

HTTP parsing is complex and security-critical — always use a library. Routing can be simple (switch statement) or sophisticated (trie or regex), depending on scale. Middleware chains elegantly handle cross-cutting concerns as long as order is documented and enforced.

Backpressure (connection limits, body size limits, timeouts) and graceful shutdown are not afterthoughts — they are architectural requirements. A production server must handle resource exhaustion and shutdown without losing requests or hanging indefinitely.

The scope of what you build is intentionally small: routing, handlers, middleware, shutdown. The scope of what you *don't* build is large: TLS, HTTP/2, WebSockets, cookie handling. Know the difference, use libraries wisely, and your server will be both maintainable and secure.

---

> **[← Previous: Designing a CLI Tool](01-designing-a-cli-tool.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: A Database Layer →](03-a-database-layer.md)**
