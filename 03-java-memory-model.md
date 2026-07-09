# 3. The Java Memory Model (JMM)

## What is the Java Memory Model?

The JMM is a **contract** between the Java language and the JVM.

It specifies:
- **Visibility**: When writes by one thread become visible to other threads
- **Ordering**: Whether instructions can be reordered
- **Atomicity**: Which operations are guaranteed to complete without interruption

**What it does NOT define**: CPU behavior. The JMM is Java's guarantee, not the CPU's.

---

## The Three Guarantees

### 1. Visibility

**Question**: When Thread A writes a value, when does Thread B see it?

**Without synchronization**: Java gives NO guarantee.

```java
Thread A:
x = 5;

Thread B:
System.out.println(x);  // May print 0, or 5, or undefined
```

The write might stay in Thread A's cache forever.

**With synchronization** (e.g., `synchronized`, `volatile`):

Thread B is guaranteed to see the write.

---

### 2. Ordering

**Question**: Can the JVM reorder my instructions?

**Yes.** But with rules.

```java
a = 1;
b = 2;
```

The JVM might execute as:

```java
b = 2;
a = 1;
```

**In single-threaded code**, this is invisible (same final result).

**In multithreaded code**, this breaks assumptions:

```java
// Thread A
x = 5;
ready = true;

// Thread B
while (!ready) { }
System.out.println(x);  // Expects 5
```

If Thread A reorders to `ready = true; x = 5;`, Thread B might see `ready = true` and `x = 0`.

---

### 3. Atomicity

**Question**: Which operations complete without interruption?

**Answer**: Only a few:
- Reading/writing a reference
- Reading/writing a primitive (except `long` and `double`, which are 64-bit)

**NOT atomic**:
- `counter++` (three operations)
- Multiple reads/writes
- Compound operations

---

## Happens-Before Relationships

This is the most important concept in the JMM.

**Definition**:

If action A **happens-before** action B, then:
- All writes by A become visible to B
- The relative order of A and B is respected

### Examples of Happens-Before

#### 1. Lock Release → Lock Acquire

```java
synchronized(lock) {
    x = 5;
}

// ... later ...

synchronized(lock) {
    System.out.println(x);  // Guaranteed to see 5
}
```

**Why**: Acquiring a lock acts as a memory barrier.

#### 2. Volatile Write → Volatile Read

```java
volatile int x = 0;

// Thread A
x = 5;

// Thread B
int y = x;  // Guaranteed to see 5
```

**Why**: `volatile` prevents caching and instruction reordering.

#### 3. Thread.start() → Started Thread

```java
Thread t = new Thread(() -> {
    System.out.println(x);  // Guaranteed to see 5
});

x = 5;
t.start();
```

**Why**: Starting a thread establishes a happens-before relationship.

#### 4. Thread.join() → Caller Resumes

```java
Thread t = new Thread(() -> {
    x = 5;
});

t.start();
t.join();  // Waits for t to finish

System.out.println(x);  // Guaranteed to see 5
```

**Why**: Joining a thread establishes a happens-before relationship.

---

## Visibility Example: Without Synchronization

```java
public class VisibilityProblem {
    private int x = 0;

    public void writer() {
        x = 5;
    }

    public void reader() {
        System.out.println(x);  // May print 0, or 5
    }
}
```

**Timeline (worst case)**:

```
Thread A (writer):
x = 5 (stored in Thread A's cache)

Thread B (reader):
System.out.println(x);  (reads from Thread B's cache, sees 0)

Result: 0 printed, even though x was set to 5
```

---

## Visibility Example: With Synchronization

```java
public class VisibilityFixed {
    private int x = 0;

    public synchronized void writer() {
        x = 5;
    }

    public synchronized void reader() {
        System.out.println(x);  // Guaranteed to see 5
    }
}
```

**Timeline (guaranteed)**:

```
Thread A (writer):
Acquire lock
x = 5
Flush caches to main memory
Release lock

Thread B (reader):
Acquire lock  (blocks until lock is available)
Reload values from main memory
System.out.println(x);  // Guaranteed to see 5
Release lock
```

---

## Ordering Example: Reordering Problem

```java
class Holder {
    private int a = 0;
    private int b = 0;

    void writer() {
        a = 1;
        b = 2;
    }

    void reader() {
        while (b == 0) { }  // Wait until b is set
        assert a == 1;      // Assume a is set
    }
}
```

**Problem**:

The compiler might reorder writer() to:

```java
void writer() {
    b = 2;  // reordered!
    a = 1;
}
```

**Timeline**:

```
Thread A (writer):
b = 2

Thread B (reader):
b == 2, exit loop
assert a == 1  (FAILS! a is still 0)

Thread A (writer):
a = 1  (too late)
```

**Solution**: Use `synchronized` or `volatile`:

```java
volatile int a = 0;
volatile int b = 0;
```

Now reordering is prevented by the JMM.

---

## Atomicity: counter++ is Broken

```java
private int counter = 0;

public void increment() {
    counter++;  // NOT atomic
}
```

Compiles to:

```
1. load counter from memory (getfield)
2. increment (add 1)
3. store counter to memory (putfield)
```

**Timeline with two threads**:

```
Thread A: load (0)
Thread B: load (0)
Thread A: increment (1)
Thread B: increment (1)
Thread A: store (1)
Thread B: store (1)
```

**Expected: 2, Actual: 1**

**Solution**: Use `synchronized` or `AtomicInteger`:

```java
private synchronized void increment() {
    counter++;
}

// or

private AtomicInteger counter = new AtomicInteger(0);
public void increment() {
    counter.incrementAndGet();
}
```

---

## Code Examples

### Example 1: Broken Visibility

```java
public class BrokenVisibility {
    private boolean done = false;
    private int result = 0;

    public void writer() {
        result = 42;
        done = true;
    }

    public void reader() {
        while (!done) { }
        System.out.println(result);  // May print 0!
    }
}
```

**Fix**:

```java
public class FixedVisibility {
    private volatile boolean done = false;
    private volatile int result = 0;

    public void writer() {
        result = 42;
        done = true;
    }

    public void reader() {
        while (!done) { }
        System.out.println(result);  // Guaranteed to print 42
    }
}
```

---

### Example 2: Lost Update (Race Condition)

```java
public class LostUpdate {
    private int counter = 0;

    public void increment() {
        counter++;  // Race condition
    }

    public int get() {
        return counter;
    }

    public static void main(String[] args) throws InterruptedException {
        LostUpdate obj = new LostUpdate();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) obj.increment();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) obj.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(obj.get());  // Expected: 20000, Actual: ~15000
    }
}
```

**Fix Option 1: Synchronized**

```java
public synchronized void increment() {
    counter++;
}
```

**Fix Option 2: AtomicInteger**

```java
private AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();
}
```

---

## The JMM Guarantees

| Scenario | Guaranteed? |
|----------|-------------|
| Write by Thread A, read by Thread B (no sync) | **No** |
| Write by Thread A (in `synchronized`), read by Thread B (in `synchronized`) | **Yes** |
| Write to `volatile` by Thread A, read by Thread B | **Yes** |
| `Thread.start()`, then actions in thread | **Yes** |
| Actions in thread, then `Thread.join()` | **Yes** |
| `counter++` completes without interruption | **No** |
| Single read/write to reference | **Yes** |

---

## Interview Questions

1. **What does "happens-before" mean in the JMM?**
   - If A happens-before B, all writes by A are visible to B, and the order is respected.

2. **Why is `volatile` NOT sufficient for `counter++`?**
   - Because `volatile` guarantees visibility and prevents reordering, but not atomicity. `counter++` is three operations that can be interleaved.

3. **Can the JVM reorder `a = 1; b = 2;` to `b = 2; a = 1;`?**
   - Yes, if there's no happens-before relationship preventing it. Adding `volatile` or `synchronized` prevents this.

4. **What does acquiring a lock do?**
   - It establishes a happens-before relationship. All writes before releasing the lock are visible to threads that acquire it later.

5. **Why does `Thread.start()` guarantee visibility?**
   - The JMM defines that `Thread.start()` happens-before all actions in the started thread.

---

## Self-Check

- Can you explain why `counter++` causes lost updates?
- What's the difference between visibility and atomicity?
- Why does `synchronized` solve both visibility and ordering problems?
- What would happen if you use `volatile` on `counter` but not `synchronized` around `counter++`?
