# Q6: What Are Phantom References in Java?

[← Back to Memory Management](./README.md)

---

## Overview

A `PhantomReference<T>` is the weakest reference type in Java. Its `get()` method **always returns null**, and it is enqueued into a `ReferenceQueue` only **after** the referent has been finalized and is about to be reclaimed. Phantom references are used for **pre-mortem cleanup** — scheduling cleanup actions when an object is about to be collected.

---

## Reference Type Comparison

| Type               | `get()`         | Cleared when             | Use case                     |
|--------------------|-----------------|--------------------------|------------------------------|
| **Strong**         | Always object   | Never (while reachable)  | Normal references            |
| **Soft**           | Object or null  | Under memory pressure    | Memory-sensitive caches      |
| **Weak**           | Object or null  | Next GC cycle            | Metadata, canonical maps     |
| **Phantom**        | Always null     | After finalization       | Resource cleanup, tracking   |

---

## Basic Usage

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

public class PhantomReferenceDemo {

    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        Object obj = new Object();

        // PhantomReference REQUIRES a ReferenceQueue
        PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);

        System.out.println("get(): " + phantom.get());              // Always null
        System.out.println("isEnqueued: " + phantom.isEnqueued());  // false

        // Remove strong reference
        obj = null;
        System.gc();
        Thread.sleep(200);

        // Check if the reference was enqueued
        if (queue.poll() != null) {
            System.out.println("Object has been finalized and is ready for collection");
            // Perform cleanup actions here
        }
    }
}
```

### Why does get() return null?

Phantom references prevent you from **resurrecting** the object. Since the object may already be finalized, accessing it would be unsafe. By returning null, the API enforces this safety guarantee.

---

## Phantom References for Native Resource Cleanup

The primary use case is cleaning up native resources associated with Java objects:

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class NativeMemoryTracker {

    private static final ReferenceQueue<Object> QUEUE = new ReferenceQueue<>();
    private static final Map<PhantomReference<?>, Long> NATIVE_HANDLES =
            new ConcurrentHashMap<>();

    // Register an object that holds a native resource
    public static <T> PhantomReference<T> track(T obj, long nativeHandle) {
        PhantomReference<T> ref = new PhantomReference<>(obj, QUEUE);
        NATIVE_HANDLES.put(ref, nativeHandle);
        return ref;
    }

    // Run this in a background thread
    public static void startCleanupThread() {
        Thread cleanupThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // Block until a phantom reference is enqueued
                    PhantomReference<?> ref = (PhantomReference<?>) QUEUE.remove();

                    Long handle = NATIVE_HANDLES.remove(ref);
                    if (handle != null) {
                        freeNativeMemory(handle);
                        System.out.println("Freed native handle: " + handle);
                    }

                    ref.clear();   // Allow the PhantomReference itself to be GC'd
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "NativeMemory-Cleanup");

        cleanupThread.setDaemon(true);
        cleanupThread.start();
    }

    private static void freeNativeMemory(long handle) {
        // JNI call to free native memory
        System.out.println("Native memory at " + handle + " released");
    }
}
```

```java
// Usage
public class LargeNativeBuffer {

    private final long nativePtr;

    public LargeNativeBuffer(int sizeBytes) {
        this.nativePtr = allocateNative(sizeBytes);
        NativeMemoryTracker.track(this, nativePtr);
    }

    private static long allocateNative(int size) {
        // Simulated native allocation
        return System.nanoTime();   // Pretend this is a pointer
    }
}

// When LargeNativeBuffer is GC'd, the cleanup thread frees the native memory
```

---

## The Cleaner API (Java 9+) — Modern Alternative

`java.lang.ref.Cleaner` is built on phantom references internally but provides a much cleaner API:

```java
import java.lang.ref.Cleaner;

public class ManagedBuffer implements AutoCloseable {

    private static final Cleaner CLEANER = Cleaner.create();

    // Cleaning state — must be a static class or a separate class
    // ❌ NEVER use a lambda that captures 'this'
    private static class BufferDeallocator implements Runnable {
        private long nativeAddress;
        private int size;

        BufferDeallocator(long address, int size) {
            this.nativeAddress = address;
            this.size = size;
        }

        @Override
        public void run() {
            if (nativeAddress != 0) {
                System.out.printf("Cleaner: freeing %d bytes at 0x%X%n", size, nativeAddress);
                // freeNativeMemory(nativeAddress);
                nativeAddress = 0;
            }
        }
    }

    private final long nativeAddress;
    private final Cleaner.Cleanable cleanable;

    public ManagedBuffer(int size) {
        this.nativeAddress = allocateNative(size);
        BufferDeallocator deallocator = new BufferDeallocator(nativeAddress, size);
        this.cleanable = CLEANER.register(this, deallocator);
    }

    // ✅ Deterministic cleanup (preferred path)
    @Override
    public void close() {
        cleanable.clean();   // Idempotent
    }

    private static long allocateNative(int size) {
        return 0xCAFEBABEL;
    }
}
```

```java
// ✅ Best usage: try-with-resources calls close() deterministically
try (ManagedBuffer buffer = new ManagedBuffer(4096)) {
    // Use buffer
}   // close() called here — Cleaner acts as a safety net if close() is forgotten
```

---

## Phantom Reference vs finalize()

| Aspect              | `finalize()`                    | PhantomReference / Cleaner     |
|---------------------|---------------------------------|--------------------------------|
| **Object access**   | Full access to object           | Cannot access (get() = null)   |
| **Resurrection**    | Possible (dangerous)            | Impossible                     |
| **Thread**          | Arbitrary finalizer thread      | Your cleanup thread / Cleaner  |
| **Ordering**        | No guaranteed order             | FIFO in ReferenceQueue         |
| **Performance**     | Adds GC overhead                | Minimal overhead               |
| **Status**          | Deprecated (Java 9+)            | Recommended approach           |

---

## Direct ByteBuffer — Real-World Phantom Reference Usage

The JDK uses phantom references internally for `DirectByteBuffer`:

```java
import java.nio.ByteBuffer;

public class DirectBufferDemo {

    public static void main(String[] args) {
        // Allocates native memory — cleaned via internal phantom reference (Cleaner)
        ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024 * 1024);   // 1MB native

        directBuffer.putInt(42);
        directBuffer.flip();
        System.out.println(directBuffer.getInt());   // 42

        // When directBuffer is GC'd, the Cleaner frees the native memory
        // You can also manually free it:
        // ((sun.misc.Cleaner) ((DirectBuffer) directBuffer).cleaner()).clean();
    }
}
```

---

## Building a Resource Leak Detector

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class LeakDetector {

    private static final ReferenceQueue<Object> QUEUE = new ReferenceQueue<>();
    private static final Set<TrackedRef> TRACKED = ConcurrentHashMap.newKeySet();

    static class TrackedRef extends PhantomReference<Object> {
        final String description;
        final Throwable allocationSite;
        boolean closed = false;

        TrackedRef(Object referent, String description) {
            super(referent, QUEUE);
            this.description = description;
            this.allocationSite = new Throwable("Allocated here");
        }
    }

    public static TrackedRef track(Object obj, String description) {
        TrackedRef ref = new TrackedRef(obj, description);
        TRACKED.add(ref);
        return ref;
    }

    public static void markClosed(TrackedRef ref) {
        ref.closed = true;
    }

    public static void checkForLeaks() {
        TrackedRef ref;
        while ((ref = (TrackedRef) QUEUE.poll()) != null) {
            TRACKED.remove(ref);
            if (!ref.closed) {
                System.err.println("LEAK DETECTED: " + ref.description);
                ref.allocationSite.printStackTrace(System.err);
            }
            ref.clear();
        }
    }
}
```

```java
// Usage
public class DatabaseConnection implements AutoCloseable {

    private final LeakDetector.TrackedRef leakRef;

    public DatabaseConnection(String url) {
        // ... open connection ...
        this.leakRef = LeakDetector.track(this, "DBConnection to " + url);
    }

    @Override
    public void close() {
        LeakDetector.markClosed(leakRef);
        // ... close connection ...
    }
}
// If close() is never called and the object is GC'd, LeakDetector reports the leak
```

---

## Best Practices

```java
// ✅ Always use PhantomReference with a ReferenceQueue (it's required)
// ✅ Use Cleaner API (Java 9+) instead of raw PhantomReference when possible
// ✅ Keep cleanup state separate from the tracked object (static inner class)
// ✅ Call ref.clear() after processing to allow the PhantomReference itself to be GC'd
// ✅ Use a daemon thread for cleanup polling

// ❌ Don't try to access the referent via get() — it always returns null
// ❌ Don't use finalize() — use Cleaner or PhantomReference instead
// ❌ Don't capture 'this' in the Cleaner's Runnable — it prevents GC
// ❌ Don't rely solely on phantom references — always provide a close() method
```

---

## Interview Tips
- `PhantomReference.get()` always returns null — this is by design to prevent resurrection
- Phantom references are enqueued **after** finalization, not before
- The `Cleaner` API (Java 9+) is essentially a built-in phantom reference manager
- `DirectByteBuffer` is the most famous real-world use of phantom references in the JDK
- Always pair phantom-reference cleanup with an explicit `close()` — the cleanup is a safety net

---

[← Q5: Soft References](./q05-soft-references.md) | [Next: Q7 - Reference Queues →](./q07-reference-queues.md)
