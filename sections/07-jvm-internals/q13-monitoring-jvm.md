# Q13: How to Monitor JVM Performance?

## 📋 Question
What tools and techniques are available for monitoring JVM performance? How do you diagnose memory leaks, thread issues, and performance bottlenecks?

## Difficulty: 🟠 Hard

---

## 💡 Answer

JVM monitoring involves tracking **memory usage**, **garbage collection**, **thread behavior**, **CPU consumption**, and **application-level metrics**. The JDK ships with powerful built-in tools, and there are excellent third-party options for production monitoring.

### JVM Monitoring Tools Overview

```
┌═══════════════════════════════════════════════════════════════════┐
║                    JVM MONITORING TOOLS                          ║
║                                                                   ║
║  ┌──────────────────── GUI TOOLS ─────────────────────────────┐  ║
║  │  JConsole        — Basic monitoring (included in JDK)      │  ║
║  │  VisualVM        — Advanced profiling (download separately) │  ║
║  │  JMC + JFR       — Low-overhead production profiling       │  ║
║  │  IntelliJ/Eclipse — IDE-integrated profiling               │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                   ║
║  ┌──────────────── COMMAND-LINE TOOLS ────────────────────────┐  ║
║  │  jcmd            — Send diagnostic commands to JVM         │  ║
║  │  jstat           — GC and class loading statistics          │  ║
║  │  jmap            — Heap dump and memory map                 │  ║
║  │  jstack          — Thread dump                              │  ║
║  │  jinfo           — JVM configuration info                   │  ║
║  │  jps             — List running Java processes              │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                   ║
║  ┌──────────────── ANALYSIS TOOLS ────────────────────────────┐  ║
║  │  MAT (Eclipse)   — Heap dump analysis                      │  ║
║  │  GCEasy          — GC log analysis (online)                │  ║
║  │  async-profiler  — Low-overhead CPU/allocation profiling   │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                   ║
║  ┌──────────────── APM TOOLS ─────────────────────────────────┐  ║
║  │  Prometheus + Grafana — Metrics collection + dashboards    │  ║
║  │  Datadog, New Relic   — Commercial APM                     │  ║
║  │  Micrometer           — Metrics facade for Spring Boot     │  ║
║  └────────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════╝
```

### Command-Line Tools

#### jcmd — The Swiss Army Knife

```bash
# List all Java processes
jcmd

# Get the main class and arguments
jcmd <pid> VM.command_line

# Print JVM flags
jcmd <pid> VM.flags

# Trigger a heap dump
jcmd <pid> GC.heap_dump /tmp/heapdump.hprof

# Print thread dump
jcmd <pid> Thread.print

# Print class histogram (objects by class)
jcmd <pid> GC.class_histogram

# Trigger garbage collection
jcmd <pid> GC.run

# Print Metaspace details (Java 11+)
jcmd <pid> VM.metaspace

# Start Java Flight Recorder
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr

# Print system properties
jcmd <pid> VM.system_properties

# Native memory tracking (requires -XX:NativeMemoryTracking=summary)
jcmd <pid> VM.native_memory summary
```

#### jstat — GC Statistics

```bash
# GC utilization — most useful for quick checks
jstat -gcutil <pid> 1000    # Print every 1 second

# Output columns:
#  S0     S1     E      O      M     CCS    YGC   YGCT   FGC  FGCT   CGC  CGCT    GCT
# 0.00  98.44  65.34  23.12  97.89  95.23   152   1.234    3  0.567    12  0.089   1.890
#
# S0/S1:  Survivor space utilization (%)
# E:      Eden utilization (%)
# O:      Old Gen utilization (%)
# M:      Metaspace utilization (%)
# CCS:    Compressed Class Space utilization (%)
# YGC:    Young GC count
# YGCT:   Young GC total time (seconds)
# FGC:    Full GC count
# FGCT:   Full GC total time (seconds)
# GCT:    Total GC time

# ⚠️ Warning signs:
# O > 90% consistently     → Old Gen filling up, potential OOM
# M > 95%                  → Metaspace nearly full
# FGC increasing rapidly   → Too many Full GCs (performance impact)
# FGCT/FGC > 1 second      → Full GC pauses too long

# GC capacity details
jstat -gccapacity <pid> 1000

# Metaspace capacity
jstat -gcmetacapacity <pid> 1000

# Class loading stats
jstat -class <pid> 1000
```

#### jmap — Heap Analysis

```bash
# Create a heap dump
jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# Create a heap dump of live objects only (triggers GC first)
jmap -dump:live,format=b,file=/tmp/heap-live.hprof <pid>

# Print class histogram (quick memory analysis)
jmap -histo <pid> | head -30

# Output:
#  num     #instances  #bytes  class name
# ───────────────────────────────────────
#    1:       1234567  98765432  [B  (byte arrays)
#    2:        567890  45678900  java.lang.String
#    3:        345678  27654240  java.util.HashMap$Node
#    4:        234567  18765360  [Ljava.lang.Object;
# ...

# ⚠️ If you see millions of instances of a specific class
#    that shouldn't be there → likely memory leak
```

#### jstack — Thread Analysis

```bash
# Print thread dump
jstack <pid>

# Print with lock information
jstack -l <pid>

# Save to file for analysis
jstack -l <pid> > /tmp/threaddump.txt

# ⚡ Take 3 dumps 5 seconds apart to identify stuck threads:
for i in 1 2 3; do
  jstack -l <pid> > /tmp/thread_$i.txt
  sleep 5
done
```

```
# Example thread dump output:
"http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0
   java.lang.Thread.State: RUNNABLE
   at com.example.service.UserService.processRequest(UserService.java:42)
   at com.example.controller.UserController.getUser(UserController.java:28)
   ...

"http-nio-8080-exec-2" #26 daemon prio=5 os_prio=0
   java.lang.Thread.State: BLOCKED (on object monitor)
   - waiting to lock <0x00000000c0d03e10> (a java.util.HashMap)
   at com.example.cache.SharedCache.get(SharedCache.java:15)
   - locked <0x00000000c0d04120> (a java.lang.Object)
   ...

# Key states to look for:
# RUNNABLE      — actively executing or ready to run
# BLOCKED       — waiting for a monitor lock (⚠️ contention)
# WAITING       — waiting indefinitely (Object.wait, Thread.join)
# TIMED_WAITING — waiting with timeout (Thread.sleep, wait(timeout))

# Deadlock detection (at bottom of jstack output):
# "Found one Java-level deadlock:"
# Thread A waiting for lock held by Thread B
# Thread B waiting for lock held by Thread A
```

### GUI Tools

#### JConsole — Basic Monitoring

```bash
# Launch JConsole (included in JDK)
jconsole

# Or connect to remote JVM:
jconsole hostname:port

# Remote JVM must be started with:
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

# JConsole provides:
# ✅ Real-time heap usage graphs
# ✅ Thread count and states
# ✅ Class loading count
# ✅ MBean inspection
# ❌ No profiling or heap dump analysis
```

#### VisualVM — Advanced Profiling

```bash
# Download: https://visualvm.github.io/
# Provides:
# ✅ Real-time memory, CPU, thread monitoring
# ✅ CPU and memory profiling (sampling or instrumentation)
# ✅ Heap dump creation and basic analysis
# ✅ Thread dump analysis
# ✅ MBean inspection
# ✅ Plugin ecosystem (Visual GC, etc.)
```

#### Java Mission Control (JMC) + Java Flight Recorder (JFR)

```bash
# JFR — ultra-low overhead production profiling (~1-2% overhead)
# Built into the JVM since Java 11 (free for commercial use)

# Start recording from command line
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar myapp.jar

# Start recording on a running JVM
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr

# Continuous recording (ring buffer)
java -XX:StartFlightRecording=disk=true,maxsize=500m,maxage=1d -jar myapp.jar

# Dump the current ring buffer
jcmd <pid> JFR.dump filename=/tmp/current.jfr

# Open recording in JMC for analysis
jmc   # GUI tool included in JDK 11+
```

```java
// Programmatic JFR events (Java 14+)
import jdk.jfr.*;

@Label("Order Processing")
@Category("Business")
@Description("Tracks order processing time")
public class OrderEvent extends Event {
    @Label("Order ID")
    long orderId;

    @Label("Processing Time (ms)")
    long processingTimeMs;

    @Label("Status")
    String status;
}

// Usage:
OrderEvent event = new OrderEvent();
event.begin();
// ... process order ...
event.orderId = 12345;
event.processingTimeMs = elapsed;
event.status = "SUCCESS";
event.commit();  // Recorded in JFR with minimal overhead
```

### Heap Dump Analysis with Eclipse MAT

```bash
# Generate heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof
# OR
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
# OR automatically on OOM:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/

# Open in Eclipse MAT (Memory Analyzer Tool)
# Download: https://eclipse.dev/mat/

# Key MAT Features:
# 1. Leak Suspects Report — automated detection of likely leaks
# 2. Dominator Tree — largest objects and what retains them
# 3. Histogram — object count and size by class
# 4. Path to GC Roots — why an object isn't collected
# 5. OQL (Object Query Language) — SQL-like queries on heap

# Example OQL queries:
# SELECT * FROM java.lang.String WHERE toString().length() > 1000
# SELECT * FROM INSTANCEOF java.util.HashMap WHERE size > 10000
```

### Practical Monitoring Workflow

```
Performance Issue Detected
│
├── High CPU?
│   ├── jstack (3 dumps, 5s apart) → find busy threads
│   ├── async-profiler → CPU flame graph
│   └── JFR recording → hottest methods
│
├── High Memory / OOM?
│   ├── jstat -gcutil → heap usage trend
│   ├── jmap -histo → top memory consumers
│   ├── Heap dump + MAT → find leak root cause
│   └── GC logs → allocation rate, promotion rate
│
├── Slow Response?
│   ├── jstack → check for BLOCKED threads (lock contention)
│   ├── JFR → method profiling, I/O waits
│   └── GC logs → check for long GC pauses
│
└── High GC Overhead?
    ├── GC logs → pause times, frequency, type (Minor/Major/Full)
    ├── jstat -gcutil → generation fill rates
    └── Tune: heap size, GC algorithm, generation ratios
```

### ⚠️ Common Mistake
Using `jmap -dump` on a production JVM during peak traffic. Heap dumps trigger a Full GC and pause the application for the duration of the dump (which can be minutes for large heaps). Use **JFR** instead for continuous production monitoring — it has < 2% overhead and doesn't stop the application. If you must take a heap dump, do it during low-traffic periods or on a single node behind a load balancer.

### ⚡ Performance Tip
Set up **proactive monitoring** rather than reactive debugging. Configure GC logging from day one (`-Xlog:gc*`), enable JFR continuous recording (`-XX:StartFlightRecording`), and expose JMX metrics to Prometheus/Grafana. When problems occur, you'll have historical data to compare against. For Spring Boot, add `micrometer-registry-prometheus` and expose the `/actuator/prometheus` endpoint for automatic JVM metrics collection.

```bash
# Minimum production monitoring setup:
java \
  -Xlog:gc*=info:file=gc.log:time:filecount=5,filesize=20m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/ \
  -XX:StartFlightRecording=disk=true,maxsize=500m,maxage=1d \
  -Dcom.sun.management.jmxremote.port=9010 \
  -jar myapp.jar
```

---

## 🎯 Interviewer Insight
Interviewers want to see that you have hands-on experience with JVM monitoring tools. Mentioning `jcmd`, `jstat`, `jstack`, and `jmap` with specific use cases shows practical knowledge. Knowing about JFR for production profiling is a strong differentiator. The best candidates can describe a complete diagnostic workflow: "I would start with jstat to check GC, then take a thread dump with jstack, and if needed, create a heap dump for MAT analysis."

## ⚡ Follow-up Questions
1. How would you diagnose a memory leak in production?
2. What is the overhead of Java Flight Recorder?
3. How do you detect a deadlock?

## 📚 Related Topics
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)
- [GC Collectors](./q06-gc-collectors.md)
- [Stack vs Heap](./q09-stack-vs-heap.md)
