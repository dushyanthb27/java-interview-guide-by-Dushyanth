# Caching Patterns

## Overview

Caching is a fundamental technique for improving application performance by storing frequently accessed data in fast storage. This guide covers application-level caching strategies for Java applications, including local caches, distributed caches, and cache patterns.

## Why Cache?

### Performance Benefits

```java
// Without cache: Every request hits database
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}
// Response time: 50-200ms per request

// With cache: First request hits database, subsequent requests hit cache
public User getUser(Long id) {
    User user = cache.get(id);
    if (user == null) {
        user = userRepository.findById(id).orElse(null);
        if (user != null) {
            cache.put(id, user);
        }
    }
    return user;
}
// Response time: 0-5ms per cached request (10-100x faster)
```

### Cache Use Cases

- **Read-heavy workloads** - Data read more often than written
- **Expensive computations** - Complex calculations, aggregations
- **Reference data** - Configuration, lookup tables
- **Session data** - User sessions, shopping carts
- **API responses** - External API results

## Cache Types

### Local Cache (In-Memory)

```java
// Caffeine - High-performance local cache
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import java.util.concurrent.TimeUnit;

@Configuration
public class CacheConfig {

    @Bean
    public Cache<Long, User> userCache() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)           // Max 10,000 entries
            .expireAfterWrite(10, TimeUnit.MINUTES)  // Expire after 10 minutes
            .expireAfterAccess(5, TimeUnit.MINUTES) // Expire after 5 minutes idle
            .refreshAfterWrite(8, TimeUnit.MINUTES)  // Refresh after 8 minutes
            .recordStats()                  // Enable statistics
            .build();
    }
}

@Service
public class UserService {

    @Autowired
    private Cache<Long, User> userCache;

    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        return userCache.get(id, key -> 
            userRepository.findById(key).orElse(null)
        );
    }
}
```

### Distributed Cache (Redis)

```java
// Spring Data Redis
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        LettuceConnectionFactory factory = new LettuceConnectionFactory();
        factory.setHostName("localhost");
        factory.setPort(6379);
        return factory;
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(getCacheConfigurations())
            .build();
    }

    private Map<String, RedisCacheConfiguration> getCacheConfigurations() {
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        
        // User cache: 30 minutes
        configMap.put("users", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30)));
        
        // Product cache: 1 hour
        configMap.put("products", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)));
        
        return configMap;
    }
}
```

## Spring Cache Abstraction

### @Cacheable Annotation

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    // Cache result based on product ID
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    // Cache with custom key
    @Cacheable(value = "products", key = "#category + '-' + #page")
    public List<Product> getProductsByCategory(String category, int page) {
        return productRepository.findByCategory(category, PageRequest.of(page, 20));
    }

    // Cache with condition
    @Cacheable(value = "products", 
                key = "#id",
                condition = "#id > 0",
                unless = "#result == null")
    public Product getProductIfValid(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    // Cache with SpEL expression
    @Cacheable(value = "products", 
                key = "#root.methodName + '-' + #id")
    public Product getProductWithCustomKey(Long id) {
        return productRepository.findById(id).orElse(null);
    }
}
```

### @CachePut Annotation

```java
@Service
public class ProductService {

    // Update cache when product is updated
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // Cache and return
    @Cacheable(value = "products", key = "#id")
    @CachePut(value = "products", key = "#id")
    public Product refreshProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }
}
```

### @CacheEvict Annotation

```java
@Service
public class ProductService {

    // Remove specific entry from cache
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // Remove all entries from cache
    @CacheEvict(value = "products", allEntries = true)
    public void clearAllProductsCache() {
        // Clears entire products cache
    }

    // Evict before method execution
    @CacheEvict(value = "products", key = "#id", beforeInvocation = true)
    public void deleteProductWithEvictBefore(Long id) {
        productRepository.deleteById(id);
    }

    // Evict multiple caches
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#product.id"),
        @CacheEvict(value = "productCategories", allEntries = true)
    })
    public void updateProductAndClearCategories(Product product) {
        productRepository.save(product);
    }
}
```

### @Caching Annotation

```java
@Service
public class ProductService {

    // Combine multiple cache operations
    @Caching(
        cacheable = {
            @Cacheable(value = "products", key = "#id")
        },
        put = {
            @CachePut(value = "productDetails", key = "#id")
        }
    )
    public Product getProductWithDetails(Long id) {
        Product product = productRepository.findById(id).orElse(null);
        // Load details
        return product;
    }
}
```

## Cache Patterns

### Cache-Aside (Lazy Loading)

```java
@Service
public class CacheAsideService {

    @Autowired
    private Cache<Long, User> cache;

    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        // 1. Check cache
        User user = cache.getIfPresent(id);
        
        if (user == null) {
            // 2. Cache miss - load from database
            user = userRepository.findById(id).orElse(null);
            
            if (user != null) {
                // 3. Populate cache
                cache.put(id, user);
            }
        }
        
        return user;
    }
}
```

### Read-Through

```java
@Service
public class ReadThroughService {

    @Autowired
    private Cache<Long, User> cache;

    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        // Cache automatically loads from database on miss
        return cache.get(id, key -> 
            userRepository.findById(key).orElse(null)
        );
    }
}
```

### Write-Through

```java
@Service
public class WriteThroughService {

    @Autowired
    private Cache<Long, User> cache;

    @Autowired
    private UserRepository userRepository;

    public void updateUser(User user) {
        // 1. Write to database
        userRepository.save(user);
        
        // 2. Update cache
        cache.put(user.getId(), user);
    }
}
```

### Write-Behind (Write-Back)

```java
@Service
public class WriteBehindService {

    @Autowired
    private Cache<Long, User> cache;

    @Autowired
    private UserRepository userRepository;

    private final ExecutorService executor = Executors.newFixedThreadPool(5);

    public void updateUserAsync(User user) {
        // 1. Update cache immediately
        cache.put(user.getId(), user);
        
        // 2. Write to database asynchronously
        executor.submit(() -> {
            try {
                userRepository.save(user);
            } catch (Exception e) {
                log.error("Failed to persist user", e);
                // Handle failure - retry, queue, etc.
            }
        });
    }
}
```

### Refresh-Ahead

```java
@Service
public class RefreshAheadService {

    @Autowired
    private Cache<Long, User> cache;

    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        User user = cache.getIfPresent(id);
        
        if (user != null) {
            // Refresh cache in background before expiry
            executor.submit(() -> {
                User freshUser = userRepository.findById(id).orElse(null);
                if (freshUser != null) {
                    cache.put(id, freshUser);
                }
            });
        }
        
        return user;
    }
}
```

## Cache Invalidation Strategies

### Time-Based Expiration

```java
@Configuration
public class CacheConfig {

    @Bean
    public Cache<Long, User> userCache() {
        return Caffeine.newBuilder()
            // Expire after write (absolute time)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            
            // Expire after access (sliding expiration)
            .expireAfterAccess(5, TimeUnit.MINUTES)
            
            // Refresh before expiry
            .refreshAfterWrite(8, TimeUnit.MINUTES)
            
            .build();
    }
}
```

### Size-Based Eviction

```java
@Bean
public Cache<Long, User> userCache() {
    return Caffeine.newBuilder()
        .maximumSize(10_000)  // Evict when > 10,000 entries
        .build();
}
```

### Weight-Based Eviction

```java
@Bean
public Cache<String, byte[]> fileCache() {
    return Caffeine.newBuilder()
        .maximumWeight(100_000_000)  // 100MB
        .weigher((String key, byte[] value) -> value.length)
        .build();
}
```

### Manual Invalidation

```java
@Service
public class CacheInvalidationService {

    @Autowired
    private CacheManager cacheManager;

    public void invalidateUserCache(Long userId) {
        Cache cache = cacheManager.getCache("users");
        if (cache != null) {
            cache.evict(userId);
        }
    }

    public void invalidateAllUserCache() {
        Cache cache = cacheManager.getCache("users");
        if (cache != null) {
            cache.clear();
        }
    }

    public void invalidateAllCaches() {
        cacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.clear();
            }
        });
    }
}
```

## Multi-Level Caching

### L1 (Local) + L2 (Distributed)

```java
@Service
public class MultiLevelCacheService {

    @Autowired
    private Cache<Long, User> localCache;  // Caffeine

    @Autowired
    private RedisTemplate<String, User> redisTemplate;  // Redis

    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        // L1: Check local cache
        User user = localCache.getIfPresent(id);
        if (user != null) {
            return user;
        }

        // L2: Check distributed cache
        String key = "user:" + id;
        user = redisTemplate.opsForValue().get(key);
        if (user != null) {
            localCache.put(id, user);
            return user;
        }

        // L3: Database
        user = userRepository.findById(id).orElse(null);
        if (user != null) {
            redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
            localCache.put(id, user);
        }

        return user;
    }
}
```

## Cache Warming

### Startup Cache Warming

```java
@Component
public class CacheWarmer {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private Cache<Long, User> userCache;

    @PostConstruct
    public void warmCache() {
        log.info("Warming cache...");
        
        // Load frequently accessed users
        List<User> activeUsers = userRepository.findByStatus("ACTIVE");
        activeUsers.forEach(user -> userCache.put(user.getId(), user));
        
        log.info("Cache warmed with {} entries", activeUsers.size());
    }
}
```

### Scheduled Cache Refresh

```java
@Service
public class CacheRefreshService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private Cache<Long, User> userCache;

    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void refreshCache() {
        log.info("Refreshing cache...");
        
        // Refresh hot entries
        userCache.asMap().keySet().forEach(id -> {
            User user = userRepository.findById(id).orElse(null);
            if (user != null) {
                userCache.put(id, user);
            }
        });
        
        log.info("Cache refreshed");
    }
}
```

## Cache Monitoring

### Cache Statistics

```java
@Service
public class CacheMonitorService {

    @Autowired
    private Cache<Long, User> userCache;

    public void printCacheStats() {
        CacheStats stats = userCache.stats();
        
        log.info("Cache Statistics:");
        log.info("  Hit Rate: {}%", stats.hitRate() * 100);
        log.info("  Hit Count: {}", stats.hitCount());
        log.info("  Miss Count: {}", stats.missCount());
        log.info("  Load Success Count: {}", stats.loadSuccessCount());
        log.info("  Load Failure Count: {}", stats.loadFailureCount());
        log.info("  Total Load Time: {}ms", stats.totalLoadTime());
        log.info("  Eviction Count: {}", stats.evictionCount());
        log.info("  Size: {}", userCache.estimatedSize());
    }
}
```

### Metrics Integration

```java
@Component
public class CacheMetrics {

    @Autowired
    private MeterRegistry meterRegistry;

    @Autowired
    private Cache<Long, User> userCache;

    @PostConstruct
    public void bindMetrics() {
        Gauge.builder("cache.size", userCache, Cache::estimatedSize)
            .tags("cache", "users")
            .register(meterRegistry);

        Gauge.builder("cache.hit.rate", userCache, cache -> cache.stats().hitRate())
            .tags("cache", "users")
            .register(meterRegistry);

        Gauge.builder("cache.miss.rate", userCache, cache -> cache.stats().missRate())
            .tags("cache", "users")
            .register(meterRegistry);
    }
}
```

## Common Pitfalls

### 1. Cache Stampede

```java
// ❌ Problem: Multiple threads miss cache and load simultaneously
public User getUser(Long id) {
    User user = cache.getIfPresent(id);
    if (user == null) {
        // All threads hit database at once
        user = userRepository.findById(id).orElse(null);
        cache.put(id, user);
    }
    return user;
}

// ✅ Solution: Use cache loader with locking
public User getUser(Long id) {
    return cache.get(id, key -> 
        userRepository.findById(key).orElse(null)
    );
    // Caffeine handles concurrent loading
}
```

### 2. Stale Data

```java
// ❌ Problem: Cache not invalidated on update
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return productRepository.findById(id).orElse(null);
}

public void updateProduct(Product product) {
    productRepository.save(product);
    // Cache not updated - stale data!
}

// ✅ Solution: Use @CachePut
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}
```

### 3. Memory Leaks

```java
// ❌ Problem: Unbounded cache grows indefinitely
Cache<Long, User> cache = Caffeine.newBuilder().build();
// No size limit, no expiration

// ✅ Solution: Set limits
Cache<Long, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterAccess(10, TimeUnit.MINUTES)
    .build();
```

### 4. Serialization Issues

```java
// ❌ Problem: Objects not serializable for distributed cache
public class User {
    // Not Serializable
    private transient String password;
}

// ✅ Solution: Implement Serializable
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String password;
}
```

### 5. Cache Penetration

```java
// ❌ Problem: Repeated queries for non-existent data
public User getUser(Long id) {
    User user = cache.getIfPresent(id);
    if (user == null) {
        user = userRepository.findById(id).orElse(null);
        cache.put(id, user); // Caches null!
    }
    return user;
}

// ✅ Solution: Cache null values with short TTL or use bloom filter
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

## Best Practices

1. **Monitor cache hit rates** - Aim for >80% hit rate
2. **Set appropriate TTL** - Balance freshness and performance
3. **Use cache loaders** - Handle concurrent loading
4. **Invalidate on updates** - Keep cache consistent
5. **Set size limits** - Prevent memory issues
6. **Use multi-level caching** - Local + distributed
7. **Warm cache on startup** - Improve initial performance
8. **Handle cache failures** - Fallback to database
9. **Document cache strategy** - Why each cache exists
10. **Test cache behavior** - Verify hit/miss scenarios

## Performance Comparison

| Cache Type | Read Latency | Write Latency | Scalability | Use Case |
|------------|--------------|---------------|-------------|----------|
| No Cache | 50-200ms | 10-50ms | Database limit | Small datasets |
| Local Cache | 0-5ms | 0-1ms | Single node | Single instance |
| Distributed Cache | 1-10ms | 1-5ms | Horizontal scale | Distributed system |
| Multi-Level | 0-10ms | 1-5ms | Horizontal scale | High performance |

## Interview Questions

1. What is the difference between cache-aside and read-through?
2. How do you handle cache invalidation?
3. What is cache stampede and how do you prevent it?
4. When would you use local cache vs distributed cache?
5. How do you monitor cache performance?
6. What is the write-through pattern?
7. How do you handle cache failures?
8. What is multi-level caching?

## Related Topics

- [ORM Patterns](./orm-patterns.md)
- [Connection Pooling](./connection-pooling-patterns.md)
- [Database Sharding](./database-sharding.md)
