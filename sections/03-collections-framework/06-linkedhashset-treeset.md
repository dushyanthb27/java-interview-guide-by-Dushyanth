# Q6: Compare LinkedHashSet and TreeSet

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> Compare LinkedHashSet and TreeSet. When would you use each?

---

## Answer

### LinkedHashSet - Hash Table + Doubly-Linked List

LinkedHashSet maintains **insertion order** by connecting entries with a doubly-linked list on top of the hash table structure.

```java
// LinkedHashSet extends HashSet, backed by LinkedHashMap internally
public class LinkedHashSet<E> extends HashSet<E>
        implements Set<E>, Cloneable, Serializable {

    public LinkedHashSet() {
        // Internally creates a LinkedHashMap
        super(16, .75f, true);  // special constructor in HashSet
    }
}

// Internal node structure (in LinkedHashMap):
// Each entry has: hash, key, value, next (bucket chain), before, after (order chain)
//
// Hash Table:  [bucket0] -> [entry] -> [entry]
//              [bucket1] -> [entry]
//              [bucket2] -> [entry] -> [entry]
//
// Linked List: entry1 <-> entry2 <-> entry3 <-> entry4 (insertion order)
```

### LinkedHashSet Usage

```java
Set<String> linkedSet = new LinkedHashSet<>();
linkedSet.add("Cherry");
linkedSet.add("Apple");
linkedSet.add("Banana");
linkedSet.add("Apple");  // Duplicate - ignored, order unchanged

for (String fruit : linkedSet) {
    System.out.println(fruit);
}
// Output (insertion order preserved):
// Cherry
// Apple
// Banana

// ✅ Perfect for maintaining insertion order with O(1) operations
// ✅ Iteration order is predictable and deterministic
```

### TreeSet - Red-Black Tree (Sorted Set)

TreeSet maintains elements in **sorted order** using a self-balancing Red-Black tree. It implements `NavigableSet`.

```java
// TreeSet is backed by a TreeMap
public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, Serializable {

    private transient NavigableMap<E, Object> m;  // TreeMap
    private static final Object PRESENT = new Object();

    public TreeSet() {
        this(new TreeMap<>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }
}

// Red-Black Tree visualization:
//           [D:black]
//          /          \
//     [B:red]        [F:red]
//     /    \         /    \
// [A:black] [C:black] [E:black] [G:black]
```

### TreeSet Usage

```java
// Natural ordering (Comparable)
Set<String> treeSet = new TreeSet<>();
treeSet.add("Cherry");
treeSet.add("Apple");
treeSet.add("Banana");

for (String fruit : treeSet) {
    System.out.println(fruit);
}
// Output (sorted alphabetically):
// Apple
// Banana
// Cherry

// Custom ordering with Comparator
Set<String> reverseSet = new TreeSet<>(Comparator.reverseOrder());
reverseSet.addAll(List.of("Cherry", "Apple", "Banana"));
// Iteration: Cherry, Banana, Apple

// Case-insensitive ordering
Set<String> caseInsensitive = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
caseInsensitive.add("apple");
caseInsensitive.add("Apple");  // Treated as duplicate!
caseInsensitive.size();         // 1
```

### NavigableSet Methods (TreeSet-Specific)

```java
TreeSet<Integer> nums = new TreeSet<>(List.of(10, 20, 30, 40, 50, 60, 70));

// Element retrieval
nums.first();              // 10 (smallest)
nums.last();               // 70 (largest)

// Closest element queries
nums.lower(30);            // 20 (strictly less than 30)
nums.floor(30);            // 30 (less than or equal to 30)
nums.ceiling(35);          // 40 (greater than or equal to 35)
nums.higher(30);           // 40 (strictly greater than 30)

nums.floor(25);            // 20
nums.ceiling(25);          // 30

// Polling (retrieve and remove)
nums.pollFirst();          // 10 (removes smallest)
nums.pollLast();           // 70 (removes largest)

// Range views (returns VIEWS - changes reflect back)
SortedSet<Integer> head = nums.headSet(40);           // [20, 30]
SortedSet<Integer> tail = nums.tailSet(40);            // [40, 50, 60]
SortedSet<Integer> sub  = nums.subSet(20, 50);         // [20, 30, 40]

// NavigableSet range views with inclusive flags
NavigableSet<Integer> sub2 = nums.subSet(20, true, 50, true);  // [20, 30, 40, 50]
NavigableSet<Integer> sub3 = nums.subSet(20, false, 50, false); // [30, 40]

// Descending
NavigableSet<Integer> desc = nums.descendingSet();     // [60, 50, 40, 30, 20]
Iterator<Integer> descIt = nums.descendingIterator();
```

### Null Handling

```java
// LinkedHashSet: allows ONE null element
LinkedHashSet<String> lhs = new LinkedHashSet<>();
lhs.add(null);       // ✅ OK
lhs.add(null);       // Ignored (duplicate)

// TreeSet: does NOT allow null (needs comparison)
TreeSet<String> ts = new TreeSet<>();
ts.add(null);        // ❌ NullPointerException!
// Because: how do you compare null with other elements?

// Exception: TreeSet with null-safe Comparator
TreeSet<String> ts2 = new TreeSet<>(Comparator.nullsFirst(Comparator.naturalOrder()));
ts2.add(null);       // ✅ OK with null-safe comparator
ts2.add("Apple");
// [null, Apple]
```

### Custom Objects in TreeSet

```java
// Option 1: Implement Comparable (natural ordering)
public class Student implements Comparable<Student> {
    private String name;
    private double gpa;

    @Override
    public int compareTo(Student other) {
        int result = Double.compare(other.gpa, this.gpa); // Descending GPA
        if (result == 0) {
            result = this.name.compareTo(other.name);      // Ascending name
        }
        return result;
    }
}

TreeSet<Student> students = new TreeSet<>();
students.add(new Student("Alice", 3.8));
students.add(new Student("Bob", 3.9));
// Sorted by GPA descending: [Bob(3.9), Alice(3.8)]

// Option 2: Provide Comparator (external ordering)
TreeSet<Student> byName = new TreeSet<>(
    Comparator.comparing(Student::getName)
              .thenComparing(Student::getGpa)
);

// ❌ IMPORTANT: TreeSet uses compareTo (not equals) for equality!
// If compareTo returns 0, TreeSet treats elements as EQUAL (duplicates)
public class BadStudent implements Comparable<BadStudent> {
    private String name;
    private int age;

    @Override
    public int compareTo(BadStudent other) {
        return Integer.compare(this.age, other.age);  // Only compares age
    }
}

TreeSet<BadStudent> set = new TreeSet<>();
set.add(new BadStudent("Alice", 20));
set.add(new BadStudent("Bob", 20));    // NOT added! compareTo returns 0
set.size();  // 1! Bob is "equal" to Alice because same age
```

### Performance Comparison

```java
// Operation           | HashSet  | LinkedHashSet | TreeSet
// --------------------|----------|---------------|--------
// add(e)              | O(1)     | O(1)          | O(log n)
// remove(e)           | O(1)     | O(1)          | O(log n)
// contains(e)         | O(1)     | O(1)          | O(log n)
// first()/last()      | N/A      | O(1)*         | O(log n)
// floor()/ceiling()   | N/A      | N/A           | O(log n)
// iteration           | O(cap)   | O(n)          | O(n)
// ordered?            | No       | Insertion      | Sorted
// null elements?      | Yes (1)  | Yes (1)       | No**

// * LinkedHashSet doesn't expose first/last but iteration is ordered
// ** Unless using null-safe Comparator
```

### Real-World Use Cases

```java
// ✅ LinkedHashSet: Remove duplicates while preserving order
public static <T> List<T> removeDuplicates(List<T> list) {
    return new ArrayList<>(new LinkedHashSet<>(list));
}

// ✅ LinkedHashSet: Maintaining a recently-used cache of unique items
LinkedHashSet<String> recentSearches = new LinkedHashSet<>();
recentSearches.add("java collections");
recentSearches.add("spring boot");
recentSearches.add("java collections");  // Move to... wait, LinkedHashSet
// doesn't reorder. For access-order, use LinkedHashMap.

// ✅ TreeSet: Maintaining a leaderboard
TreeSet<Player> leaderboard = new TreeSet<>(
    Comparator.comparingInt(Player::getScore).reversed()
              .thenComparing(Player::getName)
);

// ✅ TreeSet: Range queries
TreeSet<LocalDate> holidays = new TreeSet<>();
holidays.add(LocalDate.of(2024, 1, 1));
holidays.add(LocalDate.of(2024, 7, 4));
holidays.add(LocalDate.of(2024, 12, 25));
// Find holidays in Q1
SortedSet<LocalDate> q1 = holidays.subSet(
    LocalDate.of(2024, 1, 1),
    LocalDate.of(2024, 4, 1)
);

// ✅ TreeSet: Finding nearest values
TreeSet<Integer> prices = new TreeSet<>(List.of(10, 25, 50, 75, 100));
int budget = 60;
Integer bestPrice = prices.floor(budget);  // 50 (highest price <= budget)
```

### Common Anti-Patterns

```java
// ❌ Using TreeSet when you don't need sorting (performance cost)
Set<String> set = new TreeSet<>();  // O(log n) per operation
// Use HashSet if order doesn't matter: O(1) per operation

// ❌ Forgetting that TreeSet uses compareTo for equality
// Always ensure compareTo is consistent with equals

// ❌ Modifying elements after adding to TreeSet
TreeSet<StringBuilder> badIdea = new TreeSet<>();
StringBuilder sb = new StringBuilder("A");
badIdea.add(sb);
sb.append("B");  // Tree structure is now corrupted!
```

### Best Practices

```java
// ✅ Use HashSet when order doesn't matter (fastest)
Set<String> lookup = new HashSet<>();

// ✅ Use LinkedHashSet when insertion order matters
Set<String> orderedUnique = new LinkedHashSet<>();

// ✅ Use TreeSet when sorted order or range queries are needed
NavigableSet<Integer> sorted = new TreeSet<>();

// ✅ Use EnumSet for enum values (fastest Set implementation)
Set<DayOfWeek> workdays = EnumSet.range(DayOfWeek.MONDAY, DayOfWeek.FRIDAY);

// ✅ Ensure compareTo is consistent with equals for TreeSet elements
// ✅ Use immutable objects as Set elements
```

---

## Interview Tips
- LinkedHashSet = HashSet + insertion order (doubly-linked list overhead)
- TreeSet = sorted order via Red-Black tree (O(log n) operations)
- TreeSet does NOT use hashCode/equals - it uses Comparable/Comparator
- compareTo returning 0 means "equal" in TreeSet (potential data loss)
- NavigableSet methods (floor, ceiling, lower, higher) are powerful interview topics

---

[Prev: Q5 - HashSet](./q05-hashset.md) | [Next: Q7 - HashMap Internals](./q07-hashmap-internals.md)
