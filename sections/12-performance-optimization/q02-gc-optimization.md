# How to Optimize Garbage Collection?

[Home](../../README.md) > [Performance Optimization](./README.md) > GC Optimization

---

## Overview

Garbage collection optimization is critical for achieving consistent application performance. The goal is to minimize GC pause times while maintaining adequate throughput, choosing the right trade-off for your workload.

---

## 1. Throughput vs Latency Trade-Off

```
Throughput = (Time NOT in GC) / (Total Time) * 100

High Throughput (e.g., 99%):  Long but infrequent pauses (batch jobs)
Low Latency (e.g., < 10ms):  Short but possibly more frequent pauses (web apps)
```

```bash
# Optimize for throughput (Parallel GC)
java -XX:+UseParallelGC \
     -XX:GCTimeRatio=19 \          # Target: 1/(1+19) = 5% GC time
     -XX:MaxGCPauseMillis=500 \
     -jar batch-app.jar

# Optimize for latency (G1 GC)
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=50 \     # Target: 50ms max pause
     -jar web-app.jar

# Ultra-low latency (ZGC)
java -XX:+UseZGC \
     -jar latency-critical-app.jar  # Sub-millisecond pauses
```

---

## 2. G1 GC Tuning

### Key G1 Parameters

```bash
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:G1HeapRegionSize=16m \
     -XX:G1NewSizePercent=20 \
     -XX:G1MaxNewSizePercent=60 \
     -XX:G1ReservePercent=10 \
     -XX:ParallelGCThreads=8 \
     -XX:ConcGCThreads=2 \
     -jar app.jar
```

### G1 Tuning Guidelines

| Parameter                          | Default | Tuning Advice                           |
|------------------------------------|---------|-----------------------------------------|
| `MaxGCPauseMillis`                 | 200ms   | Lower = more frequent, shorter GCs      |
| `InitiatingHeapOccupancyPercent`   | 45%     | Lower if concurrent cycle starts late   |
| `G1HeapRegionSize`                 | auto    | 1-32 MB; larger for big heaps           |
| `G1NewSizePercent`                 | 5%      | Increase for high allocation rates      |
| `G1MaxNewSizePercent`              | 60%     | Decrease to limit young gen size        |
| `G1ReservePercent`                 | 10%     | Increase if evacuation failures occur   |
| `G1MixedGCCountTarget`            | 8       | Spread mixed GC work across N cycles    |

```java
// Trigger G1 tuning problem: humongous allocations
public class G1HumongousDemo {
    public static void main(String[] args) {
        // G1 region size = heap / 2048 (approx)
        // Object > region_size / 2 is "humongous"
        // With 4GB heap: region ~2MB, humongous threshold ~1MB

        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            // Each allocation is humongous (> 50% of region)
            list.add(new byte[2 * 1024 * 1024]); // 2 MB
            if (i % 100 == 0) {
                list.clear(); // allow GC
            }
        }
    }
}
```

---

## 3. Young Generation Sizing

```bash
# Explicit young gen sizing
java -XX:NewSize=1g -XX:MaxNewSize=1g -jar app.jar

# Using ratio (Old:Young = 2:1 means 1/3 of heap is young)
java -XX:NewRatio=2 -jar app.jar

# Survivor space ratio (Eden:Survivor = 8:1)
java -XX:SurvivorRatio=8 -jar app.jar
```

### Young Gen Sizing Strategy

```
Short-lived objects (web requests) -> Larger young gen
  - Objects die in young gen (minor GC only)
  - Less promotion to old gen
  - Fewer major/full GCs

Long-lived objects (caches) -> Smaller young gen
  - Objects quickly promoted to old gen
  - Avoids unnecessary copying in survivor spaces
```

```java
// Monitor GC behavior programmatically
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;

public class GCMonitor {
    public static void printGCStats() {
        for (GarbageCollectorMXBean gc :
                ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.printf("GC Name: %s%n", gc.getName());
            System.out.printf("  Collection Count: %d%n", gc.getCollectionCount());
            System.out.printf("  Collection Time : %d ms%n", gc.getCollectionTime());
            System.out.printf("  Memory Pools    : %s%n",
                String.join(", ", gc.getMemoryPoolNames()));
        }
    }

    public static void main(String[] args) {
        // Run workload
        List<byte[]> temp = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            temp.add(new byte[1024]);
            if (i % 1000 == 0) temp.clear();
        }
        printGCStats();
    }
}
```

---

## 4. Humongous Object Management

Objects larger than 50% of a G1 region are "humongous" and allocated directly in old generation.

```java
// Problem: frequent humongous allocations cause full GCs
public class HumongousObjectProblem {

    // BAD: allocating large byte arrays repeatedly
    public byte[] processLargeData(int sizeMB) {
        return new byte[sizeMB * 1024 * 1024]; // humongous if > region/2
    }

    // BETTER: reuse buffers with object pooling
    private static final ThreadLocal<byte[]> BUFFER_POOL =
        ThreadLocal.withInitial(() -> new byte[4 * 1024 * 1024]);

    public byte[] processWithPooling() {
        return BUFFER_POOL.get(); // reuse same buffer
    }
}
```

### Mitigation Strategies

```bash
# Increase region size to reduce humongous objects
java -XX:G1HeapRegionSize=32m -jar app.jar
# Now humongous threshold = 16 MB (instead of default ~1-2 MB)

# Enable eager reclamation of humongous objects (Java 8u60+)
# Enabled by default in modern JVMs
java -XX:+G1EagerReclaimHumongousObjects -jar app.jar
```

---

## 5. GC Log Analysis

### Enabling GC Logging

```bash
# Java 9+ (Unified Logging)
java -Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=10m \
     -jar app.jar

# Java 8
java -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -XX:+PrintGCTimeStamps \
     -Xloggc:gc.log \
     -jar app.jar
```

### Reading GC Logs

```
# Sample G1 GC log entry (Java 17)
[2024-01-15T10:30:45.123+0000][info][gc] GC(42) Pause Young (Normal)
  (G1 Evacuation Pause) 1024M->512M(2048M) 15.234ms

# Breakdown:
# GC(42)           - 42nd GC event
# Pause Young      - Young generation collection
# 1024M->512M      - Heap before -> after
# (2048M)          - Total heap size
# 15.234ms         - Pause duration
```

```java
// Notification-based GC monitoring
import com.sun.management.GarbageCollectionNotificationInfo;
import javax.management.Notification;
import javax.management.NotificationEmitter;
import javax.management.NotificationListener;
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;

public class GCNotificationListener {
    public static void installGCMonitoring() {
        for (GarbageCollectorMXBean gcBean :
                ManagementFactory.getGarbageCollectorMXBeans()) {
            NotificationEmitter emitter = (NotificationEmitter) gcBean;
            emitter.addNotificationListener((notification, handback) -> {
                if (notification.getType().equals(
                        GarbageCollectionNotificationInfo
                            .GARBAGE_COLLECTION_NOTIFICATION)) {
                    GarbageCollectionNotificationInfo info =
                        GarbageCollectionNotificationInfo.from(
                            (javax.management.openmbean.CompositeData)
                                notification.getUserData());

                    System.out.printf("[GC] %s - %s: %d ms%n",
                        info.getGcName(),
                        info.getGcAction(),
                        info.getGcInfo().getDuration());
                }
            }, null, null);
        }
    }
}
```

---

## 6. GC Optimization Checklist

### Step-by-Step Tuning Process

```
1. MEASURE   -> Enable GC logging (-Xlog:gc*)
2. ANALYZE   -> Use GCViewer, GCEasy, or JFR
3. IDENTIFY  -> Find the bottleneck:
                - Long pauses? -> Reduce MaxGCPauseMillis or switch GC
                - Full GCs?    -> Heap too small or memory leak
                - High GC %?   -> Reduce allocation rate
4. TUNE      -> Adjust ONE parameter at a time
5. VALIDATE  -> Compare before/after metrics
6. REPEAT    -> Iterate until targets are met
```

### Common GC Problems and Solutions

| Problem                    | Symptom                 | Solution                              |
|----------------------------|-------------------------|---------------------------------------|
| Frequent full GCs          | Long pauses, high CPU   | Increase heap or fix memory leak      |
| Premature promotion        | Old gen fills quickly    | Increase young gen or survivor space  |
| Humongous allocation       | G1 full GC              | Increase region size or pool objects  |
| Long concurrent marking    | Mixed GC too slow       | Increase `ConcGCThreads`             |
| Allocation stall           | Thread pauses            | Lower `InitiatingHeapOccupancyPercent`|
| Metaspace full GC          | Metaspace OOM           | Set `MaxMetaspaceSize`, check leaks   |

---

## 7. GC Pause Time Optimization Patterns

```java
// Pattern 1: Reduce object allocation rate
// BAD - creates many temporary objects
public String buildReport(List<Record> records) {
    String result = "";
    for (Record r : records) {
        result += r.toString() + "\n"; // new String each iteration
    }
    return result;
}

// GOOD - minimal allocation
public String buildReport(List<Record> records) {
    StringBuilder sb = new StringBuilder(records.size() * 64);
    for (Record r : records) {
        sb.append(r.toString()).append('\n');
    }
    return sb.toString();
}

// Pattern 2: Avoid premature promotion with object pooling
public class ObjectPool<T> {
    private final Queue<T> pool = new ConcurrentLinkedQueue<>();
    private final Supplier<T> factory;

    public ObjectPool(Supplier<T> factory, int initialSize) {
        this.factory = factory;
        for (int i = 0; i < initialSize; i++) {
            pool.offer(factory.get());
        }
    }

    public T borrow() {
        T obj = pool.poll();
        return (obj != null) ? obj : factory.get();
    }

    public void returnObj(T obj) {
        pool.offer(obj);
    }
}
```

---

## 8. ZGC and Shenandoah for Low Latency

```bash
# ZGC: Sub-millisecond pauses regardless of heap size
java -XX:+UseZGC \
     -XX:SoftMaxHeapSize=4g \    # Soft limit before aggressive GC
     -Xmx8g \                    # Hard limit
     -jar app.jar

# Shenandoah: Low-pause concurrent GC
java -XX:+UseShenandoahGC \
     -XX:ShenandoahGCHeuristics=adaptive \
     -Xmx8g \
     -jar app.jar
```

### ZGC vs Shenandoah Comparison

| Feature            | ZGC              | Shenandoah         |
|--------------------|------------------|---------------------|
| Max Pause          | < 1 ms           | < 10 ms typically   |
| Heap Range         | 8 MB - 16 TB     | Any size            |
| JDK Support        | Oracle, Temurin   | Red Hat, Temurin    |
| Concurrent Compact | Yes (colored ptrs)| Yes (Brooks ptrs)   |
| Production Ready   | Java 15+          | Java 12+            |

---

## Interview Tips

- Know the throughput vs latency trade-off and when to prioritize each
- G1's `MaxGCPauseMillis` is a _target_, not a guarantee
- Lower `InitiatingHeapOccupancyPercent` if concurrent marking starts too late
- Always analyze GC logs before tuning; never guess
- ZGC is the future for latency-sensitive workloads; pauses are O(1), not O(heap)
- Reducing allocation rate is often more effective than GC tuning

---

[<< Previous: JVM Tuning](./q01-jvm-tuning.md) | [Next: Memory Management >>](./q03-memory-management-perf.md)
