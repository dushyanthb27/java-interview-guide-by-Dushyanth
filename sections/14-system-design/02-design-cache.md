# Q2: Design a Distributed Cache

## 📋 Question
Design a distributed cache system. Discuss the data structures, consistency models, and eviction strategies.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Get and Put operations with TTL support
- Distributed consistency across nodes
- Cache invalidation and eviction
- High availability and fault tolerance

**Non-Functional Requirements:**
- Low latency (< 10ms for 99th percentile)
- High throughput (100K+ ops/sec)
- Horizontal scalability
- Data consistency guarantees

### Architecture Design

```
┌─────────────────────────────────────────────────────────┐
│                   Client Application                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Load Balancer (Consistent Hash)        │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Cache Node 1 │ │  Cache Node 2 │ │  Cache Node 3 │
│  (LRU Cache)  │ │  (LRU Cache)  │ │  (LRU Cache)  │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Backend Database (PostgreSQL)               │
└─────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Consistent Hashing

```java
public class ConsistentHashing {
    private final TreeMap<Integer, CacheNode> ring = new TreeMap<>();
    private final int virtualNodes = 150;

    public void addNode(CacheNode node) {
        for (int i = 0; i < virtualNodes; i++) {
            int hash = hash(node.getId() + ":" + i);
            ring.put(hash, node);
        }
    }

    public void removeNode(CacheNode node) {
        for (int i = 0; i < virtualNodes; i++) {
            int hash = hash(node.getId() + ":" + i);
            ring.remove(hash);
        }
    }

    public CacheNode getNode(String key) {
        int hash = hash(key);
        Map.Entry<Integer, CacheNode> entry = ring.ceilingEntry(hash);
        if (entry == null) {
            entry = ring.firstEntry();
        }
        return entry.getValue();
    }

    private int hash(String key) {
        return Math.abs(key.hashCode() % Integer.MAX_VALUE);
    }
}
```

#### 2. LRU Cache Implementation

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> cache;
    private final Node<K, V> head, tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = cache.get(key);
        if (node == null) return null;
        
        // Move to head (most recently used)
        removeNode(node);
        addToHead(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> node = cache.get(key);
        if (node != null) {
            node.value = value;
            removeNode(node);
            addToHead(node);
        } else {
            if (cache.size() >= capacity) {
                // Remove LRU node
                Node<K, V> lru = tail.prev;
                removeNode(lru);
                cache.remove(lru.key);
            }
            node = new Node<>(key, value);
            cache.put(key, node);
            addToHead(node);
        }
    }

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

    static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
}
```

#### 3. Cache with TTL

```java
public class CacheWithTTL<K, V> {
    private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleanupExecutor;

    public CacheWithTTL() {
        this.cleanupExecutor = Executors.newSingleThreadScheduledExecutor();
        this.cleanupExecutor.scheduleAtFixedRate(
            this::cleanupExpired, 1, 1, TimeUnit.MINUTES
        );
    }

    public void put(K key, V value, long ttl, TimeUnit unit) {
        long expiryTime = System.currentTimeMillis() + unit.toMillis(ttl);
        cache.put(key, new CacheEntry<>(value, expiryTime));
    }

    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null) return null;
        
        if (System.currentTimeMillis() > entry.expiryTime) {
            cache.remove(key);
            return null;
        }
        return entry.value;
    }

    private void cleanupExpired() {
        long now = System.currentTimeMillis();
        cache.entrySet().removeIf(entry -> entry.getValue().expiryTime < now);
    }

    static class CacheEntry<V> {
        final V value;
        final long expiryTime;
        
        CacheEntry(V value, long expiryTime) {
            this.value = value;
            this.expiryTime = expiryTime;
        }
    }
}
```

### Consistency Models

#### Write-Through Cache

```java
public class WriteThroughCache<K, V> {
    private final Map<K, V> cache;
    private final Database database;

    public void put(K key, V value) {
        // Write to cache first
        cache.put(key, value);
        // Then write to database
        database.save(key, value);
    }

    public V get(K key) {
        V value = cache.get(key);
        if (value == null) {
            value = database.load(key);
            if (value != null) {
                cache.put(key, value);
            }
        }
        return value;
    }
}
```

#### Write-Behind Cache

```java
public class WriteBehindCache<K, V> {
    private final Map<K, V> cache;
    private final Database database;
    private final BlockingQueue<WriteOperation<K, V>> writeQueue;
    private final ExecutorService writeExecutor;

    public WriteBehindCache() {
        this.writeQueue = new LinkedBlockingQueue<>(1000);
        this.writeExecutor = Executors.newSingleThreadExecutor();
        this.writeExecutor.submit(this::writeToDatabase);
    }

    public void put(K key, V value) {
        cache.put(key, value);
        writeQueue.offer(new WriteOperation<>(key, value));
    }

    private void writeToDatabase() {
        while (true) {
            try {
                WriteOperation<K, V> op = writeQueue.take();
                database.save(op.key, op.value);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    static class WriteOperation<K, V> {
        final K key;
        final V value;
        
        WriteOperation(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
}
```

### Eviction Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| LRU | Evict least recently used | General purpose |
| LFU | Evict least frequently used | Access patterns with hot spots |
| FIFO | Evict oldest entries | Simple scenarios |
| TTL | Time-based expiration | Time-sensitive data |
| Random | Evict random entry | Simple, good distribution |

### Performance Considerations

| Metric | Target | Optimization |
|--------|--------|--------------|
| Latency (p50) | < 5ms | In-memory cache, efficient hashing |
| Latency (p99) | < 10ms | Consistent hashing, connection pooling |
| Throughput | 100K+ ops/sec | Non-blocking I/O, batch operations |
| Memory | < 80% of heap | LRU eviction, size limits |

---

## Interviewer Insight
Interviewers want to see that you understand cache design trade-offs: consistency vs latency, eviction strategies, and distributed challenges. Strong candidates discuss consistent hashing, TTL handling, and different consistency models (write-through, write-behind).

## Follow-up Questions
1. How do you handle cache invalidation in a distributed system?
2. What is the difference between write-through and write-behind caching?
3. How does consistent hashing work and why is it important?

## 📚 Related Topics
- [Connection Pooling](./03-design-connection-pool.md)
- [Rate Limiting](./05-design-rate-limiter.md)
- [System Design Principles](../12-performance-optimization/README.md)
