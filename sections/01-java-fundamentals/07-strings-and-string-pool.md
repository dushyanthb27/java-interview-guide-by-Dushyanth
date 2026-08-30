# Q7: How Does String Work in Java and What Is the String Pool?

## 📋 Question
How does String work in Java and what is the String Pool? Explain String immutability, `StringBuilder` vs `StringBuffer`, and modern String features.

## Difficulty: 🟡 Medium

---

## 💡 Answer

`String` in Java is a **final, immutable class** that represents a sequence of characters. Once a `String` object is created, its content **cannot be changed** — any operation that appears to modify a String actually creates a **new String object**. This immutability is a deliberate design choice that enables thread safety, caching, security, and hash code optimization.

### String Immutability

The `String` class is declared as `final class String` (cannot be subclassed) and stores characters in a `private final byte[]` array (since Java 9; previously `char[]`). There are no setter methods. When you call `str.toUpperCase()`, `str.concat()`, or use `+`, a **new** String is created — the original is untouched.

**Why immutable?**
1. **Thread Safety**: Strings can be shared across threads without synchronization.
2. **String Pool**: Immutability makes interning (sharing) safe — if one reference could change the String, all references would see the change.
3. **Security**: Strings are used for class loading, file paths, network connections, and DB URLs. Immutability prevents tampering after validation.
4. **Hash Code Caching**: `String.hashCode()` is computed once and cached, making Strings ideal `HashMap` keys.

### String Pool (Intern Pool)

The String Pool is a special **memory area** (in the heap since Java 7; previously in PermGen) where Java stores unique String literals. When you write `"Hello"` in source code, the JVM checks the pool first — if an identical String exists, the existing reference is returned instead of creating a new object. This saves memory when the same strings appear frequently.

- **String literals** are automatically interned: `String s = "Hello";` → goes to pool
- **`new String("Hello")`** creates a new object on the heap **outside** the pool (even if "Hello" is already in the pool)
- **`str.intern()`** explicitly adds a String to the pool (or returns the existing pooled instance)

### String vs StringBuilder vs StringBuffer

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|---------------|--------------|
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Yes (immutable) | No | Yes (synchronized) |
| **Performance** | Slow for concatenation | Fast | Slower (sync overhead) |
| **Since** | JDK 1.0 | JDK 1.5 | JDK 1.0 |
| **Use When** | Few modifications | Many modifications (single thread) | Many modifications (multi-thread) |

### Code Example

```java
package com.interview.fundamentals;

public class StringPoolDemo {

    public static void main(String[] args) {

        // === STRING POOL BEHAVIOR ===

        // ✅ String literals are interned (same pool reference)
        String s1 = "Hello";
        String s2 = "Hello";
        System.out.println(s1 == s2);       // true  (same reference from pool)
        System.out.println(s1.equals(s2));   // true  (same content)

        // ❌ new String() creates a SEPARATE object on the heap
        String s3 = new String("Hello");
        System.out.println(s1 == s3);        // false (different references)
        System.out.println(s1.equals(s3));   // true  (same content)

        // ✅ intern() returns the pooled reference
        String s4 = s3.intern();
        System.out.println(s1 == s4);        // true  (s4 points to pooled "Hello")

        // === IMMUTABILITY DEMONSTRATION ===
        String original = "Java";
        String modified = original.concat(" 17");
        System.out.println(original);  // "Java"    — unchanged (immutable!)
        System.out.println(modified);  // "Java 17" — new String object

        // ✅ String concatenation at compile time
        String compile1 = "Hello" + " " + "World";  // Compiler folds to "Hello World"
        String compile2 = "Hello World";
        System.out.println(compile1 == compile2);    // true (same pooled constant)

        // ❌ Runtime concatenation creates new objects
        String part = "World";
        String runtime = "Hello " + part;   // Not pooled (runtime computation)
        System.out.println(runtime == compile2);  // false

        // === STRING METHODS ===
        stringMethodsDemo();
    }

    static void stringMethodsDemo() {
        String text = "  Hello, Java Developer!  ";

        // ✅ Common String methods (all return NEW Strings)
        System.out.println(text.trim());                    // "Hello, Java Developer!"
        System.out.println(text.strip());                   // "Hello, Java Developer!" (Java 11+, Unicode-aware)
        System.out.println(text.stripLeading());            // "Hello, Java Developer!  "
        System.out.println(text.stripTrailing());           // "  Hello, Java Developer!"
        System.out.println(text.trim().toLowerCase());      // "hello, java developer!"
        System.out.println(text.trim().toUpperCase());      // "HELLO, JAVA DEVELOPER!"
        System.out.println(text.trim().contains("Java"));   // true
        System.out.println(text.trim().startsWith("Hello")); // true
        System.out.println(text.trim().indexOf("Java"));    // 7
        System.out.println(text.trim().substring(7, 11));   // "Java"
        System.out.println(text.trim().replace("Java", "Spring")); // "Hello, Spring Developer!"
        System.out.println(text.trim().charAt(0));          // 'H'
        System.out.println(text.isBlank());                 // false (Java 11+)
        System.out.println("   ".isBlank());                // true  (Java 11+)

        // ✅ Java 11+ repeat()
        System.out.println("Ha".repeat(3));  // "HaHaHa"

        // ✅ Java 12+ formatted() / transform()
        String formatted = "Name: %s, Age: %d".formatted("Alice", 30);
        System.out.println(formatted);  // "Name: Alice, Age: 30"
    }
}
```

### StringBuilder and Text Blocks

```java
package com.interview.fundamentals;

public class StringBuilderAndTextBlocks {

    public static void main(String[] args) {

        // === STRINGBUILDER (mutable, not thread-safe, fast) ===

        // ❌ Bad: String concatenation in a loop (creates many intermediate objects)
        String result = "";
        for (int i = 0; i < 1000; i++) {
            result += i + ", ";  // Each += creates a new String object
        }

        // ✅ Good: StringBuilder for dynamic string building
        StringBuilder sb = new StringBuilder(4096);  // Pre-allocate capacity
        for (int i = 0; i < 1000; i++) {
            sb.append(i).append(", ");  // Modifies internal buffer in-place
        }
        String efficient = sb.toString();

        // ✅ StringBuilder fluent API
        String html = new StringBuilder()
            .append("<html>")
            .append("<body>")
            .append("<h1>Hello</h1>")
            .append("</body>")
            .append("</html>")
            .toString();

        // === TEXT BLOCKS (Java 15+) ===

        // ❌ Old way: escaped strings
        String jsonOld = "{\n" +
            "  \"name\": \"Alice\",\n" +
            "  \"age\": 30\n" +
            "}";

        // ✅ New way: text blocks — preserve formatting, no escaping quotes
        String jsonNew = """
                {
                  "name": "Alice",
                  "age": 30
                }
                """;

        System.out.println(jsonNew);

        // ✅ Text block with formatted()
        String sql = """
                SELECT *
                FROM users
                WHERE name = '%s'
                  AND age > %d
                """.formatted("Alice", 25);

        System.out.println(sql);

        // ✅ Text block indentation is relative to closing """
        String compact = """
                Line 1
                Line 2
                """;  // Closing """ position determines base indentation
    }
}
```

### ⚠️ Common Mistake
**Using `==` instead of `.equals()` for String comparison**: `==` compares references, not content. It "works" for literals (because they're pooled) but fails for dynamically created Strings. This creates bugs that pass in testing (with hardcoded strings) but fail in production (with user input or API responses). Also, **never use `new String("literal")`** — it creates an unnecessary extra object outside the pool. Sonar and most linters flag this as a code smell.

```java
// ❌ This bug is hiding in YOUR codebase
String userInput = new String("admin");
if (userInput == "admin") {
    // NEVER REACHED — different references!
}

// ✅ Always use .equals()
if ("admin".equals(userInput)) {
    // Correct! Also null-safe (won't NPE if userInput is null)
}
```

### ⚡ Performance Tip
Since Java 9, Strings use **Compact Strings** (`byte[]` instead of `char[]`). Latin-1 characters (ASCII) use 1 byte per character (LATIN1 encoding), while characters beyond Latin-1 use 2 bytes per character (UTF-16 encoding). This cuts memory usage nearly in half for most English/ASCII text. The JVM also performs **String deduplication** with G1 GC (`-XX:+UseStringDeduplication`) — it identifies Strings with identical content during GC and points them to the same `byte[]` array, reducing heap usage significantly in applications with many duplicate strings (like JSON parsing or log processing).

### 🔧 JVM Internals
The String Pool is implemented as a **hash table** in native memory (C++ side of the JVM), not in the Java heap. String literals are interned during class loading by the constant pool resolution process. The pool size is configurable with `-XX:StringTableSize=<n>` (default ~60,000 buckets). You can monitor the pool with `jcmd <pid> VM.stringtable`. Since Java 7, the pool was moved from **PermGen** to the **main heap**, meaning pooled strings are eligible for garbage collection when no longer referenced. The `intern()` method performs a hash lookup in this table — if found, returns the existing reference; if not, adds the string and returns it. Excessive `intern()` calls with unique strings can fill the table and degrade performance due to hash collisions.

---

## 🎯 Interviewer Insight
This is one of the most frequently asked Java interview questions. Interviewers want you to explain **why** Strings are immutable (security, thread safety, caching, pool), the exact behavior of the String Pool with concrete examples, and the performance difference between String concatenation and StringBuilder. Strong candidates mention Compact Strings (Java 9+), text blocks (Java 15+), and the JVM's string deduplication feature. Putting `.equals()` on the **literal side** (`"admin".equals(input)`) to avoid NPE is a practical best practice that shows production experience.

## ⚡ Follow-up Questions
1. How many String objects are created by `String s = new String("Hello")`?
2. Why is String a popular HashMap key, and what would happen if String were mutable?
3. What is the difference between `trim()` and `strip()` (Java 11+)?

## 📚 Related Topics
- [Data Types and Primitives](./q03-data-types-primitives.md)
- [Final Keyword](./q11-final-keyword.md)
- [Java 8+ Features](../06-java8-plus-features/README.md)
