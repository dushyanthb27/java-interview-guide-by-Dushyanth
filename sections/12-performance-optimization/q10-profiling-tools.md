# Profiling Tools and Techniques

[Home](../../README.md) > [Performance Optimization](./README.md) > Profiling Tools

---

## Overview

Profiling is the practice of measuring application behavior at runtime to identify performance bottlenecks. The golden rule: **measure first, optimize second**. Java offers world-class profiling tools from built-in JFR to open-source async-profiler.

---

## 1. Java Flight Recorder (JFR)

JFR is a low-overhead profiling framework built into the JVM. Safe for production use (< 1% overhead).

```bash
# Start recording from command line
java -XX:StartFlightRecording=filename=recording.jfr,duration=60s,settings=profile \
     -jar app.jar

# Start recording on a running JVM
jcmd <pid> JFR.start name=myrecording duration=60s filename=recording.jfr

# Stop recording
jcmd <pid> JFR.stop name=myrecording

# Dump recording from running JVM
jcmd <pid> JFR.dump name=myrecording filename=dump.jfr

# List active recordings
jcmd <pid> JFR.check
```

### Programmatic JFR API (Java 14+)

```java
import jdk.jfr.*;
import jdk.jfr.consumer.*;

public class JfrProgrammatic {

    // Define custom JFR event
    @Name("com.myapp.DatabaseQuery")
    @Label("Database Query")
    @Category("Database")
    @StackTrace(true)
    static class DatabaseQueryEvent extends Event {
        @Label("SQL Query")
        String sql;

        @Label("Duration (ms)")
        long durationMs;

        @Label("Row Count")
        int rowCount;
    }

    // Emit custom events
    public List<User> executeQuery(String sql) {
        DatabaseQueryEvent event = new DatabaseQueryEvent();
        event.sql = sql;
        event.begin();

        try {
            List<User> results = jdbcTemplate.query(sql, userMapper);
            event.rowCount = results.size();
            return results;
        } finally {
            event.end();
            event.durationMs = event.getDuration().toMillis();
            event.commit();  // recorded by JFR
        }
    }

    // Stream JFR events in real-time
    public static void streamEvents() throws Exception {
        try (RecordingStream stream = new RecordingStream()) {
            stream.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(1));
            stream.enable("jdk.GarbageCollection");
            stream.enable("com.myapp.DatabaseQuery");

            stream.onEvent("jdk.CPULoad", event -> {
                float machineTotal = event.getFloat("machineTotal");
                float jvmUser = event.getFloat("jvmUser");
                System.out.printf("CPU: machine=%.1f%%, jvm=%.1f%%%n",
                    machineTotal * 100, jvmUser * 100);
            });

            stream.onEvent("jdk.GarbageCollection", event -> {
                System.out.printf("GC: %s, duration=%d ms%n",
                    event.getString("name"),
                    event.getDuration().toMillis());
            });

            stream.start(); // blocks until stream is closed
        }
    }
}
```

### Key JFR Events

| Category    | Event                         | What It Reveals              |
|-------------|-------------------------------|------------------------------|
| CPU         | `jdk.CPULoad`                 | CPU usage by JVM and machine |
| Memory      | `jdk.ObjectAllocationSample`  | Hot allocation sites         |
| GC          | `jdk.GarbageCollection`       | GC pauses and types          |
| GC          | `jdk.G1GarbageCollection`     | G1-specific details          |
| Threads     | `jdk.ThreadPark`              | Thread waiting/parking       |
| Locks       | `jdk.JavaMonitorEnter`        | Lock contention              |
| I/O         | `jdk.FileRead` / `FileWrite` | File I/O operations          |
| I/O         | `jdk.SocketRead` / `Write`   | Network I/O                  |
| Code        | `jdk.ExecutionSample`         | CPU-sampled stack traces     |

---

## 2. async-profiler

Open-source low-overhead profiler. Uses OS-level mechanisms (perf_events on Linux, DTrace on macOS).

```bash
# Download async-profiler
# https://github.com/async-profiler/async-profiler

# CPU profiling (30 seconds)
./asprof -d 30 -f cpu-profile.html <pid>

# Allocation profiling
./asprof -d 30 -e alloc -f alloc-profile.html <pid>

# Lock contention profiling
./asprof -d 30 -e lock -f lock-profile.html <pid>

# Wall-clock profiling (includes I/O wait time)
./asprof -d 30 -e wall -f wall-profile.html <pid>

# Generate flame graph
./asprof -d 60 -f flamegraph.html <pid>

# Attach as Java agent
java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.jfr \
     -jar app.jar
```

### async-profiler vs JFR

| Feature             | async-profiler    | JFR               |
|---------------------|-------------------|--------------------|
| Overhead            | ~1-5%             | < 1%               |
| Accuracy            | Very high (perf)  | High (safepoints)  |
| Native frames       | Yes               | Limited            |
| Allocation profiling| Yes               | Yes (sampled)      |
| Lock profiling      | Yes               | Yes                |
| Production safe     | Yes               | Yes                |
| Built-in            | No (install)      | Yes (JDK 11+)      |
| Flame graph output  | Yes               | Via JMC            |

---

## 3. VisualVM

```bash
# Launch VisualVM (bundled with JDK 8, separate download for later versions)
jvisualvm

# Or download from https://visualvm.github.io/
```

### VisualVM Capabilities

```
1. Monitor Tab:    CPU usage, heap usage, classes loaded, threads
2. Threads Tab:    Thread states, deadlock detection
3. Sampler Tab:    CPU sampling (hot methods), memory sampling (allocations)
4. Profiler Tab:   Instrumented profiling (more accurate, higher overhead)
5. Heap Dump:      Analyze object retention, find memory leaks
6. Thread Dump:    Snapshot of all thread stack traces
```

```java
// Enable JMX for remote VisualVM connection
// java -Dcom.sun.management.jmxremote \
//      -Dcom.sun.management.jmxremote.port=9010 \
//      -Dcom.sun.management.jmxremote.authenticate=false \
//      -Dcom.sun.management.jmxremote.ssl=false \
//      -jar app.jar
```

---

## 4. JDK Mission Control (JMC)

```bash
# Open JMC (bundled with Oracle JDK or download separately)
jmc

# JMC features:
# 1. JFR analysis (open .jfr recordings)
# 2. Automated analysis (detects common problems)
# 3. Flame graph visualization
# 4. Memory leak detection
# 5. Lock contention analysis
```

### JMC Automated Analysis Rules

```
JMC automatically detects:
- GC pauses exceeding thresholds
- High allocation rates
- Lock contention hotspots
- Thread starvation
- Heap fragmentation
- File/socket I/O bottlenecks
- Code cache issues
- Class loading overhead
```

---

## 5. CPU Profiling

```java
// Identify hot methods consuming the most CPU

// Step 1: Collect CPU profile
// jcmd <pid> JFR.start name=cpu duration=30s settings=profile

// Step 2: Analyze top methods in JMC or programmatically
import jdk.jfr.consumer.RecordingFile;
import jdk.jfr.consumer.RecordedEvent;
import jdk.jfr.consumer.RecordedStackTrace;

public class CpuProfileAnalyzer {
    public static void analyzeHotMethods(Path jfrFile) throws IOException {
        Map<String, Long> methodCounts = new HashMap<>();

        try (RecordingFile recording = new RecordingFile(jfrFile)) {
            while (recording.hasMoreEvents()) {
                RecordedEvent event = recording.readEvent();
                if (event.getEventType().getName().equals("jdk.ExecutionSample")) {
                    RecordedStackTrace stackTrace = event.getStackTrace();
                    if (stackTrace != null && !stackTrace.getFrames().isEmpty()) {
                        String topMethod = stackTrace.getFrames().get(0)
                            .getMethod().getType().getName() + "." +
                            stackTrace.getFrames().get(0).getMethod().getName();
                        methodCounts.merge(topMethod, 1L, Long::sum);
                    }
                }
            }
        }

        // Print top 10 hot methods
        methodCounts.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(10)
            .forEach(e -> System.out.printf("%6d  %s%n", e.getValue(), e.getKey()));
    }
}
```

---

## 6. Memory Profiling

```bash
# Heap dump (snapshot of all objects on heap)
jmap -dump:format=b,file=heap.hprof <pid>

# Or trigger on OutOfMemoryError
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heap.hprof \
     -jar app.jar

# Histogram (quick object count summary)
jmap -histo <pid> | head -30

# JFR allocation profiling
jcmd <pid> JFR.start name=alloc settings=profile
```

```java
// Analyze heap dump programmatically
// Use Eclipse MAT (Memory Analyzer Tool) for:
// 1. Dominator tree - what retains the most memory
// 2. Leak suspects report - automatic leak detection
// 3. Histogram - object count and size by class
// 4. Path to GC roots - why objects are retained

// Common memory leak patterns to look for:
// - Growing collections (HashMap, ArrayList) never cleared
// - Listeners/callbacks never unregistered
// - ThreadLocal values never removed
// - Static fields holding references
// - Unclosed resources (connections, streams)
```

---

## 7. Lock Contention Profiling

```bash
# JFR lock profiling
java -XX:StartFlightRecording=settings=profile -jar app.jar
# Monitor: jdk.JavaMonitorEnter events

# async-profiler lock profiling
./asprof -e lock -d 30 -f locks.html <pid>

# Thread dump for quick contention check
jstack <pid>
# Look for: BLOCKED, WAITING states
```

```java
// Programmatic thread contention monitoring
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;
import java.lang.management.ThreadInfo;

public class ContentionMonitor {
    public static void detectContention() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        threadBean.setThreadContentionMonitoringEnabled(true);

        ThreadInfo[] threads = threadBean.dumpAllThreads(true, true);
        for (ThreadInfo thread : threads) {
            if (thread.getBlockedCount() > 0) {
                System.out.printf("Thread: %s%n", thread.getThreadName());
                System.out.printf("  Blocked count: %d%n", thread.getBlockedCount());
                System.out.printf("  Blocked time:  %d ms%n", thread.getBlockedTime());
                System.out.printf("  Waited count:  %d%n", thread.getWaitedCount());
                System.out.printf("  Waited time:   %d ms%n", thread.getWaitedTime());
                if (thread.getLockName() != null) {
                    System.out.printf("  Waiting on:    %s%n", thread.getLockName());
                    System.out.printf("  Lock owner:    %s%n", thread.getLockOwnerName());
                }
            }
        }

        // Deadlock detection
        long[] deadlockedThreads = threadBean.findDeadlockedThreads();
        if (deadlockedThreads != null) {
            System.out.println("DEADLOCK DETECTED!");
            for (ThreadInfo info : threadBean.getThreadInfo(deadlockedThreads)) {
                System.out.println("  " + info.getThreadName());
            }
        }
    }
}
```

---

## 8. Flame Graphs

```
Flame graphs visualize sampled stack traces:
- X-axis: stack trace population (wider = more samples)
- Y-axis: stack depth (bottom = entry point, top = leaf method)
- Color: random or grouped by category

Reading a flame graph:
1. Wide plateaus at the top = hot methods (optimization targets)
2. Wide bases = common code paths
3. Narrow spikes = rarely executed deep stacks
```

```bash
# Generate flame graph with async-profiler
./asprof -d 30 -f flamegraph.html <pid>

# Generate from JFR recording (using JMC or converter)
# In JMC: Window -> Show View -> Flame Graph

# Convert JFR to flame graph format
jfr print --events jdk.ExecutionSample --stack-depth 64 recording.jfr \
  | stackcollapse-jfr.pl | flamegraph.pl > flamegraph.svg
```

---

## 9. Continuous Profiling in Production

```java
// Always-on lightweight profiling with JFR
// Configuration for production (minimal overhead):

// jfr-production.jfc settings:
// - CPU sampling: every 20ms (not 10ms)
// - Allocation sampling: threshold 1MB (not 100KB)
// - Disable heavy events: jdk.ObjectAllocationInNewTLAB, etc.

// Start continuous recording with rotation
// jcmd <pid> JFR.start name=continuous \
//   maxage=1h maxsize=100m disk=true \
//   settings=default filename=continuous.jfr
```

```bash
# Production-safe JFR configuration
java -XX:StartFlightRecording=\
  name=continuous,\
  maxage=6h,\
  maxsize=500m,\
  disk=true,\
  dumponexit=true,\
  settings=default \
  -jar app.jar

# Dump on demand when issue occurs
jcmd <pid> JFR.dump name=continuous filename=incident.jfr
```

---

## 10. Profiling Tool Selection Guide

| Need                      | Tool                  | When to Use                |
|---------------------------|-----------------------|----------------------------|
| Production CPU profiling  | JFR + JMC             | Always-on monitoring       |
| Accurate CPU profiling    | async-profiler        | Performance investigation  |
| Memory leak detection     | Eclipse MAT + heap    | OOM or growing memory      |
| Quick overview            | VisualVM              | Development/debugging      |
| Lock contention           | JFR or async-profiler | Threading issues           |
| Flame graph               | async-profiler        | Identifying hot code paths |
| GC analysis               | GCViewer / GCEasy     | GC tuning                  |
| Continuous monitoring     | JFR (always-on)       | Production monitoring      |

---

## Interview Tips

- JFR is production-safe (< 1% overhead) and built into the JDK since Java 11
- async-profiler provides the most accurate CPU profiles (no safepoint bias)
- Always start with profiling before optimizing; never optimize by guessing
- Flame graphs are the best visualization for finding CPU hotspots
- JMC's automated analysis can detect common issues automatically
- Use `-XX:+HeapDumpOnOutOfMemoryError` in all production JVMs
- Continuous JFR recording is like a flight recorder: always on, dump on incident

---

[<< Previous: Caching Strategies](./q09-caching-strategies.md) | [Next: Benchmarking with JMH >>](./q11-benchmarking-jmh.md)
