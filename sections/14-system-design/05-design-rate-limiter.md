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
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.refillInterval = refillInterval;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
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
        long now = System.currentTimeMillis();
        long timeSinceLastRefill = now - lastRefillTime;
        
        if (timeSinceLastRefill >= refillInterval) {
            int tokensToAdd = (int) (timeSinceLastRefill / refillInterval) * refillRate;
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTime = now;
        }
    }

    public int getAvailableTokens() {
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
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.requestTimestamps = new LinkedList<>();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        
        // Remove timestamps outside the window
        while (!requestTimestamps.isEmpty() && 
               now - requestTimestamps.peek() > windowSize) {
            requestTimestamps.poll();
        }
        
        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.offer(now);
            return true;
        }
        return false;
    }

    public int getCurrentRequestCount() {
        long now = System.currentTimeMillis();
        while (!requestTimestamps.isEmpty() && 
               now - requestTimestamps.peek() > windowSize) {
            requestTimestamps.poll();
        }
        return requestTimestamps.size();
    }
}
```

#### 3. Fixed Window Counter

```java
import java.util.concurrent.atomic.AtomicInteger;

public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowSize;
    private final AtomicInteger counter;
    private long windowStart;

    public FixedWindowRateLimiter(int maxRequests, long windowSize) {
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.counter = new AtomicInteger(0);
        this.windowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        
        if (now - windowStart >= windowSize) {
            // Reset window
            windowStart = now;
            counter.set(0);
        }
        
        if (counter.get() < maxRequests) {
            counter.incrementAndGet();
            return true;
        }
        return false;
    }
}
```

### Distributed Rate Limiter

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class DistributedRateLimiter {
    private final JedisPool jedisPool;
    private final int maxRequests;
    private final long windowSize;

    public DistributedRateLimiter(JedisPool jedisPool, int maxRequests, long windowSize) {
        this.jedisPool = jedisPool;
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
    }

    public boolean allowRequest(String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = "rate_limit:" + userId;
            long now = System.currentTimeMillis();
            long windowStart = now - windowSize;
            
            // Remove old entries
            jedis.zremrangeByScore(key, 0, windowStart);
            
            // Count current requests
            long currentCount = jedis.zcard(key);
            
            if (currentCount < maxRequests) {
                jedis.zadd(key, now, String.valueOf(now));
                jedis.expire(key, windowSize / 1000 + 1);
                return true;
            }
            return false;
        }
    }
}
```

### Rate Limiter with Multiple Tiers

```java
import java.util.*;

public class TieredRateLimiter {
    private final Map<String, RateLimitConfig> configs;
    private final Map<String, TokenBucketRateLimiter> limiters;

    public TieredRateLimiter() {
        this.configs = new HashMap<>();
        this.limiters = new HashMap<>();
        
        // Default configurations
        configs.put("default", new RateLimitConfig(100, 60, 1000)); // 100 req/min
        configs.put("premium", new RateLimitConfig(1000, 60, 1000)); // 1000 req/min
        configs.put("api", new RateLimitConfig(10000, 60, 1000)); // 10K req/min
    }

    public boolean allowRequest(String userId, String tier) {
        RateLimitConfig config = configs.getOrDefault(tier, configs.get("default"));
        TokenBucketRateLimiter limiter = limiters.computeIfAbsent(
            tier, 
            t -> new TokenBucketRateLimiter(config.capacity, config.refillRate, config.refillInterval)
        );
        return limiter.allowRequest();
    }

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
| Token Bucket | Smooth rate limiting, handles bursts | More complex | API rate limiting |
| Leaky Bucket | Constant rate, no bursts | Wastes capacity | Network traffic shaping |
| Sliding Window | Accurate, no spikes | Memory intensive | Precise rate limiting |
| Fixed Window | Simple, low memory | Bursts at boundaries | Simple scenarios |

### API Gateway Integration

```java
import javax.servlet.*;
import javax.servlet.http.*;
import java.io.IOException;

public class RateLimitFilter implements Filter {
    private final SlidingWindowRateLimiter rateLimiter;

    public RateLimitFilter(int maxRequests, long windowSize) {
        this.rateLimiter = new SlidingWindowRateLimiter(maxRequests, windowSize);
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String userId = httpRequest.getHeader("X-User-ID");
        
        if (userId == null) {
            userId = httpRequest.getRemoteAddr();
        }
        
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
