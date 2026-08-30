# Q9: What Is the Diamond Operator?

[Home](../../README.md) > [Generics & Type System](./README.md) > Diamond Operator

---

## 🟢 Difficulty: Basic

## Question
> What is the diamond operator (`<>`) in Java? How has it evolved across Java versions?

---

## Quick Answer
The **diamond operator** (`<>`) was introduced in Java 7 to eliminate redundant type arguments on the right-hand side of generic declarations. Instead of repeating `new HashMap<String, List<Integer>>()`, you write `new HashMap<>()` — the compiler infers the type arguments from the left-hand side. Java 9 extended diamond to work with **anonymous inner classes**.

---

## Detailed Explanation

### Before the Diamond Operator (Java 5-6)

```java
// Java 5/6: Type arguments repeated on BOTH sides
List<String> names = new ArrayList<String>();
Map<String, List<Integer>> scores = new HashMap<String, List<Integer>>();
Map<String, Map<Integer, List<Double>>> nested =
    new HashMap<String, Map<Integer, List<Double>>>();  // Very verbose!

// The right-hand side types are completely redundant
// The compiler already knows the types from the left-hand side
```

### Diamond Operator (Java 7+)

```java
// Java 7+: Diamond operator infers from left-hand side
List<String> names = new ArrayList<>();            // ✅ Clean
Map<String, List<Integer>> scores = new HashMap<>(); // ✅ Clean
Map<String, Map<Integer, List<Double>>> nested =
    new HashMap<>();                                 // ✅ Much better

// Works with any generic class constructor
Set<Integer> numbers = new HashSet<>();
Queue<String> queue = new LinkedList<>();
Pair<String, Integer> pair = new Pair<>("age", 30);
```

### Diamond vs Raw Type

```java
// ⚠️ CRITICAL DISTINCTION: Diamond is NOT the same as raw type

// Diamond operator — type-safe, compiler infers generics
List<String> safe = new ArrayList<>();    // ArrayList<String>

// Raw type — NO type safety, generates warnings
List<String> unsafe = new ArrayList();    // ArrayList (RAW) — warning!

// The difference at compile time:
List<String> list1 = new ArrayList<>();   // Inferred as ArrayList<String>
list1.add("hello");   // ✅
// list1.add(42);     // ❌ COMPILE ERROR — type-safe

List<String> list2 = new ArrayList();     // Raw ArrayList — warning
list2.add("hello");   // ✅
// Compiler warning: unchecked call
```

### Diamond with Constructors That Take Arguments

```java
// Diamond works when constructor has arguments that help inference
List<String> copy = new ArrayList<>(existingStringList);  // ✅
Set<Integer> fromList = new HashSet<>(integerList);       // ✅

// Constructor arguments don't change inference — LHS determines type
Map<String, Integer> map = new HashMap<>(16, 0.75f);     // ✅
// Type args inferred from LHS, constructor args are capacity/loadFactor
```

### Diamond with Anonymous Classes (Java 9+)

```java
// Java 7-8: Diamond NOT allowed with anonymous inner classes
// ❌ COMPILE ERROR in Java 7/8
// Comparator<String> comp = new Comparator<>() {
//     public int compare(String a, String b) { return a.compareTo(b); }
// };

// Java 9+: Diamond ALLOWED with anonymous inner classes ✅
Comparator<String> comp = new Comparator<>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
};

// More Java 9+ examples
Iterator<Integer> counter = new Iterator<>() {
    private int current = 0;

    @Override
    public boolean hasNext() { return current < 10; }

    @Override
    public Integer next() { return current++; }
};

// Abstract class with diamond
abstract class Handler<T> {
    abstract void handle(T item);
}

Handler<String> stringHandler = new Handler<>() {   // Java 9+
    @Override
    void handle(String item) {
        System.out.println(item);
    }
};
```

### When Diamond Inference Fails

```java
// ❌ Diamond cannot infer when there's no target type context
// var list = new ArrayList<>();  // Inferred as ArrayList<Object> — probably wrong

// ❌ Diamond fails in complex generic method arguments (pre-Java 8)
// processItems(new ArrayList<>());  // May fail in Java 7

// ✅ Java 8+ improved target-type inference for method args
void processItems(List<String> items) { }
processItems(new ArrayList<>());  // ✅ Works in Java 8+

// ❌ Cannot infer when method is overloaded
void handle(List<String> strings) { }
void handle(List<Integer> integers) { }
// handle(new ArrayList<>());  // ❌ Ambiguous — which overload?

// ✅ Fix: be explicit
handle(new ArrayList<String>());  // ✅ Resolves ambiguity
```

### Diamond with var (Java 10+)

```java
// ⚠️ var + diamond = Object type (usually wrong)
var list = new ArrayList<>();           // ArrayList<Object> — NOT useful
var map = new HashMap<>();              // HashMap<Object, Object> — NOT useful

// ✅ var without diamond — preserves specific type
var list2 = new ArrayList<String>();    // ArrayList<String> ✅
var map2 = new HashMap<String, Integer>(); // HashMap<String, Integer> ✅

// ✅ Or skip var and use diamond with explicit LHS type
List<String> list3 = new ArrayList<>(); // List<String> ✅

// Rule of thumb:
// - Use EITHER var OR diamond, not both
// - var infers from RIGHT side, diamond infers from LEFT side
// - Together, neither side has type info → defaults to Object
```

### Diamond in Generic Method Calls

```java
// Diamond is for constructors only — not for method type arguments

// ✅ Method type inference (no diamond needed)
List<String> list = Collections.emptyList();    // T inferred as String

// ✅ Explicit type witness for methods (not diamond)
List<String> list2 = Collections.<String>emptyList();

// Diamond in constructor within method call
List<String> list3 = new ArrayList<>(List.of("a", "b"));  // ✅ Diamond on constructor
```

### Real-World Usage Patterns

```java
// ✅ Service layer — clean declarations
Map<String, List<UserDTO>> usersByRole = new HashMap<>();
Set<Permission> permissions = new TreeSet<>();
Optional<User> user = Optional.empty();

// ✅ Builder collections
List<Predicate<String>> filters = new ArrayList<>();
filters.add(s -> s.length() > 3);
filters.add(s -> s.startsWith("A"));

// ✅ Concurrent collections
ConcurrentMap<String, AtomicInteger> counters = new ConcurrentHashMap<>();
BlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<>();

// ✅ Multi-level generics — diamond eliminates massive repetition
Map<String, Map<Integer, List<Pair<String, Double>>>> complexData = new HashMap<>();
// Without diamond: new HashMap<String, Map<Integer, List<Pair<String, Double>>>>()
```

---

## Anti-Patterns

```java
// ❌ Using raw type instead of diamond
List<String> names = new ArrayList();     // Raw type — unchecked warning

// ❌ Combining var and diamond
var data = new HashMap<>();               // HashMap<Object, Object>

// ❌ Specifying types on both sides (pre-Java 7 style)
List<String> names = new ArrayList<String>();  // Redundant in Java 7+

// ❌ Over-relying on diamond in ambiguous contexts
// process(new ArrayList<>());  // May fail if overloaded
```

## Best Practices

```java
// ✅ Always use diamond operator for constructor calls
List<String> names = new ArrayList<>();
Map<String, Integer> ages = new HashMap<>();

// ✅ Be explicit when diamond cannot infer correctly
var names = new ArrayList<String>();     // Explicit type arg with var

// ✅ Use diamond with Java 9+ anonymous classes
Comparator<String> comp = new Comparator<>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// ✅ Prefer lambdas over anonymous classes when possible
Comparator<String> comp2 = (a, b) -> a.length() - b.length();  // Even cleaner

// ✅ Choose one: var for RHS inference OR diamond for LHS inference
List<String> a = new ArrayList<>();      // Diamond from LHS ✅
var b = new ArrayList<String>();         // var from RHS ✅
// var c = new ArrayList<>();            // Neither side — bad ❌
```

---

## Interview Tips
- Diamond (`<>`) infers type arguments from the **assignment target** (left-hand side)
- Diamond is **not** a raw type — it preserves full type safety
- Java 9 extended diamond to anonymous inner classes
- `var` + `<>` together defaults to `Object` — avoid this combination
- Diamond only applies to **constructors**, not method calls
- For methods, Java uses **type inference** automatically (no special syntax needed)

---

[← Q8: Type Inference](./q08-type-inference.md) | [Next: Q10 - Bridge Methods →](./q10-bridge-methods.md)
