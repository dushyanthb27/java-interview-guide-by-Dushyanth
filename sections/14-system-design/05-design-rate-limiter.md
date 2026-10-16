# Q5: Design a Rate Limiter

## 📋 Question
Design a rate limiter to control API request rates. Discuss the algorithms (token bucket, leaky bucket, sliding window) and their trade-offs.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Limit requests per time window
- Support different rate limits per user/API
- Distributed rate limiting across multiple instances
- Configurable rate limits

**Non-Functional Requirements:**
- Low overhead (< 1ms per request)
- High accuracy
- Thread-safe operations
- Minimal memory footprint

### Rate Limiting Algorithms

#### 1. Token Bucket Algorithm

```java
public class TokenBucketRateLimiter {
    private final int capacity;
    private final int refillRate;
    private final long refillInterval;
    private int tokens;
    private long lastRefillTime;

    public TokenBucketRateLimiter(int capacity, int refillRate, long refillInterval) {
        if (capacity <= 0 || refillRate <= 0 || refillInterval <= 0) {
            throw new IllegalArgumentException("Capacity, refill rate, and interval must be positive");
        }
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.refillInterval = refillInterval;
        this.tokens = capacity;
        this.lastRefillTime = System.nanoTime();
    }

    public synchronized boolean allowRequest() {
        refillTokens();
        
        if (tokens > 0) {
            tokens--;
            return true;
        }
        return false;
    }

    private void refillTokens() {
        long now = System.nanoTime();
        long elapsed = now - lastRefillTime;
        long intervalNanos = java.util.concurrent.TimeUnit.MILLISECONDS.toNanos(refillInterval);
        if (elapsed >= intervalNanos) {
            long intervals = elapsed / intervalNanos;
            long replenished = Math.min((long) capacity, (long) tokens + intervals * refillRate);
            tokens = (int) replenished;
            // Keep the fractional interval so frequent calls do not slow token refill.
            lastRefillTime += intervals * intervalNanos;
        }
    }

    public synchronized int getAvailableTokens() {
        refillTokens();
        return tokens;
    }
}
```

#### 2. Sliding Window Algorithm

```java
import java.util.*;

public class SlidingWindowRateLimiter {
    private final int maxRequests;
    private final long windowSize;
    private final Queue<Long> requestTimestamps;

    public SlidingWindowRateLimiter(int maxRequests, long windowSize) {
        if (maxRequests <= 0 || windowSize <= 0) {
            throw new IllegalArgumentException("Request limit and window size must be positive");
        }
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.requestTimestamps = new LinkedList<>();
    }

    public synchronized boolean allowRequest() {
        long now = System.nanoTime();
        
        // Remove timestamps outside the window
        while (!requestTimestamps.isEmpty() && 
               now - requestTimestamps.peek() >= java.util.concurrent.TimeUnit.MILLISECONDS.toNanos(windowSize)) {
            requestTimestamps.poll();
        }
        
        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.offer(now);
            return true;
        }
        return false;
    }

    public synchronized int getCurrentRequestCount() {
        long now = System.nanoTime();
        while (!requestTimestamps.isEmpty() && 
               now - requestTimestamps.peek() >= java.util.concurrent.TimeUnit.MILLISECONDS.toNanos(windowSize)) {
            requestTimestamps.poll();
        }
        return requestTimestamps.size();
    }
}
```

#### 3. Fixed Window Counter

```java
public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowSize;
    private int counter;
    private long windowStart;

    public FixedWindowRateLimiter(int maxRequests, long windowSize) {
        if (maxRequests <= 0 || windowSize <= 0) {
            throw new IllegalArgumentException("Request limit and window size must be positive");
        }
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.counter = 0;
        this.windowStart = System.nanoTime();
    }

    public synchronized boolean allowRequest() {
        long now = System.nanoTime();
        long windowNanos = java.util.concurrent.TimeUnit.MILLISECONDS.toNanos(windowSize);
        
        if (now - windowStart >= windowNanos) {
            // Reset window
            windowStart = now;
            counter = 0;
        }
        
        if (counter < maxRequests) {
            counter++;
            return true;
        }
        return false;
    }
}
```

### Distributed Sliding-Window Limiter

The Redis operations that prune old entries, count requests, and add a new request
must be atomic. Separate client calls race: two application instances can both see
one remaining slot and both admit a request. Run the operation in one Lua script.
Redis supplies the clock so application-server clock skew cannot change the window.
The caller must provide a unique request ID; using only the timestamp as the sorted-set
member would overwrite requests that arrive in the same millisecond.

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import java.util.Collections;
import java.util.List;

public class DistributedRateLimiter {
    private final JedisPool jedisPool;
    private final int maxRequests;
    private final long windowSize;
    private static final String SCRIPT = """
        local t = redis.call('TIME')
        local now = t[1] * 1000 + math.floor(t[2] / 1000)
        local windowStart = now - tonumber(ARGV[1])
        redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', windowStart)
        local count = redis.call('ZCARD', KEYS[1])
        if count >= tonumber(ARGV[2]) then return 0 end
        redis.call('ZADD', KEYS[1], now, ARGV[3])
        redis.call('PEXPIRE', KEYS[1], ARGV[1])
        return 1
        """;

    public DistributedRateLimiter(JedisPool jedisPool, int maxRequests, long windowSize) {
        if (maxRequests <= 0 || windowSize <= 0) {
            throw new IllegalArgumentException("Request limit and window size must be positive");
        }
        this.jedisPool = jedisPool;
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
    }

    public boolean allowRequest(String userId, String requestId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = "rate_limit:" + userId;
            Object result = jedis.eval(SCRIPT, Collections.singletonList(key),
                    List.of(Long.toString(windowSize), Integer.toString(maxRequests), requestId));
            return ((Number) result).longValue() == 1L;
        }
    }
}
```

### Rate Limiter with Multiple Tiers

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class TieredRateLimiter {
    private final Map<String, RateLimitConfig> configs;
    private final ConcurrentMap<LimiterKey, TokenBucketRateLimiter> limiters = new ConcurrentHashMap<>();

    public TieredRateLimiter() {
        this.configs = Map.of(
            "default", new RateLimitConfig(100, 1, 600),
            "premium", new RateLimitConfig(1000, 10, 600),
            "api", new RateLimitConfig(10000, 100, 600)
        );
        
    }

    public boolean allowRequest(String userId, String tier) {
        Objects.requireNonNull(userId, "userId");
        String effectiveTier = configs.containsKey(tier) ? tier : "default";
        RateLimitConfig config = configs.get(effectiveTier);
        LimiterKey key = new LimiterKey(userId, effectiveTier);
        TokenBucketRateLimiter limiter = limiters.computeIfAbsent(
            key,
            ignored -> new TokenBucketRateLimiter(config.capacity, config.refillRate, config.refillInterval)
        );
        return limiter.allowRequest();
    }

    private record LimiterKey(String userId, String tier) {}

    static class RateLimitConfig {
        final int capacity;
        final int refillRate;
        final long refillInterval;

        RateLimitConfig(int capacity, int refillRate, long refillInterval) {
            this.capacity = capacity;
            this.refillRate = refillRate;
            this.refillInterval = refillInterval;
        }
    }
}
```

### Algorithm Comparison

| Algorithm | Pros | Cons | Use Case |
|-----------|------|------|----------|
| Token Bucket | Enforces a refill rate while allowing a bounded burst | Needs per-key state | APIs that permit short bursts |
| Leaky Bucket | Smooths output to a steady rate | Can delay or reject bursts | Traffic shaping / paced work |
| Sliding-Log Window | Exact count over the trailing interval | Stores one timestamp per admitted request | Strict per-user limits at modest volume |
| Sliding-Window Counter | Lower memory than a log, approximate near the boundary | Approximation depends on previous-window traffic | High-volume distributed APIs |
| Fixed Window | Simple, low memory | Bursts at boundaries | Simple scenarios |

The examples above are teaching implementations. A production design also needs a
policy for Redis failure (fail open or closed), key expiry and eviction, request-ID
uniqueness, user identity trust, cardinality limits, and how limit changes affect
existing buckets. The in-memory examples are process-local; they do not enforce a
shared limit across application instances. The tiered example creates one limiter per
user and tier, so a real service needs bounded/expiring state or a distributed store.

### API Gateway Integration

```java
import javax.servlet.*;
import javax.servlet.http.*;
import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class RateLimitFilter implements Filter {
    private final ConcurrentMap<String, SlidingWindowRateLimiter> limiters = new ConcurrentHashMap<>();
    private final int maxRequests;
    private final long windowSizeMillis;

    public RateLimitFilter(int maxRequests, long windowSize) {
        this.maxRequests = maxRequests;
        this.windowSizeMillis = windowSize;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        // Use an identity established by trusted authentication middleware; clients
        // must not be able to choose another user's rate-limit key.
        String userId = httpRequest.getUserPrincipal() == null
                ? httpRequest.getRemoteAddr()
                : httpRequest.getUserPrincipal().getName();
        
        SlidingWindowRateLimiter rateLimiter = limiters.computeIfAbsent(userId,
                ignored -> new SlidingWindowRateLimiter(maxRequests, windowSizeMillis));
        
        if (!rateLimiter.allowRequest()) {
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setStatus(429); // Too Many Requests
            httpResponse.getWriter().write("Rate limit exceeded");
            return;
        }
        
        chain.doFilter(request, response);
    }
}
```

### Performance Considerations

| Metric | Target | Optimization |
|--------|--------|--------------|
| Overhead | < 1ms | Efficient data structures, minimal locking |
| Memory | < 10MB per 1M users | Sliding window cleanup, compact storage |
| Accuracy | ±5% | Synchronized clocks, atomic operations |
| Scalability | Horizontal | Distributed rate limiting |

---

## Interviewer Insight
Interviewers want to see that you understand different rate limiting algorithms and their trade-offs. Strong candidates discuss token bucket vs sliding window, distributed challenges, and integration patterns with API gateways.

## Follow-up Questions
1. What is the difference between token bucket and leaky bucket algorithms?
2. How do you handle rate limiting in a distributed system?
3. What are the challenges of implementing sliding window rate limiting?

## 📚 Related Topics
- [Cache Design](./02-design-cache.md)
- [System Design Principles](../12-performance-optimization/README.md)
- [API Design Patterns](../13-framework-integration/README.md)
