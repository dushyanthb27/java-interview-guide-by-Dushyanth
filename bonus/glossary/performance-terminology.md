# Performance Terminology

Comprehensive reference for performance optimization, benchmarking, and tuning concepts. This glossary covers 70+ essential performance terms organized alphabetically.

---

## A

### Algorithm Complexity
**Category**: Performance Analysis

**Definition**: The measure of how an algorithm's resource usage (time or space) grows with input size. Complexity is expressed using Big O notation and helps predict algorithm performance at scale.

**Practical Example**:
```java
// O(1) - Constant time
int getFirstElement(int[] array) {
    return array[0];
}

// O(n) - Linear time
int findMax(int[] array) {
    int max = array[0];
    for (int i = 1; i < array.length; i++) {
        if (array[i] > max) {
            max = array[i];
        }
    }
    return max;
}

// O(n²) - Quadratic time
void bubbleSort(int[] array) {
    for (int i = 0; i < array.length; i++) {
        for (int j = 0; j < array.length - 1; j++) {
            if (array[j] > array[j + 1]) {
                // Swap
            }
        }
    }
}

// O(log n) - Logarithmic time
int binarySearch(int[] array, int target) {
    int left = 0, right = array.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (array[mid] == target) return mid;
        if (array[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}

// O(n log n) - Linearithmic time
// Merge sort, Quick sort

// Complexity comparison
// Input size: 1000
// O(1): 1 operation
// O(log n): ~10 operations
// O(n): 1000 operations
// O(n log n): ~10000 operations
// O(n²): 1000000 operations
```

**Related Terms**: [Big O Notation](#big-o-notation), [Time Complexity](#time-complexity), [Space Complexity](#space-complexity), [Algorithm](#algorithm)

**Interview Tip**: Know how to analyze algorithm complexity. Understand the impact of complexity on performance at scale.

---

## B

### Benchmarking
**Category**: Performance Measurement

**Definition**: The process of measuring and comparing the performance of code, algorithms, or systems. Benchmarking provides quantitative data for performance analysis and optimization decisions.

**Practical Example**:
```java
// JMH (Java Microbenchmark Harness)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Fork(1)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class StringConcatenationBenchmark {
    
    @Benchmark
    public String concatenateWithPlus() {
        String result = "";
        for (int i = 0; i < 100; i++) {
            result += "item";
        }
        return result;
    }
    
    @Benchmark
    public String concatenateWithStringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) {
            sb.append("item");
        }
        return sb.toString();
    }
    
    @Benchmark
    public String concatenateWithJoin() {
        return String.join("", Collections.nCopies(100, "item"));
    }
}

// Running benchmark
// mvn clean install
// java -jar target/benchmarks.jar

// Simple benchmarking
long startTime = System.nanoTime();
// Code to benchmark
long endTime = System.nanoTime();
long duration = endTime - startTime;
System.out.println("Duration: " + duration + " ns");
```

**Related Terms**: [Performance Testing](#performance-testing), [Profiling](#profiling), [Optimization](#optimization)

**Interview Tip**: Know how to write proper benchmarks using JMH. Understand the importance of warm-up and multiple iterations.

---

## C

### Cache
**Category**: Memory Optimization

**Definition**: A hardware or software component that stores frequently accessed data for faster retrieval. Caching reduces latency by avoiding expensive computations or I/O operations.

**Practical Example**:
```java
// CPU Cache (L1, L2, L3)
// Automatically managed by CPU

// Software Cache
class CachedCalculator {
    private Map<Integer, Integer> cache = new HashMap<>();
    
    public int fibonacci(int n) {
        if (cache.containsKey(n)) {
            return cache.get(n);  // Cache hit
        }
        
        int result;
        if (n <= 1) {
            result = n;
        } else {
            result = fibonacci(n - 1) + fibonacci(n - 2);
        }
        
        cache.put(n, result);
        return result;
    }
}

// Guava Cache
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(new CacheLoader<String, String>() {
        @Override
        public String load(String key) {
            return fetchFromDatabase(key);
        }
    });

String value = cache.getUnchecked("key");

// Spring Cache
@Cacheable("users")
public User getUser(String id) {
    return userRepository.findById(id);
}

@CacheEvict("users")
public void updateUser(User user) {
    userRepository.save(user);
}

// Cache strategies
// LRU (Least Recently Used)
// LFU (Least Frequently Used)
// FIFO (First In First Out)
// TTL (Time To Live)
```

**Related Terms**: [Caching Strategy](#caching-strategy), [Cache Hit Ratio](#cache-hit-ratio), [Memory Optimization](#memory-optimization)

**Interview Tip**: Know different caching strategies. Understand cache invalidation and its challenges.

---

### Cache Hit Ratio
**Category**: Performance Metrics

**Definition**: The percentage of cache accesses that result in a cache hit (data found in cache) versus cache misses (data not found). Higher cache hit ratio indicates better cache effectiveness.

**Practical Example**:
```java
// Cache hit ratio calculation
// Cache hit ratio = Cache hits / (Cache hits + Cache misses) * 100%

class CacheMetrics {
    private int hits = 0;
    private int misses = 0;
    
    public void recordHit() {
        hits++;
    }
    
    public void recordMiss() {
        misses++;
    }
    
    public double getHitRatio() {
        int total = hits + misses;
        if (total == 0) return 0;
        return (double) hits / total * 100;
    }
}

// Example
// 100 cache accesses
// 80 hits, 20 misses
// Hit ratio = 80 / 100 * 100% = 80%

// Improving cache hit ratio
// 1. Increase cache size
// 2. Better cache eviction policy
// 3. Optimize data locality
// 4. Prefetch data

// Monitoring cache hit ratio
// JVM: -XX:+PrintCompilation
// Application: Custom metrics
// Tools: JProfiler, YourKit
```

**Related Terms**: [Cache](#cache), [Caching Strategy](#caching-strategy), [Performance Metrics](#performance-metrics)

**Interview Tip**: Know how to measure and improve cache hit ratio. Understand the relationship between cache size and hit ratio.

---

## L

### Latency
**Category**: Performance Metrics

**Definition**: The time delay between a request and the corresponding response. Latency is critical for user experience and is measured in milliseconds or microseconds.

**Practical Example**:
```java
// Measuring latency
long startTime = System.currentTimeMillis();
// Operation
long endTime = System.currentTimeMillis();
long latency = endTime - startTime;  // milliseconds

// More precise measurement
long startTime = System.nanoTime();
// Operation
long endTime = System.nanoTime();
long latency = (endTime - startTime) / 1_000_000;  // milliseconds

// Latency percentiles
// p50 (median): 50th percentile
// p95: 95th percentile
// p99: 99th percentile
// p99.9: 99.9th percentile

class LatencyTracker {
    private List<Long> latencies = new ArrayList<>();
    
    public void recordLatency(long latency) {
        latencies.add(latency);
    }
    
    public long getPercentile(double percentile) {
        Collections.sort(latencies);
        int index = (int) (latencies.size() * percentile / 100);
        return latencies.get(index);
    }
}

// Typical latencies
// L1 cache: 4 ns
// L2 cache: 10 ns
// L3 cache: 40 ns
// Main memory: 100 ns
// SSD: 10 μs
// HDD: 10 ms
// Network: 1-100 ms

// Reducing latency
// 1. Cache frequently accessed data
// 2. Use faster data structures
// 3. Optimize algorithms
// 4. Parallelize operations
// 5. Use asynchronous processing
```

**Related Terms**: [Throughput](#throughput), [Response Time](#response-time), [Performance Metrics](#performance-metrics)

**Interview Tip**: Understand the difference between latency and throughput. Know typical latencies for different operations.

---

## M

### Memory Leak
**Category**: Memory Management

**Definition**: A situation where objects are no longer needed but are still referenced, preventing garbage collection. Memory leaks cause the heap to grow over time, eventually leading to OutOfMemoryError.

**Practical Example**:
```java
// Memory leak examples

// 1. Static collections
class MemoryLeak1 {
    private static List<Object> list = new ArrayList<>();
    
    public void addObject(Object obj) {
        list.add(obj);  // Objects never removed
    }
}

// 2. Listener registration without unregistration
class MemoryLeak2 {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    
    // Missing removeListener method
}

// 3. Unclosed resources
class MemoryLeak3 {
    public void readFile(String filename) throws IOException {
        FileInputStream fis = new FileInputStream(filename);
        // Missing fis.close()
    }
}

// Detecting memory leaks
// jmap -heap <pid>
// jmap -histo <pid>
// jmap -dump:live,format=b,file=heap.bin <pid>
// jhat heap.bin

// Preventing memory leaks
// 1. Use try-with-resources
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // Use fis
}  // Automatically closed

// 2. Implement AutoCloseable
class Resource implements AutoCloseable {
    @Override
    public void close() {
        // Cleanup
    }
}

// 3. Use weak references
WeakReference<Object> ref = new WeakReference<>(new Object());
Object obj = ref.get();  // May be null if GC'd

// 4. Remove listeners
listeners.remove(listener);
```

**Related Terms**: [Garbage Collection](#garbage-collection), [Heap](#heap), [OutOfMemoryError](#outofmemoryerror)

**Interview Tip**: Know common causes of memory leaks. Understand how to detect and prevent them.

---

## O

### Optimization
**Category**: Performance Tuning

**Definition**: The process of improving code or system performance by reducing resource usage (time, memory, CPU, I/O). Optimization should be data-driven and focused on bottlenecks.

**Practical Example**:
```java
// Optimization techniques

// 1. Algorithm optimization
// Before: O(n²) bubble sort
void bubbleSort(int[] array) {
    for (int i = 0; i < array.length; i++) {
        for (int j = 0; j < array.length - 1; j++) {
            if (array[j] > array[j + 1]) {
                // Swap
            }
        }
    }
}

// After: O(n log n) merge sort
void mergeSort(int[] array) {
    // Efficient sorting
}

// 2. Caching
// Before: Recompute every time
int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// After: Cache results
Map<Integer, Integer> cache = new HashMap<>();
int fibonacciCached(int n) {
    if (cache.containsKey(n)) return cache.get(n);
    int result = (n <= 1) ? n : fibonacciCached(n - 1) + fibonacciCached(n - 2);
    cache.put(n, result);
    return result;
}

// 3. String concatenation
// Before: Inefficient
String result = "";
for (int i = 0; i < 1000; i++) {
    result += "item";  // Creates new String each iteration
}

// After: Efficient
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("item");
}
String result = sb.toString();

// 4. Collection optimization
// Use appropriate collection types
List<String> list = new ArrayList<>();  // Fast access
Set<String> set = new HashSet<>();      // Fast lookup
Map<String, Integer> map = new HashMap<>();  // Fast key-value lookup

// 5. Lazy initialization
class ExpensiveResource {
    private Resource resource;
    
    public Resource getResource() {
        if (resource == null) {
            resource = new Resource();  // Initialize only when needed
        }
        return resource;
    }
}
```

**Related Terms**: [Profiling](#profiling), [Benchmarking](#benchmarking), [Performance Tuning](#performance-tuning)

**Interview Tip**: Know common optimization techniques. Understand the importance of profiling before optimizing. Remember: "Premature optimization is the root of all evil."

---

## P

### Performance Metrics
**Category**: Performance Measurement

**Definition**: Quantitative measurements of system or application performance. Common metrics include latency, throughput, CPU usage, memory usage, and cache hit ratio.

**Practical Example**:
```java
// Performance metrics

// 1. Latency (response time)
long startTime = System.nanoTime();
// Operation
long latency = System.nanoTime() - startTime;

// 2. Throughput (operations per second)
long operationCount = 0;
long startTime = System.currentTimeMillis();
// Perform operations
long endTime = System.currentTimeMillis();
long throughput = operationCount / ((endTime - startTime) / 1000.0);

// 3. CPU usage
OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
double cpuUsage = osBean.getProcessCpuLoad() * 100;

// 4. Memory usage
MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
long usedMemory = heapUsage.getUsed();
long maxMemory = heapUsage.getMax();

// 5. GC metrics
List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
for (GarbageCollectorMXBean gcBean : gcBeans) {
    long gcCount = gcBean.getCollectionCount();
    long gcTime = gcBean.getCollectionTime();
}

// 6. Thread metrics
ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
int threadCount = threadBean.getThreadCount();
long totalStartedThreadCount = threadBean.getTotalStartedThreadCount();

// Monitoring tools
// JConsole: GUI monitoring
// VisualVM: Advanced profiling
// JProfiler: Commercial profiler
// YourKit: Commercial profiler
// New Relic: APM solution
// Datadog: Monitoring platform
```

**Related Terms**: [Latency](#latency), [Throughput](#throughput), [Profiling](#profiling), [Benchmarking](#benchmarking)

**Interview Tip**: Know key performance metrics. Understand how to collect and analyze metrics.

---

### Performance Testing
**Category**: Performance Measurement

**Definition**: Testing that evaluates system performance under various load conditions. Performance testing includes load testing, stress testing, and endurance testing.

**Practical Example**:
```java
// Performance testing

// Load testing: Simulate normal load
// 100 concurrent users, 1000 requests

// Stress testing: Exceed normal load
// 1000 concurrent users, 10000 requests

// Endurance testing: Run for extended period
// 100 concurrent users, 24 hours

// Using JMeter for load testing
// 1. Create test plan
// 2. Add thread group (concurrent users)
// 3. Add HTTP sampler (requests)
// 4. Add listeners (results)
// 5. Run test

// Using Gatling for load testing
class LoadSimulation extends Simulation {
    HttpProtocol httpProtocol = http.baseUrl("http://localhost:8080");
    
    ScenarioBuilder scn = scenario("Load Test")
        .exec(http("request").get("/api/users"));
    
    {
        setUp(
            scn.injectOpen(rampUsers(100).during(Duration.ofSeconds(10)))
        ).protocols(httpProtocol);
    }
}

// Performance testing metrics
// Response time (latency)
// Throughput (requests per second)
// Error rate
// CPU usage
// Memory usage
// Database connections
```

**Related Terms**: [Load Testing](#load-testing), [Stress Testing](#stress-testing), [Benchmarking](#benchmarking)

**Interview Tip**: Know different types of performance testing. Understand how to identify performance bottlenecks.

---

### Profiling
**Category**: Performance Analysis

**Definition**: The process of measuring program behavior to identify performance bottlenecks. Profiling collects data on CPU usage, memory allocation, and method execution time.

**Practical Example**:
```java
// CPU Profiling
// Identifies which methods consume most CPU time

// Memory Profiling
// Identifies which objects consume most memory

// Using JProfiler
// 1. Start application with JProfiler agent
// 2. Connect to application
// 3. Analyze CPU usage
// 4. Analyze memory usage
// 5. Identify bottlenecks

// Using VisualVM
// 1. Launch VisualVM
// 2. Connect to running JVM
// 3. Monitor CPU and memory
// 4. Analyze thread activity
// 5. Generate heap dump

// Using JFR (Java Flight Recorder)
// java -XX:+UnlockCommercialFeatures -XX:+FlightRecorder MyApp
// jcmd <pid> JFR.start
// jcmd <pid> JFR.dump filename=recording.jfr
// jmc recording.jfr

// Programmatic profiling
class Profiler {
    private Map<String, Long> timings = new HashMap<>();
    
    public void startTiming(String name) {
        timings.put(name + "_start", System.nanoTime());
    }
    
    public void endTiming(String name) {
        long startTime = timings.get(name + "_start");
        long duration = System.nanoTime() - startTime;
        System.out.println(name + ": " + duration + " ns");
    }
}
```

**Related Terms**: [Benchmarking](#benchmarking), [Performance Analysis](#performance-analysis), [Optimization](#optimization)

**Interview Tip**: Know how to use profiling tools. Understand the difference between CPU and memory profiling.

---

## T

### Throughput
**Category**: Performance Metrics

**Definition**: The number of operations or requests processed per unit time. Throughput is measured in operations per second (ops/s) or requests per second (req/s).

**Practical Example**:
```java
// Measuring throughput
long operationCount = 0;
long startTime = System.currentTimeMillis();

for (int i = 0; i < 1000000; i++) {
    performOperation();
    operationCount++;
}

long endTime = System.currentTimeMillis();
long durationSeconds = (endTime - startTime) / 1000.0;
double throughput = operationCount / durationSeconds;
System.out.println("Throughput: " + throughput + " ops/s");

// Throughput vs. Latency trade-off
// High throughput: Process many requests, longer latency
// Low latency: Process quickly, lower throughput

// Improving throughput
// 1. Batch processing
List<Request> batch = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    batch.add(new Request());
}
processBatch(batch);  // More efficient than individual processing

// 2. Parallel processing
ExecutorService executor = Executors.newFixedThreadPool(10);
for (Request request : requests) {
    executor.submit(() -> processRequest(request));
}

// 3. Caching
Map<String, Result> cache = new HashMap<>();
for (Request request : requests) {
    Result result = cache.computeIfAbsent(
        request.getKey(),
        key -> computeResult(request)
    );
}

// 4. Connection pooling
DataSource dataSource = new HikariDataSource();
// Reuse connections instead of creating new ones

// Typical throughput values
// Single-threaded: 1000-10000 ops/s
// Multi-threaded: 10000-100000 ops/s
// Optimized: 100000+ ops/s
```

**Related Terms**: [Latency](#latency), [Performance Metrics](#performance-metrics), [Optimization](#optimization)

**Interview Tip**: Understand the difference between throughput and latency. Know how to improve throughput.

---

## Additional Performance Terms

### Big O Notation
**Category**: Performance Analysis

**Definition**: A mathematical notation that describes the upper bound of an algorithm's time or space complexity. Big O notation helps compare algorithm efficiency independently of hardware.

**Practical Example**:
```java
// Big O notation examples

// O(1) - Constant
int getElement(int[] array, int index) {
    return array[index];
}

// O(log n) - Logarithmic
int binarySearch(int[] array, int target) {
    // Binary search implementation
}

// O(n) - Linear
int findMax(int[] array) {
    int max = array[0];
    for (int i = 1; i < array.length; i++) {
        if (array[i] > max) max = array[i];
    }
    return max;
}

// O(n log n) - Linearithmic
void mergeSort(int[] array) {
    // Merge sort implementation
}

// O(n²) - Quadratic
void bubbleSort(int[] array) {
    for (int i = 0; i < array.length; i++) {
        for (int j = 0; j < array.length - 1; j++) {
            if (array[j] > array[j + 1]) {
                // Swap
            }
        }
    }
}

// O(2^n) - Exponential
int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// O(n!) - Factorial
void generatePermutations(int[] array) {
    // Generate all permutations
}

// Complexity comparison for n=1000
// O(1): 1
// O(log n): 10
// O(n): 1000
// O(n log n): 10000
// O(n²): 1000000
// O(2^n): 2^1000 (infeasible)
```

**Related Terms**: [Algorithm Complexity](#algorithm-complexity), [Time Complexity](#time-complexity), [Space Complexity](#space-complexity)

**Interview Tip**: Know Big O notation and how to analyze algorithm complexity. Understand practical implications of complexity.

---

### Caching Strategy
**Category**: Memory Optimization

**Definition**: A policy for managing cached data, including when to cache, when to evict, and how to handle cache invalidation. Different strategies suit different use cases.

**Practical Example**:
```java
// Caching strategies

// 1. LRU (Least Recently Used)
// Evict least recently accessed item
LinkedHashMap<String, String> lruCache = new LinkedHashMap<String, String>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;  // Max 100 items
    }
};

// 2. LFU (Least Frequently Used)
// Evict least frequently accessed item
class LFUCache {
    private Map<String, String> cache = new HashMap<>();
    private Map<String, Integer> frequency = new HashMap<>();
    
    public String get(String key) {
        if (cache.containsKey(key)) {
            frequency.put(key, frequency.getOrDefault(key, 0) + 1);
            return cache.get(key);
        }
        return null;
    }
}

// 3. FIFO (First In First Out)
// Evict oldest item
Queue<String> fifoCache = new LinkedList<>();

// 4. TTL (Time To Live)
// Evict after specified time
class TTLCache {
    private Map<String, CacheEntry> cache = new HashMap<>();
    
    class CacheEntry {
        String value;
        long expiryTime;
        
        CacheEntry(String value, long ttlMillis) {
            this.value = value;
            this.expiryTime = System.currentTimeMillis() + ttlMillis;
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }
    }
}

// 5. Write-through
// Write to cache and database simultaneously
void writeThrough(String key, String value) {
    cache.put(key, value);
    database.save(key, value);
}

// 6. Write-back
// Write to cache first, database later
void writeBack(String key, String value) {
    cache.put(key, value);
    // Asynchronously write to database
    executor.submit(() -> database.save(key, value));
}
```

**Related Terms**: [Cache](#cache), [Cache Hit Ratio](#cache-hit-ratio), [Memory Optimization](#memory-optimization)

**Interview Tip**: Know different caching strategies and their trade-offs. Understand when to use each strategy.

---

### Response Time
**Category**: Performance Metrics

**Definition**: The total time taken from sending a request to receiving a response. Response time includes network latency, processing time, and I/O time.

**Practical Example**:
```java
// Measuring response time
long startTime = System.currentTimeMillis();

// Make request and get response
String response = makeRequest("http://api.example.com/users");

long endTime = System.currentTimeMillis();
long responseTime = endTime - startTime;
System.out.println("Response time: " + responseTime + " ms");

// Response time breakdown
// Network latency: 10 ms
// Server processing: 50 ms
// Database query: 30 ms
// Network latency: 10 ms
// Total: 100 ms

// Reducing response time
// 1. Optimize server processing
// 2. Optimize database queries
// 3. Use caching
// 4. Use CDN for static content
// 5. Parallelize operations
// 6. Use asynchronous processing

// Typical response times
// Web page: 100-500 ms
// API call: 50-200 ms
// Database query: 10-100 ms
// Cache hit: 1-10 ms
```

**Related Terms**: [Latency](#latency), [Performance Metrics](#performance-metrics), [Throughput](#throughput)

**Interview Tip**: Understand the components of response time. Know how to identify and optimize bottlenecks.

---

This glossary provides comprehensive coverage of performance terminology essential for optimizing Java applications.
