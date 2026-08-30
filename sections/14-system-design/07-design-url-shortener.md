# Q7: Design a URL Shortener

## 📋 Question
Design a URL shortener service like bit.ly. Discuss the data structures, encoding scheme, and scalability considerations.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Generate short URLs from long URLs
- Redirect short URLs to original URLs
- Custom short URLs
- Analytics (click tracking)
- Expiration dates

**Non-Functional Requirements:**
- Low latency (< 100ms for redirect)
- High availability
- Scalable to billions of URLs
- Unique short URL generation

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Client Browser                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Load Balancer                            │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  API Server  │ │  API Server  │ │  API Server  │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Database Cluster                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │  Primary DB  │ │  Replica DB  │ │  Replica DB  │  │
│  └──────────────┘ └──────────────┘ └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### URL Encoding

```java
public class URLEncoder {
    private static final String ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private static final int BASE = ALPHABET.length();

    public static String encode(long id) {
        StringBuilder shortUrl = new StringBuilder();
        
        while (id > 0) {
            shortUrl.append(ALPHABET.charAt((int) (id % BASE)));
            id /= BASE;
        }
        
        return shortUrl.reverse().toString();
    }

    public static long decode(String shortUrl) {
        long id = 0;
        
        for (int i = 0; i < shortUrl.length(); i++) {
            char c = shortUrl.charAt(i);
            id = id * BASE + ALPHABET.indexOf(c);
        }
        
        return id;
    }
}
```

### Database Schema

```sql
CREATE TABLE url_mappings (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    long_url VARCHAR(2048) NOT NULL,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    user_id VARCHAR(50),
    click_count INT DEFAULT 0
);

CREATE INDEX idx_short_code ON url_mappings(short_code);
CREATE INDEX idx_user_id ON url_mappings(user_id);
```

### Service Implementation

```java
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

public class URLShortenerService {
    private final Map<String, URLMapping> urlMap;
    private final AtomicLong idGenerator;
    private final URLEncoder urlEncoder;

    public URLShortenerService() {
        this.urlMap = new ConcurrentHashMap<>();
        this.idGenerator = new AtomicLong(1);
        this.urlEncoder = new URLEncoder();
    }

    public String shortenURL(String longUrl) {
        return shortenURL(longUrl, null, null);
    }

    public String shortenURL(String longUrl, String customCode, String userId) {
        // Validate URL
        if (!isValidURL(longUrl)) {
            throw new IllegalArgumentException("Invalid URL");
        }

        long id = idGenerator.getAndIncrement();
        String shortCode = customCode != null ? customCode : urlEncoder.encode(id);

        // Check for collision
        if (urlMap.containsKey(shortCode)) {
            throw new IllegalArgumentException("Short code already exists");
        }

        URLMapping mapping = new URLMapping(id, longUrl, shortCode, userId);
        urlMap.put(shortCode, mapping);
        
        return shortCode;
    }

    public String getLongURL(String shortCode) {
        URLMapping mapping = urlMap.get(shortCode);
        if (mapping == null) {
            throw new IllegalArgumentException("Short URL not found");
        }

        // Check expiration
        if (mapping.isExpired()) {
            urlMap.remove(shortCode);
            throw new IllegalArgumentException("Short URL has expired");
        }

        // Increment click count
        mapping.incrementClickCount();
        
        return mapping.getLongUrl();
    }

    public URLStats getStats(String shortCode) {
        URLMapping mapping = urlMap.get(shortCode);
        if (mapping == null) {
            throw new IllegalArgumentException("Short URL not found");
        }

        return new URLStats(
            mapping.getLongUrl(),
            mapping.getClickCount(),
            mapping.getCreatedAt(),
            mapping.getUserId()
        );
    }

    private boolean isValidURL(String url) {
        try {
            new java.net.URL(url).toURI();
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    static class URLMapping {
        private final long id;
        private final String longUrl;
        private final String shortCode;
        private final String userId;
        private final long createdAt;
        private final Long expiresAt;
        private final AtomicInteger clickCount;

        URLMapping(long id, String longUrl, String shortCode, String userId) {
            this.id = id;
            this.longUrl = longUrl;
            this.shortCode = shortCode;
            this.userId = userId;
            this.createdAt = System.currentTimeMillis();
            this.expiresAt = null;
            this.clickCount = new AtomicInteger(0);
        }

        URLMapping(long id, String longUrl, String shortCode, String userId, long expiresAt) {
            this.id = id;
            this.longUrl = longUrl;
            this.shortCode = shortCode;
            this.userId = userId;
            this.createdAt = System.currentTimeMillis();
            this.expiresAt = expiresAt;
            this.clickCount = new AtomicInteger(0);
        }

        String getLongUrl() { return longUrl; }
        String getShortCode() { return shortCode; }
        String getUserId() { return userId; }
        long getCreatedAt() { return createdAt; }
        Long getExpiresAt() { return expiresAt; }
        int getClickCount() { return clickCount.get(); }
        
        boolean isExpired() {
            return expiresAt != null && System.currentTimeMillis() > expiresAt;
        }
        
        void incrementClickCount() {
            clickCount.incrementAndGet();
        }
    }

    static class URLStats {
        private final String longUrl;
        private final int clickCount;
        private final long createdAt;
        private final String userId;

        URLStats(String longUrl, int clickCount, long createdAt, String userId) {
            this.longUrl = longUrl;
            this.clickCount = clickCount;
            this.createdAt = createdAt;
            this.userId = userId;
        }

        // Getters
        public String getLongUrl() { return longUrl; }
        public int getClickCount() { return clickCount; }
        public long getCreatedAt() { return createdAt; }
        public String getUserId() { return userId; }
    }
}
```

### API Endpoint

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class URLShortenerController {
    private final URLShortenerService urlShortener;

    public URLShortenerController(URLShortenerService urlShortener) {
        this.urlShortener = urlShortener;
    }

    @PostMapping("/shorten")
    public Map<String, String> shortenURL(@RequestBody Map<String, String> request) {
        String longUrl = request.get("longUrl");
        String customCode = request.get("customCode");
        String userId = request.get("userId");

        String shortCode = urlShortener.shortenURL(longUrl, customCode, userId);
        
        Map<String, String> response = new HashMap<>();
        response.put("shortCode", shortCode);
        response.put("shortUrl", "https://short.ly/" + shortCode);
        return response;
    }

    @GetMapping("/{shortCode}")
    public Map<String, String> redirect(@PathVariable String shortCode) {
        String longUrl = urlShortener.getLongURL(shortCode);
        
        Map<String, String> response = new HashMap<>();
        response.put("longUrl", longUrl);
        return response;
    }

    @GetMapping("/stats/{shortCode}")
    public URLStats getStats(@PathVariable String shortCode) {
        return urlShortener.getStats(shortCode);
    }
}
```

### Scalability Considerations

| Aspect | Solution | Impact |
|--------|----------|--------|
| Database | Master-slave replication | Read scalability |
| Caching | Redis for hot URLs | Reduced database load |
| Encoding | Base62 encoding | Compact short URLs |
| Load Balancing | Round-robin DNS | High availability |
| Analytics | Async processing | Non-blocking redirects |

---

## Interviewer Insight
Interviewers want to see that you understand URL shortener design: encoding schemes, database schema, and scalability. Strong candidates discuss base62 encoding, collision handling, and caching strategies.

## Follow-up Questions
1. How do you handle collisions in short URL generation?
2. What encoding scheme would you use and why?
3. How do you handle custom short URLs?

## 📚 Related Topics
- [Cache Design](./02-design-cache.md)
- [Database Patterns](../../bonus/06-database-patterns/README.md)
- [System Design Principles](../12-performance-optimization/README.md)
