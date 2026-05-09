# Chapter 83 — Protocol Design

A protocol is a contract. When two programs talk over a network, through a file format, or via inter-process communication, they are making a promise to each other: "if you send me these bytes in this order, I will interpret them this way and send you back these bytes." Break the contract, and one side will misinterpret data, crash, or silently corrupt state. Build a protocol that can evolve, and you can ship new features for years without breaking existing systems. Get it wrong, and a seemingly small change — reordering two fields, adding a flag — becomes a deployment incident.

This chapter is about how to design a protocol that works *today* and can survive changes *tomorrow*. The hard part is not the wire format. It is evolution without breaking, idempotency in the face of network failures, and the discipline to decide "what goes in the protocol and what goes in the implementation."

## Learning Objectives

By the end of this chapter, you will be able to:

1. Specify what a protocol must define — framing, encoding, message sequences, versioning, and error handling — and recognize when a design is incomplete.
2. Choose a framing strategy (length-prefix, delimiter, fixed-size header) and trade off the complexity and robustness of each.
3. Distinguish request-response, streaming, and pub-sub patterns, and understand how each shapes the protocol and the code.
4. Design a versioning scheme that survives backward-incompatible changes without versioning hell.
5. Reason about idempotency and retry safety in the face of network failures and duplicate messages.
6. Implement forward compatibility — old clients talking to new servers — and backward compatibility at the same time.
7. Recognize when security belongs in the protocol layer (TLS, replay attack prevention) and when it belongs in the application.
8. Understand the tradeoffs in real protocols (HTTP, gRPC, MQTT) and why every long-lived protocol accumulates complexity.

---

## What a Protocol Specifies

A protocol is a layered specification. At minimum, it must answer:

**1. Framing:** How does the receiver know where one message ends and the next begins?

**2. Encoding:** What do the bytes mean? Is this big-endian or little-endian? Are strings null-terminated or length-prefixed?

**3. Message types and fields:** What are the valid message kinds? What fields must each contain?

**4. Message sequences:** In what order are messages sent? Can the client send multiple requests before receiving a response?

**5. Versioning:** How do we send new field types, new message kinds, or change semantics without breaking the other side?

**6. Error handling:** What happens if a message is malformed, too large, or arrives out of order? Is the connection still usable?

**7. Flow control:** Can the sender overwhelm the receiver? Are there limits on message size, rate, or window size?

**8. Security boundaries:** Does this protocol authenticate? Encrypt? Prevent replay attacks?

A minimal protocol might answer only the first three. A robust one answers all eight. And every layer of the stack — the transport (TCP), the serialization (JSON, Protocol Buffers), the application layer — has its own protocol. A request-response HTTP call involves:

- TCP (connection setup, flow control, retransmission).
- HTTP (request line, headers, body).
- Whatever serialization format you chose for the body (JSON, XML, binary).

If any layer fails to specify something, you get ambiguity. Ambiguity becomes a bug when two implementations make different choices.

---

## Framing: How Messages End

The most fundamental question: how does the receiver know where one message stops and the next starts? There is no "message boundary" in the network stack — TCP is a stream of bytes. You must impose one.

### Strategy 1: Length-Prefix

Prefix each message with its length:

```
[ 4 bytes: length ] [ N bytes: message ][ 4 bytes: length ] [ M bytes: message ]
```

Example:

```
Bytes:  0x00 0x00 0x00 0x05  H    e    l    l    o  0x00 0x00 0x00 0x06  W  o  r  l  d  !
        ^^ length of "hello" ^^  ^^ message ^^     ^^ length of "world!" ^^  ^^ message ^^
```

**Pros:**
- Simple to implement: read 4 bytes, decode length, read that many bytes.
- Supports binary data — no escape characters needed.
- Efficient — zero overhead beyond the length.
- Suitable for large messages.

**Cons:**
- An error in the length field breaks framing for the rest of the connection.
- You must buffer the entire message before processing it (no streaming).
- Requires knowing the length upfront.

### Strategy 2: Delimiter

Use a special byte sequence (delimiter) to mark message boundaries:

```
[ message bytes ]\n[ message bytes ]\n
```

Example (SMTP):

```
MAIL FROM:<sender@example.com>\r\n
RCPT TO:<recipient@example.com>\r\n
DATA\r\n
```

**Pros:**
- Human-readable protocols often use this (HTTP headers, SMTP).
- Can process messages as you receive them (no buffering needed).
- Recovering from errors is sometimes easier — rescan for the delimiter.

**Cons:**
- The delimiter byte(s) must not appear inside the message. This requires escaping, which adds complexity and overhead.
- Binary data is harder. You must base64-encode or escape.
- Scanning for a delimiter is O(N) per message.

### Strategy 3: Fixed-Size Headers

Use a fixed header that describes the message:

```
[ 4 bytes: message type ] [ 4 bytes: length ] [ N bytes: payload ]
```

Example (many RPC frameworks):

```cpp
struct Header {
    uint32_t type;       // 0 = request, 1 = response, 2 = error
    uint32_t request_id; // for matching requests to responses
    uint32_t length;     // payload length
    uint32_t flags;      // compressed? encrypted? final?
};
```

**Pros:**
- Type-aware framing — receiver knows what kind of message is coming before reading the full payload.
- Can route based on type before full deserialization.
- Natural fit for structured protocols (gRPC, Apache Thrift).

**Cons:**
- More overhead than length-prefix alone.
- Must align with the serialization format (binary headers + Protobuf, binary headers + JSON, etc.).

---

## Message Patterns: Request-Response vs Streaming vs Pub-Sub

The pattern shapes the protocol from the ground up.

### Request-Response (RPC Style)

Client sends a request; server sends a response. Examples: HTTP, DNS, most web APIs.

**Protocol shape:**

```
Client → [ request ] → Server
Client ← [ response ] ← Server
```

**Consequences:**
- Simple to reason about — every request has a response.
- Idempotency is achievable (more on this later).
- Latency is at least one round-trip.
- The server must buffer until it can respond.

**Example: Simple key-value protocol**

```
Client: GET key1
Server: VALUE abc123
Client: SET key2 def456
Server: OK
```

### Streaming (Unidirectional or Bidirectional)

One or both sides can send multiple messages without waiting for responses. Examples: gRPC streaming, WebSocket.

**Protocol shape:**

```
Client → [ req1 ] → Server
Client → [ req2 ] → Server
Client ← [ resp1 ] ← Server
Client → [ req3 ] → Server
Client ← [ resp2 ] ← Server
```

**Consequences:**
- Messages may arrive out of order.
- Must use request IDs or sequence numbers to match requests to responses.
- Better throughput if processing can be pipelined.
- More complex to reason about — you must handle concurrent requests.

**Example: gRPC streaming**

```cpp
// Client sends multiple requests; server streams responses
stream<Request> requests;
stream<Response> responses;

requests.send(req1); requests.send(req2);
responses.receive(); // could be in any order relative to requests
```

### Pub-Sub (One-to-Many)

A publisher sends messages to a broker; subscribers receive copies. Examples: MQTT, RabbitMQ, Kafka.

**Protocol shape:**

```
Publisher → [ message ] → Broker
Broker → [ message ] → Subscriber A
Broker → [ message ] → Subscriber B
Broker → [ message ] → Subscriber C
```

**Consequences:**
- No direct connection between publishers and subscribers.
- Messages may be buffered at the broker.
- Subscribers may miss messages if the broker doesn't persist them.
- Must define subscription semantics — exactly-once, at-least-once, at-most-once?

---

## Versioning: The Cost of Change

Every protocol eventually needs to evolve: new fields, new message types, fixes to existing types. Versioning is how you avoid breaking old clients.

### Strategy 1: Version Field in Every Message

Add a version number to every message:

```
[ 4 bytes: version ] [ rest of message ]
```

Example:

```cpp
struct Request {
    uint32_t version = 1;
    std::string key;
    std::string value;
};
```

When you change the message, increment the version and handle both old and new:

```cpp
Request decode(const bytes& data) {
    uint32_t version = read_u32(data, 0);
    if (version == 1) return decode_v1(data);
    if (version == 2) return decode_v2(data);
    throw UnknownVersionError();
}
```

**Pros:**
- Explicit — you control exactly when the version changes.
- Works for any encoding (JSON, binary, Protocol Buffers).

**Cons:**
- Every message carries version overhead.
- You must maintain decoders for all past versions.
- If you have 100 message types, you now have 100 version fields.

### Strategy 2: Version at Connection Time

Negotiate a version once, at connection setup:

```
Client: HELLO version=2
Server: OK version=2
Client: [ messages assume version 2 ]
Server: [ messages assume version 2 ]
```

Example (HTTP/2):

```
Client sends: PRI * HTTP/2.0\r\n
Server responds: 200 OK (HTTP/2 understood)
```

**Pros:**
- Zero per-message overhead.
- Clients and servers can assume a single version for the entire connection.
- Natural fit for long-lived connections.

**Cons:**
- If either side doesn't understand the negotiated version, the connection is useless.
- Doesn't work for stateless protocols (HTTP/1.0).

### Strategy 3: Capability Negotiation

Instead of versions, exchange capabilities:

```
Client: HELLO capabilities=[feature_a, feature_b]
Server: OK capabilities=[feature_a, feature_c]
Client: [ only use feature_a, which both support ]
```

Example (TLS):

```
Client: I support cipher suites [AES-GCM, ChaCha20]
Server: I choose AES-GCM
```

**Pros:**
- Flexible — doesn't assume a linear version progression.
- Handles missing features gracefully.

**Cons:**
- More complex negotiation.
- Requires both sides to advertise what they support.

### Strategy 4: Optional Fields (JSON, Protobuf)

If using a self-describing format (JSON, Protocol Buffers), new fields are optional by default. Old parsers ignore unknown fields; new parsers provide defaults for missing fields.

```json
{
  "key": "name",
  "value": "Alice",
  "ttl_seconds": 3600
}
```

An old client that doesn't know about `ttl_seconds` still works; it ignores the field. A new server can check if the field exists and provide a default if not.

**Pros:**
- Forward and backward compatible by design.
- No explicit versioning needed.

**Cons:**
- Requires a self-describing format.
- Semantics of missing fields must be clear (is it omitted or explicitly null?).

---

## Idempotency: Surviving Retries

The network can fail. A request may be sent but never received, received but the response lost, or received twice. Without idempotency, retries become dangerous.

### The Idempotency Problem

```
Client: SET account:123 balance=500
Server: [ crashes before responding ]
Client: [ times out, retries ]
Client: SET account:123 balance=500
Server: SET account:123 balance=500  // twice!
```

If SET means "add 500 to the balance," you get 1000. If SET means "set the balance to 500," you're fine. The protocol must be explicit about which operations are idempotent.

### Solution 1: Idempotency Keys

The client includes a unique identifier for each operation:

```
Client: SET key=account:123 value=500 idempotency_key=req:abc123
Server: [ processes, stores the key and result ]
Client: [ retries, sends the same key ]
Server: [ sees the key, returns the cached result without re-executing ]
```

Example (HTTP):

```http
POST /transfer HTTP/1.1
Idempotency-Key: client-req-12345
Content-Type: application/json

{"from": "account_a", "to": "account_b", "amount": 500}
```

The server caches the result of the operation keyed by `client-req-12345`. If the same key arrives again, it returns the cached result.

**Pros:**
- Works for any operation.
- Client controls the key; easy to guarantee uniqueness.

**Cons:**
- Server must cache results, costing memory.
- Cache invalidation (when can we forget a result?) is a separate problem.

### Solution 2: Versioned State (Conditional Updates)

The client sends the expected current state and only updates if it matches:

```
Client: SET key=account:123 value=600 IF version=5
Server: [ checks version, if it matches, updates and increments version to 6 ]
Server: [ if version doesn't match, returns conflict ]
```

Example (HTTP):

```http
PUT /resource/123 HTTP/1.1
If-Match: "etag-v5"
Content-Type: application/json

{"data": "updated"}
```

This is called **optimistic concurrency control**. Retries are automatically idempotent because the retry includes the old version.

**Pros:**
- Detects concurrent modifications.
- No server-side caching required.

**Cons:**
- Client must track versions.
- Fails if the state has changed (conflict, not silent success).

### Solution 3: Sequence Numbers

The client includes a sequence number; the server processes messages in order and deduplicates:

```
Client: [ seq=1 ] SET key=account:123 value=500
Server: [ seq=1 accepted, stored ]
Client: [ timeout, retries seq=1 ]
Server: [ seq=1, already seen, returns cached result ]
Client: [ seq=2 ] SET key=account:456 value=300
Server: [ seq=2 accepted, seq=1 was before, now processing seq=2 ]
```

**Pros:**
- Automatic deduplication of retries.
- Works for streaming protocols.

**Cons:**
- Must sequence on the client; if the client crashes, sequence numbers reset and collisions are possible.
- The server must maintain a window of accepted sequence numbers.

---

## Backward and Forward Compatibility

A robust protocol must survive two scenarios:

### Scenario 1: New Client, Old Server

The client wants to use a new feature that the old server doesn't know about:

```cpp
// New client wants a TTL field
{
  "key": "name",
  "value": "Alice",
  "ttl_seconds": 3600   // old server doesn't understand this
}
```

**Solution:** New clients must tolerate the server ignoring unknown fields. Better: the client should explicitly request backward-compatible behavior:

```cpp
{
  "key": "name",
  "value": "Alice",
  "ttl_seconds": 3600,
  "features": ["ttl"]  // advertise what we need
}
```

The old server, not seeing support for `ttl`, silently ignores the field.

### Scenario 2: Old Client, New Server

The old client sends a message in an old format; the new server must understand it:

```cpp
// Old client doesn't have ttl_seconds
{
  "key": "name",
  "value": "Alice"
}

// New server must provide a default
if (!request.has_field("ttl_seconds")) {
  request.ttl_seconds = 0;  // or whatever the default is
}
```

**Solution:** New servers should:
1. Accept messages in old formats.
2. Provide sensible defaults for missing fields.
3. Document the defaults clearly.

---

## Security from the Start

Security is often bolted on after the fact. Protocol-level security is cheaper.

### TLS by Default

If the protocol transmits anything valuable (passwords, tokens, user data), use TLS:

```cpp
// Good: mandatory encryption
client.connect("example.com", 443, TLS);

// Bad: unencrypted, adds a separate security layer
client.connect("example.com", 8080, NO_TLS);
```

TLS gives you:
- **Encryption**: eavesdropping requires computational effort.
- **Authentication**: the server's certificate proves its identity.
- **Integrity**: man-in-the-middle attacks can be detected.

### Authentication Tokens

Require every request to authenticate:

```
Client: GET /resource Authorization: Bearer token_abc123
Server: [ checks token validity, rate limit, permissions ]
```

Tokens should:
- Be issued by a trusted authority.
- Have an expiration time.
- Be transmitted over TLS only (not HTTP).
- Not be logged or cached unnecessarily.

### Replay Attack Prevention

An attacker intercepts a valid request and replays it:

```
Attacker intercepts: POST /transfer amount=100 nonce=X
Attacker replays: POST /transfer amount=100 nonce=X
Server executes the transfer twice.
```

**Solution:** Include a nonce (number used once):

```cpp
struct Request {
    uint64_t timestamp;      // when was this created?
    std::string nonce;       // unique per request
    std::string signature;   // HMAC(request, secret)
};
```

The server rejects requests with:
- Timestamps too old (request took too long to arrive).
- Nonces it has seen before.

### Message Size Limits

An attacker sends a gigabyte "message" to exhaust the server's memory:

```cpp
// Bad: allocate whatever the client claims
size_t length = read_u32(socket);
char* buf = malloc(length);  // attacker sends length=1GB

// Good: enforce a maximum
size_t length = read_u32(socket);
if (length > MAX_MESSAGE_SIZE) {
    close_connection();
    return;
}
char* buf = malloc(length);
```

### Rate Limiting

An attacker floods the server with requests:

```cpp
struct ClientQuota {
    int requests_this_second = 0;
    std::chrono::steady_clock::time_point reset_at;
};

if (client_quota.requests_this_second > MAX_RPS) {
    send_429_too_many_requests();
    return;
}
```

---

## Worked Example: A Key-Value RPC Protocol

Let's design a simple protocol from the ground up.

### Requirements

1. Three operations: GET, SET, DEL.
2. Support key/value of arbitrary binary size.
3. Support concurrent requests.
4. Survive retries and server restarts.

### Version 1: Basic Protocol

**Framing:** 4-byte big-endian length prefix.

**Message format:**

```
[ 4 bytes: message length N ]
[ 1 byte: op (1=GET, 2=SET, 3=DEL, 4=RESPONSE) ]
[ 4 bytes: request_id (for matching requests to responses) ]
[ 4 bytes: key length K ]
[ K bytes: key ]
[ 4 bytes: value length V ] (only for SET)
[ V bytes: value ] (only for SET)
```

**Example: SET foo=bar**

```
Length:     0x00 0x00 00 0x17  (23 bytes)
Op:         0x02               (SET)
Request ID: 0x00 0x00 0x00 0x01
Key length: 0x00 0x00 0x00 0x03
Key:        'f' 'o' 'o'
Value len:  0x00 0x00 0x00 0x03
Value:      'b' 'a' 'r'
```

**Response format:**

```
[ 4 bytes: message length ]
[ 1 byte: op (4=RESPONSE) ]
[ 4 bytes: request_id ]
[ 1 byte: status (0=OK, 1=ERROR, 2=NOT_FOUND) ]
[ 4 bytes: value length V ] (only if GET and found)
[ V bytes: value ] (only if GET and found)
```

**Implementation (sender):**

```cpp
void send_set(int sock, uint32_t request_id, const std::string& key, const std::string& value) {
    std::vector<uint8_t> msg;
    msg.push_back(2);  // SET op
    
    append_u32(msg, request_id);
    append_u32(msg, key.size());
    msg.insert(msg.end(), key.begin(), key.end());
    append_u32(msg, value.size());
    msg.insert(msg.end(), value.begin(), value.end());
    
    // Length prefix
    uint32_t len = msg.size();
    std::vector<uint8_t> frame;
    append_u32(frame, len);
    frame.insert(frame.end(), msg.begin(), msg.end());
    
    write(sock, frame.data(), frame.size());
}
```

**Implementation (receiver):**

```cpp
std::pair<uint32_t, std::string> recv_message(int sock) {
    uint8_t len_bytes[4];
    read_exact(sock, len_bytes, 4);
    uint32_t len = read_u32(len_bytes);
    
    if (len > MAX_MESSAGE_SIZE) {
        throw std::runtime_error("Message too large");
    }
    
    std::vector<uint8_t> msg(len);
    read_exact(sock, msg.data(), len);
    
    uint8_t op = msg[0];
    uint32_t req_id = read_u32(&msg[1]);
    // ... parse rest based on op
    
    return {req_id, parsed_msg};
}
```

### Version 2: Add Idempotency

Add an `idempotency_key` field for retries:

```
[ ... existing fields ... ]
[ 4 bytes: idempotency_key length K ]
[ K bytes: idempotency_key ]
```

The server stores the idempotency_key alongside the result. On retry, it returns the cached result.

**Database schema:**

```sql
CREATE TABLE requests (
    idempotency_key TEXT PRIMARY KEY,
    result BLOB,
    created_at TIMESTAMP
);
```

**Lookup before processing:**

```cpp
auto cached = db.query("SELECT result FROM requests WHERE idempotency_key = ?", idempotency_key);
if (cached) {
    return cached->result;  // cached response
}

// ... process the request ...

db.execute("INSERT INTO requests (idempotency_key, result) VALUES (?, ?)", idempotency_key, result);
```

### Version 3: Add Compare-and-Swap (CAS)

Add a new operation for atomic updates:

```
CAS: [ op=5 ] [ request_id ] [ key ] [ expected_value ] [ new_value ]
```

Response: OK if the value matched and was swapped; CONFLICT if it didn't.

**Key insight:** This is backward compatible because old clients don't send op=5. The server, even in v3, still handles ops 1–4 from v1 clients.

---

## Lessons From Real Protocols

### HTTP: Born Simple, Became Complex

**HTTP/0.9 (1991):**
```
GET /doc.txt
```

One request type, one response, close the connection. Total protocol: ~50 lines.

**HTTP/1.0 (1996):**
Added headers, status codes, multiple content types. One request per connection still.

**HTTP/1.1 (1997):**
Keep-alive (reuse connections), chunked transfer encoding, pipelining, conditional requests (`If-Modified-Since`).

**HTTP/2 (2015):**
Binary framing, multiplexing (multiple requests on one connection), push.

**HTTP/3 (2022):**
Built on QUIC, not TCP. Eliminates head-of-line blocking.

**Lesson:** Every feature addresses a real problem:
- Keep-alive reduced connection overhead (from 3-way handshake for every request).
- Pipelining reduced latency.
- Chunked encoding allowed streaming responses without knowing size upfront.
- Multiplexing allowed parallelism without spawning connections.
- QUIC removed TCP's head-of-line blocking.

But each feature added complexity. A modern HTTP client must handle: redirects, cookies, proxy negotiation, TLS negotiation, compression, caching headers, etags, 303 redirects, 307 redirects, retry-after, content-encoding negotiation, conditional requests, range requests...

**Takeaway:** Design the simplest protocol that solves your problem. Then prepare for it to acquire features. Build versioning in from the start.

### gRPC: Binary, Multiplexed, Streaming

gRPC is HTTP/2 + Protocol Buffers. It got right:
- Multiplexing: many concurrent requests on one connection.
- Streaming: client and server can send multiple messages.
- Binary: compact, fast to parse.
- Type-safe: code generation from `.proto` files.

It got wrong (or at least, added friction):
- Binary protocol is hard to debug (no curl, no netcat).
- Tight coupling between transport (HTTP/2) and application logic.
- Requires a `.proto` schema; can't easily send dynamic data.

**Takeaway:** Choose binary for speed, text for debuggability. There's no neutral ground.

### MQTT: Pub-Sub on Constrained Networks

MQTT was designed for IoT: devices with limited bandwidth, unreliable connections, variable connectivity.

Key ideas:
- **QoS levels:** at-most-once, at-least-once, exactly-once (each trades off complexity for reliability).
- **Will messages:** if a device disconnects unexpectedly, publish a message on its behalf.
- **Retained messages:** the broker keeps the last message for each topic, so new subscribers immediately get the current state.

**Takeaway:** Different problem domains need different protocols. MQTT's QoS/retry/will logic would be overkill for datacenter RPC; for wireless devices, it's essential.

---

## Tradeoffs

| Aspect | Choice | Tradeoff |
|---|---|---|
| Framing | Length-prefix | Binary-friendly, no escaping, but length errors break the stream |
| Framing | Delimiter | Human-readable, streaming-friendly, but requires escaping |
| Framing | Fixed header | Type-aware, structured, but more overhead |
| Encoding | Binary (Protobuf, MessagePack) | Compact, fast to parse, hard to debug |
| Encoding | Text (JSON, XML) | Self-describing, debuggable, verbose |
| Versioning | Per-message version | Explicit control, more payload per message |
| Versioning | Connection-time negotiation | Zero per-message overhead, but all-or-nothing |
| Versioning | Capability negotiation | Flexible, but complex |
| Idempotency | Idempotency keys | Works for any operation, requires caching |
| Idempotency | Versioned state | Detects conflicts, no caching, but fails on conflict |
| Idempotency | Sequence numbers | Natural for streaming, but complex for stateless |
| Compatibility | Old fields required | Simple schema, fragile to changes |
| Compatibility | New fields optional | Forward/backward compatible, but defaults must be sensible |

---

## Common Misconceptions

| Misconception | Reality |
|---|---|
| "The protocol is just the wire format." | Protocols must specify framing, versioning, sequencing, error handling, and security. The format is the smallest part. |
| "I'll design the protocol and figure out versioning later." | Versioning is hardest to add after launch. Build it in from the start. |
| "If I retry, it must be safe." | Retries are unsafe without idempotency. Net protocols must assume failures; retry == danger without explicit support. |
| "TLS is the security layer; the protocol doesn't need security features." | TLS provides encryption and transport integrity, not rate limiting, replay protection, or application-level auth. Both layers matter. |
| "We'll support both old and new clients indefinitely." | Every old version costs maintenance. Set a sunset date and deprecate older versions. |
| "JSON is self-describing so I don't need to version it." | Missing fields still need defaults. Removing fields breaks old clients. JSON reduces the *syntax* problem; it doesn't eliminate the *semantic* one. |
| "Streaming protocols are faster." | Streaming adds complexity (sequencing, deduplication). Only use it if you need the throughput or if latency per request is the bottleneck. |

---

## Exercises

1. **Framing comparison:** You need to send binary blobs over TCP. Each blob is ~100 KB to 1 GB. Compare length-prefix, delimiter, and fixed-header framing. What breaks? When would you choose each?

2. **Versioning under pressure:** You shipped a protocol with `version` as a per-message field. Now you have v1, v2, v3 clients and v4 servers in production. A new v5 encoding is coming. What's your upgrade strategy? What happens if a v1 client somehow reaches a v5 server?

3. **Retry safety:** Design a GET protocol for a cache server. In what cases is GET inherently idempotent? When would you need explicit idempotency keys?

4. **Backward compatibility challenge:** You want to add a new field `priority` to your request messages, defaulting to `"normal"`. Clients that don't set it should work with old servers. Old clients' missing field should work with new servers. How do you encode this in JSON? In binary with a length-prefix protocol?

5. **Implement the worked example:** Code the v1 key-value protocol from the worked example in your language. Write a client and server. Test:
   - SET and GET.
   - Concurrent requests (two GETs at the same time).
   - Large values (1 MB).
   - Protocol error: what happens if you send a malformed length prefix?

---

## Summary

A protocol is a contract, and contracts are enforceable only if both parties understand them. The hard part of protocol design is not the bytes — it is choosing what goes in the protocol (framing, versioning, security) and what goes in the application (business logic), then evolving gracefully when requirements change. Choose a framing strategy that matches your data and debugging needs; design versioning in from the start; make idempotency explicit; and accept that every protocol you ship today will acquire features and complexity over time. The measure of a good protocol is not its simplicity at launch, but its ability to survive five years of changes without breaking the systems that speak it.

---

> **[← Previous: Serialization Internals](05-serialization-internals.md)** · **[↑ Part 8](README.md)** · **[Next: Performance Engineering →](07-performance-engineering.md)**
