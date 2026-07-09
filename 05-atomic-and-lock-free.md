# 5. Atomic Classes and Lock-Free Programming

## CAS: Compare-And-Swap

The foundation of lock-free algorithms.

### How CAS Works

```
Memory: counter = 5

Thread wants to increment to 6

CAS Instruction (atomic):
  if (counter == 5)
    counter = 6
    return true
  else
    return false
    (don't modify)
```

**Key**: This entire operation is atomic (cannot be interrupted).

### CAS in Java

```java
AtomicInteger counter = new AtomicInteger(5);

// Atomic compare-and-swap
boolean success = counter.compareAndSet(5, 6);
// if counter == 5, set to 6 and return true
// if counter != 5, don't modify and return false
```

---

## Atomic Classes

Java provides atomic wrappers for common types:

```java
AtomicInteger
AtomicLong
AtomicReference<T>
AtomicBoolean
AtomicIntegerArray
AtomicReferenceArray<T>
```

### AtomicInteger Example

```java
AtomicInteger counter = new AtomicInteger(0);

// Atomic increment
counter.incrementAndGet();  // Returns new value

// Atomic decrement
counter.decrementAndGet();  // Returns new value

// Atomic add
counter.addAndGet(5);  // Add 5, returns new value

// Atomic get
int value = counter.get();

// Atomic compare-and-swap
counter.compareAndSet(10, 20);

// Atomic get-and-set
int old = counter.getAndSet(100);  // Set to 100, return old value
```

### AtomicReference Example

```java
class User {
    String name;
    int age;
}

AtomicReference<User> userRef = new AtomicReference<>(new User("Alice", 30));

// Atomic swap
User oldUser = userRef.getAndSet(new User("Bob", 25));

// Atomic compare-and-swap
User expected = new User("Bob", 25);
User update = new User("Charlie", 35);
userRef.compareAndSet(expected, update);
```

---

## Why Atomic Classes Instead of synchronized?

### Option 1: Synchronized

```java
private int counter = 0;

public synchronized void increment() {
    counter++;
}

public synchronized int get() {
    return counter;
}
```

**Cost**: Lock acquisition/release on every operation

### Option 2: AtomicInteger

```java
private AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();
}

public int get() {
    return counter.get();
}
```

**Cost**: CAS operation (cheaper than acquiring a lock, especially under low contention)

### When Atomic Wins

- **Low contention** (few threads competing): Atomic is faster
- **Simple operations** (just increment/swap): Atomic is simpler
- **High throughput** (many operations): Atomic avoids lock overhead

---

## Lock-Free vs Blocking: The Tradeoff

### Lock-Free Advantage: No Blocking

```java
// Thread A
counter.incrementAndGet();  // Never blocks, even if threads compete

// Thread B
counter.incrementAndGet();  // Also never blocks
```

### Lock-Free Disadvantage: Retry Loop

Under heavy contention, CAS fails frequently:

```java
// Pseudocode for CAS
while (!compareAndSet(expected, update)) {
    // CAS failed, retry
    expected = get();
    update = expected + 1;
}
```

**Timeline (heavy contention)**:

```
Thread A: CAS (fails)
Thread B: CAS (succeeds)
Thread A: CAS (fails)
Thread B: CAS (succeeds)
Thread A: CAS (fails)  (retries many times)
...
```

**CPU usage**: Explodes due to busy-waiting.

---

## When Lock-Free Isn't Better

### Scenario: Heavy Contention (Many threads, high update rate)

```
CAS with retries:
  Thread A: CAS fail, fail, fail, fail, retry, retry...
  Thread B: CAS fail, fail, fail, fail, retry, retry...
  Thread C: CAS fail, fail, fail, fail, retry, retry...
```

**CPU**: Spinning on CAS, wasting cycles.

### vs Synchronized with Blocking:

```
Thread A: Acquire lock (blocks)
Thread B: Waits in queue
Thread C: Waits in queue

Thread A: Complete operation, release lock
Thread B: Acquire lock, run
...
```

**CPU**: Threads sleep, not spinning.

---

## The Correct Answer for Interviews

**Question**: "Should we use lock-free algorithms instead of locks?"

**Wrong answer**: "Yes, lock-free is always faster."

**Right answer**: 
> It depends on contention. Under low contention, atomic classes (CAS) are faster because no thread blocks. Under high contention, locks perform better because threads sleep instead of spinning. The tradeoff is context-switching cost vs CPU wasted in retry loops.

---

## Code Examples

### Example 1: Thread-Safe Counter with Atomic

```java
public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int get() {
        return count.get();
    }

    public static void main(String[] args) throws InterruptedException {
        AtomicCounter counter = new AtomicCounter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 50000; i++) counter.increment();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 50000; i++) counter.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(counter.get());  // Always 100000
    }
}
```

### Example 2: Compare-And-Swap Pattern

```java
public class CASExample {
    private AtomicReference<String> state = new AtomicReference<>("INITIAL");

    public boolean transitionTo(String nextState) {
        return state.compareAndSet("INITIAL", nextState);
    }

    public static void main(String[] args) {
        CASExample obj = new CASExample();

        // First thread succeeds
        boolean success1 = obj.transitionTo("READY");
        System.out.println("Thread 1: " + success1);  // true

        // Second thread fails (state is no longer "INITIAL")
        boolean success2 = obj.transitionTo("READY");
        System.out.println("Thread 2: " + success2);  // false
    }
}
```

### Example 3: Retry Loop (Manual CAS)

```java
public class ManualCAS {
    private AtomicInteger value = new AtomicInteger(0);

    public void incrementByRetrying() {
        int expected;
        int update;
        do {
            expected = value.get();
            update = expected + 1;
        } while (!value.compareAndSet(expected, update));
    }

    public static void main(String[] args) throws InterruptedException {
        ManualCAS obj = new ManualCAS();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) obj.incrementByRetrying();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) obj.incrementByRetrying();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(obj.value.get());  // 20000
    }
}
```

### Example 4: Atomic vs Synchronized Performance

```java
public class AtomicVsSynchronized {
    private AtomicInteger atomicCounter = new AtomicInteger(0);
    private int syncCounter = 0;

    public void incrementAtomic() {
        atomicCounter.incrementAndGet();
    }

    public synchronized void incrementSync() {
        syncCounter++;
    }

    public static void main(String[] args) throws InterruptedException {
        AtomicVsSynchronized obj = new AtomicVsSynchronized();

        long atomicStart = System.nanoTime();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) obj.incrementAtomic();
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) obj.incrementAtomic();
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long atomicEnd = System.nanoTime();

        long syncStart = System.nanoTime();
        Thread t3 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) obj.incrementSync();
        });
        Thread t4 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) obj.incrementSync();
        });
        t3.start();
        t4.start();
        t3.join();
        t4.join();
        long syncEnd = System.nanoTime();

        System.out.println("Atomic time: " + (atomicEnd - atomicStart) + " ns");
        System.out.println("Synchronized time: " + (syncEnd - syncStart) + " ns");
    }
}
```

---

## AtomicInteger Methods

| Method | Purpose |
|--------|---------|
| `get()` | Get current value |
| `set(int)` | Set value |
| `getAndSet(int)` | Set and return old value |
| `incrementAndGet()` | Increment and return new value |
| `getAndIncrement()` | Increment and return old value |
| `decrementAndGet()` | Decrement and return new value |
| `getAndDecrement()` | Decrement and return old value |
| `addAndGet(int)` | Add and return new value |
| `getAndAdd(int)` | Add and return old value |
| `compareAndSet(int, int)` | CAS operation |

---

## Interview Questions

1. **What is CAS and how does it differ from lock-based synchronization?**
   - CAS (Compare-And-Swap) is an atomic CPU instruction. Lock-free algorithms use CAS instead of acquiring locks. Under low contention, CAS is faster; under high contention, locks are better.

2. **Why would heavy contention cause CAS to perform poorly?**
   - When many threads compete, CAS fails frequently, forcing retry loops. Threads spin-wait (busy-wait), wasting CPU cycles instead of sleeping like they would with locks.

3. **When should you use AtomicInteger instead of synchronized int?**
   - Use AtomicInteger for simple counters under low-to-medium contention. Use synchronized for complex multi-field updates or high-contention scenarios.

4. **What's the difference between `getAndIncrement()` and `incrementAndGet()`?**
   - `getAndIncrement()` returns the old value, then increments. `incrementAndGet()` increments first, then returns the new value.

5. **Can AtomicReference be used to make any object thread-safe?**
   - No. AtomicReference only guarantees atomic assignment. If the object is mutable and shared, you still need synchronization for field access.

---

## Self-Check

- Explain the retry loop in manual CAS
- Why does atomic not use locks?
- When does lock-free perform worse than locks?
- What are the memory guarantees of AtomicInteger?
