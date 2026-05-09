# Chapter 79 — Networking Internals

## Opening

Every TCP connection your code creates touches dozens of layers — sockets, the kernel network stack, NIC drivers, the wire, and back. Most developers treat TCP as a black box: you call `connect`, data magically arrives at the other end, and you assume delivery happened. But in production, that assumption breaks. Packets are lost. Handshakes time out. Buffers overflow. Connections stall. This chapter walks the path from your `send()` call to the physical network interface and back, so you can reason about latency, throughput, and failures from a place of understanding rather than guesswork.

By the end of this chapter, you will understand:

1. The **socket API** — how `socket`, `bind`, `listen`, `accept`, and `connect` construct a network endpoint, and why they are blocking by default.
2. The **TCP three-way handshake** — SYN, SYN-ACK, ACK, and what it costs in round-trip time.
3. **Send and receive buffers** — why `send` returns quickly (it copies to a kernel buffer), and why buffer sizing matters.
4. The **listen backlog** — the queue of accepted connections, and why it can overflow in overload.
5. **Nagle's algorithm** — why small writes are batched, and how `TCP_NODELAY` disables it.
6. **I/O multiplexing** — `select`, `poll`, `epoll`, `kqueue`, and how they let one thread serve thousands of connections (forward to Chapter 52, Event Loops).
7. **Zero-copy techniques** — `sendfile()` and `splice()`, and why a static-file server can saturate a network without touching the CPU.
8. **TCP vs UDP vs QUIC** — when to pick each, and what trade-offs each makes.

---

## 79.1 The Socket API

A **socket** is an endpoint for network communication. On Unix, it is represented as a file descriptor. The socket API is small and elegant:

- `socket(domain, type, protocol)` — create a socket.
- `bind(fd, address, length)` — assign a local address and port.
- `listen(fd, backlog)` — mark as accepting incoming connections.
- `accept(fd, ...)` — block until a client connects; return a new FD.
- `connect(fd, address, length)` — initiate a connection to a server.
- `send(fd, buffer, length, flags)` — transmit bytes.
- `recv(fd, buffer, length, flags)` — receive bytes.
- `close(fd)` — close the socket.

### A Simple TCP Echo Server

Here is a minimal 30-line echo server that demonstrates the socket API in practice:

```cpp
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main() {
    // Create a TCP socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    
    // Bind to localhost:8000
    struct sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(8000);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    
    // Listen for incoming connections
    listen(listen_fd, 128);
    std::cout << "Listening on 127.0.0.1:8000\n";
    
    // Accept loop
    while (true) {
        struct sockaddr_in peer = {};
        socklen_t peer_len = sizeof(peer);
        
        int client_fd = accept(listen_fd, (struct sockaddr*)&peer, &peer_len);
        if (client_fd < 0) {
            perror("accept");
            continue;
        }
        
        std::cout << "Client connected from " 
                  << inet_ntoa(peer.sin_addr) << ":" 
                  << ntohs(peer.sin_port) << "\n";
        
        // Echo data back
        char buf[4096];
        ssize_t n;
        while ((n = recv(client_fd, buf, sizeof(buf), 0)) > 0) {
            send(client_fd, buf, n, 0);
        }
        
        close(client_fd);
    }
    
    close(listen_fd);
    return 0;
}
```

**Key observations:**

- `socket()` creates an endpoint. `AF_INET` means IPv4; `SOCK_STREAM` means TCP.
- `bind()` assigns a local address (port) so clients know where to connect.
- `listen(fd, backlog)` transitions the socket to *listening* state. The `backlog` parameter sets the queue size for completed handshakes (see section 79.4).
- `accept()` blocks until a client connects, then returns a *new* FD for that connection. The listening FD is reused.
- `recv()` and `send()` are blocking by default: `recv` waits until data arrives; `send` waits until the kernel can buffer it.
- `close()` terminates the connection.

### Non-Blocking Sockets

By default, sockets are **blocking**: `send` waits if the kernel's send buffer is full, and `recv` waits if no data has arrived. For high-concurrency servers, this is impractical — you cannot spawn a thread per connection.

To make a socket non-blocking:

```cpp
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

Now `recv` returns immediately with `EAGAIN` or `EWOULDBLOCK` if no data is available, and `send` returns immediately with `EAGAIN` if the buffer is full. The caller can then use `select`, `poll`, or `epoll` to wait until the socket becomes ready.

---

## 79.2 The TCP Three-Way Handshake

Before any data flows, TCP performs a **three-way handshake**:

```
Client                           Server

                    SYN
                    (seq=x)
            -------->
                                  SYN-ACK
                                  (seq=y, ack=x+1)
            <--------
                    ACK
                    (seq=x+1, ack=y+1)
            -------->
[Connection established]
```

1. **Client sends SYN.** The client picks a random sequence number `x` and sends a segment with the SYN flag set and no payload.
2. **Server sends SYN-ACK.** The server picks its own sequence number `y`, acknowledges the client's sequence (`ack=x+1`), and sets SYN.
3. **Client sends ACK.** The client acknowledges the server's sequence (`ack=y+1`).

At this point the connection is established and both sides can exchange data.

### The RTT Cost

Each step is a round trip over the network. On a LAN, a round-trip time (RTT) is ~1 millisecond; on a continental network, ~50 milliseconds; on an intercontinental link, ~150 milliseconds. **The handshake costs 1 RTT before any data flows.**

This is why:
- HTTP/2 and HTTP/3 prioritize **connection reuse**: opening one connection and pipelining many requests is far faster than opening a new connection per request.
- **TLS adds 1–2 RTTs** (client hello → server hello → encrypted handshake → ready). Unless 0-RTT is negotiated, every HTTPS connection starts with a 2–3 RTT penalty.
- **TCP Fast Open (TFO)** and **QUIC's 0-RTT** try to send data before the handshake completes, cutting this cost.

For interactive protocols like chat, the latency impact is severe: a client sending a message must wait for the handshake before sending the first byte. This is why persistent connections are critical.

---

## 79.3 Send and Receive Buffers

When you call `send(fd, buf, 1000, 0)`, the kernel does *not* immediately transmit the bytes over the network. Instead:

1. The kernel **copies** your 1000 bytes into the socket's **send buffer** (kernel memory).
2. `send()` returns immediately with 1000 (or less if the buffer is full).
3. The kernel's TCP stack later constructs **segments** from the send buffer (up to ~1460 bytes per segment, accounting for IP/TCP headers), adds TCP headers, checksums, and sequence numbers, and transmits them over the network.

Similarly, when data arrives from the network:

1. The NIC receives the bytes.
2. The NIC driver copies them into the socket's **receive buffer** (kernel memory).
3. Your code calls `recv()`, which copies from the kernel buffer into your user-space buffer and returns.

This is why `send()` returning does not mean "the bytes reached the server"; it means "the kernel accepted my bytes." The actual transmission happens asynchronously.

### Buffer Sizes

Each socket has two tunable buffer sizes:

```cpp
int sndbuf_size = 128 * 1024;  // 128 KB send buffer
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &sndbuf_size, sizeof(sndbuf_size));

int rcvbuf_size = 128 * 1024;  // 128 KB receive buffer
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf_size, sizeof(rcvbuf_size));
```

**Why buffer size matters:**

- **Small buffers** mean `send()` can fail with `EAGAIN` if you're sending faster than the network can drain. This creates backpressure: you must slow down or handle the error.
- **Large buffers** let you queue more data, but consume kernel memory. On a server with 10,000 connections, a 1 MB send buffer per connection = 10 GB of RAM just for buffers.
- **Too large** and you risk losing data on crash: if the server dies before the kernel finishes transmitting queued data, those bytes are gone forever.

For high-throughput, low-latency scenarios, the optimal size depends on the **bandwidth-delay product**: `BDP = bandwidth × RTT`. For a 10 Gbps network with 50 ms RTT:

```
BDP = (10,000,000,000 bits/sec) × 0.050 sec
    = 500,000,000 bits
    = 62.5 MB
```

A send buffer much smaller than this will saturate below the network's maximum throughput.

---

## 79.4 Backlog and Accept Queue

When you call `listen(fd, backlog)`, the `backlog` parameter sets the maximum length of a **queue of completed connections** waiting to be accepted.

Here's what happens:

1. A client sends SYN.
2. The server's kernel completes the handshake (sends SYN-ACK, receives ACK).
3. The completed connection is queued in the *accept queue*, capped at `backlog`.
4. Your code calls `accept()`, which dequeues a connection.

If the accept queue is full and a new SYN arrives, the kernel's behavior depends on the OS and sysctls. On Linux:

- By default, the SYN is **dropped** (the client later retransmits it).
- Some versions defer to `/proc/sys/net/ipv4/tcp_syncookies`, which uses a cryptographic trick to avoid storing the connection at all.

### A Common Production Bug

Imagine a web server that calls `accept()` and then spends 100 ms processing the connection (writing a large response, waiting for a database query). The kernel's accept queue fills. New clients' SYNs are dropped. Those clients time out and retry, adding more load.

**Fix:**

1. Call `accept()` in a tight loop on a separate thread, and queue the connection for processing.
2. Or use `SO_REUSEADDR` and create multiple listening sockets, each with its own accept loop.
3. Or tune `/proc/sys/net/ipv4/somaxconn` on Linux (cap on the accept queue size).

---

## 79.5 Nagle's Algorithm and TCP_NODELAY

**Nagle's algorithm** (RFC 896) is a TCP feature that batches small writes to improve efficiency on slow networks. The rule:

- If the send buffer has unacknowledged data, hold off sending a new segment until either (a) the previous segment is acknowledged, or (b) the send buffer is full.

**Why it exists:** On old networks (1980s ARPANET), sending many small packets was expensive. Batching reduced packet count.

**Why it hurts latency today:** A chat application where each keystroke sends a 1-byte message will stall waiting for a previous segment to be acknowledged. On a 50 ms RTT network, each keystroke waits up to 50 ms. Unacceptable.

### Disabling Nagle

```cpp
int nodelay = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
```

**When to use `TCP_NODELAY`:**

- **Interactive protocols** (chat, games, remote shell): set it. Low latency matters more than packet efficiency.
- **Bulk transfer** (file upload, database replication): leave it off. Nagle batches small writes efficiently.
- **HTTP**: HTTP/2 and HTTP/3 don't care because they use persistent connections and multiplexing. Legacy HTTP/1.1 with persistent connections should set it to avoid stalling on small responses.

---

## 79.6 I/O Multiplexing

To serve thousands of concurrent connections, blocking sockets are impractical. Instead, use **I/O multiplexing**: one thread waits on multiple sockets, and the kernel reports which ones are ready.

Chapter 52 (Event Loops) covers this extensively. Here is a brief overview:

### select()

The oldest API:

```cpp
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(listen_fd, &readfds);
// ... add other FDs ...

struct timeval timeout = {.tv_sec = 1, .tv_usec = 0};
int nready = select(max_fd + 1, &readfds, NULL, NULL, &timeout);

for (int fd = 0; fd < max_fd + 1; ++fd) {
    if (FD_ISSET(fd, &readfds)) {
        // fd is ready for reading
    }
}
```

**Problem:** O(n) per call; iterating through all FDs to find ready ones is slow. Limited to ~1024 FDs on some systems.

### poll()

Slightly better:

```cpp
struct pollfd fds[1024];
fds[0].fd = listen_fd;
fds[0].events = POLLIN;
// ... populate more FDs ...

int nready = poll(fds, num_fds, timeout_ms);

for (int i = 0; i < num_fds; ++i) {
    if (fds[i].revents & POLLIN) {
        // fds[i].fd is readable
    }
}
```

Still O(n), but scales better and supports more than 1024 FDs.

### epoll (Linux)

Modern, efficient API:

```cpp
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[128];
int nready = epoll_wait(epfd, events, 128, timeout_ms);

for (int i = 0; i < nready; ++i) {
    int fd = events[i].data.fd;
    // fd is ready
}
```

**Efficiency:** O(1) per ready FD, not per total FD. The kernel maintains an internal ready list. When a packet arrives for a registered FD, the kernel adds it to the ready list. `epoll_wait` returns only the ready FDs.

**Edge-triggered vs Level-triggered:**

- **Level-triggered** (default): `epoll_wait` returns an FD as ready *every time* you call it until the condition clears. Safe but can cause redundant wakeups.
- **Edge-triggered** (`EPOLLET` flag): `epoll_wait` returns an FD only *once*, when it transitions from not-ready to ready. Efficient but requires careful handling: if you don't drain all data, you miss the next batch.

### kqueue (BSD/macOS)

Similar efficiency to epoll:

```cpp
int kq = kqueue();

struct kevent ev;
EV_SET(&ev, listen_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &ev, 1, NULL, 0, NULL);

struct kevent events[128];
int nready = kevent(kq, NULL, 0, events, 128, NULL);

for (int i = 0; i < nready; ++i) {
    int fd = events[i].ident;
    // fd is ready
}
```

More general than epoll: can register timers, signals, and filesystem events in one API.

---

## 79.7 Zero-Copy Techniques

When your application sends a static file (HTML, image, etc.), the traditional path is:

1. `read()` from disk into user-space buffer.
2. Your code calls `send()`.
3. Kernel copies from user buffer into socket send buffer.
4. Kernel transmits over network.

Three data copies. On a saturated network link, this uses significant CPU.

### sendfile()

`sendfile()` (or `sendfile64()` on some systems) bypasses user space:

```cpp
off_t offset = 0;
ssize_t sent = sendfile(socket_fd, file_fd, &offset, file_size);
// The kernel reads from file_fd starting at offset,
// and transmits directly to socket_fd.
// offset is updated.
```

**Benefits:**

- One less data copy (kernel to user space is eliminated).
- CPU overhead drops dramatically.
- On systems with hardware offload (NIC can DMA directly from page cache), zero copies are possible.

A static-file web server using `sendfile()` can saturate a 100 Gbps NIC almost entirely with kernel/NIC doing the work. The application thread barely consumes CPU.

### splice()

More general than `sendfile()`: move data between any two file descriptors (files, sockets, pipes) without touching user space:

```cpp
ssize_t moved = splice(infd, NULL, outfd, NULL, 4096, SPLICE_F_MOVE);
```

Useful for proxies and load balancers that forward traffic without inspecting it.

---

## 79.8 TCP vs UDP vs QUIC

### TCP (Transmission Control Protocol)

- **Reliable**: lost segments are retransmitted.
- **Ordered**: segments arrive in order (duplicates removed, reordered segments buffered until gaps are filled).
- **Slow start**: congestion control gradually increases sending rate; the handshake adds latency.
- **Typical latency**: ~1–2 RTTs minimum (handshake + first data).
- **Use for**: HTTP, SMTP, SSH, FTP — anything where correctness matters more than speed.

### UDP (User Datagram Protocol)

- **Unreliable**: datagrams can be lost, duplicated, or reordered.
- **No handshake**: send immediately.
- **Low latency**: minimal overhead.
- **Typical latency**: ~0.5 RTTs (one-way transmission).
- **Use for**: DNS, online games, video streaming, VoIP — where loss is tolerable and latency matters.

### QUIC (Quick UDP Internet Connections)

QUIC runs on top of UDP but provides TCP-like guarantees:

- **Reliable and ordered**: like TCP.
- **0-RTT**: data can be sent before the handshake completes (if the client has cached parameters from a previous connection).
- **Multiplexed streams**: multiple streams share one connection; packet loss on one stream doesn't block others.
- **Uses UDP**: no kernel changes needed; IP fragmentation concerns avoided.
- **Typical latency**: ~0.5–1 RTTs depending on 0-RTT availability.
- **HTTP/3 uses QUIC** exclusively.

**Trade-off matrix:**

| Feature | TCP | UDP | QUIC |
|---------|-----|-----|------|
| Reliable | Yes | No | Yes |
| Ordered | Yes | No | Yes |
| Handshake | 1 RTT | None | 1 RTT (or 0 with prior data) |
| Streams | Single | N/A | Multiple |
| CPU overhead | Moderate | Low | Low |
| Kernel support | Native | Native | User-space |

---

## 79.9 Worked Example: A 10K Concurrent TCP Server with epoll

Here is a complete TCP echo server handling 10,000 concurrent connections using `epoll` and non-blocking sockets:

```cpp
#include <iostream>
#include <unordered_map>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <errno.h>

struct Connection {
    int fd;
    std::string buffer;
};

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void enable_nodelay(int fd) {
    int nodelay = 1;
    setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
}

int main() {
    int epfd = epoll_create1(0);
    std::unordered_map<int, Connection*> connections;
    
    // Create listening socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
    
    struct sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8000);
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 1024);
    set_nonblocking(listen_fd);
    
    // Register listener
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
    
    std::cout << "Server listening on 0.0.0.0:8000\n";
    
    // Main event loop
    struct epoll_event events[128];
    uint64_t total_connections = 0;
    
    while (true) {
        int nready = epoll_wait(epfd, events, 128, 1000);
        
        for (int i = 0; i < nready; ++i) {
            int fd = events[i].data.fd;
            
            if (fd == listen_fd) {
                // Accept new connection
                struct sockaddr_in peer = {};
                socklen_t peer_len = sizeof(peer);
                
                while (true) {
                    int client_fd = accept(listen_fd, (struct sockaddr*)&peer, &peer_len);
                    if (client_fd < 0) {
                        if (errno != EAGAIN && errno != EWOULDBLOCK) {
                            perror("accept");
                        }
                        break;
                    }
                    
                    set_nonblocking(client_fd);
                    enable_nodelay(client_fd);
                    
                    auto* conn = new Connection{client_fd, ""};
                    connections[client_fd] = conn;
                    
                    struct epoll_event ev2;
                    ev2.events = EPOLLIN;
                    ev2.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev2);
                    
                    total_connections++;
                    if (total_connections % 100 == 0) {
                        std::cout << "Total connections: " << total_connections 
                                  << ", Active: " << connections.size() << "\n";
                    }
                }
            } else {
                // Handle client data
                Connection* conn = connections[fd];
                char buf[4096];
                
                while (true) {
                    ssize_t n = recv(fd, buf, sizeof(buf), 0);
                    
                    if (n > 0) {
                        // Echo back
                        ssize_t sent = 0;
                        while (sent < n) {
                            ssize_t s = send(fd, buf + sent, n - sent, 0);
                            if (s < 0) {
                                if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                                perror("send");
                                goto close_conn;
                            }
                            sent += s;
                        }
                    } else if (n == 0) {
                        // Client closed
                        goto close_conn;
                    } else if (errno != EAGAIN && errno != EWOULDBLOCK) {
                        // Error
                        perror("recv");
                        goto close_conn;
                    }
                    break;
                }
                continue;
                
            close_conn:
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
                delete conn;
                connections.erase(fd);
            }
        }
    }
    
    close(listen_fd);
    close(epfd);
    return 0;
}
```

**To measure latency with wrk:**

```bash
g++ -o server server.cpp
./server &

# In another terminal
wrk -c 100 -t 4 -d 30s http://localhost:8000/
```

`wrk` will report latency percentiles. With `TCP_NODELAY` enabled, latency should be in the low milliseconds. Disabling it (remove the `enable_nodelay` call) will degrade latency noticeably, especially under light load.

---

## 79.10 Tradeoffs

| Approach | Pros | Cons | When to use |
|----------|------|------|------------|
| **Blocking sockets + threads** | Simple mental model; one thread per connection | High memory (~1 MB per thread); context switch overhead | <1000 concurrent connections |
| **Non-blocking + select/poll** | Better than threads for moderate scale | Still O(n); hard to debug | 1000–10000 connections on older systems |
| **Non-blocking + epoll/kqueue** | O(1) scalability; handles 100k+ connections | More complex code; must handle EAGAIN | Modern production servers |
| **Async + io_uring** | Lowest syscall overhead; saturates NIC bandwidth | Linux-only; newest (still evolving) | Extreme-performance scenarios (finance, CDNs) |
| **sendfile + zero-copy** | Minimal CPU for static files | Doesn't help with dynamic content | Static file servers, proxies |
| **UDP + custom reliability** | Low latency; no handshake overhead | Must handle loss, reordering, duplicates | Real-time gaming, VoIP |
| **QUIC** | TCP reliability + 0-RTT + multiplexing | User-space implementation (slower); newer (adoption risk) | Modern web (HTTP/3); long-lived connections |

---

## 79.11 Common Misconceptions

**"TCP guarantees delivery."**

TCP guarantees *in-order, best-effort* delivery. If both endpoints crash simultaneously, data in flight can be lost. If the server crashes and reboots, it forgets about sockets that were in its send buffer. TCP trades off latency for reliability — it retransmits lost segments, but the guarantee is only "we tried hard."

**"Larger send buffer = faster throughput."**

Not always. Buffer size affects how much data the kernel queues before exerting backpressure. Too large, and you hide the fact that the network is slow, delaying error detection. Too small, and you stall unnecessarily. The optimal size is close to the bandwidth-delay product (section 79.3).

**"non-blocking sockets are always non-blocking."**

Even with `O_NONBLOCK`, the `accept()` call on a non-blocking listening socket can block briefly if the TCP handshake isn't done. Only sockets in the *accept queue* are truly ready. This is why tight loops over `accept()` use a sparse `epoll_wait`.

**"TCP_NODELAY makes TCP faster."**

No. It makes TCP lower-latency by disabling batching. It may actually reduce throughput slightly (more packet headers) because small writes are no longer coalesced. It is essential for interactive protocols, where latency matters more than efficiency.

**"epoll handles 1 million connections easily."**

Scale is limited not by epoll itself (it can handle millions of FDs) but by RAM. Each socket needs a kernel structure (~1–2 KB) and user-space tracking. 1 million sockets = ~1–2 GB of kernel memory, plus application overhead. Also, event storms (all 1 million sockets becoming ready at once) can overwhelm the CPU.

---

## 79.12 Exercises

1. **Socket teardown.** Write a client that connects to a server, sends data, then closes the connection. On the server, use `strace -e write` to watch what syscalls happen. Does `close()` on the client immediately tear down the server's side? What if the server is still trying to send data?

2. **Buffer overflow.** Write a server that listens but never calls `accept()`. Use `wrk` or a custom client script to open many connections. Watch the kernel's accept queue fill, then observe timeout behavior. Increase the backlog parameter and re-test. How does accept-queue size affect client behavior?

3. **Nagle vs no Nagle.** Write a protocol that sends 100-byte messages in a tight loop. Measure latency and throughput with `TCP_NODELAY` on and off. Explain the trade-off in terms of packet count and RTT.

4. **Zero-copy.** Write a static-file server using `sendfile()` and another using traditional `read()`+`send()`. Time both on a large file (>100 MB) with multiple concurrent clients (`wrk`). Measure CPU usage with `top`. Explain the difference.

5. **RTT simulation.** On Linux, use `tc` (traffic control) to add 50 ms latency: `sudo tc qdisc add dev lo root netem delay 50ms`. Measure TCP handshake time with `curl -w %{time_connect}` on a local server. Remove the delay and compare. Do the numbers match your expectation of 1 RTT?

---

## 79.13 Summary

Every byte that travels across a network touches the socket API, kernel buffers, and TCP/IP stack. Understanding this path is the difference between code that works until it doesn't and code that degrades gracefully under load.

**Key takeaways:**

- **Sockets are file descriptors.** The blocking API is simple but doesn't scale. Non-blocking + epoll/kqueue is the production standard.
- **The TCP handshake costs 1 RTT.** Connection reuse and multiplexing (HTTP/2, QUIC) amortize this cost.
- **Send buffer is asynchronous.** Returning from `send()` means the kernel accepted your data, not that it reached the server.
- **Accept queue overflow is real.** Tuning backlog and ensuring timely `accept()` calls prevents connection storms from being dropped.
- **Nagle's algorithm hurts interactive latency.** Set `TCP_NODELAY` for chat-like protocols.
- **epoll scales to tens of thousands of connections.** One thread, one kernel wait syscall per event loop, O(1) per ready FD.
- **Zero-copy (sendfile) eliminates user-space copies** for static files, freeing CPU for other work.
- **QUIC is the future** for new protocols, but TCP will dominate for decades.

Once you grasp how sockets work from the inside, debugging network performance becomes tractable. You know where to measure, what syscalls to trace, and which tuning parameters matter.

---

> **[← Previous: Memory Mapped Files](01-memory-mapped-files.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: Syscalls →](03-syscalls.md)**
