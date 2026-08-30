# Q11: How Does ReadWriteLock Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is `ReadWriteLock`? How does `ReentrantReadWriteLock` work and when should you use it? What is `StampedLock`?

---

## Answer

### The Problem: Readers vs Writers

```
With synchronized (or ReentrantLock):
  Reader A ──LOCK──read──UNLOCK──
  Reader B ──────────────────────LOCK──read──UNLOCK──
  Reader C ─────────────────────────────────────────LOCK──read──UNLOCK──
  ❌ Readers block each other even though reads are safe in parallel!

With ReadWriteLock:
  Reader A ──RLOCK──read──RUNLOCK──
  Reader B ──RLOCK──read──RUNLOCK──   ← Concurrent! No blocking!
  Reader C ──RLOCK──read──RUNLOCK──
  Writer X ─────────────────────────WLOCK──write──WUNLOCK──
  ✅ Multiple readers in parallel; writers get exclusive access
```

### ReadWriteLock Interface

```java
public interface ReadWriteLock {
    Lock readLock();   // Shared lock - multiple readers allowed
    Lock writeLock();  // Exclusive lock - only one writer, no readers
}
```

### ReentrantReadWriteLock

```java
public class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple threads can read simultaneously
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public boolean containsKey(K key) {
        readLock.lock();
        try {
            return cache.containsKey(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only one thread can write; blocks all readers
    public V put(K key, V value) {
        writeLock.lock();
        try {
            return cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    public void clear() {
        writeLock.lock();
        try {
            cache.clear();
        } finally {
            writeLock.unlock();
        }
    }

    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
}
```

### Lock Compatibility Matrix

```
┌─────────────────┬────────────┬─────────────┐
│ Requesting \    │ Read Lock  │ Write Lock  │
│ Holding         │ (shared)   │ (exclusive) │
├─────────────────┼────────────┼─────────────┤
│ Read Lock       │  ✅ Allow  │  ❌ Block   │
│ Write Lock      │  ❌ Block  │  ❌ Block   │
└─────────────────┴────────────┴─────────────┘

- Multiple readers: ✅ Allowed (shared access)
- Reader + Writer: ❌ Blocked (mutual exclusion)
- Multiple writers: ❌ Blocked (exclusive access)
```

### Lock Downgrading (Write → Read)

```java
public class LockDowngradeDemo {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private volatile Object cachedData;
    private volatile boolean cacheValid;

    public Object processData() {
        rwLock.readLock().lock();
        try {
            if (!cacheValid) {
                // Must release read lock before acquiring write lock
                rwLock.readLock().unlock();
                rwLock.writeLock().lock();
                try {
                    // Double-check after acquiring write lock
                    if (!cacheValid) {
                        cachedData = loadFromDatabase();
                        cacheValid = true;
                    }
                    // ✅ Downgrade: acquire read lock while holding write lock
                    rwLock.readLock().lock();
                } finally {
                    // Release write lock; still holding read lock
                    rwLock.writeLock().unlock();
                }
            }
            // Use data under read lock
            return cachedData;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // ❌ Lock UPGRADING (Read → Write) is NOT supported!
    // This will DEADLOCK:
    public void brokenUpgrade() {
        rwLock.readLock().lock();
        try {
            // rwLock.writeLock().lock(); // ❌ DEADLOCK! Cannot upgrade
        } finally {
            rwLock.readLock().unlock();
        }
    }

    private Object loadFromDatabase() {
        return "data from DB";
    }
}
```

### StampedLock (Java 8+)

`StampedLock` adds **optimistic reading** for even better performance:

```java
public class StampedLockDemo {
    private final StampedLock stampedLock = new StampedLock();
    private double x, y;

    // Write lock (exclusive)
    public void move(double deltaX, double deltaY) {
        long stamp = stampedLock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            stampedLock.unlockWrite(stamp);
        }
    }

    // ✅ Optimistic read (no locking!) - best case: zero overhead
    public double distanceFromOrigin() {
        long stamp = stampedLock.tryOptimisticRead(); // Non-blocking!
        double currentX = x;
        double currentY = y;

        // Validate: check if a write occurred during our read
        if (!stampedLock.validate(stamp)) {
            // A write happened! Fall back to pessimistic read lock
            stamp = stampedLock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlockRead(stamp);
            }
        }

        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    // Pessimistic read lock (same as ReadWriteLock)
    public double[] getPosition() {
        long stamp = stampedLock.readLock();
        try {
            return new double[]{x, y};
        } finally {
            stampedLock.unlockRead(stamp);
        }
    }

    // Lock conversion: read → write
    public void moveIfAt(double expectedX, double expectedY, double newX, double newY) {
        long stamp = stampedLock.readLock();
        try {
            while (x == expectedX && y == expectedY) {
                // Try to upgrade to write lock
                long writeStamp = stampedLock.tryConvertToWriteLock(stamp);
                if (writeStamp != 0L) {
                    stamp = writeStamp; // Upgrade successful!
                    x = newX;
                    y = newY;
                    break;
                } else {
                    // Upgrade failed - release read, acquire write
                    stampedLock.unlockRead(stamp);
                    stamp = stampedLock.writeLock();
                }
            }
        } finally {
            stampedLock.unlock(stamp);
        }
    }
}
```

### Comparison: ReadWriteLock vs StampedLock

```
┌──────────────────┬─────────────────────┬──────────────────────┐
│ Feature          │ ReentrantRWLock      │ StampedLock          │
├──────────────────┼─────────────────────┼──────────────────────┤
│ Optimistic reads │ ❌                  │ ✅                   │
│ Reentrant        │ ✅                  │ ❌                   │
│ Conditions       │ ✅ (write lock)     │ ❌                   │
│ Fair mode        │ ✅                  │ ❌                   │
│ Lock upgrading   │ ❌                  │ ✅ tryConvertTo..()  │
│ Performance      │ Good                │ Better (read-heavy)  │
│ Complexity       │ Moderate            │ Higher               │
│ Since            │ JDK 1.5             │ JDK 1.8              │
└──────────────────┴─────────────────────┴──────────────────────┘
```

### When to Use What

```java
// Use synchronized: Simple, low-contention cases
public synchronized void simpleUpdate() { /* ... */ }

// Use ReentrantLock: Need tryLock, fairness, conditions
private final ReentrantLock lock = new ReentrantLock(true);

// Use ReentrantReadWriteLock: Read-heavy, write-rare workloads
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// Use StampedLock: Maximum read performance, non-reentrant OK
private final StampedLock stampedLock = new StampedLock();
```

### Practical Example: Configuration Store

```java
public class ConfigStore {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Map<String, String> config = new HashMap<>();
    private volatile long lastModified;

    // Frequent operation - multiple readers allowed
    public String getProperty(String key) {
        rwLock.readLock().lock();
        try {
            return config.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public Map<String, String> getAllProperties() {
        rwLock.readLock().lock();
        try {
            return new HashMap<>(config); // Defensive copy
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // Rare operation - exclusive access
    public void reloadConfig(Map<String, String> newConfig) {
        rwLock.writeLock().lock();
        try {
            config.clear();
            config.putAll(newConfig);
            lastModified = System.currentTimeMillis();
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## Anti-Patterns

❌ **Attempting to upgrade read lock to write lock (ReentrantReadWriteLock):**
```java
readLock.lock();
writeLock.lock(); // ❌ DEADLOCK!
```

❌ **Using ReadWriteLock when writes are frequent:**
```java
// If write ratio > 20%, ReadWriteLock overhead may not be worth it
```

❌ **Using StampedLock reentrantly:**
```java
long stamp = stampedLock.writeLock();
long stamp2 = stampedLock.writeLock(); // ❌ DEADLOCK! Not reentrant
```

## Best Practices

✅ Use `ReadWriteLock` when reads significantly outnumber writes  
✅ Use `StampedLock` with optimistic reads for maximum read throughput  
✅ Remember: lock downgrading (write→read) is allowed; upgrading (read→write) is not  
✅ Always unlock in finally blocks  
✅ Consider `ConcurrentHashMap` before implementing your own read-write cache

---

[← ReentrantLock](./q10-reentrant-lock.md) | [Next: Condition Variables →](./q12-condition-variables.md)
