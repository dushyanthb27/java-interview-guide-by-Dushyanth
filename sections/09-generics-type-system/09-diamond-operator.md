# Q9: What Is the Diamond Operator?

[Home](../../README.md) > [Generics & Type System](./README.md) > Diamond Operator

---

## 🟢 Difficulty: Basic

## Question

> What is the diamond operator (`<>`) in Java? How does it differ from raw types, and how has its support evolved for anonymous inner classes and local variables?

---

## Quick Answer

The **diamond operator** (`<>`) was introduced in Java 7 to eliminate redundant generic type specifications on constructor invocations. Instead of repeating `new HashMap<String, List<Integer>>()`, you write `new HashMap<>()` — the compiler infers the exact type arguments from the assignment target (left-hand side). Java 9 extended the diamond operator to work with **anonymous inner classes** (provided the inferred type is denotable). It is distinct from raw types because it fully preserves compile-time type safety.

---

## Detailed Explanation

### Before the Diamond Operator (Java 5-6)

```java
// Java 5/6: Type arguments repeated on BOTH sides
List<String> names = new ArrayList<String>();
Map<String, List<Integer>> scores = new HashMap<String, List<Integer>>();
Map<String, Map<Integer, List<Double>>> nested =
    new HashMap<String, Map<Integer, List<Double>>>();  // Highly verbose!
```

### Diamond Operator (Java 7+)

```java
// Java 7+: Diamond operator infers from left-hand side assignment context
List<String> names = new ArrayList<>();            // ✅ Clean
Map<String, List<Integer>> scores = new HashMap<>(); // ✅ Clean
Map<String, Map<Integer, List<Double>>> nested =
    new HashMap<>();                                 // ✅ Deep nesting simplified
```

### Diamond vs Raw Type

```java
// ⚠️ CRITICAL DISTINCTION: Diamond is NOT a raw type

// Diamond operator — fully type-safe, compiler infers generic arguments
List<String> safe = new ArrayList<>();    // Instantiates ArrayList<String>

// Raw type — NO type safety, generates unchecked compiler warnings
List<String> unsafe = new ArrayList();    // Instantiates raw ArrayList (erased to Object)

// Compile-time enforcement:
List<String> list1 = new ArrayList<>();   
list1.add("hello");   // ✅
// list1.add(42);     // ❌ COMPILE ERROR — type-safe

List<String> list2 = new ArrayList();     
list2.add("hello");   // ✅
list2.add((Object) 42); // ⚠️ Compiles with raw warning, corrupts type safety!
```

### Diamond with Anonymous Classes (Java 9+)

```java
// Java 7-8: Diamond NOT allowed with anonymous inner classes
// ❌ COMPILE ERROR in Java 7/8
// Comparator<String> comp = new Comparator<>() { ... };

// Java 9+: Diamond ALLOWED with anonymous inner classes if denotable
Comparator<String> comp = new Comparator<>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
};

// Abstract class handler with diamond
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
// ❌ Diamond cannot infer when there is no assignment target type context
// var list = new ArrayList<>();  // Inferred as ArrayList<Object> ⚠️

// ❌ Cannot infer when method is overloaded and target type is ambiguous
void handle(List<String> strings) { }
void handle(List<Integer> integers) { }
// handle(new ArrayList<>());  // ❌ Ambiguous overload compilation error

// ✅ Fix: be explicit
handle(new ArrayList<String>());  // ✅ Resolves overload ambiguity
```

### Diamond with `var` (Java 10+)

```java
// ⚠️ COMBINING TRAP: var + diamond = ArrayList<Object>
var list = new ArrayList<>();           // ArrayList<Object> — type safety lost!

// ✅ Correct usage: choose ONE inference direction
var list2 = new ArrayList<String>();    // RHS inference (`var` from RHS) ✅
List<String> list3 = new ArrayList<>(); // LHS inference (`<>` from LHS) ✅
```

---

## Gotchas

- **Anonymous Class Non-Denotable Types**: Java 9 permits diamond with anonymous classes *only* when the inferred type is denotable. If the compiler infers an intersection type or non-denotable anonymous type for the constructor, compilation fails.
- **Overload Ambiguity Trap**: Passing `new ArrayList<>()` directly into overloaded methods without casting or explicit type witnesses often results in ambiguous method resolution errors.
- **Raw Type Fallback Silencing**: Mixing raw types and diamond operator declarations across legacy module boundaries can suppress compiler warnings while introducing runtime heap pollution.
- **IDE Refactoring Pitfalls**: Automated refactoring tools converting explicit generic instantiations to diamond operators can occasionally break method type inference when target typing is subtle.

---

## Anti-Patterns

```java
// ❌ Using raw types instead of diamond operator
List<String> names = new ArrayList();     // Unchecked warning and raw type risk

// ❌ Combining var and diamond operator
var data = new HashMap<>();               // HashMap<Object, Object> — silent type degradation

// ❌ Redundant explicit type arguments on both sides (pre-Java 7 style)
List<String> names = new ArrayList<String>();  // Unnecessary verbosity in Java 7+
```

## Best Practices

```java
// ✅ Always use diamond operator for standard constructor calls
List<String> names = new ArrayList<>();
Map<String, Integer> ages = new HashMap<>();

// ✅ Be explicit when diamond inference lacks context
var names = new ArrayList<String>();     // Explicit RHS with var

// ✅ Use diamond with Java 9+ anonymous classes for concise custom implementations
Comparator<String> comp = new Comparator<>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};
```

## Interview Tips

- Diamond (`<>`) infers type arguments from the **left-hand side** assignment context.
- Diamond is **not** a raw type; it enforces full compile-time generic type checks.
- Java 9 enabled diamond usage with anonymous inner classes (a frequent interview trivia point).
- Combining `var` and `<>` defaults to `Object`, which is a common interview trap.

---

[← Q8: Type Inference](./q08-type-inference.md) | [Next: Q10 - Bridge Methods →](./q10-bridge-methods.md)
