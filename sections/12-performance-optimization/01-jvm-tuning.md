# How to Tune the JVM for Performance?

[Home](../../README.md) > [Performance Optimization](./README.md) > JVM Tuning

---

## Overview

JVM tuning is the process of adjusting runtime parameters to optimize application throughput, latency, and resource utilization. Effective tuning starts with understanding workload characteristics and selecting appropriate JVM flags.

---

## 1. Heap Sizing (-Xms, -Xmx)

```bash
# Set initial and maximum heap size
java -Xms2g -Xmx2g -jar app.jar

# Rule of thumb: Set -Xms = -Xmx to avoid heap resizing overhead
# Typical: 50-70% of available physical memory
```

### Sizing Guidelines

| Workload Type      | Recommended Heap | Rationale                            |
|--------------------|------------------|--------------------------------------|
| Microservice       | 256m - 1g        | Small footprint, fast GC             |
| Web Application    | 1g - 4g          | Moderate object allocation            |
| Batch Processing   | 4g - 16g         | Large datasets in memory              |
| In-Memory Database | 16g - 64g+       | Entire dataset resident in heap       |

```java
// Programmatically check heap settings
public class HeapInfo {
    public static void main(String[] args) {
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();       // -Xmx
        long totalMemory = runtime.totalMemory();   // current heap size
        long freeMemory = runtime.freeMemory();     // free within current heap

        System.out.printf("Max Memory  : %d MB%n", maxMemory / (1024 * 1024));
        System.out.printf("Total Memory: %d MB%n", totalMemory / (1024 * 1024));
        System.out.printf("Free Memory : %d MB%n", freeMemory / (1024 * 1024));
        System.out.printf("Used Memory : %d MB%n",
            (totalMemory - freeMemory) / (1024 * 1024));
    }
}
```

---

## 2. Garbage Collector Selection

```bash
# Serial GC - single-threaded, small heaps
java -XX:+UseSerialGC -jar app.jar

# Parallel GC - throughput-oriented (default in Java 8)
java -XX:+UseParallelGC -jar app.jar

# G1 GC - balanced throughput/latency (default in Java 9+)
java -XX:+UseG1GC -jar app.jar

# ZGC - ultra-low latency (Java 15+)
java -XX:+UseZGC -jar app.jar

# Shenandoah - low latency alternative
java -XX:+UseShenandoahGC -jar app.jar
```

### GC Selection Matrix

| GC            | Max Pause | Throughput | Heap Range | Best For               |
|---------------|-----------|------------|------------|------------------------|
| Serial        | High      | Low        | < 256 MB   | Single-core, small app |
| Parallel      | Medium    | Highest    | 1-8 GB     | Batch processing       |
| G1            | Low       | High       | 4-64 GB    | General purpose        |
| ZGC           | < 1 ms    | High       | 8 GB-16 TB | Latency-critical       |
| Shenandoah    | < 10 ms   | High       | 4-64 GB    | Latency-sensitive      |

---

## 3. JIT Compiler Tuning

```bash
# Use tiered compilation (default in Java 8+)
java -XX:+TieredCompilation -jar app.jar

# Print compilation events
java -XX:+PrintCompilation -jar app.jar

# Set compilation threshold (invocations before JIT compiles)
java -XX:CompileThreshold=5000 -jar app.jar

# Reserve code cache size (important for large applications)
java -XX:ReservedCodeCacheSize=256m -jar app.jar

# Disable tiered compilation for faster startup (short-lived processes)
java -XX:-TieredCompilation -jar app.jar
```

```java
// Force method compilation hint (informational only)
public class HotMethod {

    // This method will be JIT-compiled after enough invocations
    public static int computeSum(int n) {
        int sum = 0;
        for (int i = 0; i < n; i++) {
            sum += i;
        }
        return sum;
    }

    public static void main(String[] args) {
        // Warm up the JIT
        for (int i = 0; i < 100_000; i++) {
            computeSum(1000);
        }
        // After warmup, method is compiled to native code
        long start = System.nanoTime();
        int result = computeSum(1_000_000);
        long elapsed = System.nanoTime() - start;
        System.out.printf("Result: %d in %d ns%n", result, elapsed);
    }
}
```

---

## 4. Thread Stack Size

```bash
# Default thread stack size (512k-1m depending on OS/JVM)
java -Xss512k -jar app.jar

# Reduce for applications with many threads
java -Xss256k -jar app.jar

# Increase for deep recursion
java -Xss2m -jar app.jar
```

### Impact Calculation

```
Memory per thread = stack size (default ~1 MB)
1000 threads * 1 MB = 1 GB just for thread stacks!
1000 threads * 256 KB = 250 MB (75% reduction)
```

---

## 5. Metaspace Configuration

```bash
# Set maximum Metaspace size (unbounded by default)
java -XX:MaxMetaspaceSize=256m -jar app.jar

# Set initial Metaspace size
java -XX:MetaspaceSize=128m -jar app.jar

# Useful for applications with many classes (e.g., heavy reflection/proxies)
```

```java
// Monitor Metaspace usage
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryPoolMXBean;

public class MetaspaceMonitor {
    public static void main(String[] args) {
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            if (pool.getName().contains("Metaspace")) {
                System.out.printf("Metaspace Used: %d MB%n",
                    pool.getUsage().getUsed() / (1024 * 1024));
                System.out.printf("Metaspace Max : %d MB%n",
                    pool.getUsage().getMax() / (1024 * 1024));
            }
        }
    }
}
```

---

## 6. Container-Aware Settings (Docker/Kubernetes)

```bash
# Java 10+ automatically detects container memory limits
# Java 8u191+ with explicit flag
java -XX:+UseContainerSupport -jar app.jar

# Limit heap to percentage of container memory
java -XX:MaxRAMPercentage=75.0 -jar app.jar

# Set initial RAM percentage
java -XX:InitialRAMPercentage=50.0 -jar app.jar

# Limit active processor count
java -XX:ActiveProcessorCount=2 -jar app.jar
```

```dockerfile
# Dockerfile best practices
FROM eclipse-temurin:21-jre-alpine
COPY app.jar /app/app.jar

# Let JVM auto-detect container limits
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 \
               -XX:InitialRAMPercentage=50.0 \
               -XX:+UseG1GC \
               -XX:MaxGCPauseMillis=200"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

---

## 7. JVM Ergonomics

The JVM automatically selects defaults based on the environment:

```bash
# Print all default flag values
java -XX:+PrintFlagsFinal -version 2>&1 | grep -E "MaxHeapSize|GC|Thread"

# Print flags that were set (by ergonomics or user)
java -XX:+PrintCommandLineFlags -version
```

| Factor        | Ergonomic Decision                         |
|---------------|--------------------------------------------|
| CPU cores     | GC threads = cores (up to 8, then 5/8)     |
| Available RAM | Heap = 1/4 of physical memory (max 32 GB)  |
| Architecture  | Compressed oops enabled for heaps < 32 GB  |
| OS            | Thread stack size, large pages support      |

---

## 8. Common Tuning Scenarios

### Web Application (Low Latency)
```bash
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:+UseStringDeduplication \
     -XX:ReservedCodeCacheSize=256m \
     -jar webapp.jar
```

### Batch Processing (High Throughput)
```bash
java -Xms8g -Xmx8g \
     -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -XX:+UseCompressedOops \
     -jar batch.jar
```

### Microservice in Container
```bash
java -XX:MaxRAMPercentage=75.0 \
     -XX:+UseZGC \
     -XX:+UseStringDeduplication \
     -Xss256k \
     -jar microservice.jar
```

---

## 9. JVM Flags Cheat Sheet

| Category    | Flag                             | Purpose                            |
|-------------|----------------------------------|------------------------------------|
| Heap        | `-Xms` / `-Xmx`                 | Initial / max heap size            |
| Heap        | `-XX:NewRatio=2`                 | Old:Young generation ratio         |
| Heap        | `-XX:SurvivorRatio=8`            | Eden:Survivor ratio                |
| GC          | `-XX:+UseG1GC`                   | Enable G1 garbage collector        |
| GC          | `-XX:MaxGCPauseMillis=200`       | Target max GC pause time           |
| GC          | `-XX:ParallelGCThreads=N`        | Number of parallel GC threads      |
| JIT         | `-XX:+TieredCompilation`         | Enable tiered JIT compilation      |
| JIT         | `-XX:ReservedCodeCacheSize=256m` | Code cache for compiled methods    |
| Thread      | `-Xss512k`                       | Thread stack size                  |
| Metaspace   | `-XX:MaxMetaspaceSize=256m`      | Max class metadata memory          |
| Container   | `-XX:MaxRAMPercentage=75.0`      | Heap as % of container memory      |
| Diagnostics | `-XX:+HeapDumpOnOutOfMemoryError`| Dump heap on OOM                   |
| Diagnostics | `-XX:HeapDumpPath=/tmp/dump.hprof`| Heap dump file location           |
| Diagnostics | `-XX:+PrintGCDetails`            | Detailed GC logging (pre-Java 9)   |
| Diagnostics | `-Xlog:gc*`                      | Unified GC logging (Java 9+)       |

---

## Interview Tips

- Always set `-Xms` equal to `-Xmx` in production to avoid resize pauses
- Use `-XX:MaxRAMPercentage` in containers instead of fixed `-Xmx`
- G1 is the default and best general-purpose collector from Java 9+
- ZGC for sub-millisecond pauses; Parallel GC for maximum throughput
- Monitor with `-Xlog:gc*` before tuning; never tune blindly
- Compressed oops are enabled automatically when heap < 32 GB

---

[Next: GC Optimization >>](./q02-gc-optimization.md)
