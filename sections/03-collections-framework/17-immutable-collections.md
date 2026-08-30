# Q17: Immutable Collections in Java

[Back to Collections Framework](./README.md) | **Difficulty**: 🟠 Hard

---

## Question
> How do you create immutable collections in Java? What is the difference between unmodifiable and immutable?

---

## Answer

### Unmodifiable vs Immutable

```java
// UNMODIFIABLE = a read-only VIEW of a mutable collection
// Changes to the backing collection ARE visible through the view

List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
List<String> unmodifiable = Collections.unmodifiableList(mutable);

// unmodifiable.add("D");  // ❌ UnsupportedOperationException
mutable.add("D");           // ✅ Modifies the backing list
System.out.println(unmodifiable);  // [A, B, C, D] - change is visible!

// IMMUTABLE = truly cannot be modified, no backing collection to change
List<String> immutable = List.of("A", "B", "C");
// immutable.add("D");     // ❌ UnsupportedOperationException
// No way to modify this collection at all!
```

### Java 9+ Factory Methods (List.of, Set.of, Map.of)

```java
// ✅ List.of() - immutable list
List<String> list = List.of("A", "B", "C");
// list.add("D");     // ❌ UnsupportedOperationException
// list.set(0, "X");  // ❌ UnsupportedOperationException
// list.remove(0);    // ❌ UnsupportedOperationException

// Overloaded versions (0 to 10 elements, then varargs)
List<String> empty = List.of();
List<String> one = List.of("A");
List<String> two = List.of("A", "B");
// ... up to 10 fixed-arity overloads (no array allocation)
List<String> many = List.of("A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K");

// ❌ No null elements allowed!
List<String> bad = List.of("A", null, "C");  // NullPointerException!

// ❌ No duplicate elements for Set.of()!
Set<String> bad = Set.of("A", "B", "A");  // IllegalArgumentException!

// ✅ Set.of() - immutable set
Set<String> set = Set.of("A", "B", "C");
// Iteration order is NOT guaranteed (and deliberately randomized between runs)

// ✅ Map.of() - immutable map (up to 10 key-value pairs)
Map<String, Integer> map = Map.of(
    "Alice", 85,
    "Bob", 92,
    "Charlie", 78
);

// ✅ Map.ofEntries() - for more than 10 entries
Map<String, Integer> largeMap = Map.ofEntries(
    Map.entry("Alice", 85),
    Map.entry("Bob", 92),
    Map.entry("Charlie", 78),
    Map.entry("Diana", 95),
    Map.entry("Eve", 88)
    // ... any number of entries
);
```

### Java 10+ copyOf Methods

```java
// ✅ List.copyOf() - creates immutable copy (Java 10+)
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
List<String> immutableCopy = List.copyOf(mutable);

mutable.add("D");
System.out.println(mutable);        // [A, B, C, D]
System.out.println(immutableCopy);  // [A, B, C] - NOT affected

// Already immutable? copyOf returns same instance (optimization)
List<String> original = List.of("X", "Y");
List<String> copy = List.copyOf(original);
System.out.println(original == copy);  // true (same reference!)

// ✅ Set.copyOf() and Map.copyOf() work the same way
Set<String> immutableSet = Set.copyOf(mutableSet);
Map<String, Integer> immutableMap = Map.copyOf(mutableMap);

// ❌ null elements not allowed in copyOf
List<String> withNull = Arrays.asList("A", null, "C");
List.copyOf(withNull);  // NullPointerException!
```

### Collections.unmodifiableXxx (Pre-Java 9)

```java
// These create VIEWS (not copies) - backing collection can still change

// Unmodifiable List
List<String> original = new ArrayList<>(Arrays.asList("A", "B", "C"));
List<String> unmodList = Collections.unmodifiableList(original);

// Unmodifiable Set
Set<String> origSet = new HashSet<>(Set.of("X", "Y", "Z"));
Set<String> unmodSet = Collections.unmodifiableSet(origSet);

// Unmodifiable Map
Map<String, Integer> origMap = new HashMap<>(Map.of("a", 1));
Map<String, Integer> unmodMap = Collections.unmodifiableMap(origMap);

// Sorted variants
SortedSet<String> unmodSorted = Collections.unmodifiableSortedSet(sortedSet);
SortedMap<String, Integer> unmodSortedMap = Collections.unmodifiableSortedMap(sortedMap);
NavigableSet<String> unmodNav = Collections.unmodifiableNavigableSet(navSet);
NavigableMap<String, Integer> unmodNavMap = Collections.unmodifiableNavigableMap(navMap);

// ✅ To make truly immutable: wrap a NEW collection
List<String> trulyImmutable = Collections.unmodifiableList(
    new ArrayList<>(originalList)  // Copy first, then wrap
);
// No reference to the backing list exists, so it can't change
```

### Structural vs Deep Immutability

```java
// STRUCTURAL immutability: can't add, remove, or replace elements
// DEEP immutability: the elements themselves are also immutable

// ✅ Structurally immutable with immutable elements (truly safe)
List<String> safe = List.of("A", "B", "C");
// String is immutable, so this is deeply immutable

// ⚠️ Structurally immutable with MUTABLE elements (not fully safe!)
List<StringBuilder> risky = List.of(
    new StringBuilder("A"),
    new StringBuilder("B")
);
// risky.add(new StringBuilder("C"));  // ❌ Can't add
risky.get(0).append(" modified");       // ✅ CAN modify the element!
// risky is now: [A modified, B]

// ⚠️ Same issue with List of mutable objects
List<Date> dates = List.of(new Date(), new Date());
dates.get(0).setTime(0);  // Modifies the Date object!

// ✅ For true deep immutability:
// 1. Use immutable element types (String, Integer, LocalDate, records)
// 2. Defensive copy mutable objects before storing
// 3. Return defensive copies from getters
```

### Defensive Copying Pattern

```java
public class ImmutableTeam {
    private final List<String> members;

    public ImmutableTeam(List<String> members) {
        // ✅ Defensive copy in constructor
        this.members = List.copyOf(members);  // Java 10+
        // Or: this.members = Collections.unmodifiableList(new ArrayList<>(members));
    }

    public List<String> getMembers() {
        // ✅ Already immutable, safe to return directly
        return members;
        // If using unmodifiable wrapper: return Collections.unmodifiableList(members);
    }
}

// Caller can't modify internal state
ImmutableTeam team = new ImmutableTeam(List.of("Alice", "Bob"));
// team.getMembers().add("Charlie");  // ❌ UnsupportedOperationException

// With mutable elements, you need deeper defensive copying
public class ImmutableDepartment {
    private final List<Employee> employees;

    public ImmutableDepartment(List<Employee> employees) {
        // Deep defensive copy
        this.employees = employees.stream()
            .map(Employee::copy)  // Assuming Employee has a copy method
            .collect(Collectors.toUnmodifiableList());
    }

    public List<Employee> getEmployees() {
        // Return defensive copies
        return employees.stream()
            .map(Employee::copy)
            .collect(Collectors.toUnmodifiableList());
    }
}
```

### Guava ImmutableCollections (Third-Party)

```java
// Google Guava provides robust immutable collections
import com.google.common.collect.ImmutableList;
import com.google.common.collect.ImmutableSet;
import com.google.common.collect.ImmutableMap;

// Builder pattern
ImmutableList<String> list = ImmutableList.<String>builder()
    .add("A")
    .add("B")
    .addAll(otherList)
    .build();

// Factory methods
ImmutableList<String> list = ImmutableList.of("A", "B", "C");
ImmutableSet<String> set = ImmutableSet.of("X", "Y", "Z");
ImmutableMap<String, Integer> map = ImmutableMap.of("a", 1, "b", 2);

// From existing collections
ImmutableList<String> copy = ImmutableList.copyOf(mutableList);

// Advantages over Java's List.of():
// - Builder pattern for complex construction
// - Well-defined iteration order for ImmutableSet
// - Richer API (asList, reverse, etc.)
// - Better documented guarantees
```

### Collectors for Immutable Results (Java 10+)

```java
// ✅ Collect stream results into immutable collections
List<String> immutableList = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toUnmodifiableList());

Set<String> immutableSet = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toUnmodifiableSet());

Map<Integer, String> immutableMap = employees.stream()
    .collect(Collectors.toUnmodifiableMap(
        Employee::getId,
        Employee::getName
    ));

// Pre-Java 10: wrap with Collections.unmodifiableList()
List<String> immutable = Collections.unmodifiableList(
    employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList())
);
```

### Records + Immutable Collections (Java 16+)

```java
// Records are a natural fit for immutable data
public record Team(String name, List<String> members) {
    // ✅ Compact constructor with defensive copy
    public Team {
        members = List.copyOf(members);  // Defensive copy
    }
}

Team team = new Team("Engineering", List.of("Alice", "Bob"));
// team.members().add("Charlie");  // ❌ UnsupportedOperationException
```

### Common Anti-Patterns

```java
// ❌ Assuming unmodifiable means immutable
List<String> backing = new ArrayList<>(List.of("A", "B"));
List<String> unmod = Collections.unmodifiableList(backing);
backing.add("C");  // unmod now shows [A, B, C]!

// ❌ Returning internal mutable collections
public class UserService {
    private List<User> users = new ArrayList<>();
    public List<User> getUsers() { return users; }  // ❌ Exposes internal state!
}

// ❌ Using Arrays.asList() and thinking it's immutable
List<String> list = Arrays.asList("A", "B", "C");
list.set(0, "X");   // ✅ This WORKS! (fixed-size, not immutable)
// list.add("D");   // ❌ UnsupportedOperationException (can't change size)

// ❌ Forgetting that List.of() doesn't allow null
List<String> oops = List.of("A", null);  // NullPointerException!
```

### Best Practices

```java
// ✅ Use List.of() / Set.of() / Map.of() for constant collections (Java 9+)
private static final List<String> VALID_STATUSES = List.of("ACTIVE", "INACTIVE", "PENDING");

// ✅ Use List.copyOf() for immutable copies of dynamic data (Java 10+)
List<String> safeCopy = List.copyOf(userInput);

// ✅ Use defensive copying in constructors and getters
// ✅ Use Collectors.toUnmodifiableList() for stream results (Java 10+)

// ✅ Use immutable element types (String, Integer, LocalDate, records)
// ✅ Document whether collections returned by your APIs are modifiable

// ✅ Return empty immutable collections instead of null
public List<String> getItems() {
    return items.isEmpty() ? List.of() : List.copyOf(items);
}
```

---

## Comparison Table

| Approach | Truly Immutable | Null Elements | Java Version |
|----------|----------------|---------------|-------------|
| `Collections.unmodifiableList()` | No (view only) | Yes | 1.2 |
| `Arrays.asList()` | No (set works) | Yes | 1.2 |
| `List.of()` | Yes | No | 9 |
| `List.copyOf()` | Yes | No | 10 |
| `Collectors.toUnmodifiableList()` | Yes | No | 10 |
| Guava `ImmutableList` | Yes | No | Third-party |

---

## Interview Tips
- Key distinction: unmodifiable (view) vs immutable (true copy)
- `List.of()` creates a truly immutable list; `Collections.unmodifiableList()` is a view
- `Arrays.asList()` is fixed-size but NOT immutable (set() works!)
- Structural immutability does NOT guarantee deep immutability
- Defensive copying is essential when storing/returning mutable objects
- `List.of()` throws NullPointerException on null elements, `Set.of()` throws on duplicates

---

[Prev: Q16 - Optional Handling](./q16-optional-handling.md) | [Next: Q18 - Collection Performance](./q18-collection-performance.md)
