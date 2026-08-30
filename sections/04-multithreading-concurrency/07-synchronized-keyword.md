# Q7: How Does the Synchronized Keyword Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How does the `synchronized` keyword work in Java? What are monitor locks, intrinsic locks, and the happens-before guarantee?

---

## Answer

### What is Synchronization?

Synchronization ensures that only **one thread at a time** can execute a critical section of code, providing both **mutual exclusion** and **memory visibility**.

```
Without synchronization:           With synchronization:
Thread A: read(count=5)            Thread A: LOCK → read(5) → write(6) → UNLOCK
Thread B: read(count=5)            Thread B: WAIT...
Thread A: write(count=6)           Thread B: LOCK → read(6) → write(7) → UNLOCK
Thread B: write(count=6)
Result: 6 (WRONG! lost update)    Result: 7 (CORRECT!)
```

### Monitor Lock / Intrinsic Lock

Every Java object has an associated **monitor** (intrinsic lock):

```java
public class MonitorLockDemo {
    private int count = 0;

    // 1. Synchronized instance method — locks on 'this'
    public synchronized void increment() {
        count++; // Only one thread at a time per instance
    }

    // 2. Synchronized static method — locks on Class object
    public static synchronized void staticMethod() {
        // Locks on MonitorLockDemo.class
    }

    // 3. Synchronized block — locks on specified object
    private final Object lock = new Object();

    public void blockSync() {
        synchronized (lock) {
            count++; // More granular control
        }
    }
}
```

### Synchronized Method vs Block

```java
public class SyncComparison {
    private final List<String> list = new ArrayList<>();
    private int count = 0;

    // ❌ Synchronized method - locks entire method on 'this'
    public synchronized void addAndCount(String item) {
        // Pre-processing (doesn't need synchronization)
        String processed = item.trim().toLowerCase();

        // Both operations share the same lock, even if they're independent
        list.add(processed);
        count++;
    }

    // ✅ Synchronized block - only lock what needs protection
    private final Object listLock = new Object();
    private final Object countLock = new Object();

    public void addAndCountBetter(String item) {
        String processed = item.trim().toLowerCase(); // No lock needed

        synchronized (listLock) {
            list.add(processed);
        }
        synchronized (countLock) {
            count++;
        }
    }
}
```

### Reentrant Synchronization

Java's intrinsic locks are **reentrant** - the same thread can acquire the lock it already holds:

```java
public class ReentrantDemo {
    public synchronized void outer() {
        System.out.println("In outer");
        inner(); // ✅ Same thread can re-enter
    }

    public synchronized void inner() {
        System.out.println("In inner");
    }

    // Without reentrancy, calling outer() would DEADLOCK
    // because inner() tries to acquire the lock already held

    // Reentrancy works with inheritance too
    static class Parent {
        public synchronized void doWork() {
            System.out.println("Parent.doWork()");
        }
    }

    static class Child extends Parent {
        @Override
        public synchronized void doWork() {
            System.out.println("Child.doWork()");
            super.doWork(); // ✅ Reentrant - same lock (this)
        }
    }
}
```

### Happens-Before Guarantee

The `synchronized` keyword establishes a **happens-before** relationship:

```java
public class HappensBeforeDemo {
    private int x = 0;
    private int y = 0;
    private final Object lock = new Object();

    // Thread A:
    public void writer() {
        synchronized (lock) {
            x = 1;   // Write to x
            y = 2;   // Write to y
        }
        // All writes inside synchronized are visible to the
        // next thread that enters the same synchronized block
    }

    // Thread B (executes after Thread A releases lock):
    public void reader() {
        synchronized (lock) {
            // Guaranteed to see x=1, y=2 (not stale values)
            System.out.println("x=" + x + ", y=" + y);
        }
    }
}
```

### Double-Checked Locking Pattern

```java
public class Singleton {
    // ❌ Broken double-checked locking (without volatile)
    // private static Singleton instance;

    // ✅ Correct double-checked locking
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                // First check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {        // Second check (with lock)
                    instance = new Singleton();
                    // Without volatile, another thread might see
                    // a partially constructed object!
                }
            }
        }
        return instance;
    }

    // ✅ Better alternative: Initialization-on-Demand Holder
    // (see q20-thread-safety-patterns.md)
}
```

> ⚠️ **Why volatile is required:** Without `volatile`, the JVM may reorder the constructor and field assignment. Thread B could see a non-null `instance` that is not fully constructed.

### Common Synchronization Mistakes

```java
public class SyncMistakes {

    // ❌ Mistake 1: Synchronizing on a non-final object
    private Object lock = new Object();
    public void wrongLock() {
        synchronized (lock) {
            // If someone does: lock = new Object(); → no protection!
        }
    }
    // ✅ Fix: make lock final
    private final Object safeLock = new Object();

    // ❌ Mistake 2: Synchronizing on boxed types or String literals
    private final Integer badLock = 42;
    public void wrongLock2() {
        synchronized (badLock) { // Integers are cached! Multiple classes may share!
        }
    }

    // ❌ Mistake 3: Synchronizing on 'this' in a library class
    public synchronized void publicMethod() {
        // External code can also synchronize on this object!
        // someObj.synchronized { ... } → potential deadlock
    }
    // ✅ Fix: use a private lock object
    private final Object privateLock = new Object();
    public void betterMethod() {
        synchronized (privateLock) { /* safe */ }
    }

    // ❌ Mistake 4: Inconsistent locking
    private int count = 0;
    public synchronized void increment() { count++; }
    public int getCount() { return count; } // ❌ Not synchronized! Stale reads possible.
    // ✅ Fix: synchronize both read and write
    public synchronized int getCountSafe() { return count; }
}
```

### Performance Impact

```java
public class SyncPerformance {

    // ❌ Over-synchronized: locks the entire method
    public synchronized List<String> processAll(List<String> items) {
        List<String> results = new ArrayList<>();
        for (String item : items) {
            results.add(expensiveTransform(item)); // Holds lock during CPU work
        }
        return results;
    }

    // ✅ Minimize synchronized scope
    private final List<String> sharedResults = new ArrayList<>();

    public void processBetter(List<String> items) {
        // Do expensive work OUTSIDE the lock
        List<String> localResults = new ArrayList<>();
        for (String item : items) {
            localResults.add(expensiveTransform(item));
        }

        // Only lock for the shared mutation
        synchronized (sharedResults) {
            sharedResults.addAll(localResults);
        }
    }

    private String expensiveTransform(String item) {
        // CPU-intensive work
        return item.toUpperCase();
    }
}
```

### Thread-Safe Counter Example

```java
public class ThreadSafeCounter {
    private int count = 0;

    public synchronized void increment() { count++; }
    public synchronized void decrement() { count--; }
    public synchronized int getCount() { return count; }

    public static void main(String[] args) throws InterruptedException {
        ThreadSafeCounter counter = new ThreadSafeCounter();

        // Launch 100 threads, each incrementing 1000 times
        Thread[] threads = new Thread[100];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }

        for (Thread t : threads) { t.join(); }

        System.out.println("Final count: " + counter.getCount()); // Always 100000
    }
}
```

---

## Anti-Patterns

❌ **Synchronizing on a non-final reference** - lock object can change  
❌ **Synchronizing on String literals or boxed primitives** - shared across JVM  
❌ **Over-synchronization** - holding locks during I/O or CPU-intensive work  
❌ **Inconsistent locking** - some access paths skip synchronization  
❌ **Nested synchronized blocks** - risk of deadlock

## Best Practices

✅ Use `private final Object lock` for lock objects  
✅ Keep synchronized blocks as short as possible  
✅ Synchronize all access paths to shared mutable state  
✅ Prefer `java.util.concurrent` utilities over raw synchronization  
✅ Document the locking policy (which lock protects which state)  
✅ Consider `ReentrantLock` for advanced features (tryLock, fairness)

---

[← Executor Framework](./q06-executor-framework.md) | [Next: Volatile Keyword →](./q08-volatile-keyword.md)
