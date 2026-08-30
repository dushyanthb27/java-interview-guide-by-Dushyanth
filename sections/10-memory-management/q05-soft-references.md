# Q5: What Are Soft References in Java?

[← Back to Memory Management](./README.md)

---

## Overview

A `SoftReference<T>` is a reference type that allows the GC to collect its referent only when the JVM is **running low on memory**. This makes soft references ideal for implementing **memory-sensitive caches** — data stays cached as long as memory permits, and is evicted gracefully under pressure.

---

## How Soft References Differ from Weak References

```
                   Memory Available        Memory Low
                 ┌──────────────────┐  ┌──────────────────┐
SoftReference    │  Kept alive ✅    │  │  Collected ♻️     │
                 └──────────────────┘  └──────────────────┘
WeakReference    │  Collected ♻️      │  │  Collected ♻️     │
                 └──────────────────┘  └──────────────────┘
```

| Aspect              | SoftReference              | WeakReference                  |
|---------------------|----------------------------|--------------------------------|
| **Collected when**  | Under memory pressure      | At next GC cycle               |
| **Best for**        | Memory-sensitive caches    | Metadata, canonical maps       |
| **Lifetime**        | Potentially long           | Very short                     |
| **GC guarantee**    | Cleared before OOM thrown  | Cleared eagerly                |

---

## Basic Usage

```java
import java.lang.ref.SoftReference;

public class SoftReferenceDemo {

    public static void main(String[] args) {
        // Create a large object and wrap it in a soft reference
        byte[] largeData = new byte[10_000_000];   // 10MB
        SoftReference<byte[]> softRef = new SoftReference<>(largeData);

        largeData = null;   // Remove strong reference

        // Object is still accessible (memory isn't under pressure)
        System.out.println("Data available: " + (softRef.get() != null));   // true

        // Under memory pressure, the GC will clear this soft reference
        // before throwing OutOfMemoryError
    }
}
```

---

## Memory-Sensitive Cache Implementation

```java
import java.lang.ref.SoftReference;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;

public class SoftCache<K, V> {

    private final Map<K, SoftReference<V>> cache = new ConcurrentHashMap<>();

    /**
     * Gets a value from the cache, computing it if absent or collected.
     */
    public V get(K key, Function<K, V> loader) {
        SoftReference<V> ref = cache.get(key);
        V value = (ref != null) ? ref.get() : null;

        if (value == null) {
            value = loader.apply(key);   // Recompute on cache miss
            cache.put(key, new SoftReference<>(value));
        }

        return value;
    }

    /**
     * Explicitly put a value into the cache.
     */
    public void put(K key, V value) {
        cache.put(key, new SoftReference<>(value));
    }

    /**
     * Remove stale entries whose referents have been collected.
     */
    public void cleanupStale() {
        cache.entrySet().removeIf(entry -> entry.getValue().get() == null);
    }

    public int size() {
        return cache.size();
    }

    public int activeSize() {
        return (int) cache.values().stream()
                .filter(ref -> ref.get() != null)
                .count();
    }
}
```

```java
// Usage: image cache
SoftCache<String, byte[]> imageCache = new SoftCache<>();

byte[] image = imageCache.get("banner.png", path -> loadImageFromDisk(path));
// Image stays cached until memory is needed
```

---

## Soft References with ReferenceQueue

Use a `ReferenceQueue` to detect and react to evictions:

```java
import java.lang.ref.SoftReference;
import java.lang.ref.ReferenceQueue;
import java.util.Map;
import java.util.HashMap;

public class TrackedSoftCache<K, V> {

    private final Map<K, KeyedSoftReference<K, V>> cache = new HashMap<>();
    private final ReferenceQueue<V> queue = new ReferenceQueue<>();

    /**
     * Custom SoftReference that remembers its key for cleanup.
     */
    static class KeyedSoftReference<K, V> extends SoftReference<V> {
        final K key;

        KeyedSoftReference(K key, V value, ReferenceQueue<V> queue) {
            super(value, queue);
            this.key = key;
        }
    }

    public void put(K key, V value) {
        cleanupStale();   // Purge collected entries
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

---

## LRU Behavior of Soft References

The JVM considers the **time since last access** when deciding which soft references to clear. The formula used by HotSpot:

```
interval (ms) = free_heap_MB * SoftRefLRUPolicyMSPerMB
```

- Default `SoftRefLRUPolicyMSPerMB` = **1000** (1 second per MB of free heap)
- Recently accessed soft references survive longer
- Least-recently accessed are cleared first

### JVM tuning

```bash
# Soft references live longer (more aggressive caching)
java -XX:SoftRefLRUPolicyMSPerMB=2000 MyApp

# Soft references cleared more aggressively (less memory usage)
java -XX:SoftRefLRUPolicyMSPerMB=0 MyApp

# With 500MB free heap and default setting:
# A soft reference last accessed 500 seconds ago MAY be cleared
# A soft reference last accessed 1 second ago will likely survive
```

---

## Practical Example: Document Cache

```java
import java.lang.ref.SoftReference;
import java.util.concurrent.ConcurrentHashMap;

public class DocumentCache {

    private final ConcurrentHashMap<String, SoftReference<Document>> cache =
            new ConcurrentHashMap<>();

    private final DocumentRepository repository;

    public DocumentCache(DocumentRepository repository) {
        this.repository = repository;
    }

    public Document getDocument(String id) {
        // Try cache first
        SoftReference<Document> ref = cache.get(id);
        if (ref != null) {
            Document doc = ref.get();
            if (doc != null) {
                System.out.println("Cache hit for: " + id);
                return doc;
            }
        }

        // Cache miss — load from repository
        System.out.println("Cache miss for: " + id);
        Document doc = repository.findById(id);

        if (doc != null) {
            cache.put(id, new SoftReference<>(doc));
        }

        return doc;
    }

    public void invalidate(String id) {
        cache.remove(id);
    }

    public void invalidateAll() {
        cache.clear();
    }
}
```

---

## When NOT to Use Soft References

```java
// ❌ Anti-pattern: using SoftReference for small, cheap-to-create objects
SoftReference<String> name = new SoftReference<>(computeName());
// Overhead of SoftReference object > savings from caching a small String

// ❌ Anti-pattern: relying on SoftReference for correctness
SoftReference<Config> config = new SoftReference<>(loadConfig());
Config c = config.get();
if (c == null) {
    // If your app breaks because the config was collected, don't use SoftReference!
    throw new IllegalStateException("Config was garbage collected!");
}

// ❌ Anti-pattern: soft reference in a tight loop
for (int i = 0; i < 1_000_000; i++) {
    SoftReference<byte[]> ref = new SoftReference<>(new byte[1024]);
    // Creates garbage rapidly; GC thrashes clearing soft refs
}
```

---

## Soft References vs Guava/Caffeine Caches

Modern cache libraries are generally preferred over manual soft references:

```java
// ✅ Using Caffeine (recommended for production)
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

Cache<String, Document> cache = Caffeine.newBuilder()
        .maximumSize(1_000)
        .softValues()                         // Uses soft references for values
        .build();

Document doc = cache.get("doc-123", id -> repository.findById(id));
```

```java
// Comparison table
// ┌────────────────────────┬──────────────────┬────────────────────┐
// │ Feature                │ Manual SoftRef   │ Caffeine/Guava     │
// ├────────────────────────┼──────────────────┼────────────────────┤
// │ Eviction control       │ JVM decides      │ Size/time/weight   │
// │ Statistics             │ None             │ Hit rate, evictions│
// │ Thread safety          │ Manual           │ Built-in           │
// │ Cleanup                │ Manual           │ Automatic          │
// │ Loading                │ Manual           │ CacheLoader        │
// │ Removal listener       │ ReferenceQueue   │ RemovalListener    │
// └────────────────────────┴──────────────────┴────────────────────┘
```

---

## Best Practices

```java
// ✅ Use SoftReference for expensive-to-compute, reproducible data
// ✅ Always check get() for null — the referent may have been collected
// ✅ Use ReferenceQueue to clean up associated resources
// ✅ Prefer Caffeine or Guava Cache for production cache implementations
// ✅ Tune SoftRefLRUPolicyMSPerMB based on your application's needs

// ❌ Don't use SoftReference for essential application state
// ❌ Don't use SoftReference for small or cheap objects
// ❌ Don't create soft references in tight loops
// ❌ Don't forget that all soft references are cleared before OOM is thrown
```

---

## Interview Tips
- Soft references are the JVM's way of saying "keep this if you can, discard if you must"
- The GC guarantees all soft references are cleared **before** throwing `OutOfMemoryError`
- Soft references have LRU-like behavior — recently accessed ones survive longer
- In practice, cache libraries (Caffeine) are preferred over raw `SoftReference`
- `SoftRefLRUPolicyMSPerMB` controls the aggressiveness of soft reference eviction

---

[← Q4: Weak References](./q04-weak-references.md) | [Next: Q6 - Phantom References →](./q06-phantom-references.md)
