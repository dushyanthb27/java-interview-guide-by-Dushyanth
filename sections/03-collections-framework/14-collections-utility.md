# Q14: Collections Utility Class Methods

[Back to Collections Framework](./README.md) | **Difficulty**: 🟢 Easy

---

## Question
> What are the key methods in the Collections utility class?

---

## Answer

### Overview

`java.util.Collections` is a utility class containing **static methods** for operating on collections. It provides sorting, searching, synchronization, immutability wrappers, and more.

```java
// Collections is a utility class (not to be confused with Collection interface)
// Collection = interface (List, Set, Queue extend it)
// Collections = utility class with static helper methods
```

### Sorting

```java
List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob", "David"));

// Natural order sort
Collections.sort(names);
// [Alice, Bob, Charlie, David]

// Custom comparator sort
Collections.sort(names, Comparator.reverseOrder());
// [David, Charlie, Bob, Alice]

// ✅ Prefer List.sort() (Java 8+) - same result, more readable
names.sort(Comparator.naturalOrder());
names.sort(Comparator.reverseOrder());
names.sort(String.CASE_INSENSITIVE_ORDER);
```

### Binary Search

```java
List<Integer> sorted = new ArrayList<>(List.of(10, 20, 30, 40, 50, 60));

// ⚠️ List MUST be sorted before calling binarySearch!
int index = Collections.binarySearch(sorted, 30);   // Returns 2
int notFound = Collections.binarySearch(sorted, 35); // Returns -(insertion point) - 1 = -4

// With custom comparator
List<String> words = new ArrayList<>(List.of("apple", "banana", "cherry"));
int idx = Collections.binarySearch(words, "banana", Comparator.naturalOrder()); // 1

// ❌ Don't use on unsorted list - undefined behavior
List<Integer> unsorted = List.of(5, 3, 1, 4, 2);
Collections.binarySearch(unsorted, 3);  // Undefined result!
```

### Unmodifiable Wrappers

```java
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));

// Unmodifiable VIEW (not a copy!)
List<String> unmodifiable = Collections.unmodifiableList(mutable);
// unmodifiable.add("D");    // ❌ UnsupportedOperationException
// unmodifiable.remove(0);   // ❌ UnsupportedOperationException

// ⚠️ Changes to the backing list STILL reflect through the view!
mutable.add("D");
System.out.println(unmodifiable);  // [A, B, C, D] - view updated!

// Same for Set and Map
Set<String> unmodSet = Collections.unmodifiableSet(new HashSet<>(Set.of("X", "Y")));
Map<String, Integer> unmodMap = Collections.unmodifiableMap(new HashMap<>(Map.of("a", 1)));

// ✅ For truly immutable collections (Java 9+), use:
List<String> immutable = List.of("A", "B", "C");           // Immutable copy
List<String> copied = List.copyOf(mutable);                 // Java 10+ immutable copy
Map<String, Integer> immMap = Map.of("a", 1, "b", 2);      // Immutable map

// Also available:
Collections.unmodifiableSortedSet(sortedSet);
Collections.unmodifiableSortedMap(sortedMap);
Collections.unmodifiableNavigableSet(navSet);
Collections.unmodifiableNavigableMap(navMap);
```

### Synchronized Wrappers

```java
// Wrap any collection to make it thread-safe (coarse-grained synchronization)
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// Individual operations are thread-safe
syncList.add("A");     // synchronized
syncList.get(0);       // synchronized

// ⚠️ Iteration MUST be manually synchronized!
synchronized (syncList) {
    for (String s : syncList) {
        System.out.println(s);
    }
}

// ⚠️ Compound operations are NOT atomic
// This is NOT thread-safe:
if (!syncList.contains("item")) {   // Lock released between these two calls
    syncList.add("item");            // Another thread might add it in between!
}

// ✅ For better concurrency, use java.util.concurrent collections
// ConcurrentHashMap, CopyOnWriteArrayList, etc.
```

### Ordering Operations

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5));

// Reverse
Collections.reverse(nums);        // [5, 4, 3, 2, 1]

// Shuffle (random order)
Collections.shuffle(nums);        // Random order, e.g., [3, 1, 5, 2, 4]
Collections.shuffle(nums, new Random(42));  // Reproducible shuffle with seed

// Rotate elements
List<String> items = new ArrayList<>(List.of("A", "B", "C", "D", "E"));
Collections.rotate(items, 2);     // [D, E, A, B, C] (rotate right by 2)
Collections.rotate(items, -1);    // [E, A, B, C, D] (rotate left by 1)

// Swap elements
Collections.swap(items, 0, 3);    // Swap positions 0 and 3
```

### Frequency and Disjoint

```java
List<String> words = List.of("apple", "banana", "apple", "cherry", "apple");

// Count occurrences
int count = Collections.frequency(words, "apple");  // 3
int none = Collections.frequency(words, "grape");    // 0

// Check if two collections have NO common elements
List<String> list1 = List.of("A", "B", "C");
List<String> list2 = List.of("D", "E", "F");
List<String> list3 = List.of("C", "D", "E");

boolean disjoint1 = Collections.disjoint(list1, list2);  // true (no common elements)
boolean disjoint2 = Collections.disjoint(list1, list3);  // false ("C" is common)
```

### Min and Max

```java
List<Integer> numbers = List.of(42, 17, 93, 5, 68);

// Natural ordering
int min = Collections.min(numbers);  // 5
int max = Collections.max(numbers);  // 93

// Custom comparator
String shortest = Collections.min(
    List.of("hello", "hi", "hey"),
    Comparator.comparingInt(String::length)
);  // "hi"

// With complex objects
Employee youngest = Collections.min(employees,
    Comparator.comparingInt(Employee::getAge));
```

### Fill, Copy, and Replace

```java
// Fill entire list with a single value
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Collections.fill(list, "X");      // [X, X, X]

// Copy source into destination (dest must be at least as large as src)
List<String> src = List.of("A", "B", "C");
List<String> dest = new ArrayList<>(Arrays.asList("X", "X", "X", "X", "X"));
Collections.copy(dest, src);      // [A, B, C, X, X]

// Replace all occurrences
List<String> items = new ArrayList<>(List.of("A", "B", "A", "C", "A"));
boolean changed = Collections.replaceAll(items, "A", "Z");
// [Z, B, Z, C, Z], changed = true
```

### Singleton and Empty Collections

```java
// Singleton - immutable collection with exactly one element
List<String> singleList = Collections.singletonList("only");
Set<String> singleSet = Collections.singleton("only");
Map<String, Integer> singleMap = Collections.singletonMap("key", 42);
// singleList.add("more");  // ❌ UnsupportedOperationException

// Empty collections - immutable, shared instances
List<String> emptyList = Collections.emptyList();
Set<String> emptySet = Collections.emptySet();
Map<String, String> emptyMap = Collections.emptyMap();
Iterator<String> emptyIt = Collections.emptyIterator();

// ✅ Prefer returning empty collections over null
public List<String> getResults() {
    if (noResults) {
        return Collections.emptyList();  // ✅ Never return null
        // Or: return List.of();         // Java 9+
    }
    return results;
}

// ✅ Java 9+ alternatives
List.of()          // empty immutable list
Set.of()           // empty immutable set
Map.of()           // empty immutable map
```

### Enumeration Conversion

```java
// Convert between Iterator and Enumeration (legacy interop)
List<String> list = List.of("A", "B", "C");

// Iterator -> Enumeration
Enumeration<String> enumeration = Collections.enumeration(list);

// Enumeration -> List
Vector<String> vector = new Vector<>(List.of("X", "Y"));
ArrayList<String> fromEnum = Collections.list(vector.elements());
```

### nCopies and addAll

```java
// Create immutable list of n copies
List<String> fiveHellos = Collections.nCopies(5, "Hello");
// [Hello, Hello, Hello, Hello, Hello]
// Immutable! Memory efficient (stores only one reference)

// Initialize a list with default values
List<Integer> zeros = new ArrayList<>(Collections.nCopies(100, 0));

// Add multiple elements at once
List<String> list = new ArrayList<>();
Collections.addAll(list, "A", "B", "C", "D");  // Varargs
// More efficient than multiple list.add() calls
```

### Checked Collections (Type-Safe Views)

```java
// Enforce type safety at runtime (useful with raw types from legacy code)
List<String> safeList = Collections.checkedList(new ArrayList<>(), String.class);
safeList.add("OK");        // ✅ Fine
// safeList.add(42);       // ❌ ClassCastException at runtime!

Map<String, Integer> safeMap = Collections.checkedMap(
    new HashMap<>(), String.class, Integer.class
);
```

### Common Anti-Patterns

```java
// ❌ Returning null instead of empty collection
public List<String> findItems() {
    if (noItems) return null;  // Forces null checks everywhere
}

// ❌ Assuming unmodifiableList creates a copy
List<String> original = new ArrayList<>(List.of("A", "B"));
List<String> unmod = Collections.unmodifiableList(original);
original.add("C");  // unmod now shows [A, B, C]!

// ❌ Forgetting to synchronize iteration on synchronized collections
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
for (String s : syncList) {  // NOT thread-safe!
    // ...
}

// ❌ Binary search on unsorted list
Collections.binarySearch(unsortedList, target);  // Wrong results!
```

### Best Practices

```java
// ✅ Return empty collections, never null
return results.isEmpty() ? Collections.emptyList() : results;

// ✅ Use List.of() / Map.of() for immutable collections (Java 9+)
List<String> immutable = List.of("A", "B", "C");

// ✅ Use List.copyOf() for truly immutable copies (Java 10+)
List<String> safeCopy = List.copyOf(mutableList);

// ✅ Sort with List.sort() instead of Collections.sort() (Java 8+)
list.sort(Comparator.naturalOrder());

// ✅ Use Collections.unmodifiableXxx() for defensive returns
public List<String> getItems() {
    return Collections.unmodifiableList(items);
}

// ✅ Use ConcurrentHashMap instead of Collections.synchronizedMap()
Map<String, Integer> safe = new ConcurrentHashMap<>();
```

---

## Quick Reference

| Method | Description | Returns |
|--------|-------------|---------|
| `sort(list)` | Sort in natural order | void |
| `binarySearch(list, key)` | Binary search (list must be sorted) | index or -(insertion)-1 |
| `reverse(list)` | Reverse order | void |
| `shuffle(list)` | Random order | void |
| `swap(list, i, j)` | Swap elements at positions | void |
| `frequency(coll, obj)` | Count occurrences | int |
| `disjoint(c1, c2)` | Check no common elements | boolean |
| `min(coll)` / `max(coll)` | Find min/max | element |
| `unmodifiableList(list)` | Read-only view | List |
| `synchronizedList(list)` | Thread-safe wrapper | List |
| `emptyList()` | Empty immutable list | List |
| `singletonList(e)` | Single-element immutable list | List |
| `nCopies(n, obj)` | Immutable list of n copies | List |

---

## Interview Tips
- `Collections` (utility class) vs `Collection` (interface) - know the difference
- `unmodifiableList` is a VIEW, not a copy - changes to the original are reflected
- `synchronizedList` requires manual sync during iteration
- Always return empty collections instead of null
- `binarySearch` requires a pre-sorted list
- Know Java 9+ alternatives: `List.of()`, `Set.of()`, `Map.of()`

---

[Prev: Q13 - Comparable vs Comparator](./q13-comparable-comparator.md) | [Next: Q15 - Streams API with Collections](./q15-streams-api-collections.md)
