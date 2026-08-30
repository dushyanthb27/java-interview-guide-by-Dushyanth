# Caching Strategies in Java

[Home](../../README.md) > [Performance Optimization](./README.md) > Caching Strategies

---

## Overview

Caching stores frequently accessed data in fast-access memory to avoid expensive recomputation or I/O. Choosing the right caching strategy and eviction policy is crucial for performance, consistency, and memory efficiency.

---

## 1. In-Process Caching with Caffeine

Caffeine is the highest-performance in-process cache for Java, successor to Guava Cache.

```java
import com.github.benmanes.caffeine.cache.*;
import java.util.concurrent.TimeUnit;

public class CaffeineExample {

    // Basic cache with size-based eviction
    Cache<String, User> userCache = Caffeine.newBuilder()
        .maximumSize(10_000)                          // max entries
        .expireAfterWrite(Duration.ofMinutes(10))     // TTL
        .expireAfterAccess(Duration.ofMinutes(5))     // idle timeout
        .recordStats()                                 // enable statistics
        .build();

    // Manual put/get
    public User getUser(String userId) {
        return userCache.get(userId, key -> {
            // Called only on cache miss (loading function)
            return userRepository.findById(key);
        });
    }

    // Loading cache (automatic loading)
    LoadingCache<String, User> loadingCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .refreshAfterWrite(Duration.ofMinutes(5))    // async refresh
        .build(userId -> userRepository.findById(userId));

    // Async loading cache
    AsyncLoadingCache<String, User> asyncCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .buildAsync(userId ->
            CompletableFuture.supplyAsync(() ->
                userRepository.findById(userId)));

    // Print cache statistics
    public void printStats() {
        CacheStats stats = userCache.stats();
        System.out.printf("Hit rate: %.2f%%%n", stats.hitRate() * 100);
        System.out.printf("Hits: %d, Misses: %d%n",
            stats.hitCount(), stats.missCount());
        System.out.printf("Evictions: %d%n", stats.evictionCount());
        System.out.printf("Avg load time: %.2f ms%n",
            stats.averageLoadPenalty() / 1_000_000.0);
    }
}
```

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
```

---

## 2. Guava Cache

```java
import com.google.common.cache.*;

public class GuavaCacheExample {

    LoadingCache<String, User> cache = CacheBuilder.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .expireAfterAccess(5, TimeUnit.MINUTES)
        .recordStats()
        .removalListener(notification ->
            System.out.printf("Evicted: %s (reason: %s)%n",
                notification.getKey(), notification.getCause()))
        .build(new CacheLoader<>() {
            @Override
            public User load(String key) {
                return userRepository.findById(key);
            }
        });

    public User getUser(String userId) throws ExecutionException {
        return cache.get(userId); // loads automatically on miss
    }
}
```

### Caffeine vs Guava Cache

| Feature             | Caffeine         | Guava Cache    |
|--------------------|------------------|----------------|
| Performance        | Fastest (W-TinyLFU)| Good (LRU)    |
| Hit rate           | Higher           | Lower          |
| Async loading      | Yes              | No             |
| Refresh support    | Yes              | Yes            |
| Spring integration | Default (5.x)    | Legacy         |
| Active development | Yes              | Maintenance    |

---

## 3. Distributed Caching (Redis)

```java
// Spring Boot + Redis caching
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users",
                config.entryTtl(Duration.ofMinutes(10)))
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofHours(1)))
            .build();
    }
}

@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    public User findById(String userId) {
        return userRepository.findById(userId); // cached in Redis
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user); // updates cache
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(String userId) {
        userRepository.deleteById(userId); // removes from cache
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearUserCache() {
        // evicts all entries from "users" cache
    }
}
```

---

## 4. Cache Patterns

### Cache-Aside (Lazy Loading)

```java
// Application manages cache explicitly
public class CacheAsideService {
    private final Cache<String, Product> cache;
    private final ProductRepository repository;

    public Product getProduct(String id) {
        // 1. Check cache first
        Product product = cache.getIfPresent(id);
        if (product != null) {
            return product;  // cache hit
        }

        // 2. On miss, load from database
        product = repository.findById(id);

        // 3. Store in cache
        if (product != null) {
            cache.put(id, product);
        }
        return product;
    }

    public void updateProduct(Product product) {
        repository.save(product);
        cache.invalidate(product.getId()); // invalidate stale cache
    }
}
```

### Read-Through

```java
// Cache handles loading transparently
LoadingCache<String, Product> readThroughCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(30))
    .build(productId -> repository.findById(productId));
    //      ^^^^^^^^ cache loads from DB on miss

Product product = readThroughCache.get("PROD-123"); // transparent loading
```

### Write-Through

```java
// Write to cache AND database simultaneously
public class WriteThroughService {
    private final Cache<String, Product> cache;
    private final ProductRepository repository;

    public void saveProduct(Product product) {
        repository.save(product);            // 1. Write to DB
        cache.put(product.getId(), product); // 2. Write to cache
        // Both are always in sync
    }
}
```

### Write-Behind (Write-Back)

```java
// Buffer writes and flush to database asynchronously
public class WriteBehindService {
    private final Cache<String, Product> cache;
    private final Queue<Product> writeBuffer = new ConcurrentLinkedQueue<>();
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    public WriteBehindService() {
        // Flush write buffer every 5 seconds
        scheduler.scheduleAtFixedRate(this::flushWrites,
            5, 5, TimeUnit.SECONDS);
    }

    public void saveProduct(Product product) {
        cache.put(product.getId(), product); // instant cache update
        writeBuffer.offer(product);           // queue for async DB write
    }

    private void flushWrites() {
        List<Product> batch = new ArrayList<>();
        Product p;
        while ((p = writeBuffer.poll()) != null) {
            batch.add(p);
        }
        if (!batch.isEmpty()) {
            repository.saveAll(batch); // batch write to DB
        }
    }
}
```

---

## 5. Eviction Policies

### TTL (Time-To-Live), LRU, LFU

```java
// TTL: Expire entries after fixed time since creation
Cache<String, Data> ttlCache = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofMinutes(10))   // 10 min after write
    .build();

// TTI (Time-To-Idle): Expire after last access
Cache<String, Data> ttiCache = Caffeine.newBuilder()
    .expireAfterAccess(Duration.ofMinutes(5))   // 5 min after last access
    .build();

// Size-based eviction (Caffeine uses W-TinyLFU, not simple LRU)
Cache<String, Data> sizeCache = Caffeine.newBuilder()
    .maximumSize(10_000)  // evict based on frequency + recency
    .build();

// Weight-based eviction (for variable-size entries)
Cache<String, byte[]> weightCache = Caffeine.newBuilder()
    .maximumWeight(100_000_000)  // 100 MB total
    .weigher((String key, byte[] value) -> value.length)
    .build();
```

### Eviction Policy Comparison

| Policy   | Description                      | Best For                          |
|----------|----------------------------------|-----------------------------------|
| LRU      | Evict least recently used        | General purpose                   |
| LFU      | Evict least frequently used      | Stable access patterns            |
| W-TinyLFU| Window + frequency (Caffeine)    | Best overall hit rate             |
| TTL      | Expire after fixed time          | Data freshness guarantees         |
| FIFO     | First in, first out              | Simple, predictable               |

---

## 6. Cache Stampede Prevention

When a popular cache entry expires, many threads simultaneously attempt to reload it.

```java
// Problem: Cache stampede (thundering herd)
// 100 threads hit expired cache key -> 100 DB queries for same data

// Solution 1: Caffeine's built-in protection
// LoadingCache ensures only ONE thread loads; others wait
LoadingCache<String, Data> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(10))
    .build(key -> loadFromDb(key)); // only one thread loads per key

// Solution 2: Refresh-ahead (serve stale while refreshing)
LoadingCache<String, Data> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(30))     // hard expiry
    .refreshAfterWrite(Duration.ofMinutes(10))    // async refresh
    .build(key -> loadFromDb(key));
// After 10 min: returns stale value, refreshes async in background
// After 30 min: blocks until fresh value is loaded

// Solution 3: Probabilistic early expiration
public Data getWithJitter(String key) {
    CacheEntry<Data> entry = cache.getIfPresent(key);
    if (entry == null) {
        return loadAndCache(key);
    }

    // Probabilistically refresh before actual expiry
    double ttlRemaining = entry.getExpiryTime() - System.currentTimeMillis();
    double probability = Math.exp(-ttlRemaining / 60_000); // increases near expiry
    if (Math.random() < probability) {
        CompletableFuture.runAsync(() -> loadAndCache(key)); // async refresh
    }
    return entry.getValue();
}
```

---

## 7. Multi-Level Caching

```java
// L1 (in-process) + L2 (distributed) cache
public class MultiLevelCache<K, V> {
    private final Cache<K, V> l1Cache;        // Caffeine (fast, local)
    private final RedisTemplate<K, V> l2Cache; // Redis (shared, durable)

    public MultiLevelCache() {
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(1_000)
            .expireAfterWrite(Duration.ofMinutes(1))
            .build();
    }

    public V get(K key) {
        // L1 check (nanoseconds)
        V value = l1Cache.getIfPresent(key);
        if (value != null) return value;

        // L2 check (milliseconds)
        value = l2Cache.opsForValue().get(key);
        if (value != null) {
            l1Cache.put(key, value); // promote to L1
            return value;
        }

        // Load from source (tens of milliseconds)
        value = loadFromDatabase(key);
        if (value != null) {
            l2Cache.opsForValue().set(key, value, Duration.ofMinutes(30));
            l1Cache.put(key, value);
        }
        return value;
    }

    public void invalidate(K key) {
        l1Cache.invalidate(key);
        l2Cache.delete(key);
    }
}
```

### Access Latency Comparison

```
L1 (Caffeine, in-process):   ~50 ns     (same JVM)
L2 (Redis, same datacenter): ~0.5 ms    (network hop)
L3 (Database):               ~5-50 ms   (query + network)
L4 (External API):           ~100-500 ms (cross-network)
```

---

## 8. Spring Boot Caching Integration

```java
@SpringBootApplication
@EnableCaching
public class Application {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return manager;
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id",
               unless = "#result == null")
    public Product findById(String id) {
        return productRepository.findById(id).orElse(null);
    }

    @Cacheable(value = "products", key = "#name + '_' + #category")
    public List<Product> search(String name, String category) {
        return productRepository.findByNameAndCategory(name, category);
    }

    @CachePut(value = "products", key = "#product.id")
    public Product save(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#id")
    public void delete(String id) {
        productRepository.deleteById(id);
    }
}
```

---

## Interview Tips

- Caffeine is the best in-process cache (W-TinyLFU eviction, highest hit rate)
- Cache-aside is the most common pattern; application manages cache explicitly
- Use `refreshAfterWrite` to prevent cache stampede (serve stale, refresh async)
- Multi-level caching: L1 (Caffeine) + L2 (Redis) for best latency and consistency
- Always set max size AND TTL to prevent unbounded memory growth
- `@Cacheable(unless = "#result == null")` avoids caching null values
- Monitor cache hit rate; below 80% means the cache may not be effective

---

[<< Previous: JDBC Optimization](./q08-jdbc-optimization.md) | [Next: Profiling Tools >>](./q10-profiling-tools.md)
