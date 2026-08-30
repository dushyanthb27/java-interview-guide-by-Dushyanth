# Q3: How Do Memory Leaks Occur in Java?

[← Back to Memory Management](./README.md)

---

## Overview

Despite automatic garbage collection, Java applications **can and do** leak memory. A memory leak in Java occurs when objects are no longer needed but remain **reachable** through unintended references, preventing the GC from reclaiming them.

---

## Common Causes of Memory Leaks

### 1. Static Collections Growing Unbounded

```java
// ❌ Anti-pattern: static collection that only grows
public class EventLog {

    private static final List<Event> events = new ArrayList<>();

    public static void log(Event event) {
        events.add(event);   // Never removed — list grows forever
    }
}
```

```java
// ✅ Fix: use bounded collections or eviction policies
public class EventLog {

    private static final int MAX_SIZE = 10_000;
    private static final Deque<Event> events = new ArrayDeque<>();

    public static void log(Event event) {
        if (events.size() >= MAX_SIZE) {
            events.pollFirst();   // Evict oldest entry
        }
        events.addLast(event);
    }
}
```

### 2. Listeners and Callbacks Not Removed

```java
// ❌ Anti-pattern: registering listeners without cleanup
public class SensorMonitor {

    public void startMonitoring(EventBus bus) {
        bus.register(this);   // 'this' is now strongly referenced by EventBus
    }

    // No unregister — SensorMonitor can never be GC'd while EventBus lives
}
```

```java
// ✅ Fix: always provide an unregister mechanism
public class SensorMonitor implements AutoCloseable {

    private final EventBus bus;

    public SensorMonitor(EventBus bus) {
        this.bus = bus;
        bus.register(this);
    }

    @Override
    public void close() {
        bus.unregister(this);   // Clean removal
    }
}
```

### 3. Non-Static Inner Class References

```java
// ❌ Anti-pattern: inner class holds implicit reference to outer class
public class DataProcessor {

    private byte[] largeBuffer = new byte[10_000_000];   // 10MB

    public Runnable createTask() {
        // Anonymous inner class holds reference to DataProcessor
        return new Runnable() {
            @Override
            public void run() {
                System.out.println("Processing...");
                // Does NOT use largeBuffer, but still holds reference to outer class!
            }
        };
    }
}
```

```java
// ✅ Fix: use a static inner class or lambda (if no outer reference needed)
public class DataProcessor {

    private byte[] largeBuffer = new byte[10_000_000];

    public Runnable createTask() {
        return () -> System.out.println("Processing...");
        // Lambda captures only what it uses — no implicit outer reference
    }

    // Or use a static nested class
    private static class ProcessingTask implements Runnable {
        @Override
        public void run() {
            System.out.println("Processing...");
        }
    }
}
```

### 4. ThreadLocal Not Cleaned Up

```java
// ❌ Anti-pattern: ThreadLocal without cleanup (especially in thread pools)
public class RequestContext {

    private static final ThreadLocal<UserSession> SESSION = new ThreadLocal<>();

    public static void setSession(UserSession session) {
        SESSION.set(session);
    }

    public static UserSession getSession() {
        return SESSION.get();
    }
    // Thread pool reuses threads — old session data never cleared!
}
```

```java
// ✅ Fix: always remove ThreadLocal values when done
public class RequestContext {

    private static final ThreadLocal<UserSession> SESSION = new ThreadLocal<>();

    public static void setSession(UserSession session) {
        SESSION.set(session);
    }

    public static UserSession getSession() {
        return SESSION.get();
    }

    public static void clear() {
        SESSION.remove();   // Must be called in a finally block
    }
}

// Usage in a servlet filter or interceptor
public class ContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        try {
            RequestContext.setSession(extractSession(req));
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear();   // ✅ Always cleanup
        }
    }
}
```

### 5. Unclosed Resources

```java
// ❌ Anti-pattern: resources not closed on exception path
public String readFile(String path) throws IOException {
    BufferedReader reader = new BufferedReader(new FileReader(path));
    String content = reader.readLine();
    reader.close();   // Skipped if readLine() throws!
    return content;
}
```

```java
// ✅ Fix: try-with-resources
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.readLine();   // Reader always closed, even on exception
    }
}
```

### 6. String.intern() Abuse

```java
// ❌ Anti-pattern: interning dynamic strings fills the string pool
public class BadIntern {

    public void processRequests(List<String> requestIds) {
        for (String id : requestIds) {
            String interned = id.intern();   // Unique IDs fill the pool
            process(interned);
        }
    }
}
```

```java
// ✅ intern() is only appropriate for a bounded, repetitive set of strings
public class GoodIntern {

    public void categorize(String status) {
        // Status has limited values: "ACTIVE", "INACTIVE", "PENDING"
        String canonical = status.intern();   // ✅ Bounded set — saves memory
    }
}
```

### 7. Class Loader Leaks

```java
// ❌ Common in application servers with hot-deployment
public class PluginLoader {

    private static final Map<String, Class<?>> cache = new HashMap<>();

    public Class<?> loadPlugin(String name, URL jarUrl) throws Exception {
        URLClassLoader loader = new URLClassLoader(new URL[]{jarUrl});
        Class<?> clazz = loader.loadClass(name);
        cache.put(name, clazz);   // Class holds reference to ClassLoader
        // ClassLoader holds reference to all its loaded classes
        // All static fields of those classes are also retained
        return clazz;
    }
    // Even after removing the plugin, the old ClassLoader can't be GC'd
}
```

```java
// ✅ Fix: use WeakReferences for class caches, close ClassLoaders
public class PluginLoader implements AutoCloseable {

    private final Map<String, WeakReference<Class<?>>> cache = new ConcurrentHashMap<>();
    private URLClassLoader currentLoader;

    public Class<?> loadPlugin(String name, URL jarUrl) throws Exception {
        currentLoader = new URLClassLoader(new URL[]{jarUrl});
        Class<?> clazz = currentLoader.loadClass(name);
        cache.put(name, new WeakReference<>(clazz));
        return clazz;
    }

    @Override
    public void close() throws Exception {
        if (currentLoader != null) {
            currentLoader.close();   // Release resources
        }
    }
}
```

---

## Detection Strategies

### 1. Monitor heap over time

```
Healthy:        ┌─┐ ┌─┐ ┌─┐ ┌─┐      (sawtooth pattern — GC reclaims memory)
               ╱  ╲╱  ╲╱  ╲╱  ╲

Leak:          ╱  ╱  ╱  ╱  ╱  ╱       (baseline keeps rising after each GC)
```

### 2. JVM flags for leak detection

```bash
# Enable GC logging
java -Xlog:gc*:file=gc.log:time MyApp

# Auto heap dump on OOM
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof MyApp
```

### 3. Programmatic monitoring

```java
// ✅ Monitor memory usage in your application
public class MemoryMonitor {

    public static void logMemoryUsage() {
        Runtime rt = Runtime.getRuntime();
        long totalMB = rt.totalMemory() / (1024 * 1024);
        long freeMB = rt.freeMemory() / (1024 * 1024);
        long usedMB = totalMB - freeMB;
        long maxMB = rt.maxMemory() / (1024 * 1024);

        System.out.printf("Memory — Used: %dMB | Free: %dMB | Total: %dMB | Max: %dMB%n",
                usedMB, freeMB, totalMB, maxMB);
    }
}
```

---

## Prevention Checklist

| Practice                                   | Why                                               |
|--------------------------------------------|----------------------------------------------------|
| Use try-with-resources                     | Ensures resource cleanup even on exceptions        |
| Remove listeners/callbacks                 | Prevents observer-pattern leaks                    |
| Call `ThreadLocal.remove()`                | Critical in thread-pool environments               |
| Avoid static mutable collections           | They live for the application's lifetime           |
| Use weak/soft references for caches        | Allows GC to reclaim under pressure               |
| Prefer lambdas over anonymous classes      | No implicit reference to enclosing instance        |
| Close ClassLoaders                         | Prevents class-loader leaks in plugin systems      |
| Review `HashMap`/`HashSet` key contracts   | Mutable keys can cause "lost" entries              |

---

## Real-World Example: HashMap Key Leak

```java
// ❌ Anti-pattern: mutable key in HashMap
public class MutableKey {
    private int id;

    public MutableKey(int id) { this.id = id; }

    public void setId(int id) { this.id = id; }

    @Override
    public int hashCode() { return id; }

    @Override
    public boolean equals(Object o) {
        return o instanceof MutableKey && ((MutableKey) o).id == this.id;
    }
}

Map<MutableKey, String> map = new HashMap<>();
MutableKey key = new MutableKey(1);
map.put(key, "value");

key.setId(2);                   // Hash code changes!
map.get(key);                   // Returns null — can't find it
map.get(new MutableKey(1));     // Returns null — hash bucket mismatch
// Entry is stuck in the map — can't be retrieved or removed = leak!
```

```java
// ✅ Fix: use immutable keys
public record ImmutableKey(int id) { }

Map<ImmutableKey, String> map = new HashMap<>();
map.put(new ImmutableKey(1), "value");
map.get(new ImmutableKey(1));   // Returns "value" — always consistent
```

---

## Interview Tips
- Memory leaks in Java are not "forgotten `free()` calls" — they are **unintended strong references**
- The most common leaks in enterprise apps: `ThreadLocal`, listeners, and static caches
- Heap dumps are the definitive tool for diagnosing leaks — look for the dominator tree
- A rising baseline after GC cycles is the telltale sign of a leak

---

[← Q2: Object Lifecycle](./q02-object-lifecycle.md) | [Next: Q4 - Weak References →](./q04-weak-references.md)
