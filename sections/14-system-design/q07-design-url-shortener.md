# Q7: Design a URL Shortener

## Question
Design and implement a URL Shortener service in Java. Use base62 encoding, HashMap storage, counter-based ID generation, collision handling, redirection, and statistics tracking.

## Difficulty: Easy

---

## Solution

### Requirements

**Functional Requirements:**
- Shorten a long URL to a short URL (e.g., `http://short.ly/abc123`)
- Redirect a short URL to the original long URL
- Custom alias support (optional vanity URLs)
- Link expiration (TTL)
- Click statistics tracking (total clicks, clicks by date)

**Non-Functional Requirements:**
- Thread-safe for concurrent shortening and redirection
- O(1) lookup for redirection
- Collision-free short URL generation
- Unique short codes guaranteed

### API Design

```java
public interface UrlShortenerService {
    ShortenResult shorten(String longUrl);
    ShortenResult shorten(String longUrl, String customAlias);
    ShortenResult shorten(String longUrl, long ttlMillis);
    String resolve(String shortCode);
    UrlStats getStats(String shortCode);
    boolean delete(String shortCode);
}

public class ShortenResult {
    private final String shortCode;
    private final String shortUrl;
    private final String longUrl;
    private final long createdAt;
    private final long expiresAt;
    // constructor, getters ...
}

public class UrlStats {
    private final String shortCode;
    private final String longUrl;
    private final long totalClicks;
    private final Map<String, Long> clicksByDate; // "2024-01-15" -> count
    private final long createdAt;
    // constructor, getters ...
}
```

### Implementation

#### Base62 Encoder

```java
public class Base62Encoder {

    private static final String BASE62_CHARS =
        "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    public static String encode(long number) {
        if (number == 0) return String.valueOf(BASE62_CHARS.charAt(0));

        StringBuilder sb = new StringBuilder();
        while (number > 0) {
            sb.append(BASE62_CHARS.charAt((int) (number % 62)));
            number /= 62;
        }
        return sb.reverse().toString();
    }

    public static long decode(String encoded) {
        long result = 0;
        for (char c : encoded.toCharArray()) {
            result = result * 62 + BASE62_CHARS.indexOf(c);
        }
        return result;
    }
}
```

#### URL Entry

```java
import java.time.LocalDate;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class UrlEntry {

    private final String shortCode;
    private final String longUrl;
    private final long createdAt;
    private final long expiresAt; // 0 = never expires
    private final AtomicLong totalClicks = new AtomicLong(0);
    private final ConcurrentHashMap<String, AtomicLong> clicksByDate = new ConcurrentHashMap<>();

    public UrlEntry(String shortCode, String longUrl, long ttlMillis) {
        this.shortCode = shortCode;
        this.longUrl = longUrl;
        this.createdAt = System.currentTimeMillis();
        this.expiresAt = (ttlMillis > 0) ? createdAt + ttlMillis : 0;
    }

    public boolean isExpired() {
        return expiresAt > 0 && System.currentTimeMillis() > expiresAt;
    }

    public void recordClick() {
        totalClicks.incrementAndGet();
        String today = LocalDate.now().toString();
        clicksByDate.computeIfAbsent(today, k -> new AtomicLong(0)).incrementAndGet();
    }

    public String getShortCode() { return shortCode; }
    public String getLongUrl() { return longUrl; }
    public long getCreatedAt() { return createdAt; }
    public long getExpiresAt() { return expiresAt; }
    public long getTotalClicks() { return totalClicks.get(); }
    public ConcurrentHashMap<String, AtomicLong> getClicksByDate() { return clicksByDate; }
}
```

#### URL Shortener Service

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class InMemoryUrlShortener implements UrlShortenerService {

    private static final String BASE_URL = "http://short.ly/";
    private static final int MIN_CODE_LENGTH = 6;

    private final ConcurrentHashMap<String, UrlEntry> shortCodeToEntry = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> longUrlToShortCode = new ConcurrentHashMap<>();
    private final AtomicLong counter = new AtomicLong(100_000); // start above 0 for longer codes

    @Override
    public ShortenResult shorten(String longUrl) {
        return shorten(longUrl, 0);
    }

    @Override
    public ShortenResult shorten(String longUrl, String customAlias) {
        validateUrl(longUrl);

        if (customAlias == null || customAlias.isEmpty()) {
            return shorten(longUrl);
        }

        // Check alias availability
        if (shortCodeToEntry.containsKey(customAlias)) {
            throw new IllegalArgumentException("Alias already taken: " + customAlias);
        }

        UrlEntry entry = new UrlEntry(customAlias, longUrl, 0);
        shortCodeToEntry.put(customAlias, entry);
        longUrlToShortCode.put(longUrl, customAlias);

        return new ShortenResult(customAlias, BASE_URL + customAlias, longUrl,
                                  entry.getCreatedAt(), entry.getExpiresAt());
    }

    @Override
    public ShortenResult shorten(String longUrl, long ttlMillis) {
        validateUrl(longUrl);

        // Deduplication: return existing short URL if already shortened
        String existing = longUrlToShortCode.get(longUrl);
        if (existing != null) {
            UrlEntry existingEntry = shortCodeToEntry.get(existing);
            if (existingEntry != null && !existingEntry.isExpired()) {
                return new ShortenResult(existing, BASE_URL + existing, longUrl,
                                          existingEntry.getCreatedAt(), existingEntry.getExpiresAt());
            }
        }

        // Generate unique short code
        String shortCode = generateShortCode();

        UrlEntry entry = new UrlEntry(shortCode, longUrl, ttlMillis);
        shortCodeToEntry.put(shortCode, entry);
        longUrlToShortCode.put(longUrl, shortCode);

        return new ShortenResult(shortCode, BASE_URL + shortCode, longUrl,
                                  entry.getCreatedAt(), entry.getExpiresAt());
    }

    @Override
    public String resolve(String shortCode) {
        UrlEntry entry = shortCodeToEntry.get(shortCode);

        if (entry == null) {
            return null; // 404
        }

        if (entry.isExpired()) {
            shortCodeToEntry.remove(shortCode);
            longUrlToShortCode.remove(entry.getLongUrl());
            return null; // expired
        }

        // Record click for statistics
        entry.recordClick();
        return entry.getLongUrl();
    }

    @Override
    public UrlStats getStats(String shortCode) {
        UrlEntry entry = shortCodeToEntry.get(shortCode);
        if (entry == null) {
            throw new IllegalArgumentException("Short code not found: " + shortCode);
        }

        Map<String, Long> dailyClicks = entry.getClicksByDate().entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().get()));

        return new UrlStats(shortCode, entry.getLongUrl(), entry.getTotalClicks(),
                            dailyClicks, entry.getCreatedAt());
    }

    @Override
    public boolean delete(String shortCode) {
        UrlEntry entry = shortCodeToEntry.remove(shortCode);
        if (entry != null) {
            longUrlToShortCode.remove(entry.getLongUrl());
            return true;
        }
        return false;
    }

    // --- Internal helpers ---

    private String generateShortCode() {
        // Counter-based ID ensures uniqueness — no collision possible
        long id = counter.incrementAndGet();
        String code = Base62Encoder.encode(id);

        // Pad to minimum length
        while (code.length() < MIN_CODE_LENGTH) {
            code = "0" + code;
        }
        return code;
    }

    private void validateUrl(String url) {
        if (url == null || url.isEmpty()) {
            throw new IllegalArgumentException("URL cannot be null or empty");
        }
        if (!url.startsWith("http://") && !url.startsWith("https://")) {
            throw new IllegalArgumentException("URL must start with http:// or https://");
        }
    }
}
```

### Why Counter-Based IDs Avoid Collisions

| Approach | Collision Risk | Thread Safety | Notes |
|----------|---------------|---------------|-------|
| Counter + Base62 | None (monotonic) | `AtomicLong` | Simple, predictable |
| Random + Check | Low, but possible | Needs CAS loop | Unpredictable codes |
| MD5/SHA hash | Very low, but possible | Stateless | Wasted bits (truncation) |
| UUID | None | Built-in | Too long for short URLs |

The counter approach is ideal because `AtomicLong.incrementAndGet()` guarantees a unique value per call — no collision handling needed.

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Short code generation | `AtomicLong` counter — lock-free, unique |
| URL lookup | `ConcurrentHashMap` — lock-free reads |
| Click statistics | `AtomicLong` per counter — no locking |
| Daily click tracking | `ConcurrentHashMap` with `AtomicLong` values |
| Deduplication | `longUrlToShortCode` reverse map |
| Expiration | Lazy cleanup on `resolve()` |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `shorten()` | O(1) | Counter increment + map put |
| `resolve()` | O(1) | HashMap lookup |
| `getStats()` | O(D) | D = number of distinct days with clicks |
| `delete()` | O(1) | Map remove |

### Testing Strategy

```java
public class UrlShortenerTest {

    @Test
    void testShortenAndResolve() {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ShortenResult result = shortener.shorten("https://www.example.com/very/long/url");

        assertNotNull(result.getShortCode());
        assertTrue(result.getShortUrl().startsWith("http://short.ly/"));

        String resolved = shortener.resolve(result.getShortCode());
        assertEquals("https://www.example.com/very/long/url", resolved);
    }

    @Test
    void testCustomAlias() {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ShortenResult result = shortener.shorten("https://example.com", "my-link");

        assertEquals("my-link", result.getShortCode());
        assertEquals("https://example.com", shortener.resolve("my-link"));
    }

    @Test
    void testDuplicateAlias() {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        shortener.shorten("https://example.com", "taken");

        assertThrows(IllegalArgumentException.class,
            () -> shortener.shorten("https://other.com", "taken"));
    }

    @Test
    void testExpiration() throws InterruptedException {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ShortenResult result = shortener.shorten("https://example.com", 200); // 200ms TTL

        assertNotNull(shortener.resolve(result.getShortCode()));
        Thread.sleep(300);
        assertNull(shortener.resolve(result.getShortCode()));
    }

    @Test
    void testClickStatistics() {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ShortenResult result = shortener.shorten("https://example.com");

        shortener.resolve(result.getShortCode());
        shortener.resolve(result.getShortCode());
        shortener.resolve(result.getShortCode());

        UrlStats stats = shortener.getStats(result.getShortCode());
        assertEquals(3, stats.getTotalClicks());
    }

    @Test
    void testDeduplication() {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ShortenResult r1 = shortener.shorten("https://example.com");
        ShortenResult r2 = shortener.shorten("https://example.com");

        assertEquals(r1.getShortCode(), r2.getShortCode()); // same URL returns same code
    }

    @Test
    void testConcurrentShortening() throws Exception {
        InMemoryUrlShortener shortener = new InMemoryUrlShortener();
        ExecutorService exec = Executors.newFixedThreadPool(10);
        Set<String> codes = ConcurrentHashMap.newKeySet();

        for (int i = 0; i < 1000; i++) {
            final int idx = i;
            exec.submit(() -> {
                ShortenResult r = shortener.shorten("https://example.com/page/" + idx);
                codes.add(r.getShortCode());
            });
        }

        exec.shutdown();
        exec.awaitTermination(10, TimeUnit.SECONDS);
        assertEquals(1000, codes.size()); // all codes unique
    }
}
```

---

## Interviewer Insight
This is often a warm-up system design question. Interviewers want to see Base62 encoding knowledge, a clear uniqueness strategy (counter vs. hash), and awareness of edge cases (expiration, deduplication, collision). For senior candidates, discuss scaling: distributed ID generation (Snowflake IDs), database sharding, caching hot URLs, and analytics pipelines.

## Follow-up Questions
1. How would you generate unique IDs in a distributed system with multiple servers?
2. How would you handle analytics at scale (millions of clicks per second)?
3. How would you prevent abuse (e.g., someone shortening millions of spam URLs)?

## Related Topics
- [Collections Framework](../03-collections-framework/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)
