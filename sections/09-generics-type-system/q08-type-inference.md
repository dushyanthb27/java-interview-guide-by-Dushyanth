# Q8: How Does Type Inference Work?

[Home](../../README.md) > [Generics & Type System](./README.md) > Type Inference

---

## 🟡 Difficulty: Intermediate

## Question
> How does Java infer generic types? How has type inference evolved from Java 5 through Java 10+?

---

## Quick Answer
**Type inference** is the compiler's ability to determine type arguments automatically from context — method arguments, return type, or assignment target. Java 7 introduced the **diamond operator** (`<>`) for constructor inference. Java 8 added **target-type inference** for lambdas and method references. Java 10 added `var` for local variable type inference. Each version has reduced boilerplate while maintaining full type safety.

---

## Detailed Explanation

### Method Type Inference (Java 5+)

```java
public class Util {
    public static <T> List<T> emptyList() {
        return new ArrayList<>();
    }

    public static <T> T firstNonNull(T a, T b) {
        return a != null ? a : b;
    }

    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) >= 0 ? a : b;
    }
}

// Type inferred from arguments
String name = Util.firstNonNull(null, "default");   // T = String
Integer bigger = Util.max(10, 20);                   // T = Integer

// Type inferred from assignment context
List<String> names = Util.emptyList();               // T = String
List<Integer> nums = Util.emptyList();               // T = Integer

// Explicit type witness (rarely needed)
List<String> list = Util.<String>emptyList();
```

### Inference from Method Arguments

```java
public static <T> Set<T> setOf(T... elements) {
    return new HashSet<>(Arrays.asList(elements));
}

// Compiler infers T from the arguments
Set<String> strings = setOf("a", "b", "c");    // T = String
Set<Integer> ints = setOf(1, 2, 3);            // T = Integer

// Mixed types — compiler finds common supertype
Set<Number> mixed = setOf(1, 2.0, 3L);
// T inferred as Number (common supertype of Integer, Double, Long)

// Comparable inference
Set<Comparable<?>> odds = setOf("hello", 42);
// T inferred as Comparable & Serializable (common interface)
```

### Diamond Operator Inference (Java 7+)

```java
// Before Java 7 — repetitive type specification
Map<String, List<Integer>> map1 = new HashMap<String, List<Integer>>();

// Java 7+ — diamond operator infers from left-hand side
Map<String, List<Integer>> map2 = new HashMap<>();   // ✅ Clean

// Works with nested generics
List<Map.Entry<String, Integer>> entries = new ArrayList<>();

// Works with constructors that take arguments
List<String> names = new ArrayList<>(existingCollection);
```

### Target-Type Inference (Java 8+)

```java
// Java 8 improved inference for lambdas and nested generics

// Inference through method chains
List<String> names = Collections.emptyList();  // Works in Java 5+

// Java 8: inference in method call context (not just assignment)
void processNames(List<String> names) { }
processNames(Collections.emptyList());  // ✅ T = String inferred from parameter

// Lambda parameter types inferred from context
List<String> sorted = names.stream()
    .filter(s -> s.length() > 3)      // s inferred as String
    .sorted((a, b) -> a.compareTo(b))  // a, b inferred as String
    .collect(Collectors.toList());

// Method reference inference
Function<String, Integer> parser = Integer::parseInt;
// T = String, R = Integer inferred from Function<String, Integer>
```

### Chained Method Inference

```java
// Java 8 improved inference for chained calls

// This works in Java 8+ but failed in Java 7
List<String> result = Collections.<String>emptyList();  // Java 7 needed this
List<String> result = Collections.emptyList();           // Java 8+ handles it

// Complex chain inference
Map<String, List<Integer>> map = Stream.of("a", "b", "c")
    .collect(Collectors.toMap(
        Function.identity(),           // Key mapper: String → String
        s -> new ArrayList<>()         // Value mapper: String → List<Integer>
    ));                                // Types inferred through entire chain
```

### Local Variable Type Inference — var (Java 10+)

```java
// var infers the type from the right-hand side
var names = new ArrayList<String>();     // Inferred as ArrayList<String>
var map = new HashMap<String, Integer>(); // Inferred as HashMap<String, Integer>
var entry = Map.entry("key", 42);        // Inferred as Map.Entry<String, Integer>

// var with diamond — infers as Object if no context
var list1 = new ArrayList<String>();  // ArrayList<String> ✅
var list2 = new ArrayList<>();        // ArrayList<Object> ⚠️ probably not what you want

// var preserves the specific type, not the interface
var names2 = new ArrayList<String>();  // Type is ArrayList<String>, NOT List<String>
List<String> names3 = new ArrayList<>(); // Type is List<String> — more abstract

// var with generics in loops
Map<String, List<Integer>> data = getData();
for (var entry2 : data.entrySet()) {
    // entry2 is Map.Entry<String, List<Integer>>
    var key = entry2.getKey();        // String
    var values = entry2.getValue();   // List<Integer>
}

// var with streams
var result = names.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());   // List<String>
```

### var Limitations

```java
// ❌ Cannot use var without initializer
// var x;                        // COMPILE ERROR

// ❌ Cannot use var with null
// var x = null;                 // COMPILE ERROR — type unclear

// ❌ Cannot use var for method parameters
// void process(var item) { }    // COMPILE ERROR

// ❌ Cannot use var for fields
// class MyClass { var field = 5; }  // COMPILE ERROR

// ❌ Cannot use var with lambda without target type
// var func = (x) -> x + 1;     // COMPILE ERROR

// ✅ CAN use var with explicitly typed lambda (Java 11+)
// var func = (Function<Integer, Integer>) x -> x + 1;  // OK but verbose
```

### Inference with Conditional (Ternary) Expressions

```java
// Compiler infers common type for ternary branches
var result = condition ? new ArrayList<String>() : new LinkedList<String>();
// Type: AbstractList<String> & Serializable & Cloneable (intersection type)

// Can be surprising:
var value = condition ? 42 : 3.14;
// Type: double (int widened to double)

var text = condition ? "hello" : null;
// Type: String (null is compatible)
```

### Inference Failures and Type Witnesses

```java
// Sometimes inference fails and you need a type witness

// ❌ Ambiguous — compiler can't decide T
// Collections.emptyList().add("test");  // T = Object, not String

// ✅ Type witness resolves ambiguity
Collections.<String>emptyList();

// ✅ Assignment context resolves it
List<String> list = Collections.emptyList();

// ❌ Nested inference may fail
// Map<String, List<String>> map = Map.of("key", List.of());  // May need help

// ✅ Explicit type for inner expression
Map<String, List<String>> map = Map.of("key", List.<String>of());
```

### Inference with Anonymous Classes

```java
// Java 9: diamond with anonymous inner classes
// Java 8: ❌ new Comparator<>() {}  // COMPILE ERROR
// Java 9: ✅ new Comparator<>() {}  // Inferred from context

Comparator<String> comp = new Comparator<>() {  // Java 9+
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
};
```

---

## Anti-Patterns

```java
// ❌ Relying on var when the type is not obvious
var result = service.process(data);  // What type is result?

// ❌ Using var with diamond operator — loses type info
var list = new ArrayList<>();  // ArrayList<Object> — probably wrong

// ❌ Over-using explicit type witnesses
Util.<String>firstNonNull("a", "b");  // Unnecessary — already inferred

// ❌ Assuming var makes Java dynamically typed
var x = "hello";
// x = 42;  // ❌ COMPILE ERROR — x is still String
```

## Best Practices

```java
// ✅ Use diamond operator for constructors
Map<String, List<Integer>> map = new HashMap<>();

// ✅ Let method inference do its job
List<String> names = List.of("Alice", "Bob");  // T inferred as String

// ✅ Use var when the type is obvious from the RHS
var buffer = new StringBuilder();          // Obviously StringBuilder
var formatter = DateTimeFormatter.ISO_DATE; // Obviously DateTimeFormatter

// ✅ Avoid var when the type is not clear
UserService service = ServiceFactory.create();  // Better than var

// ✅ Use var in for-each to reduce verbosity
for (var entry : map.entrySet()) {
    // entry type is clear from context
}

// ✅ Use type witness only when inference fails
List<String> empty = Collections.<String>emptyList();  // Only if needed
```

---

## Interview Tips
- Type inference is **compile-time only** — runtime behavior is unchanged
- Diamond operator (`<>`) infers from the **left-hand side** (assignment target)
- Java 8 target-type inference works through **method parameter context**
- `var` infers from the **right-hand side** — it's the opposite of diamond
- `var` is NOT dynamic typing — the type is fixed at compile time
- `var` + `<>` together = `ArrayList<Object>` — avoid this combination

---

[← Q7: Generic Constraints](./q07-generic-constraints.md) | [Next: Q9 - Diamond Operator →](./q09-diamond-operator.md)
