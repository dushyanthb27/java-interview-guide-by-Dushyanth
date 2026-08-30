# Q12: Key JVM Tuning Parameters

## 📋 Question
What are the most important JVM tuning parameters? How do you configure heap size, GC behavior, and other critical settings?

## Difficulty: 🟠 Hard

---

## 💡 Answer

JVM tuning parameters (flags) control memory allocation, garbage collection behavior, compilation strategy, and diagnostics. Understanding these flags is essential for optimizing Java application performance in production.

### Memory Configuration — The Essential Flags

```
┌══════════════════════════════════════════════════════════════════┐
║                   JVM MEMORY LAYOUT + FLAGS                     ║
║                                                                  ║
║  ┌──────────────────────────────────────────┐   Native Memory   ║
║  │              JVM HEAP                    │                    ║
║  │         (-Xms / -Xmx)                   │   ┌────────────┐  ║
║  │                                          │   │ Metaspace  │  ║
║  │  ┌──────────────────┐ ┌──────────────┐   │   │ -XX:Meta-  │  ║
║  │  │   Young Gen      │ │   Old Gen    │   │   │ spaceSize  │  ║
║  │  │ -XX:NewSize      │ │              │   │   │ -XX:MaxMeta│  ║
║  │  │ -XX:MaxNewSize   │ │              │   │   │ spaceSize  │  ║
║  │  │ -XX:NewRatio     │ │              │   │   └────────────┘  ║
║  │  │                  │ │              │   │   ┌────────────┐  ║
║  │  │ ┌────┐┌──┐┌──┐  │ │              │   │   │Thread Stack│  ║
║  │  │ │Eden││S0││S1│  │ │              │   │   │  -Xss      │  ║
║  │  │ └────┘└──┘└──┘  │ │              │   │   │  (per thrd)│  ║
║  │  │ -XX:SurvivorRatio│ │              │   │   └────────────┘  ║
║  │  └──────────────────┘ └──────────────┘   │   ┌────────────┐  ║
║  └──────────────────────────────────────────┘   │ Code Cache │  ║
║                                                  │ -XX:Reserve│  ║
║                                                  │ dCodeCache │  ║
║                                                  │ Size       │  ║
║                                                  └────────────┘  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Heap Sizing

```bash
# ═══════════════════════════════════════════════
# HEAP SIZE (most important flags)
# ═══════════════════════════════════════════════

-Xms512m        # Initial heap size (minimum)
-Xmx2g          # Maximum heap size
                # ⚡ Best practice: Set -Xms = -Xmx to avoid resize overhead

-Xmn256m        # Young Generation size (shortcut for NewSize + MaxNewSize)
                # Alternative to -XX:NewRatio

# Size formats: k/K (KB), m/M (MB), g/G (GB)
# java -Xms1g -Xmx4g -jar myapp.jar
```

### Generation Sizing

```bash
# ═══════════════════════════════════════════════
# YOUNG / OLD GENERATION RATIO
# ═══════════════════════════════════════════════

-XX:NewRatio=2          # Old:Young = 2:1 → Young = 1/3 of heap
                        # Default: 2 (Server VM)
                        # Lower value = larger Young Gen
                        # Example: -Xmx3g -XX:NewRatio=2 → Young=1g, Old=2g

-XX:NewSize=256m        # Initial Young Gen size
-XX:MaxNewSize=512m     # Maximum Young Gen size

-XX:SurvivorRatio=8     # Eden:Survivor = 8:1
                        # Eden = 8/10 of Young Gen
                        # Each Survivor = 1/10 of Young Gen
                        # Default: 8

-XX:MaxTenuringThreshold=15   # Max GC cycles before promotion to Old Gen
                               # Range: 0-15, Default: 15 (G1: 15, Parallel: 15)
                               # 0 = never use survivors (direct to Old Gen)
```

### Stack Size

```bash
# ═══════════════════════════════════════════════
# THREAD STACK SIZE
# ═══════════════════════════════════════════════

-Xss512k       # Stack size per thread
               # Default: ~512KB-1MB (platform dependent)
               # Larger: supports deeper recursion
               # Smaller: allows more threads (less native memory per thread)

# ⚠️ Trade-off:
# 1000 threads × 1MB = 1 GB native memory for stacks alone
# 1000 threads × 256KB = 250 MB — saves 750 MB
# But too small → StackOverflowError
```

### Metaspace Configuration

```bash
# ═══════════════════════════════════════════════
# METASPACE (Class metadata — native memory)
# ═══════════════════════════════════════════════

-XX:MetaspaceSize=128m       # Initial high-water mark
                              # GC triggered when usage reaches this
                              # Default: ~21 MB

-XX:MaxMetaspaceSize=256m    # Maximum Metaspace size
                              # Default: unlimited (⚠️ dangerous!)
                              # Always set in production

-XX:CompressedClassSpaceSize=256m  # Max compressed class space
                                    # Default: 1 GB
```

### Garbage Collector Selection

```bash
# ═══════════════════════════════════════════════
# GC SELECTION
# ═══════════════════════════════════════════════

-XX:+UseSerialGC          # Serial — single-threaded, small heaps
-XX:+UseParallelGC        # Parallel — throughput-focused (Java 8 default)
-XX:+UseG1GC              # G1 — balanced (Java 9+ default)
-XX:+UseZGC               # ZGC — ultra-low latency (Java 15+ production)
-XX:+UseShenandoahGC      # Shenandoah — low latency (OpenJDK only)

# Only ONE can be active. Default for Java 9+ is G1.
```

### G1 GC Tuning

```bash
# ═══════════════════════════════════════════════
# G1 SPECIFIC TUNING
# ═══════════════════════════════════════════════

-XX:MaxGCPauseMillis=200         # Target max pause time (default: 200ms)
                                  # G1 adjusts collection set to meet goal
                                  # Lower = more frequent, smaller collections

-XX:G1HeapRegionSize=16m         # Region size (1-32 MB, power of 2)
                                  # Default: auto-calculated based on heap size
                                  # Larger regions: fewer regions, less overhead
                                  # Smaller regions: better granularity

-XX:InitiatingHeapOccupancyPercent=45  # Start concurrent marking at this heap %
                                        # Default: 45
                                        # Lower = start GC earlier (more CPU, less OOM risk)

-XX:G1MixedGCCountTarget=8      # Number of mixed GC cycles after marking
-XX:G1ReservePercent=10          # Reserve % of heap as buffer
```

### ZGC Tuning

```bash
# ═══════════════════════════════════════════════
# ZGC SPECIFIC TUNING (minimal tuning needed!)
# ═══════════════════════════════════════════════

-XX:+UseZGC                     # Enable ZGC
-XX:+ZGenerational              # Use generational ZGC (Java 21+, default in 23)
-Xms4g -Xmx4g                   # Heap size (ZGC works best with same min/max)

# ZGC is designed to require very little tuning
# Main knob: heap size — give it enough memory and it handles the rest
```

### GC Logging

```bash
# ═══════════════════════════════════════════════
# GC LOGGING (Java 9+ Unified Logging)
# ═══════════════════════════════════════════════

# Basic GC logging
-Xlog:gc:file=gc.log:time,uptime,level,tags

# Detailed GC logging (recommended for production)
-Xlog:gc*=info:file=gc.log:time,uptime,level,tags:filecount=5,filesize=10m

# Verbose GC logging (for debugging)
-Xlog:gc*=debug:file=gc-debug.log:time,uptime,level,tags

# Java 8 (legacy) GC logging
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=10m
```

### Diagnostics and Debugging

```bash
# ═══════════════════════════════════════════════
# HEAP DUMP ON OOM (critical for production!)
# ═══════════════════════════════════════════════

-XX:+HeapDumpOnOutOfMemoryError          # Auto heap dump on OOM
-XX:HeapDumpPath=/var/log/java/          # Dump file location
-XX:OnOutOfMemoryError="kill -9 %p"      # Script to run on OOM

# ═══════════════════════════════════════════════
# JIT COMPILER DIAGNOSTICS
# ═══════════════════════════════════════════════

-XX:+PrintCompilation                     # Print JIT compilation events
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintInlining                        # Print inlining decisions
-XX:ReservedCodeCacheSize=256m            # Code cache for JIT output

# ═══════════════════════════════════════════════
# THREAD DIAGNOSTICS
# ═══════════════════════════════════════════════

-XX:+UnlockDiagnosticVMOptions
-XX:NativeMemoryTracking=summary          # Track native memory usage
                                           # Use: jcmd <pid> VM.native_memory
```

### Container-Aware JVM Settings

```bash
# ═══════════════════════════════════════════════
# DOCKER / KUBERNETES CONTAINER SETTINGS
# ═══════════════════════════════════════════════

# Java 10+ automatically detects container limits
# -XX:+UseContainerSupport  (enabled by default since Java 10)

# Control heap as percentage of container memory
-XX:MaxRAMPercentage=75.0        # Use 75% of container RAM for heap
                                  # Default: 25% (too conservative!)
-XX:InitialRAMPercentage=50.0    # Initial heap = 50% of container RAM
-XX:MinRAMPercentage=50.0        # Min heap when container RAM is small

# Example: Container with 4 GB limit
# -XX:MaxRAMPercentage=75.0 → -Xmx ≈ 3 GB
# Leaves ~1 GB for Metaspace, threads, native libs, OS

# ⚠️ Common mistake: Using -Xmx in containers
# If container has 4 GB and you set -Xmx4g:
#   Heap (4 GB) + Metaspace + Threads + Native = > 4 GB → OOM killed!
# Use -XX:MaxRAMPercentage instead for safety
```

### Production-Ready Configuration Template

```bash
# ═══════════════════════════════════════════════
# RECOMMENDED PRODUCTION CONFIGURATION
# ═══════════════════════════════════════════════

java \
  # Memory
  -Xms4g -Xmx4g \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m \
  \
  # GC (G1 — default, but be explicit)
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  \
  # GC Logging
  -Xlog:gc*=info:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m \
  \
  # Diagnostics
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/ \
  \
  # Container support (if containerized)
  -XX:+UseContainerSupport \
  \
  -jar myapp.jar

# ═══════════════════════════════════════════════
# CONTAINER / KUBERNETES CONFIGURATION
# ═══════════════════════════════════════════════

java \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=75.0 \
  -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -Xlog:gc*=info:file=/tmp/gc.log:time:filecount=3,filesize=10m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/ \
  -jar myapp.jar
```

### Viewing Current JVM Settings

```bash
# Print all JVM flags and their values
java -XX:+PrintFlagsFinal -version

# Print only non-default flags
java -XX:+PrintFlagsFinal -version 2>&1 | grep -E "\:=|management"

# For a running JVM
jcmd <pid> VM.flags                  # Active flags
jcmd <pid> VM.system_properties      # System properties
jcmd <pid> VM.command_line           # Original command line
jinfo -flags <pid>                   # Flag values
```

### ⚠️ Common Mistake
Setting `-Xms` much smaller than `-Xmx`. This causes the JVM to repeatedly resize the heap (allocate memory from OS, move objects), which adds latency. For production servers, always set `-Xms` equal to `-Xmx` so the heap is fully allocated at startup. Similarly, not setting `-XX:MaxMetaspaceSize` is dangerous — it allows class loader leaks to consume unbounded native memory.

### ⚡ Performance Tip
Start with a minimal configuration and add flags based on observed behavior:
1. Set heap size (`-Xms` = `-Xmx`)
2. Enable GC logging
3. Enable heap dumps on OOM
4. **Then benchmark and tune** based on GC logs

Avoid copying JVM flag configurations from the internet — every application has different workload characteristics. Use GC log analysis tools (GCEasy, GCViewer) to make data-driven tuning decisions.

---

## 🎯 Interviewer Insight
Interviewers want to see that you know the essential flags (heap sizing, GC selection, GC logging, OOM diagnostics) and understand **why** each flag matters. Strong candidates explain the relationship between flags (e.g., `NewRatio` affects Young Gen size which affects Minor GC frequency) and can describe a production-ready configuration. Container-aware settings are increasingly important.

## ⚡ Follow-up Questions
1. Why should `-Xms` equal `-Xmx` in production?
2. What happens if you don't set `-XX:MaxMetaspaceSize`?
3. How do you configure JVM flags in a Kubernetes pod?

## 📚 Related Topics
- [GC Collectors](./q06-gc-collectors.md)
- [Monitoring JVM](./q13-monitoring-jvm.md)
- [Metaspace](./q10-metaspace.md)
- [Stack vs Heap](./q09-stack-vs-heap.md)
