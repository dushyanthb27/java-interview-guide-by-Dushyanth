# Q7: How Do Reference Queues Work?

[← Back to Memory Management](./README.md)

---

## Overview

A `ReferenceQueue<T>` is a queue that the garbage collector populates with `Reference` objects after their referents have been collected (or are about to be collected). It acts as a **notification mechanism** — allowing your code to react when objects are garbage collected.

---

## How It Works

```
  ┌──────────────┐       ┌──────────────────┐       ┌────────────────┐
  │  Your Object │◄──────│  Reference       │──────▶│ ReferenceQueue │
  │  (referent)  │       │  (Weak/Soft/     │       │                │
  └──────────────┘       │   Phantom)       │       │  Enqueued when │
         │               └──────────────────┘       │  referent is   │
         ▼                                          │  collected     │
  ┌──────────────┐                                  └────────────────┘
  │ GC collects  │                                         │
  │ the referent │─────────────────────────────────────────┘
  └──────────────┘       (GC enqueues the Reference object)
```

### Lifecycle steps:
1. Create a `ReferenceQueue`
2. Create a `Reference` (Weak/Soft/Phantom) and associate it with the queue
3. When the GC determines the referent should be collected, it **enqueues** the `Reference` into the queue
4. Your code **polls** or **blocks** on the queue to process enqueued references

---

## Core API

```java
import java.lang.ref.ReferenceQueue;
import java.lang.ref.Reference;

ReferenceQueue<MyObject> queue = new ReferenceQueue<>();

// Non-blocking check — returns null if queue is empty
Reference<? extends MyObject> ref = queue.poll();

// Blocking wait — blocks until a reference is available
Reference<? extends MyObject> ref2 = queue.remove();

// Blocking wait with timeout (milliseconds)
Reference<? extends MyObject> ref3 = queue.remove(5000);   // Wait up to 5 seconds
```

---

## Registering References with a Queue

```java
import java.lang.ref.*;

public class ReferenceQueueBasics {

    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();

        // WeakReference with queue
        Object weakObj = new Object();
        WeakReference<Object> weakRef = new WeakReference<>(weakObj, queue);

        // SoftReference with queue
        Object softObj = new Object();
        SoftReference<Object> softRef = new SoftReference<>(softObj, queue);

        // PhantomReference with queue (required — can't create without queue)
        Object phantomObj = new Object();
        PhantomReference<Object> phantomRef = new PhantomReference<>(phantomObj, queue);

        // Make weak and phantom objects eligible for GC
        weakObj = null;
        phantomObj = null;

        System.gc();
        Thread.sleep(200);

        // Process enqueued references
        Reference<?> ref;
        while ((ref = queue.poll()) != null) {
            if (ref == weakRef) {
                System.out.println("Weak reference enqueued");
            } else if (ref == phantomRef) {
                System.out.println("Phantom reference enqueued");
            }
            // softRef is NOT enqueued — memory isn't under pressure
        }
    }
}
```

---

## Building a Resource Cleanup System

A complete example showing how to use reference queues for automatic resource cleanup:

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Automatic cleanup system using PhantomReference + ReferenceQueue.
 * Cleans up resources when their owning objects are garbage collected.
 */
public class ResourceCleanupManager {

    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
    private final Map<PhantomReference<?>, Runnable> cleanupActions =
            new ConcurrentHashMap<>();
    private volatile boolean running = true;

    /**
     * Register an object for automatic cleanup.
     *
     * @param tracked      the object to monitor
     * @param cleanupAction the action to run when the object is GC'd
     *                      (must NOT reference the tracked object!)
     */
    public <T> void register(T tracked, Runnable cleanupAction) {
        PhantomReference<T> ref = new PhantomReference<>(tracked, queue);
        cleanupActions.put(ref, cleanupAction);
    }

    /**
     * Start the cleanup thread.
     */
    public void start() {
        Thread cleanupThread = new Thread(() -> {
            while (running) {
                try {
                    // Block until a reference is enqueued (with timeout for shutdown)
                    PhantomReference<?> ref =
                            (PhantomReference<?>) queue.remove(1000);

                    if (ref != null) {
                        Runnable action = cleanupActions.remove(ref);
                        if (action != null) {
                            try {
                                action.run();
                            } catch (Exception e) {
                                System.err.println("Cleanup action failed: " + e);
                            }
                        }
                        ref.clear();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "ResourceCleanup");

        cleanupThread.setDaemon(true);
        cleanupThread.start();
    }

    public void shutdown() {
        running = false;
    }

    public int pendingCleanups() {
        return cleanupActions.size();
    }
}
```

```java
// Usage
public class FileHandle {

    private static final ResourceCleanupManager MANAGER = new ResourceCleanupManager();

    static {
        MANAGER.start();
    }

    private final String path;

    public FileHandle(String path) {
        this.path = path;
        openFile(path);

        // Register cleanup — uses a captured variable, NOT 'this'
        String capturedPath = path;
        MANAGER.register(this, () -> {
            System.out.println("Auto-closing file: " + capturedPath);
            closeFile(capturedPath);
        });
    }

    private void openFile(String path) {
        System.out.println("Opened file: " + path);
    }

    private static void closeFile(String path) {
        System.out.println("Closed file: " + path);
    }
}
```

---

## Integration with the Cleaner API (Java 9+)

The `Cleaner` API wraps `PhantomReference` and `ReferenceQueue` internally:

```java
import java.lang.ref.Cleaner;

public class CleanerIntegration {

    private static final Cleaner CLEANER = Cleaner.create();

    static class Resource {
        private final Cleaner.Cleanable cleanable;

        Resource(String name) {
            // The cleanup state — separate from 'this'
            CleanupState state = new CleanupState(name);
            this.cleanable = CLEANER.register(this, state);
        }

        void close() {
            cleanable.clean();   // Deterministic cleanup
        }
    }

    // ✅ Static class — no reference to Resource instance
    static class CleanupState implements Runnable {
        private final String name;

        CleanupState(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            System.out.println("Cleaned up: " + name);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Resource res = new Resource("my-resource");
        res = null;   // Eligible for GC

        System.gc();
        Thread.sleep(500);
        // Output: "Cleaned up: my-resource"
    }
}
```

### How Cleaner works internally

```
Cleaner.create()
    └── Creates a ReferenceQueue
    └── Starts a daemon thread that blocks on queue.remove()

Cleaner.register(obj, action)
    └── Creates a PhantomReference(obj, queue)
    └── Maps reference → action

When obj is GC'd:
    └── GC enqueues PhantomReference into the queue
    └── Cleaner's daemon thread wakes up
    └── Runs the associated action
```

---

## Custom Reference-Based Cache with Queue Cleanup

```java
import java.lang.ref.SoftReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.HashMap;

public class AutoCleaningCache<K, V> {

    private final Map<K, SoftEntry<K, V>> cache = new HashMap<>();
    private final ReferenceQueue<V> queue = new ReferenceQueue<>();

    /**
     * SoftReference that remembers its key for removal from the cache map.
     */
    static class SoftEntry<K, V> extends SoftReference<V> {
        final K key;

        SoftEntry(K key, V value, ReferenceQueue<V> queue) {
            super(value, queue);
            this.key = key;
        }
    }

    public void put(K key, V value) {
        expungeStaleEntries();
        cache.put(key, new SoftEntry<>(key, value, queue));
    }

    public V get(K key) {
        expungeStaleEntries();
        SoftEntry<K, V> entry = cache.get(key);
        return (entry != null) ? entry.get() : null;
    }

    public int size() {
        expungeStaleEntries();
        return cache.size();
    }

    /**
     * Remove entries whose values have been garbage collected.
     * This is the same pattern used by WeakHashMap internally.
     */
    @SuppressWarnings("unchecked")
    private void expungeStaleEntries() {
        SoftEntry<K, V> entry;
        while ((entry = (SoftEntry<K, V>) queue.poll()) != null) {
            K key = entry.key;
            // Only remove if the current entry is the stale one
            // (it might have been replaced with a new entry for the same key)
            if (cache.get(key) == entry) {
                cache.remove(key);
            }
        }
    }
}
```

---

## Polling vs Blocking

| Method                  | Behavior                                        | Use case                  |
|-------------------------|-------------------------------------------------|---------------------------|
| `queue.poll()`          | Returns immediately; null if empty              | Periodic cleanup          |
| `queue.remove()`        | Blocks until a reference is available           | Dedicated cleanup thread  |
| `queue.remove(timeout)` | Blocks for up to `timeout` ms                   | Interruptible cleanup     |

```java
// Polling pattern — good for periodic cleanup in application threads
public void periodicCleanup() {
    Reference<?> ref;
    while ((ref = queue.poll()) != null) {
        processCollectedReference(ref);
    }
}

// Blocking pattern — good for dedicated cleanup threads
public void blockingCleanup() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            Reference<?> ref = queue.remove();   // Blocks
            processCollectedReference(ref);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}
```

---

## Best Practices

```java
// ✅ Use a dedicated daemon thread for blocking queue processing
// ✅ Always call ref.clear() after processing to help GC
// ✅ Store cleanup data OUTSIDE the referent (static class, captured variables)
// ✅ Use ConcurrentHashMap for the reference → action map (multiple threads)
// ✅ Handle exceptions in cleanup actions — don't let them kill the cleanup thread
// ✅ Prefer Cleaner API (Java 9+) over raw PhantomReference + ReferenceQueue

// ❌ Don't poll() in a tight loop without sleeping (CPU waste)
// ❌ Don't store a strong reference to the referent in the cleanup action
// ❌ Don't forget to remove processed references from tracking maps
// ❌ Don't ignore InterruptedException in cleanup threads
```

---

## Interview Tips
- `ReferenceQueue` is the GC's callback mechanism — it notifies you when references are cleared
- `WeakHashMap` uses reference queues internally to expunge stale entries
- `PhantomReference` always requires a `ReferenceQueue`; for Soft/Weak it's optional
- The `Cleaner` API is built on `PhantomReference` + `ReferenceQueue` internally
- `queue.remove()` blocks; `queue.poll()` returns immediately — choose based on your cleanup strategy

---

[← Q6: Phantom References](./q06-phantom-references.md) | [Next: Q8 - Object Pooling →](./q08-object-pooling.md)
