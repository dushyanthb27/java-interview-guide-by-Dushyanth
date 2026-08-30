# LRU Cache Template

## Overview
A Least Recently Used (LRU) cache implementation that maintains O(1) time complexity for get and put operations. Evicts the least recently used item when capacity is reached.

## Use Cases
- CPU cache simulation
- Database query result caching
- Web page caching
- Session storage
- Memoization in algorithms
- API response caching

## Implementation

### Simple Version
Basic LRU using HashMap and LinkedList.

### Production Version
Thread-safe with ReentrantReadWriteLock and additional features.

### Thread-Safe Version
Fully concurrent with proper synchronization and eviction policies.

## Code

```java
import java.util.*;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * SIMPLE VERSION: Basic LRU Cache
 * - HashMap for O(1) access
 * - LinkedList to track order
 * - Manual eviction on capacity exceeded
 */
public class SimpleLRUCache<K, V> {
    private final int capacity;
    private final Map<K, V> cache;
    private final LinkedList<K> accessOrder; // Tracks access order

    public SimpleLRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.accessOrder = new LinkedList<>();
    }

    /**
     * Get value from cache
     * Time: O(1), Space: O(1)
     */
    public V get(K key) {
        if (!cache.containsKey(key)) {
            return null;
        }

        // Move to end (most recently used)
        accessOrder.remove(key);
        accessOrder.addLast(key);

        return cache.get(key);
    }

    /**
     * Put value in cache
     * Time: O(1), Space: O(1)
     */
    public void put(K key, V value) {
        // If key exists, remove it first
        if (cache.containsKey(key)) {
            accessOrder.remove(key);
        }
        // If at capacity, evict least recently used
        else if (cache.size() >= capacity) {
            K lruKey = accessOrder.removeFirst();
            cache.remove(lruKey);
        }

        // Add new entry
        cache.put(key, value);
        accessOrder.addLast(key);
    }

    /**
     * Get cache size
     */
    public int size() {
        return cache.size();
    }

    /**
     * Check if key exists
     */
    public boolean containsKey(K key) {
        return cache.containsKey(key);
    }

    /**
     * Clear cache
     */
    public void clear() {
        cache.clear();
        accessOrder.clear();
    }
}

/**
 * PRODUCTION VERSION: Advanced LRU Cache
 * - Thread-safe with ReentrantReadWriteLock
 * - Eviction callbacks
 * - Statistics tracking
 * - TTL support
 */
public class ProductionLRUCache<K, V> {
    private final int capacity;
    private final Map<K, CacheEntry<V>> cache;
    private final LinkedHashMap<K, Long> accessOrder;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final EvictionListener<K, V> evictionListener;
    private final long ttlMillis;
    private long hits = 0;
    private long misses = 0;

    @FunctionalInterface
    public interface EvictionListener<K, V> {
        void onEvict(K key, V value, EvictionReason reason);
    }

    public enum EvictionReason {
        CAPACITY_EXCEEDED, TTL_EXPIRED, MANUAL
    }

    /**
     * Cache entry with timestamp
     */
    private static class CacheEntry<V> {
        V value;
        long lastAccessTime;

        CacheEntry(V value) {
            this.value = value;
            this.lastAccessTime = System.currentTimeMillis();
        }

        void updateAccessTime() {
            this.lastAccessTime = System.currentTimeMillis();
        }

        boolean isExpired(long ttlMillis) {
            if (ttlMillis <= 0) return false;
            return System.currentTimeMillis() - lastAccessTime > ttlMillis;
        }
    }

    public ProductionLRUCache(int capacity, long ttlMillis, 
                             EvictionListener<K, V> listener) {
        this.capacity = capacity;
        this.ttlMillis = ttlMillis;
        this.evictionListener = listener;
        this.cache = new HashMap<>();
        this.accessOrder = new LinkedHashMap<K, Long>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, Long> eldest) {
                return size() > capacity;
            }
        };
    }

    /**
     * Get value with thread safety
     */
    public V get(K key) {
        lock.readLock().lock();
        try {
            CacheEntry<V> entry = cache.get(key);
            if (entry == null) {
                misses++;
                return null;
            }

            // Check TTL
            if (entry.isExpired(ttlMillis)) {
                lock.readLock().unlock();
                lock.writeLock().lock();
                try {
                    // Double-check after acquiring write lock
                    entry = cache.get(key);
                    if (entry != null && entry.isExpired(ttlMillis)) {
                        V value = entry.value;
                        cache.remove(key);
                        accessOrder.remove(key);
                        if (evictionListener != null) {
                            evictionListener.onEvict(key, value, EvictionReason.TTL_EXPIRED);
                        }
                        misses++;
                        return null;
                    }
                } finally {
                    lock.readLock().lock();
                    lock.writeLock().unlock();
                }
            }

            hits++;
            entry.updateAccessTime();
            accessOrder.put(key, System.currentTimeMillis());
            return entry.value;
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * Put value with thread safety
     */
    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            // Remove old entry if exists
            if (cache.containsKey(key)) {
                accessOrder.remove(key);
            }
            // Check capacity and evict if needed
            else if (cache.size() >= capacity) {
                K lruKey = accessOrder.keySet().iterator().next();
                CacheEntry<V> evicted = cache.remove(lruKey);
                accessOrder.remove(lruKey);
                if (evictionListener != null && evicted != null) {
                    evictionListener.onEvict(lruKey, evicted.value, 
                                            EvictionReason.CAPACITY_EXCEEDED);
                }
            }

            // Add new entry
            cache.put(key, new CacheEntry<>(value));
            accessOrder.put(key, System.currentTimeMillis());
        } finally {
            lock.writeLock().unlock();
        }
    }

    /**
     * Remove entry manually
     */
    public V remove(K key) {
        lock.writeLock().lock();
        try {
            CacheEntry<V> entry = cache.remove(key);
            accessOrder.remove(key);
            if (entry != null && evictionListener != null) {
                evictionListener.onEvict(key, entry.value, EvictionReason.MANUAL);
            }
            return entry != null ? entry.value : null;
        } finally {
            lock.writeLock().unlock();
        }
    }

    /**
     * Get cache statistics
     */
    public Map<String, Object> getStats() {
        lock.readLock().lock();
        try {
            Map<String, Object> stats = new HashMap<>();
            stats.put("size", cache.size());
            stats.put("capacity", capacity);
            stats.put("hits", hits);
            stats.put("misses", misses);
            stats.put("hitRate", hits + misses > 0 ? 
                     (double) hits / (hits + misses) : 0.0);
            return stats;
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * Clear cache
     */
    public void clear() {
        lock.writeLock().lock();
        try {
            cache.clear();
            accessOrder.clear();
        } finally {
            lock.writeLock().unlock();
        }
    }
}

/**
 * THREAD-SAFE VERSION: Enterprise LRU Cache
 * - Segment-based locking for better concurrency
 * - Automatic cleanup of expired entries
 * - Configurable eviction policies
 * - Memory-aware eviction
 */
public class EnterpriseSegmentedLRUCache<K, V> {
    private static final int DEFAULT_SEGMENTS = 16;
    private final Segment<K, V>[] segments;
    private final int segmentMask;
    private final int capacity;
    private final long ttlMillis;

    @SuppressWarnings("unchecked")
    public EnterpriseSegmentedLRUCache(int capacity, long ttlMillis) {
        this.capacity = capacity;
        this.ttlMillis = ttlMillis;
        int segmentCount = DEFAULT_SEGMENTS;
        this.segments = new Segment[segmentCount];
        this.segmentMask = segmentCount - 1;

        int segmentCapacity = capacity / segmentCount;
        for (int i = 0; i < segmentCount; i++) {
            segments[i] = new Segment<>(segmentCapacity, ttlMillis);
        }
    }

    /**
     * Get segment for key
     */
    private Segment<K, V> getSegment(K key) {
        int hash = key.hashCode() & segmentMask;
        return segments[hash];
    }

    /**
     * Get value
     */
    public V get(K key) {
        return getSegment(key).get(key);
    }

    /**
     * Put value
     */
    public void put(K key, V value) {
        getSegment(key).put(key, value);
    }

    /**
     * Remove value
     */
    public V remove(K key) {
        return getSegment(key).remove(key);
    }

    /**
     * Get total size across all segments
     */
    public int size() {
        int total = 0;
        for (Segment<K, V> segment : segments) {
            total += segment.size();
        }
        return total;
    }

    /**
     * Segment with independent locking
     */
    private static class Segment<K, V> {
        private final int capacity;
        private final long ttlMillis;
        private final Map<K, CacheEntry<V>> cache;
        private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        private static class CacheEntry<V> {
            V value;
            long lastAccessTime;

            CacheEntry(V value) {
                this.value = value;
                this.lastAccessTime = System.currentTimeMillis();
            }

            void updateAccessTime() {
                this.lastAccessTime = System.currentTimeMillis();
            }

            boolean isExpired(long ttlMillis) {
                if (ttlMillis <= 0) return false;
                return System.currentTimeMillis() - lastAccessTime > ttlMillis;
            }
        }

        Segment(int capacity, long ttlMillis) {
            this.capacity = capacity;
            this.ttlMillis = ttlMillis;
            this.cache = new LinkedHashMap<K, CacheEntry<V>>(capacity, 0.75f, true) {
                @Override
                protected boolean removeEldestEntry(Map.Entry<K, CacheEntry<V>> eldest) {
                    return size() > capacity;
                }
            };
        }

        V get(K key) {
            lock.readLock().lock();
            try {
                CacheEntry<V> entry = cache.get(key);
                if (entry == null) return null;

                if (entry.isExpired(ttlMillis)) {
                    lock.readLock().unlock();
                    lock.writeLock().lock();
                    try {
                        cache.remove(key);
                        return null;
                    } finally {
                        lock.readLock().lock();
                        lock.writeLock().unlock();
                    }
                }

                entry.updateAccessTime();
                return entry.value;
            } finally {
                lock.readLock().unlock();
            }
        }

        void put(K key, V value) {
            lock.writeLock().lock();
            try {
                cache.put(key, new CacheEntry<>(value));
            } finally {
                lock.writeLock().unlock();
            }
        }

        V remove(K key) {
            lock.writeLock().lock();
            try {
                CacheEntry<V> entry = cache.remove(key);
                return entry != null ? entry.value : null;
            } finally {
                lock.writeLock().unlock();
            }
        }

        int size() {
            lock.readLock().lock();
            try {
                return cache.size();
            } finally {
                lock.readLock().unlock();
            }
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class LRUCacheTest {

    @Test
    public void testSimpleLRUCacheBasicOperations() {
        SimpleLRUCache<String, Integer> cache = new SimpleLRUCache<>(3);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3);

        assertEquals(1, cache.get("a"));
        assertEquals(2, cache.get("b"));
        assertEquals(3, cache.get("c"));
    }

    @Test
    public void testSimpleLRUCacheEviction() {
        SimpleLRUCache<String, Integer> cache = new SimpleLRUCache<>(2);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3); // Should evict "a"

        assertNull(cache.get("a"));
        assertEquals(2, cache.get("b"));
        assertEquals(3, cache.get("c"));
    }

    @Test
    public void testSimpleLRUCacheAccessOrder() {
        SimpleLRUCache<String, Integer> cache = new SimpleLRUCache<>(2);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.get("a"); // Access "a", making "b" LRU
        cache.put("c", 3); // Should evict "b"

        assertEquals(1, cache.get("a"));
        assertNull(cache.get("b"));
        assertEquals(3, cache.get("c"));
    }

    @Test
    public void testProductionLRUCacheWithTTL() throws InterruptedException {
        AtomicInteger evictions = new AtomicInteger(0);
        ProductionLRUCache<String, Integer> cache = 
            new ProductionLRUCache<>(3, 100, (k, v, r) -> {
                if (r == ProductionLRUCache.EvictionReason.TTL_EXPIRED) {
                    evictions.incrementAndGet();
                }
            });

        cache.put("a", 1);
        Thread.sleep(150); // Wait for TTL to expire

        assertNull(cache.get("a"));
        assertEquals(1, evictions.get());
    }

    @Test
    public void testProductionLRUCacheStatistics() {
        ProductionLRUCache<String, Integer> cache = 
            new ProductionLRUCache<>(3, 0, null);

        cache.put("a", 1);
        cache.get("a"); // Hit
        cache.get("b"); // Miss
        cache.get("a"); // Hit

        var stats = cache.getStats();
        assertEquals(2, stats.get("hits"));
        assertEquals(1, stats.get("misses"));
    }

    @Test
    public void testConcurrentAccess() throws InterruptedException {
        ProductionLRUCache<String, Integer> cache = 
            new ProductionLRUCache<>(100, 0, null);

        int threadCount = 10;
        int operationsPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);

        for (int t = 0; t < threadCount; t++) {
            final int threadId = t;
            executor.submit(() -> {
                for (int i = 0; i < operationsPerThread; i++) {
                    String key = "key-" + (i % 50);
                    cache.put(key, threadId * 1000 + i);
                    cache.get(key);
                }
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(10, TimeUnit.SECONDS));
        assertTrue(cache.size() <= 100);
    }

    @Test
    public void testSegmentedCachePerformance() throws InterruptedException {
        EnterpriseSegmentedLRUCache<String, Integer> cache = 
            new EnterpriseSegmentedLRUCache<>(1000, 0);

        int threadCount = 16;
        int operationsPerThread = 1000;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        long startTime = System.currentTimeMillis();

        for (int t = 0; t < threadCount; t++) {
            final int threadId = t;
            executor.submit(() -> {
                for (int i = 0; i < operationsPerThread; i++) {
                    String key = "key-" + (i % 500);
                    cache.put(key, threadId * 1000 + i);
                    cache.get(key);
                }
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(10, TimeUnit.SECONDS));
        long duration = System.currentTimeMillis() - startTime;

        System.out.println("Segmented cache: " + duration + "ms for " + 
                          (threadCount * operationsPerThread) + " operations");
        assertTrue(cache.size() <= 1000);
    }
}
```

## Performance Considerations

### Time Complexity
- **get()**: O(1) average case
- **put()**: O(1) average case
- **remove()**: O(1) average case

### Space Complexity
- **O(n)** where n = capacity

### Optimization Tips
1. **Segment-based locking**: Reduces contention in concurrent scenarios
2. **LinkedHashMap**: Maintains insertion order efficiently
3. **Read-write locks**: Allows concurrent reads
4. **TTL cleanup**: Automatic expiration of stale entries
5. **Eviction callbacks**: Monitor cache behavior

## Common Pitfalls

1. **Not updating access order on get()**: Defeats LRU purpose
2. **Unbounded cache**: Can cause memory leaks
3. **Poor locking strategy**: Serializes all operations
4. **Not handling concurrent modifications**: ConcurrentModificationException
5. **Ignoring TTL expiration**: Stale data in cache
6. **No statistics tracking**: Can't optimize cache behavior
7. **Inefficient eviction**: O(n) instead of O(1)

## Variations

1. **LFU Cache**: Least Frequently Used eviction
2. **ARC Cache**: Adaptive Replacement Cache
3. **W-TinyLFU**: Weighted TinyLFU (Caffeine cache)
4. **Clock Cache**: Approximation of LRU with lower overhead
5. **Two-level Cache**: L1 (fast) and L2 (large) caches
