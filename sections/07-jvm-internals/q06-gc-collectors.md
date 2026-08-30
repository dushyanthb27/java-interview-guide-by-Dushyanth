# Q6: Compare Garbage Collectors (G1, ZGC, Shenandoah)

## 📋 Question
Compare the major JVM garbage collectors — G1, ZGC, Shenandoah, Serial, Parallel, and CMS. When should you use each?

## Difficulty: 🔴 Expert

---

## 💡 Answer

The JVM offers multiple garbage collectors, each designed for different workload characteristics. The right choice depends on your application's **heap size**, **latency requirements**, and **throughput needs**.

### Garbage Collector Overview

| Collector | Introduced | Default? | Focus | Max Pause | Heap Size | Status |
|-----------|-----------|----------|-------|-----------|-----------|--------|
| **Serial** | JDK 1.0 | No | Simplicity | High (seconds) | Small (<100 MB) | Active |
| **Parallel** | JDK 1.4 | Java 8 default | Throughput | Medium (100ms+) | Medium (1-4 GB) | Active |
| **CMS** | JDK 1.4 | No | Low latency | Low-Medium | Medium | **Removed (Java 14)** |
| **G1** | JDK 7u4 | **Java 9+ default** | Balanced | Low (10-200ms) | Large (4-64 GB) | Active |
| **ZGC** | JDK 11 | No | Ultra-low latency | **Sub-ms** | **Multi-TB** | Active |
| **Shenandoah** | JDK 12 | No | Low latency | **Sub-ms** | Large | Active (OpenJDK) |

### Serial Collector

```
┌──────────────────────────────────────────┐
│         Serial GC (-XX:+UseSerialGC)     │
│                                          │
│  App Threads: ████████░░░░░░░████████    │
│  GC Thread:   ░░░░░░░░████████░░░░░░░   │
│               running  STW     running   │
│                                          │
│  ✅ Simple, low overhead                 │
│  ✅ Best for single-core / small heaps   │
│  ❌ Single GC thread — long pauses       │
│  ❌ Not suitable for server apps         │
└──────────────────────────────────────────┘

Young Gen: Copying algorithm (single thread)
Old Gen:   Mark-Compact (single thread)
```

### Parallel Collector (Throughput Collector)

```
┌──────────────────────────────────────────┐
│     Parallel GC (-XX:+UseParallelGC)     │
│                                          │
│  App Threads: ████████░░░░████████████   │
│  GC Thread 1: ░░░░░░░░██░░░░░░░░░░░░░   │
│  GC Thread 2: ░░░░░░░░██░░░░░░░░░░░░░   │
│  GC Thread 3: ░░░░░░░░██░░░░░░░░░░░░░   │
│  GC Thread 4: ░░░░░░░░██░░░░░░░░░░░░░   │
│               running  STW  running      │
│                                          │
│  ✅ High throughput (max CPU for app)    │
│  ✅ Multiple GC threads → shorter pauses │
│  ❌ Still STW for all phases             │
│  ❌ Pauses grow with heap size           │
└──────────────────────────────────────────┘

Young Gen: Parallel Copying
Old Gen:   Parallel Mark-Compact
```

### G1 Collector (Garbage-First) — Default Since Java 9

```
┌════════════════════════════════════════════════════════════════┐
║                 G1 GC (-XX:+UseG1GC)                          ║
║                                                                ║
║  Heap divided into equal-sized REGIONS (1-32 MB each):         ║
║                                                                ║
║  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐     ║
║  │ E  │ E  │ S  │    │ O  │ O  │ H  │ H  │ O  │ E  │    │     ║
║  ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤     ║
║  │ O  │    │ E  │ O  │ O  │    │ E  │ S  │    │ O  │ O  │     ║
║  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘     ║
║                                                                ║
║  E = Eden    S = Survivor    O = Old    H = Humongous          ║
║  (blank) = Free                                                ║
║                                                                ║
║  Key Features:                                                 ║
║  • Region-based heap — no contiguous Young/Old Gen             ║
║  • Pause time goal: -XX:MaxGCPauseMillis=200 (default)         ║
║  • Collects regions with most garbage first (Garbage-First)    ║
║  • Concurrent marking phase (mostly non-STW)                   ║
║  • Mixed collections: collect Young + some Old regions          ║
╚════════════════════════════════════════════════════════════════╝
```

#### G1 Collection Phases

```
G1 Collection Cycle:

1. Young-Only Collection (STW)
   ├── Evacuate all Eden + Survivor regions
   └── Copy live objects to Survivor or Old regions

2. Concurrent Marking (mostly concurrent)
   ├── Initial Mark (STW, piggyback on young GC)
   ├── Root Region Scan (concurrent)
   ├── Concurrent Mark (concurrent — marks live objects)
   ├── Remark (STW — finalize marking)
   └── Cleanup (STW — identify empty regions, sort by garbage)

3. Mixed Collection (STW)
   ├── Collect ALL young regions + SOME old regions
   ├── Selects old regions with most garbage (bang for buck)
   └── Targets -XX:MaxGCPauseMillis goal
```

#### Humongous Objects

```java
// Objects larger than 50% of a G1 region are "humongous"
// They are allocated in contiguous regions in Old Gen
// -XX:G1HeapRegionSize=16m → objects > 8 MB are humongous

byte[] huge = new byte[10 * 1024 * 1024];  // 10 MB — humongous!

// ⚠️ Humongous allocations:
//  - Skip Young Gen entirely
//  - May trigger Full GC if not enough contiguous free regions
//  - Can cause fragmentation
//  - Collected during cleanup or Full GC (Java 8u40+ can collect in young GC)
```

### ZGC (Z Garbage Collector) — Sub-millisecond Pauses

```
┌════════════════════════════════════════════════════════════════┐
║            ZGC (-XX:+UseZGC)  [Java 15+ production-ready]     ║
║                                                                ║
║  Design Goals:                                                 ║
║  • Pause times < 1 ms (regardless of heap size!)               ║
║  • Support heaps from 8 MB to 16 TB                            ║
║  • Throughput reduction < 15% vs Parallel GC                   ║
║                                                                ║
║  Key Innovation: COLORED POINTERS + LOAD BARRIERS              ║
║                                                                ║
║  64-bit Object Pointer:                                        ║
║  ┌────────┬──┬──┬──┬──┬──────────────────────────────────┐    ║
║  │ unused │M1│M0│Re│Fi│   Object Address (42 bits = 4 TB) │    ║
║  └────────┴──┴──┴──┴──┴──────────────────────────────────┘    ║
║   bits:       Marked0, Marked1, Remapped, Finalizable          ║
║                                                                ║
║  Load Barrier: Injected when loading a reference               ║
║  ┌─────────────────────────────────────────────────┐           ║
║  │  Object ref = obj.field;  // triggers barrier   │           ║
║  │  if (ref.color != expected) {                   │           ║
║  │      ref = slowPath(ref); // fix-up pointer     │           ║
║  │  }                                              │           ║
║  └─────────────────────────────────────────────────┘           ║
║                                                                ║
║  Almost ALL work is concurrent — no STW for marking/compacting ║
╚════════════════════════════════════════════════════════════════╝
```

#### ZGC Phases

```
ZGC Collection Cycle (almost entirely concurrent):

1. Pause Mark Start (STW)         ~microseconds
   └── Mark GC roots

2. Concurrent Mark                 runs alongside app
   └── Traverse object graph, mark live objects

3. Pause Mark End (STW)            ~microseconds
   └── Reference processing, weak references

4. Concurrent Relocate Prepare     runs alongside app
   └── Select regions to compact, build forwarding tables

5. Pause Relocate Start (STW)      ~microseconds
   └── Relocate root-referenced objects

6. Concurrent Relocate             runs alongside app
   └── Move objects, update references via load barriers

Total STW: < 1 ms (three very short pauses)
```

### Shenandoah GC — Concurrent Compaction

```
┌════════════════════════════════════════════════════════════════┐
║       Shenandoah (-XX:+UseShenandoahGC)  [OpenJDK only]       ║
║                                                                ║
║  Key Innovation: BROOKS POINTERS (forwarding pointers)         ║
║                                                                ║
║  Every object has an extra indirection pointer:                 ║
║                                                                ║
║  Before compaction:          During compaction:                 ║
║  ┌────────────────┐          ┌────────────────┐                ║
║  │ Brooks Pointer─┐│          │ Brooks Pointer──┐               ║
║  │ Object data    ││          │ Object data    │ │              ║
║  └───────┬────────┘│          └────────────────┘ │              ║
║          └─────────┘(self)                       ▼              ║
║                                         ┌────────────────┐     ║
║                                         │ Brooks Pointer─┐│     ║
║                                         │ Copied Object  ││     ║
║                                         └───────┬────────┘│     ║
║                                                 └─────────┘     ║
║                                                                ║
║  • Concurrent compaction (moves objects while app runs)         ║
║  • Uses CAS to update Brooks pointers atomically                ║
║  • Similar pause goals to ZGC (< 10 ms typical)                ║
║  • Available in OpenJDK (not Oracle JDK)                        ║
╚════════════════════════════════════════════════════════════════╝
```

### CMS (Concurrent Mark-Sweep) — Deprecated & Removed

```
CMS: Deprecated in Java 9, Removed in Java 14

Phases:
1. Initial Mark (STW)        — Mark direct GC root references
2. Concurrent Mark            — Traverse object graph (concurrent)
3. Concurrent Preclean        — Handle reference changes
4. Remark (STW)               — Final marking pass
5. Concurrent Sweep           — Free unmarked objects (concurrent)
6. Concurrent Reset           — Prepare for next cycle

⚠️ No compaction → fragmentation → eventual Full GC (Serial fallback)
⚠️ Promotion failure → long STW Full GC
Replaced by G1 as the recommended low-latency collector
```

### Selecting the Right GC

```
                        ┌──────────────────┐
                        │  What matters    │
                        │     most?        │
                        └────────┬─────────┘
                   ┌─────────────┼──────────────┐
                   ▼             ▼               ▼
            ┌──────────┐  ┌───────────┐   ┌──────────────┐
            │Throughput│  │Low Latency│   │Ultra-Low     │
            │(batch)   │  │(< 100ms)  │   │Latency(<1ms) │
            └────┬─────┘  └─────┬─────┘   └──────┬───────┘
                 │              │                 │
                 ▼              ▼                 ▼
           ┌──────────┐  ┌──────────┐     ┌─────────────┐
           │ Parallel │  │   G1     │     │  ZGC or     │
           │   GC     │  │          │     │ Shenandoah  │
           └──────────┘  └──────────┘     └─────────────┘

Decision Guide:
┌─────────────────────────┬─────────────────────────────────────┐
│ Scenario                │ Recommended GC                      │
├─────────────────────────┼─────────────────────────────────────┤
│ Small heap (< 100 MB)   │ Serial (-XX:+UseSerialGC)           │
│ Batch processing        │ Parallel (-XX:+UseParallelGC)       │
│ Web apps (4-64 GB)      │ G1 (-XX:+UseG1GC) ← DEFAULT        │
│ Trading / real-time     │ ZGC (-XX:+UseZGC)                   │
│ Large heap (> 16 GB)    │ ZGC or G1 with tuning               │
│ Multi-TB heap           │ ZGC (-XX:+UseZGC)                   │
│ Container / microservice│ G1 or Serial (small container)      │
└─────────────────────────┴─────────────────────────────────────┘
```

### GC Logging and Analysis

```bash
# Java 9+ unified GC logging
java -Xlog:gc*=info:file=gc.log:time,uptime,level,tags -jar myapp.jar

# Detailed G1 logging
java -Xlog:gc*=debug:file=gc.log:time,pid,tid,level -XX:+UseG1GC -jar myapp.jar

# Key metrics to watch:
# - Pause times (STW duration)
# - Throughput (% time not in GC)
# - Allocation rate (MB/s of new objects)
# - Promotion rate (MB/s moved to Old Gen)
# - Heap occupancy after GC

# Tools for GC log analysis:
# - GCEasy (gceasy.io) — online analyzer
# - GCViewer — open-source desktop tool
# - JClarity Censum — commercial analyzer
```

```java
// Programmatic GC monitoring
import java.lang.management.*;

public class GCMonitor {
    public static void main(String[] args) {
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.println("GC Name: " + gc.getName());
            System.out.println("  Collection count: " + gc.getCollectionCount());
            System.out.println("  Collection time:  " + gc.getCollectionTime() + " ms");
            System.out.println("  Memory pools:     " + String.join(", ", gc.getMemoryPoolNames()));
        }
        // Example output with G1:
        //   GC Name: G1 Young Generation
        //   GC Name: G1 Old Generation
    }
}
```

### ⚠️ Common Mistake
Choosing ZGC/Shenandoah without understanding the trade-offs. While they offer sub-millisecond pauses, they consume more CPU and memory overhead for their concurrent operations. For batch processing or small-heap applications, Parallel or Serial GC may deliver better overall throughput. Also, ZGC is only available on 64-bit platforms and was experimental before Java 15.

### ⚡ Performance Tip
Start with the **default G1 GC** and tune `MaxGCPauseMillis` before switching collectors. If G1 can't meet your latency requirements, then consider ZGC. Always benchmark with **production-representative workloads** — GC behavior varies dramatically with allocation patterns, object lifetimes, and heap sizes. Use GC logs (`-Xlog:gc*`) to make data-driven decisions.

```bash
# Start here — G1 with pause time goal
java -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -Xms4g -Xmx4g -jar myapp.jar

# If latency requirements < 10ms, try ZGC
java -XX:+UseZGC -Xms4g -Xmx4g -jar myapp.jar
```

---

## 🎯 Interviewer Insight
This is an expert-level question. Interviewers want to see that you can compare collectors along multiple dimensions (throughput, latency, heap size) and justify your choice for a given scenario. Strong candidates explain the key innovation of each modern collector (G1: regions + pause goals, ZGC: colored pointers, Shenandoah: Brooks pointers) and know that G1 is the default since Java 9.

## ⚡ Follow-up Questions
1. Why was CMS removed from the JVM?
2. How does ZGC achieve sub-millisecond pauses?
3. When would you choose Parallel GC over G1?

## 📚 Related Topics
- [GC Algorithms](./q05-gc-algorithms.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)
- [Monitoring JVM](./q13-monitoring-jvm.md)
