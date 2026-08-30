# Q12: Heap Dump Analysis Techniques

[← Back to Memory Management](./README.md)

---

## Overview

A **heap dump** is a snapshot of all objects in the Java heap at a specific point in time. Analyzing heap dumps is the definitive technique for diagnosing memory leaks, understanding memory consumption patterns, and optimizing application memory usage.

---

## Generating Heap Dumps

### 1. Automatic on OutOfMemoryError (Recommended for Production)

```bash
# ✅ Always enable this in production
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/dumps/app-heap.hprof \
     -jar myapp.jar

# The dump captures the exact state when OOM occurs
# File can be very large (equal to -Xmx)
```

### 2. Using jmap

```bash
# Generate heap dump from a running process
jmap -dump:format=b,file=/tmp/heapdump.hprof <pid>

# Live objects only (forces a GC first)
jmap -dump:live,format=b,file=/tmp/heapdump.hprof <pid>

# Quick histogram (no full dump — lighter weight)
jmap -histo <pid> | head -30
jmap -histo:live <pid> | head -30     # Live objects only
```

### 3. Using jcmd (Preferred over jmap)

```bash
# Generate heap dump
jcmd <pid> GC.heap_dump /tmp/heapdump.hprof

# With live-object filtering
jcmd <pid> GC.heap_dump -all=false /tmp/heapdump.hprof

# Class histogram
jcmd <pid> GC.class_histogram
```

### 4. Programmatically (via HotSpotDiagnosticMXBean)

```java
import com.sun.management.HotSpotDiagnosticMXBean;
import java.lang.management.ManagementFactory;

public class HeapDumper {

    public static void dumpHeap(String filePath, boolean live) throws Exception {
        HotSpotDiagnosticMXBean diagnosticMXBean =
                ManagementFactory.getPlatformMXBean(HotSpotDiagnosticMXBean.class);

        diagnosticMXBean.dumpHeap(filePath, live);
        System.out.println("Heap dump written to: " + filePath);
    }

    // Trigger via JMX, REST endpoint, or signal handler
    public static void main(String[] args) throws Exception {
        dumpHeap("/tmp/programmatic-dump.hprof", true);
    }
}
```

### 5. Using VisualVM / JMC

```
VisualVM: Monitor tab → Heap Dump button
JMC: MBean Server → com.sun.management → HotSpotDiagnostic → dumpHeap()
```

---

## Eclipse MAT (Memory Analyzer Tool)

The most powerful open-source heap dump analyzer:

```bash
# Download from: https://eclipse.dev/mat/
# Launch with increased memory for large dumps:
MemoryAnalyzer -vmargs -Xmx8g
```

### Key MAT Features

#### 1. Overview / Pie Chart

```
MAT Overview:
┌───────────────────────────────────────┐
│  Total Heap: 2.1 GB                   │
│  ┌─────────────────────────────────┐  │
│  │  byte[]           35%  █████    │  │
│  │  String            12%  ██      │  │
│  │  HashMap$Node       8%  █       │  │
│  │  UserSession        7%  █       │  │  ← Suspect?
│  │  char[]             6%  █       │  │
│  │  Other             32%  ████    │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
```

#### 2. Histogram

Shows object counts and memory per class:

```
Class Name                    | Objects |  Shallow Heap | Retained Heap
─────────────────────────────────────────────────────────────────────────
byte[]                        | 1,500,000 |  750,000,000 |  750,000,000
java.lang.String              |   800,000 |   19,200,000 |  400,000,000
java.util.HashMap$Node        |   600,000 |   19,200,000 |  300,000,000
com.example.UserSession       |   200,000 |   12,800,000 |  850,000,000 ← !
com.example.RequestData       |   200,000 |    9,600,000 |  200,000,000
```

> **Key insight:** Look for classes with **high retained heap** relative to **shallow heap** — they hold references to large subgraphs.

#### 3. Dominator Tree

Shows which objects "dominate" (keep alive) the most memory:

```
Dominator Tree:
├── java.util.HashMap @ 0x7f001000                    1,200 MB (57%)
│   ├── java.util.HashMap$Node[2048] @ 0x7f001100       800 MB
│   │   ├── com.example.UserSession @ 0x7f002000        4.2 MB
│   │   │   ├── byte[] (requestBody)                    4.0 MB
│   │   │   └── java.lang.String (userId)               0.1 MB
│   │   ├── com.example.UserSession @ 0x7f003000        4.1 MB
│   │   │   └── ...
│   │   └── ... (200,000 more UserSession objects)
│   └── ...
├── java.lang.Thread "main" @ 0x7f005000                 50 MB (2.4%)
└── ...
```

> **Interpretation:** A single `HashMap` dominates 57% of the heap. It holds 200K `UserSession` objects. This is likely a cache or map that's never cleaned.

#### 4. Leak Suspects Report

MAT automatically identifies potential leaks:

```
Leak Suspect #1:
  Problem: One instance of "java.util.HashMap" loaded by "<system class loader>"
  occupies 1,200 MB (57.14%) of the heap.

  The memory is accumulated in one instance of "java.util.HashMap$Node[]"
  loaded by "<system class loader>".

  Keywords: java.util.HashMap
  
  Shortest Path to GC Root:
    com.example.SessionManager.activeSessions  ← STATIC FIELD
      └── java.util.HashMap
          └── java.util.HashMap$Node[]
              └── com.example.UserSession (× 200,000)
```

---

## Shallow Size vs Retained Size

Understanding these concepts is critical for heap dump analysis:

```
Shallow Size:
  Memory consumed by the object itself (header + fields)
  Does NOT include referenced objects

Retained Size:
  Memory that would be freed if THIS object were garbage collected
  Includes all objects exclusively reachable from this object


Example:
  HashMap (shallow: 48 bytes)
    └── Node[] (shallow: 8,192 bytes)
        ├── Node → Key: "user1" → Value: UserSession (4.2 MB)
        ├── Node → Key: "user2" → Value: UserSession (4.1 MB)
        └── ... (200K entries)

  HashMap shallow size:  48 bytes
  HashMap retained size: 850 MB (the entire subgraph it dominates)
```

```java
// Common interview question: What's the shallow size of this object?
public class Example {
    int x;           // 4 bytes
    long y;          // 8 bytes
    String name;     // 4 bytes (reference, compressed oops)
    Object[] data;   // 4 bytes (reference)
}
// Object header: 12 bytes
// Fields: 4 + 8 + 4 + 4 = 20 bytes
// Padding: 0 bytes (12 + 20 = 32, already aligned to 8)
// Shallow size: 32 bytes
// Retained size: 32 bytes + size of String + size of Object[]
//                (only if they're not referenced from elsewhere)
```

---

## GC Roots

Understanding GC roots is essential for tracing why objects are alive:

```
GC Roots — objects that are always reachable:

1. Thread stacks       → Local variables and method parameters
2. Static fields       → Referenced from Class objects
3. JNI references      → Objects referenced from native code
4. Monitor locks       → Objects currently synchronized on
5. System classes      → java.lang.*, etc.
```

### Tracing a path from an object to its GC root in MAT

```
Right-click object → Path to GC Roots → exclude weak/soft references

Result:
  com.example.UserSession @ 0x7f002000
    └── value in java.util.HashMap$Node @ 0x7f001200
        └── element in java.util.HashMap$Node[] @ 0x7f001100
            └── table in java.util.HashMap @ 0x7f001000
                └── activeSessions in com.example.SessionManager    ← GC ROOT (static field)

Diagnosis: UserSession is kept alive because it's in the static
           SessionManager.activeSessions HashMap that's never cleaned.
```

---

## OQL (Object Query Language)

MAT supports SQL-like queries over heap dumps:

```sql
-- Find all Strings longer than 10,000 characters
SELECT s, s.value.@length
FROM java.lang.String s
WHERE s.value.@length > 10000

-- Find all HashMaps with more than 1000 entries
SELECT m, m.size
FROM java.util.HashMap m
WHERE m.size > 1000

-- Find UserSessions created with a specific userId
SELECT s
FROM com.example.UserSession s
WHERE s.userId.toString() = "admin"

-- Find objects retained by a specific class
SELECT OBJECTS dominators(h)
FROM java.util.HashMap h
WHERE h.size > 10000

-- Find duplicate Strings (memory waste)
SELECT s.toString(), COUNT(s), SUM(s.@retainedHeapSize)
FROM java.lang.String s
GROUP BY s.toString()
HAVING COUNT(s) > 100
```

---

## VisualVM Heap Analysis

```
Steps:
1. Open VisualVM → Connect to application
2. Monitor tab → Click "Heap Dump"
3. In the Heap Dump tab:
   - Summary: top classes by instance count and size
   - Classes: sortable list of all loaded classes
   - Instances: browse individual instances
   - OQL Console: run queries
```

---

## Practical Walkthrough: Finding a Memory Leak

### Scenario
A Spring Boot application's Old Gen keeps growing, eventually causing OOM.

### Step 1: Capture heap dumps at intervals

```bash
# Take dump when healthy (baseline)
jcmd <pid> GC.heap_dump /tmp/baseline.hprof

# Wait for memory to grow
# Take dump when memory is high
jcmd <pid> GC.heap_dump /tmp/suspect.hprof

# Or automatically on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/ -jar app.jar
```

### Step 2: Open in MAT and compare

```
1. Open baseline.hprof in MAT
2. Open suspect.hprof in MAT
3. Compare histograms:
   
   Class                    | Baseline  | Suspect   | Delta
   ──────────────────────────────────────────────────────────
   com.example.AuditEvent   |   1,000   | 500,000   | +499,000 ← Growing!
   byte[]                   |  50,000   | 600,000   | +550,000
   java.lang.String         |  30,000   | 350,000   | +320,000
```

### Step 3: Find what's holding AuditEvents

```
In suspect.hprof:
1. Histogram → Right-click "com.example.AuditEvent"
2. List Objects → with incoming references
3. Path to GC Roots → exclude weak references

Result:
  AuditEvent @ 0x7f002000
    └── elementData in java.util.ArrayList @ 0x7f001000
        └── auditLog in com.example.AuditService        ← STATIC FIELD!
```

### Step 4: Find the root cause in code

```java
// ❌ The leak: static list that only grows
@Service
public class AuditService {

    private static final List<AuditEvent> auditLog = new ArrayList<>();  // LEAK!

    public void recordEvent(AuditEvent event) {
        auditLog.add(event);   // Never removed, never bounded
    }
}
```

### Step 5: Fix the leak

```java
// ✅ Fix: bounded buffer + async persistence
@Service
public class AuditService {

    private final BlockingQueue<AuditEvent> buffer =
            new LinkedBlockingQueue<>(10_000);   // Bounded

    private final AuditRepository repository;

    @Scheduled(fixedRate = 5000)
    public void flushEvents() {
        List<AuditEvent> batch = new ArrayList<>();
        buffer.drainTo(batch, 1000);
        if (!batch.isEmpty()) {
            repository.saveAll(batch);   // Persist and release
        }
    }

    public void recordEvent(AuditEvent event) {
        if (!buffer.offer(event)) {
            log.warn("Audit buffer full — dropping event");
        }
    }
}
```

### Step 6: Verify the fix

```bash
# Run with monitoring, confirm sawtooth heap pattern
java -Xlog:gc*:file=gc.log:time \
     -XX:+HeapDumpOnOutOfMemoryError \
     -jar app.jar

# Monitor with VisualVM — Old Gen should stabilize
```

---

## Common Heap Dump Patterns

| Pattern                                    | Likely Cause                            |
|--------------------------------------------|-----------------------------------------|
| One HashMap dominates heap                 | Unbounded cache/registry                |
| Millions of same class instances           | Missing eviction or cleanup             |
| Large `byte[]` attached to Sessions        | Session data not expired                |
| Duplicate Strings                          | Missing `intern()` or dedup             |
| ClassLoader retaining many classes         | Class loader leak (app server redeploy) |
| `ThreadLocal` entries in old threads       | ThreadLocal not cleaned in thread pool  |

---

## Best Practices

```bash
# ✅ Always set these JVM flags in production:
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/dumps/

# ✅ Take periodic heap dumps during load tests
# ✅ Compare "before" and "after" dumps to find growing classes
# ✅ Use MAT's dominator tree first — it shows the biggest impact
# ✅ Use "Path to GC Roots" to understand WHY objects are retained
# ✅ Use OQL for complex queries (finding duplicates, filtering by field values)
```

```java
// ✅ Automate leak detection in CI/CD
// Run load test → take heap dump → assert no unexpected growth
// Tools: JMH + heap dump comparison in test pipeline

// ❌ Don't analyze heap dumps on the production server (CPU/memory intensive)
// ❌ Don't ignore retained size — shallow size can be misleading
// ❌ Don't forget to exclude weak/soft references when tracing GC roots
// ❌ Don't take heap dumps during peak traffic (application pauses)
```

---

## Interview Tips
- A heap dump captures **all live objects** — it's the definitive tool for memory leak diagnosis
- **Dominator tree** shows which objects keep the most memory alive — start there
- **Shallow size** = the object alone; **Retained size** = the object + everything it exclusively holds
- **GC roots** are the starting points of reachability — static fields and thread stacks are the most common
- Always use `-XX:+HeapDumpOnOutOfMemoryError` in production — you won't get a second chance
- **Eclipse MAT** is the industry standard for heap analysis; **OQL** lets you query the dump like a database

---

[← Q11: Profiling Memory](./q11-profiling-memory.md) | [Back to Memory Management →](./README.md)
