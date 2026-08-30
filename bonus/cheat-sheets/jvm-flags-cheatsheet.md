# JVM Flags & Tuning - Cheat Sheet

Essential JVM tuning flags and configurations for performance optimization.

## 🚀 Running Java with Flags

```bash
# Basic syntax
java -Xmx1024m -Xms512m -jar application.jar

# With multiple flags
java -Xmx2g -Xms1g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar

# Print all flags
java -XX:+PrintFlagsFinal -version

# Print command line flags
java -XX:+PrintCommandLineFlags -version
```

## 💾 Memory Management Flags

### Heap Size
```bash
# Set maximum heap size
-Xmx1024m          # 1 GB
-Xmx2g              # 2 GB
-Xmx4g              # 4 GB

# Set initial heap size
-Xms512m            # 512 MB
-Xms1g              # 1 GB

# Best practice: Set Xms = Xms for predictable performance
java -Xms2g -Xmx2g -jar app.jar

# Heap size rules:
# - Xms: Initial heap (smaller = faster startup)
# - Xmx: Maximum heap (should be 80% of available RAM)
# - Xms = Xmx: Prevents heap resizing (better for production)
```

### Young Generation
```bash
# Set young generation size
-Xmn512m            # 512 MB young generation

# Young gen as percentage of heap
-XX:NewRatio=3      # Young gen = 1/4 of heap (default)
-XX:NewRatio=2      # Young gen = 1/3 of heap

# Young gen percentage
-XX:NewSize=256m    # Initial young gen size
-XX:MaxNewSize=512m # Maximum young gen size

# Survivor ratio
-XX:SurvivorRatio=8 # Eden:Survivor = 8:1 (default)
-XX:SurvivorRatio=6 # Eden:Survivor = 6:1
```

### Metaspace (Java 8+)
```bash
# Metaspace size (replaces PermGen)
-XX:MetaspaceSize=128m      # Initial metaspace
-XX:MaxMetaspaceSize=256m   # Maximum metaspace

# Class metadata
-XX:CompressedClassSpaceSize=256m  # Compressed class space

# Metaspace growth
-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m
```

### PermGen (Java 7 and earlier)
```bash
# Permanent generation (deprecated in Java 8)
-XX:PermSize=128m           # Initial PermGen
-XX:MaxPermSize=256m        # Maximum PermGen
```

## 🗑️ Garbage Collection Flags

### GC Selection
```bash
# Serial GC (single-threaded, small heaps)
-XX:+UseSerialGC

# Parallel GC (multi-threaded, throughput-oriented)
-XX:+UseParallelGC
-XX:ParallelGCThreads=8     # Number of GC threads

# CMS GC (concurrent, low-latency)
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75  # Start CMS at 75% heap

# G1GC (generational, low-latency, Java 9+)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200    # Target pause time
-XX:G1HeapRegionSize=16m    # Region size

# ZGC (ultra-low latency, Java 11+)
-XX:+UseZGC
-XX:ZUncommitDelay=300      # Uncommit delay in seconds

# Shenandoah (ultra-low latency, Java 12+)
-XX:+UseShenandoahGC
```

### GC Logging
```bash
# GC logging (Java 8)
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-Xloggc:gc.log

# GC logging (Java 9+)
-Xlog:gc*:file=gc.log:time,level,tags

# Detailed GC logging
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintSafepointStatistics
-Xloggc:gc.log
```

### GC Tuning
```bash
# Disable explicit GC (System.gc())
-XX:+DisableExplicitGC

# GC pause target
-XX:MaxGCPauseMillis=200    # Target 200ms pause time

# GC thread count
-XX:ParallelGCThreads=8     # Parallel GC threads
-XX:ConcGCThreads=2         # Concurrent GC threads

# Full GC frequency
-XX:+ScavengeBeforeFullGC   # Scavenge before full GC
-XX:+UseAdaptiveSizePolicy  # Adaptive sizing
```

## 🔍 Performance Monitoring Flags

### Profiling
```bash
# CPU profiling
-XX:+UnlockDiagnosticVMOptions
-XX:+TraceClassLoading
-XX:+TraceClassUnloading

# Method profiling
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintCompilation
-XX:+PrintInlining

# Memory profiling
-XX:+PrintHeapAtGC
-XX:+PrintTenuringDistribution
```

### Monitoring
```bash
# Enable JMX
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

# Flight Recorder (Java 11+)
-XX:+UnlockCommercialFeatures
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=recording.jfr

# Verbose output
-verbose:gc        # GC verbose
-verbose:class     # Class loading verbose
```

## 🎯 Compilation Flags

### JIT Compilation
```bash
# Compilation threshold
-XX:CompileThreshold=10000  # Invocations before compilation

# Compilation modes
-XX:TieredStopAtLevel=4     # Tiered compilation level

# Inlining
-XX:MaxInlineSize=35        # Maximum inlinable method size
-XX:FreqInlineSize=325      # Frequent method inlining size
-XX:InlineSmallCode=1000    # Small code inlining size

# Code cache
-XX:ReservedCodeCacheSize=256m  # Code cache size
-XX:InitialCodeCacheSize=160m   # Initial code cache
```

### Optimization
```bash
# String deduplication
-XX:+UseStringDeduplication

# Escape analysis
-XX:+DoEscapeAnalysis

# Branch prediction
-XX:+UseBiasedLocking
-XX:BiasedLockingStartupDelay=0

# Loop unrolling
-XX:LoopUnrollLimit=50
```

## 🔐 Security Flags

```bash
# Disable biased locking
-XX:-UseBiasedLocking

# Enable security manager
-Djava.security.manager

# Trust store
-Djavax.net.ssl.trustStore=/path/to/truststore.jks
-Djavax.net.ssl.trustStorePassword=password

# Key store
-Djavax.net.ssl.keyStore=/path/to/keystore.jks
-Djavax.net.ssl.keyStorePassword=password
```

## 📊 Common Flag Combinations

### Development Environment
```bash
java -Xms512m -Xmx1024m \
     -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -Xloggc:gc.log \
     -jar app.jar
```

### Production - Low Latency
```bash
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+ParallelRefProcEnabled \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:G1SummarizeRSetStatsPeriod=1 \
     -jar app.jar
```

### Production - High Throughput
```bash
java -Xms8g -Xmx8g \
     -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -XX:+DisableExplicitGC \
     -XX:+UseAdaptiveSizePolicy \
     -jar app.jar
```

### Production - Balanced
```bash
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -Xloggc:/var/log/gc.log \
     -jar app.jar
```

## 📈 Monitoring Commands

### Check JVM Flags
```bash
# Print all flags
jps -l                      # List Java processes
jinfo -flags <pid>          # Print JVM flags for process
jinfo -sysprops <pid>       # Print system properties

# Check heap usage
jmap -heap <pid>            # Heap summary
jmap -histo <pid>           # Histogram of objects
jmap -dump:live,file=dump.hprof <pid>  # Dump heap

# Monitor GC
jstat -gc <pid> 1000        # GC stats every 1 second
jstat -gcutil <pid> 1000    # GC utilization
jstat -gccause <pid>        # Last GC cause

# Thread analysis
jstack <pid>                # Thread dump
jstack <pid> > threads.txt  # Save to file
```

## 🎯 Tuning Guidelines

### Heap Size
- **Small apps**: -Xmx512m to -Xmx1g
- **Medium apps**: -Xmx2g to -Xmx4g
- **Large apps**: -Xmx8g to -Xmx16g
- **Rule**: 80% of available RAM

### Young Generation
- **Default**: 1/3 of heap
- **Short-lived objects**: Increase to 1/2 of heap
- **Long-lived objects**: Decrease to 1/4 of heap

### GC Selection
- **Latency critical**: G1GC or ZGC
- **Throughput critical**: ParallelGC
- **Memory constrained**: SerialGC
- **Balanced**: G1GC (default in Java 9+)

### Metaspace
- **Default**: Unlimited (Java 8+)
- **Constrained**: -XX:MaxMetaspaceSize=256m
- **Large apps**: -XX:MaxMetaspaceSize=512m

## ⚠️ Common Pitfalls

1. **Heap too large**
   ```bash
   # ❌ Wrong - GC pauses will be long
   java -Xmx32g -jar app.jar
   
   # ✅ Correct - reasonable heap size
   java -Xmx8g -jar app.jar
   ```

2. **Xms != Xmx**
   ```bash
   # ❌ Wrong - heap resizing causes pauses
   java -Xms512m -Xmx4g -jar app.jar
   
   # ✅ Correct - fixed heap size
   java -Xms4g -Xmx4g -jar app.jar
   ```

3. **Wrong GC for workload**
   ```bash
   # ❌ Wrong - SerialGC for multi-core server
   java -XX:+UseSerialGC -jar app.jar
   
   # ✅ Correct - G1GC for server
   java -XX:+UseG1GC -jar app.jar
   ```

4. **No GC logging**
   ```bash
   # ❌ Wrong - can't diagnose GC issues
   java -jar app.jar
   
   # ✅ Correct - enable GC logging
   java -Xlog:gc*:file=gc.log:time,level,tags -jar app.jar
   ```

## 🎓 Interview Tips

- **Know memory regions**: Heap, stack, metaspace, code cache
- **Understand GC**: Young vs old generation, GC algorithms
- **Know common flags**: Xmx, Xms, UseG1GC, PrintGCDetails
- **Profiling tools**: jps, jstat, jmap, jstack, jcmd
- **Tuning process**: Measure, analyze, adjust, verify

---

**Related**: See Performance Optimization cheat sheet for profiling and analysis
