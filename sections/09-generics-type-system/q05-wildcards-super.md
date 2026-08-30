# Q5: Explain Lower-Bounded Wildcards (? super)

[Home](../../README.md) > [Generics & Type System](./README.md) > Lower-Bounded Wildcards

---

## 🟡 Difficulty: Intermediate

## Question
> What are lower-bounded wildcards (`? super`) in Java generics? When should you use them?

---

## Quick Answer
`? super T` is a **lower-bounded wildcard** that matches T or any supertype of T. A `List<? super Integer>` can reference a `List<Integer>`, `List<Number>`, or `List<Object>`. You can safely **write** (add) Integer values to it, but when you **read**, you only get `Object` back because the compiler doesn't know the exact supertype. This makes `? super` ideal for **consumer** parameters — destinations that receive data.

---

## Detailed Explanation

### Basic Syntax and Usage

```java
// Accepts List<Integer>, List<Number>, List<Object>
public static void addIntegers(List<? super Integer> list) {
    list.add(1);      // ✅ Safe — Integer fits in any of the above
    list.add(2);      // ✅
    list.add(3);      // ✅

    // Reading returns Object — compiler doesn't know the exact type
    Object item = list.get(0);   // ✅ But you only get Object
    // Integer i = list.get(0);  // ❌ COMPILE ERROR
}

// Usage
List<Integer> ints = new ArrayList<>();
List<Number> nums = new ArrayList<>();
List<Object> objs = new ArrayList<>();

addIntegers(ints);  // ✅ Integer is Integer
addIntegers(nums);  // ✅ Number is a supertype of Integer
addIntegers(objs);  // ✅ Object is a supertype of Integer
```

### Why Writing Is Safe

```java
List<? super Integer> list;

// Scenario 1: actual type is List<Integer>
list = new ArrayList<Integer>();
list.add(42);   // ✅ Integer → List<Integer>

// Scenario 2: actual type is List<Number>
list = new ArrayList<Number>();
list.add(42);   // ✅ Integer → List<Number> (Integer IS-A Number)

// Scenario 3: actual type is List<Object>
list = new ArrayList<Object>();
list.add(42);   // ✅ Integer → List<Object> (Integer IS-A Object)

// In ALL cases, adding an Integer is safe because Integer
// is a subtype of Integer, Number, and Object
```

### Why Reading Returns Object

```java
List<? super Integer> list;

// Could be List<Integer> → elements are Integer
// Could be List<Number>  → elements are Number
// Could be List<Object>  → elements are Object

// The only SAFE common type is Object
Object item = list.get(0);    // ✅ Always safe

// Cannot assume anything more specific
// Integer i = list.get(0);   // ❌ Might be Number or Object
// Number n = list.get(0);    // ❌ Might be Object
```

### Visual Explanation

```
List<? super Integer> can be:
  ┌──────────────────┐
  │ List<Integer>    │ ── can add Integer ✅  read as Object
  │ List<Number>     │ ── can add Integer ✅  read as Object
  │ List<Object>     │ ── can add Integer ✅  read as Object
  └──────────────────┘

  Writing Integer is SAFE:
  - Integer → Integer container ✅ (exact match)
  - Integer → Number container  ✅ (Integer IS-A Number)
  - Integer → Object container  ✅ (Integer IS-A Object)
```

### Common Use Case: Comparable<? super T>

```java
// ❌ Too restrictive — won't work with inherited Comparable
public static <T extends Comparable<T>> T max(Collection<T> items) {
    return Collections.max(items);
}

// ✅ Flexible — works when parent class implements Comparable
public static <T extends Comparable<? super T>> T max(Collection<T> items) {
    return Collections.max(items);
}

// Why this matters:
class Animal implements Comparable<Animal> {
    String name;
    @Override
    public int compareTo(Animal other) {
        return name.compareTo(other.name);
    }
}

class Dog extends Animal { }

// Dog does NOT implement Comparable<Dog>
// Dog inherits Comparable<Animal> from Animal
// So Dog extends Comparable<? super Dog> ✅
// But Dog does NOT extend Comparable<Dog> ❌

List<Dog> dogs = List.of(new Dog(), new Dog());
Dog maxDog = max(dogs);  // ✅ Works with Comparable<? super T>
```

### Use Case: Consumer/Destination Parameters

```java
// Method that copies elements into a destination
public static <T> void fill(List<? super T> dest, T value, int count) {
    for (int i = 0; i < count; i++) {
        dest.add(value);  // ✅ Writing T into a ? super T container
    }
}

// Usage
List<Number> numbers = new ArrayList<>();
fill(numbers, 42, 5);       // ✅ Integer fits in List<Number>
fill(numbers, 3.14, 5);     // ✅ Double fits in List<Number>

List<Object> objects = new ArrayList<>();
fill(objects, "hello", 3);   // ✅ String fits in List<Object>
```

### Use Case: Predicate and Consumer

```java
// Java's Predicate uses ? super for flexibility
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);

    // Note the ? super T parameter type
    default Predicate<T> and(Predicate<? super T> other) {
        return (t) -> test(t) && other.test(t);
    }
}

// A Predicate<Object> can test any type
Predicate<Object> notNull = obj -> obj != null;
Predicate<String> notEmpty = str -> !str.isEmpty();

// Composing: Predicate<? super String> accepts Predicate<Object>
Predicate<String> validString = notEmpty.and(notNull);  // ✅
```

### Use Case: Event Handlers

```java
// Event handler using ? super for flexibility
public class EventBus {
    private Map<Class<?>, List<Consumer<? super Object>>> handlers = new HashMap<>();

    @SuppressWarnings("unchecked")
    public <T> void subscribe(Class<T> eventType, Consumer<? super T> handler) {
        handlers.computeIfAbsent(eventType, k -> new ArrayList<>())
                .add((Consumer<? super Object>) handler);
    }

    // A Consumer<Object> can handle any event type
    // A Consumer<Event> can handle Event and its subtypes
}
```

### Comparison: ? extends vs ? super

```java
// ? extends → READ (produce values)
public static double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number n : numbers) {
        total += n.doubleValue();  // ✅ Read as Number
    }
    // numbers.add(42);           // ❌ Cannot write
    return total;
}

// ? super → WRITE (consume values)
public static void addDefaults(List<? super Integer> list) {
    list.add(0);                  // ✅ Write Integer
    list.add(1);
    Object obj = list.get(0);     // Only get Object back
}

// Combined: Collections.copy() uses both
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {          // Read from src (extends)
        dest.add(item);           // Write to dest (super)
    }
}
```

---

## Anti-Patterns

```java
// ❌ Using ? super when you need to read specific types
public static Number getFirst(List<? super Number> list) {
    // return list.get(0);  // ❌ Returns Object, not Number
    return (Number) list.get(0);  // Unsafe cast — could be anything
}

// ❌ Using ? super when ? extends is appropriate
public static void printNumbers(List<? super Number> list) {
    // Wrong — you want to READ from the list → use ? extends
    for (Object item : list) {
        System.out.println(item);  // Can only use Object methods
    }
}

// ❌ Confusing extends and super
List<? super Number> list = new ArrayList<Object>();
// list.add(new Object());  // ❌ COMPILE ERROR — can only add Number or subtypes
list.add(42);               // ✅ Integer extends Number
list.add(3.14);             // ✅ Double extends Number
```

## Best Practices

```java
// ✅ Use ? super for consumer/destination parameters
public static <T> void populateList(List<? super T> dest, T[] source) {
    for (T item : source) {
        dest.add(item);
    }
}

// ✅ Use Comparable<? super T> for maximum flexibility
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    Collections.sort(list);
}

// ✅ Remember: "super" means "writes" (PECS — Consumer Super)
// If you only WRITE to a collection → use ? super

// ✅ Use both together for transfer operations
public static <T> void transfer(
        List<? extends T> source,    // Read from here
        List<? super T> destination  // Write to here
) {
    for (T item : source) {
        destination.add(item);
    }
}
```

---

## Interview Tips
- `? super T` makes the structure a **consumer** — you can write T (or subtypes) into it
- Reading from `? super T` always returns `Object`
- `Comparable<? super T>` is essential for classes that inherit their Comparable from a parent
- You can add T and any **subtype** of T to a `List<? super T>`
- The mnemonic: **extends = read, super = write** (formalized as PECS in Q6)

---

[← Q4: Upper-Bounded Wildcards](./q04-wildcards-extends.md) | [Next: Q6 - PECS Principle →](./q06-pecs-principle.md)
