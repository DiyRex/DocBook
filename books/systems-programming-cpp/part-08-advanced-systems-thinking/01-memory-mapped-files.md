# Chapter 78 — Memory-Mapped Files

When you `mmap` a file, something remarkable happens: the kernel lets you treat a file as a region of memory. You do not call `read()` or `write()`. You do not manage buffers. You do not think about file positions or EOF. Instead, you get back a pointer into your address space that refers to the file's contents. The kernel handles all the I/O — lazily, on-demand, with the same page-fault machinery that backs heap memory and the stack.

This single primitive — `mmap(2)` on POSIX, `CreateFileMappingW` on Windows — underpins databases (SQLite, LMDB, BoltDB), debuggers, fast file parsing, inter-process communication, and most "I made my code fast" stories. It is also one of the most commonly misunderstood tools in systems programming. This chapter will show you how it works, when to use it, and how to avoid its traps.

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain how `mmap` maps a file into your address space and why the kernel does not load the entire file into RAM upfront.
2. Distinguish between file-backed and anonymous mappings, and between shared and private mappings — and predict the behavior of each.
3. Understand page faults, their costs, and why a cascade of major faults in your hot path is catastrophic.
4. Know when `mmap` outperforms `read`/`write`, and when it does not.
5. Recognize common `mmap` traps: files that grow during use, `SIGBUS`, sparse files, and concurrent writer issues.
6. Read a memory-mapped database design (LMDB, SQLite mmap mode) and understand why memory-mapping is central to its speed.
7. Benchmark `mmap` vs `read` on your own files and make the right choice for your workload.

---

## How `mmap` Actually Works

The system call is deceptively simple:

```cpp
#include <sys/mman.h>

void* mmap(void* addr,           // hint (usually NULL)
           size_t length,         // how many bytes to map
           int prot,              // PROT_READ, PROT_WRITE, PROT_EXEC
           int flags,             // MAP_SHARED, MAP_PRIVATE, etc.
           int fd,                // file descriptor
           off_t offset);         // byte offset in file
```

You give it a file descriptor, a length, and some flags. The kernel returns a virtual address. That is all. But the magic is in what happens next.

### Page Faults and Demand Paging

The key insight: **the kernel does not read the file into memory when `mmap` returns**. It could not, even if it wanted to — the file might be larger than available RAM. Instead, the kernel:

1. Records a mapping in your process's page table: "virtual addresses X to X+N are backed by file F from byte offset O."
2. Marks those pages as **not present** in physical memory.
3. Returns the virtual address.

Your program can now read and write those addresses as if they were regular memory. But the first time you access a byte, the CPU triggers a **page fault** — a hardware exception that traps to the kernel.

The kernel then:

1. Checks the page table and sees: "this address is backed by file F, offset O."
2. Reads the file page (usually 4096 bytes, or a larger page size if configured) from disk into a physical page frame.
3. Updates the page table to mark the page as present and pointing to that frame.
4. Returns from the exception.
5. The CPU retries the load/store instruction. This time it hits.

Subsequent accesses to that page are just memory reads — they are in the page cache, backed by physical RAM. No more file I/O until that page is evicted.

### Diagram: The Path of a Byte

Consider reading `file.bin` via `mmap`:

```
Process address space:
  [0x7000_0000] ← start of mmap'd region
  [0x7000_1000] ← end of first page
  [0x7000_2000] ← ... 

First access to offset 0:
  CPU fetches byte at 0x7000_0000 → page table lookup → "not present"
  → hardware exception → kernel trap
  → kernel checks: "this address is in an mmap'd region of file.bin, offset 0"
  → kernel allocates a physical frame (say, frame 0xABCD_0000)
  → kernel reads 4096 bytes from file.bin offset 0 → disk I/O
  → kernel updates page table: 0x7000_0000 → 0xABCD_0000 (present=1)
  → return from exception
  → CPU retries the load → memory hit

Second access to offset 100 (same page):
  CPU fetches byte at 0x7000_0064 → page table lookup → "present at 0xABCD_0064"
  → direct memory access, < 10 nanoseconds

Access to page 2, offset 4100:
  CPU fetches byte at 0x7000_1004 → page table lookup → "not present"
  → hardware exception → kernel → repeat above
```

This is called **demand paging**. You only pay for the pages you actually touch. If your 1 GB file maps but you only read the first 100 KB, you only incur 100 KB worth of I/O (and RAM use).

---

## Anonymous vs File-Backed Mappings

### File-Backed Mapping

What we just described is a **file-backed mapping**:

```cpp
int fd = open("data.bin", O_RDONLY);
void* p = mmap(nullptr, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
// p now refers to the file's contents
```

The pages are initially backed by the file. When you access them, the kernel reads them from disk. If you modify them (with `PROT_WRITE` and `MAP_PRIVATE`), those changes are written to a private copy (copy-on-write), not to the file.

### Anonymous Mapping

An **anonymous mapping** has no file; it is just zero-filled memory:

```cpp
void* p = mmap(nullptr, 1000000, PROT_READ | PROT_WRITE, 
               MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
// p points to 1 MB of zero-filled memory
```

No file descriptor needed (`fd = -1`). The pages do not initially exist in RAM; they are allocated on-demand, zero-filled. On first access, a page fault triggers, and the kernel marks a page as present with zeros.

This is how large allocators use `mmap` for the heap. When you call `malloc(1 GB)`, the C library might call `mmap(MAP_ANONYMOUS)` and get back a 1 GB virtual address range. The pages are not committed until you write to them. If you never write to large chunks, they do not consume RAM.

---

## Shared vs Private: `MAP_SHARED` and `MAP_PRIVATE`

This determines what happens when you write to a mapped region.

### `MAP_PRIVATE`: Copy-on-Write

```cpp
int fd = open("data.bin", O_RDONLY);  // file is read-only
void* p = mmap(nullptr, 100, PROT_READ | PROT_WRITE, 
               MAP_PRIVATE, fd, 0);
*(int*)p = 999;  // write to the mapping
```

The write does not go to the file. Instead:

1. The kernel detects the write to a page marked `MAP_PRIVATE`.
2. It makes a private copy of that page in RAM (copy-on-write).
3. The mapping now points to the copy.
4. The file is untouched.

This is why you can `mmap` a read-only file with `PROT_WRITE` — the changes stay private to your process.

**Use case:** multiple processes read the same file but each wants its own modifications. Each process gets its own copy of modified pages, but unmodified pages are shared (saving RAM).

### `MAP_SHARED`: Shared Modifications

```cpp
int fd = open("data.bin", O_RDWR);
void* p = mmap(nullptr, 100, PROT_READ | PROT_WRITE, 
               MAP_SHARED, fd, 0);
*(int*)p = 999;  // write to the mapping
```

The write goes directly to the page in the page cache. The next `read()` of that file (from any process, any time) sees the changed data. When that page is written back to disk (by the kernel's writeback daemon, or by explicit `msync()`), the file is updated.

**Use case:** inter-process communication. Two processes map the same file as `MAP_SHARED`. When one writes, the other sees the changes instantly (via the page cache). This is much faster than `write()` + `read()`.

### Concrete Example

Process A and B both map `shared.txt` as `MAP_SHARED`:

```
Process A:                        Process B:
void* p = mmap(..., MAP_SHARED)   void* p = mmap(..., MAP_SHARED)
*(int*)p = 42                     // A writes 42
                                  sleep(1)
                                  printf("%d\n", *(int*)p)  // prints 42
```

B sees A's write instantly, without any syscall. The kernel's page cache makes this work. For a truly concurrent application (game engine, real-time simulation, database), `MAP_SHARED` can be orders of magnitude faster than `read()`/`write()` message passing.

---

## When `mmap` Beats `read`/`write`

`mmap` is not universally better. Consider these scenarios:

### mmap is superior:

1. **Random access to large files**: If you want to jump to offset 500 MB of a 2 GB file, `lseek()` + `read()` works, but `mmap` is simpler and avoids explicit seeking and buffer management.

   ```cpp
   // With read/write
   lseek(fd, 500'000'000, SEEK_SET);
   char buf[4096];
   read(fd, buf, 4096);
   // ...
   
   // With mmap
   char* p = mmap(..., 2'000'000'000, ..., fd, 0);
   memcpy(somewhere, p + 500'000'000, 4096);  // just a pointer + memcpy
   ```

2. **Multiple processes sharing data**: `MAP_SHARED` with a file is faster than message queues, pipes, or sockets because data goes through the page cache, not the socket/pipe buffers.

3. **Database engines**: Treating a data file as memory lets you use the CPU's memory-addressing hardware (page tables, TLB, prefetchers) instead of manually managing buffers. LMDB, BoltDB, and SQLite's mmap mode all exploit this.

4. **Large constants or lookup tables**: Loading a 100 MB lookup table? `mmap` it once; the kernel handles all the paging.

### read/write are superior:

1. **Sequential streaming**: If you are reading a 10 GB file sequentially once, `mmap` is no faster than `read()`. In fact, `read()` may be faster because:
   - You control buffer size (tuned for your disk).
   - The kernel's I/O prefetcher may not anticipate mmap's demand pattern as well.
   - You can reuse the same buffer across multiple reads.

2. **Small files**: The overhead of setting up an mmap (page table updates, fault handlers) exceeds the benefit for a 10 KB file. Just `read()` it.

3. **Explicit error handling**: With `read()`, you get a return value and `errno`. With `mmap`, page faults are asynchronous — you get `SIGBUS` on a bad access (e.g., file shrinks). Some systems find that easier to reason about.

4. **Portability**: `mmap` behavior varies by OS (especially Windows vs POSIX). `read()`/`write()` is more standardized.

---

## Page Faults and Their Cost

A page fault is expensive. Understanding the cost tiers is crucial for tuning:

### Major Fault (page-in from disk)

A byte you access is not in the page cache. The kernel must read it from disk.

- **Cost**: 10–50 milliseconds (a disk seek + read).
- **In CPU cycles**: ~50 M cycles (at 3 GHz).
- **Equivalently**: the cost of ~1 million arithmetic instructions.

If you have a hot loop that triggers major faults, you are dead. Your CPU spends most of its time waiting.

### Minor Fault (already in page cache, just not mapped)

A byte is in the page cache (because another process accessed it, or the kernel prefetched it), but your page table does not point to it yet.

- **Cost**: < 1 microsecond.
- **In CPU cycles**: ~3,000 cycles.
- **Equivalently**: ~100 arithmetic instructions.

Still expensive, but tolerable in a loop.

### Soft Fault (zero-fill)

An anonymous page that has never been touched. The kernel allocates a zero page.

- **Cost**: < 1 microsecond.
- **In CPU cycles**: ~1,000–2,000 cycles.

Cheaper than a minor fault because no I/O is involved.

### Hit (page is already mapped and present)

- **Cost**: ~10 nanoseconds.
- **In CPU cycles**: ~30 cycles.

This is the goal. Once a page is mapped, access is memory-speed.

### Example: The Cost of Unpredicted Faults

```cpp
void process_file_bad(const char* path) {
    int fd = open(path, O_RDONLY);
    void* p = mmap(nullptr, 1'000'000'000, PROT_READ, MAP_PRIVATE, fd, 0);
    
    // Random access to the file
    for (int i = 0; i < 1'000'000; ++i) {
        int offset = rand() % 1'000'000'000;
        int value = *(int*)(p + offset);  // random access
        // ...
    }
}
```

With 1 million random accesses to a 1 GB file, you will fault on nearly every access (different pages). If the page cache is cold, each fault is a major fault. Cost: ~1 million × 50 ms = **50,000 seconds**, or ~14 hours.

Compare to sequential access:

```cpp
void process_file_good(const char* path) {
    int fd = open(path, O_RDONLY);
    char buf[65536];
    ssize_t n;
    long long sum = 0;
    while ((n = read(fd, buf, 65536)) > 0) {
        for (int i = 0; i < n / 4; ++i) {
            sum += ((int*)buf)[i];
        }
    }
}
```

Sequential I/O is prefetched by the kernel. Most accesses are cache hits. Cost: ~1 GB / (disk speed, say 200 MB/s) = ~5 seconds.

The lesson: **mmap is fast when your access is predictable and hits the page cache. Random access to cold pages is catastrophic.**

---

## Memory-Mapped Databases

The reason databases care about `mmap` is simple: if your data is in the page cache and accessed via normal memory instructions, the CPU's caches and prefetchers help you automatically.

### LMDB and Copy-on-Write

LMDB (Lightning Memory-Mapped Database) is a key-value store that `mmap`'s its entire data file as `MAP_SHARED`. The data is organized as B-trees. To update:

1. The writer creates a new B-tree node on the mmap'd region.
2. Because the mapping is `MAP_SHARED` with copy-on-write, writes do not go to the file immediately.
3. The writer atomically swaps the tree pointer (a single write to a known offset).
4. Readers see the old tree until the swap is visible; after, they see the new tree.

The result: readers are never blocked; writers do not block readers. The OS handles all the paging.

### SQLite in mmap Mode

SQLite traditionally uses `read()` and `write()`. In recent versions, you can enable mmap mode:

```cpp
// Enable mmap for 30 MB of the database file
sqlite3_exec(db, "PRAGMA mmap_size = 30000000;", nullptr, nullptr, nullptr);
```

SQLite then memory-maps that portion of its database file. Reads become pointer dereferences; writes go through the page cache. This is faster for small-to-medium databases where the working set fits in available RAM.

---

## Worked Example: Scanning a File with `mmap`

Let's compare three approaches: `read()`, `mmap`, and hand-optimized `mmap`.

### Approach 1: Traditional `read()`

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <cstdio>

long long count_pattern_read(const char* path, int pattern) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return -1;
    
    char buf[1 << 20];  // 1 MB buffer
    ssize_t n;
    long long count = 0;
    
    while ((n = read(fd, buf, sizeof(buf))) > 0) {
        for (ssize_t i = 0; i < n; ++i) {
            if (buf[i] == pattern) count++;
        }
    }
    
    close(fd);
    return count;
}
```

Pro: simple, explicit control over buffering, works on any file size.
Con: requires copying bytes from kernel buffers to user buffers.

### Approach 2: Simple `mmap`

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <cstdio>

long long count_pattern_mmap(const char* path, int pattern) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return -1;
    
    struct stat st;
    if (fstat(fd, &st) < 0) { close(fd); return -1; }
    
    void* p = mmap(nullptr, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (p == MAP_FAILED) { close(fd); return -1; }
    
    long long count = 0;
    unsigned char* data = (unsigned char*)p;
    for (off_t i = 0; i < st.st_size; ++i) {
        if (data[i] == pattern) count++;
    }
    
    munmap(p, st.st_size);
    close(fd);
    return count;
}
```

Pro: simpler code, no buffer management.
Con: byte-by-byte access is cache-hostile; triggers more page faults.

### Approach 3: Optimized `mmap`

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <cstdio>

long long count_pattern_mmap_opt(const char* path, int pattern) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return -1;
    
    struct stat st;
    if (fstat(fd, &st) < 0) { close(fd); return -1; }
    
    void* p = mmap(nullptr, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (p == MAP_FAILED) { close(fd); return -1; }
    
    long long count = 0;
    unsigned char* data = (unsigned char*)p;
    unsigned char* end = data + st.st_size;
    
    // Process in chunks to improve cache locality
    constexpr size_t chunk = 1 << 20;  // 1 MB chunks
    unsigned char* ptr = data;
    while (ptr < end) {
        unsigned char* chunk_end = std::min(ptr + chunk, end);
        while (ptr < chunk_end) {
            if (*ptr == pattern) count++;
            ptr++;
        }
    }
    
    munmap(p, st.st_size);
    close(fd);
    return count;
}
```

The chunking hints to the kernel that we are doing sequential access and helps with prefetching.

### Benchmark (on a 1 GB file, counting bytes)

On a typical system, with a cold page cache:

```
read():           ~2.5 seconds    (sequential I/O, kernel-paced)
mmap() simple:    ~4.5 seconds    (byte-by-byte, cache misses)
mmap() optimized: ~2.8 seconds    (chunked, better locality)
```

With a warm page cache:

```
read():           ~0.8 seconds
mmap() simple:    ~0.5 seconds    (memory-speed access)
mmap() optimized: ~0.4 seconds    (good cache locality)
```

The lesson: once data is hot in cache, `mmap` wins. But access pattern (sequential, chunked) matters as much as the I/O mechanism.

---

## Pitfalls

### 1. File Grows or Shrinks During Use

If you mmap a file at 1 MB, and the file later grows to 2 MB, your mapping still covers only 1 MB. Accesses beyond that are undefined (likely SIGBUS).

```cpp
// Bad: file might grow
void* p = mmap(nullptr, file_size, ..., fd, 0);
// ... meanwhile, another process extends the file
// If you access beyond file_size, SIGBUS
```

Solution: if the file can grow, either:
- Mmap the expected maximum size (wastes virtual address space if the file is small).
- Re-mmap periodically (expensive).
- Use `MAP_SHARED` and `msync()` carefully; monitor file size changes.

### 2. `SIGBUS` on Truncation

If the file is truncated while mapped:

```cpp
void* p = mmap(nullptr, 1000, PROT_READ | PROT_WRITE, ..., fd, 0);
// Process B truncates the file to 100 bytes
// You access offset 500 → SIGBUS
```

The signal kills the process by default. You can catch it, but it is not a normal control flow. Prefer `read()` + explicit EOF handling if truncation is possible.

### 3. Sparse Files and Fragmentation

A sparse file (one with holes) will trigger major faults when you access the holes, even if the file's logical size is huge. The OS must zero-fill the page, but the disk does not have data there.

```cpp
// Create a sparse file
int fd = open("sparse.bin", O_CREAT | O_WRONLY, 0644);
lseek(fd, 10'000'000'000LL, SEEK_SET);
write(fd, "x", 1);  // file is now 10 GB (9.99... GB is holes)
close(fd);

// Now mmap it
void* p = mmap(nullptr, 10'000'000'000LL, PROT_READ, ..., fd, 0);
// Accessing the holes triggers major faults (zero-fill)
```

### 4. Cache Coherence with Concurrent Writers

If multiple processes map the same file as `MAP_SHARED` and write, the kernel's cache coherence keeps them consistent *within the page cache*. But:

- If process A writes via `mmap` and process B reads via `read()`, B may see stale data (depends on writeback timing).
- If process A writes via `write()` and process B reads via `mmap`, B may see stale data.

Solutions: use `msync(MS_SYNC)` before handing off, or use read-write locks, or ensure all access is via mmap.

### 5. Not All Filesystems Support All Flags

Some filesystems (tmpfs, NFS, FAT32) have limited mmap support. `MAP_SHARED | PROT_WRITE` may not work on NFS without explicit configuration. Always check `mmap`'s return value.

---

## Common Misconceptions

### Misconception 1: "mmap loads the entire file into RAM"

No. `mmap` maps virtual addresses to the file. Pages are loaded lazily, on demand. If you never access a byte, it never enters RAM. If your access is sequential and the page cache is warm, you never pay for disk I/O either.

### Misconception 2: "mmap is always faster than read/write"

No. `mmap` is faster when:
- Data fits in the page cache (already in RAM).
- You access it randomly or in patterns the prefetcher can anticipate.
- You want multiple processes to share data.

`read()`/`write()` is faster when:
- You are streaming the file once, sequentially.
- You can control buffer size and reuse.
- The file is very large and does not fit in cache.
- You want explicit error handling via return values.

### Misconception 3: "mmap is simpler than read/write"

Maybe, for the happy path. But `mmap` error handling is messier: SIGBUS, file truncation, growing files, cache coherence. `read()` returns `0` for EOF and `-1` for errors. Simple. `mmap` requires vigilance.

### Misconception 4: "I should use mmap for IPC instead of pipes/sockets"

Maybe. `MAP_SHARED` is fast for data passing, but:
- No automatic signaling (reader must poll or use eventfd).
- No framing (pipes and sockets have message boundaries).
- Synchronization is your problem (you need locks or atomics).

Use `mmap` for high-frequency data sharing between trusted processes. Use pipes/sockets for general IPC.

### Misconception 5: "mmap is portable; I can use it everywhere"

It is POSIX, but behavior varies by OS. Windows `CreateFileMappingW` is similar but not identical. Linux's `mremap`, madvise tricks, and huge pages are Linux-specific. Write portable code or test aggressively on your target platforms.

---

## Tradeoffs

| Approach | Pros | Cons | Best for |
|---|---|---|---|
| **read/write** | Simple, portable, explicit EOF/error, good for streaming | Extra buffer copying, must manage position | Sequential files, small files, explicit error flow |
| **mmap (MAP_PRIVATE)** | Simple pointer access, lazy loading, zero-copy for reads | Byte-by-byte access is cache-hostile, SIGBUS on shrink, file-grow issues | Large random-access files, lookup tables, single-process |
| **mmap (MAP_SHARED)** | Zero-copy IPC, fast multi-process data sharing, cache coherence automatic | Complex error handling, SIGBUS, truncation issues, sync is your problem | Databases, real-time IPC, shared buffers between trusted processes |
| **Custom buffer pool** | Full control over I/O paging, cache-aware tuning, error predictability | Complex to implement well, reinvents allocators | Specialized workloads, when measurement says mmap is a bottleneck |

---

## Exercises

1. **Implement count_pattern with all three approaches** (read, mmap simple, mmap optimized). Generate a 1 GB file with random bytes. Benchmark all three on cold cache (drop caches with `echo 3 > /proc/sys/vm/drop_caches`, Linux only) and warm cache. Which is fastest in each scenario? Why?

2. **Implement a simple memory-mapped key-value store.** Use a single file, mmap it as `MAP_SHARED | MAP_PRIVATE`, and implement `put(key, value)` and `get(key)`. Use a simple hash table layout. Test with two processes — one writing, one reading. Do reads see writes in real time?

3. **Observe page faults with `perf`.** Write a program that randomly accesses a large mmap'd file. Run with `perf stat -e page-faults,major-faults` and compare to sequential access. How many major faults for random access? Sequential?

4. **Test MAP_PRIVATE vs MAP_SHARED.** Create a file, mmap it two ways in the same process, write different values, and observe the behavior. Does the file change? What does each process see?

5. **Truncate a file and catch SIGBUS.** Write a program that mmap's a file, then (from another process) truncates it. Set up a SIGBUS handler and observe. Is catching SIGBUS easier than detecting shrinkage another way?

6. **Benchmark mmap'd database access.** Use a library like LMDB or SQLite with mmap enabled. Insert 10 million random key-value pairs. Measure insert time, lookup time, and memory use. Compare to a traditional in-memory hash table. Why is the mmap'd version slower for inserts but competitive for lookups?

7. **Analyze a real-world mmap use.** Pick a library (e.g., jemalloc uses mmap for large allocations, or LLVM uses mmap for object files). Read the relevant code and explain why mmap was the right choice.

---

## Summary

Memory-mapped files are a powerful tool for treating large files as memory, leveraging the OS's page cache and the CPU's memory hierarchy. They are invaluable for databases, multi-process IPC, and random access to large data. But they are not a silver bullet: sequential I/O may be faster, error handling is messier, and correctness requires vigilance around file growth, truncation, and cache coherence.

Use `mmap` when you have random access, multiple processes, or a working set larger than cache. Use `read()`/`write()` for streaming, small files, and simple error handling. Measure both on your workload before deciding. The difference between the two is often within a 2× factor; do not let ideology guide you — let benchmarks.

---

> **[← Previous: Part 7 — Building Real Systems](../part-07-building-real-systems/README.md)**  ·  **[↑ Part 8](README.md)**  ·  **[Next: Networking Internals →](02-networking-internals.md)**
