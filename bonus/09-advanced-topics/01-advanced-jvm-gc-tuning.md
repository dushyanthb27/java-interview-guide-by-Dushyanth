# Advanced JVM GC Tuning Deep-Dive

## 📋 Question
Explain advanced JVM garbage collection tuning strategies for G1, ZGC, and Shenandoah. How do you optimize for low latency vs high throughput? What are the key tuning parameters and monitoring techniques? Provide real-world tuning examples.

## Difficulty: 🔴 Expert

---

## 💡 Answer

GC tuning is both art and science. The right configuration depends on your workload characteristics, latency requirements, and hardware resources. Modern JVMs offer sophisticated collectors that can handle most workloads with minimal tuning, but understanding the internals is crucial for production systems.

### G1 GC Tuning Strategies

G1 is the default since Java 9 and provides a good balance between throughput and latency for heaps up to 64GB.

**Key G1 Parameters:**

```bash
# Basic G1 configuration
-XX:+UseG1GC                    # Enable G1 GC
-XX:MaxGCPauseMillis=200       # Target max pause time (default 200ms)
-XX:G1HeapRegionSize=16m       # Region size (1-32MB, based on heap)
-XX:G1ReservePercent=15        # Reserve for to-space (default 10%)
-XX:InitiatingHeapOccupancyPercent=45  # Start concurrent cycle at 45% (default 45)
```

**Tuning for Low Latency:**

```bash
# Low-latency G1 configuration (target < 50ms pauses)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50         # Aggressive pause target
-XX:G1HeapRegionSize=8m        # Smaller regions = finer-grained collection
-XX:G1MixedGCCountTarget=8     # More mixed collections per cycle
-XX:G1OldCSetRegionThreshold=5 # Fewer old regions per mixed collection
-XX:G1RSetUpdatingPauseTimePercent=5  # Limit time updating remembered sets
-XX:InitiatingHeapOccupancyPercent=35  # Start cycle earlier
-Xms4g -Xmx4g                   # Fixed heap to avoid resizing pauses
```

**Tuning for High Throughput:**

```bash
# High-throughput G1 configuration (batch processing)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=500        # Allow longer pauses for more work
-XX:G1HeapRegionSize=32m        # Larger regions = less overhead
-XX:G1MixedGCCountTarget=4     # Fewer mixed collections
-XX:G1OldCSetRegionThreshold=10 # More old regions per collection
-XX:InitiatingHeapOccupancyPercent=60  # Start cycle later
-Xms8g -Xmx8g                   # Larger heap for more working set
```

**Humongous Object Tuning:**

```bash
# G1 treats objects > 50% of region size as "humongous"
# These are allocated directly in old gen and can cause fragmentation

# For applications with many large objects (e.g., big arrays, buffers)
-XX:G1HeapRegionSize=32m       # Larger regions = fewer humongous objects
-XX:G1MixedGCLiveThresholdPercent=85  # Keep more live objects in old gen
-XX:G1MaxVerifyJobSize=1g      # Limit verification overhead

# Monitor humongous allocations
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
```

**G1 Monitoring and Diagnostics:**

```bash
# Enable detailed G1 logging
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=100m

# Key metrics to watch:
# - G1 Evacuation Pause time (should be < MaxGCPauseMillis)
# - G1 Concurrent Mark time (should be mostly concurrent)
# - To-space exhausted (indicates need for larger heap or earlier cycle)
# - Region fragmentation

# JFR (Java Flight Recorder) for production profiling
-XX:StartFlightRecording=duration=60s,filename=recording.jfr,dumponexit=true
```

### ZGC Tuning Strategies

ZGC (Z Garbage Collector) is designed for sub-millisecond pause times and multi-terabyte heaps. It's production-ready since Java 15.

**Key ZGC Parameters:**

```bash
# Basic ZGC configuration
-XX:+UseZGC                    # Enable ZGC
-Xms16g -Xmx16g                # ZGC requires at least 8GB heap (Java 15+)
-XX:ZCollectionInterval=5      # Target collection interval in seconds (default auto)
```

**Tuning for Ultra-Low Latency:**

```bash
# Ultra-low latency ZGC (< 1ms pauses)
-XX:+UseZGC
-Xms32g -Xmx32g                # Larger heap reduces GC frequency
-XX:ZCollectionInterval=3      # More frequent collections = shorter pauses
-XX:ZAllocationSpikeTolerance=2  # Tolerance for allocation spikes (default 2.0)
-XX:ZFragmentationLimit=25     # Fragmentation threshold (default 25%)
-XX:ConcGCThreads=4            # Concurrent GC threads (default: ~25% of CPUs)
-XX:ParallelGCThreads=8        # Parallel GC threads for STW phases
```

**Tuning for Large Heaps (100GB+):**

```bash
# Large heap ZGC configuration
-XX:+UseZGC
-Xms256g -Xmx256g              # Multi-terabyte heap support
-XX:ZCollectionInterval=10     # Longer interval for large heaps
-XX:ConcGCThreads=16           # More concurrent threads for large heap
-XX:ParallelGCThreads=32       # More parallel threads
-XX:ZAllocationSpikeTolerance=5  # Higher tolerance for large heaps
```

**ZGC Memory Layout Tuning:**

```bash
# ZGC uses multiple memory spaces
-XX:ZPageSize=2m               # Page size (default 2m, options: 2m, 4m, 8m, 16m, 32m)
-XX:ZPageCacheFlushMin=1m      # Minimum cache flush size
-XX:ZPageCacheFlushMax=10m     # Maximum cache flush size

# For applications with many small objects:
-XX:ZPageSize=2m               # Smaller pages = less waste

# For applications with large objects:
-XX:ZPageSize=16m              # Larger pages = fewer page allocations
```

**ZGC Monitoring:**

```bash
# ZGC-specific logging
-Xlog:gc*:file=zgc.log:time,uptime,level,tags

# Key ZGC metrics:
# - Pause times (should be < 1ms)
# - Allocation rate (MB/s)
# - Page cache efficiency
# - Relocation set size

# ZGC provides detailed statistics via JMX
jcmd <pid> GC.zgc
```

### Shenandoah Tuning Strategies

Shenandoah is similar to ZGC but available in OpenJDK distributions. It uses Brooks pointers for concurrent compaction.

**Key Shenandoah Parameters:**

```bash
# Basic Shenandoah configuration
-XX:+UseShenandoahGC           # Enable Shenandoah
-Xms8g -Xmx8g                  # Heap size
-XX:ShenandoahGCHeuristics=adaptive  # Heuristic: adaptive, compact, aggressive, passive
```

**Tuning for Low Latency:**

```bash
# Low-latency Shenandoah
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=compact  # Compact heuristic for low fragmentation
-XX:ShenandoahGarbageThreshold=5    # Start GC at 5% garbage (default 10%)
-XX:ShenandoahAllocationThreshold=0 # No allocation threshold
-XX:ShenandoahSATBBarrier=true      # SATB barrier (default)
-XX:ShenandoahUpdateRefsEarly=true  # Early reference updates
```

**Tuning for High Throughput:**

```bash
# High-throughput Shenandoah
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=passive   # Passive heuristic (manual triggers)
-XX:ShenandoahGarbageThreshold=30   # Start GC at 30% garbage
-XX:ShenandoahAllocationThreshold=10 # Wait for 10% allocation before GC
```

### GC Selection Decision Tree

```
┌─────────────────────────────────────────────────┐
│                 Heap Size                       │
└─────────────────┬───────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
  < 4GB        4-64GB        > 64GB
    │             │             │
    ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Serial  │  │   G1    │  │   ZGC   │
│/Parallel│  │ (default)│  │Shenandoah│
└─────────┘  └─────────┘  └─────────┘
    │             │             │
    │      ┌──────┴──────┐      │
    │      │  Latency    │      │
    │      │  Goal?     │      │
    │      └──────┬──────┘      │
    │    ┌─────────┼─────────┐  │
    │    ▼         ▼         ▼  │
    │  < 100ms   < 10ms   < 1ms │
    │    │         │         │  │
    │    ▼         ▼         ▼  │
    │   G1        G1       ZGC   │
    │  (tuned)  (tuned)        │
    └────────────────────────────┘
```

### Real-World Tuning Examples

**Example 1: High-Frequency Trading (Sub-millisecond latency)**

```bash
# Requirements: < 1ms GC pauses, 16GB heap, high allocation rate
java -XX:+UseZGC \
     -Xms16g -Xmx16g \
     -XX:ZCollectionInterval=2 \
     -XX:ConcGCThreads=8 \
     -XX:ParallelGCThreads=16 \
     -XX:ZAllocationSpikeTolerance=3 \
     -Xlog:gc*:file=hft-gc.log:time,uptime,level,tags \
     -jar trading-app.jar
```

**Example 2: E-commerce Web Application (Low latency, moderate heap)**

```bash
# Requirements: < 100ms pauses, 8GB heap, read-heavy
java -XX:+UseG1GC \
     -Xms8g -Xmx8g \
     -XX:MaxGCPauseMillis=100 \
     -XX:G1HeapRegionSize=8m \
     -XX:InitiatingHeapOccupancyPercent=40 \
     -XX:G1MixedGCCountTarget=8 \
     -XX:+UseStringDeduplication \
     -Xlog:gc*:file=web-gc.log:time,uptime,level,tags \
     -jar webapp.jar
```

**Example 3: Batch Data Processing (High throughput)**

```bash
# Requirements: Maximize throughput, 32GB heap, long-running
java -XX:+UseG1GC \
     -Xms32g -Xmx32g \
     -XX:MaxGCPauseMillis=500 \
     -XX:G1HeapRegionSize=32m \
     -XX:InitiatingHeapOccupancyPercent=60 \
     -XX:G1MixedGCCountTarget=4 \
     -XX:G1OldCSetRegionThreshold=15 \
     -Xlog:gc*:file=batch-gc.log:time,uptime,level,tags \
     -jar batch-processor.jar
```

**Example 4: Microservice in Container (Small heap, low memory)**

```bash
# Requirements: 2GB heap, containerized, predictable pauses
java -XX:+UseG1GC \
     -Xms2g -Xmx2g \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=2m \
     -XX:G1ReservePercent=20 \
     -XX:+UseStringDeduplication \
     -XX:+PrintGCDetails \
     -XX:+PrintGCTimeStamps \
     -jar microservice.jar
```

### GC Log Analysis

**Analyzing G1 GC logs:**

```bash
# Use GCViewer or GCEasy for log analysis
java -jar gcviewer.jar gc.log

# Key G1 log patterns to look for:
# 1. "to-space exhausted" → Need larger heap or earlier concurrent cycle
# 2. Long evacuation pauses → Reduce MaxGCPauseMillis or increase region size
# 3. Frequent full GC → Increase InitiatingHeapOccupancyPercent
# 4. High mixed GC count → Increase G1MixedGCCountTarget

# Example G1 log output:
# [GC pause (G1 Evacuation Pause) (young), 0.0456789 secs]
#   [Parallel Time: 35.2 ms, GC Workers: 8]
#   [Eden: 512.0M(512.0M)->0.0B(512.0M) Survivors: 64.0M->64.0M Heap: 2048.0M(2048.0M)->1536.0M(2048.0M)]
```

**Analyzing ZGC GC logs:**

```bash
# ZGC log patterns:
# 1. "Pause Mark Start" and "Pause Mark End" should be < 1ms
# 2. "Concurrent Mark" should take most of the cycle time
# 3. "Concurrent Relocate" should not cause long pauses

# Example ZGC log output:
# [0.123s][info][gc,start     ] GC(0) Garbage Collection (Proactive)
# [0.123s][info][gc,task      ] GC(0) Using 4 of 4 workers for concurrent marking
# [0.125s][info][gc,phases    ] GC(0) Pause Mark Start 0.125ms
# [0.150s][info][gc,phases    ] GC(0) Concurrent Mark 25.000ms
# [0.151s][info][gc,phases    ] GC(0) Pause Mark End 0.125ms
```

### Code Example: GC Monitoring Program

```java
package com.interview.advanced.jvm;

import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class GCMonitor {
    private static final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    public static void main(String[] args) {
        startMonitoring();
        
        // Simulate application workload
        generateGarbage();
    }
    
    public static void startMonitoring() {
        scheduler.scheduleAtFixedRate(() -> {
            printGCStats();
            printMemoryStats();
        }, 0, 5, TimeUnit.SECONDS);
    }
    
    private static void printGCStats() {
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
        
        System.out.println("\n=== GC Statistics ===");
        for (GarbageCollectorMXBean gc : gcBeans) {
            System.out.printf("Collector: %s%n", gc.getName());
            System.out.printf("  Collections: %d%n", gc.getCollectionCount());
            System.out.printf("  Time: %d ms%n", gc.getCollectionTime());
            System.out.printf("  Memory Pools: %s%n", 
                String.join(", ", gc.getMemoryPoolNames()));
        }
    }
    
    private static void printMemoryStats() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        System.out.println("\n=== Memory Statistics ===");
        System.out.printf("Heap: %d MB / %d MB (%.1f%%)%n",
            heapUsage.getUsed() / (1024 * 1024),
            heapUsage.getMax() / (1024 * 1024),
            (heapUsage.getUsed() * 100.0) / heapUsage.getMax());
        
        System.out.printf("Non-Heap: %d MB / %d MB%n",
            nonHeapUsage.getUsed() / (1024 * 1024),
            nonHeapUsage.getMax() / (1024 * 1024));
        
        // Print individual pool stats
        List<MemoryPoolMXBean> poolBeans = 
            ManagementFactory.getMemoryPoolMXBeans();
        
        System.out.println("\n=== Memory Pools ===");
        for (MemoryPoolMXBean pool : poolBeans) {
            MemoryUsage usage = pool.getUsage();
            System.out.printf("%s: %d MB / %d MB (%.1f%%)%n",
                pool.getName(),
                usage.getUsed() / (1024 * 1024),
                usage.getMax() / (1024 * 1024),
                usage.getMax() > 0 ? (usage.getUsed() * 100.0) / usage.getMax() : 0);
        }
    }
    
    private static void generateGarbage() {
        System.out.println("Generating garbage for 30 seconds...");
        
        ExecutorService executor = Executors.newFixedThreadPool(4);
        for (int i = 0; i < 4; i++) {
            executor.submit(() -> {
                for (int j = 0; j < 1000; j++) {
                    byte[] data = new byte[1024 * 1024];  // 1MB
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }
        
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        executor.shutdownNow();
        scheduler.shutdown();
        System.out.println("Monitoring complete.");
    }
}
```

### ⚠️ Common Mistakes

**1. Setting heap too large for G1**

```bash
# ❌ G1 with 100GB heap — not optimal
java -XX:+UseG1GC -Xms100g -Xmx100g -jar app.jar

# ✅ Use ZGC for large heaps
java -XX:+UseZGC -Xms100g -Xmx100g -jar app.jar
```

**2. Aggressive pause goals that cause thrashing**

```bash
# ❌ Too aggressive — causes excessive GC cycles
java -XX:+UseG1GC -XX:MaxGCPauseMillis=10 -jar app.jar

# ✅ Realistic goal based on workload
java -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -jar app.jar
```

**3. Not monitoring GC in production**

```bash
# ❌ No GC logging — blind to performance issues
java -jar app.jar

# ✅ Always enable GC logging in production
java -Xlog:gc*:file=gc.log:time,uptime,level,tags -jar app.jar
```

### ⚡ Performance Tip

**Start with defaults, then tune incrementally.** Modern GCs are well-tuned for most workloads. Only adjust parameters when you have:
1. Measured performance issues (GC logs, metrics)
2. Clear requirements (latency SLA, throughput goals)
3. A baseline to compare against

**Tuning order:**
1. Set heap size (Xms/Xmx)
2. Choose collector based on heap size and latency requirements
3. Adjust pause time goal (MaxGCPauseMillis)
4. Tune region size (G1) or collection interval (ZGC)
5. Adjust concurrent cycle trigger (InitiatingHeapOccupancyPercent)

### 🔧 JVM Internals

**G1's region-based layout** allows it to collect only the most garbage-filled regions first. The `-XX:MaxGCPauseMillis` goal drives the collector to select regions that can be collected within the target time. The concurrent marking phase identifies live objects and builds a collection set.

**ZGC's colored pointers** use unused bits in 64-bit object references to store marking information (Marked0, Marked1, Remapped, Finalizable). Load barriers check these bits and perform fix-up operations, allowing almost all GC work to be concurrent.

**Shenandoah's Brooks pointers** add an indirection to every object, allowing the object to be moved while references are updated concurrently via CAS operations.

---

## 🎯 Interviewer Insight

GC tuning questions test your understanding of JVM internals and production experience. Interviewers want to see that you:
- Know the strengths and weaknesses of each collector
- Can make data-driven tuning decisions based on requirements
- Understand the trade-offs between latency, throughput, and memory
- Have experience monitoring and analyzing GC in production

Strong candidates can walk through a real tuning scenario, explaining each parameter choice and its impact on the application.

## ⚡ Follow-up Questions

1. How does G1 decide which regions to collect in a mixed GC?
2. What causes "to-space exhausted" errors in G1 and how do you fix them?
3. Why does ZGC require a minimum heap size of 8GB (in older versions)?

## 📚 Related Topics

- [GC Collectors](../../sections/07-jvm-internals/q06-gc-collectors.md)
- [GC Algorithms](../../sections/07-jvm-internals/q05-gc-algorithms.md)
- [JVM Tuning Parameters](../../sections/07-jvm-internals/q12-jvm-tuning-parameters.md)
- [Monitoring JVM](../../sections/07-jvm-internals/q13-monitoring-jvm.md)
