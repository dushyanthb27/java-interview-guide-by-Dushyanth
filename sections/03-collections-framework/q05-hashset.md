# Q5: How Does HashSet Work Internally?

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> How does HashSet work internally? Explain the hashCode/equals contract and its importance.

---

## Answer

### The Secret: HashSet is Backed by HashMap

HashSet internally uses a **HashMap** where each element is stored as a **key**, and a shared dummy `Object` is used as the value.

```java
// Simplified internal structure from OpenJDK
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, Serializable {

    private transient HashMap<E, Object> map;

    // Dummy value used as the map value for every entry
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // null means new key
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public int size() {
        return map.size();
    }

    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
}
```

### hashCode() and equals() Contract

This is one of the most critical concepts in Java. **If two objects are equal, they MUST have the same hash code.**

```java
// THE CONTRACT:
// 1. If a.equals(b) is true  -> a.hashCode() == b.hashCode() MUST be true
// 2. If a.hashCode() == b.hashCode() -> a.equals(b) MAY be false (collision)
// 3. hashCode() must be consistent: same object returns same hash within JVM session
// 4. equals() must be: reflexive, symmetric, transitive, consistent, null-safe

public class Employee {
    private int id;
    private String name;
    private String department;

    // ✅ Correct implementation
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                       // reflexive
        if (o == null || getClass() != o.getClass()) return false;  // null-safe
        Employee employee = (Employee) o;
        return id == employee.id &&
               Objects.equals(name, employee.name);       // symmetric, transitive
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);  // Uses same fields as equals()
    }
}
```

### What Happens When the Contract is Violated

```java
// ❌ BROKEN: Override equals() but NOT hashCode()
public class BrokenKey {
    private String value;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        return Objects.equals(value, ((BrokenKey) o).value);
    }

    // Missing hashCode() override!
    // Uses default Object.hashCode() based on memory address
}

Set<BrokenKey> set = new HashSet<>();
BrokenKey key1 = new BrokenKey("hello");
BrokenKey key2 = new BrokenKey("hello");

set.add(key1);
set.contains(key2);  // FALSE! Different hashCode -> different bucket
set.add(key2);       // Duplicate added! Set now has 2 "equal" elements
set.size();           // Returns 2 instead of expected 1

// ❌ BROKEN: hashCode() uses mutable fields
public class MutableKey {
    private String name;

    public void setName(String name) { this.name = name; }

    @Override
    public int hashCode() { return Objects.hash(name); }

    @Override
    public boolean equals(Object o) { /* uses name */ }
}

Set<MutableKey> set = new HashSet<>();
MutableKey key = new MutableKey();
key.setName("original");
set.add(key);
set.contains(key);     // true

key.setName("modified");  // Hash code changes!
set.contains(key);         // FALSE! Wrong bucket now - element is LOST
set.remove(key);           // FALSE! Can't find it - MEMORY LEAK
```

### Capacity, Load Factor, and Rehashing

```java
// Default initial capacity: 16 (must be power of 2)
// Default load factor: 0.75
// Threshold = capacity * load_factor = 16 * 0.75 = 12

HashSet<String> set = new HashSet<>();        // capacity=16, load=0.75
HashSet<String> set = new HashSet<>(32);      // capacity=32, load=0.75
HashSet<String> set = new HashSet<>(32, 0.5f); // capacity=32, load=0.5

// When size > threshold, the internal HashMap REHASHES:
// 1. Creates new array with 2x capacity
// 2. Recalculates hash for every element
// 3. Places each element in new bucket
// This is O(n) and can cause a performance spike!

// ✅ Pre-size when you know the number of elements
int expectedSize = 1000;
// Account for load factor: capacity = size / loadFactor + 1
HashSet<String> set = new HashSet<>(expectedSize * 4 / 3 + 1);
// Or simpler: just use a larger initial capacity
HashSet<String> set = new HashSet<>(2000);
```

### Collision Handling

```java
// When two different objects have the same bucket index:
// Java 7: linked list in the bucket
// Java 8+: linked list -> red-black tree when bucket size > 8

// Example of collision:
// "Aa".hashCode() == "BB".hashCode() == 2112 (known collision pair)

Set<String> set = new HashSet<>();
set.add("Aa");   // bucket = hash("Aa") % capacity
set.add("BB");   // Same bucket! Stored as next node in linked list

// Internally in the bucket:
// [Aa] -> [BB] -> null   (linked list, both at same index)
//
// When bucket has > 8 entries (TREEIFY_THRESHOLD):
// Linked list converts to red-black tree for O(log n) lookup
```

### Null Handling

```java
HashSet<String> set = new HashSet<>();

set.add(null);       // ✅ Allowed - HashSet permits ONE null element
set.add(null);       // Returns false - null already exists
set.contains(null);  // true
set.remove(null);    // true - null removed

// Internally: null key is stored at index 0 in HashMap's bucket array
// HashMap handles null specially in its hash() method
```

### LinkedHashSet for Insertion Order

```java
// LinkedHashSet maintains insertion order using a doubly-linked list
Set<String> ordered = new LinkedHashSet<>();
ordered.add("Banana");
ordered.add("Apple");
ordered.add("Cherry");

for (String fruit : ordered) {
    System.out.println(fruit);
}
// Output: Banana, Apple, Cherry (insertion order preserved)

// Compare with HashSet (no guaranteed order):
Set<String> unordered = new HashSet<>();
unordered.add("Banana");
unordered.add("Apple");
unordered.add("Cherry");
// Iteration order: unpredictable (depends on hash codes)
```

### Practical Examples

```java
// ✅ Removing duplicates from a list
List<String> withDuplicates = List.of("A", "B", "A", "C", "B", "D");
List<String> unique = new ArrayList<>(new HashSet<>(withDuplicates));
// unique = [A, B, C, D] (order not preserved)

// ✅ Removing duplicates while preserving order
List<String> uniqueOrdered = new ArrayList<>(new LinkedHashSet<>(withDuplicates));
// uniqueOrdered = [A, B, A, C, B, D] -> [A, B, C, D] (order preserved)

// ✅ Set operations
Set<Integer> setA = new HashSet<>(Set.of(1, 2, 3, 4, 5));
Set<Integer> setB = new HashSet<>(Set.of(4, 5, 6, 7, 8));

// Union
Set<Integer> union = new HashSet<>(setA);
union.addAll(setB);     // [1, 2, 3, 4, 5, 6, 7, 8]

// Intersection
Set<Integer> intersection = new HashSet<>(setA);
intersection.retainAll(setB);  // [4, 5]

// Difference (A - B)
Set<Integer> difference = new HashSet<>(setA);
difference.removeAll(setB);    // [1, 2, 3]

// Symmetric difference
Set<Integer> symDiff = new HashSet<>(union);
symDiff.removeAll(intersection);  // [1, 2, 3, 6, 7, 8]
```

### Common Anti-Patterns

```java
// ❌ Using mutable objects as Set elements
Set<List<String>> badSet = new HashSet<>();
List<String> list = new ArrayList<>(List.of("A", "B"));
badSet.add(list);
badSet.contains(list);  // true
list.add("C");           // Mutate the element!
badSet.contains(list);  // FALSE - hash code changed, element is LOST

// ❌ Forgetting to override hashCode when overriding equals
// See BrokenKey example above

// ❌ Using HashSet when order matters
Set<String> items = new HashSet<>();  // No order guarantee
```

### Best Practices

```java
// ✅ Use immutable objects as Set elements
Set<String> set = new HashSet<>();  // String is immutable - safe

// ✅ Override both equals() and hashCode() using same fields
// Use @Override annotation to catch signature mistakes
// Use Objects.hash() and Objects.equals() for null-safe implementations

// ✅ Use Set.of() for immutable sets (Java 9+)
Set<String> immutable = Set.of("A", "B", "C");

// ✅ Pre-size when you know the expected number of elements
Set<String> set = new HashSet<>(expectedCount * 4 / 3 + 1);

// ✅ Use EnumSet for enum types (extremely fast, backed by bit vector)
Set<DayOfWeek> weekend = EnumSet.of(DayOfWeek.SATURDAY, DayOfWeek.SUNDAY);

// ✅ Use LinkedHashSet when insertion order matters
Set<String> orderedSet = new LinkedHashSet<>();
```

---

## Performance Summary

| Operation | HashSet | LinkedHashSet |
|-----------|---------|---------------|
| `add(e)` | O(1) avg | O(1) avg |
| `remove(e)` | O(1) avg | O(1) avg |
| `contains(e)` | O(1) avg | O(1) avg |
| Iteration | O(capacity) | O(n) |
| Order | None | Insertion order |
| Memory | Lower | Slightly higher (linked list) |

---

## Interview Tips
- HashSet is backed by HashMap - know this internal detail
- The equals/hashCode contract is a TOP interview question
- Explain why mutable keys are dangerous
- Know the difference between capacity, size, load factor, and threshold
- Mention Java 8's treeification of collision chains (TREEIFY_THRESHOLD = 8)

---

[Prev: Q4 - Vector and Stack](./q04-vector-stack.md) | [Next: Q6 - LinkedHashSet and TreeSet](./q06-linkedhashset-treeset.md)
