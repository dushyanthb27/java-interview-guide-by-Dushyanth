# Rate Limiter Template

## Overview
Rate limiting implementations using Token Bucket and Sliding Window algorithms to control request rates and prevent system overload.

## Use Cases
- API rate limiting
- DDoS protection
- Request throttling
- Bandwidth management
- Load shedding
- Fair resource allocation

## Implementation

### Simple Version
Basic token bucket implementation.

### Production Version
Advanced with multiple algorithms and statistics.

### Thread-Safe Version
Fully concurrent with proper synchronization.

## Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.locks.ReentrantLock;

/**
 * SIMPLE VERSION: Token Bucket Rate Limiter
 * - Fixed capacity
 * - Constant refill rate
 * - Simple thread-safe implementation
 */
public class SimpleTokenBucketRateLimiter {
    private final long capacity;
    private final long refillRate; // tokens per second
    private long tokens;
    private long lastRefillTime;
    private final Object lock = new Object();

    /**
     * @param capacity Maximum tokens in bucket
     * @param refillRate Tokens added per second
     */
    public SimpleTokenBucketRateLimiter(long capacity, long refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
    }

    /**
     * Try to consume tokens
     * Returns true if allowed, false if rate limit exceeded
     */
    public boolean allowRequest(long tokensNeeded) {
        synchronized (lock) {
            refillTokens();

            if (tokens >= tokensNeeded) {
                tokens -= tokensNeeded;
                return true;
            }
            return false;
        }
    }

    /**
     * Refill tokens based on elapsed time
     */
    private void refillTokens() {
        long now = System.currentTimeMillis();
        long elapsedMs = now - lastRefillTime;
        long tokensToAdd = (elapsedMs * refillRate) / 1000;

        if (tokensToAdd > 0) {
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTime = now;
        }
    }

    /**
     * Get current token count
     */
    public long getAvailableTokens() {
        synchronized (lock) {
            refillTokens();
            return tokens;
        }
    }
}

/**
 * PRODUCTION VERSION: Advanced Rate Limiter
 * - Token Bucket algorithm
 * - Sliding Window algorithm
 * - Statistics tracking
 * - Multiple rate limit strategies
 */
public class ProductionRateLimiter {
    private final RateLimitStrategy strategy;
    private final long capacity;
    private final long refillRate;
    private final AtomicLong allowedRequests = new AtomicLong(0);
    private final AtomicLong rejectedRequests = new AtomicLong(0);

    public enum RateLimitStrategy {
        TOKEN_BUCKET,
        SLIDING_WINDOW,
        SLIDING_WINDOW_LOG
    }

    private final TokenBucketLimiter tokenBucket;
    private final SlidingWindowLimiter slidingWindow;
    private final SlidingWindowLogLimiter slidingWindowLog;

    public ProductionRateLimiter(RateLimitStrategy strategy, long capacity, 
                                long refillRate) {
        this.strategy = strategy;
        this.capacity = capacity;
        this.refillRate = refillRate;

        this.tokenBucket = new TokenBucketLimiter(capacity, refillRate);
        this.slidingWindow = new SlidingWindowLimiter(capacity, refillRate);
        this.slidingWindowLog = new SlidingWindowLogLimiter(capacity, refillRate);
    }

    /**
     * Check if request is allowed
     */
    public boolean allowRequest() {
        boolean allowed = switch (strategy) {
            case TOKEN_BUCKET -> tokenBucket.allowRequest();
            case SLIDING_WINDOW -> slidingWindow.allowRequest();
            case SLIDING_WINDOW_LOG -> slidingWindowLog.allowRequest();
        };

        if (allowed) {
            allowedRequests.incrementAndGet();
        } else {
            rejectedRequests.incrementAndGet();
        }

        return allowed;
    }

    /**
     * Get rate limiter statistics
     */
    public Map<String, Object> getStats() {
        Map<String, Object> stats = new HashMap<>();
        long total = allowedRequests.get() + rejectedRequests.get();
        stats.put("allowed", allowedRequests.get());
        stats.put("rejected", rejectedRequests.get());
        stats.put("total", total);
        stats.put("acceptanceRate", total > 0 ? 
                 (double) allowedRequests.get() / total : 0.0);
        return stats;
    }

    /**
     * Token Bucket Implementation
     */
    private static class TokenBucketLimiter {
        private final long capacity;
        private final long refillRate;
        private long tokens;
        private long lastRefillTime;
        private final ReentrantLock lock = new ReentrantLock();

        TokenBucketLimiter(long capacity, long refillRate) {
            this.capacity = capacity;
            this.refillRate = refillRate;
            this.tokens = capacity;
            this.lastRefillTime = System.currentTimeMillis();
        }

        boolean allowRequest() {
            lock.lock();
            try {
                refillTokens();
                if (tokens >= 1) {
                    tokens--;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        private void refillTokens() {
            long now = System.currentTimeMillis();
            long elapsedMs = now - lastRefillTime;
            long tokensToAdd = (elapsedMs * refillRate) / 1000;

            if (tokensToAdd > 0) {
                tokens = Math.min(capacity, tokens + tokensToAdd);
                lastRefillTime = now;
            }
        }
    }

    /**
     * Sliding Window Counter Implementation
     */
    private static class SlidingWindowLimiter {
        private final long capacity;
        private final long windowSizeMs;
        private long windowStart;
        private long requestCount;
        private final ReentrantLock lock = new ReentrantLock();

        SlidingWindowLimiter(long capacity, long refillRate) {
            this.capacity = capacity;
            this.windowSizeMs = 1000; // 1 second window
            this.windowStart = System.currentTimeMillis();
            this.requestCount = 0;
        }

        boolean allowRequest() {
            lock.lock();
            try {
                long now = System.currentTimeMillis();
                long elapsedMs = now - windowStart;

                // Reset window if expired
                if (elapsedMs >= windowSizeMs) {
                    windowStart = now;
                    requestCount = 0;
                }

                if (requestCount < capacity) {
                    requestCount++;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }
    }

    /**
     * Sliding Window Log Implementation
     * More accurate but uses more memory
     */
    private static class SlidingWindowLogLimiter {
        private final long capacity;
        private final long windowSizeMs;
        private final Queue<Long> requestLog;
        private final ReentrantLock lock = new ReentrantLock();

        SlidingWindowLogLimiter(long capacity, long refillRate) {
            this.capacity = capacity;
            this.windowSizeMs = 1000; // 1 second window
            this.requestLog = new LinkedList<>();
        }

        boolean allowRequest() {
            lock.lock();
            try {
                long now = System.currentTimeMillis();
                long windowStart = now - windowSizeMs;

                // Remove old requests outside window
                while (!requestLog.isEmpty() && requestLog.peek() < windowStart) {
                    requestLog.poll();
                }

                if (requestLog.size() < capacity) {
                    requestLog.add(now);
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }
    }
}

/**
 * THREAD-SAFE VERSION: Distributed Rate Limiter
 * - Supports multiple clients/users
 * - Per-user rate limiting
 * - Distributed rate limiting ready
 */
public class DistributedRateLimiter {
    private final Map<String, ClientRateLimiter> clientLimiters;
    private final long capacity;
    private final long refillRate;
    private final ReentrantLock lock = new ReentrantLock();
    private final long clientTimeoutMs;

    /**
     * Per-client rate limiter
     */
    private static class ClientRateLimiter {
        private long tokens;
        private long lastRefillTime;
        private long lastAccessTime;
        private final long capacity;
        private final long refillRate;

        ClientRateLimiter(long capacity, long refillRate) {
            this.capacity = capacity;
            this.refillRate = refillRate;
            this.tokens = capacity;
            this.lastRefillTime = System.currentTimeMillis();
            this.lastAccessTime = System.currentTimeMillis();
        }

        boolean allowRequest() {
            refillTokens();
            lastAccessTime = System.currentTimeMillis();

            if (tokens >= 1) {
                tokens--;
                return true;
            }
            return false;
        }

        private void refillTokens() {
            long now = System.currentTimeMillis();
            long elapsedMs = now - lastRefillTime;
            long tokensToAdd = (elapsedMs * refillRate) / 1000;

            if (tokensToAdd > 0) {
                tokens = Math.min(capacity, tokens + tokensToAdd);
                lastRefillTime = now;
            }
        }

        boolean isExpired(long timeoutMs) {
            return System.currentTimeMillis() - lastAccessTime > timeoutMs;
        }
    }

    public DistributedRateLimiter(long capacity, long refillRate, 
                                 long clientTimeoutMs) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.clientTimeoutMs = clientTimeoutMs;
        this.clientLimiters = new HashMap<>();
    }

    /**
     * Check if request is allowed for client
     */
    public boolean allowRequest(String clientId) {
        lock.lock();
        try {
            // Clean up expired clients
            clientLimiters.entrySet().removeIf(e -> 
                e.getValue().isExpired(clientTimeoutMs));

            // Get or create client limiter
            ClientRateLimiter limiter = clientLimiters.computeIfAbsent(clientId, 
                k -> new ClientRateLimiter(capacity, refillRate));

            return limiter.allowRequest();
        } finally {
            lock.unlock();
        }
    }

    /**
     * Get client statistics
     */
    public Map<String, Object> getClientStats(String clientId) {
        lock.lock();
        try {
            ClientRateLimiter limiter = clientLimiters.get(clientId);
            if (limiter == null) {
                return new HashMap<>();
            }

            Map<String, Object> stats = new HashMap<>();
            stats.put("capacity", capacity);
            stats.put("availableTokens", limiter.tokens);
            stats.put("lastAccessTime", limiter.lastAccessTime);
            return stats;
        } finally {
            lock.unlock();
        }
    }

    /**
     * Get all clients count
     */
    public int getActiveClientsCount() {
        lock.lock();
        try {
            return clientLimiters.size();
        } finally {
            lock.unlock();
        }
    }
}

/**
 * ADAPTIVE RATE LIMITER: Adjusts rate based on system load
 */
public class AdaptiveRateLimiter {
    private volatile long capacity;
    private volatile long refillRate;
    private long tokens;
    private long lastRefillTime;
    private final ReentrantLock lock = new ReentrantLock();
    private final long minCapacity;
    private final long maxCapacity;

    public AdaptiveRateLimiter(long initialCapacity, long initialRefillRate,
                              long minCapacity, long maxCapacity) {
        this.capacity = initialCapacity;
        this.refillRate = initialRefillRate;
        this.tokens = initialCapacity;
        this.lastRefillTime = System.currentTimeMillis();
        this.minCapacity = minCapacity;
        this.maxCapacity = maxCapacity;
    }

    /**
     * Allow request and adjust rate based on load
     */
    public boolean allowRequest(double systemLoad) {
        lock.lock();
        try {
            refillTokens();

            // Adjust capacity based on system load (0.0 to 1.0)
            long newCapacity = (long) (minCapacity + 
                (maxCapacity - minCapacity) * (1 - systemLoad));
            this.capacity = Math.max(minCapacity, 
                Math.min(maxCapacity, newCapacity));

            if (tokens >= 1) {
                tokens--;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    private void refillTokens() {
        long now = System.currentTimeMillis();
        long elapsedMs = now - lastRefillTime;
        long tokensToAdd = (elapsedMs * refillRate) / 1000;

        if (tokensToAdd > 0) {
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTime = now;
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class RateLimiterTest {

    @Test
    public void testSimpleTokenBucket() throws InterruptedException {
        SimpleTokenBucketRateLimiter limiter = 
            new SimpleTokenBucketRateLimiter(10, 5); // 10 capacity, 5 tokens/sec

        // Should allow first 10 requests
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.allowRequest(1));
        }

        // Should reject 11th request
        assertFalse(limiter.allowRequest(1));

        // Wait for refill
        Thread.sleep(1000);
        assertTrue(limiter.allowRequest(1));
    }

    @Test
    public void testProductionRateLimiterTokenBucket() {
        ProductionRateLimiter limiter = new ProductionRateLimiter(
            ProductionRateLimiter.RateLimitStrategy.TOKEN_BUCKET, 100, 10);

        int allowed = 0;
        for (int i = 0; i < 150; i++) {
            if (limiter.allowRequest()) {
                allowed++;
            }
        }

        assertTrue(allowed <= 100);
        var stats = limiter.getStats();
        assertEquals(allowed, stats.get("allowed"));
    }

    @Test
    public void testSlidingWindowLimiter() {
        ProductionRateLimiter limiter = new ProductionRateLimiter(
            ProductionRateLimiter.RateLimitStrategy.SLIDING_WINDOW, 10, 10);

        int allowed = 0;
        for (int i = 0; i < 15; i++) {
            if (limiter.allowRequest()) {
                allowed++;
            }
        }

        assertEquals(10, allowed);
    }

    @Test
    public void testDistributedRateLimiter() {
        DistributedRateLimiter limiter = new DistributedRateLimiter(10, 5, 60000);

        // Client 1
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.allowRequest("client1"));
        }
        assertFalse(limiter.allowRequest("client1"));

        // Client 2 should have separate limit
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.allowRequest("client2"));
        }
        assertFalse(limiter.allowRequest("client2"));

        assertEquals(2, limiter.getActiveClientsCount());
    }

    @Test
    public void testConcurrentRateLimiting() throws InterruptedException {
        ProductionRateLimiter limiter = new ProductionRateLimiter(
            ProductionRateLimiter.RateLimitStrategy.TOKEN_BUCKET, 1000, 100);

        int threadCount = 10;
        int requestsPerThread = 200;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        AtomicInteger allowed = new AtomicInteger(0);

        for (int t = 0; t < threadCount; t++) {
            executor.submit(() -> {
                for (int i = 0; i < requestsPerThread; i++) {
                    if (limiter.allowRequest()) {
                        allowed.incrementAndGet();
                    }
                }
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(10, TimeUnit.SECONDS));
        assertTrue(allowed.get() <= 1000);
    }

    @Test
    public void testAdaptiveRateLimiter() {
        AdaptiveRateLimiter limiter = new AdaptiveRateLimiter(100, 10, 50, 200);

        // Low load - should allow more requests
        int allowedLowLoad = 0;
        for (int i = 0; i < 200; i++) {
            if (limiter.allowRequest(0.2)) { // 20% load
                allowedLowLoad++;
            }
        }

        // High load - should allow fewer requests
        int allowedHighLoad = 0;
        for (int i = 0; i < 200; i++) {
            if (limiter.allowRequest(0.8)) { // 80% load
                allowedHighLoad++;
            }
        }

        assertTrue(allowedLowLoad > allowedHighLoad);
    }
}
```

## Performance Considerations

### Time Complexity
- **allowRequest()**: O(1) for token bucket and sliding window
- **allowRequest()**: O(n) for sliding window log (n = requests in window)

### Space Complexity
- **Token Bucket**: O(1)
- **Sliding Window**: O(1)
- **Sliding Window Log**: O(n) where n = requests in window
- **Distributed**: O(m) where m = number of clients

### Optimization Tips
1. **Token Bucket**: Most efficient, O(1) space and time
2. **Sliding Window**: Good balance of accuracy and efficiency
3. **Per-client limiting**: Use distributed limiter for multi-tenant
4. **Adaptive limiting**: Adjust based on system metrics
5. **Batch requests**: Allow multiple tokens per request

## Common Pitfalls

1. **Not handling concurrent access**: Race conditions
2. **Incorrect refill calculation**: Off-by-one errors
3. **Memory leaks in distributed limiter**: Not cleaning up old clients
4. **Bursty traffic**: Token bucket allows bursts, may need smoothing
5. **Clock skew**: System time changes break calculations
6. **No backpressure**: Requests pile up instead of being rejected
7. **Unfair distribution**: Some clients get more tokens

## Variations

1. **Leaky Bucket**: Constant outflow rate
2. **Hierarchical Rate Limiting**: Multiple levels of limits
3. **Weighted Rate Limiting**: Different costs for different requests
4. **Distributed Rate Limiting**: Across multiple servers
5. **Adaptive Rate Limiting**: Adjusts based on system load
