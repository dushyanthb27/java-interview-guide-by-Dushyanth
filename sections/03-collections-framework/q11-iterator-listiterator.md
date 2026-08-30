# Q11: Iterator vs ListIterator

[Back to Collections Framework](./README.md) | **Difficulty**: 🟢 Easy

---

## Question
> What is the difference between Iterator and ListIterator? When do you use each?

---

## Answer

### Iterator - Forward-Only Traversal

Iterator provides a universal way to traverse **any Collection** in one direction.

```java
public interface Iterator<E> {
    boolean hasNext();       // Check if more elements exist
    E next();                // Get next element and advance cursor
    default void remove();   // Remove last returned element (optional)

    // Java 8+
    default void forEachRemaining(Consumer<? super E> action);
}

// Works with ANY Collection
List<String> list = List.of("A", "B", "C");
Set<String> set = Set.of("X", "Y", "Z");
Queue<String> queue = new ArrayDeque<>(List.of("1", "2", "3"));

Iterator<String> listIt = list.iterator();
Iterator<String> setIt = set.iterator();
Iterator<String> queueIt = queue.iterator();

// Basic traversal
while (listIt.hasNext()) {
    String element = listIt.next();
    System.out.println(element);
}
```

### ListIterator - Bidirectional Traversal for Lists

ListIterator extends Iterator and works **only with Lists**. It supports bidirectional traversal, element modification, and index tracking.

```java
public interface ListIterator<E> extends Iterator<E> {
    // Forward traversal (inherited)
    boolean hasNext();
    E next();
    int nextIndex();

    // Backward traversal (new!)
    boolean hasPrevious();
    E previous();
    int previousIndex();

    // Modification operations
    void remove();     // Remove last returned element
    void set(E e);     // Replace last returned element
    void add(E e);     // Insert element before cursor position
}

// Only available from List
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D"));
ListIterator<String> lit = list.listIterator();      // Start at index 0
ListIterator<String> lit2 = list.listIterator(2);    // Start at index 2
```

### Side-by-Side Comparison

```java
List<String> items = new ArrayList<>(List.of("A", "B", "C", "D", "E"));

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// ITERATOR - Forward only
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Iterator<String> it = items.iterator();

it.hasNext();   // true
it.next();      // "A" - cursor moves forward
it.next();      // "B"
// it.previous(); // ❌ COMPILE ERROR - Iterator has no previous()
it.remove();    // Removes "B" (last returned by next())
// it.set("X");  // ❌ COMPILE ERROR - Iterator has no set()
// it.add("X");  // ❌ COMPILE ERROR - Iterator has no add()

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// LIST ITERATOR - Bidirectional + Modification
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ListIterator<String> lit = items.listIterator();

lit.hasNext();      // true
lit.nextIndex();    // 0
lit.next();         // "A" - cursor between ^ and "C"
lit.next();         // "C" (B was removed earlier)

lit.hasPrevious();  // true
lit.previousIndex(); // 0
lit.previous();     // "C" - cursor moves backward

lit.set("X");       // Replaces "C" with "X"
lit.add("NEW");     // Inserts "NEW" before cursor position
```

### Cursor Position Visualization

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D"));

// ListIterator cursor sits BETWEEN elements:
//  Cursor positions:  |  A  |  B  |  C  |  D  |
//  Index:             0     1     2     3     4
//                    ↑ listIterator()       ↑ listIterator(4)

ListIterator<String> lit = list.listIterator();
// Cursor: | A | B | C | D |
//         ^

lit.next();     // returns "A", cursor moves right
// Cursor: | A | B | C | D |
//             ^

lit.next();     // returns "B"
// Cursor: | A | B | C | D |
//                 ^

lit.previous(); // returns "B", cursor moves left
// Cursor: | A | B | C | D |
//             ^

lit.add("X");   // inserts "X" at cursor position
// Cursor: | A | X | B | C | D |
//                 ^

lit.nextIndex();     // 2
lit.previousIndex(); // 1
```

### Practical Use Cases

```java
// ✅ Use Case 1: Remove elements during iteration
List<String> names = new ArrayList<>(List.of("Alice", "Bob", "Charlie", "David"));
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("C")) {
        it.remove();  // Safe removal during iteration
    }
}
// names = [Alice, Bob, David]

// ✅ Use Case 2: Reverse iteration with ListIterator
List<String> list = List.of("A", "B", "C", "D");
ListIterator<String> lit = list.listIterator(list.size());  // Start at end
while (lit.hasPrevious()) {
    System.out.println(lit.previous());
}
// Output: D, C, B, A

// ✅ Use Case 3: Replace elements during iteration
List<String> items = new ArrayList<>(List.of("hello", "world", "java"));
ListIterator<String> lit2 = items.listIterator();
while (lit2.hasNext()) {
    String current = lit2.next();
    lit2.set(current.toUpperCase());  // Replace with uppercase
}
// items = [HELLO, WORLD, JAVA]

// ✅ Use Case 4: Insert elements during iteration
List<Integer> nums = new ArrayList<>(List.of(1, 3, 5, 7));
ListIterator<Integer> lit3 = nums.listIterator();
while (lit3.hasNext()) {
    int val = lit3.next();
    lit3.add(val + 1);  // Insert after each element
}
// nums = [1, 2, 3, 4, 5, 6, 7, 8]

// ✅ Use Case 5: forEachRemaining (Java 8+)
List<String> data = List.of("A", "B", "C", "D", "E");
Iterator<String> it2 = data.iterator();
it2.next();  // Skip first
it2.next();  // Skip second
it2.forEachRemaining(System.out::println);  // C, D, E
```

### Spliterator - Parallel Processing (Java 8+)

```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);  // Process one element
    Spliterator<T> trySplit();     // Split for parallel processing
    long estimateSize();           // Estimated remaining elements
    int characteristics();         // ORDERED, DISTINCT, SORTED, SIZED, etc.
}

// Spliterator is used internally by parallel streams
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8);

// Creating a Spliterator
Spliterator<Integer> spliterator = numbers.spliterator();
spliterator.estimateSize();        // 8
spliterator.characteristics();     // ORDERED | SIZED | SUBSIZED

// Splitting for parallel processing
Spliterator<Integer> secondHalf = spliterator.trySplit();
// spliterator now covers: [5, 6, 7, 8]
// secondHalf covers: [1, 2, 3, 4]

// Processing
spliterator.forEachRemaining(System.out::println);  // 5, 6, 7, 8
secondHalf.forEachRemaining(System.out::println);   // 1, 2, 3, 4

// Characteristics
// ORDERED:   Elements have defined encounter order
// DISTINCT:  No duplicate elements (Set)
// SORTED:    Elements are sorted (SortedSet, TreeMap)
// SIZED:     estimateSize() returns exact count
// NONNULL:   No null elements
// IMMUTABLE: Source cannot be modified
// CONCURRENT: Source can be modified concurrently
// SUBSIZED:  All splits are SIZED
```

### Custom Iterator Example

```java
// Implementing a custom iterator for a range of numbers
public class RangeIterator implements Iterator<Integer> {
    private int current;
    private final int end;

    public RangeIterator(int start, int end) {
        this.current = start;
        this.end = end;
    }

    @Override
    public boolean hasNext() {
        return current < end;
    }

    @Override
    public Integer next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        return current++;
    }
}

// Making a class Iterable
public class NumberRange implements Iterable<Integer> {
    private final int start;
    private final int end;

    public NumberRange(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    public Iterator<Integer> iterator() {
        return new RangeIterator(start, end);
    }
}

// Usage
NumberRange range = new NumberRange(1, 6);
for (int n : range) {
    System.out.println(n);  // 1, 2, 3, 4, 5
}
```

### Common Anti-Patterns

```java
// ❌ Calling next() without checking hasNext()
Iterator<String> it = list.iterator();
String item = it.next();  // NoSuchElementException if empty!

// ❌ Calling next() multiple times per loop iteration
while (it.hasNext()) {
    System.out.println(it.next() + " - " + it.next());  // May throw!
}

// ❌ Modifying collection during for-each loop
for (String s : list) {
    list.remove(s);  // ConcurrentModificationException!
}

// ❌ Calling remove() without calling next() first
Iterator<String> it = list.iterator();
it.remove();  // IllegalStateException!

// ❌ Calling remove() twice after a single next()
it.next();
it.remove();
it.remove();  // IllegalStateException!
```

### Best Practices

```java
// ✅ Use enhanced for-each when you don't need modification
for (String item : list) {
    System.out.println(item);
}

// ✅ Use Iterator when you need to remove during traversal
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next())) {
        it.remove();
    }
}

// ✅ Use ListIterator when you need bidirectional traversal or modification
ListIterator<String> lit = list.listIterator();
while (lit.hasNext()) {
    String current = lit.next();
    lit.set(transform(current));
}

// ✅ Use removeIf() instead of manual Iterator removal (Java 8+)
list.removeIf(s -> s.startsWith("temp"));

// ✅ Use forEach() for simple operations (Java 8+)
list.forEach(System.out::println);

// ✅ Use streams for complex processing (Java 8+)
list.stream().filter(s -> s.length() > 3).forEach(System.out::println);
```

---

## Quick Reference

| Feature | Iterator | ListIterator | Spliterator |
|---------|----------|-------------|-------------|
| Direction | Forward only | Bidirectional | Forward + split |
| Works with | Any Collection | List only | Any Collection |
| `remove()` | Yes | Yes | No |
| `set()` | No | Yes | No |
| `add()` | No | Yes | No |
| Index access | No | Yes | No |
| Parallel support | No | No | Yes |
| Java version | 1.2 | 1.2 | 8 |

---

## Interview Tips
- Iterator works with ALL Collections; ListIterator works only with Lists
- ListIterator can go backward (previous), Iterator cannot
- ListIterator can add and set elements; Iterator can only remove
- Always call next() before remove() - otherwise IllegalStateException
- Spliterator is the backbone of parallel streams
- Know how to implement Iterable to enable for-each support

---

[Prev: Q10 - Queue and Deque](./q10-queue-deque.md) | [Next: Q12 - Fail-Fast vs Fail-Safe](./q12-fail-fast-fail-safe.md)
