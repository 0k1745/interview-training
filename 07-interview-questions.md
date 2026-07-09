# 7. Java Concurrency Interview Questions & Answers

## Fundamental Questions

### Q1: What is the difference between a process and a thread?

**Answer:**

A **process** is an independent execution environment with its own memory space, file descriptors, and security context. Only the OS can create processes.

A **thread** is an execution path within a process. Threads share the process's heap but have isolated stacks. Creating threads is cheaper than creating processes.

**Key difference**: Process isolation is enforced by the OS; thread isolation is cooperative (shared heap = race conditions possible).

**Example:**
```
Process A (isolated memory)
├── Thread 1
├── Thread 2
└── Thread 3

Process B (isolated memory, cannot access A's memory)
├── Thread 1
├── Thread 2
```

If Thread 1 in Process A crashes, Process B is unaffected.
If Thread 1 crashes, all threads in Process A crash.

---

### Q2: What does "thread-safe" mean?

**Answer:**

An object or method is **thread-safe** if it behaves correctly when accessed by multiple threads simultaneously, without additional synchronization.

**Thread-safe examples:**
- `java.util.concurrent.ConcurrentHashMap` (internal synchronization)
- Immutable objects (no state to corrupt)
- Objects accessed via `synchronized` (external synchronization)

**NOT thread-safe:**
- `HashMap` (not synchronized internally)
- Plain mutable objects without locks

```java
// NOT thread-safe
public class Counter {
    private int count = 0;
    public void increment() { count++; }  // Race condition
}

// Thread-safe
public class SafeCounter {
    private int count = 0;
    public synchronized void increment() { count++; }
}
```

---

### Q3: What is a race condition?

**Answer:**

A **race condition** occurs when multiple threads access shared mutable state, and the result depends on the timing of their execution.

**Example:**
```java
private int counter = 0;

// Thread A
counter++;

// Thread B
counter++;
```

Expected result: 2
Actual result: 1 (one write is lost)

Why: `counter++` is not atomic.
```
Thread A: read (0) → increment (1) → write (1)
Thread B: read (0) → increment (1) → write (1)
```

Both threads see the same initial value, so both compute the same increment.

**Solution**: Use `synchronized`, `volatile`, or `AtomicInteger`.

---

### Q4: Explain the Java Memory Model (JMM).

**Answer:**

The JMM specifies:
1. **Visibility**: When writes by one thread become visible to others
2. **Ordering**: What reorderings are allowed
3. **Atomicity**: Which operations complete without interruption

**Without synchronization**: No guarantees. A write might never be visible to other threads.

**With synchronization** (locks, volatile): Visibility and ordering are guaranteed via **happens-before relationships**.

**Example:**
```java
// NO guarantee of visibility
boolean done = false;
int value = 0;

Thread A:
value = 10;
done = true;  // May stay in cache

Thread B:
while (!done) { }  // May never see true
System.out.println(value);  // May print 0
```

**With volatile:**
```java
volatile boolean done = false;
volatile int value = 0;

// NOW guaranteed visibility
Thread B is guaranteed to see value = 10
```

---

### Q5: What is the difference between volatile and synchronized?

**Answer:**

| Aspect | volatile | synchronized |
|--------|----------|--------------|
| **Atomicity** | No | Yes |
| **Visibility** | Yes | Yes |
| **Ordering** | Yes (limited) | Yes |
| **Performance** | Faster | Slower (lock overhead) |
| **Use Case** | Simple flags | Complex state |

**volatile:**
- Prevents caching and reordering
- Does NOT provide atomicity
- Good for: `volatile boolean ready`

**synchronized:**
- Provides mutual exclusion (only one thread)
- Provides visibility and ordering
- Good for: `synchronized void increment()`

**Example:**
```java
// WRONG: volatile counter++
private volatile int counter = 0;
public void increment() {
    counter++;  // Still loses updates
}

// CORRECT: synchronized counter++
private int counter = 0;
public synchronized void increment() {
    counter++;
}
```

---

## Advanced Questions

### Q6: What does "happens-before" mean?

**Answer:**

If action A **happens-before** action B:
- All writes by A are visible to B
- A's effects are ordered before B's effects

**Examples:**

1. **Lock release → Lock acquire:**
```java
synchronized(lock) {
    x = 5;  // write
}
synchronized(lock) {
    System.out.println(x);  // guaranteed to see 5
}
```

2. **Thread.start() → Started thread:**
```java
x = 5;
thread.start();  // All prior writes are visible in thread
```

3. **Thread.join() → Caller resumes:**
```java
thread.join();  // All writes in thread are visible
System.out.println(x);
```

4. **volatile write → volatile read:**
```java
volatile int x = 0;
x = 5;
int y = x;  // guaranteed to see 5
```

Without happens-before: No guarantees.

---

### Q7: Why is counter++ not atomic?

**Answer:**

`counter++` is three operations:
```
1. Read counter from memory
2. Increment (add 1)
3. Write counter back
```

Any two threads can interleave:
```
Thread A: read (0)
Thread B: read (0)
Thread A: increment (1)
Thread B: increment (1)
Thread A: write (1)
Thread B: write (1)
```

Result: counter = 1 (one write lost)

**Solutions:**
- `synchronized void increment() { counter++; }`
- `private AtomicInteger counter = new AtomicInteger(0); counter.incrementAndGet();`

---

### Q8: What is the difference between wait() and sleep()?

**Answer:**

| Aspect | wait() | sleep() |
|--------|--------|---------|
| **Releases Lock** | Yes | No |
| **Wakes on notify()** | Yes | No |
| **Class** | Object | Thread |
| **Used for** | Coordination | Delays |

**wait():**
```java
synchronized(lock) {
    while (!condition) {
        lock.wait();  // Release lock and wait
    }
    // Lock reacquired, continue
}
```

**sleep():**
```java
Thread.sleep(1000);  // Sleep 1 second, don't release lock
```

**Interview trap:**
```java
synchronized(lock) {
    Thread.sleep(1000);  // Lock is HELD (bad!)
}

// vs

synchronized(lock) {
    lock.wait();  // Lock is RELEASED (good)
}
```

---

### Q9: Explain the Producer-Consumer pattern.

**Answer:**

```java
public class BoundedQueue<T> {
    private Queue<T> queue = new LinkedList<>();
    private int capacity;
    private Object lock = new Object();

    public void put(T item) throws InterruptedException {
        synchronized(lock) {
            while (queue.size() == capacity) {
                lock.wait();  // Producer waits if full
            }
            queue.add(item);
            lock.notifyAll();  // Wake consumers
        }
    }

    public T take() throws InterruptedException {
        synchronized(lock) {
            while (queue.isEmpty()) {
                lock.wait();  // Consumer waits if empty
            }
            T item = queue.remove();
            lock.notifyAll();  // Wake producers
            return item;
        }
    }
}
```

**Key points:**
- Producer waits if queue is full
- Consumer waits if queue is empty
- Both signal to wake the other side
- Use `notifyAll()` to wake all waiting threads

---

### Q10: What is a deadlock? How can it occur?

**Answer:**

**Deadlock** occurs when two or more threads are blocked forever, each waiting for a resource the other holds.

**Classic example:**
```java
Thread A:
  lock.a.lock();
  Thread.sleep(100);  // Give B time to acquire lock.b
  lock.b.lock();  // Blocked: B holds lock.b

Thread B:
  lock.b.lock();
  Thread.sleep(100);  // Give A time to acquire lock.a
  lock.a.lock();  // Blocked: A holds lock.a
```

**Conditions for deadlock:**
1. **Mutual exclusion**: Only one thread can hold a lock
2. **Hold and wait**: Thread holds a lock while waiting for another
3. **No preemption**: Locks can't be forcibly taken
4. **Circular wait**: T1 waits for T2, T2 waits for T1

**How to prevent:**
- **Lock ordering**: Always acquire locks in the same order
```java
// SAFE: Always lock.a before lock.b
Thread A and B:
  lock.a.lock();
  lock.b.lock();
```

- **Timeout**:
```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    // Got lock
}
```

- **Avoid nested locks**: Don't acquire locks while holding others

---

### Q11: What is the difference between CountDownLatch and CyclicBarrier?

**Answer:**

| Aspect | CountDownLatch | CyclicBarrier |
|--------|----------------|---------------|
| **Purpose** | Wait for N events | Wait for N threads |
| **Reusable** | No (one-time use) | Yes (can reuse) |
| **Action** | Decrement counter | Await at barrier |
| **Threads waiting** | Any number | Fixed number |

**CountDownLatch:**
```java
CountDownLatch latch = new CountDownLatch(3);

// Threads 1, 2, 3
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown();  // Signal completion
    }).start();
}

latch.await();  // Main thread waits for all
System.out.println("All done");
```

**CyclicBarrier:**
```java
CyclicBarrier barrier = new CyclicBarrier(3);

// Threads 1, 2, 3
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        barrier.await();  // Wait for all threads
        // All proceed together
    }).start();
}
```

---

### Q12: What is ThreadLocal and when should you use it?

**Answer:**

**ThreadLocal** provides per-thread storage. Each thread gets its own independent copy of the variable.

```java
ThreadLocal<Connection> connectionHolder = ThreadLocal.withInitial(() -> 
    DriverManager.getConnection("jdbc:..."));

// Thread 1
Connection conn1 = connectionHolder.get();  // Thread 1's connection

// Thread 2
Connection conn2 = connectionHolder.get();  // Thread 2's connection (different!)
```

**Use cases:**
- Database connections (each thread has one)
- HTTP request context (servlet request)
- Transactions (thread-local transaction state)

**Warning: Memory leaks**
```java
// BAD: ThreadLocal not cleaned up
threadLocal.set(largeObject);
// largeObject stays in memory even after thread reuse

// GOOD: Clean up in finally
try {
    threadLocal.set(value);
    doWork();
} finally {
    threadLocal.remove();  // Clean up
}
```

---

## Tricky Questions

### Q13: Can two threads call two different synchronized methods on the same object simultaneously?

**Answer:** **No.**

Both methods lock on `this`. Only one can hold the lock at a time.

```java
public synchronized void methodA() { }
public synchronized void methodB() { }

// Thread 1
obj.methodA();  // Acquires lock on obj

// Thread 2
obj.methodB();  // Blocked: obj's lock is held by Thread 1
```

**But:** Different objects can be locked simultaneously:

```java
obj1.methodA();  // Locks obj1
obj2.methodA();  // Locks obj2 (different objects, different locks)
```

---

### Q14: What's the difference between notify() and notifyAll()?

**Answer:**

- **notify()**: Wakes ONE arbitrary waiting thread
- **notifyAll()**: Wakes ALL waiting threads

**When to use notifyAll():**
- When multiple threads might be waiting for different conditions
- When waking the wrong thread causes a deadlock

```java
// BAD: Using notify()
synchronized(lock) {
    if (hasWork) {
        notify();  // Wakes ONE thread (might be wrong one)
    }
}

// GOOD: Using notifyAll()
synchronized(lock) {
    if (hasWork) {
        notifyAll();  // Wakes all threads (they check condition)
    }
}
```

---

### Q15: Why must wait() be called in a loop?

**Answer:**

Spurious wakeups: A thread can wake without being notified.

```java
// WRONG
synchronized(lock) {
    if (!condition) {
        lock.wait();
    }
    // Use condition
}

// CORRECT
synchronized(lock) {
    while (!condition) {  // Loop! Condition might be false after waking
        lock.wait();
    }
    // Use condition
}
```

**Example:**
```java
while (queue.isEmpty()) {  // Loop
    lock.wait();  // Might wake spuriously
}
// Now queue is guaranteed non-empty
item = queue.remove();
```

---

## Self-Check Scenarios

### Scenario 1: Fix the Race Condition

```java
public class Broken {
    private int counter = 0;
    
    public void increment() {
        counter++;  // BROKEN
    }
    
    public int get() {
        return counter;
    }
}
```

**Fix:**
```java
public class Fixed {
    private int counter = 0;
    
    public synchronized void increment() {
        counter++;
    }
    
    public synchronized int get() {
        return counter;
    }
}
```

### Scenario 2: Identify the Deadlock

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread A:
synchronized(lock1) {
    Thread.sleep(100);
    synchronized(lock2) { }  // DEADLOCK
}

Thread B:
synchronized(lock2) {
    Thread.sleep(100);
    synchronized(lock1) { }  // DEADLOCK
}
```

**Fix: Lock ordering**
```java
Thread A and B:
synchronized(lock1) {
    synchronized(lock2) { }
}
```

### Scenario 3: Producer-Consumer Bug

```java
synchronized(lock) {
    if (queue.isEmpty()) {  // BUG: should be while
        lock.wait();
    }
    item = queue.remove();
}
```

**Fix:**
```java
synchronized(lock) {
    while (queue.isEmpty()) {  // Use while
        lock.wait();
    }
    item = queue.remove();
}
```
