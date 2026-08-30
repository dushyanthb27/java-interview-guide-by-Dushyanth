# Memory Management Optimization Techniques

[Home](../../README.md) > [Performance Optimization](./README.md) > Memory Management

---

## Overview

Memory optimization in Java involves reducing allocation rates, leveraging JVM optimizations like escape analysis and scalar replacement, and understanding when to use off-heap memory. Efficient memory usage reduces GC pressure and improves overall performance.

---

## 1. Object Allocation Rates

High allocation rates lead to frequent GC pauses. The key metric is **allocation rate** (MB/s allocated in the young generation).

```java
// BAD: High allocation rate - creates millions of temporary objects
public List<String> processRecords(List<Record> records) {
    List<String> results = new ArrayList<>();
    for (Record record : records) {
        // Each iteration creates: formatted string, temp StringBuilder,
        // Date object, DateFormat internal objects...
        String formatted = String.format("Name: %s, Date: %s",
            record.getName(),
            new SimpleDateFormat("yyyy-MM-dd").format(record.getDate()));
        results.add(formatted);
    }
    return results;
}

// GOOD: Reduced allocation rate
public List<String> processRecords(List<Record> records) {
    List<String> results = new ArrayList<>(records.size()); // pre-sized
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    StringBuilder sb = new StringBuilder(128);

    for (Record record : records) {
        sb.setLength(0); // reuse StringBuilder
        sb.append("Name: ").append(record.getName())
          .append(", Date: ").append(record.getDate().format(formatter));
        results.add(sb.toString());
    }
    return results;
}
```

### Measuring Allocation Rate

```bash
# Using JFR to track allocation
java -XX:StartFlightRecording=filename=alloc.jfr,settings=profile \
     -jar app.jar

# GC log shows allocation in young gen
# [gc] GC(5) Pause Young 256M->32M(512M) 5.2ms
# Allocation rate = 256 - 32 = 224 MB per young GC
```

---

## 2. TLAB (Thread-Local Allocation Buffer)

Each thread gets a private allocation buffer in Eden space, eliminating synchronization for object allocation.

```bash
# TLAB is enabled by default; tuning options:
java -XX:+UseTLAB \                      # enabled by default
     -XX:TLABSize=512k \                 # initial TLAB size
     -XX:+ResizeTLAB \                   # allow dynamic resizing
     -XX:MinTLABSize=2k \                # minimum TLAB size
     -jar app.jar
```

```
Thread 1: [----TLAB----][  Eden  ]
Thread 2: [----TLAB----][  Eden  ]
Thread 3: [----TLAB----][  Eden  ]

Allocation within TLAB = bump pointer (no lock needed)
Allocation outside TLAB = CAS on shared Eden pointer (slower)
Objects > TLAB size = allocated directly in Eden with synchronization
```

```java
// TLAB-friendly allocation pattern
// Small, frequent allocations are fast (TLAB bump pointer)
public void efficientAllocation() {
    for (int i = 0; i < 1_000_000; i++) {
        Point p = new Point(i, i * 2); // small object, fits in TLAB
        process(p);
    }
}

// TLAB-unfriendly: large objects bypass TLAB
public void inefficientAllocation() {
    for (int i = 0; i < 1000; i++) {
        byte[] large = new byte[1024 * 1024]; // 1 MB, won't fit in TLAB
        process(large);
    }
}
```

---

## 3. Escape Analysis

The JIT compiler analyzes whether an object "escapes" the method/thread that created it. Non-escaping objects can be optimized.

```java
// Object does NOT escape - eligible for optimization
public int computeDistance() {
    Point p = new Point(3, 4);       // local object, never escapes
    return (int) Math.sqrt(p.x * p.x + p.y * p.y);
}
// JIT may eliminate the Point allocation entirely!

// Object DOES escape - must be heap-allocated
public Point createPoint() {
    Point p = new Point(3, 4);       // returned to caller = escapes
    return p;
}

// Object escapes to another thread - no optimization possible
public void startThread() {
    Point p = new Point(3, 4);
    new Thread(() -> System.out.println(p)).start(); // escapes to thread
}
```

### Escape Analysis Optimizations

```bash
# Escape analysis is enabled by default (Java 6+)
java -XX:+DoEscapeAnalysis -jar app.jar    # default: on
java -XX:-DoEscapeAnalysis -jar app.jar    # disable for testing
```

---

## 4. Scalar Replacement

When escape analysis determines an object doesn't escape, the JIT can replace it with its individual fields (scalars) on the stack.

```java
// Before scalar replacement (conceptual)
public int sumCoordinates() {
    Point p = new Point(10, 20);  // heap allocation
    return p.x + p.y;
}

// After scalar replacement (what JIT actually does)
public int sumCoordinates() {
    int p_x = 10;  // field on stack (no heap allocation!)
    int p_y = 20;  // field on stack
    return p_x + p_y;
}
```

```java
// Benchmark demonstrating scalar replacement
public class ScalarReplacementDemo {

    static class ComplexNumber {
        final double real;
        final double imaginary;

        ComplexNumber(double real, double imaginary) {
            this.real = real;
            this.imaginary = imaginary;
        }

        ComplexNumber multiply(ComplexNumber other) {
            return new ComplexNumber(
                this.real * other.real - this.imaginary * other.imaginary,
                this.real * other.imaginary + this.imaginary * other.real
            );
        }

        double magnitude() {
            return Math.sqrt(real * real + imaginary * imaginary);
        }
    }

    // The temporary ComplexNumber in multiply() is scalar-replaced
    // No heap allocation occurs in the hot loop
    public static double mandelbrotIteration(double cr, double ci, int maxIter) {
        ComplexNumber z = new ComplexNumber(0, 0);
        ComplexNumber c = new ComplexNumber(cr, ci);
        for (int i = 0; i < maxIter; i++) {
            z = z.multiply(z);  // scalar-replaced after JIT
            z = new ComplexNumber(z.real + c.real, z.imaginary + c.imaginary);
            if (z.magnitude() > 2.0) return i;
        }
        return maxIter;
    }
}
```

```bash
# Verify scalar replacement is happening
java -XX:+PrintEscapeAnalysis \
     -XX:+PrintEliminateAllocations \
     -jar app.jar
```

---

## 5. Object Reuse vs Creation

### When to Reuse Objects

```java
// Object Pooling - useful for expensive-to-create objects
public class ConnectionPool {
    private final BlockingQueue<Connection> pool;

    public ConnectionPool(int size, String url) throws SQLException {
        pool = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            pool.offer(DriverManager.getConnection(url));
        }
    }

    public Connection borrow() throws InterruptedException {
        return pool.take();
    }

    public void release(Connection conn) {
        pool.offer(conn);
    }
}

// ThreadLocal reuse for non-thread-safe objects
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatDate(Date date) {
    return DATE_FORMAT.get().format(date); // reused per thread
}
```

### When to Create New Objects

```java
// Small immutable objects - JIT optimizes these well
// DON'T pool these; creation is faster than pool overhead
public record Point(int x, int y) {} // value-like, short-lived

// DON'T reuse mutable objects across threads (thread-safety issues)
// DON'T pool objects that are cheap to create (POJO, wrappers)
```

### Decision Matrix

| Factor                  | Reuse (Pool) | Create New   |
|-------------------------|--------------|--------------|
| Creation cost           | High (DB conn)| Low (POJO)  |
| Object size             | Large        | Small        |
| Lifetime                | Long         | Short        |
| Thread safety           | Managed pool | Immutable    |
| GC pressure concern     | Yes          | JIT handles  |

---

## 6. Off-Heap Memory

```java
import java.nio.ByteBuffer;

public class OffHeapMemoryDemo {

    // Direct ByteBuffer - allocated outside Java heap
    public static void directBufferExample() {
        // Allocated in native memory, not subject to GC pauses
        ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024 * 1024); // 1 MB

        // Write data
        directBuffer.putInt(42);
        directBuffer.putDouble(3.14);
        directBuffer.flip();

        // Read data
        int intVal = directBuffer.getInt();
        double doubleVal = directBuffer.getDouble();

        System.out.printf("Int: %d, Double: %.2f%n", intVal, doubleVal);
    }

    // Heap ByteBuffer - allocated in Java heap
    public static void heapBufferExample() {
        ByteBuffer heapBuffer = ByteBuffer.allocate(1024 * 1024); // 1 MB on heap
        // Subject to GC, but faster for small buffers
    }
}
```

### Direct vs Heap Buffers

| Aspect          | Direct (Off-Heap)         | Heap (On-Heap)          |
|-----------------|---------------------------|-------------------------|
| Allocation      | Slow (OS malloc)          | Fast (TLAB)             |
| Deallocation    | Slow (OS free)            | Fast (GC)               |
| I/O performance | Fast (zero-copy possible) | Slower (copy to native) |
| GC impact       | None                      | Subject to GC           |
| Best for        | Long-lived I/O buffers    | Short-lived data        |

```bash
# Monitor direct memory usage
java -XX:MaxDirectMemorySize=256m \
     -XX:NativeMemoryTracking=summary \
     -jar app.jar

# Print native memory summary
jcmd <pid> VM.native_memory summary
```

---

## 7. Value Types (Project Valhalla Preview)

```java
// Future Java (Project Valhalla) - value classes
// Inline/flattened in memory, no object header overhead

// Current: Each Point is a separate heap object (16+ bytes header)
record Point(int x, int y) {}  // 16 bytes header + 8 bytes data = 24 bytes

// Future (Valhalla value class): No header, inlined in arrays
// value record Point(int x, int y) {}  // only 8 bytes!

// Current workaround: Use primitive arrays instead of object arrays
public class FlattenedPoints {
    private final int[] xs;
    private final int[] ys;

    public FlattenedPoints(int capacity) {
        xs = new int[capacity]; // flat memory layout
        ys = new int[capacity]; // cache-friendly access
    }

    public void set(int index, int x, int y) {
        xs[index] = x;
        ys[index] = y;
    }

    public int getX(int index) { return xs[index]; }
    public int getY(int index) { return ys[index]; }
}
// Array of 1M points: 
//   Object array: ~24 MB (with headers + references)
//   Flat arrays:  ~8 MB  (just int data)
```

---

## 8. Memory-Efficient Patterns

```java
// Pattern 1: Flyweight - share immutable data
public class CharacterStyle {
    private static final Map<String, CharacterStyle> CACHE = new HashMap<>();

    private final String font;
    private final int size;
    private final Color color;

    public static CharacterStyle of(String font, int size, Color color) {
        String key = font + ":" + size + ":" + color.getRGB();
        return CACHE.computeIfAbsent(key,
            k -> new CharacterStyle(font, size, color));
    }
}

// Pattern 2: Compact data representation
public class CompactDate {
    // Instead of LocalDate (object header + 3 int fields = 28+ bytes)
    // Store as a single int: YYYYMMDD
    private final int packedDate;

    public CompactDate(int year, int month, int day) {
        this.packedDate = year * 10000 + month * 100 + day;
    }

    public int getYear()  { return packedDate / 10000; }
    public int getMonth() { return (packedDate / 100) % 100; }
    public int getDay()   { return packedDate % 100; }
}

// Pattern 3: Lazy initialization to defer memory usage
public class ExpensiveResource {
    private volatile byte[] data; // not allocated until needed

    public byte[] getData() {
        if (data == null) {
            synchronized (this) {
                if (data == null) {
                    data = loadExpensiveData(); // deferred allocation
                }
            }
        }
        return data;
    }
}
```

---

## Interview Tips

- Reducing allocation rate is the most impactful memory optimization
- TLAB makes small object allocation nearly free (bump pointer, no lock)
- Escape analysis + scalar replacement can eliminate heap allocations entirely
- Pool expensive objects (DB connections, threads), not cheap ones (POJOs)
- Off-heap memory avoids GC but requires manual lifecycle management
- Project Valhalla will bring value types to eliminate object header overhead
- Always profile allocation with JFR before optimizing; target the top allocators

---

[<< Previous: GC Optimization](./q02-gc-optimization.md) | [Next: Concurrency Optimization >>](./q04-concurrency-optimization.md)
