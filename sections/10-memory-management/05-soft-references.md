# Q5: What Are Soft References in Java?

## 📋 Question
What are Soft References in Java? When should you use them, and how do they differ from Weak References?

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

A `SoftReference<T>` is a reference type that allows the GC to collect its referent only when the JVM is **running low on memory**. This makes soft references ideal for implementing **memory-sensitive caches** — data stays cached as long as memory permits, and is evicted gracefully under pressure.

### How Soft References Differ from Weak References

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

### Basic Usage

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

### Memory-Sensitive Cache Implementation

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

### LRU Behavior of Soft References

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
```

### When NOT to Use Soft References

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
```

### Best Practices

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

## 📚 Related Topics
- [Weak References](./04-weak-references.md)
- [Memory Leaks](./03-memory-leaks.md)
- [Object Lifecycle](./02-object-lifecycle.md)
