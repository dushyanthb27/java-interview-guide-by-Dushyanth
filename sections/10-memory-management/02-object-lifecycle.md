# Q2: Explain the Object Lifecycle in Java

[← Back to Memory Management](./README.md)

---

## Overview

Every Java object goes through a well-defined lifecycle — from creation to garbage collection. Understanding each phase helps you write memory-efficient code and avoid subtle bugs like object resurrection.

---

## Object Creation

Objects can be created through several mechanisms:

### 1. The `new` keyword (most common)

```java
Person person = new Person("Alice", 30);
```

**What happens internally:**
1. JVM allocates memory on the heap (typically in Eden space)
2. Memory is zero-initialized (fields get default values)
3. Constructor chain is invoked (`<init>` method)
4. Reference is returned to the caller

### 2. Reflection

```java
// Using Class.newInstance() — deprecated since Java 9
Person p1 = Person.class.newInstance();

// ✅ Using Constructor (preferred)
Constructor<Person> ctor = Person.class.getDeclaredConstructor(String.class, int.class);
Person p2 = ctor.newInstance("Bob", 25);
```

### 3. Clone

```java
public class Person implements Cloneable {
    private String name;
    private Address address;

    @Override
    public Person clone() throws CloneNotSupportedException {
        Person cloned = (Person) super.clone();     // Shallow copy
        cloned.address = this.address.clone();       // Deep copy for mutable fields
        return cloned;
    }
}

Person original = new Person("Alice", new Address("NYC"));
Person copy = original.clone();   // No constructor called!
```

### 4. Deserialization

```java
// Object is reconstructed from bytes — no constructor called
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"));
Person deserialized = (Person) ois.readObject();
```

### 5. Factory methods and builders

```java
// These ultimately use 'new' internally
List<String> list = List.of("a", "b", "c");
Optional<String> opt = Optional.of("value");
```

---

## Object Reachability States

The JVM classifies objects into **five reachability states** that determine GC eligibility:

```
  Strongly Reachable
         │
         ▼
  Softly Reachable      (SoftReference)
         │
         ▼
  Weakly Reachable      (WeakReference)
         │
         ▼
  Phantom Reachable     (PhantomReference)
         │
         ▼
    Unreachable          (eligible for GC)
```

### 1. Strongly Reachable

```java
// Object reachable through a chain of strong references from a GC root
Person person = new Person("Alice");   // Strongly reachable
// GC will NEVER collect a strongly reachable object
```

### 2. Softly Reachable

```java
// Only reachable through SoftReference — collected under memory pressure
SoftReference<byte[]> cache = new SoftReference<>(new byte[10_000_000]);

byte[] data = cache.get();   // May return null if GC cleared it
if (data == null) {
    data = loadFromDisk();   // Reload on cache miss
    cache = new SoftReference<>(data);
}
```

### 3. Weakly Reachable

```java
// Only reachable through WeakReference — collected at next GC cycle
WeakReference<Person> weakRef = new WeakReference<>(new Person("Bob"));

Person p = weakRef.get();   // May return null after any GC
```

### 4. Phantom Reachable

```java
// Only reachable through PhantomReference — get() always returns null
ReferenceQueue<Person> queue = new ReferenceQueue<>();
PhantomReference<Person> phantom = new PhantomReference<>(new Person("Charlie"), queue);

phantom.get();   // Always returns null
// Used for post-mortem cleanup actions
```

### 5. Unreachable

```java
public void process() {
    Person temp = new Person("Dave");
    // ... use temp ...
}   // After method returns, 'temp' object is unreachable — eligible for GC
```

---

## GC Eligibility

An object becomes eligible for garbage collection when no strong reference chain from any **GC root** reaches it.

### GC Roots include:
- Local variables and parameters of currently executing methods
- Active thread objects
- Static fields of loaded classes
- JNI references

```java
public class GCEligibility {

    public static void main(String[] args) {
        Object a = new Object();   // Object A — reachable via 'a'
        Object b = new Object();   // Object B — reachable via 'b'

        a = b;                     // Object A is now unreachable — eligible for GC
        b = null;                  // Object B still reachable via 'a'
        a = null;                  // Object B now unreachable — eligible for GC
    }
}
```

### Circular references are collected

```java
// ✅ GC handles circular references (unlike reference counting)
class Node {
    Node next;
}

Node a = new Node();
Node b = new Node();
a.next = b;
b.next = a;    // Circular reference

a = null;
b = null;      // Both objects are unreachable — GC collects both
```

---

## Finalization (Deprecated)

> ⚠️ `finalize()` has been **deprecated since Java 9** and **removed for deprecation in Java 18**.

```java
// ❌ Anti-pattern: using finalize()
public class ResourceHolder {

    private long nativeHandle;

    @Override
    protected void finalize() throws Throwable {
        try {
            releaseNativeResource(nativeHandle);
        } finally {
            super.finalize();
        }
    }
}
```

### Problems with finalize()

| Problem                | Description                                             |
|------------------------|---------------------------------------------------------|
| Non-deterministic      | No guarantee when (or if) it will run                   |
| Performance penalty    | Finalizable objects require 2 GC cycles to collect      |
| Resurrection risk      | Object can make itself reachable again inside finalize  |
| Thread-safety issues   | Finalizer runs in an arbitrary thread                   |
| Blocks GC              | Slow finalizers create backlog in finalizer queue        |

---

## The Cleaner API (Java 9+)

The modern replacement for `finalize()`:

```java
import java.lang.ref.Cleaner;

public class NativeResource implements AutoCloseable {

    private static final Cleaner CLEANER = Cleaner.create();

    // Cleaning action — must NOT reference the enclosing object!
    private static class CleanupAction implements Runnable {
        private long nativeHandle;

        CleanupAction(long handle) {
            this.nativeHandle = handle;
        }

        @Override
        public void run() {
            if (nativeHandle != 0) {
                releaseNative(nativeHandle);   // Free native resource
                nativeHandle = 0;
                System.out.println("Native resource cleaned up");
            }
        }

        private static void releaseNative(long handle) {
            // JNI call to free native memory
        }
    }

    private final Cleaner.Cleanable cleanable;
    private final CleanupAction cleanupAction;

    public NativeResource() {
        long handle = allocateNative();
        this.cleanupAction = new CleanupAction(handle);
        this.cleanable = CLEANER.register(this, cleanupAction);
    }

    // ✅ Primary cleanup mechanism — deterministic
    @Override
    public void close() {
        cleanable.clean();   // Idempotent — safe to call multiple times
    }

    private static long allocateNative() {
        return 0xDEADBEEFL;   // Simulated handle
    }
}
```

```java
// ✅ Best practice: use try-with-resources for deterministic cleanup
try (NativeResource resource = new NativeResource()) {
    // Use resource
}   // close() called automatically — Cleaner is just a safety net
```

---

## Object Resurrection

An object can be "resurrected" during finalization by making itself reachable again:

```java
// ❌ Anti-pattern: object resurrection
public class Zombie {

    static Zombie instance;   // Static field holds resurrected reference

    @Override
    protected void finalize() throws Throwable {
        System.out.println("Finalize called — resurrecting!");
        instance = this;      // Object becomes reachable again!
    }

    public static void main(String[] args) throws Exception {
        instance = new Zombie();
        instance = null;                   // Eligible for GC

        System.gc();
        Thread.sleep(500);

        System.out.println(instance);      // Not null — resurrected!

        instance = null;                   // Eligible again
        System.gc();
        Thread.sleep(500);

        System.out.println(instance);      // null — finalize() runs only ONCE
    }
}
```

> **Key insight:** `finalize()` is called **at most once** per object. After resurrection and a second unreachability, the object is collected without finalization.

---

## Complete Lifecycle Diagram

```
  ┌──────────────┐
  │  Allocation  │   new, reflection, clone, deserialization
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Initialization│   Constructor chain, field assignment
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │   In Use     │   Strongly reachable from GC roots
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Unreachable  │   No strong reference path from roots
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │  Finalizable │   If finalize() overridden (legacy)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │  Collected   │   Memory reclaimed by GC
  └──────────────┘
```

---

## Best Practices

```java
// ✅ Use try-with-resources for deterministic cleanup
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // Use resources
}

// ✅ Use Cleaner API as a safety net (Java 9+)
// ✅ Nullify references in long-lived scopes when done
largeCache = null;   // Help GC in long-running methods

// ❌ Don't override finalize()
// ❌ Don't rely on GC for resource cleanup
// ❌ Don't resurrect objects in finalizers
// ❌ Don't call System.gc() in production code
```

---

## Interview Tips
- Know all four creation mechanisms and which ones skip the constructor
- Understand the five reachability states and their ordering
- Be ready to explain why `finalize()` is deprecated and what replaces it
- Circular references are NOT a problem in Java — the GC uses reachability, not reference counting

---

[← Q1: Stack vs Heap](./q01-stack-vs-heap.md) | [Next: Q3 - Memory Leaks →](./q03-memory-leaks.md)
