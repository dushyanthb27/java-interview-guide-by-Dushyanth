# Q2: How Does ArrayList Work Internally?

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> How does ArrayList work internally? Explain its capacity management, growth strategy, and performance characteristics.

---

## Answer

### Internal Structure

ArrayList is backed by a **resizable array** (`Object[] elementData`). It implements `List`, `RandomAccess`, `Cloneable`, and `Serializable`.

```java
// Simplified internal structure of ArrayList
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, Serializable {

    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    transient Object[] elementData;  // the backing array
    private int size;                // actual number of elements
}
```

### Capacity vs Size

```java
ArrayList<String> list = new ArrayList<>();
// elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA (empty array)
// size = 0

list.add("first");
// Now: elementData = new Object[10] (expanded to DEFAULT_CAPACITY)
// size = 1
// Capacity = 10, but size = 1

list.add("second");
// size = 2, capacity still 10
```

### Growth Strategy (1.5x Factor)

When the internal array is full, ArrayList creates a new array **1.5 times** the current capacity and copies elements using `System.arraycopy()`.

```java
// Simplified grow logic from OpenJDK source
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // New capacity = oldCapacity + (oldCapacity >> 1) = 1.5x
        int newCapacity = ArraysSupport.newLength(
            oldCapacity,
            minCapacity - oldCapacity,  // minimum growth
            oldCapacity >> 1            // preferred growth (50%)
        );
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}

// Growth progression: 10 -> 15 -> 22 -> 33 -> 49 -> 73 -> 109 -> ...
```

### Key Operations and Time Complexity

```java
ArrayList<String> list = new ArrayList<>();

// O(1) amortized - add at end
list.add("element");

// O(n) - add at index (shifts elements right)
list.add(0, "first");          // worst case: shift all elements

// O(1) - random access by index (this is ArrayList's superpower)
String item = list.get(5);

// O(n) - remove by index (shifts elements left)
list.remove(0);                // worst case: shift all elements

// O(n) - remove by object (linear search + shift)
list.remove("element");

// O(n) - contains (linear search)
boolean exists = list.contains("element");

// O(1) - size
int size = list.size();

// O(n) - indexOf (linear scan)
int idx = list.indexOf("element");
```

### Pre-sizing for Performance

```java
// ❌ Don't do this when you know the size
List<String> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);  // Multiple resizes: 10->15->22->...->1M+
}

// ✅ Pre-size when you know (or can estimate) the number of elements
List<String> list = new ArrayList<>(1_000_000);
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);  // No resizing needed
}

// ✅ Pre-size when converting from another collection
List<String> result = new ArrayList<>(sourceCollection.size());
```

### Capacity Management Methods

```java
ArrayList<String> list = new ArrayList<>(1000);
// Fill with 50 elements...

// Trim internal array to actual size (1000 -> 50)
list.trimToSize();

// Pre-allocate capacity before bulk additions
list.ensureCapacity(500);  // Avoids incremental resizes
```

### RandomAccess Marker Interface

```java
// RandomAccess is a marker interface (no methods)
// Signals that random access (get(i)) is O(1)
// Algorithms can check this to optimize behavior

public static <T> void process(List<T> list) {
    if (list instanceof RandomAccess) {
        // Use index-based loop for O(1) access per element
        for (int i = 0; i < list.size(); i++) {
            doSomething(list.get(i));
        }
    } else {
        // Use iterator for sequential access (LinkedList)
        for (T item : list) {
            doSomething(item);
        }
    }
}
```

### Memory Characteristics

```java
// ArrayList memory layout:
// [Header] -> [Object[] reference] -> [Object ref | Object ref | ... | null | null]
//                                      ^-- used --^              ^-- wasted --^
//
// Each element slot: 4-8 bytes (object reference)
// Unused slots waste memory until trimToSize() is called

// Example: ArrayList with 6 elements, capacity 10
// Wasted: 4 empty reference slots = 16-32 bytes
```

### Iteration Performance

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D", "E"));

// ✅ Enhanced for loop - clean and efficient
for (String s : list) {
    System.out.println(s);
}

// ✅ Index-based - slightly faster for ArrayList (no Iterator object)
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}

// ✅ forEach with lambda (Java 8+)
list.forEach(System.out::println);

// ❌ Don't modify during iteration (ConcurrentModificationException)
for (String s : list) {
    if (s.equals("C")) {
        list.remove(s);  // THROWS ConcurrentModificationException
    }
}

// ✅ Use removeIf instead (Java 8+)
list.removeIf(s -> s.equals("C"));

// ✅ Or use explicit Iterator with remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("C")) {
        it.remove();
    }
}
```

### Sublist Behavior (View, Not Copy)

```java
List<String> original = new ArrayList<>(List.of("A", "B", "C", "D", "E"));
List<String> sub = original.subList(1, 4);  // ["B", "C", "D"]

// ❌ subList is a VIEW - modifications reflect in original
sub.set(0, "X");
// original = ["A", "X", "C", "D", "E"]

// ❌ Modifying original after subList invalidates the subList
original.add("F");
sub.get(0);  // THROWS ConcurrentModificationException

// ✅ Create a copy if you need independence
List<String> copy = new ArrayList<>(original.subList(1, 4));
```

### Serialization Optimization

```java
// ArrayList uses transient for elementData and custom serialization
// Only actual elements are serialized (not empty slots)
// This saves bandwidth when capacity >> size

// private void writeObject(ObjectOutputStream s) {
//     s.writeInt(size);
//     for (int i = 0; i < size; i++) {
//         s.writeObject(elementData[i]);
//     }
// }
```

### Common Anti-Patterns

```java
// ❌ Frequent insertions at the beginning
List<String> log = new ArrayList<>();
log.add(0, "new entry");  // O(n) every time! Use ArrayDeque instead

// ❌ Using ArrayList as a Queue
list.remove(0);  // O(n) - use ArrayDeque for queue behavior

// ❌ Not specifying initial capacity for known sizes
List<Record> records = new ArrayList<>();  // will resize multiple times
records.addAll(loadAllRecords());          // might be thousands

// ✅ Better approach
int expectedSize = getRecordCount();
List<Record> records = new ArrayList<>(expectedSize);
records.addAll(loadAllRecords());
```

### Best Practices

```java
// ✅ ArrayList is the default choice for most List needs
List<String> items = new ArrayList<>();

// ✅ Pre-size when possible
List<String> items = new ArrayList<>(expectedSize);

// ✅ Use List.of() for immutable lists (Java 9+)
List<String> constants = List.of("A", "B", "C");

// ✅ Use trimToSize() after building a large list that won't grow
List<Data> bigList = new ArrayList<>(10_000);
// ... populate ...
bigList.trimToSize();  // Release unused capacity

// ✅ Convert to array for fixed-size hot paths
String[] array = list.toArray(new String[0]);
// Or Java 11+: String[] array = list.toArray(String[]::new);
```

---

## Performance Summary

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `get(index)` | O(1) | Direct array access |
| `add(element)` | O(1) amortized | O(n) when resize needed |
| `add(index, element)` | O(n) | Shift elements right |
| `remove(index)` | O(n) | Shift elements left |
| `contains(element)` | O(n) | Linear search |
| `size()` | O(1) | Stored field |
| `iterator.next()` | O(1) | Sequential access |

---

## Interview Tips
- Emphasize the 1.5x growth factor (not 2x - that's Vector)
- Explain amortized O(1) for add - most adds are O(1), occasional resize is O(n)
- Mention `System.arraycopy()` as a native method for fast element shifting
- Highlight RandomAccess marker interface for algorithm optimization
- Know that `subList()` returns a view, not a copy

## ⚡ Follow-up Questions
1. Why does ArrayList use a 1.5x growth factor instead of 2x?
2. What is the difference between `Arrays.asList()` and `List.of()` regarding mutability?
3. How does ArrayList handle concurrent modifications, and what is `ConcurrentModificationException`?
4. When would you choose LinkedList over ArrayList, and vice versa?
5. How does the `trimToSize()` method affect memory and subsequent performance?

## 📚 Related Topics
- [LinkedList](./03-linkedlist.md)
- [Vector and Stack](./04-vector-stack.md)
- [Collection Performance](./18-collection-performance.md)

---

[Prev: Q1 - Collection Hierarchy](./01-collection-hierarchy.md) | [Next: Q3 - LinkedList](./03-linkedlist.md)
