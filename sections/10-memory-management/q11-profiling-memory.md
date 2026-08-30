# Q11: How to Profile Memory Usage in Java?

[← Back to Memory Management](./README.md)

---

## Overview

Memory profiling is the process of measuring and analyzing how your Java application uses memory — what's allocated, what's retained, and where leaks occur. Java provides a rich ecosystem of tools ranging from built-in JVM flags to full-featured profilers.

---

## Tool Landscape

| Tool               | Type         | Best For                                    |
|---------------------|-------------|----------------------------------------------|
| **VisualVM**        | GUI profiler | Overall memory monitoring, heap analysis     |
| **JConsole**        | GUI monitor  | Quick JMX-based monitoring                   |
| **JFR + JMC**       | Low-overhead | Production profiling, allocation tracking    |
| **jstat**           | CLI          | GC statistics and memory pool sizes          |
| **jmap**            | CLI          | Heap dumps and histograms                    |
| **jcmd**            | CLI          | Multi-purpose diagnostics                    |
| **async-profiler**  | Native agent | Low-overhead allocation profiling            |
| **NMT**             | JVM feature  | Native (non-heap) memory tracking            |

---

## 1. VisualVM

A free, all-in-one GUI tool for monitoring and profiling Java applications:

```bash
# Launch VisualVM (included in JDK, or download separately)
jvisualvm

# Or with GraalVM / standalone:
visualvm
```

### Key features:
- **Monitor tab**: real-time heap/metaspace/threads
- **Sampler tab**: CPU and memory sampling with low overhead
- **Profiler tab**: allocation tracking with call trees
- **Heap dump**: capture and analyze heap snapshots

```
Monitor Tab:
┌─────────────────────────────────────────┐
│  Heap Memory                            │
│  ┌─────────────────────────────────┐    │
│  │          ╱╲   ╱╲   ╱╲          │    │ ← Sawtooth = healthy GC
│  │        ╱    ╲╱    ╲╱    ╲       │    │
│  │  ╱╲  ╱                    ╲     │    │
│  │╱    ╲╱                          │    │
│  └─────────────────────────────────┘    │
│  Used: 256MB / 512MB committed / 2GB max│
└─────────────────────────────────────────┘
```

---

## 2. JConsole

Lightweight JMX-based monitoring tool:

```bash
jconsole                              # GUI launcher
jconsole <pid>                        # Connect to specific process
jconsole hostname:port                # Remote connection
```

```bash
# Enable JMX remote access on your application
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     MyApp
```

### Memory metrics available via JMX

```java
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryPoolMXBean;
import java.lang.management.MemoryUsage;

public class MemoryMetrics {

    public static void printMemoryInfo() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();

        // Heap memory
        MemoryUsage heap = memoryBean.getHeapMemoryUsage();
        System.out.printf("Heap — Init: %dMB, Used: %dMB, Committed: %dMB, Max: %dMB%n",
                toMB(heap.getInit()), toMB(heap.getUsed()),
                toMB(heap.getCommitted()), toMB(heap.getMax()));

        // Non-heap memory (Metaspace, code cache, etc.)
        MemoryUsage nonHeap = memoryBean.getNonHeapMemoryUsage();
        System.out.printf("Non-Heap — Used: %dMB, Committed: %dMB%n",
                toMB(nonHeap.getUsed()), toMB(nonHeap.getCommitted()));

        // Per-pool details
        System.out.println("\nMemory Pools:");
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            MemoryUsage usage = pool.getUsage();
            System.out.printf("  %-30s Used: %8d KB, Max: %8d KB%n",
                    pool.getName(),
                    usage.getUsed() / 1024,
                    usage.getMax() / 1024);
        }
    }

    private static long toMB(long bytes) {
        return bytes / (1024 * 1024);
    }
}
```

### Memory concepts: Used vs Committed vs Max

```
┌─────────────────────────────────────────────────────┐
│ Max (-Xmx)                                          │
│ ┌─────────────────────────────────────────┐         │
│ │ Committed (OS-allocated)                │         │
│ │ ┌───────────────────────┐               │         │
│ │ │ Used (live objects)   │               │         │
│ │ └───────────────────────┘               │         │
│ └─────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────┘

- Used:      Memory actually occupied by live objects
- Committed: Memory allocated from the OS (may include free space after GC)
- Max:       Upper limit the JVM will allocate (-Xmx)
```

---

## 3. Java Flight Recorder (JFR) + Mission Control (JMC)

Low-overhead production-safe profiling built into the JVM:

```bash
# Start recording from command line
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp

# Start recording on a running process
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Stop recording
jcmd <pid> JFR.stop name=1

# Continuous recording with max size
java -XX:StartFlightRecording=maxsize=500m,disk=true,dumponexit=true,filename=app.jfr MyApp
```

```bash
# Analyze with JDK Mission Control
jmc                # Launch GUI
# Open the .jfr file → Memory tab → Allocation tracking
```

### Programmatic JFR (Java 11+)

```java
import jdk.jfr.*;

@Name("com.example.AllocationEvent")
@Label("Custom Allocation Event")
@Description("Tracks large allocations")
class AllocationEvent extends Event {
    @Label("Object Type")
    String type;

    @Label("Size (bytes)")
    long size;
}

public class JFRExample {

    public void trackAllocation(String type, long size) {
        AllocationEvent event = new AllocationEvent();
        event.type = type;
        event.size = size;
        event.commit();   // Records the event if JFR is active
    }
}
```

---

## 4. GC Logs Analysis

```bash
# Java 9+ unified logging
java -Xlog:gc*:file=gc.log:time,uptime,level,tags MyApp

# Detailed GC log with heap details
java -Xlog:gc*,gc+heap=debug:file=gc.log:time MyApp

# Java 8 (legacy flags)
java -verbose:gc -Xloggc:gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps MyApp
```

### Sample GC log output (G1)

```
[2024-01-15T10:30:15.123+0000] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2024-01-15T10:30:15.123+0000] GC(42)   Eden: 256M->0M(256M)
[2024-01-15T10:30:15.123+0000] GC(42)   Survivors: 32M->32M(32M)
[2024-01-15T10:30:15.123+0000] GC(42)   Old: 512M->520M(1024M)        ← Old gen growing?
[2024-01-15T10:30:15.123+0000] GC(42)   Heap: 800M->552M(1536M)
[2024-01-15T10:30:15.123+0000] GC(42) Pause 12.5ms
```

### GC log analysis tools

```bash
# GCViewer — open-source GUI
java -jar gcviewer.jar gc.log

# GCEasy — online analysis (upload gc.log)
# https://gceasy.io

# JClarity Censum — commercial GC analysis
```

### Signs of trouble in GC logs

```
⚠️ Frequent Full GC → possible memory leak or undersized heap
⚠️ Long GC pauses → heap too large for pause-time goal
⚠️ Old gen keeps growing → objects promoted faster than collected
⚠️ Allocation failures → Eden too small
```

---

## 5. jstat — GC Statistics

```bash
# GC summary for a running process
jstat -gc <pid> 1000 10         # Every 1s, 10 samples

# GC capacity
jstat -gccapacity <pid>

# GC cause
jstat -gccause <pid> 1000

# Output columns explained:
#  S0C/S1C  - Survivor space 0/1 capacity (KB)
#  S0U/S1U  - Survivor space 0/1 used (KB)
#  EC/EU    - Eden capacity/used (KB)
#  OC/OU    - Old gen capacity/used (KB)
#  MC/MU    - Metaspace capacity/used (KB)
#  YGC/YGCT - Young GC count/time
#  FGC/FGCT - Full GC count/time
```

---

## 6. jcmd — Multi-Purpose Diagnostics

```bash
# List all Java processes
jcmd -l

# Class histogram (top memory consumers by class)
jcmd <pid> GC.class_histogram

# Heap info
jcmd <pid> GC.heap_info

# Trigger GC
jcmd <pid> GC.run

# Heap dump
jcmd <pid> GC.heap_dump /tmp/heapdump.hprof

# VM info
jcmd <pid> VM.info

# JFR commands
jcmd <pid> JFR.start duration=30s filename=recording.jfr
```

### Sample class histogram output

```
 num     #instances   #bytes  class name
---------------------------------------------
   1:       2450000  196000000  [B             (byte arrays)
   2:       1200000   48000000  java.lang.String
   3:        800000   32000000  java.util.HashMap$Node
   4:        500000   24000000  com.example.UserSession  ← Suspect!
   5:         50000   20000000  [Ljava.lang.Object;
```

---

## 7. Native Memory Tracking (NMT)

Track JVM's native memory usage (outside the Java heap):

```bash
# Enable NMT (slight performance overhead: ~5-10%)
java -XX:NativeMemoryTracking=summary MyApp

# Or detailed tracking
java -XX:NativeMemoryTracking=detail MyApp

# Query NMT data
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail

# Set baseline and compare later
jcmd <pid> VM.native_memory baseline
# ... wait ...
jcmd <pid> VM.native_memory summary.diff
```

### Sample NMT output

```
Total:  reserved=4500MB, committed=1200MB

-                 Java Heap (reserved=2048MB, committed=512MB)
-                     Class (reserved=1100MB, committed=80MB)
-                    Thread (reserved=256MB, committed=256MB)
                             (thread #250)                      ← 250 threads × ~1MB each
-                      Code (reserved=250MB, committed=60MB)
-                        GC (reserved=200MB, committed=150MB)
-                  Internal (reserved=50MB, committed=30MB)
-                    Symbol (reserved=20MB, committed=20MB)
-    Native Memory Tracking (reserved=5MB, committed=5MB)
-           Direct Buffers  (reserved=100MB, committed=100MB)   ← Off-heap buffers
```

---

## 8. async-profiler

Ultra-low-overhead profiler that supports allocation profiling:

```bash
# Download async-profiler
# https://github.com/async-profiler/async-profiler

# Profile allocations
./asprof -e alloc -d 30 -f alloc-profile.html <pid>

# Profile allocations with flamegraph
./asprof -e alloc -d 60 --total -f alloc-flame.html <pid>

# Live allocations (objects that survived GC)
./asprof -e alloc --live -d 30 -f live-alloc.html <pid>

# Combine CPU and allocation profiling
./asprof -e cpu,alloc -d 30 -f combined.html <pid>
```

---

## Programmatic Memory Monitoring

```java
import java.lang.management.*;
import javax.management.NotificationEmitter;
import javax.management.Notification;

public class MemoryAlertSystem {

    /**
     * Set up a memory threshold alert.
     */
    public static void setupMemoryAlert(double thresholdPercent) {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();

        // Register notification listener
        NotificationEmitter emitter = (NotificationEmitter) memoryBean;
        emitter.addNotificationListener((notification, handback) -> {
            if (notification.getType().equals(
                    MemoryNotificationInfo.MEMORY_THRESHOLD_EXCEEDED)) {
                System.err.println("⚠️ MEMORY THRESHOLD EXCEEDED!");
                printMemoryDetails();
            }
        }, null, null);

        // Set threshold on Old Gen pool
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            if (pool.getType() == MemoryType.HEAP && pool.isUsageThresholdSupported()) {
                long maxMemory = pool.getUsage().getMax();
                long threshold = (long) (maxMemory * thresholdPercent);
                pool.setUsageThreshold(threshold);
                System.out.printf("Alert set on %s at %dMB (%.0f%%)%n",
                        pool.getName(), threshold / (1024 * 1024), thresholdPercent * 100);
            }
        }
    }

    private static void printMemoryDetails() {
        Runtime rt = Runtime.getRuntime();
        System.out.printf("Heap: %dMB used / %dMB max%n",
                (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024),
                rt.maxMemory() / (1024 * 1024));
    }
}
```

---

## Profiling Workflow

```
1. Monitor  → VisualVM/JConsole/Grafana — observe heap trends
      │
      ▼
2. Suspect  → Rising baseline after GC? Frequent Full GC? OOM?
      │
      ▼
3. Profile  → JFR allocation tracking, async-profiler
      │
      ▼
4. Dump     → jcmd GC.heap_dump or -XX:+HeapDumpOnOutOfMemoryError
      │
      ▼
5. Analyze  → Eclipse MAT (dominator tree, leak suspects)
      │
      ▼
6. Fix      → Remove leak, resize pool, optimize data structures
      │
      ▼
7. Verify   → Re-run monitoring to confirm fix
```

---

## Best Practices

```bash
# ✅ Always enable in production:
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/dumps/
-Xlog:gc*:file=gc.log:time

# ✅ Use JFR for continuous low-overhead monitoring
-XX:StartFlightRecording=maxsize=500m,disk=true,dumponexit=true

# ✅ Enable NMT for native memory visibility (staging/debugging)
-XX:NativeMemoryTracking=summary
```

```java
// ✅ Expose memory metrics via Micrometer/Prometheus in Spring Boot
// spring-boot-starter-actuator provides /actuator/metrics/jvm.memory.*

// ❌ Don't call System.gc() to "fix" memory issues
// ❌ Don't profile in production with high-overhead tools (full profiler)
// ❌ Don't ignore GC log warnings
// ❌ Don't tune GC without measuring first
```

---

## Interview Tips
- **JFR** is the go-to for production profiling — <1% overhead, built into the JVM
- **VisualVM** is the best free GUI tool for development-time profiling
- A **sawtooth** heap pattern is healthy; a **rising baseline** indicates a leak
- **NMT** tracks native memory — essential for diagnosing thread stacks, direct buffers, and JNI allocations
- Know the difference between **used**, **committed**, and **max** heap

---

[← Q10: Memory Optimization](./q10-memory-optimization.md) | [Next: Q12 - Heap Dump Analysis →](./q12-heap-dump-analysis.md)
