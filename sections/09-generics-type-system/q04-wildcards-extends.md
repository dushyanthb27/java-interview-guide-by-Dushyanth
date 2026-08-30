# Q4: Explain Upper-Bounded Wildcards (? extends)

[Home](../../README.md) > [Generics & Type System](./README.md) > Upper-Bounded Wildcards

---

## 🟡 Difficulty: Intermediate

## Question
> What are upper-bounded wildcards (`? extends`) in Java generics? Why can't you add elements to a `List<? extends Number>`?

---

## Quick Answer
`? extends T` is an **upper-bounded wildcard** that matches T or any subtype of T. A `List<? extends Number>` can hold a `List<Integer>`, `List<Double>`, or `List<Number>`. You can **read** elements as Number, but you **cannot add** elements (except `null`) because the compiler doesn't know the exact subtype the list holds — adding an Integer to what might be a `List<Double>` would be type-unsafe.

---

## Detailed Explanation

### Basic Syntax and Usage

```java
// Accepts List<Number>, List<Integer>, List<Double>, etc.
public static double sumAll(List<? extends Number> numbers) {
    double total = 0;
    for (Number num : numbers) {      // ✅ Read as Number
        total += num.doubleValue();
    }
    return total;
}

// Usage
List<Integer> ints = List.of(1, 2, 3);
List<Double> doubles = List.of(1.1, 2.2, 3.3);
List<Number> nums = List.of(1, 2.5, 3L);

double sum1 = sumAll(ints);     // ✅ Works — Integer extends Number
double sum2 = sumAll(doubles);  // ✅ Works — Double extends Number
double sum3 = sumAll(nums);     // ✅ Works — Number extends Number (itself)
```

### Why You Cannot Add Elements

```java
List<? extends Number> list;

// Scenario 1: list could reference a List<Integer>
list = new ArrayList<Integer>();

// Scenario 2: list could reference a List<Double>
list = new ArrayList<Double>();

// The compiler doesn't know which, so it BLOCKS all adds:
// list.add(42);       // ❌ COMPILE ERROR — might be List<Double>
// list.add(3.14);     // ❌ COMPILE ERROR — might be List<Integer>
// list.add(new Number(){}); // ❌ COMPILE ERROR

list.add(null);        // ✅ null is the only safe value (matches any type)
```

### Visual Explanation

```
List<? extends Number> can be:
  ┌──────────────────┐
  │ List<Number>     │ ── read elements as Number ✅
  │ List<Integer>    │ ── read elements as Number ✅
  │ List<Double>     │ ── read elements as Number ✅
  │ List<Float>      │ ── read elements as Number ✅
  │ List<Long>       │ ── read elements as Number ✅
  └──────────────────┘

  Writing ANY specific subtype is UNSAFE:
  - Add Integer? → Breaks if it's List<Double>
  - Add Double?  → Breaks if it's List<Integer>
  - Conclusion:  → Cannot add anything (except null)
```

### Reading Is Always Safe

```java
public static void printNumbers(List<? extends Number> numbers) {
    // ✅ Reading always returns Number (the upper bound)
    Number first = numbers.get(0);
    System.out.println(first.doubleValue());

    // ✅ Iterating is safe
    for (Number n : numbers) {
        System.out.println(n);
    }

    // ✅ Stream operations work
    double avg = numbers.stream()
        .mapToDouble(Number::doubleValue)
        .average()
        .orElse(0);
}
```

### Unbounded Wildcard (?)

```java
// Unbounded wildcard — equivalent to ? extends Object
public static void printList(List<?> list) {
    for (Object item : list) {     // Read as Object
        System.out.println(item);
    }
    // list.add("hello");  // ❌ Cannot add — same restriction
    list.add(null);        // ✅ Only null allowed
}

// Accepts ANY List — List<String>, List<Integer>, List<Object>, etc.
printList(List.of(1, 2, 3));
printList(List.of("a", "b"));
printList(List.of());
```

### Comparison: Wildcard vs Type Parameter

```java
// These two are similar but not identical:

// Option 1: Wildcard — more flexible, no name to reference
public static double sum(List<? extends Number> numbers) { ... }

// Option 2: Type parameter — can reference T elsewhere
public static <T extends Number> double sum(List<T> numbers) { ... }

// When to prefer type parameter:
// - Need to return the same type
public static <T extends Number> T firstOrDefault(List<T> list, T defaultVal) {
    return list.isEmpty() ? defaultVal : list.get(0);
}

// - Need to relate multiple parameters
public static <T extends Number> void copy(List<T> src, List<T> dest) {
    dest.addAll(src);
}

// When to prefer wildcard:
// - Simple read-only access, no need to name the type
public static void printAll(List<? extends Number> nums) {
    nums.forEach(System.out::println);
}
```

### Upper-Bounded Wildcards with Interfaces

```java
// Works with interface bounds too
public static <T> void sortList(List<? extends Comparable<T>> list) {
    // Elements are Comparable — can compare them
}

// Common pattern in collections API
public static double average(Collection<? extends Number> numbers) {
    if (numbers.isEmpty()) return 0;
    double sum = 0;
    for (Number n : numbers) {
        sum += n.doubleValue();
    }
    return sum / numbers.size();
}

// Works with Set, Queue, any Collection subtype
Set<Integer> intSet = Set.of(1, 2, 3);
Queue<Double> doubleQueue = new LinkedList<>(List.of(1.0, 2.0));
System.out.println(average(intSet));       // ✅
System.out.println(average(doubleQueue));  // ✅
```

### Nested Wildcards

```java
// List of lists of anything extending Number
public static void processMatrix(List<? extends List<? extends Number>> matrix) {
    for (List<? extends Number> row : matrix) {
        for (Number value : row) {
            System.out.print(value + " ");
        }
        System.out.println();
    }
}

List<List<Integer>> intMatrix = List.of(
    List.of(1, 2, 3),
    List.of(4, 5, 6)
);
processMatrix(intMatrix);  // ✅
```

### Upper-Bounded Wildcards with Generic Classes

```java
public class Container<T> {
    private T value;

    public Container(T value) { this.value = value; }
    public T getValue() { return value; }
}

// Method accepting Container of any Number subtype
public static void processContainer(Container<? extends Number> container) {
    Number num = container.getValue();   // ✅ Safe read
    System.out.println(num.doubleValue());
    // container.setValue(42);  // ❌ Cannot write
}

Container<Integer> intContainer = new Container<>(42);
Container<Double> dblContainer = new Container<>(3.14);
processContainer(intContainer);  // ✅
processContainer(dblContainer);  // ✅
```

---

## Anti-Patterns

```java
// ❌ Trying to add elements to ? extends
public static void addNumber(List<? extends Number> list) {
    // list.add(42);          // COMPILE ERROR
    // list.add(0);           // COMPILE ERROR
}

// ❌ Using concrete type when wildcard suffices
public static void printNums(List<Integer> nums) { }   // Too restrictive
// Cannot pass List<Double> or List<Number>

// ❌ Over-nesting wildcards unnecessarily
Map<? extends String, ? extends List<? extends Number>> map;  // Hard to read

// ❌ Confusing ? extends with raw types
List rawList = new ArrayList();     // ❌ Loses ALL type safety
List<?> wildcard = new ArrayList<>();  // ✅ Type-safe, read-only
```

## Best Practices

```java
// ✅ Use ? extends for read-only (producer) parameters
public static double sum(List<? extends Number> numbers) {
    return numbers.stream().mapToDouble(Number::doubleValue).sum();
}

// ✅ Prefer wildcard over type parameter for simple consumption
// Good: cleaner API
void process(List<? extends Foo> foos) { }

// Less ideal: type parameter not used elsewhere
<T extends Foo> void process(List<T> foos) { }

// ✅ Use Collections.unmodifiableList with ? extends
List<? extends Number> readOnly = Collections.unmodifiableList(integers);

// ✅ Remember: "extends" means "reads" (PECS — Producer Extends)
// If you only READ from a collection → use ? extends
```

---

## Interview Tips
- `? extends T` makes the structure a **producer** — you can only read from it
- The compiler blocks writes because it cannot guarantee type safety
- `null` is the only value that can be added to `? extends` collections
- Unbounded wildcard `<?>` is shorthand for `<? extends Object>`
- Use type parameters instead of wildcards when you need to reference the type

---

[← Q3: Bounded Type Parameters](./q03-bounded-type-parameters.md) | [Next: Q5 - Lower-Bounded Wildcards →](./q05-wildcards-super.md)
