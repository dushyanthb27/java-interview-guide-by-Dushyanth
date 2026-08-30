# Q6: What Are Reference Queues in Java?

## 📋 Question
What are Reference Queues in Java? How do they work with different reference types, and when should you use them?

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

A `ReferenceQueue<T>` is a queue that the GC uses to notify you when a reference's referent has been collected. When you create a reference (Weak, Soft, or Phantom) with a ReferenceQueue, the GC enqueues the reference to the queue when the referent is collected, allowing you to perform cleanup actions.

### Basic Usage

```java
import java.lang.ref.WeakReference;
import java.lang.ref.ReferenceQueue;

public class ReferenceQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        Object obj = new Object();

        WeakReference<Object> weakRef = new WeakReference<>(obj, queue);

        System.out.println("Before GC: " + weakRef.get());   // Not null

        obj = null;    // Remove strong reference
        System.gc();
        Thread.sleep(100);

        System.out.println("After GC: " + weakRef.get());    // null

        // Check if the reference was enqueued
        if (queue.poll() == weakRef) {
            System.out.println("WeakReference was enqueued — object collected");
        }
    }
}
```

### Reference Queue with Weak References

```java
import java.lang.ref.WeakReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.HashMap;

public class TrackedWeakCache<K, V> {

    private final Map<K, KeyedWeakReference<K, V>> cache = new HashMap<>();
    private final ReferenceQueue<V> queue = new ReferenceQueue<>();

    /**
     * Custom WeakReference that remembers its key for cleanup.
     */
    static class KeyedWeakReference<K, V> extends WeakReference<V> {
        final K key;

        KeyedWeakReference(K key, V value, ReferenceQueue<V> queue) {
            super(value, queue);
            this.key = key;
        }
    }

    public void put(K key, V value) {
        cleanupStale();   // Purge collected entries
        cache.put(key, new KeyedWeakReference<>(key, value, queue));
    }

    public V get(K key) {
        cleanupStale();
        KeyedWeakReference<K, V> ref = cache.get(key);
        return (ref != null) ? ref.get() : null;
    }

    @SuppressWarnings("unchecked")
    private void cleanupStale() {
        KeyedWeakReference<K, V> ref;
        while ((ref = (KeyedWeakReference<K, V>) queue.poll()) != null) {
            cache.remove(ref.key);   // Remove the map entry
            System.out.println("Evicted cache entry for key: " + ref.key);
        }
    }

    public int size() {
        cleanupStale();
        return cache.size();
    }
}
```

### Reference Queue with Soft References

```java
import java.lang.ref.SoftReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.HashMap;

public class TrackedSoftCache<K, V> {

    private final Map<K, KeyedSoftReference<K, V>> cache = new HashMap<>();
    private final ReferenceQueue<V> queue = new ReferenceQueue<>();

    static class KeyedSoftReference<K, V> extends SoftReference<V> {
        final K key;

        KeyedSoftReference(K key, V value, ReferenceQueue<V> queue) {
            super(value, queue);
            this.key = key;
        }
    }

    public void put(K key, V value) {
        cleanupStale();
        cache.put(key, new KeyedSoftReference<>(key, value, queue));
    }

    public V get(K key) {
        cleanupStale();
        KeyedSoftReference<K, V> ref = cache.get(key);
        return (ref != null) ? ref.get() : null;
    }

    @SuppressWarnings("unchecked")
    private void cleanupStale() {
        KeyedSoftReference<K, V> ref;
        while ((ref = (KeyedSoftReference<K, V>) queue.poll()) != null) {
            cache.remove(ref.key);
            System.out.println("Evicted soft cache entry for key: " + ref.key);
        }
    }
}
```

### Reference Queue with Phantom References

Phantom references are different — `get()` always returns null, and they're used for post-mortem cleanup:

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

public class PhantomReferenceDemo {

    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        Object obj = new Object();

        PhantomReference<Object> phantomRef = new PhantomReference<>(obj, queue);

        System.out.println("PhantomRef.get(): " + phantomRef.get());  // Always null!

        obj = null;
        System.gc();
        Thread.sleep(100);

        // The reference is enqueued when the object is collected
        if (queue.poll() == phantomRef) {
            System.out.println("Object was collected, phantom reference enqueued");
            // Perform cleanup here (e.g., releasing native resources)
        }
    }
}
```

### Use Cases

**WeakReference + ReferenceQueue:**
- Metadata caches that need cleanup when objects are GC'd
- Observer/listener patterns to remove stale listeners
- Canonical mapping to clean up unused entries

**SoftReference + ReferenceQueue:**
- Memory-sensitive caches with eviction notification
- Resource cleanup when cached data is evicted under memory pressure

**PhantomReference + ReferenceQueue:**
- Releasing native resources (file handles, sockets) when objects are GC'd
- Implementing finalizers without the finalizer problems
- Tracking object lifecycle for debugging

### Common Pitfalls

```java
// ❌ Forgetting to poll the queue
// The queue will fill up and you'll never get notifications
ReferenceQueue<Object> queue = new ReferenceQueue<>();
// Never call queue.poll() or queue.remove()

// ✅ Poll the queue regularly
private void cleanup() {
    Reference<?> ref;
    while ((ref = queue.poll()) != null) {
        // Handle the enqueued reference
    }
}

// ❌ Blocking on queue.remove() without timeout
// This can block indefinitely if GC doesn't collect the object
queue.remove();  // Blocks forever if object not collected

// ✅ Use timeout
queue.remove(1000);  // Wait up to 1 second
```

### Best Practices

```java
// ✅ Always poll the ReferenceQueue regularly
// ✅ Use ReferenceQueue with WeakReference for metadata caches
// ✅ Use ReferenceQueue with SoftReference for cache eviction handling
// ✅ Use ReferenceQueue with PhantomReference for native resource cleanup
// ✅ Clean up the map entries when references are enqueued

// ❌ Don't forget to poll the queue (memory leak)
// ❌ Don't block indefinitely on queue.remove()
// ❌ Don't assume get() returns non-null for PhantomReference (always null)
```

---

## Interview Tips
- ReferenceQueue is the only way to know when a reference's referent has been collected
- PhantomReference.get() always returns null — they're for post-mortem cleanup only
- ReferenceQueue is essential for implementing resource cleanup tied to object lifecycle
- Common pattern: custom reference subclass that stores the key for map cleanup

---

## 📚 Related Topics
- [Weak References](./04-weak-references.md)
- [Soft References](./05-soft-references.md)
- [Memory Leaks](./03-memory-leaks.md)
