# Q10: What Is Metaspace?

## 📋 Question
What is Metaspace in the JVM? What does it store, how is it managed, and how do you configure and monitor it?

## Difficulty: 🟡 Medium

---

## 💡 Answer

**Metaspace** is the JVM memory area (introduced in Java 8) that stores **class metadata** — the runtime representation of loaded classes including class structures, method bytecode, constant pools, and annotations. Unlike its predecessor PermGen, Metaspace uses **native memory** (off-heap) and can grow dynamically.

### What Metaspace Stores

```
┌═══════════════════════════════════════════════════════════┐
║                     METASPACE                             ║
║               (Native Memory — off-heap)                  ║
║                                                           ║
║  ┌─────────────────────────────────────────────────────┐  ║
║  │  CLASS METADATA                                     │  ║
║  │  ├── Class structure (fields, methods, interfaces)  │  ║
║  │  ├── Method bytecode                                │  ║
║  │  ├── Constant pool (resolved references)            │  ║
║  │  ├── Annotations                                    │  ║
║  │  ├── Method counters (for JIT profiling)            │  ║
║  │  └── Virtual method tables (vtables)                │  ║
║  └─────────────────────────────────────────────────────┘  ║
║                                                           ║
║  ┌─────────────────────────────────────────────────────┐  ║
║  │  COMPRESSED CLASS SPACE (if compressed oops enabled)│  ║
║  │  ├── Klass structures (internal class representation)│  ║
║  │  └── Default max: 1 GB                              │  ║
║  │      -XX:CompressedClassSpaceSize=256m               │  ║
║  └─────────────────────────────────────────────────────┘  ║
║                                                           ║
║  NOT in Metaspace:                                        ║
║  ✗ String pool (moved to Heap in Java 7)                  ║
║  ✗ Static variables (in Heap since Java 8)                ║
║  ✗ Object instances (always in Heap)                      ║
╚═══════════════════════════════════════════════════════════╝
```

### Metaspace Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    NATIVE MEMORY                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                   METASPACE                            │  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │
│  │  │ Chunk 1  │  │ Chunk 2  │  │ Chunk 3  │  ...       │  │
│  │  │ (CL: App)│  │(CL: Web1)│  │(CL: Web2)│            │  │
│  │  │          │  │          │  │          │            │  │
│  │  │ Classes: │  │ Classes: │  │ Classes: │            │  │
│  │  │ MyApp    │  │ Servlet  │  │ Filter   │            │  │
│  │  │ Config   │  │ JSP      │  │ Handler  │            │  │
│  │  └──────────┘  └──────────┘  └──────────┘            │  │
│  │                                                        │  │
│  │  Each ClassLoader gets its own chunk of Metaspace      │  │
│  │  When a ClassLoader is GC'd, its chunk is freed        │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Grows automatically until:                                  │
│  • -XX:MaxMetaspaceSize is reached (default: unlimited)      │
│  • OS runs out of native memory                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Metaspace Parameters

```bash
# Initial high-water mark — triggers first GC when reached
-XX:MetaspaceSize=256m        # Default: ~21 MB (platform dependent)
                               # When usage hits this, GC is triggered
                               # After GC, the high-water mark adjusts

# Maximum Metaspace size — hard limit
-XX:MaxMetaspaceSize=512m     # Default: unlimited (constrained only by OS)
                               # ⚠️ Always set this in production!
                               # Without it, a class loader leak can consume
                               # all available native memory

# Compressed class space size
-XX:CompressedClassSpaceSize=256m  # Default: 1 GB, max: 3 GB
                                    # Only used with -XX:+UseCompressedOops (default)

# Minimum free ratio after GC (controls shrinking)
-XX:MinMetaspaceFreeRatio=40   # Min % free after GC before expanding

# Maximum free ratio after GC (controls expanding)
-XX:MaxMetaspaceFreeRatio=70   # Max % free after GC before shrinking
```

### How Metaspace Grows

```
MetaspaceSize                    MaxMetaspaceSize
(initial HWM)                    (hard limit)
     ↓                                ↓
     ├────────────┬────────────────────┤
     │            │                    │
     │  Usage     │   Available        │
     │████████░░░░░░░░░░░░░░░░░░░░░░░░│
     │            ↑                    │
     │     Current HWM                 │
     │  (adjusts after each GC)        │
     │                                 │

Growth behavior:
1. Classes loaded → Metaspace grows
2. Usage reaches current HWM → GC triggered (unload dead classes)
3. If freed enough → HWM may decrease
4. If NOT freed enough → HWM increases
5. If HWM reaches MaxMetaspaceSize → OutOfMemoryError: Metaspace
```

### Monitoring Metaspace

```bash
# Command-line monitoring
jcmd <pid> VM.metaspace          # Detailed metaspace stats (Java 11+)
jcmd <pid> GC.class_stats        # Per-class metadata sizes
jstat -gcmetacapacity <pid> 1000 # Metaspace capacity stats every 1 second
jstat -gcutil <pid> 1000         # Utilization stats including Metaspace (M column)

# Sample jstat output:
#   S0     S1     E      O      M     CCS    YGC    FGC
#  0.00  93.76  65.43  12.01  97.52  94.67   45     2
#                              ↑
#                      Metaspace 97.52% full — ⚠️ close to OOM!
```

```java
// Programmatic monitoring
import java.lang.management.*;

public class MetaspaceMonitor {
    public static void main(String[] args) {
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            if (pool.getName().contains("Metaspace")) {
                MemoryUsage usage = pool.getUsage();
                System.out.println("Pool: " + pool.getName());
                System.out.println("  Init:      " + usage.getInit() / 1024 / 1024 + " MB");
                System.out.println("  Used:      " + usage.getUsed() / 1024 / 1024 + " MB");
                System.out.println("  Committed: " + usage.getCommitted() / 1024 / 1024 + " MB");
                System.out.println("  Max:       " + usage.getMax() / 1024 / 1024 + " MB");
            }
        }
    }
}
// Output:
// Pool: Metaspace
//   Init:      0 MB
//   Used:      42 MB
//   Committed: 45 MB
//   Max:       -1 MB  (-1 = unlimited)
```

### Class Unloading

Classes are unloaded from Metaspace when their **class loader** is garbage collected. This requires:

1. All instances of all classes loaded by that class loader are unreachable
2. The class loader itself is unreachable
3. The `java.lang.Class` objects are unreachable

```java
// Classes loaded by the Bootstrap class loader are NEVER unloaded
// Classes loaded by custom class loaders CAN be unloaded

public class ClassUnloadingDemo {
    public static void main(String[] args) throws Exception {
        // Create a custom class loader
        URLClassLoader loader = new URLClassLoader(
            new URL[]{ new File("/plugins").toURI().toURL() }
        );

        // Load a class
        Class<?> pluginClass = loader.loadClass("com.plugin.MyPlugin");
        Object plugin = pluginClass.getDeclaredConstructor().newInstance();

        // Use the plugin...

        // To enable class unloading:
        plugin = null;         // 1. Remove all instances
        pluginClass = null;    // 2. Remove Class reference
        loader.close();
        loader = null;         // 3. Remove class loader reference

        System.gc();           // 4. Request GC — class may be unloaded

        // The Metaspace chunk for this loader is freed
    }
}

// Ensure class unloading is enabled (it is by default):
// -XX:+ClassUnloading           (enabled by default)
// -XX:+CMSClassUnloadingEnabled (for CMS, if used)
```

### Metaspace Fragmentation

```
Problem: Class loaders allocate chunks of different sizes.
When small class loaders are GC'd, they leave small free gaps.
New class loaders may need bigger contiguous chunks.

Fragmented Metaspace:
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ Used │ Free │ Used │ Free │ Free │ Used │ Free │
│  1K  │  2K  │  4K  │  1K  │  1K  │  8K  │  3K  │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┘
New request for 5K → No single free block is big enough!
→ Must request more OS memory even though 7K is free total

Mitigation:
- Avoid frequent class loader creation/destruction
- Use -XX:MaxMetaspaceSize to detect leaks early
- JDK improvements reduce fragmentation over versions
```

### Common Metaspace Issues

```java
// 1. OutOfMemoryError: Metaspace
// Cause: Too many classes loaded (often a class loader leak)
// Common in: Tomcat redeployments, heavy use of reflection/proxies

// 2. Class loader leaks
// Cause: A reference chain prevents the class loader from being GC'd
// Common patterns:
//   - ThreadLocal holding a class from the web app class loader
//   - Logging framework holding class references
//   - JDBC driver not deregistered
//   - Shutdown hooks referencing web app classes

// 3. Dynamic proxy/bytecode generation
// Frameworks like Spring, Hibernate, and CGLIB generate classes at runtime
// Each generated class consumes Metaspace
// Spring Boot apps typically use 50-100 MB of Metaspace
```

### ⚠️ Common Mistake
Not setting `-XX:MaxMetaspaceSize` in production. Without a limit, Metaspace can grow until the OS runs out of native memory, potentially crashing the entire server (not just the JVM). Always set a reasonable limit and monitor usage. A growing Metaspace that never shrinks usually indicates a class loader leak.

### ⚡ Performance Tip
For Spring Boot applications, set `MetaspaceSize` equal to `MaxMetaspaceSize` to avoid the overhead of repeated Metaspace GC cycles during startup (when many classes are loaded rapidly):

```bash
# Recommended for Spring Boot:
java -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m -jar myapp.jar

# This avoids:
# 1. Multiple GC triggers as Metaspace grows from 21 MB to 250 MB
# 2. HWM adjustment overhead
# 3. Startup latency from unnecessary GC pauses
```

---

## 🎯 Interviewer Insight
Interviewers want to hear that Metaspace stores class metadata in native memory, replaces PermGen from Java 8, and grows dynamically. Strong candidates explain the relationship between class loaders and Metaspace chunks, class unloading requirements, and the importance of setting `MaxMetaspaceSize`. Mentioning class loader leaks in application servers shows practical experience.

## ⚡ Follow-up Questions
1. What happens when Metaspace runs out of memory?
2. How do you detect a class loader leak?
3. Why does Spring Boot use so much Metaspace?

## 📚 Related Topics
- [PermGen vs Metaspace](./q11-permgen-vs-metaspace.md)
- [Class Loaders](./q03-class-loaders.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)
- [Monitoring JVM](./q13-monitoring-jvm.md)
