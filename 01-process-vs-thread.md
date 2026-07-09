# 1. Process vs Thread - Deep Dive

## What is a Process?

A **process** is an independent execution environment managed by the operating system.

Each process has:

- Its own **virtual memory space** (isolated heap, stack, data segment)
- Its own **file descriptors** (open files, sockets)
- Its own **security context** (permissions, user ID)
- One or more threads

### Real-World Example

```
Chrome Browser

├── Process 1 (Main Browser)
│   ├── Thread 1 (UI)
│   ├── Thread 2 (Network)
│   └── Thread 3 (Rendering)
│
├── Process 2 (Tab 1)
│   ├── Thread 1 (Page Execution)
│   └── Thread 2 (Worker)
│
├── Process 3 (Tab 2)
└── Process 4 (GPU Rendering)
```

### Key Insight: Isolation

If **Process 2 crashes**, the others continue running.

This is **process isolation** — the OS guarantees that one process cannot directly corrupt another process's memory.

---

## What is a Thread?

A **thread** is an execution path inside a process.

### Shared Resources (All threads in a process)

Multiple threads **share**:

```
┌─────────────────────────────────┐
│         HEAP (Shared)           │
├─────────────────────────────────┤
│  Customer Objects               │
│  Order Objects                  │
│  Cache (static variables)       │
│  File Handles                   │
│  Database Connection Pools      │
└─────────────────────────────────┘
```

### Per-Thread Resources (Each thread owns)

Each thread has its own:

```
┌──────────────────┐
│   Thread Stack   │
├──────────────────┤
│ Program Counter  │  (current instruction)
│ Registers        │  (CPU state)
│ Local Variables  │  (isolated)
└──────────────────┘
```

### Visual: Thread vs Process Memory

```
Process Address Space

┌───────────────────────────┐
│    Shared Heap            │
│                           │
│  Customer order = new ... │
│  Cache cache = new ...    │
│  static int counter = 0   │
└───────────────────────────┘
        ↑
    Shared by all threads

┌─────────────────┬─────────────────┐
│  Thread A Stack │  Thread B Stack │
├─────────────────┼─────────────────┤
│ localVar = 10   │ localVar = 20   │
│ String name;    │ String name;    │
│                 │                 │
│ (isolated)      │ (isolated)      │
└─────────────────┴─────────────────┘
```

---

## Why are Threads Faster to Create?

### Creating a Process is Expensive

```
1. Allocate virtual memory space
2. Create page tables
3. Load program binary into memory
4. Set up security context
5. Initialize file descriptors
6. Register with OS scheduler
```

**Time**: Milliseconds to microseconds.

**Memory**: Megabytes per process.

### Creating a Thread is Cheap

```
1. Allocate a stack (usually 1-8 MB)
2. Register with OS scheduler
3. Done
```

**Time**: Microseconds.

**Memory**: Kilobytes per thread.

### Context Switching Overhead

**Between processes**: Flush CPU cache, update memory management unit (MMU), reload page tables.

**Between threads**: Just switch stacks and registers (same address space, so no page table reload).

---

## Interview Question: "Why not create one thread per request?"

### Poor Answer
> "Too many threads would be created."

**Problem**: Vague. Doesn't explain WHY it's a problem.

### Good Answer
> "Each thread consumes memory for its stack and kernel resources. Thousands of active threads increase context switching overhead, cache misses, and scheduling latency. Thread pools are more efficient."

### Senior Answer
> "Platform threads have fixed overhead: typically 1-8 MB per thread for stack allocation, plus kernel thread structures. At scale, this creates three problems:
>
> 1. **Memory**: 10,000 threads × 2 MB per thread = 20 GB just for stacks.
> 2. **Context switching**: The OS scheduler must switch between thousands of runnable threads, causing cache thrashing and CPU pipeline flushes.
> 3. **Latency**: With high context switching overhead, request latency increases.
>
> Thread pools (fixed size) or virtual threads (Java 21) solve this by reusing threads or eliminating the 1:1 thread-per-request model. The choice depends on workload: CPU-bound work prefers small thread pools; I/O-bound work with virtual threads can scale to millions of lightweight tasks."

---

## Key Differences Table

| Aspect | Process | Thread |
|--------|---------|--------|
| **Memory Space** | Independent virtual address space | Shared heap, isolated stack |
| **Isolation** | Complete (OS-enforced) | Partial (cooperation required) |
| **Creation Cost** | High (milliseconds) | Low (microseconds) |
| **Context Switch** | Expensive (flush caches, reload MMU) | Cheap (same address space) |
| **Communication** | IPC (pipes, sockets, shared memory) | Direct (shared heap) |
| **Crash Impact** | Isolated (other processes unaffected) | Affects entire process |
| **Number Practical Limit** | Dozens (resource-constrained) | Thousands (thread-pool sized) |

---

## Real Scenario: A Web Server

### Thread-Per-Request Model (Historical)

```
Client 1 → Thread 1
Client 2 → Thread 2
Client 3 → Thread 3
...
Client 10,000 → Thread 10,000
```

**Problem**: 10,000 threads = massive memory, scheduling overhead.

### Thread Pool Model (Standard)

```
Client 1 ┐
Client 2 ├─→ Thread Pool (50 threads) → Queue pending requests
Client 3 ┤
...       └
```

**Advantage**: Reuse threads, bound resource usage.

### Virtual Threads Model (Java 21+)

```
Client 1 ┐
Client 2 ├─→ Virtual Threads (millions) → OS threads (limited)
Client 3 ┤
...       └
```

**Advantage**: Lightweight tasks that suspend on I/O, efficient scheduling.

---

## Summary

1. **Processes** are OS-managed, completely isolated, heavyweight.
2. **Threads** are lightweight execution units that share a process's heap.
3. The tradeoff: Threads are cheaper but require careful synchronization.
4. Modern solutions: Thread pools or virtual threads, not one thread per request.

---

## Questions to Verify Understanding

1. Why can't one thread's corruption affect another thread's local variables?
2. Why does a thread pool use fewer resources than creating a new thread per request?
3. What happens if a thread reads from the shared heap while another thread modifies it?
4. Why is process isolation more secure than thread isolation?
5. In a 10-core CPU, why would creating 1,000 threads cause performance degradation?
