# Q4: Explain the Java Memory Model (JMM)

## 📋 Question
What is the Java Memory Model? Explain happens-before relationships, memory ordering, and visibility guarantees.

## Difficulty: 🔴 Expert

---

## 💡 Answer

The **Java Memory Model (JMM)**, defined in JSR-133 and JLS §17.4, specifies how threads interact through memory. It defines when a write by one thread is **guaranteed to be visible** to a read by another thread. Without the JMM's guarantees, compilers, CPUs, and caches can **reorder** and **cache** operations in ways that break multithreaded programs.

### The Problem JMM Solves

```
┌──────────────────┐                    ┌──────────────────┐
│     Thread 1     │                    │     Thread 2     │
│                  │                    │                  │
│  ┌────────────┐  │                    │  ┌────────────┐  │
│  │ CPU Cache  │  │                    │  │ CPU Cache  │  │
│  │  x = 42    │  │                    │  │  x = 0 ??  │  │
│  └─────┬──────┘  │                    │  └─────┬──────┘  │
└────────┼─────────┘                    └────────┼─────────┘
         │          ┌──────────────┐             │
         └─────────▶│ Main Memory  │◀────────────┘
                    │   x = ???    │
                    └──────────────┘

Without proper synchronization:
  - Thread 1 may write x=42 to its cache but not flush to main memory
  - Thread 2 may read a stale value of x from its own cache
  - JMM defines rules to prevent this
```

### Happens-Before Relationship

The **happens-before** relationship is the foundation of the JMM. If action A **happens-before** action B, then:
1. A's results are **visible** to B
2. A is **ordered before** B (from B's perspective)

This does NOT mean A executes chronologically before B — it means B can **see** A's effects.

### The Eight Happens-Before Rules

```java
// ═══════════════════════════════════════════════════════════
// Rule 1: PROGRAM ORDER RULE
// ═══════════════════════════════════════════════════════════
// Within a single thread, each statement happens-before the next
int a = 1;    // happens-before
int b = a + 1; // this line → sees a = 1

// ═══════════════════════════════════════════════════════════
// Rule 2: MONITOR LOCK RULE (synchronized)
// ═══════════════════════════════════════════════════════════
// An unlock on a monitor happens-before every subsequent lock on that monitor
synchronized (lock) {
    sharedVar = 42;        // Write inside synchronized
}   // unlock happens-before...

synchronized (lock) {      // ...this lock
    int x = sharedVar;     // Guaranteed to see 42
}

// ═══════════════════════════════════════════════════════════
// Rule 3: VOLATILE VARIABLE RULE
// ═══════════════════════════════════════════════════════════
// A write to a volatile field happens-before every subsequent read of that field
volatile boolean ready = false;

// Thread 1:
data = 42;           // Non-volatile write
ready = true;        // Volatile write — flushes ALL prior writes

// Thread 2:
if (ready) {         // Volatile read — refreshes ALL cached values
    // data is guaranteed to be 42 (piggyback on volatile)
}

// ═══════════════════════════════════════════════════════════
// Rule 4: THREAD START RULE
// ═══════════════════════════════════════════════════════════
// thread.start() happens-before any action in the started thread
sharedVar = 10;
Thread t = new Thread(() -> {
    // Guaranteed to see sharedVar = 10
    System.out.println(sharedVar);
});
t.start();  // start() establishes happens-before

// ═══════════════════════════════════════════════════════════
// Rule 5: THREAD JOIN RULE
// ═══════════════════════════════════════════════════════════
// All actions in a thread happen-before another thread's join() returns
Thread t = new Thread(() -> {
    sharedVar = 42;  // Write in thread
});
t.start();
t.join();            // join() establishes happens-before
// Guaranteed to see sharedVar = 42

// ═══════════════════════════════════════════════════════════
// Rule 6: THREAD INTERRUPT RULE
// ═══════════════════════════════════════════════════════════
// thread.interrupt() happens-before the interrupted thread detects it
t.interrupt();        // happens-before
// Thread t: Thread.interrupted() or catch InterruptedException

// ═══════════════════════════════════════════════════════════
// Rule 7: FINALIZER RULE
// ═══════════════════════════════════════════════════════════
// Constructor completion happens-before finalize() begins
// (Object is fully constructed before finalizer can run)

// ═══════════════════════════════════════════════════════════
// Rule 8: TRANSITIVITY
// ═══════════════════════════════════════════════════════════
// If A happens-before B, and B happens-before C,
// then A happens-before C
```

### Volatile Semantics — Deep Dive

```java
public class VolatileSemantics {
    private int a = 0;
    private int b = 0;
    private volatile boolean flag = false;

    // Thread 1
    public void writer() {
        a = 1;              // (1) Normal write
        b = 2;              // (2) Normal write
        flag = true;        // (3) Volatile write — MEMORY BARRIER
        // At (3): ALL writes before the volatile write (a=1, b=2)
        //         are flushed to main memory
    }

    // Thread 2
    public void reader() {
        if (flag) {         // (4) Volatile read — MEMORY BARRIER
            // At (4): ALL reads after the volatile read
            //         see up-to-date values from main memory
            assert a == 1;  // ✅ Guaranteed by happens-before
            assert b == 2;  // ✅ Guaranteed by happens-before
        }
    }
}
```

```
Volatile Write:                    Volatile Read:
┌──────────────┐                   ┌──────────────┐
│ Normal writes│                   │ Volatile read │ ← LoadLoad + LoadStore barrier
│     ...      │                   ├──────────────┤
├──────────────┤                   │ Normal reads  │
│Volatile write│ ← StoreStore +   │     ...       │
│              │   StoreLoad      └──────────────┘
└──────────────┘   barrier
```

### Synchronized Semantics

```java
public class SynchronizedSemantics {
    private int counter = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {
            // ACQUIRE: Read barrier — invalidate CPU cache
            //          Reads see latest values from main memory

            counter++;       // Read-modify-write is atomic here

            // RELEASE: Write barrier — flush to main memory
            //          All writes are visible to the next thread that locks
        }
    }

    // Equivalent visibility guarantees to:
    // 1. Read all shared state from main memory (acquire)
    // 2. Execute critical section
    // 3. Flush all writes to main memory (release)
}
```

### Final Field Semantics

```java
public class FinalFieldSemantics {
    private final int x;
    private final List<String> items;

    public FinalFieldSemantics() {
        x = 42;
        items = new ArrayList<>();
        items.add("hello");
        // Constructor end establishes happens-before for final fields
    }

    // Any thread that sees a properly-constructed object is
    // guaranteed to see the correct values of its final fields
    // WITHOUT any synchronization

    // ⚠️ This guarantee BREAKS if 'this' escapes during construction:
    // public FinalFieldSemantics() {
    //     SomeRegistry.register(this);  // ❌ 'this' escapes!
    //     x = 42;                        // Another thread might see x = 0
    // }
}
```

### Reordering — Compiler & CPU

```java
// The compiler and CPU may reorder instructions for performance
// as long as single-threaded semantics are preserved

// Original code:
int a = 1;      // (1)
int b = 2;      // (2)
int c = a + b;  // (3) — depends on (1) and (2)

// CPU/Compiler may reorder to:
int b = 2;      // (2) — moved before (1)
int a = 1;      // (1)
int c = a + b;  // (3) — still correct in single thread

// But in multi-threaded code, reordering breaks things:
// Thread 1:                    Thread 2:
// x = 42;          (A)        while (!ready) {}  (C)
// ready = true;    (B)        print(x);          (D)
//
// If CPU reorders B before A in Thread 1:
//   Thread 2 may see ready=true but x=0 (stale!)
//   Making 'ready' volatile prevents this reordering
```

### Memory Barriers (Fences)

```
StoreStore barrier: All stores before the barrier complete before stores after
LoadLoad barrier:   All loads before the barrier complete before loads after
StoreLoad barrier:  All stores before the barrier complete before loads after
LoadStore barrier:  All loads before the barrier complete before stores after

volatile write  =  StoreStore + StoreLoad barriers
volatile read   =  LoadLoad + LoadStore barriers
synchronized    =  All four barriers (acquire + release semantics)
```

### Practical Example — Double-Checked Locking

```java
public class Singleton {
    // ❌ BROKEN without volatile — possible reordering
    // private static Singleton instance;

    // ✅ CORRECT — volatile prevents reordering of constructor
    private static volatile Singleton instance;

    private final String config;

    private Singleton() {
        config = loadConfig();  // Expensive initialization
    }

    public static Singleton getInstance() {
        if (instance == null) {                 // 1st check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {         // 2nd check (with lock)
                    instance = new Singleton(); // Without volatile, this can be reordered:
                    //  1. Allocate memory
                    //  2. Assign reference to instance (instance != null now!)
                    //  3. Call constructor  ← another thread sees non-null but uninitialized!
                    // Volatile prevents step 2 from executing before step 3
                }
            }
        }
        return instance;
    }
}
```

### ⚠️ Common Mistake
Relying on timing or `Thread.sleep()` for visibility. Without a happens-before relationship, a thread may **never** see another thread's writes — not just a delay, but **never**. The JIT compiler can hoist a non-volatile field read out of a loop, making it effectively invisible to updates from other threads:

```java
// ❌ May loop forever — JIT can hoist 'running' read
boolean running = true;
// Thread 2: running = false;
while (running) { /* may never terminate! */ }

// ✅ Fixed with volatile
volatile boolean running = true;
while (running) { /* correctly checks each iteration */ }
```

### ⚡ Performance Tip
`volatile` is cheaper than `synchronized` — it only inserts memory barriers without blocking. Use `volatile` for simple flags and single-variable visibility. Use `synchronized` or `Lock` when you need atomicity across multiple operations. For atomic counters, prefer `AtomicInteger` which uses `CAS` (Compare-And-Swap) — even cheaper than `volatile` writes in some cases.

---

## 🎯 Interviewer Insight
The JMM is an expert-level topic. Interviewers want to hear about happens-before rules, not just "volatile makes things visible." Strong candidates explain **why** volatile fixes double-checked locking (instruction reordering), describe memory barriers, and know the difference between visibility and atomicity. Mentioning the transitivity rule and final field semantics demonstrates deep understanding.

## ⚡ Follow-up Questions
1. Why is double-checked locking broken without `volatile`?
2. What's the difference between visibility and atomicity?
3. Can `volatile` replace `synchronized`?

## 📚 Related Topics
- [Stack vs Heap](./q09-stack-vs-heap.md)
- [JVM Architecture](./q01-jvm-architecture.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
