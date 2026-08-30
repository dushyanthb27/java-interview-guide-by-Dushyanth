# Q8: Design a Distributed Counter

## 📋 Question
Design a distributed counter that can handle high throughput while maintaining consistency. Discuss the data structures and consistency models.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Increment and decrement operations
- Get current count
- Support multiple counters
- High throughput (100K+ ops/sec)

**Non-Functional Requirements:**
- Strong consistency preferred
- Low latency (< 50ms)
- Fault tolerance
- Horizontal scalability

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Client Application                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 Load Balancer (Sharding)                │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Shard 1     │ │  Shard 2     │ │  Shard 3     │
│  (Redis)     │ │  (Redis)     │ │  (Redis)     │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Redis-based Implementation

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class DistributedCounter {
    private final JedisPool jedisPool;
    private final String counterKey;

    public DistributedCounter(JedisPool jedisPool, String counterKey) {
        this.jedisPool = jedisPool;
        this.counterKey = counterKey;
    }

    public long increment() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.incr(counterKey);
        }
    }

    public long increment(long delta) {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.incrBy(counterKey, delta);
        }
    }

    public long decrement() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.decr(counterKey);
        }
    }

    public long decrement(long delta) {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.decrBy(counterKey, delta);
        }
    }

    public long get() {
        try (Jedis jedis = jedisPool.getResource()) {
            String value = jedis.get(counterKey);
            return value != null ? Long.parseLong(value) : 0;
        }
    }

    public void reset() {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.set(counterKey, "0");
        }
    }
}
```

### Sharded Counter

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class ShardedCounter {
    private final int numShards;
    private final Map<Integer, DistributedCounter> shards;
    private final HashFunction hashFunction;

    public ShardedCounter(JedisPool jedisPool, String counterName, int numShards) {
        this.numShards = numShards;
        this.shards = new HashMap<>();
        this.hashFunction = new HashFunction(numShards);

        for (int i = 0; i < numShards; i++) {
            String shardKey = counterName + ":shard:" + i;
            shards.put(i, new DistributedCounter(jedisPool, shardKey));
        }
    }

    public long increment() {
        int shardId = hashFunction.hash(Thread.currentThread().getName());
        return shards.get(shardId).increment();
    }

    public long increment(long delta) {
        int shardId = hashFunction.hash(Thread.currentThread().getName());
        return shards.get(shardId).increment(delta);
    }

    public long decrement() {
        int shardId = hashFunction.hash(Thread.currentThread().getName());
        return shards.get(shardId).decrement();
    }

    public long get() {
        long total = 0;
        for (DistributedCounter shard : shards.values()) {
            total += shard.get();
        }
        return total;
    }

    static class HashFunction {
        private final int numShards;

        HashFunction(int numShards) {
            this.numShards = numShards;
        }

        int hash(String key) {
            return Math.abs(key.hashCode() % numShards);
        }
    }
}
```

### Rate-Limited Counter

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicReference;

public class RateLimitedCounter {
    private final AtomicLong counter;
    private final AtomicLong lastUpdate;
    private final long maxRate;
    private final long timeWindow;

    public RateLimitedCounter(long maxRate, long timeWindow) {
        this.counter = new AtomicLong(0);
        this.lastUpdate = new AtomicLong(System.currentTimeMillis());
        this.maxRate = maxRate;
        this.timeWindow = timeWindow;
    }

    public boolean increment() {
        long now = System.currentTimeMillis();
        long timeSinceLastUpdate = now - lastUpdate.get();

        if (timeSinceLastUpdate >= timeWindow) {
            // Reset window
            counter.set(0);
            lastUpdate.set(now);
        }

        if (counter.get() < maxRate) {
            counter.incrementAndGet();
            return true;
        }
        return false;
    }

    public long get() {
        return counter.get();
    }
}
```

### Consistency Models

#### Strong Consistency with Locks

```java
import java.util.concurrent.locks.ReentrantLock;

public class StrongConsistentCounter {
    private long count;
    private final ReentrantLock lock = new ReentrantLock();

    public StrongConsistentCounter() {
        this.count = 0;
    }

    public long increment() {
        lock.lock();
        try {
            return ++count;
        } finally {
            lock.unlock();
        }
    }

    public long decrement() {
        lock.lock();
        try {
            return --count;
        } finally {
            lock.unlock();
        }
    }

    public long get() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

#### Eventual Consistency with CRDTs

```java
import java.util.concurrent.atomic.AtomicLong;

public class EventualCounter {
    private final AtomicLong count;
    private final AtomicLong localIncrement;

    public EventualCounter() {
        this.count = new AtomicLong(0);
        this.localIncrement = new AtomicLong(0);
    }

    public long increment() {
        localIncrement.incrementAndGet();
        return count.incrementAndGet();
    }

    public long get() {
        return count.get();
    }

    public void merge(long remoteCount) {
        count.addAndGet(remoteCount);
    }
}
```

### Performance Considerations

| Model | Throughput | Latency | Consistency | Use Case |
|-------|-----------|---------|-------------|----------|
| Redis (single) | High | Low | Strong | Simple counters |
| Sharded Redis | Very High | Low | Strong | High throughput |
| Rate-limited | High | Very Low | Strong | Rate limiting |
| CRDT | Very High | Very Low | Eventual | Distributed systems |

---

## Interviewer Insight
Interviewers want to see that you understand distributed counter design: sharding, consistency models, and performance trade-offs. Strong candidates discuss Redis-based solutions, sharding strategies, and different consistency models.

## Follow-up Questions
1. How do you handle counter consistency in a distributed system?
2. What is the difference between strong and eventual consistency for counters?
3. How do you implement a rate-limited counter?

## 📚 Related Topics
- [Cache Design](./02-design-cache.md)
- [System Design Principles](../12-performance-optimization/README.md)
- [Concurrency Patterns](../04-multithreading-concurrency/README.md)
