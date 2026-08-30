# String Handling Optimization

[Home](../../README.md) > [Performance Optimization](./README.md) > String Optimization

---

## Overview

Strings are the most commonly used objects in Java applications. Since `String` is immutable, every modification creates a new object. Understanding string internals and optimization techniques can significantly reduce memory usage and improve performance.

---

## 1. StringBuilder vs String Concatenation

```java
// BAD: String concatenation in a loop
public String buildCsv(List<String> values) {
    String result = "";
    for (String value : values) {
        result += value + ",";  // creates new String EACH iteration
    }
    return result;
}
// For 10,000 elements: creates ~10,000 intermediate String objects
// Time complexity: O(n^2) due to repeated copying

// GOOD: StringBuilder
public String buildCsv(List<String> values) {
    StringBuilder sb = new StringBuilder(values.size() * 16); // pre-size estimate
    for (String value : values) {
        sb.append(value).append(',');
    }
    if (sb.length() > 0) sb.setLength(sb.length() - 1); // remove trailing comma
    return sb.toString();
}
// Time complexity: O(n) - single buffer, amortized resizing
```

### When Concatenation Is Fine

```java
// The compiler optimizes simple concatenation (no loop)
// These are equivalent after compilation:
String msg = "Hello, " + name + "! Age: " + age;
String msg = new StringBuilder().append("Hello, ").append(name)
    .append("! Age: ").append(age).toString();

// Java 9+: Uses invokedynamic for string concatenation (even better)
// The JIT can optimize further at runtime
```

### Benchmark Results

```
10,000 concatenations:
  String += :     ~850 ms    (O(n^2), creates 10K objects)
  StringBuilder:  ~0.5 ms    (O(n), single buffer)
  StringBuffer:   ~0.8 ms    (O(n), synchronized - unnecessary overhead)
```

---

## 2. String.join() and StringJoiner

```java
// Java 8+: String.join() for simple delimited joining
List<String> words = List.of("Hello", "World", "Java");

String joined = String.join(", ", words);
// Result: "Hello, World, Java"

// StringJoiner for more control
StringJoiner sj = new StringJoiner(", ", "[", "]"); // delimiter, prefix, suffix
sj.add("alpha").add("beta").add("gamma");
String result = sj.toString();
// Result: "[alpha, beta, gamma]"

// Collectors.joining() in streams
String csv = words.stream()
    .filter(w -> w.length() > 3)
    .collect(Collectors.joining(", ", "Words: ", "."));
// Result: "Words: Hello, World, Java."
```

---

## 3. Compact Strings (Java 9+)

Java 9 introduced Compact Strings: strings containing only Latin-1 characters use a `byte[]` with 1 byte per character instead of `char[]` with 2 bytes per character.

```java
// Latin-1 string (1 byte per character) - compact encoding
String ascii = "Hello World";  // 11 bytes (was 22 bytes pre-Java 9)

// Non-Latin-1 string (2 bytes per character) - UTF-16 encoding
String unicode = "Hello ";   // 14 bytes (UTF-16 encoding)

// Memory savings: ~50% for ASCII-only strings
// Most English/code strings are ASCII -> significant heap savings
```

```bash
# Compact Strings enabled by default in Java 9+
# Disable for benchmarking:
java -XX:-CompactStrings -jar app.jar
```

### Memory Impact

```
Pre-Java 9:  "Hello" = 12 bytes (header) + 10 bytes (char[5]) = ~40 bytes total
Java 9+:     "Hello" = 12 bytes (header) + 5 bytes (byte[5])  = ~32 bytes total
                                                                  ~20% less
For applications with millions of strings, savings are substantial.
```

---

## 4. String Deduplication

G1 GC can automatically detect and deduplicate identical `char[]`/`byte[]` arrays across different String objects.

```bash
# Enable String Deduplication (G1 GC only)
java -XX:+UseG1GC -XX:+UseStringDeduplication -jar app.jar

# Tune deduplication age threshold
java -XX:StringDeduplicationAgeThreshold=3 -jar app.jar
```

```java
// How deduplication works internally:
String s1 = new String("Hello");  // has its own byte[]
String s2 = new String("Hello");  // has its own byte[]
// Without dedup: s1.value != s2.value (two byte[] arrays)
// With dedup: G1 makes s1.value == s2.value (shared byte[] array)

// The String objects remain separate, but their backing arrays are shared
// This is transparent to the application
```

### When String Deduplication Helps

```
Scenario                         Typical Savings
--------------------------------------------------
Web apps with repeated headers   10-20% heap reduction
XML/JSON parsing                 15-25% heap reduction
Log message formatting           5-15% heap reduction
Internationalization strings     10-30% heap reduction
```

---

## 5. String.intern() Considerations

```java
// intern() adds string to the internal string pool (in Metaspace since Java 7)
String s1 = new String("Hello").intern();
String s2 = "Hello"; // string literal - already in pool
System.out.println(s1 == s2);  // true (same reference)

// Useful: small set of frequently compared strings
public class StatusCode {
    // Intern known values for fast == comparison
    public static final String OK = "OK".intern();
    public static final String ERROR = "ERROR".intern();
    public static final String PENDING = "PENDING".intern();
}
```

### When NOT to Use intern()

```java
// DANGER: intern() on arbitrary user input -> memory leak
public void processRequest(String userId) {
    String interned = userId.intern(); // DON'T DO THIS!
    // If millions of unique userIds, the string pool grows unboundedly
}

// intern() performance characteristics:
// - Lookup is O(1) amortized (hash table in native memory)
// - But acquiring the intern table lock is expensive under contention
// - String pool size: -XX:StringTableSize=60013 (default, prime number)
```

```bash
# Tune string table size for heavy intern() usage
java -XX:StringTableSize=200003 -jar app.jar

# Monitor string table
jcmd <pid> VM.stringtable
```

---

## 6. Text Blocks (Java 15+)

```java
// Text blocks are more readable AND avoid concatenation overhead
// BAD: String concatenation for multi-line text
String json = "{\n" +
    "  \"name\": \"" + name + "\",\n" +
    "  \"age\": " + age + "\n" +
    "}";

// GOOD: Text block (compiled as single string constant)
String jsonTemplate = """
    {
      "name": "%s",
      "age": %d
    }
    """;
String json = String.format(jsonTemplate, name, age);

// SQL queries
String sql = """
    SELECT u.name, u.email, o.total
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE u.status = 'ACTIVE'
    ORDER BY o.total DESC
    LIMIT 100
    """;
```

---

## 7. StringBuffer vs StringBuilder

```java
// StringBuffer: synchronized (thread-safe) - LEGACY, avoid in new code
StringBuffer buffer = new StringBuffer();  // every method is synchronized

// StringBuilder: unsynchronized (not thread-safe) - PREFERRED
StringBuilder builder = new StringBuilder();  // no synchronization overhead

// Performance difference: 20-30% faster for StringBuilder
// StringBuffer is only needed if shared across threads (extremely rare)
```

### Benchmark

```
1 million appends:
  StringBuilder: ~12 ms
  StringBuffer:  ~16 ms  (~30% slower due to synchronization)
  String +=:     ~85000 ms  (catastrophically slow)
```

---

## 8. Advanced String Optimization Patterns

```java
// Pattern 1: Pre-compiled regex (avoid recompilation)
// BAD
public boolean isValid(String input) {
    return input.matches("\\d{3}-\\d{2}-\\d{4}"); // compiles regex each call!
}

// GOOD
private static final Pattern SSN_PATTERN =
    Pattern.compile("\\d{3}-\\d{2}-\\d{4}");

public boolean isValid(String input) {
    return SSN_PATTERN.matcher(input).matches(); // reuses compiled pattern
}

// Pattern 2: charAt() instead of substring for single characters
// BAD
if (str.substring(0, 1).equals("A")) { ... }

// GOOD
if (str.charAt(0) == 'A') { ... }

// Pattern 3: isEmpty() / isBlank() instead of length comparison
// BAD
if (str.length() == 0) { ... }
if (str.trim().length() == 0) { ... }

// GOOD
if (str.isEmpty()) { ... }
if (str.isBlank()) { ... }  // Java 11+ (checks whitespace too)

// Pattern 4: Avoid unnecessary toString()
Object obj = "Hello";
// BAD
System.out.println(obj.toString());
// GOOD
System.out.println(obj);  // println calls toString() internally

// Pattern 5: Use char instead of String for single characters
StringBuilder sb = new StringBuilder();
// BAD
sb.append(",");
// GOOD
sb.append(',');  // avoids String object overhead
```

---

## 9. String Performance Quick Reference

| Operation                | Fast                        | Slow                          |
|-------------------------|-----------------------------|-------------------------------|
| Concatenation in loop   | `StringBuilder`              | `String +=`                   |
| Joining with delimiter  | `String.join()`              | Manual loop                   |
| Regex matching          | Pre-compiled `Pattern`       | `String.matches()`           |
| Single char check       | `charAt()`                   | `substring(0,1)`             |
| Empty check             | `isEmpty()`                  | `length() == 0`              |
| String building         | `StringBuilder`              | `StringBuffer`               |
| Comparison              | `equals()` (or `==` interned)| Manual char-by-char          |
| Multi-line strings      | Text blocks                  | Concatenation with `\n`      |

---

## Interview Tips

- `StringBuilder` in loops is essential; compiler optimizes single-line concatenation
- Java 9+ Compact Strings save ~50% memory for ASCII strings automatically
- `String.intern()` is useful for small, known sets but dangerous for arbitrary input
- Pre-compile `Pattern` objects; `String.matches()` recompiles every call
- G1 String Deduplication helps memory-heavy apps with repeated strings
- Use `char` literals (`','`) instead of `String` literals (`","`) in `StringBuilder.append()`
- `StringBuffer` is effectively obsolete; always prefer `StringBuilder`

---

[<< Previous: Collection Performance](./q05-collection-performance.md) | [Next: I/O Optimization >>](./q07-io-optimization.md)
