# Q4: What Are Weak References in Java?

[← Back to Memory Management](./README.md)

---

## Overview

A `WeakReference<T>` is a reference type that does **not** prevent its referent from being garbage collected. An object that is only weakly reachable will be collected at the **next GC cycle**, regardless of available memory.

---

## Reference Strength Hierarchy

```
Strong  >  Soft  >  Weak  >  Phantom
  │          │        │         │
  │          │        │         └── get() always null; for cleanup scheduling
  │          │        └──────────── Collected at next GC
  │          └───────────────────── Collected under memory pressure
  └──────────────────────────────── Never collected while reachable
```

---

## Basic Usage

```java
import java.lang.ref.WeakReference;

public class WeakReferenceDemo {

    public static void main(String[] args) {
        // Strong reference
        Object strongRef = new Object();

        // Weak reference to the same object
        WeakReference<Object> weakRef = new WeakReference<>(strongRef);

        System.out.println(weakRef.get());   // Not null — strong ref exists

        // Remove the strong reference
        strongRef = null;

        // Suggest GC (for demonstration only)
        System.gc();

        System.out.println(weakRef.get());   // Likely null — object was collected
    }
}
```

---

## WeakReference with ReferenceQueue

A `ReferenceQueue` lets you detect when a weakly referenced object has been collected:

```java
import java.lang.ref.WeakReference;
import java.lang.ref.ReferenceQueue;

public class WeakRefQueueDemo {

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

---

## WeakHashMap

`WeakHashMap<K, V>` is a `Map` implementation where **keys** are held via weak references. When a key is garbage collected, the entry is automatically removed.

```java
import java.util.WeakHashMap;
import java.util.Map;

public class WeakHashMapDemo {

    public static void main(String[] args) {
        Map<Object, String> map = new WeakHashMap<>();

        Object key1 = new Object();
        Object key2 = new Object();

        map.put(key1, "Value 1");
        map.put(key2, "Value 2");

        System.out.println("Before GC: " + map.size());   // 2

        key1 = null;   // Remove strong reference to key1
        System.gc();

        // Give GC time to process
        try { Thread.sleep(100); } catch (InterruptedException ignored) {}

        System.out.println("After GC: " + map.size());    // Likely 1
    }
}
```

### ⚠️ Important: String literal keys don't work

```java
// ❌ Anti-pattern: String literals are always strongly referenced from the pool
Map<String, Object> map = new WeakHashMap<>();
map.put("key", someValue);   // "key" is interned — never collected!

// ✅ Use new String() to avoid the string pool (for demonstration)
map.put(new String("key"), someValue);   // This key CAN be collected
```

---

## Use Case 1: Metadata/Annotation Cache

Attach extra data to objects without preventing their collection:

```java
import java.util.Map;
import java.util.WeakHashMap;
import java.util.Collections;

public class MetadataCache {

    // Weak keys: metadata is discarded when the object is GC'd
    private final Map<Object, Map<String, Object>> metadata =
            Collections.synchronizedMap(new WeakHashMap<>());

    public void setMetadata(Object target, String key, Object value) {
        metadata.computeIfAbsent(target, k -> new java.util.HashMap<>())
                .put(key, value);
    }

    public Object getMetadata(Object target, String key) {
        Map<String, Object> attrs = metadata.get(target);
        return attrs != null ? attrs.get(key) : null;
    }

    public int size() {
        return metadata.size();
    }
}
```

```java
// Usage
MetadataCache cache = new MetadataCache();
Object widget = new Object();

cache.setMetadata(widget, "color", "blue");
System.out.println(cache.size());   // 1

widget = null;
System.gc();
Thread.sleep(100);

System.out.println(cache.size());   // 0 — metadata auto-cleaned
```

---

## Use Case 2: Canonical Mapping (Interning)

Ensure only one instance of equivalent objects exists, without preventing collection:

```java
import java.lang.ref.WeakReference;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CanonicalMap<T> {

    private final Map<T, WeakReference<T>> map = new ConcurrentHashMap<>();

    /**
     * Returns the canonical instance equal to the given value.
     * If no canonical instance exists (or was collected), the given value becomes canonical.
     */
    public T canonicalize(T value) {
        WeakReference<T> existingRef = map.get(value);
        if (existingRef != null) {
            T existing = existingRef.get();
            if (existing != null) {
                return existing;   // Return the canonical instance
            }
        }

        // Register this value as canonical
        map.put(value, new WeakReference<>(value));
        return value;
    }

    public int size() {
        return map.size();
    }
}
```

---

## Use Case 3: Observer Pattern Without Leaks

```java
import java.lang.ref.WeakReference;
import java.util.List;
import java.util.Iterator;
import java.util.concurrent.CopyOnWriteArrayList;

public class EventEmitter<T> {

    private final List<WeakReference<EventListener<T>>> listeners =
            new CopyOnWriteArrayList<>();

    public void addListener(EventListener<T> listener) {
        listeners.add(new WeakReference<>(listener));
    }

    public void emit(T event) {
        Iterator<WeakReference<EventListener<T>>> it = listeners.iterator();
        while (it.hasNext()) {
            WeakReference<EventListener<T>> ref = it.next();
            EventListener<T> listener = ref.get();
            if (listener != null) {
                listener.onEvent(event);
            }
            // Note: CopyOnWriteArrayList doesn't support iterator.remove()
            // Cleanup stale refs periodically or on emit
        }
    }

    public void cleanupStale() {
        listeners.removeIf(ref -> ref.get() == null);
    }
}

@FunctionalInterface
interface EventListener<T> {
    void onEvent(T event);
}
```

---

## Weak vs Strong Reference Comparison

| Aspect              | Strong Reference       | WeakReference                        |
|---------------------|------------------------|--------------------------------------|
| **Prevents GC**     | Yes                    | No                                   |
| **Collected when**  | Never (while reachable)| Next GC cycle                        |
| **get() returns**   | Always the object      | Object or null                       |
| **Use case**        | Normal references      | Caches, metadata, observer patterns  |
| **Thread-safety**   | N/A                    | get() is atomic; map access isn't    |

---

## Common Pitfalls

```java
// ❌ Anti-pattern: creating a weak reference while also holding a strong reference
Object obj = new Object();
WeakReference<Object> weak = new WeakReference<>(obj);
// 'obj' still holds a strong reference — weak ref has no effect yet

// ❌ Anti-pattern: assuming get() returns the same result across calls
WeakReference<Data> ref = new WeakReference<>(data);
if (ref.get() != null) {
    ref.get().process();   // TOCTOU! May be null by now
}

// ✅ Correct: save the result of get()
Data d = ref.get();
if (d != null) {
    d.process();   // Safe — local variable holds strong reference
}
```

---

## Best Practices

```java
// ✅ Always null-check the result of get()
// ✅ Use WeakHashMap for automatic key-based cleanup
// ✅ Use ReferenceQueue to detect and react to collection
// ✅ Prefer WeakReference for metadata caches and listener patterns
// ✅ Hold the get() result in a local variable to prevent race conditions

// ❌ Don't use WeakReference as a general-purpose cache (use SoftReference instead)
// ❌ Don't use String literals as WeakHashMap keys
// ❌ Don't assume get() is stable across calls
```

---

## Interview Tips
- WeakReferences are collected eagerly — at the **next** GC cycle, not under memory pressure (that's SoftReference)
- `WeakHashMap` is the most common practical use of weak references in the JDK
- ClassLoader uses weak references internally to allow class unloading
- `ThreadLocal` entries use weak references to the `ThreadLocal` key (but values are strong — hence the leak risk)

---

[← Q3: Memory Leaks](./q03-memory-leaks.md) | [Next: Q5 - Soft References →](./q05-soft-references.md)
