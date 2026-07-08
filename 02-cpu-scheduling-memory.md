# 2. CPU Scheduling and Memory

## Thread States in Java

The Java thread lifecycle defines six states. But remember: **Java doesn't control when a thread runs. The OS scheduler does.**

```
NEW
  ↓
RUNNABLE (ready to run, or running)
  ↓
BLOCKED (waiting for a monitor lock)
  ↓
WAITING (waiting for another thread to notify)
  ↓
TIMED_WAITING (waiting with a timeout)
  ↓
TERMINATED
```

### Critical Insight

Java exposes `RUNNABLE`, but this state means:

- **Ready to run** (queued in the OS scheduler)
- **Currently running** (executing on a CPU core)

The OS decides which RUNNABLE thread gets CPU time.

---

## Preemptive Scheduling

Modern operating systems use **preemption**: the OS can interrupt a thread and give CPU time to another thread.

### Timeline Example

```
Time 0    Thread A starts running
          a = 1;
          b = 2;

Time 1    TIMER INTERRUPT (e.g., every 10ms)
          OS saves Thread A's state
          OS loads Thread B's state

Time 2    Thread B runs
          c = 3;
          d = 4;

Time 3    TIMER INTERRUPT
          OS saves Thread B's state
          OS loads Thread A's state

Time 4    Thread A resumes (continues after b = 2;)
          c = 5;
```

### The Problem

Your code can stop **between almost any two instructions**.

```java
int x = 10;
int y = 20;
int z = x + y;  // Could be interrupted here
```

Could be seen as:

```
Thread A        Thread B
x = 10
                // interruption
                reads x (gets 10)
                reads y (? not set yet)
y = 20
z = x + y
```

This is why **shared mutable state** is dangerous.

---

## Shared Memory Problem: The Lost Update

### Scenario

```
Account balance = 100

Thread A: withdraw(10)
Thread B: withdraw(20)
```

### Desired Outcome

```
Initial: 100
A withdraws 10: 100 - 10 = 90
B withdraws 20: 90 - 20 = 70
Final: 70
```

### Actual Outcome (Race Condition)

Each thread executes:

```
1. Read balance from memory
2. Compute new balance
3. Write balance to memory
```

Timeline:

```
Time  Thread A              Thread B              Memory
0     read balance (100)                         100
1                           read balance (100)   100
2     compute 100-10=90                          100
3                           compute 100-20=80    100
4     write 90                                    90
5                           write 80              80
```

**Expected: 70**
**Actual: 80**

Thread A's write is lost.

---

## CPU Cache Hierarchy

Many developers think variables live in RAM.

Reality:

```
Main Memory (RAM)
    ↕ (expensive access: ~100 CPU cycles)
L3 Cache (shared across cores: ~40 cycles)
    ↕
L2 Cache (per core: ~10 cycles)
    ↕
L1 Cache (per core: ~4 cycles)
    ↕
CPU Registers (immediate: ~1 cycle)
```

### The Caching Problem

When Thread A modifies `x = 5`:

```
Thread A CPU Core

L1 Cache: x = 5  (not yet in Main Memory)
L2 Cache: (empty)
L3 Cache: (empty)
Main Memory: x = 0  (stale)
```

Thread B, running on a different core:

```
Thread B CPU Core

reads x from L1 Cache: x = 0  (hasn't seen Thread A's write)
```

### Classic Example

```java
boolean ready = false;

Thread worker = new Thread(() -> {
    while (!ready) {
        // spin-wait
    }
    System.out.println("Worker done");
});

worker.start();

// Main thread
ready = true;
```

### What Happens

**Without synchronization:**

1. Main thread sets `ready = true` in its cache.
2. Worker thread reads `ready` from its cache: still `false`.
3. Worker thread spins forever.

**Why?** The JVM is allowed to assume `ready` won't change and optimize it away.

---

## Memory Visibility

### Question: When does Thread B see changes from Thread A?

Without explicit synchronization:

**Answer: Java gives NO GUARANTEE.**

Thread B might see the change immediately, eventually, or never.

This is why **synchronization primitives** exist.

---

## Instruction Reordering

### The Problem

The compiler and CPU might reorder instructions.

```java
a = 1;
b = 2;
```

Could become:

```java
b = 2;
a = 1;
```

In a single-threaded program, the result is identical (same final state).

### In Multithreading: Disaster

```java
// Thread A
value = 10;
ready = true;

// Thread B
while (!ready) { }
System.out.println(value);  // Expects 10
```

What if the compiler reorders Thread A to:

```java
ready = true;
value = 10;
```

**Timeline:**

```
Thread A: ready = true
Thread B: sees ready = true, prints value
Thread A: value = 10  (too late!)
```

**Output: 0** (or undefined, depending on initialization)

This is why **happens-before relationships** matter.

---

## Code Example: Race Condition

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int get() {
        return count;
    }
}

// Main
Counter counter = new Counter();

Thread t1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});

Thread t2 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});

t1.start();
t2.start();
t1.join();
t2.join();

System.out.println(counter.get());  // Expected: 2000, Actual: ~1500 (varies)
```

### Why It Fails

`count++` is **not atomic**. It's three operations:

```
1. read count
2. increment
3. write count
```

With two threads running concurrently:

```
Thread 1: read (0)
Thread 2: read (0)
Thread 1: increment (1)
Thread 2: increment (1)
Thread 1: write (1)
Thread 2: write (1)
```

**Expected: 2, Actual: 1**

---

## Summary

1. **Preemption**: The OS can interrupt your thread at almost any instruction.
2. **Caching**: Each CPU core has its own cache; writes may not be visible to other cores.
3. **Reordering**: The compiler and CPU can reorder instructions in ways that break multithreaded code.
4. **No Guarantees**: Without synchronization, there are no guarantees about visibility, ordering, or atomicity.

---

## Interview Questions

1. **Why can a thread "starve" if it never gets CPU time?**
   - If the OS scheduler never selects that thread, it remains in RUNNABLE but never RUNNING.

2. **In a 4-core system with 8 runnable threads, how many are actually running?**
   - 4 (one per core). The other 4 are ready but waiting for the scheduler.

3. **Why does adding a `Thread.sleep(1)` sometimes "fix" race conditions?**
   - It increases the chances of different interleaving, but doesn't solve the problem—just masks it.

4. **Why can two threads reading the same variable see different values?**
   - CPU caches. One thread's L1 cache has a stale value while another has the updated value in memory.

5. **If I read a variable twice in quick succession, am I guaranteed to see the same value?**
   - Not without `volatile`. The compiler might optimize to cache the value, but another thread might have written to it in between.

---

## Self-Check Questions

- Can you explain what happens in the `ready` boolean example above?
- Why does compiler reordering break the `value/ready` example?
- What is the difference between "in RUNNABLE state" and "currently running"?
