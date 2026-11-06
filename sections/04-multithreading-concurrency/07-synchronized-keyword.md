# Q7: How Does the Synchronized Keyword Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## 📋 Question

How does the `synchronized` keyword work in Java? What are monitor locks, intrinsic locks, object header mark words, JVM lightweight/heavyweight locking mechanisms, and the happens-before guarantee?

## Difficulty: 🟡 Intermediate

---

## 💡 Answer

### What is Synchronization?

Synchronization ensures that only **one thread at a time** can execute a critical section of code, providing both **mutual exclusion** (preventing race conditions) and **memory visibility** (ensuring modifications made by one thread are visible to subsequent threads).

```
Without synchronization:           With synchronization:
Thread A: read(count=5)            Thread A: LOCK → read(5) → write(6) → UNLOCK
Thread B: read(count=5)            Thread B: WAIT...
Thread A: write(count=6)           Thread B: LOCK → read(6) → write(7) → UNLOCK
Thread B: write(count=6)
Result: 6 (WRONG! lost update)    Result: 7 (CORRECT!)
```

### Monitor Lock / Intrinsic Lock & JVM Locking Internals

Every Java object header contains a **Mark Word** that stores synchronization state, hash codes, and GC metadata. The `synchronized` keyword relies on **intrinsic locks** (or **monitor locks**):

```java
public class MonitorLockDemo {
    private int count = 0;

    // 1. Synchronized instance method — locks on 'this'
    public synchronized void increment() {
        count++; // Only one thread at a time per instance
    }

    // 2. Synchronized static method — locks on Class object (MonitorLockDemo.class)
    public static synchronized void staticMethod() {
        // Protected by the Class object's monitor
    }

    // 3. Synchronized block — locks on a specified monitor object
    private final Object lock = new Object();

    public void blockSync() {
        synchronized (lock) {
            count++; // Granular locking
        }
    }
}
```

#### JVM Optimization Phases (HotSpot JVM)
Hotspot escalates locking overhead dynamically depending on contention:
1. **Biased Locking (Deprecated/Disabled in modern Java 17/21 due to Project Loom and complex revocation overhead):** Originally optimized for single-threaded repeated lock acquisition by recording the thread ID directly in the object header mark word.
2. **Lightweight Locking:** Uses Compare-And-Swap (CAS) operations on the thread's stack (via a Lock Record) to acquire the lock without OS kernel intervention when contention is minimal (spinning / short critical sections).
3. **Heavyweight Locking (ObjectMonitor):** When contention is high (multiple threads spinning fail), the JVM inflates the lock into an OS-level mutex (`ObjectMonitor`), putting contending threads to sleep via OS parking (`park()` / `unpark()`), which incurs context-switch overhead.

### Synchronized Method vs Block

```java
public class SyncComparison {
    private final List<String> list = new ArrayList<>();
    private int count = 0;

    // ❌ Synchronized method - locks entire method on 'this'
    public synchronized void addAndCount(String item) {
        String processed = item.trim().toLowerCase(); // Doesn't need synchronization
        list.add(processed);
        count++;
    }

    // ✅ Synchronized block - fine-grained locking
    private final Object listLock = new Object();
    private final Object countLock = new Object();

    public void addAndCountBetter(String item) {
        String processed = item.trim().toLowerCase(); // CPU work outside lock

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

Java intrinsic locks are **reentrant**—a thread can acquire a lock it already holds without deadlocking itself:

```java
public class ReentrantDemo {
    public synchronized void outer() {
        System.out.println("In outer");
        inner(); // ✅ Same thread re-enters successfully
    }

    public synchronized void inner() {
        System.out.println("In inner");
    }

    // Reentrancy also applies across inheritance hierarchies
    static class Parent {
        public synchronized void doWork() {
            System.out.println("Parent work");
        }
    }

    static class Child extends Parent {
        @Override
        public synchronized void doWork() {
            System.out.println("Child work");
            super.doWork(); // ✅ Same monitor ('this') is re-entered
        }
    }
}
```

### Happens-Before Guarantee & Memory Barriers

The Java Memory Model (JMM) defines a **happens-before** relationship for synchronization:
- **Unlock happens-before every subsequent lock** on the same monitor.
- This enforces **acquire-release semantics**: 
  - Exiting a synchronized block flushes all CPU cache writes to main memory (StoreStore / StoreLoad memory barriers).
  - Entering a synchronized block invalidates local CPU caches, forcing a reload from main memory.

```java
public class HappensBeforeDemo {
    private int x = 0;
    private int y = 0;
    private final Object lock = new Object();

    public void writer() {
        synchronized (lock) {
            x = 1;
            y = 2;
        } // Exit releases lock -> flushes writes to main memory
    }

    public void reader() {
        synchronized (lock) {
            // Entry acquires lock -> invalidates cache, guarantees x=1, y=2 are visible
            System.out.println("x=" + x + ", y=" + y);
        }
    }
}
```

---

## ⚠️ Gotchas & Pitfalls

1. **Lock Object Mutability:** Synchronizing on a non-final reference (`private Object lock = new Object();` where `lock` is reassigned later) completely breaks mutual exclusion because different threads lock on entirely different monitor instances.
2. **Synchronizing on Shared Constants, String Literals, or Boxed Primitives:** Using `synchronized(Integer.valueOf(42))` or `synchronized("myLock")` causes unintended global synchronization because String literals and Integer cache values (-128 to 127) are shared across the entire JVM classloader.
3. **Synchronizing on `this` in Public Classes:** If your class is non-final or public, external callers can synchronize on your instance (`synchronized(myObj) { ... }`), leading to unexpected deadlocks or denial of service if external code holds your object's monitor indefinitely.
4. **Holding Locks During I/O or External Calls:** Executing blocking network calls, database queries, or calling unknown foreign methods inside a `synchronized` block starves all other threads waiting for the lock and can cascade into thread pool exhaustion.
5. **Inconsistent Locking Policies:** Protecting shared mutable state with a synchronized block in setters while leaving getters un-synchronized (or vice versa) permits stale reads and data corruption.

---

## ✅ Best Practices

- Always declare lock objects as `private final Object lock = new Object();`.
- Keep critical sections as short and lightweight as possible; perform heavy computation or I/O outside the synchronized block.
- Document the locking policy clearly (e.g., annotate fields with `// Guarded by this lock`).
- Prefer `java.util.concurrent.locks` (`ReentrantLock`) or atomic variables (`java.util.concurrent.atomic`) when advanced features like `tryLock`, timed waits, or non-blocking CAS are required.

---

## 🎯 Interviewer Insight

Strong candidates articulate more than just syntax: they explain monitor acquisition via object header mark words, the cost of JVM lock inflation (from lightweight CAS locks to heavyweight OS mutexes), and acquire-release memory barriers establishing happens-before consistency. Be ready to contrast `synchronized` with explicit `ReentrantLock` semantics.

## ⚡ Follow-up Questions

1. How does the JVM perform lock inflation from biased locking (historical) to lightweight and heavyweight locking?
2. What are the memory barrier instructions generated by the CPU during monitor enter and exit?
3. Why does synchronizing on String literals cause severe concurrency bottlenecks?
4. How does `synchronized` interact with modern Project Loom virtual threads (pinning)?
5. Can a thread holding a monitor lock experience spurious wakeups during `wait()`?

## 📚 Related Topics

- [Volatile Keyword](./08-volatile-keyword.md)
- [Reentrant Lock](./10-reentrant-lock.md)
- [Atomic Classes](./09-atomic-classes.md)

---

[← Executor Framework](./06-executor-framework.md) | [Next: Volatile Keyword →](./08-volatile-keyword.md)
