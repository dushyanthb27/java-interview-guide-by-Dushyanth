# Q5: Design a Rate Limiter

## Question
Design and implement a Rate Limiter in Java. Implement both Token Bucket and Sliding Window algorithms. Support per-client limiting, thread safety, and distributed rate limiting.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Allow or reject a request based on rate limits
- Support per-client rate limiting (e.g., by API key or IP)
- Configurable rate (requests per second/minute/hour)
- Return rate limit headers (remaining tokens, reset time)
- Support burst allowance

**Non-Functional Requirements:**
- Thread-safe under high concurrency
- O(1) time per request check
- Minimal memory per client
- Extensible to distributed environments

### API Design

```java
public interface RateLimiter {
    RateLimitResult tryAcquire(String clientId);
    RateLimitResult tryAcquire(String clientId, int permits);
}

public class RateLimitResult {
    private final boolean allowed;
    private final int remainingTokens;
    private final long retryAfterMillis;

    public RateLimitResult(boolean allowed, int remainingTokens, long retryAfterMillis) {
        this.allowed = allowed;
        this.remainingTokens = remainingTokens;
        this.retryAfterMillis = retryAfterMillis;
    }

    public boolean isAllowed() { return allowed; }
    public int getRemainingTokens() { return remainingTokens; }
    public long getRetryAfterMillis() { return retryAfterMillis; }

    // For HTTP response headers
    public Map<String, String> toHeaders() {
        Map<String, String> headers = new HashMap<>();
        headers.put("X-RateLimit-Remaining", String.valueOf(remainingTokens));
        if (!allowed) {
            headers.put("Retry-After", String.valueOf(retryAfterMillis / 1000));
        }
        return headers;
    }
}
```

### Implementation 1: Token Bucket Algorithm

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicReference;

public class TokenBucketRateLimiter implements RateLimiter {

    private final int maxTokens;       // bucket capacity (burst size)
    private final double refillRate;   // tokens per millisecond
    private final ConcurrentHashMap<String, AtomicReference<Bucket>> buckets =
        new ConcurrentHashMap<>();

    private static class Bucket {
        final double tokens;
        final long lastRefillTimestamp;

        Bucket(double tokens, long lastRefillTimestamp) {
            this.tokens = tokens;
            this.lastRefillTimestamp = lastRefillTimestamp;
        }
    }

    /**
     * @param maxTokens     maximum burst size
     * @param refillPerSec  tokens added per second
     */
    public TokenBucketRateLimiter(int maxTokens, double refillPerSec) {
        this.maxTokens = maxTokens;
        this.refillRate = refillPerSec / 1000.0; // convert to per-millisecond
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        return tryAcquire(clientId, 1);
    }

    @Override
    public RateLimitResult tryAcquire(String clientId, int permits) {
        AtomicReference<Bucket> ref = buckets.computeIfAbsent(clientId,
            k -> new AtomicReference<>(new Bucket(maxTokens, System.currentTimeMillis())));

        while (true) {
            Bucket current = ref.get();
            long now = System.currentTimeMillis();
            long elapsed = now - current.lastRefillTimestamp;

            // Refill tokens
            double newTokens = Math.min(maxTokens, current.tokens + elapsed * refillRate);

            if (newTokens >= permits) {
                Bucket updated = new Bucket(newTokens - permits, now);
                if (ref.compareAndSet(current, updated)) {
                    return new RateLimitResult(true, (int) updated.tokens, 0);
                }
                // CAS failed — another thread updated; retry
            } else {
                // Not enough tokens — calculate wait time
                double deficit = permits - newTokens;
                long retryAfterMs = (long) (deficit / refillRate);
                // Update timestamp even on rejection to account for refill
                Bucket updated = new Bucket(newTokens, now);
                ref.compareAndSet(current, updated);
                return new RateLimitResult(false, (int) newTokens, retryAfterMs);
            }
        }
    }
}
```

### Implementation 2: Sliding Window Log Algorithm

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedDeque;

public class SlidingWindowRateLimiter implements RateLimiter {

    private final int maxRequests;
    private final long windowMillis;
    private final ConcurrentHashMap<String, ConcurrentLinkedDeque<Long>> requestLogs =
        new ConcurrentHashMap<>();

    /**
     * @param maxRequests  max requests in the window
     * @param windowMillis window duration in milliseconds
     */
    public SlidingWindowRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        return tryAcquire(clientId, 1);
    }

    @Override
    public synchronized RateLimitResult tryAcquire(String clientId, int permits) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowMillis;

        ConcurrentLinkedDeque<Long> log = requestLogs.computeIfAbsent(
            clientId, k -> new ConcurrentLinkedDeque<>());

        // Remove expired entries
        while (!log.isEmpty() && log.peekFirst() <= windowStart) {
            log.pollFirst();
        }

        int currentCount = log.size();

        if (currentCount + permits <= maxRequests) {
            for (int i = 0; i < permits; i++) {
                log.addLast(now);
            }
            return new RateLimitResult(true, maxRequests - currentCount - permits, 0);
        } else {
            // Calculate when the oldest entry will expire
            long oldestTimestamp = log.peekFirst() != null ? log.peekFirst() : now;
            long retryAfter = oldestTimestamp + windowMillis - now;
            return new RateLimitResult(false, 0, Math.max(retryAfter, 0));
        }
    }
}
```

### Implementation 3: Sliding Window Counter (Memory-Efficient)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicReference;

public class SlidingWindowCounterRateLimiter implements RateLimiter {

    private final int maxRequests;
    private final long windowMillis;
    private final ConcurrentHashMap<String, AtomicReference<WindowState>> windows =
        new ConcurrentHashMap<>();

    private static class WindowState {
        final long windowStart;
        final int currentCount;
        final int previousCount;

        WindowState(long windowStart, int currentCount, int previousCount) {
            this.windowStart = windowStart;
            this.currentCount = currentCount;
            this.previousCount = previousCount;
        }
    }

    public SlidingWindowCounterRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        return tryAcquire(clientId, 1);
    }

    @Override
    public RateLimitResult tryAcquire(String clientId, int permits) {
        AtomicReference<WindowState> ref = windows.computeIfAbsent(
            clientId, k -> new AtomicReference<>(new WindowState(System.currentTimeMillis(), 0, 0)));

        while (true) {
            WindowState state = ref.get();
            long now = System.currentTimeMillis();
            long elapsed = now - state.windowStart;

            WindowState newState;
            if (elapsed >= windowMillis) {
                // Slide window: current becomes previous
                newState = new WindowState(now, 0, state.currentCount);
            } else {
                newState = state;
            }

            // Weighted count: previous window's contribution proportional to overlap
            double overlapFraction = Math.max(0, 1.0 - ((double) elapsed / windowMillis));
            double estimatedCount = newState.previousCount * overlapFraction + newState.currentCount;

            if (estimatedCount + permits <= maxRequests) {
                WindowState updated = new WindowState(
                    newState.windowStart, newState.currentCount + permits, newState.previousCount);
                if (ref.compareAndSet(state, updated)) {
                    int remaining = (int) (maxRequests - estimatedCount - permits);
                    return new RateLimitResult(true, Math.max(0, remaining), 0);
                }
                // CAS failed — retry
            } else {
                long retryAfter = (long) (windowMillis * overlapFraction);
                return new RateLimitResult(false, 0, retryAfter);
            }
        }
    }
}
```

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Complexity |
|-----------|--------|----------|------------|
| Token Bucket | O(1) per client | Allows bursts up to bucket size | O(1) |
| Sliding Window Log | O(N) per client (N = requests in window) | Exact | O(N) cleanup |
| Sliding Window Counter | O(1) per client | Approximate (weighted) | O(1) |

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Per-client bucket access | `ConcurrentHashMap` + `AtomicReference` with CAS |
| Token bucket refill race | Immutable `Bucket` snapshot + `compareAndSet` |
| Sliding window cleanup | Synchronized per-client in log approach |
| Client map growth | Periodic cleanup of inactive clients |
| Lock-free token bucket | CAS loop avoids blocking entirely |

### Performance Analysis

| Operation | Token Bucket | Sliding Window Log | Sliding Window Counter |
|-----------|-------------|-------------------|----------------------|
| `tryAcquire()` | O(1) | O(expired entries) | O(1) |
| Memory/client | O(1) | O(window requests) | O(1) |

### Testing Strategy

```java
public class RateLimiterTest {

    @Test
    void testTokenBucketAllowsBurst() {
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(10, 5.0);

        // Should allow burst of 10
        for (int i = 0; i < 10; i++) {
            assertTrue(limiter.tryAcquire("client-1").isAllowed());
        }
        // 11th should be rejected
        assertFalse(limiter.tryAcquire("client-1").isAllowed());
    }

    @Test
    void testTokenBucketRefills() throws Exception {
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(5, 10.0);

        // Exhaust tokens
        for (int i = 0; i < 5; i++) limiter.tryAcquire("c1");
        assertFalse(limiter.tryAcquire("c1").isAllowed());

        // Wait for refill
        Thread.sleep(600); // ~6 tokens refilled
        assertTrue(limiter.tryAcquire("c1").isAllowed());
    }

    @Test
    void testSlidingWindowLimiting() {
        SlidingWindowRateLimiter limiter = new SlidingWindowRateLimiter(5, 1000);

        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire("c1").isAllowed());
        }
        RateLimitResult result = limiter.tryAcquire("c1");
        assertFalse(result.isAllowed());
        assertTrue(result.getRetryAfterMillis() > 0);
    }

    @Test
    void testPerClientIsolation() {
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(2, 1.0);

        assertTrue(limiter.tryAcquire("client-A").isAllowed());
        assertTrue(limiter.tryAcquire("client-A").isAllowed());
        assertFalse(limiter.tryAcquire("client-A").isAllowed());

        // client-B should be independent
        assertTrue(limiter.tryAcquire("client-B").isAllowed());
    }

    @Test
    void testConcurrentAccess() throws Exception {
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(100, 50.0);
        AtomicInteger allowed = new AtomicInteger(0);
        ExecutorService exec = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 200; i++) {
            exec.submit(() -> {
                if (limiter.tryAcquire("shared-client").isAllowed()) {
                    allowed.incrementAndGet();
                }
            });
        }

        exec.shutdown();
        exec.awaitTermination(5, TimeUnit.SECONDS);
        assertTrue(allowed.get() <= 100); // should not exceed bucket capacity
    }
}
```

---

## Interviewer Insight
Interviewers look for your understanding of the trade-offs between Token Bucket (allows bursts, simple) and Sliding Window (smoother, no bursts). Key areas: lock-free CAS for high concurrency, per-client isolation, and how to extend to distributed systems (Redis-based atomic operations, Lua scripts). Discuss HTTP 429 responses and `Retry-After` headers.

## Follow-up Questions
1. How would you implement distributed rate limiting across multiple API gateway instances?
2. What is the difference between Token Bucket and Leaky Bucket algorithms?
3. How would you rate-limit based on multiple dimensions (e.g., per user AND per endpoint)?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)
