# 6. Explicit Locks and Conditions

## ReentrantLock: More Powerful Than synchronized

`synchronized` is simple but limited. `ReentrantLock` provides advanced features.

### Comparison Table

| Feature | synchronized | ReentrantLock |
|---------|--------------|---------------|
| **Basic Locking** | Yes | Yes |
| **Reentrant** | Yes | Yes |
| **Try Lock (timeout)** | No | Yes |
| **Interruptible** | No | Yes |
| **Condition Variables** | Wait/Notify | Conditions |
| **Fair Locking** | No | Yes (optional) |
| **Lock Acquisition** | Implicit | Explicit |

---

## Why Use ReentrantLock?

### 1. tryLock() - Non-Blocking Lock Acquisition

```java
ReentrantLock lock = new ReentrantLock();

// Option 1: Block until lock is available
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}

// Option 2: Try to acquire, don't block
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Could not acquire lock");
}
```

### 2. tryLock(long, TimeUnit) - Timed Lock Acquisition

```java
ReentrantLock lock = new ReentrantLock();

try {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("Timeout: Could not acquire lock within 1 second");
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### 3. lockInterruptibly() - Interruptible Lock Acquisition

```java
ReentrantLock lock = new ReentrantLock();

try {
    lock.lockInterruptibly();  // Can be interrupted
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    System.out.println("Lock acquisition was interrupted");
    Thread.currentThread().interrupt();
}
```

### 4. Condition Variables (Multiple Conditions)

```java
ReentrantLock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

// Thread 1: Producer
lock.lock();
try {
    while (isFull()) {
        notFull.await();  // Wait until space is available
    }
    addItem();
    notEmpty.signal();  // Notify consumers
} finally {
    lock.unlock();
}

// Thread 2: Consumer
lock.lock();
try {
    while (isEmpty()) {
        notEmpty.await();  // Wait until items available
    }
    removeItem();
    notFull.signal();  // Notify producers
} finally {
    lock.unlock();
}
```

---

## Fairness

By default, locks are **unfair**: no guarantee which waiting thread gets the lock.

```java
ReentrantLock unfair = new ReentrantLock();  // Default
ReentrantLock fair = new ReentrantLock(true);  // Fair
```

### Unfair Lock

```
Thread A waits
Thread B waits
Thread C (just arrived) might acquire lock before A or B
```

**Advantage**: Better throughput (less context switching)

### Fair Lock

```
Thread A waits
Thread B waits
Thread C (just arrived) must wait for A and B
```

**Advantage**: No starvation (FIFO order)
**Disadvantage**: Worse throughput

---

## Reentrant: A Thread Can Acquire Its Own Lock

Both `synchronized` and `ReentrantLock` are **reentrant**: a thread can acquire the same lock multiple times.

```java
ReentrantLock lock = new ReentrantLock();

public void outer() {
    lock.lock();
    try {
        System.out.println("In outer");
        inner();
    } finally {
        lock.unlock();
    }
}

public void inner() {
    lock.lock();  // Can acquire the same lock again
    try {
        System.out.println("In inner");
    } finally {
        lock.unlock();
    }
}

// main
outer();  // Thread can acquire lock twice
```

**How it works**: The lock tracks the owner and acquisition count.

```
Lock state:
  owner: Thread A
  count: 0 → 1 (lock acquired)
  
Inner:
  count: 1 → 2 (reentrant acquire)
  
Finally (unlock):
  count: 2 → 1
  
Finally (unlock):
  count: 1 → 0 (lock released)
```

---

## Condition Variables: wait() vs await()

### Object.wait() (with synchronized)

```java
synchronized(lock) {
    while (condition is false) {
        lock.wait();  // Release lock and wait
    }
    // Critical section
}
```

### Condition.await() (with ReentrantLock)

```java
lock.lock();
try {
    while (condition is false) {
        condition.await();  // Release lock and wait
    }
    // Critical section
} finally {
    lock.unlock();
}
```

Both work similarly, but Condition provides more control:

```java
Condition cond = lock.newCondition();

// await with timeout
if (cond.await(1, TimeUnit.SECONDS)) {
    System.out.println("Signaled");
} else {
    System.out.println("Timeout");
}

// Signal one waiting thread
cond.signal();

// Signal all waiting threads
cond.signalAll();
```

---

## Code Examples

### Example 1: Bounded Queue with ReentrantLock

```java
public class BoundedQueue<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait for space
            }
            queue.add(item);
            notEmpty.signal();  // Wake up consumers
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait for items
            }
            T item = queue.remove();
            notFull.signal();  // Wake up producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### Example 2: tryLock Pattern

```java
public class TryLockExample {
    private ReentrantLock lock = new ReentrantLock();
    private int balance = 1000;

    public boolean transfer(int amount) {
        if (lock.tryLock()) {
            try {
                if (balance >= amount) {
                    balance -= amount;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Could not acquire lock, skipping transfer");
            return false;
        }
    }

    public boolean transferWithTimeout(int amount) throws InterruptedException {
        if (lock.tryLock(2, TimeUnit.SECONDS)) {
            try {
                if (balance >= amount) {
                    balance -= amount;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Lock acquisition timed out");
            return false;
        }
    }
}
```

### Example 3: Interruptible Lock

```java
public class InterruptibleLockExample {
    private ReentrantLock lock = new ReentrantLock();

    public void doWork() {
        try {
            lock.lockInterruptibly();
            try {
                System.out.println("Working...");
                Thread.sleep(5000);
            } finally {
                lock.unlock();
            }
        } catch (InterruptedException e) {
            System.out.println("Work interrupted");
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        InterruptibleLockExample example = new InterruptibleLockExample();

        Thread worker = new Thread(example::doWork);
        worker.start();

        Thread.sleep(1000);
        worker.interrupt();  // Interrupt the worker

        worker.join();
    }
}
```

### Example 4: Fair vs Unfair Lock

```java
public class FairnessExample {
    private ReentrantLock unfairLock = new ReentrantLock(false);
    private ReentrantLock fairLock = new ReentrantLock(true);

    public void demonstrateFairness() throws InterruptedException {
        ReentrantLock lock = fairLock;  // Change to unfairLock to see difference

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                lock.lock();
                try {
                    System.out.println("Thread 1: " + i);
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                lock.lock();
                try {
                    System.out.println("Thread 2: " + i);
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

---

## Interview Questions

1. **When would you use ReentrantLock instead of synchronized?**
   - When you need tryLock, interruptible lock acquisition, conditions, or fairness policies.

2. **What's the difference between fair and unfair locks?**
   - Fair locks guarantee FIFO order but have worse throughput. Unfair locks have better throughput but threads can starve.

3. **Why is the try-finally pattern important with ReentrantLock?**
   - If an exception occurs in the critical section, the lock must still be released. try-finally guarantees this.

4. **Can you call await() without holding the lock?**
   - No. You must hold the lock before calling await(). Otherwise, IllegalMonitorStateException is thrown.

5. **What's the difference between signal() and signalAll()?**
   - signal() wakes one waiting thread; signalAll() wakes all. Use signalAll() to be safe unless you know only one thread should proceed.

---

## Self-Check

- Explain the Producer-Consumer pattern with Conditions
- Why is the while loop necessary in await() patterns?
- What happens if you forget to unlock a ReentrantLock?
- When would fairness be important in a real system?
