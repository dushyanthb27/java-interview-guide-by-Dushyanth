# Q17: Static and Private Methods in Interfaces

## Question
What are static and private methods in interfaces? Explain their use cases, how they enable code reuse, and how Java interfaces have evolved from pure abstraction to rich contracts.

## Difficulty: 🟠 Advanced

---

## Answer

Java interfaces have evolved significantly across versions. **Static methods** (Java 8+) provide utility/factory methods, while **private methods** (Java 9+) enable code reuse within default methods without exposing implementation details.

### Evolution of Java Interfaces

| Version | Feature | Purpose |
|---------|---------|---------|
| Java 1.0 | Abstract methods + constants | Pure contract |
| Java 8 | `default` methods | Interface evolution |
| Java 8 | `static` methods | Utility/factory methods |
| Java 9 | `private` methods | Code reuse in defaults |
| Java 9 | `private static` methods | Shared helpers for static methods |
| Java 17 | `sealed` interfaces | Controlled implementation |

### Static Methods in Interfaces (Java 8+)

Static methods belong to the interface itself, not to implementing classes. They are **not inherited**.

```java
public interface Validator<T> {
    boolean isValid(T value);
    String getErrorMessage();

    // ✅ Factory methods - create common implementations
    static Validator<String> nonEmpty() {
        return new Validator<>() {
            @Override
            public boolean isValid(String value) {
                return value != null && !value.isBlank();
            }
            @Override
            public String getErrorMessage() { return "Value must not be empty"; }
        };
    }

    static Validator<String> maxLength(int max) {
        return new Validator<>() {
            @Override
            public boolean isValid(String value) {
                return value != null && value.length() <= max;
            }
            @Override
            public String getErrorMessage() { return "Max length: " + max; }
        };
    }

    static Validator<String> matchesPattern(String regex) {
        var pattern = java.util.regex.Pattern.compile(regex);
        return new Validator<>() {
            @Override
            public boolean isValid(String value) {
                return value != null && pattern.matcher(value).matches();
            }
            @Override
            public String getErrorMessage() { return "Must match: " + regex; }
        };
    }

    // ✅ Utility method - combines validators
    @SafeVarargs
    static <T> Validator<T> allOf(Validator<T>... validators) {
        return new Validator<>() {
            @Override
            public boolean isValid(T value) {
                return Arrays.stream(validators).allMatch(v -> v.isValid(value));
            }
            @Override
            public String getErrorMessage() {
                return Arrays.stream(validators)
                             .map(Validator::getErrorMessage)
                             .collect(Collectors.joining(", "));
            }
        };
    }
}

// Usage
var emailValidator = Validator.allOf(
    Validator.nonEmpty(),
    Validator.maxLength(255),
    Validator.matchesPattern("^[\\w.]+@[\\w.]+\\.[a-z]{2,}$")
);

System.out.println(emailValidator.isValid("user@example.com")); // true
System.out.println(emailValidator.isValid(""));                  // false
```

### Key Rule: Static Methods Are NOT Inherited

```java
public interface MyInterface {
    static void staticMethod() {
        System.out.println("Interface static");
    }
}

public class MyClass implements MyInterface {
    // MyClass.staticMethod();     // ❌ COMPILE ERROR
    // MyInterface.staticMethod(); // ✅ Must call via interface name
}

// This is different from abstract class static methods:
public abstract class MyAbstract {
    public static void staticMethod() { System.out.println("Abstract static"); }
}
public class MyChild extends MyAbstract {
    // MyChild.staticMethod(); // ✅ Works - inherited from abstract class
}
```

### Private Methods in Interfaces (Java 9+)

Private methods allow code reuse between default methods without polluting the public API:

```java
public interface DataExporter {

    // Abstract method - implementors provide the data
    List<Map<String, Object>> getData();

    // ✅ Default methods with shared logic
    default String exportToCsv() {
        var data = getData();
        if (data.isEmpty()) return "";

        var headers = getHeaders(data);
        var sb = new StringBuilder();
        sb.append(String.join(",", headers)).append("\n");

        for (var row : data) {
            sb.append(formatRow(row, headers)).append("\n"); // Private helper
        }
        return sb.toString();
    }

    default String exportToHtml() {
        var data = getData();
        if (data.isEmpty()) return "<table></table>";

        var headers = getHeaders(data);
        var sb = new StringBuilder("<table>\n<tr>");
        headers.forEach(h -> sb.append("<th>").append(h).append("</th>"));
        sb.append("</tr>\n");

        for (var row : data) {
            sb.append("<tr>");
            appendHtmlRow(sb, row, headers); // Private helper
            sb.append("</tr>\n");
        }
        sb.append("</table>");
        return sb.toString();
    }

    // ✅ Private method - shared between default methods
    private List<String> getHeaders(List<Map<String, Object>> data) {
        return new ArrayList<>(data.get(0).keySet());
    }

    // ✅ Private method - reusable formatting logic
    private String formatRow(Map<String, Object> row, List<String> headers) {
        return headers.stream()
                      .map(h -> String.valueOf(row.getOrDefault(h, "")))
                      .collect(Collectors.joining(","));
    }

    private void appendHtmlRow(StringBuilder sb, Map<String, Object> row, List<String> headers) {
        headers.forEach(h ->
            sb.append("<td>").append(row.getOrDefault(h, "")).append("</td>")
        );
    }
}
```

### Private Static Methods (Java 9+)

```java
public interface IdGenerator {

    // Static factory methods
    static String uuid() {
        return formatId("UUID", java.util.UUID.randomUUID().toString());
    }

    static String sequential(long sequence) {
        return formatId("SEQ", String.format("%010d", sequence));
    }

    static String timestamped() {
        return formatId("TS", String.valueOf(System.currentTimeMillis()));
    }

    // ✅ Private static helper - shared between static methods
    private static String formatId(String prefix, String value) {
        return prefix + "-" + value.toUpperCase().replace("-", "");
    }
}

// Usage
System.out.println(IdGenerator.uuid());          // UUID-A1B2C3D4E5...
System.out.println(IdGenerator.sequential(42));   // SEQ-0000000042
System.out.println(IdGenerator.timestamped());    // TS-1700000000000
```

### Interface as Mini-Utility Class

```java
public interface StringUtils {
    // ✅ Static methods turn an interface into a utility namespace
    static String capitalize(String s) {
        return s == null || s.isEmpty() ? s : Character.toUpperCase(s.charAt(0)) + s.substring(1);
    }

    static String truncate(String s, int maxLength) {
        return truncate(s, maxLength, "...");
    }

    static String truncate(String s, int maxLength, String suffix) {
        if (s == null || s.length() <= maxLength) return s;
        return s.substring(0, maxLength - suffix.length()) + suffix;
    }

    static boolean isNullOrBlank(String s) {
        return s == null || s.isBlank();
    }

    // Private static: shared utility logic
    private static String normalizeWhitespace(String s) {
        return s.replaceAll("\\s+", " ").trim();
    }
}
```

### Complete Interface Evolution Example

```java
// Showing ALL interface method types together
public interface Cacheable<K, V> {

    // 1. Abstract method (Java 1.0) - implementor contract
    V computeValue(K key);

    // 2. Default method (Java 8) - optional behavior
    default V getOrCompute(K key, Map<K, V> cache) {
        return cache.computeIfAbsent(key, this::computeValue);
    }

    default void warmUp(List<K> keys, Map<K, V> cache) {
        keys.forEach(key -> preload(key, cache)); // Uses private method
    }

    // 3. Static method (Java 8) - factory/utility
    static <K, V> Map<K, V> createCache() {
        return new ConcurrentHashMap<>();
    }

    static <K, V> Map<K, V> createCache(int initialCapacity) {
        validateCapacity(initialCapacity); // Uses private static
        return new ConcurrentHashMap<>(initialCapacity);
    }

    // 4. Private method (Java 9) - code reuse in default methods
    private void preload(K key, Map<K, V> cache) {
        if (!cache.containsKey(key)) {
            V value = computeValue(key);
            if (value != null) {
                cache.put(key, value);
            }
        }
    }

    // 5. Private static method (Java 9) - code reuse in static methods
    private static void validateCapacity(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive: " + capacity);
        }
    }
}
```

### Access Rules Summary

| Method Type | Can Access | Inherited? |
|-------------|-----------|-----------|
| Abstract | N/A (no body) | ✅ Yes |
| Default | Other default, abstract, private | ✅ Yes |
| Static | Other static, private static | ❌ No |
| Private | Other default, abstract, private | ❌ No |
| Private Static | Other static, private static | ❌ No |

### Common Mistake

```java
// ❌ Trying to call interface static method through implementing class
public interface Factory {
    static Factory create() { return null; }
}
public class MyFactory implements Factory { }
// MyFactory.create();  // COMPILE ERROR
// Factory.create();    // ✅ Correct

// ❌ Making a private method that should be default (or vice versa)
public interface Configurable {
    // This should be default if implementors might want to override
    private void configure() { } // Can't be overridden!

    // ✅ Use default for overridable behavior
    default void configure(Map<String, String> props) { }
}

// ❌ Private methods in interfaces before Java 9
// Will not compile on Java 8
```

### Performance Tip
Static interface methods are resolved at compile time (like regular static calls), making them as fast as class static methods. Private methods are inlined by the JIT compiler since they cannot be overridden. Both are zero-cost abstractions for organizing code.

---

## Interviewer Insight
Interviewers want to see you trace the **evolution** of interfaces across Java versions and explain the **motivation** for each addition. Understanding that static methods are NOT inherited (unlike abstract class statics) is a key differentiator.

## Follow-up Questions
1. Why are static interface methods not inherited by implementing classes?
2. How do private methods in interfaces relate to the DRY principle?
3. Can a Java 8 project use private methods in interfaces?

## Related Topics
- [Default Methods in Interfaces](./q16-default-methods-interfaces.md)
- [Interfaces vs Abstract Classes](./q07-interfaces-vs-abstract-classes.md)
- [Functional Interfaces](./q15-functional-interfaces.md)
