# Q11: PermGen vs Metaspace Evolution

## 📋 Question
What was PermGen and why was it replaced by Metaspace in Java 8? What changed with the String pool?

## Difficulty: 🟡 Medium

---

## 💡 Answer

**PermGen (Permanent Generation)** was a fixed-size heap region in Java 7 and earlier that stored class metadata, interned strings, and static variables. It was replaced by **Metaspace** in Java 8 because PermGen's fixed-size design caused frequent `OutOfMemoryError` issues and complicated JVM tuning.

### PermGen vs Metaspace — Visual Comparison

```
JAVA 7 AND EARLIER:
┌═══════════════════════════════════════════════════════┐
║                    JVM HEAP                           ║
║  ┌───────────────┐  ┌───────────┐  ┌──────────────┐  ║
║  │  Young Gen    │  │  Old Gen  │  │   PermGen    │  ║
║  │  (Eden + S0   │  │           │  │ (FIXED SIZE) │  ║
║  │   + S1)       │  │           │  │              │  ║
║  │               │  │           │  │ - Class meta │  ║
║  │               │  │           │  │ - String pool│  ║
║  │               │  │           │  │ - Static vars│  ║
║  │               │  │           │  │ - Byte code  │  ║
║  └───────────────┘  └───────────┘  └──────────────┘  ║
║  ←───────── -Xmx ──────────────→  ←─ -XX:MaxPerm │  ║
║                                      GenSize ────→   ║
╚═══════════════════════════════════════════════════════╝

JAVA 8 AND LATER:
┌═══════════════════════════════════════┐  ┌──────────────────┐
║              JVM HEAP                 ║  │  NATIVE MEMORY   │
║  ┌───────────────┐  ┌───────────┐    ║  │                  │
║  │  Young Gen    │  │  Old Gen  │    ║  │   METASPACE      │
║  │               │  │           │    ║  │  (DYNAMIC SIZE)  │
║  │               │  │           │    ║  │                  │
║  │               │  │ + String  │    ║  │  - Class meta    │
║  │               │  │   Pool    │    ║  │  - Byte code     │
║  │               │  │ + Static  │    ║  │  - Method data   │
║  │               │  │   vars    │    ║  │  - Annotations   │
║  └───────────────┘  └───────────┘    ║  │                  │
║  ←──────────── -Xmx ───────────→     ║  │ ← grows as      │
╚═══════════════════════════════════════╝  │    needed →      │
                                          └──────────────────┘
```

### Why PermGen Was Problematic

```
PermGen Issues:
┌─────────────────────────────────────────────────────────────┐
│ 1. FIXED SIZE                                               │
│    Default: 64 MB (32-bit) / 82 MB (64-bit)                │
│    Must be tuned with -XX:MaxPermGenSize                     │
│    Too small → OOM; Too large → wasted memory               │
│                                                              │
│ 2. FREQUENT OOM ERRORS                                       │
│    java.lang.OutOfMemoryError: PermGen space                 │
│    Common in: Tomcat redeployments, Spring apps,             │
│    Hibernate with many entities, heavy reflection            │
│                                                              │
│ 3. HARD TO TUNE                                              │
│    How big should it be? Depends on:                         │
│    - Number of classes                                       │
│    - Number of interned strings                              │
│    - Size of constant pools                                  │
│    Impossible to predict accurately                          │
│                                                              │
│ 4. GC COMPLEXITY                                             │
│    PermGen GC tied to Full GC                                │
│    Collecting PermGen required stop-the-world                │
│    Class unloading was error-prone                           │
│                                                              │
│ 5. INTERNAL FRAGMENTATION                                    │
│    Fixed size led to internal fragmentation                  │
│    Could OOM even with available space (fragmented)          │
└─────────────────────────────────────────────────────────────┘
```

### Metaspace Advantages

```
Metaspace Benefits:
┌─────────────────────────────────────────────────────────────┐
│ 1. NATIVE MEMORY ALLOCATION                                 │
│    Uses OS native memory (off-heap)                         │
│    Not limited by -Xmx (heap size)                          │
│    Grows and shrinks based on demand                        │
│                                                              │
│ 2. AUTOMATIC SIZING                                          │
│    No need to guess the right size                          │
│    Grows as needed (up to MaxMetaspaceSize)                  │
│    Shrinks after class unloading                            │
│                                                              │
│ 3. BETTER GC INTEGRATION                                    │
│    Each class loader has its own Metaspace chunk             │
│    When class loader is GC'd → its chunk is freed           │
│    More efficient memory reclamation                        │
│                                                              │
│ 4. IMPROVED PERFORMANCE                                      │
│    Less GC pressure (not part of heap)                      │
│    Better memory locality for class metadata                │
│    Faster class loading and unloading                       │
│                                                              │
│ 5. SIMPLIFIED TUNING                                         │
│    Fewer parameters to tune                                 │
│    -XX:MetaspaceSize and -XX:MaxMetaspaceSize               │
│    Reasonable defaults work for most applications            │
└─────────────────────────────────────────────────────────────┘
```

### Migration Comparison

| Aspect | PermGen (Java ≤ 7) | Metaspace (Java ≥ 8) |
|--------|--------------------|-----------------------|
| **Location** | JVM Heap (contiguous with Old Gen) | Native memory (off-heap) |
| **Default Size** | 64-82 MB (fixed) | Unlimited (grows as needed) |
| **Max Size Flag** | `-XX:MaxPermGenSize` | `-XX:MaxMetaspaceSize` |
| **Initial Size Flag** | `-XX:PermSize` | `-XX:MetaspaceSize` (HWM) |
| **String Pool** | Inside PermGen | Moved to **Heap** (Java 7+) |
| **Static Variables** | Inside PermGen | Moved to **Heap** (Java 8+) |
| **Class Metadata** | Inside PermGen | Inside Metaspace |
| **GC Trigger** | Full GC only | Metaspace HWM or Full GC |
| **Growth** | Cannot grow past max | Grows until OS limit |
| **OOM Error** | `PermGen space` | `Metaspace` |

### String Pool Evolution

```java
// ═══════════════════════════════════════════════════════
// JAVA 6: String pool in PermGen
// ═══════════════════════════════════════════════════════
// String.intern() stored strings in PermGen
// PermGen was small → OOM if too many interned strings
// Common issue: logging frameworks interning exception messages

// ═══════════════════════════════════════════════════════
// JAVA 7: String pool moved to HEAP
// ═══════════════════════════════════════════════════════
// String.intern() now stores strings in the main heap
// Benefits:
//   ✅ String pool can use the full heap
//   ✅ Interned strings can be garbage collected normally
//   ✅ No more PermGen OOM from string interning
//   ✅ -XX:StringTableSize controls hash table size

// ═══════════════════════════════════════════════════════
// JAVA 8+: Everything stabilized
// ═══════════════════════════════════════════════════════
// String pool: Heap (unchanged from Java 7)
// Static variables: Heap (moved from PermGen)
// Class metadata: Metaspace (native memory)

public class StringPoolDemo {
    public static void main(String[] args) {
        // String literal — goes to string pool (in heap)
        String s1 = "hello";

        // new String — creates new object on heap (NOT in pool)
        String s2 = new String("hello");

        // intern() — returns pool reference
        String s3 = s2.intern();

        System.out.println(s1 == s2);  // false (different objects)
        System.out.println(s1 == s3);  // true  (same pool reference)

        // In Java 7+, the pool is on the heap, so:
        //   - Interned strings are GC'd when unreachable
        //   - Pool size limited by heap, not PermGen
    }
}

// String table tuning:
// -XX:StringTableSize=60013    (default in Java 11+, prime number)
// -XX:+PrintStringTableStatistics  (print stats at JVM exit)
```

### Interned Strings — Practical Impact

```java
// ❌ Java 6: This could crash with PermGen OOM
List<String> interned = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    interned.add(String.valueOf(i).intern());
}
// In Java 6: PermGen (64 MB) fills up → OutOfMemoryError: PermGen space
// In Java 7+: Strings go to heap (potentially GBs) → works fine

// ✅ Java 7+ made String.intern() much safer
// But still use it carefully — the string table lookup has overhead
```

### Flag Migration Guide

```bash
# Java 7 (PermGen):
java -XX:PermSize=128m -XX:MaxPermGenSize=256m -jar app.jar

# Java 8+ (Metaspace):
java -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -jar app.jar

# If you see these flags on Java 8+, they are IGNORED with a warning:
# "Java HotSpot(TM) 64-Bit Server VM warning:
#  ignoring option MaxPermSize=256m; support was removed in 8.0"

# Complete migration mapping:
# -XX:PermSize          →  -XX:MetaspaceSize
# -XX:MaxPermGenSize    →  -XX:MaxMetaspaceSize
# (no equivalent)       →  -XX:CompressedClassSpaceSize
# (no equivalent)       →  -XX:MinMetaspaceFreeRatio
# (no equivalent)       →  -XX:MaxMetaspaceFreeRatio
```

### What Moved Where in Java 8

```
                    Java 7 (PermGen)          Java 8 (Metaspace)
                    ─────────────────         ──────────────────
Class metadata      PermGen                   Metaspace (native)
Method bytecode     PermGen                   Metaspace (native)
Constant pool       PermGen                   Metaspace (native)
Annotations         PermGen                   Metaspace (native)

String pool         PermGen → Heap (Java 7)   Heap
Static variables    PermGen                   Heap (java.lang.Class)
Class statics       PermGen                   Heap (mirror of Class)

Class objects       Heap                      Heap (unchanged)
Object instances    Heap                      Heap (unchanged)
```

### Detecting PermGen/Metaspace Issues

```java
// Simulating class loader leak (common in app servers)
public class MetaspaceLeakDemo {
    public static void main(String[] args) throws Exception {
        List<ClassLoader> leakedLoaders = new ArrayList<>();

        while (true) {
            // Create a new class loader
            URLClassLoader loader = new URLClassLoader(
                new URL[]{ new File("./classes").toURI().toURL() }
            );

            // Load a class
            loader.loadClass("com.example.Plugin");

            // ❌ LEAK: Keep reference to loader — prevents GC and class unloading
            leakedLoaders.add(loader);

            // Each iteration loads the class metadata into Metaspace
            // Since loaders are never GC'd, metadata is never freed
            // Eventually: OutOfMemoryError: Metaspace
        }
    }
}

// In Java 7: OutOfMemoryError: PermGen space
// In Java 8+: OutOfMemoryError: Metaspace
// Same root cause, different memory region
```

### ⚠️ Common Mistake
Thinking that Metaspace eliminates all PermGen problems. While Metaspace removes the fixed-size limitation, class loader leaks still cause `OutOfMemoryError: Metaspace`. The leak just takes longer to manifest because Metaspace can grow. Always set `-XX:MaxMetaspaceSize` to detect leaks early rather than letting them consume all native memory.

### ⚡ Performance Tip
When migrating from Java 7 to Java 8+, remove all PermGen flags (`-XX:PermSize`, `-XX:MaxPermGenSize`) and replace with Metaspace equivalents. Also increase the heap size slightly because the string pool and static variables that were in PermGen are now counted against the heap (`-Xmx`). Monitor Metaspace usage during the first few weeks after migration to establish a baseline.

```bash
# Post-migration monitoring:
jstat -gcmetacapacity <pid> 5000    # Every 5 seconds
# Watch MCMN (min), MCMX (max), MC (committed), MU (used)
# If MU keeps growing without plateauing → class loader leak
```

---

## 🎯 Interviewer Insight
Interviewers ask this to test your understanding of JVM evolution. Key points: PermGen was fixed-size on the heap and caused frequent OOMs; Metaspace uses native memory and grows dynamically. Mentioning the string pool move (PermGen → Heap in Java 7) and static variable move (PermGen → Heap in Java 8) shows deep knowledge. Senior candidates should know the flag migration path.

## ⚡ Follow-up Questions
1. Why was the string pool moved to the heap before PermGen was removed?
2. Can you still get OutOfMemoryError with Metaspace?
3. How do you migrate JVM flags from Java 7 to Java 8?

## 📚 Related Topics
- [Metaspace](./q10-metaspace.md)
- [Stack vs Heap](./q09-stack-vs-heap.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)
