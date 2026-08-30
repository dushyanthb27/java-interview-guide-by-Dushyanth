# Q9: How Does the String Pool Work?

[← Back to Memory Management](./README.md)

---

## Overview

The **String Pool** (also called the String Intern Pool) is a special memory region in the Java heap where the JVM stores unique string literals. When multiple variables refer to the same string literal, they all point to the **same object** in the pool, saving memory.

---

## String Pool Location

```
Java 6 and earlier:
  ┌──────────────────────────────────┐
  │            HEAP                  │
  │  ┌────────────────────────────┐  │
  │  │     PermGen Space          │  │
  │  │  ┌──────────────────────┐  │  │
  │  │  │   String Pool ⚠️     │  │  │   ← Fixed size, OOM risk
  │  │  └──────────────────────┘  │  │
  │  └────────────────────────────┘  │
  └──────────────────────────────────┘

Java 7+:
  ┌──────────────────────────────────┐
  │            HEAP                  │
  │  ┌──────────────────────────┐   │
  │  │   String Pool ✅          │   │   ← In main heap, GC-eligible
  │  └──────────────────────────┘   │
  │                                  │
  │  ┌──────────────────────────┐   │
  │  │   Metaspace (off-heap)   │   │   ← Replaced PermGen
  │  └──────────────────────────┘   │
  └──────────────────────────────────┘
```

> Since Java 7, the String Pool is in the **main heap**, making pooled strings eligible for garbage collection and the pool size bounded only by heap space.

---

## String Literal Pooling

String literals in source code are automatically pooled at compile time:

```java
public class StringLiteralPooling {

    public static void main(String[] args) {
        String a = "hello";
        String b = "hello";

        System.out.println(a == b);            // true — same object from pool
        System.out.println(a.equals(b));       // true — same content

        String c = new String("hello");
        System.out.println(a == c);            // false — 'c' is a new heap object
        System.out.println(a.equals(c));       // true — same content
    }
}
```

### Memory layout

```
                Pool                     Heap
           ┌──────────┐
    a ────▶│ "hello"  │◀──── b          Both point to the SAME object
           └──────────┘

    c ────▶┌──────────┐                  c is a DIFFERENT object
           │ "hello"  │                  (same content, different identity)
           └──────────┘
```

---

## The intern() Method

`String.intern()` adds a string to the pool (if not already present) and returns the pooled reference:

```java
public class InternDemo {

    public static void main(String[] args) {
        String s1 = new String("hello");   // New object on heap
        String s2 = s1.intern();           // Returns pooled "hello"
        String s3 = "hello";              // Pooled "hello"

        System.out.println(s1 == s2);      // false — s1 is heap, s2 is pool
        System.out.println(s2 == s3);      // true  — both from pool
        System.out.println(s1 == s3);      // false — s1 is heap, s3 is pool

        // After interning a dynamically created string
        String dynamic = new String(new char[]{'w', 'o', 'r', 'l', 'd'});
        String interned = dynamic.intern();

        System.out.println(dynamic == interned);               // might be true (if first)
        System.out.println(interned == "world");               // true — same pool ref
    }
}
```

---

## Compile-Time Constant Folding

The Java compiler folds constant string expressions at compile time:

```java
public class ConstantFolding {

    public static void main(String[] args) {
        // Compile-time constants — folded by javac
        String a = "hel" + "lo";          // Compiler converts to "hello"
        String b = "hello";
        System.out.println(a == b);        // true — same pool entry

        // NOT a compile-time constant — concatenation happens at runtime
        String prefix = "hel";
        String c = prefix + "lo";          // Runtime concatenation → new object
        System.out.println(b == c);        // false

        // final makes it a compile-time constant
        final String finalPrefix = "hel";
        String d = finalPrefix + "lo";     // Compiler converts to "hello"
        System.out.println(b == d);        // true

        // Constants from other constant expressions
        final String X = "Java";
        final String Y = " Rocks";
        String z = X + Y;                 // Compiler converts to "Java Rocks"
        System.out.println(z == "Java Rocks");   // true
    }
}
```

---

## new String("abc") vs "abc"

```java
public class StringCreation {

    public static void main(String[] args) {
        // "abc" — ONE object created (in the pool, if not already there)
        String a = "abc";

        // new String("abc") — UP TO TWO objects:
        //   1. "abc" in the pool (if not already there)
        //   2. A new String object on the heap (always)
        String b = new String("abc");

        // This creates exactly 1 new heap object (pool entry already exists from 'a')
        String c = new String("abc");

        System.out.println(a == b);    // false
        System.out.println(b == c);    // false (different heap objects)
    }
}
```

### Object count analysis

```java
// How many String objects are created?

// Statement 1:
String s = "hello";
// Answer: 1 (pool entry, if "hello" doesn't exist yet)
//         0 (if "hello" already exists in pool)

// Statement 2:
String s = new String("hello");
// Answer: 2 (pool entry "hello" + new heap object)
//         1 (if "hello" already in pool: just the new heap object)

// Statement 3:
String s = new String("hel") + new String("lo");
// Answer: Up to 5: "hel" (pool) + new String("hel") + "lo" (pool)
//                   + new String("lo") + StringBuilder result
// Actually more with StringBuilder internals
```

---

## Memory Implications

### The case against excessive intern()

```java
// ❌ Anti-pattern: interning unique strings wastes memory and CPU
public void processLogs(List<String> logEntries) {
    for (String entry : logEntries) {
        String interned = entry.intern();   // Each unique entry stays in the pool forever*
        // Pool lookup has synchronization overhead
    }
}
// *In Java 7+, pooled strings CAN be GC'd, but the hashtable entry overhead remains
```

### The case for strategic intern()

```java
// ✅ Interning is beneficial for a bounded, repetitive set of values
public class CountryCache {

    public Country parse(String json) {
        String countryCode = extractCode(json);
        // Only ~200 country codes exist — great candidate for interning
        return new Country(countryCode.intern());
    }
}
```

---

## String Deduplication GC Feature

Since Java 8u20, the G1 GC can automatically deduplicate string **values** (the internal `char[]`/`byte[]`):

```bash
# Enable string deduplication (G1 GC only, default off)
java -XX:+UseG1GC -XX:+UseStringDeduplication MyApp

# Set the minimum age for strings to be eligible for dedup
java -XX:+UseG1GC -XX:+UseStringDeduplication -XX:StringDeduplicationAgeThreshold=3 MyApp
```

### How it works

```
Before dedup:
  String A ──▶ byte[] {'h','e','l','l','o'}     ← 5 bytes
  String B ──▶ byte[] {'h','e','l','l','o'}     ← 5 bytes (duplicate!)

After dedup:
  String A ──▶ byte[] {'h','e','l','l','o'}     ← 5 bytes (shared)
  String B ──┘                                   ← 0 extra bytes
```

### Key differences from intern()

| Aspect              | `intern()`                 | String Deduplication        |
|---------------------|----------------------------|-----------------------------|
| **What's shared**   | Entire String object       | Only the internal byte[]    |
| **When**            | Explicit call              | Automatic by G1 GC         |
| **Identity (`==`)** | Yes — same object          | No — different objects      |
| **CPU cost**        | Hashtable lookup           | Minimal (concurrent GC)    |
| **Scope**           | Application code           | GC-managed, transparent    |

---

## Compact Strings (Java 9+)

Java 9 introduced compact strings to reduce memory usage:

```java
// Before Java 9: String stored as char[] (2 bytes per char)
// "hello" → char[5] → 10 bytes

// Java 9+: String stored as byte[] with encoding flag
// "hello" → byte[5] + LATIN1 flag → 5 bytes    (ASCII/Latin-1)
// "héllo" → byte[10] + UTF16 flag → 10 bytes   (non-Latin-1)
```

```bash
# Compact strings are enabled by default (Java 9+)
# Disable if needed:
java -XX:-CompactStrings MyApp
```

---

## String Pool Size Tuning

```bash
# View the default pool size
java -XX:+PrintStringTableStatistics -version

# The pool is implemented as a hash table
# Default bucket count: 65536 (Java 11+)
# Increase for applications with many interned strings:
java -XX:StringTableSize=1000003 MyApp

# Use a prime number for better hash distribution
```

---

## Best Practices

```java
// ✅ Use string literals for constants — they're automatically pooled
// ✅ Use intern() for small, bounded sets of repetitive strings (status codes, country codes)
// ✅ Enable -XX:+UseStringDeduplication for heap-heavy applications on G1
// ✅ Use StringBuilder for dynamic string construction in loops
// ✅ Compare strings with .equals(), not ==

// ❌ Don't use new String("literal") — creates an unnecessary heap copy
// ❌ Don't intern() unbounded/dynamic strings (UUIDs, timestamps, user input)
// ❌ Don't assume == works for string comparison (even if it sometimes does)
// ❌ Don't disable compact strings unless you have a specific reason
```

---

## Interview Tips
- String Pool moved from PermGen to the main heap in Java 7 — pooled strings can be GC'd
- `"abc"` creates 1 object (pool); `new String("abc")` creates up to 2 objects
- The compiler folds string concatenation of compile-time constants (`final` + literals)
- `intern()` is useful for bounded, repetitive strings but harmful for unique strings
- String deduplication (G1 GC) shares the internal `byte[]` — different from `intern()` which shares the String object
- Compact Strings (Java 9+) use `byte[]` instead of `char[]`, halving ASCII string memory

---

[← Q8: Object Pooling](./q08-object-pooling.md) | [Next: Q10 - Memory Optimization →](./q10-memory-optimization.md)
