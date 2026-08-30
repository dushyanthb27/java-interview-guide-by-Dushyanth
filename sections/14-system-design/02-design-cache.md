# Q2: Design a Cache (LRU)

## Question
Design and implement an LRU (Least Recently Used) Cache in Java. Discuss the data structures, concurrency considerations, and optimization strategies.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- `get(key)` — retrieve a value by key in O(1); marks entry as recently used
- `put(key, value)` — insert or update a key-value pair in O(1)
- Automatic eviction of the least recently used entry when capacity is exceeded
- Support for TTL (time-to-live) per entry
- Bounded maximum size

**Non-Functional Requirements:**
- Thread-safe for concurrent reads and writes
- O(1) time complexity for both get and put
- Minimal memory overhead
- Support for multiple eviction policies (LRU, LFU, FIFO)

### API Design

```java
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    void put(K key, V value, long ttlMillis);
    V remove(K key);
    int size();
    void clear();
    boolean containsKey(K key);
}
```

### Implementation

#### Core Data Structure: HashMap + Doubly Linked List

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class LRUCache<K, V> implements Cache<K, V> {

    // --- Doubly Linked List Node ---
    private static class Node<K, V> {
        K key;
        V value;
        long expiryTime; // 0 means no TTL
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.expiryTime = 0;
        }

        Node(K key, V value, long ttlMillis) {
            this.key = key;
            this.value = value;
            this.expiryTime = (ttlMillis > 0) ? System.currentTimeMillis() + ttlMillis : 0;
        }

        boolean isExpired() {
            return expiryTime > 0 && System.currentTimeMillis() > expiryTime;
        }
    }

    private final int capacity;
    private final ConcurrentHashMap<K, Node<K, V>> map;
    private final Node<K, V> head; // dummy head (most recent)
    private final Node<K, V> tail; // dummy tail (least recent)
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public LRUCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be positive");
        this.capacity = capacity;
        this.map = new ConcurrentHashMap<>(capacity);

        // Sentinel nodes simplify edge cases
        head = new Node<>(null, null);
        tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    @Override
    public V get(K key) {
        lock.writeLock().lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;

            // Check TTL
            if (node.isExpired()) {
                removeNode(node);
                map.remove(key);
                return null;
            }

            // Move to front (most recently used)
            moveToHead(node);
            return node.value;
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public void put(K key, V value) {
        put(key, value, 0);
    }

    @Override
    public void put(K key, V value, long ttlMillis) {
        lock.writeLock().lock();
        try {
            Node<K, V> existing = map.get(key);

            if (existing != null) {
                // Update existing entry
                existing.value = value;
                existing.expiryTime = (ttlMillis > 0)
                    ? System.currentTimeMillis() + ttlMillis : 0;
                moveToHead(existing);
            } else {
                // Add new entry
                Node<K, V> newNode = new Node<>(key, value, ttlMillis);
                map.put(key, newNode);
                addToHead(newNode);

                // Evict if over capacity
                if (map.size() > capacity) {
                    Node<K, V> lru = removeTail();
                    if (lru != null) {
                        map.remove(lru.key);
                    }
                }
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public V remove(K key) {
        lock.writeLock().lock();
        try {
            Node<K, V> node = map.remove(key);
            if (node != null) {
                removeNode(node);
                return node.value;
            }
            return null;
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public int size() {
        lock.readLock().lock();
        try {
            return map.size();
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public void clear() {
        lock.writeLock().lock();
        try {
            map.clear();
            head.next = tail;
            tail.prev = head;
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public boolean containsKey(K key) {
        lock.readLock().lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return false;
            return !node.isExpired();
        } finally {
            lock.readLock().unlock();
        }
    }

    // --- Linked List helpers ---

    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev;
        if (lru == head) return null; // empty list
        removeNode(lru);
        return lru;
    }
}
```

#### Alternative: LinkedHashMap Approach

```java
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

public class LinkedHashMapLRUCache<K, V> extends LinkedHashMap<K, V> {

    private final int capacity;

    public LinkedHashMapLRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true enables LRU
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }

    // Thread-safe wrapper
    public static <K, V> Map<K, V> createThreadSafe(int capacity) {
        return Collections.synchronizedMap(new LinkedHashMapLRUCache<>(capacity));
    }
}
```

#### Eviction Policy Abstraction

```java
public enum EvictionPolicy { LRU, LFU, FIFO }

public interface EvictionStrategy<K> {
    void onAccess(K key);
    void onInsert(K key);
    K evict();     // returns the key to evict
}

public class LRUEviction<K> implements EvictionStrategy<K> {
    private final LinkedList<K> order = new LinkedList<>();

    @Override
    public void onAccess(K key) {
        order.remove(key);
        order.addFirst(key);
    }

    @Override
    public void onInsert(K key) {
        order.addFirst(key);
    }

    @Override
    public K evict() {
        return order.removeLast();
    }
}

public class FIFOEviction<K> implements EvictionStrategy<K> {
    private final Queue<K> order = new LinkedList<>();

    @Override public void onAccess(K key) { /* no-op for FIFO */ }
    @Override public void onInsert(K key) { order.offer(key); }
    @Override public K evict() { return order.poll(); }
}
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Concurrent reads/writes | `ReentrantReadWriteLock` — allows concurrent reads, exclusive writes |
| HashMap thread safety | `ConcurrentHashMap` as the backing store |
| Linked list consistency | Write lock held during all list mutations |
| TTL expiry | Lazy check on `get()` — avoids background cleanup overhead |
| Stale entries | Could add a scheduled cleanup thread for proactive eviction |

**Why not just `ConcurrentHashMap` alone?**
`ConcurrentHashMap` doesn't maintain access order. The doubly linked list tracks recency, but it requires exclusive access during mutations — hence the write lock.

### Performance Analysis

| Operation | Time | Space |
|-----------|------|-------|
| `get()` | O(1) | — |
| `put()` | O(1) | — |
| `remove()` | O(1) | — |
| Overall space | — | O(n) where n = capacity |

**Bottlenecks:**
- Write lock serializes all writes — for extremely high-throughput scenarios, consider striped locking (partition the cache into segments, each with its own lock)
- TTL cleanup is lazy — expired entries may linger until accessed

### Testing Strategy

```java
public class LRUCacheTest {

    @Test
    void testBasicGetPut() {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3);

        assertEquals(1, cache.get("a"));
        assertEquals(3, cache.size());
    }

    @Test
    void testEviction() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3); // evicts "a"

        assertNull(cache.get("a"));
        assertEquals(2, cache.get("b"));
        assertEquals(3, cache.get("c"));
    }

    @Test
    void testLRUOrdering() {
        LRUCache<String, Integer> cache = new LRUCache<>(2);
        cache.put("a", 1);
        cache.put("b", 2);
        cache.get("a");        // "a" is now most recent
        cache.put("c", 3);     // evicts "b", not "a"

        assertNull(cache.get("b"));
        assertEquals(1, cache.get("a"));
    }

    @Test
    void testTTLExpiry() throws InterruptedException {
        LRUCache<String, Integer> cache = new LRUCache<>(10);
        cache.put("temp", 42, 200); // 200ms TTL

        assertEquals(42, cache.get("temp"));
        Thread.sleep(300);
        assertNull(cache.get("temp")); // expired
    }

    @Test
    void testConcurrentAccess() throws Exception {
        LRUCache<Integer, Integer> cache = new LRUCache<>(1000);
        ExecutorService executor = Executors.newFixedThreadPool(8);

        for (int i = 0; i < 10000; i++) {
            final int key = i;
            executor.submit(() -> {
                cache.put(key, key * 2);
                cache.get(key);
            });
        }

        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        assertTrue(cache.size() <= 1000);
    }
}
```

---

## Interviewer Insight
Interviewers want to see O(1) get and put using a HashMap + Doubly Linked List. They check whether you understand why a linked list is needed (to track recency order) and why `LinkedHashMap` alone is often sufficient. Thread safety and the trade-off between `synchronized`, `ReadWriteLock`, and striped locking are important discussion points.

## Follow-up Questions
1. How would you implement an LFU (Least Frequently Used) cache?
2. How would you partition the cache across multiple segments to reduce lock contention?
3. How does Guava's `CacheBuilder` or Caffeine handle eviction and concurrency internally?

## Related Topics
- [Collections Framework](../03-collections-framework/README.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
