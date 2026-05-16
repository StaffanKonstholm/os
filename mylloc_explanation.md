# Explanation of mylloc.pdf — "My malloc: mylloc and mhysa"

*Author: Johan Montelius, HT2016*

---

## Overview

This document is a lab exercise that guides you through building your own `malloc` and `free` in C. The goal is not to build the fastest allocator, but to understand what a memory allocator must do and how it interacts with the OS kernel. Because `malloc` is a **user-space library function**, it is easy to shadow the standard implementation by supplying your own object file at link time.

---

## 1. The Interface

The standard contract for `malloc` and `free` (from the man pages):

- `malloc(size)` — allocates `size` bytes and returns a pointer to uninitialized memory. Returns `NULL` on failure or when `size == 0`.
- `free(ptr)` — releases memory previously returned by `malloc`/`calloc`/`realloc`. Calling `free(NULL)` is a no-op. Double-freeing is undefined behavior.

---

## 2. mylloc.c — The Trivial Version

### 2.1 The Simplest Possible Allocator

The first implementation simply calls `sbrk(size)` every time memory is requested, and does nothing in `free`:

```c
void *malloc(size_t size) {
    if (size == 0) return NULL;
    void *memory = sbrk(size);
    if (memory == (void *)-1) return NULL;
    return memory;
}

void free(void *memory) { return; }
```

`sbrk(n)` asks the OS to grow the heap by `n` bytes and returns the old break pointer. It fails (returns `-1`) only when the OS cannot extend the heap further.

**Problem:** `free` never reclaims memory, so every allocation consumes new heap space. The heap grows without bound until the OS runs out of virtual memory or kills the process (the Linux *OOM Killer*).

### 2.2 The Benchmark (`bench.c`)

A simple benchmark to observe heap growth:
- Runs `ROUNDS × LOOP` iterations.
- Each iteration allocates a random-sized block, writes to it, and frees it.
- After each round it calls `sbrk(0)` to measure how much the heap has grown.

Compile by providing the custom object file *before* the standard library so the linker picks up the custom `malloc`:

```sh
gcc -c mylloc.c
gcc -o bench mylloc.o bench.c
```

### 2.3 Observing the Problem

With `ROUNDS = 100` the benchmark will fail with `malloc failed` or be killed by the OOM killer. If you comment out the write to memory and increase rounds to 1000, the virtual address space grows rapidly even though no physical pages are touched — this demonstrates lazy (on-demand) page allocation.

### 2.4 Realistic Block-Size Distribution (`rand.c`)

Real programs request small blocks far more often than large ones. The benchmark is improved with an exponential distribution:

```
size = MAX / e^r,   where r ∈ [0, log(MAX/MIN)]
```

This gives sizes in `[MIN, MAX]` with a bias toward smaller values. Implemented in `rand.c` and used via `rand.h`. Compile with the math library:

```sh
gcc -c rand.c
gcc -o bench mylloc.o rand.o bench.c -lm
```

### 2.5 Buffer of Live Blocks (`bench.c` update)

A more realistic benchmark maintains a fixed-size buffer of currently-allocated pointers. Each iteration randomly picks a slot:
- If occupied → free it.
- If empty → allocate a new block.

This simulates real programs where some objects are long-lived and some are short-lived.

```sh
gcc -o bench mylloc.o rand.o bench.c -lm
```

Compare against the standard library (omit `mylloc.o`):

```sh
gcc -o bench rand.o bench.c -lm
```

---

## 3. mhysa.c — Free-List Allocator

### The Core Idea

To reuse freed blocks, the allocator maintains a **singly-linked free list**. The challenge is that `free(ptr)` receives no size information. The trick is to allocate *slightly more memory than requested* and store a hidden header (`struct chunk`) immediately before the returned pointer.

```
┌───────────────────┬────────────────────────────────┐
│  struct chunk     │  usable memory (returned ptr)  │
│  size | next      │                                │
└───────────────────┴────────────────────────────────┘
```

### The `chunk` Structure

```c
struct chunk {
    int size;
    struct chunk *next;
};

struct chunk *flist = NULL;  // global free list head
```

### `free()` — Add to Free List

```c
void free(void *memory) {
    if (memory != NULL) {
        struct chunk *cnk = (struct chunk *)((struct chunk *)memory - 1);
        cnk->next = flist;
        flist = cnk;
    }
}
```

By subtracting one `chunk`-sized step from the user pointer, we recover the hidden header and prepend it to the free list.

### `malloc()` — Search Free List First

```c
void *malloc(size_t size) {
    if (size == 0) return NULL;

    // Search for a free block large enough
    struct chunk *next = flist, *prev = NULL;
    while (next != NULL) {
        if (next->size >= size) {
            if (prev != NULL) prev->next = next->next;
            else              flist = next->next;
            return (void *)(next + 1);
        }
        prev = next;
        next = next->next;
    }

    // No suitable block found — ask the kernel
    void *memory = sbrk(size + sizeof(struct chunk));
    if (memory == (void *)-1) return NULL;
    struct chunk *cnk = (struct chunk *)memory;
    cnk->size = size;
    return (void *)(cnk + 1);
}
```

Key points:
- We request `size + sizeof(struct chunk)` bytes from `sbrk` to make room for the header.
- The header stores the actual usable size so we can match it on reuse.
- The returned pointer points *past* the header.

Compile and test:

```sh
gcc -c mhysa.c
gcc -o bench rand.o mhysa.o bench.c -lm
./bench
```

---

## 4. How to Improve

### 4.1 Counting System Calls (`strace`)

```sh
strace ./bench 2>&1 >/dev/null | grep brk | wc -l
```

Compare the number of `brk` system calls for: standard library, `mylloc.o`, and `mhysa.o`. mhysa makes far fewer kernel calls because it reuses freed memory.

### 4.2 First Fit, Best Fit, Worst Fit

The current search returns the **first** block that fits. Alternatives:

| Strategy | Description | Trade-off |
|---|---|---|
| **First fit** | Use the first block ≥ requested size | Fast search, may waste large blocks |
| **Best fit** | Use the smallest block ≥ requested size | Less waste, slower search |
| **Worst fit** | Use the largest available block | Leaves large remainders for future use |

**Internal fragmentation** — unused bytes inside an allocated block.  
**External fragmentation** — many small free blocks that individually cannot satisfy a large request.

**Block splitting:** If the chosen block is much larger than needed, split it into two chunks. This reduces internal fragmentation.  
**Coalescing:** When two adjacent free chunks exist, merge them into one larger chunk. This counteracts external fragmentation. Splitting without coalescing causes the free list to fill with tiny unusable blocks.

### 4.3 Power-of-Two Size Classes (Keep It Simple)

An alternative to splitting/merging: round every allocation up to the nearest power of two (32, 64, 128, 256, …) and maintain a **separate free list per size class**.

Benefits:
- `malloc`/`free` are O(1).
- No fragmentation within a size class.
- Memory can be allocated in page-sized chunks (multiples of 4096 bytes) and subdivided.

Trade-off: up to ~25% internal fragmentation on average (a 33-byte request uses a 64-byte slot).

### 4.4 Thread Safety

All structures described above are global and unsynchronized. Concurrent access from multiple threads will corrupt the free list. Solutions range from a single global mutex (simple but slow) to per-thread free lists (fast but wastes memory when one thread frees objects allocated by another).

---

## 5. Summary

| Version | Heap growth | System calls | Complexity |
|---|---|---|---|
| `mylloc` (trivial) | Unbounded | One per allocation | Minimal |
| `mhysa` (free list) | Bounded (reuse freed blocks) | Much fewer | Low |
| Improved (split/coalesce) | Near-optimal | Minimal | Medium |
| Power-of-two classes | Bounded + predictable | Minimal | Low–Medium |

**Key takeaways:**
1. `malloc` and `free` run entirely in **user space** — the kernel only needs to be involved when the heap must grow.
2. Keeping the kernel out of most allocations is the primary performance goal.
3. Metadata (the `chunk` header) is hidden just before the returned pointer — a standard technique used by real allocators (e.g., glibc's `malloc_chunk`).
4. There is an inherent tension between allocation speed, memory utilization, and implementation complexity.
