# Q6: What Is the PECS Principle?

[Home](../../README.md) > [Generics & Type System](./README.md) > PECS Principle

---

## 🟠 Difficulty: Advanced

## Question
> What is the PECS principle? How does it guide wildcard usage in API design?

---

## Quick Answer
**PECS** stands for **Producer Extends, Consumer Super** (from Joshua Bloch's *Effective Java*). If a parameterized type **produces** values (you read from it), use `? extends T`. If it **consumes** values (you write to it), use `? super T`. If it does both, use an exact type `T` (no wildcard). This principle maximizes API flexibility while maintaining type safety.

---

## Detailed Explanation

### The Core Rule

```
┌─────────────────────────────────────────────────────┐
│  PECS: Producer Extends, Consumer Super             │
│                                                     │
│  Producer (READ from it)  →  ? extends T            │
│  Consumer (WRITE to it)   →  ? super T              │
│  Both (READ + WRITE)      →  T (no wildcard)        │
└─────────────────────────────────────────────────────┘
```

### Collections.copy() — The Classic Example

```java
// Java's Collections.copy() signature:
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        T item = src.get(i);    // src PRODUCES values → ? extends T
        dest.set(i, item);      // dest CONSUMES values → ? super T
    }
}

// Usage — maximum flexibility:
List<Number> numbers = new ArrayList<>(Arrays.asList(0, 0, 0));
List<Integer> integers = List.of(1, 2, 3);

Collections.copy(numbers, integers);
// dest: List<Number>  → ? super Integer  ✅ (Number is super of Integer)
// src:  List<Integer>  → ? extends Integer ✅
```

### Applying PECS: Stack Example

```java
public class Stack<E> {
    private List<E> elements = new ArrayList<>();

    public void push(E element) {
        elements.add(element);
    }

    public E pop() {
        if (elements.isEmpty()) throw new EmptyStackException();
        return elements.remove(elements.size() - 1);
    }

    // PRODUCER: src produces E values → use ? extends E
    public void pushAll(Iterable<? extends E> src) {
        for (E item : src) {
            push(item);
        }
    }

    // CONSUMER: dest consumes E values → use ? super E
    public void popAll(Collection<? super E> dest) {
        while (!elements.isEmpty()) {
            dest.add(pop());
        }
    }
}

// Usage
Stack<Number> stack = new Stack<>();

// pushAll: Iterable<? extends Number> — accepts Integer, Double, etc.
List<Integer> ints = List.of(1, 2, 3);
stack.pushAll(ints);          // ✅ Integer extends Number

List<Double> doubles = List.of(1.1, 2.2);
stack.pushAll(doubles);       // ✅ Double extends Number

// popAll: Collection<? super Number> — accepts Number, Object
List<Object> everything = new ArrayList<>();
stack.popAll(everything);     // ✅ Object is super of Number
```

### GET-PUT Principle (Alternative Name)

```java
// Same concept, different naming:
// GET from a structure  → ? extends T  (Producer)
// PUT into a structure  → ? super T    (Consumer)

public static <T> void transfer(
    Collection<? extends T> source,     // GET from source → extends
    Collection<? super T> destination   // PUT into destination → super
) {
    for (T item : source) {
        destination.add(item);
    }
}
```

### PECS in Comparator Design

```java
// Comparator<? super T> — the comparator CONSUMES T values to compare
public static <T> T max(Collection<? extends T> coll,
                         Comparator<? super T> comp) {
    Iterator<? extends T> it = coll.iterator();
    T max = it.next();
    while (it.hasNext()) {
        T item = it.next();
        if (comp.compare(item, max) > 0) {  // comp CONSUMES T values
            max = item;
        }
    }
    return max;
}

// Usage
class Animal { String name; }
class Dog extends Animal { }

Comparator<Animal> byName = Comparator.comparing(a -> a.name);
List<Dog> dogs = List.of(/* ... */);

// coll: List<Dog>        → ? extends Animal ✅ (Dog extends Animal)
// comp: Comparator<Animal> → ? super Dog   ✅ (Animal is super of Dog)
Animal topDog = max(dogs, byName);
```

### PECS in Stream API

```java
// Stream.forEach — Consumer<? super T>
List<Integer> numbers = List.of(1, 2, 3);

// Consumer<Object> works for Integer (Object is super of Integer)
Consumer<Object> printer = System.out::println;
numbers.forEach(printer);  // ✅ PECS — Consumer Super

// Stream.map — Function<? super T, ? extends R>
Function<Object, String> toString = Object::toString;
List<String> strings = numbers.stream()
    .map(toString)    // ✅ ? super Integer accepts Function<Object, ...>
    .collect(Collectors.toList());
```

### PECS in Real-World API Design

```java
// ✅ Well-designed API following PECS
public class DataProcessor<T> {

    // Source PRODUCES data → ? extends T
    public void loadFrom(Collection<? extends T> source) {
        for (T item : source) {
            process(item);
        }
    }

    // Destination CONSUMES results → ? super T
    public void saveTo(Collection<? super T> destination) {
        for (T result : getResults()) {
            destination.add(result);
        }
    }

    // Transformer: reads from source (extends), writes to dest (super)
    public void transform(
            List<? extends T> source,
            List<? super T> destination,
            UnaryOperator<T> transformer
    ) {
        for (T item : source) {
            destination.add(transformer.apply(item));
        }
    }

    private void process(T item) { /* ... */ }
    private List<T> getResults() { return List.of(); }
}
```

### When NOT to Use Wildcards

```java
// Return types should NOT use wildcards — forces callers to deal with them
// ❌ Bad — caller gets List<? extends Number>, hard to work with
public List<? extends Number> getNumbers() {
    return List.of(1, 2, 3);
}

// ✅ Good — caller gets clean type
public List<Number> getNumbers() {
    return List.of(1, 2, 3);
}

// When a parameter is both read and written → no wildcard
// ❌ Cannot use wildcard — need to read AND write
public static <T> void swap(List<T> list, int i, int j) {
    T temp = list.get(i);    // READ
    list.set(i, list.get(j)); // WRITE
    list.set(j, temp);        // WRITE
}
```

### Decision Flowchart

```
Parameter uses T?
  │
  ├── Only READS T values (get, iterate, stream)
  │     → Use ? extends T  (Producer)
  │
  ├── Only WRITES T values (add, set, accept)
  │     → Use ? super T  (Consumer)
  │
  ├── Both READS and WRITES
  │     → Use T  (no wildcard)
  │
  └── Neither reads nor writes specific T
        → Use ?  (unbounded wildcard)
```

### Complete Example: Merge Sort with PECS

```java
public class SortUtils {

    public static <T> void mergeSort(
            List<? extends T> input,             // PRODUCER: read from
            List<? super T> output,              // CONSUMER: write to
            Comparator<? super T> comparator     // CONSUMER: accepts T to compare
    ) {
        List<T> sorted = new ArrayList<>(input);  // Read from producer
        sorted.sort(comparator);                  // Comparator consumes T
        output.addAll(sorted);                    // Write to consumer
    }
}

// Usage
List<Integer> source = List.of(3, 1, 4, 1, 5);
List<Number> destination = new ArrayList<>();
Comparator<Number> byValue = Comparator.comparingDouble(Number::doubleValue);

SortUtils.mergeSort(source, destination, byValue);  // ✅ All PECS-compliant
```

---

## Anti-Patterns

```java
// ❌ Using extends when the parameter is a consumer
public static void addAll(List<? extends Number> dest, Number... values) {
    for (Number v : values) {
        // dest.add(v);  // COMPILE ERROR — extends = read-only
    }
}

// ❌ Using super when the parameter is a producer
public static double sum(List<? super Number> numbers) {
    double total = 0;
    for (Object n : numbers) {  // Only get Object, not Number
        total += ((Number) n).doubleValue();  // Unsafe cast
    }
    return total;
}

// ❌ Wildcard in return type — burdens the caller
public List<? extends Number> getValues() { ... }
```

## Best Practices

```java
// ✅ Apply PECS consistently to all parameters
public static <T> boolean addAllSafely(
        Collection<? super T> dest,       // Consumer → super
        Collection<? extends T> source    // Producer → extends
) {
    return dest.addAll(source);
}

// ✅ Use Comparable<? super T> for sorting APIs
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    Collections.sort(list);
}

// ✅ No wildcard on return types
public <T> List<T> filter(List<? extends T> input, Predicate<? super T> test) {
    List<T> result = new ArrayList<>();
    for (T item : input) {
        if (test.test(item)) result.add(item);
    }
    return result;  // Clean return type
}

// ✅ Mnemonic: "PECS from the collection's perspective"
// The COLLECTION produces or consumes, not the caller
```

---

## Interview Tips
- PECS = **P**roducer **E**xtends, **C**onsumer **S**uper
- The perspective is from the **collection**, not the method
- Return types should **never** use wildcards
- `Collections.copy(dest, src)` is the textbook PECS example
- `Comparator<? super T>` and `Consumer<? super T>` are consumer patterns
- When in doubt: if you only **get** values, use `extends`; if you only **put** values, use `super`

---

[← Q5: Lower-Bounded Wildcards](./q05-wildcards-super.md) | [Next: Q7 - Generic Constraints →](./q07-generic-constraints.md)
