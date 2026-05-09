# Chapter 54 — How Python AsyncIO Works

Python is single-threaded by default, strangled by the Global Interpreter Lock (GIL) that allows only one thread to execute Python bytecode at a time. Yet asyncio lets a single Python process handle thousands of concurrent network connections without threads. This chapter is exactly how.

The answer: **asyncio is an event loop that runs inside a single thread**. Every coroutine is a state machine (like the C++ examples in Chapter 49). The loop runs one coroutine step at a time, suspends it at `await` points, and resumes it when its I/O completes. No threads needed. The thread sleeps between work, not the coroutines.

By the end of this chapter you will understand how asyncio multiplexes thousands of coroutines on a single Python thread, how it integrates with the kernel's I/O notification systems, what patterns work and what patterns deadlock, and where asyncio fits in relation to threads and multiprocessing.

---

## 54.1 Coroutines in Python: Stackless State Machines

An `async def` function in Python is not a real function. It is a factory that returns a coroutine object — a state machine allocated on the heap. This is the same design as C++20 coroutines (Chapter 49), but implemented in Python's interpreter rather than in the compiler.

Here is a simple async function:

```python
async def read_file_simple(path: str) -> str:
    data = await asyncio.to_thread(open, path).read()
    return data
```

When you call `read_file_simple("data.txt")`, the function does not run. It returns a coroutine object:

```python
coro = read_file_simple("data.txt")
print(type(coro))  # <class 'coroutine'>
print(coro)        # <coroutine object read_file_simple at 0x...>
```

That coroutine object is a state machine. It contains:

1. **The code** — what the coroutine will execute. The bytecode is stored in the function's code object.
2. **The state** — which line it is suspended on. This is implicitly tracked by the interpreter via the coroutine's frame and instruction pointer.
3. **Local variables** — which outlive `await` points, stored in the coroutine's frame object. Unlike a normal function frame, which is deallocated when the function returns, a coroutine frame persists across `await` points.

To run the coroutine, you call `await` (inside an async context) or pass it to `asyncio.run()`:

```python
import asyncio

result = asyncio.run(read_file_simple("data.txt"))
```

`asyncio.run()` creates an event loop, runs the coroutine to completion, and tears down the loop. Under the hood:

1. It wraps the coroutine in a `Task` (a subclass of `Future`).
2. It runs the event loop until the task completes.
3. It returns the result or raises the exception.

**The stackless difference**: Unlike threads (which have their own stack), coroutines share a single stack. When a coroutine suspends, its stack frame is not saved to the OS; instead, it is saved in the coroutine object's frame field. This is why millions of coroutines can exist on a single thread — they do not need millions of stacks, only millions of small frame objects.

---

## 54.2 The Event Loop: A Single Thread's Heartbeat

The event loop is a single-threaded scheduler. Here is the simplified lifecycle:

```python
import selectors
import collections

class EventLoop:
    def __init__(self):
        self.ready = collections.deque()  # Tasks ready to run
        self.selector = selectors.DefaultSelector()  # I/O readiness
        self.fd_to_callback = {}  # Map file descriptors to wake callbacks
        
    def run_until_complete(self, future):
        # Wrap the coroutine in a Task if needed
        if isinstance(future, Coroutine):
            future = asyncio.Task(future)
        
        # Add the task to the ready queue
        self.ready.append(future)
        
        while not future.done():
            # 1. Run all ready tasks until they suspend
            while self.ready:
                task = self.ready.popleft()
                try:
                    task._step()  # Resume the task, runs until next await
                except StopIteration:
                    pass  # Task is done
            
            # 2. Wait for I/O (blocks until something is ready or timeout)
            # On Linux: epoll_wait()
            # On BSD/macOS: kevent()
            # On Windows: select()
            events = self.selector.select(timeout=1.0)
            
            # 3. Wake tasks whose I/O is ready
            for key, mask in events:
                fd = key.fd
                callback = self.fd_to_callback[fd]
                callback()  # Re-queues the task
        
        return future.result()
```

The loop cycles through three phases:

1. **Ready phase**: Execute coroutines that are queued. Each call to `task._step()` resumes the coroutine's bytecode interpreter until it hits the next `await` or completes. Control returns to the loop immediately.

2. **Blocking phase**: Call `select()` (or `poll()`, or `epoll()` on Linux) to ask the kernel "which file descriptors are ready?" The kernel does not busy-spin — it truly sleeps the thread. The CPU is released; no power is wasted.

3. **Wakeup phase**: When the kernel signals that a file descriptor is ready, find the task waiting on it and put it back on the ready queue.

This is the fundamental insight of asyncio: **one OS thread sleeps between work, and thousands of coroutines can be multiplexed onto it because they yield control at `await` points.**

The key difference from threads: when a task suspends, its CPU time is over. The OS scheduler is not involved; the event loop itself decides which task runs next based on what I/O is ready. This eliminates context switch overhead and makes it possible to have thousands of tasks without overwhelming the kernel's thread scheduler.

---

## 54.3 Tasks, Futures, and Coroutines

Three related concepts, often confused:

**Coroutine**: A generator-like object created by calling an `async def` function. It is inert until awaited. It represents potential work, not scheduled work. The coroutine is a state machine that can only be resumed by calling its `send()` method.

```python
async def fetch_data():
    return "data"

coro = fetch_data()  # Coroutine object; nothing runs yet
print(type(coro))   # <class 'coroutine'>
```

**Future**: A placeholder for a value that is not yet known. It can be set with a result or exception later. A Future has two states: pending or done. Once done, it holds either a result or an exception. Futures are the primitives that the event loop uses to wake tasks.

```python
import asyncio

future = asyncio.Future()
print(future.done())  # False

# ... sometime later ...
future.set_result("the result")  # Now it's done
print(future.done())  # True

# Elsewhere, await it:
value = await future  # Gets "the result"
```

**Task**: A subclass of Future that wraps a coroutine and drives it forward. When you `asyncio.create_task(coro)`, a Task is created, the coroutine is wrapped, and the task is scheduled on the event loop. A Task is both a scheduling unit and a Future — it is both something the loop steps through and something you can await.

```python
async def main():
    task = asyncio.create_task(fetch_data())
    result = await task  # Waits for the task to complete
```

Internally, a Task's `_step()` method calls the coroutine's `send()` method, which resumes the coroutine from its last `await`:

```python
# Pseudocode of Task._step()
def _step(self):
    try:
        # Resume the coroutine from where it last suspended
        # send(None) tells the coroutine "continue from the await"
        next_future = self.coro.send(None)
    except StopIteration as e:
        # Coroutine returned a value (completed)
        self.set_result(e.value)
        return
    
    # The coroutine hit an await; next_future is what it's waiting on
    # next_future is another Future (or Task)
    
    if next_future is None:
        # If the future is None, re-queue immediately
        self._reschedule()
    else:
        # Register a callback so when next_future completes, we call _step() again
        next_future.add_done_callback(lambda: self._reschedule())

def _reschedule(self):
    # Put this task back on the event loop's ready queue
    self._loop.ready.append(self)
```

When the awaited Future completes, the callback fires, and the Task is re-queued to run its next step. The next time the event loop's ready phase runs, the Task's `_step()` method is called again, and the coroutine resumes.

---

## 54.4 I/O Readiness and the Selector

asyncio uses Python's `selectors` module, which abstracts over the OS's I/O notification mechanisms (described in detail in Chapter 49):

- **Linux**: `selectors.EpollSelector` → `epoll()` syscall. Extremely efficient; O(1) to register FDs, returns only ready FDs.
- **macOS/BSD**: `selectors.KqueueSelector` → `kqueue()` syscall. Flexible; can monitor FDs, timers, signals, filesystem events.
- **Windows**: `selectors.SelectSelector` → `select()` syscall. Slower than epoll (O(n) to scan all FDs), but portable.

The selector is where asyncio connects to the kernel's I/O notification system. Without it, the event loop would have to busy-poll or block indefinitely. With it, the thread truly sleeps until I/O is ready.

Here is how a typical network operation works:

```python
import asyncio
import socket

async def fetch_http(host: str, port: int):
    # Create a non-blocking socket
    sock = socket.socket()
    sock.setblocking(False)
    
    try:
        # Non-blocking connect will raise BlockingIOError
        sock.connect((host, port))
    except BlockingIOError:
        pass  # Expected; connection will complete asynchronously
    
    # Register the socket with the event loop's selector
    loop = asyncio.get_running_loop()
    
    # Create a Future that will complete when the socket is writable
    future = loop.create_future()
    
    def on_writable():
        future.set_result(None)
    
    loop.add_writer(sock.fileno(), on_writable)
    
    # Await the future; this suspends until the socket is writable
    await future
    
    # Socket is now connected; do something with it
    sock.send(b"GET / HTTP/1.0\r\n\r\n")
    
    loop.remove_writer(sock.fileno())
```

Under the hood:

1. The socket is put in non-blocking mode (`sock.setblocking(False)`).
2. The event loop calls `selector.register(fd, selectors.EVENT_WRITE)` to tell the kernel "wake me when this FD is writable."
3. The event loop blocks on `selector.select(timeout)`.
4. The kernel wakes the selector when the socket is ready.
5. The callback fires, the Future is resolved, and the Task is re-queued.

In practice, you do not call `add_writer()` directly. You use higher-level functions like `asyncio.open_connection()`:

```python
async def fetch_http_simple(host: str, port: int):
    reader, writer = await asyncio.open_connection(host, port)
    writer.write(b"GET / HTTP/1.0\r\n\r\n")
    await writer.drain()
    data = await reader.read(4096)
    return data
```

This is built on the same machinery but hides the socket and selector details.

---

## 54.5 The GIL and AsyncIO

Python's Global Interpreter Lock (GIL) restricts one OS thread to execute Python bytecode at a time. Threads can run, but they cannot run Python code truly in parallel — only concurrently, taking turns.

**How this affects asyncio**: asyncio completely sidesteps the GIL's parallelism restriction, because it is single-threaded. The event loop runs in one thread, and no GIL contention occurs. The GIL is held for the entire duration of the loop, but that is fine — there is no other thread trying to acquire it.

The GIL only becomes relevant when you use `asyncio.to_thread()` or `loop.run_in_executor()` to spawn threads:

```python
async def main():
    # GIL is held in the event loop thread
    
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, expensive_sync_function)
    
    # expensive_sync_function runs in a background thread
    # That thread can run Python bytecode, but it must acquire the GIL
    # The event loop thread releases the GIL while waiting for the result
```

When the background thread starts, the event loop thread releases the GIL (because it is blocked in `await`), allowing the background thread to acquire it and run. This is why `asyncio.to_thread()` is safe — the GIL is released, and true parallelism is possible.

**Practical consequence**: For I/O-bound work, asyncio is preferable to threading because it avoids GIL contention entirely. For CPU-bound work, neither asyncio nor threads help — you need multiprocessing.

---

## 54.6 Where Python AsyncIO Goes Wrong

### Forgetting `await`

The most insidious bug: calling an async function without `await`. The function returns a coroutine object, which sits idle forever:

```python
async def fetch_data():
    print("Fetching...")
    return "data"

async def main():
    fetch_data()  # Oops! Forgot await
    # The coroutine is created but never run
    # Python will warn: "coroutine was never awaited"
```

### Blocking the Loop with Sync Code

If you call a blocking function inside an async context, the entire event loop blocks. Remember: the event loop runs in a single thread. If you call `time.sleep(1)`, that thread is sleeping, and *all* coroutines — all thousands of them — are frozen:

```python
async def slow_operation():
    for i in range(10):
        await asyncio.sleep(0.1)  # Yields; total: 1 second
        print(f"Slow op: {i}")

async def blocking_operation():
    time.sleep(1)  # BLOCKS the entire event loop
    print("Blocking op done")

async def main():
    await asyncio.gather(slow_operation(), blocking_operation())
```

If you run this, the blocking operation completes in 1 second, and the slow operation waits the entire time. It does not interleave. The blocking function starves the entire event loop.

Here is proof with timing:

```python
import asyncio
import time

async def slow_operation():
    print(f"Slow op started at {time.time():.2f}")
    for i in range(10):
        await asyncio.sleep(0.1)
    print(f"Slow op finished at {time.time():.2f}")

async def blocking_operation():
    print(f"Blocking op started at {time.time():.2f}")
    time.sleep(1)
    print(f"Blocking op finished at {time.time():.2f}")

async def main():
    await asyncio.gather(slow_operation(), blocking_operation())

asyncio.run(main())

# Output:
# Slow op started at 0.00
# Blocking op started at 0.00
# Blocking op finished at 1.00  <- After 1 second
# Slow op finished at 1.10      <- Only finishes after blocking is done
```

The fix is to run the blocking function in a thread pool:

```python
async def main():
    result = await asyncio.to_thread(time.sleep, 1)
    # Other coroutines can still run
```

Or use an async library (e.g., `aiohttp` for HTTP, `asyncpg` for Postgres, `aioredis` for Redis) that does not block:

```python
async def main():
    async with aiohttp.ClientSession() as session:
        async with session.get('http://example.com') as resp:
            data = await resp.text()
    # Network I/O happens asynchronously; event loop is free to run other tasks
```

### CPU-Bound Work Starves the Loop

Unlike I/O, CPU-bound operations cannot be suspended at `await` points. A coroutine computing prime factorization will block the entire loop:

```python
async def cpu_bound():
    result = sum(1 for i in range(10**8) if is_prime(i))
    return result

async def io_bound():
    await asyncio.sleep(0.001)
    return "fast"

async def main():
    await asyncio.gather(cpu_bound(), io_bound())
    # io_bound will not complete for seconds because cpu_bound blocks
```

Solution: use `asyncio.to_thread()` or multiprocessing:

```python
from concurrent.futures import ProcessPoolExecutor

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_bound)
    # Now cpu_bound runs in a separate process; io_bound is unaffected
```

### Race Conditions Without Locks

Single-threaded asyncio has no true data races (no two coroutines run simultaneously on different cores). But a subtle class of re-entrancy bugs occurs when a coroutine modifies shared state across `await` points. This is not a memory race (the CPU does not fetch stale values), but a logic error: the other coroutine runs *between* the read and the write.

```python
counter = 0

async def increment():
    global counter
    current = counter          # Read: get current value (e.g., 0)
    await asyncio.sleep(0.001)  # Suspend here; another increment() may run!
    counter = current + 1      # Write: but we're still using the old value

async def main():
    await asyncio.gather(
        increment(),
        increment(),
        increment()
    )
    print(counter)  # Expected 3, got 1 (not 3!)
```

Here is the interleaving:

```
Time  increment() #1           increment() #2           increment() #3
0     current = 0
1     await sleep
2                               current = 0
3                               await sleep
4                                                        current = 0
5                                                        await sleep
6     counter = 0 + 1 = 1
7                               counter = 0 + 1 = 1 (overwrites #1!)
8                                                        counter = 0 + 1 = 1 (overwrites #2!)
9     Final: counter = 1 (lost updates on #2 and #3)
```

This is not a thread-safety issue in the traditional sense (no memory race), but a logic bug caused by re-entrancy. The lock is not protecting against concurrent access, but against a time gap in the logic.

Fix: use `asyncio.Lock`, which is async-aware:

```python
counter = 0
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:
        # No other coroutine can run while holding the lock
        current = counter
        await asyncio.sleep(0.001)
        counter = current + 1

async def main():
    await asyncio.gather(increment(), increment(), increment())
    print(counter)  # Now 3 (correct)
```

The lock does not block the event loop. When a coroutine tries to acquire a held lock, it suspends, and other coroutines run. When the lock is released, the waiting coroutine is re-queued.

---

## 54.7 Patterns That Work Well

### Gathering Concurrent Tasks

`asyncio.gather()` fan-outs multiple coroutines and waits for all:

```python
async def fetch_urls(urls: list):
    async with aiohttp.ClientSession() as session:
        tasks = [session.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return responses
```

All requests are issued before any response is read. The event loop interleaves the I/O operations, making efficient use of a single thread.

### Timeouts

`asyncio.wait_for()` cancels a coroutine if it does not complete in time:

```python
async def fetch_with_timeout(url: str, timeout: float):
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                return await asyncio.wait_for(resp.text(), timeout=timeout)
    except asyncio.TimeoutError:
        print(f"Request to {url} timed out")
```

### Backpressure with Bounded Queues

`asyncio.Queue()` with a `maxsize` ensures the producer does not overwhelm the consumer:

```python
async def producer(queue: asyncio.Queue):
    for i in range(1000):
        await queue.put(i)  # Blocks if queue is full

async def consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        await asyncio.sleep(0.1)  # Slow consumer
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=10)
    producer_task = asyncio.create_task(producer(queue))
    consumer_task = asyncio.create_task(consumer(queue))
    await producer_task
```

The producer will block when 10 items are queued, preventing memory from growing unbounded.

### Server Loops

A typical asyncio server handles multiple clients with one coroutine per client:

```python
async def handle_client(reader, writer):
    data = await reader.read(1024)
    writer.write(data)  # Echo server
    await writer.drain()
    writer.close()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8000)
    async with server:
        await server.serve_forever()
```

The event loop accepts new connections and spawns a `handle_client` coroutine for each. All clients are multiplexed on a single thread.

---

## 54.8 AsyncIO vs Threads vs Multiprocessing

Here is a practical comparison. Suppose we scan 10,000 ports on 100 hosts (1 million port checks). We want to know which are open.

### Approach 1: Synchronous (Naive)

```python
import socket

def is_port_open(host: str, port: int) -> bool:
    sock = socket.socket()
    sock.settimeout(1)
    try:
        sock.connect((host, port))
        sock.close()
        return True
    except:
        return False

def main():
    hosts = [f"192.168.1.{i}" for i in range(1, 101)]
    ports = list(range(1, 10001))
    
    for host in hosts:
        for port in ports:
            if is_port_open(host, port):
                print(f"{host}:{port} is open")

# Time: ~100 hosts * 10,000 ports * 1 sec timeout = ~1 million seconds
# (This is unusable; we would timeout on most checks anyway.)
```

### Approach 2: Threads

```python
import concurrent.futures
import socket

def is_port_open(host: str, port: int) -> bool:
    sock = socket.socket()
    sock.settimeout(1)
    try:
        sock.connect((host, port))
        sock.close()
        return True
    except:
        return False

def main():
    hosts = [f"192.168.1.{i}" for i in range(1, 101)]
    ports = list(range(1, 10001))
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=100) as executor:
        futures = [
            executor.submit(is_port_open, host, port)
            for host in hosts for port in ports
        ]
        for future in concurrent.futures.as_completed(futures):
            if future.result():
                print(f"Port is open")

# Time: ~1 second (assuming most ports timeout quickly)
# Memory: 100 threads * 1-2 MB each = 100-200 MB
# Context switch overhead is significant but acceptable
```

### Approach 3: AsyncIO

```python
import asyncio

async def is_port_open(host: str, port: int) -> bool:
    try:
        _, writer = await asyncio.wait_for(
            asyncio.open_connection(host, port),
            timeout=1
        )
        writer.close()
        await writer.wait_closed()
        return True
    except:
        return False

async def main():
    hosts = [f"192.168.1.{i}" for i in range(1, 101)]
    ports = list(range(1, 10001))
    
    tasks = [
        is_port_open(host, port)
        for host in hosts for port in ports
    ]
    results = await asyncio.gather(*tasks)
    for i, (host, port, result) in enumerate(zip(
        [h for h in hosts for p in ports],
        [p for h in hosts for p in ports],
        results
    )):
        if result:
            print(f"{host}:{port} is open")

asyncio.run(main())

# Time: ~1 second (same as threads)
# Memory: ~1 MB (one event loop, thousands of coroutines on the heap)
# No context switch overhead; coroutines cooperatively suspend
```

**Summary**:

| Approach | Time | Memory | Pros | Cons |
|----------|------|--------|------|------|
| Sync | ~1M sec | ~1 MB | Simple code | Unusable |
| Threads | ~1 sec | 100-200 MB | Straightforward | Overhead, GIL on CPU |
| AsyncIO | ~1 sec | ~1 MB | Efficient | Need async libraries |
| Multiprocessing | ~10 sec | variable | True parallelism | Slow, IPC overhead |

For I/O-bound work, asyncio and threads have comparable throughput, but asyncio uses orders of magnitude less memory and has no context switch overhead. The cost: you must use async libraries and avoid blocking calls.

---

## 54.9 When to Use What

**Use asyncio when:**
- You have thousands of I/O-bound tasks (network connections, database queries).
- You want to minimize memory and latency variance.
- Your libraries support async (aiohttp, aiopg, asyncpg, motor for MongoDB).

**Use threads when:**
- You have a small number of I/O tasks and thread overhead is acceptable.
- Your I/O libraries are only synchronous (requests, psycopg2, sqlite3).
- You want simplicity over raw efficiency.

**Use multiprocessing when:**
- You have CPU-bound work (data processing, machine learning inference).
- You need to escape the GIL.
- The work is coarse-grained (not millions of tiny tasks).

**Hybrid approach:**
Modern production systems use all three. A typical web server:

```python
# Main app: asyncio for I/O
async def main():
    app = web.Application()
    app.router.add_post('/process', handle_request)
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, 'localhost', 8000)
    await site.start()
    await asyncio.Event().wait()  # Run forever

# A request handler
async def handle_request(request):
    data = await request.json()
    
    # CPU work: offload to thread pool
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(
        None,  # default ThreadPoolExecutor
        expensive_computation,
        data
    )
    
    return web.json_response(result)

# Run it
if __name__ == '__main__':
    asyncio.run(main())
```

---

## 54.10 Common Misconceptions

**"asyncio makes Python multi-core."**

No. asyncio is single-threaded. One event loop = one thread = one core. The GIL is not released. For multi-core work, use multiprocessing.

**"Await means the coroutine waits in the background while other code runs."**

Partially misleading. `await` suspends the coroutine; the event loop resumes it later. But there is no "background"—the coroutine makes no progress until the loop calls its `_step()` method again.

**"Asyncio is faster than threads."**

No. For the same I/O workload, asyncio and threads have similar throughput. asyncio is more memory-efficient and has more predictable latency (no context switches), but raw speed is comparable.

**"You cannot call synchronous code from async code."**

You can, via `asyncio.to_thread()` or `loop.run_in_executor()`. The synchronous function runs in a thread pool, and the event loop is not blocked.

**"Cancellation in asyncio is instant."**

Wrong. Cancellation is cooperative. A coroutine can only be cancelled at an `await` point. If a coroutine is in a CPU-bound section (no await), it will not be cancelled until it hits an `await`.

```python
async def task():
    # This CPU work cannot be cancelled
    result = sum(1 for i in range(10**8))
    await asyncio.sleep(0)  # Now it can be cancelled here
```

**"You should always use asyncio over threads."**

Not true. For small programs, threads are simpler and have less code overhead. asyncio shines at massive concurrency (thousands of tasks). For 10–100 concurrent I/O operations, threads are often clearer.

---

## 54.11 Exercises

1. **Coroutine Introspection.** Write a coroutine that prints which line it is on each time it suspends. Use `inspect.getcoroutinestate()` to check the state (CORO_CREATED, CORO_RUNNING, CORO_SUSPENDED, CORO_CLOSED). Verify that state changes as you await multiple times.

2. **Event Loop Simulation.** Implement a minimal asyncio-like event loop that can run two coroutines concurrently. Use `send()` to resume coroutines and catch `StopIteration` to detect completion. Demonstrate that two tasks interleave fairly.

3. **Port Scanner.** Implement the port-scanner example (54.7) in all three ways: sync, threads, and asyncio. Benchmark them on a real network (or localhost with a test server). Compare time, memory, and CPU usage.

4. **Backpressure Bug.** Create an unbounded producer-consumer with asyncio. Show that memory grows unbounded if the producer is much faster than the consumer. Then fix it with a bounded queue and measure that memory stabilizes.

5. **Race Condition in Asyncio.** Implement the lost-update bug from 54.5. Show the final counter is less than the number of increments. Then fix it with `asyncio.Lock` and verify correctness.

6. **Mixed Async/Sync.** Write a program that fetches 100 URLs concurrently (asyncio) and computes the average response size using CPU-bound code. Use `loop.run_in_executor()` for the CPU work. Verify that the event loop is not blocked.

---

## 54.12 Summary

**asyncio is a single-threaded event loop that multiplexes thousands of coroutines.** Each coroutine is a state machine allocated on the heap. The loop runs one coroutine until it hits an `await`, suspends it, and moves to the next ready coroutine. When the kernel signals that I/O is ready (via `epoll`, `kqueue`, or `select`), the waiting coroutine is re-queued.

**Tasks and Futures are the building blocks.** A Future is a placeholder for a value. A Task wraps a coroutine and drives it forward. When you `await` something, you are waiting for a Future to be resolved. The event loop wakes you when it is.

**asyncio is efficient for I/O-bound work at massive scale.** Ten thousand coroutines consume a few megabytes of memory and require no context switching. For the same workload, threads would consume hundreds of megabytes and pay context switch overhead. The tradeoff: asyncio is single-threaded (no CPU parallelism) and requires async-aware libraries.

**CPU-bound work breaks asyncio.** A single CPU-bound coroutine blocks the entire loop. Move such work to a thread pool with `asyncio.to_thread()` or a process pool with `loop.run_in_executor()`.

**The GIL is still there.** asyncio does not release the GIL. For true parallelism on multiple cores, use multiprocessing.

---

> **[← Previous: Coroutines](08-coroutines.md)**  ·  **[↑ Part 5](README.md)**  ·  **[Next: Thread Safety →](10-thread-safety.md)**
