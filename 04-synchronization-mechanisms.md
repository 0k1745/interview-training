# 4. Synchronization Mechanisms: volatile and synchronized

## volatile: The Basics

`volatile` is often misunderstood. It does **NOT** provide mutual exclusion.

### What `volatile` Does

1. **Prevents Caching**: Every read and write goes to main memory (not thread's L1/L2 cache)
2. **Prevents Certain Reorderings**: Prevents instructions from being reordered across the volatile barrier
3. **Establishes Happens-Before**: Write to volatile → read from volatile is a happens-before relationship

### What `volatile` Does NOT Do

1. **Does NOT provide atomicity**: `counter++` is still three operations
2. **Does NOT provide mutual exclusion**: Two threads can read/write simultaneously
3. **Does NOT make compound operations safe**

---

## volatile: Memory Visibility

### Without volatile

```java
private boolean ready = false;
private int value = 0;

// Thread A
public void writer() {
    value = 10;
    ready = true;  // Write to cache, may not flush to main memory
}

// Thread B
public void reader() {
    while (!ready) { }  // May see cached value (false) forever
    System.out.println(value);  // May see 0
}
```

**Problem**: Thread B's cache may never see the updates.

### With volatile

```java
private volatile boolean ready = false;
private volatile int value = 0;

// Thread A
public void writer() {
    value = 10;
    ready = true;  // Flush to main memory immediately
}

// Thread B
public void reader() {
    while (!ready) { }  // Reads from main memory
    System.out.println(value);  // Guaranteed to see 10
}
```

**Why it works**: `volatile` forces reads/writes to main memory, bypassing caches.

---

## volatile: NOT Sufficient for Atomicity

### The Counter Problem

```java
private volatile int counter = 0;

public void increment() {
    counter++;  // STILL NOT ATOMIC
}
```

**Why**: `counter++` compiles to:

```
1. getfield counter     (read from memory)
2. iconst_1             (load 1)
3. iadd                 (add)
4. putfield counter     (write to memory)
```

**Timeline with two threads**:

```
Thread A: read (0)
Thread B: read (0)
Thread A: increment (1)
Thread B: increment (1)
Thread A: write (1)
Thread B: write (1)
```

**Expected: 2, Actual: 1**

`volatile` doesn't help because the race condition is in the algorithm, not visibility.

### When volatile IS Sufficient

```java
private volatile boolean shutdown = false;

public void shutdown() {
    shutdown = true;  // Single write, no compound operation
}

public void run() {
    while (!shutdown) {  // Single read
        doWork();
    }
}
```

**Why it works**: Only single reads/writes, no compound operations. `volatile` guarantees visibility.

---

## synchronized: Three Guarantees

`synchronized` is much more powerful than `volatile`.

### 1. Mutual Exclusion

Only one thread can hold the monitor at a time.

```java
public synchronized void transfer(Account from, Account to, int amount) {
    from.withdraw(amount);
    to.deposit(amount);
}
```

**Guarantee**: Only one thread executes this block at a time.

### 2. Visibility

All writes before releasing the lock are flushed to main memory.
All reads after acquiring the lock reload from main memory.

```java
synchronized(lock) {
    x = 5;
    y = 10;
}  // Writes flushed to main memory

// ... later ...

synchronized(lock) {
    System.out.println(x);  // Guaranteed to see 5
    System.out.println(y);  // Guaranteed to see 10
}
```

### 3. Ordering

Instructions before the lock release cannot be reordered after it.
Instructions after the lock acquire cannot be reordered before it.

```java
synchronized(lock) {
    a = 1;
    b = 2;
}

synchronized(lock) {
    assert b == 2;  // Guaranteed (no reordering)
    assert a == 1;  // Guaranteed (no reordering)
}
```

---

## How synchronized Works: Monitors

Every Java object has a **monitor** (intrinsic lock).

```
Object
  ↓
Monitor (owns lock, tracks waiting threads)
  ↓
Owner thread: The thread holding the lock
  ↓
Waiting queue: Threads blocked on this lock
```

### Entering a synchronized Block

```java
synchronized(account) {  // Acquire monitor lock
    // Only one thread here at a time
}  // Release monitor lock
```

**Behind the scenes**:

1. Thread attempts to acquire the monitor
2. If another thread holds it, this thread blocks
3. Once acquired, thread is the monitor owner
4. On exit, lock is released; blocked threads are notified

---

## synchronized Method vs synchronized Block

### Method Synchronization

```java
public synchronized void withdraw(int amount) {
    balance -= amount;
}
```

This locks on `this` (the object instance).

### Static Method Synchronization

```java
public static synchronized void log(String msg) {
    System.out.println(msg);
}
```

This locks on `MyClass.class` (the Class object).

### Block Synchronization

```java
public void withdraw(int amount) {
    synchronized(this) {
        balance -= amount;
    }
}
```

Same as method synchronization, but locks a specific object.

### Different Locks: A Trap

```java
public class Trap {
    private int x = 0;

    public synchronized void methodA() {
        x = 5;
    }

    public static synchronized void methodB() {
        // Locks on Trap.class, not this
    }

    public void methodC() {
        synchronized(new Object()) {
            // Locks on a different object entirely
        }
    }
}
```

**Interview trap**: These three methods lock on **different monitors**.

```java
Thread A calls methodA() → locks on instance
Thread B calls methodB() → locks on class
Thread C calls methodC() → locks on new Object()
```

All three can run **simultaneously** because they're locking different objects!

---

## Code Examples

### Example 1: Thread-Safe Counter

```java
public class SafeCounter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        SafeCounter counter = new SafeCounter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(counter.get());  // Guaranteed: 20000
    }
}
```

### Example 2: Thread-Safe Account Transfer

```java
public class Account {
    private int balance;

    public Account(int initialBalance) {
        this.balance = initialBalance;
    }

    public synchronized void transfer(Account recipient, int amount) {
        if (this.balance >= amount) {
            this.balance -= amount;
            recipient.balance += amount;
        }
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

**Note**: This is a simplified example. In reality, transferring between two accounts requires careful locking to avoid deadlock (lock ordering).

### Example 3: Volatile vs Synchronized

```java
public class ComparisonExample {
    private volatile int volatileCounter = 0;
    private int syncCounter = 0;

    // This is WRONG - volatile doesn't make counter++ safe
    public void incrementVolatile() {
        volatileCounter++;  // Race condition
    }

    // This is CORRECT - synchronized makes counter++ safe
    public synchronized void incrementSync() {
        syncCounter++;
    }

    public static void main(String[] args) throws InterruptedException {
        ComparisonExample obj = new ComparisonExample();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                obj.incrementVolatile();
                obj.incrementSync();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                obj.incrementVolatile();
                obj.incrementSync();
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Volatile counter: " + obj.volatileCounter);  // ~15000 (lost updates)
        System.out.println("Sync counter: " + obj.syncCounter);          // 20000 (correct)
    }
}
```

---

## Performance Considerations

### volatile
- **Cost**: One memory barrier per access (cheaper)
- **Use case**: Simple flags, status variables
- **Example**: `volatile boolean shutdown`

### synchronized
- **Cost**: Lock acquisition/release (more expensive)
- **Use case**: Protecting shared mutable state, compound operations
- **Example**: `synchronized void increment()`

### When to Use What

| Scenario | Choice |
|----------|--------|
| Single boolean flag | `volatile` |
| Single counter | `synchronized` or `AtomicInteger` |
| Complex state (multiple fields) | `synchronized` |
| Multiple independent counters | `AtomicInteger` (array) |
| Read-heavy workload | `volatile` (no lock contention) |
| Write-heavy workload | `synchronized` (better contention handling) |

---

## Interview Questions

1. **Why doesn't `volatile` make `counter++` safe?**
   - Because `counter++` is three operations (read, increment, write), and `volatile` only guarantees visibility, not atomicity.

2. **What's the difference between `synchronized(this)` and `synchronized(lock)`?**
   - They lock different monitors. `synchronized(this)` locks the instance; `synchronized(lock)` locks the lock object.

3. **Can two threads execute two different `synchronized` blocks simultaneously?**
   - Yes, if they lock different objects:
     ```java
     synchronized(lockA) { ... }  // Thread 1
     synchronized(lockB) { ... }  // Thread 2 (can run concurrently)
     ```

4. **Why does `volatile` prevent certain instruction reorderings?**
   - The JMM defines that volatile writes and reads act as memory barriers, preventing reordering across them.

5. **What happens if a thread holding a lock throws an exception?**
   - The lock is automatically released. `synchronized` is exception-safe.

---

## Self-Check

- Explain when `volatile` is sufficient and when it's not
- Why can a thread starve waiting for a `synchronized` lock?
- What's the difference between method-level and block-level synchronization?
- Why is the counter example in the main section a trap?
