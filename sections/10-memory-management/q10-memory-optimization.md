# Q10: Memory Optimization Techniques in Java

[← Back to Memory Management](./README.md)

---

## Overview

Java objects carry overhead beyond their logical payload — object headers, padding, reference pointers, and collection internals all consume memory. Understanding these costs enables significant memory optimization, especially at scale.

---

## Object Header Overhead

Every Java object has a header before its fields:

```
64-bit JVM (with compressed oops — default for heaps < 32GB):
┌────────────────────┬───────────────────┬──────────┐
│  Mark Word (8 B)   │  Class Pointer    │  Fields  │
│  (hash, lock, GC)  │  (4 B compressed) │  ...     │
└────────────────────┴───────────────────┴──────────┘
  Total header: 12 bytes + padding to 8-byte boundary = 16 bytes minimum

Arrays add 4 bytes for length:
┌────────────────────┬───────────────────┬──────────┬──────────┐
│  Mark Word (8 B)   │  Class Pointer    │ Length   │ Elements │
│                    │  (4 B)            │ (4 B)    │  ...     │
└────────────────────┴───────────────────┴──────────┴──────────┘
  Total header: 16 bytes
```

### Impact example

```java
// An Integer object storing the value 42:
// Object header:  16 bytes
// int field:       4 bytes
// Padding:         4 bytes (align to 8)
// Total:          24 bytes   ← 6x the size of a raw int (4 bytes)!

// An empty Object:
// Object header:  16 bytes
// No fields:       0 bytes
// Total:          16 bytes   ← 16 bytes for... nothing
```

---

## 1. Primitives vs Wrapper Types

```java
// ❌ Anti-pattern: using wrappers unnecessarily
Integer count = 0;                 // 24 bytes (object header + int + padding)
Long timestamp = 0L;               // 24 bytes
Boolean active = true;             // 24 bytes

// ✅ Use primitives when possible
int count = 0;                     // 4 bytes
long timestamp = 0L;               // 8 bytes
boolean active = true;             // 1 byte (+ alignment)
```

### Memory comparison at scale

```java
// Storing 1 million integers:
Integer[] wrappers = new Integer[1_000_000];
// Memory: 1M × 24 bytes (Integer) + 1M × 4 bytes (references) + array overhead
// ≈ 28 MB

int[] primitives = new int[1_000_000];
// Memory: 1M × 4 bytes + 16 bytes (array header)
// ≈ 4 MB

// Savings: ~7x less memory with primitives!
```

### Collections with primitives

```java
// ❌ Standard collections require boxing
List<Integer> boxedList = new ArrayList<>();   // Stores Integer objects
Map<Integer, Double> boxedMap = new HashMap<>();

// ✅ Use primitive-specialized collections (Eclipse Collections, fastutil)
// Eclipse Collections:
IntArrayList intList = new IntArrayList();     // No boxing
IntIntHashMap intMap = new IntIntHashMap();     // Key and value are primitives

// fastutil:
Int2DoubleOpenHashMap fastMap = new Int2DoubleOpenHashMap();
```

---

## 2. Collection Sizing

### ArrayList default sizing

```java
// ❌ Anti-pattern: default capacity with known size
List<String> list = new ArrayList<>();   // Default capacity: 10
// Grows by 50% each time: 10 → 15 → 22 → 33 → ...
// Wastes memory if you have 1000 items (multiple resizes + wasted tail space)

// ✅ Pre-size when you know the approximate count
List<String> list = new ArrayList<>(expectedSize);
```

### HashMap load factor and capacity

```java
// HashMap default: capacity=16, loadFactor=0.75
// Rehashes at 16 * 0.75 = 12 entries

// ❌ Undersized HashMap for known data
Map<String, String> map = new HashMap<>();   // Will rehash multiple times for 100 entries

// ✅ Pre-size to avoid rehashing
// Formula: desiredSize / loadFactor + 1
int expectedEntries = 100;
Map<String, String> map = new HashMap<>((int) (expectedEntries / 0.75f) + 1);

// ✅ Since Java 19: factory method handles the math
Map<String, String> map = HashMap.newHashMap(100);   // Sized for 100 entries
```

### HashMap entry overhead

```java
// Each HashMap.Entry (Node) contains:
// - Object header:    16 bytes
// - int hash:          4 bytes
// - Object key ref:    4 bytes (compressed)
// - Object value ref:  4 bytes (compressed)
// - Node next ref:     4 bytes (compressed)
// - Padding:           0 bytes
// Total per entry:    32 bytes + key object + value object

// For 1 million String→String entries:
// Entries:     1M × 32 bytes        = 32 MB
// Table array: ~1.3M × 4 bytes      =  5 MB
// Keys:        1M × ~48 bytes       = 48 MB
// Values:      1M × ~48 bytes       = 48 MB
// Total:       ≈ 133 MB
```

---

## 3. Lazy Initialization

```java
// ❌ Eagerly creating expensive fields that might not be used
public class Report {
    private final List<Chart> charts = new ArrayList<>();        // Always allocated
    private final Map<String, Data> details = new HashMap<>();   // Always allocated
    // These may never be used if report is read without details
}

// ✅ Lazy initialization — create on first use
public class Report {
    private List<Chart> charts;
    private Map<String, Data> details;

    public List<Chart> getCharts() {
        if (charts == null) {
            charts = new ArrayList<>();
        }
        return charts;
    }

    public Map<String, Data> getDetails() {
        if (details == null) {
            details = new HashMap<>();
        }
        return details;
    }
}
```

```java
// ✅ Thread-safe lazy initialization
public class ConfigHolder {
    private volatile ExpensiveConfig config;

    public ExpensiveConfig getConfig() {
        ExpensiveConfig result = config;
        if (result == null) {
            synchronized (this) {
                result = config;
                if (result == null) {
                    config = result = new ExpensiveConfig();
                }
            }
        }
        return result;
    }
}
```

---

## 4. Flyweight Pattern

Share common state across many objects:

```java
// ❌ Each Character stores redundant shared data
public class Character {
    private final String name;         // Unique per character
    private final Texture texture;     // 10MB — but same for all 'A' glyphs!
    private final Font font;
    private int x, y;
}

// 1000 'A' characters = 1000 × 10MB textures = 10 GB!
```

```java
// ✅ Flyweight: separate intrinsic (shared) from extrinsic (unique) state
public class CharacterFlyweight {
    // Intrinsic state — shared
    private final Texture texture;
    private final Font font;

    CharacterFlyweight(Texture texture, Font font) {
        this.texture = texture;
        this.font = font;
    }

    public void render(int x, int y) {   // Extrinsic state passed in
        // Render texture at position
    }
}

public class CharacterFactory {
    private static final Map<Character, CharacterFlyweight> cache = new HashMap<>();

    public static CharacterFlyweight get(char c, Font font) {
        return cache.computeIfAbsent(c, k -> new CharacterFlyweight(loadTexture(c), font));
    }
}

// 1000 'A' characters = 1 shared flyweight + 1000 × (x,y) coordinates
```

---

## 5. Compact Strings (Java 9+)

```java
// Before Java 9: String stored as char[] (2 bytes per character, always)
// "hello" → char[5] → 10 bytes payload

// Java 9+: String stored as byte[] with LATIN1 or UTF16 encoding
// "hello" → byte[5] + LATIN1 flag → 5 bytes payload (50% savings!)
// "héllo" → byte[10] + UTF16 flag → 10 bytes payload (same as before)

// Compact Strings are ENABLED by default — no action needed
// Disable (not recommended):
// java -XX:-CompactStrings MyApp
```

---

## 6. Array vs Collection Memory

```java
// Comparison for storing 100 integers:

// int[] array: 16 (header) + 100 × 4 = 416 bytes
int[] arr = new int[100];

// ArrayList<Integer>: 
//   ArrayList object:     ~40 bytes
//   Object[] backing:     16 + 100 × 4 = 416 bytes (references)
//   Integer objects:      100 × 24 = 2,400 bytes
//   Total:                ≈ 2,856 bytes
List<Integer> list = new ArrayList<>(100);

// That's ~7x more memory for ArrayList<Integer> vs int[]!
```

```java
// ✅ Use arrays for large primitive data when List features aren't needed
// ✅ Use specialized collection libraries for primitive collections:
//    - Eclipse Collections: IntArrayList, LongHashSet, etc.
//    - fastutil: Int2IntOpenHashMap, IntArrayList, etc.
//    - Koloboke: HashIntIntMap, etc.
```

---

## 7. Off-Heap Memory

For large datasets, consider allocating memory outside the Java heap:

```java
import java.nio.ByteBuffer;

public class OffHeapExample {

    public static void main(String[] args) {
        // ✅ Direct ByteBuffer — off-heap, no GC pressure
        ByteBuffer directBuffer = ByteBuffer.allocateDirect(100 * 1024 * 1024);   // 100MB

        // Write data
        directBuffer.putInt(42);
        directBuffer.putDouble(3.14);

        // Read data
        directBuffer.flip();
        System.out.println(directBuffer.getInt());     // 42
        System.out.println(directBuffer.getDouble());  // 3.14

        // Memory is freed when ByteBuffer is GC'd (via Cleaner)
        // Or you can use Unsafe for manual management
    }
}
```

### Off-heap considerations

| Aspect              | Heap Memory              | Off-Heap (Direct)            |
|---------------------|--------------------------|------------------------------|
| **GC pressure**     | Yes                      | No                           |
| **Allocation speed**| Very fast                | Slower (OS syscall)          |
| **Access speed**    | Fast                     | Slightly slower (no bounds)  |
| **Max size**        | `-Xmx`                   | OS-limited + `-XX:MaxDirectMemorySize` |
| **Type safety**     | Full                     | Raw bytes only               |

```bash
# Configure max direct memory
java -XX:MaxDirectMemorySize=512m MyApp
```

---

## 8. Enum Instead of String Constants

```java
// ❌ Using String constants — each unique string is an object
public class Status {
    public static final String ACTIVE = "ACTIVE";
    public static final String INACTIVE = "INACTIVE";
    public static final String PENDING = "PENDING";
}
// Plus, each usage creates a reference

// ✅ Enums are singletons — one object per constant, ever
public enum Status {
    ACTIVE, INACTIVE, PENDING
}
// Enum values are shared across the entire application
```

---

## Memory Optimization Checklist

| Technique                          | Savings Potential | Complexity |
|-------------------------------------|-------------------|------------|
| Primitives over wrappers            | 3-6x per field    | Low        |
| Pre-sizing collections              | 10-50%            | Low        |
| Compact Strings (Java 9+)           | ~50% for ASCII    | None (auto)|
| Flyweight pattern                   | Massive for shared| Medium     |
| Lazy initialization                 | Varies            | Low        |
| Primitive collections (Eclipse/fastutil) | 3-8x         | Medium     |
| Off-heap memory                     | Eliminates GC     | High       |
| EnumSet/EnumMap                     | 2-10x vs HashSet  | Low        |
| `boolean[]` → `BitSet`             | 8x                | Low        |

---

## Best Practices

```java
// ✅ Profile before optimizing — measure, don't guess
// ✅ Use JOL (Java Object Layout) to understand actual object sizes
// ✅ Pre-size collections when the expected size is known
// ✅ Use primitives and primitive collections for large datasets
// ✅ Use BitSet instead of boolean[] or Set<Integer> for flags/membership
// ✅ Consider off-heap storage for very large, long-lived data

// ❌ Don't optimize prematurely — focus on hot paths
// ❌ Don't use int where byte suffices in large arrays (4x waste)
// ❌ Don't create unnecessary intermediate objects in hot loops
// ❌ Don't keep references to unused objects in long-lived data structures
```

---

## Interview Tips
- Every Java object has a 12-16 byte header — storing an `int` in an `Integer` costs 24 bytes vs 4 bytes
- `HashMap` has ~32 bytes overhead per entry (Node object) on top of key/value objects
- Compact Strings (Java 9+) are enabled by default and halve ASCII string memory
- Off-heap memory (`ByteBuffer.allocateDirect`) avoids GC but is harder to manage
- JOL (Java Object Layout) by Aleksey Shipilëv is the definitive tool for measuring object sizes

---

[← Q9: String Pool](./q09-string-pool.md) | [Next: Q11 - Profiling Memory →](./q11-profiling-memory.md)
