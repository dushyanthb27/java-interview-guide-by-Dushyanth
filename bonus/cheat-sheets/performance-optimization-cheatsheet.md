# Performance Optimization - Cheat Sheet

Quick reference for identifying and fixing Java performance bottlenecks.

## 🎯 Performance Optimization Process

```
1. Measure (Profile)
   ↓
2. Identify Bottleneck
   ↓
3. Analyze Root Cause
   ↓
4. Implement Fix
   ↓
5. Verify Improvement
   ↓
6. Repeat
```

## 📊 Profiling Tools

### JVM Built-in Tools
```bash
# List Java processes
jps -l

# Thread dump
jstack <pid> > threads.txt

# Heap dump
jmap -dump:live,file=heap.hprof <pid>

# Heap analysis
jmap -heap <pid>
jmap -histo <pid>

# GC statistics
jstat -gc <pid> 1000        # Every 1 second
jstat -gcutil <pid> 1000    # Utilization
jstat -gccause <pid>        # Last GC cause

# Flight Recorder (Java 11+)
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jcmd <pid> JFR.dump filename=recording.jfr
jcmd <pid> JFR.stop
```

### Third-party Tools
```
- JProfiler: Commercial, comprehensive profiling
- YourKit: Commercial, low-overhead profiling
- Async Profiler: Open source, low-overhead
- Java Mission Control: Free, flight recorder analysis
- VisualVM: Free, basic profiling
```

## 💾 Memory Optimization

### Identify Memory Leaks
```java
// Memory leak pattern - static collection
public class Cache {
    private static List<String> cache = new ArrayList<>();
    
    public void add(String item) {
        cache.add(item);  // Never removed - memory leak
    }
}

// Fix: Use bounded cache
public class Cache {
    private final Map<String, String> cache;
    
    public Cache(int maxSize) {
        cache = new LinkedHashMap<String, String>(16, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry eldest) {
                return size() > maxSize;
            }
        };
    }
}

// Or use weak references
private WeakHashMap<String, String> cache = new WeakHashMap<>();
```

### Object Creation Optimization
```java
// ❌ Wrong - creates many objects
for (int i = 0; i < 1000000; i++) {
    String str = new String("value");
    process(str);
}

// ✅ Correct - reuse objects
String value = "value";
for (int i = 0; i < 1000000; i++) {
    process(value);
}

// ❌ Wrong - boxing overhead
List<Integer> numbers = new ArrayList<>();
for (int i = 0; i < 1000000; i++) {
    numbers.add(i);  // Boxing: Integer.valueOf(i)
}

// ✅ Correct - use primitive arrays
int[] numbers = new int[1000000];
for (int i = 0; i < 1000000; i++) {
    numbers[i] = i;
}
```

### String Optimization
```java
// ❌ Wrong - creates many strings
String result = "";
for (String item : items) {
    result += item;  // Creates new string each iteration
}

// ✅ Correct - use StringBuilder
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item);
}
String result = sb.toString();

// ❌ Wrong - string concatenation in loop
for (int i = 0; i < 1000; i++) {
    System.out.println("Item: " + i);  // String concatenation
}

// ✅ Correct - use formatted strings
for (int i = 0; i < 1000; i++) {
    System.out.printf("Item: %d%n", i);
}
```

### Collection Optimization
```java
// ❌ Wrong - ArrayList with frequent inserts at beginning
List<String> list = new ArrayList<>();
for (String item : items) {
    list.add(0, item);  // O(n) operation
}

// ✅ Correct - use LinkedList for frequent inserts
List<String> list = new LinkedList<>();
for (String item : items) {
    list.addFirst(item);  // O(1) operation
}

// ❌ Wrong - HashMap with many collisions
Map<String, Integer> map = new HashMap<>();
// Add many items with similar hash codes

// ✅ Correct - pre-size HashMap
Map<String, Integer> map = new HashMap<>(expectedSize * 4 / 3);

// ❌ Wrong - iterating HashMap values
for (String key : map.keySet()) {
    String value = map.get(key);  // Extra lookup
}

// ✅ Correct - iterate entries
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}
```

## ⚡ CPU Optimization

### Algorithm Optimization
```java
// ❌ Wrong - O(n²) algorithm
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        if (array[i] == array[j]) {
            // Found duplicate
        }
    }
}

// ✅ Correct - O(n) with HashSet
Set<Integer> seen = new HashSet<>();
for (int num : array) {
    if (seen.contains(num)) {
        // Found duplicate
    }
    seen.add(num);
}
```

### Loop Optimization
```java
// ❌ Wrong - method call in loop condition
for (int i = 0; i < list.size(); i++) {
    process(list.get(i));
}

// ✅ Correct - cache size
int size = list.size();
for (int i = 0; i < size; i++) {
    process(list.get(i));
}

// ❌ Wrong - unnecessary object creation
for (int i = 0; i < 1000000; i++) {
    Integer obj = new Integer(i);  // Boxing
}

// ✅ Correct - use primitives
for (int i = 0; i < 1000000; i++) {
    // Use primitive int
}
```

### Caching
```java
// ❌ Wrong - recalculate expensive operation
for (String item : items) {
    int length = item.length();
    int hash = item.hashCode();
    process(length, hash);
}

// ✅ Correct - cache results
int length = item.length();
int hash = item.hashCode();
for (String item : items) {
    process(length, hash);
}

// Memoization pattern
private Map<Integer, Integer> cache = new HashMap<>();

public int fibonacci(int n) {
    if (cache.containsKey(n)) {
        return cache.get(n);
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
```

## 🔄 Concurrency Optimization

### Lock Optimization
```java
// ❌ Wrong - lock entire method
public synchronized void process() {
    setup();
    criticalSection();
    cleanup();
}

// ✅ Correct - lock only critical section
public void process() {
    setup();
    synchronized (this) {
        criticalSection();
    }
    cleanup();
}

// ❌ Wrong - coarse-grained locking
private final Object lock = new Object();
private int counter1, counter2;

public synchronized void increment() {
    counter1++;
    counter2++;
}

// ✅ Correct - fine-grained locking
private final Object lock1 = new Object();
private final Object lock2 = new Object();
private int counter1, counter2;

public void increment() {
    synchronized (lock1) {
        counter1++;
    }
    synchronized (lock2) {
        counter2++;
    }
}
```

### Concurrent Collections
```java
// ❌ Wrong - synchronized wrapper
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());

// ✅ Correct - concurrent collection
Map<String, Integer> map = new ConcurrentHashMap<>();

// ❌ Wrong - CopyOnWriteArrayList for frequent writes
List<String> list = new CopyOnWriteArrayList<>();
for (int i = 0; i < 1000000; i++) {
    list.add("item");  // O(n) operation
}

// ✅ Correct - ArrayList with synchronization
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

### Parallel Processing
```java
// ❌ Wrong - sequential for large dataset
List<Integer> numbers = generateMillionNumbers();
List<Integer> doubled = numbers.stream()
    .map(n -> n * 2)
    .collect(Collectors.toList());

// ✅ Correct - parallel for large dataset
List<Integer> doubled = numbers.parallelStream()
    .map(n -> n * 2)
    .collect(Collectors.toList());

// ❌ Wrong - parallel for small dataset (overhead)
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> doubled = numbers.parallelStream()
    .map(n -> n * 2)
    .collect(Collectors.toList());

// ✅ Correct - sequential for small dataset
List<Integer> doubled = numbers.stream()
    .map(n -> n * 2)
    .collect(Collectors.toList());
```

## 📈 GC Optimization

### Reduce GC Pressure
```java
// ❌ Wrong - creates many temporary objects
public String processData(List<String> items) {
    String result = "";
    for (String item : items) {
        result += item + ",";  // Creates new string each iteration
    }
    return result;
}

// ✅ Correct - minimize object creation
public String processData(List<String> items) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < items.size(); i++) {
        if (i > 0) sb.append(",");
        sb.append(items.get(i));
    }
    return sb.toString();
}
```

### Monitor GC
```bash
# Enable GC logging
java -Xlog:gc*:file=gc.log:time,level,tags -jar app.jar

# Analyze GC logs
# Look for:
# - Frequency of GC pauses
# - Duration of pauses
# - Full GC frequency
# - Heap utilization trends
```

## 🔍 Common Performance Issues

### Issue: High CPU Usage
```
Diagnosis:
1. Use jstack to get thread dump
2. Look for busy-waiting loops
3. Check for inefficient algorithms
4. Profile with async-profiler

Solutions:
- Optimize algorithm (reduce complexity)
- Add caching
- Use more efficient data structures
- Reduce lock contention
```

### Issue: High Memory Usage
```
Diagnosis:
1. Use jmap to dump heap
2. Analyze with Eclipse MAT or YourKit
3. Look for memory leaks
4. Check for large collections

Solutions:
- Fix memory leaks
- Use object pooling
- Reduce collection sizes
- Use weak references
```

### Issue: Long GC Pauses
```
Diagnosis:
1. Enable GC logging
2. Analyze GC logs
3. Check heap size
4. Profile with Flight Recorder

Solutions:
- Increase heap size
- Reduce object creation
- Use G1GC or ZGC
- Tune GC parameters
```

### Issue: High Latency
```
Diagnosis:
1. Use Flight Recorder
2. Check for GC pauses
3. Profile lock contention
4. Check network I/O

Solutions:
- Reduce GC pauses
- Optimize locks
- Use async I/O
- Add caching
```

## 📊 Performance Benchmarking

### JMH (Java Microbenchmark Harness)
```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Fork(1)
@Warmup(iterations = 5)
@Measurement(iterations = 5)
public class StringConcatenationBenchmark {
    
    @Benchmark
    public String stringConcat() {
        String result = "";
        for (int i = 0; i < 100; i++) {
            result += "x";
        }
        return result;
    }
    
    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) {
            sb.append("x");
        }
        return sb.toString();
    }
}

// Run: java -jar benchmarks.jar
```

## 🎓 Interview Tips

- **Measure first**: Don't optimize without profiling
- **Know the tools**: jstack, jmap, jstat, Flight Recorder
- **Understand trade-offs**: Speed vs memory, simplicity vs performance
- **Common bottlenecks**: GC, locks, I/O, algorithms
- **Optimization order**: Algorithm > Data structures > JVM tuning

---

**Related**: See JVM Flags cheat sheet for GC tuning and monitoring
