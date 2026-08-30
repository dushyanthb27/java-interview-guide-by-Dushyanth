# Q13: Comparable vs Comparator

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> What is the difference between Comparable and Comparator? How do you use Comparator chaining?

---

## Answer

### Comparable - Natural Ordering (Internal)

`Comparable` defines the **natural ordering** of a class. The class itself decides how its instances are compared.

```java
// Comparable is in java.lang package
public interface Comparable<T> {
    int compareTo(T o);
    // Returns:
    //   negative -> this < other
    //   zero     -> this == other
    //   positive -> this > other
}

// Example: Employee sorted by ID (natural ordering)
public class Employee implements Comparable<Employee> {
    private int id;
    private String name;
    private double salary;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);  // Sort by ID ascending
    }
}

// Usage
List<Employee> employees = new ArrayList<>();
employees.add(new Employee(3, "Charlie", 50000));
employees.add(new Employee(1, "Alice", 60000));
employees.add(new Employee(2, "Bob", 55000));

Collections.sort(employees);  // Uses compareTo (natural ordering)
// Result: [Employee(1, Alice), Employee(2, Bob), Employee(3, Charlie)]

// Also works with TreeSet/TreeMap
TreeSet<Employee> sortedSet = new TreeSet<>(employees);
```

### Comparator - External/Custom Ordering

`Comparator` defines an **external ordering** strategy. It allows multiple sort orders without modifying the class.

```java
// Comparator is in java.util package
public interface Comparator<T> {
    int compare(T o1, T o2);
    // Returns:
    //   negative -> o1 < o2
    //   zero     -> o1 == o2
    //   positive -> o1 > o2
}

// Traditional anonymous class approach
Comparator<Employee> byName = new Comparator<Employee>() {
    @Override
    public int compare(Employee e1, Employee e2) {
        return e1.getName().compareTo(e2.getName());
    }
};

// Lambda approach (Java 8+)
Comparator<Employee> byName = (e1, e2) -> e1.getName().compareTo(e2.getName());

// Method reference approach (Java 8+) - PREFERRED
Comparator<Employee> byName = Comparator.comparing(Employee::getName);

// Usage
Collections.sort(employees, byName);
employees.sort(byName);  // List.sort() method (Java 8+)
```

### Side-by-Side Comparison

```java
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
//  Feature          | Comparable         | Comparator
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
//  Package          | java.lang          | java.util
//  Method           | compareTo(T)       | compare(T, T)
//  Implementation   | In the class itself | External class/lambda
//  Ordering         | Single (natural)   | Multiple (custom)
//  Modifies class?  | Yes                | No
//  Sort call        | Collections.sort(list) | Collections.sort(list, comp)
//  Used by          | TreeSet, TreeMap   | TreeSet, TreeMap, sort()
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Comparator Factory Methods (Java 8+)

```java
// ✅ Comparator.comparing() - extract key and compare
Comparator<Employee> byName = Comparator.comparing(Employee::getName);
Comparator<Employee> bySalary = Comparator.comparing(Employee::getSalary);
Comparator<Employee> byId = Comparator.comparingInt(Employee::getId);

// Primitive-specific (avoids boxing):
Comparator<Employee> byAge = Comparator.comparingInt(Employee::getAge);
Comparator<Employee> bySalary = Comparator.comparingDouble(Employee::getSalary);
Comparator<Employee> byEmpNo = Comparator.comparingLong(Employee::getEmpNumber);

// ✅ Reverse order
Comparator<Employee> bySalaryDesc = Comparator.comparing(Employee::getSalary,
                                                          Comparator.reverseOrder());
// Or:
Comparator<Employee> bySalaryDesc2 = Comparator.comparing(Employee::getSalary).reversed();

// ✅ Natural order
Comparator<String> natural = Comparator.naturalOrder();     // A, B, C
Comparator<String> reverse = Comparator.reverseOrder();     // C, B, A
```

### Comparator Chaining with thenComparing

```java
// Chain multiple comparators for multi-level sorting
List<Employee> employees = getEmployees();

// Sort by department, then by salary descending, then by name
Comparator<Employee> multiSort = Comparator
    .comparing(Employee::getDepartment)
    .thenComparing(Employee::getSalary, Comparator.reverseOrder())
    .thenComparing(Employee::getName);

employees.sort(multiSort);

// Equivalent to:
// 1. Sort by department (ascending)
// 2. Within same department, sort by salary (descending)
// 3. Within same salary, sort by name (ascending)

// Using primitive-specific thenComparing:
Comparator<Employee> optimized = Comparator
    .comparing(Employee::getDepartment)
    .thenComparingDouble(Employee::getSalary)
    .thenComparingInt(Employee::getAge);
```

### Null Handling with nullsFirst/nullsLast

```java
List<String> names = new ArrayList<>(Arrays.asList("Charlie", null, "Alice", null, "Bob"));

// ❌ Without null handling -> NullPointerException
// names.sort(Comparator.naturalOrder());

// ✅ nullsFirst: nulls come first
names.sort(Comparator.nullsFirst(Comparator.naturalOrder()));
// [null, null, Alice, Bob, Charlie]

// ✅ nullsLast: nulls come last
names.sort(Comparator.nullsLast(Comparator.naturalOrder()));
// [Alice, Bob, Charlie, null, null]

// Combining with chained comparators
Comparator<Employee> safeSort = Comparator
    .comparing(Employee::getDepartment,
               Comparator.nullsLast(Comparator.naturalOrder()))
    .thenComparing(Employee::getName,
               Comparator.nullsFirst(Comparator.naturalOrder()));

// Nested null handling for complex objects
Comparator<Employee> withNullManager = Comparator
    .comparing(
        Employee::getManager,
        Comparator.nullsLast(Comparator.comparing(Manager::getName))
    );
```

### Practical Sorting Examples

```java
// ✅ Sort strings case-insensitively
List<String> words = List.of("banana", "Apple", "cherry", "Date");
List<String> sorted = words.stream()
    .sorted(String.CASE_INSENSITIVE_ORDER)
    .collect(Collectors.toList());
// [Apple, banana, cherry, Date]

// ✅ Sort by string length, then alphabetically
Comparator<String> byLengthThenAlpha = Comparator
    .comparingInt(String::length)
    .thenComparing(Comparator.naturalOrder());
words.stream().sorted(byLengthThenAlpha).collect(Collectors.toList());
// [Date, Apple, banana, cherry]

// ✅ Sort map entries by value
Map<String, Integer> scores = Map.of("Alice", 85, "Bob", 92, "Charlie", 78);
List<Map.Entry<String, Integer>> sorted = scores.entrySet().stream()
    .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
    .collect(Collectors.toList());
// [Bob=92, Alice=85, Charlie=78]

// ✅ Sort with custom priority (enum ordering)
enum Priority { LOW, MEDIUM, HIGH, CRITICAL }

List<Ticket> tickets = getTickets();
tickets.sort(Comparator
    .comparing(Ticket::getPriority, Comparator.reverseOrder())  // CRITICAL first
    .thenComparing(Ticket::getCreatedAt));                       // Then by age
```

### Implementing Comparable Correctly

```java
public class Product implements Comparable<Product> {
    private String name;
    private double price;
    private String category;

    @Override
    public int compareTo(Product other) {
        // ✅ Use helper methods to avoid overflow issues
        int result = this.category.compareTo(other.category);
        if (result != 0) return result;

        result = Double.compare(this.price, other.price);
        if (result != 0) return result;

        return this.name.compareTo(other.name);
    }

    // ❌ NEVER do this for integer comparison:
    // return this.id - other.id;  // Integer overflow risk!
    // ✅ Use Integer.compare(this.id, other.id) instead

    // Consistency with equals: compareTo() == 0 should imply equals() == true
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Product product = (Product) o;
        return Double.compare(product.price, price) == 0 &&
               Objects.equals(name, product.name) &&
               Objects.equals(category, product.category);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, price, category);
    }
}
```

### Common Anti-Patterns

```java
// ❌ Subtraction-based comparison (integer overflow risk!)
Comparator<Integer> bad = (a, b) -> a - b;
// Integer.MAX_VALUE - (-1) = Integer.MIN_VALUE (wrong result!)

// ✅ Use Integer.compare() or Comparator.comparingInt()
Comparator<Integer> good = Integer::compare;
Comparator<Integer> alsoGood = Comparator.naturalOrder();

// ❌ Inconsistent Comparable implementation
// compareTo returns 0 but equals returns false
// This causes issues with TreeSet/TreeMap

// ❌ Non-transitive comparator
// If a > b and b > c, then a > c MUST hold
// Violating this causes unpredictable sorting behavior

// ❌ Modifying objects used as TreeSet/TreeMap keys
// The tree ordering becomes corrupted
TreeSet<Employee> set = new TreeSet<>();
Employee e = new Employee(1, "Alice", 50000);
set.add(e);
e.setSalary(70000);  // If compareTo uses salary, tree is now broken!
```

### Best Practices

```java
// ✅ Use Comparable for the most common/natural ordering
// ✅ Use Comparator for alternative orderings

// ✅ Use factory methods instead of lambdas
Comparator.comparing(Employee::getName)  // Cleaner than lambda

// ✅ Use primitive-specific methods to avoid boxing
Comparator.comparingInt(Employee::getAge)
Comparator.comparingDouble(Employee::getSalary)

// ✅ Chain comparators with thenComparing for multi-level sorts
Comparator.comparing(Employee::getDept).thenComparing(Employee::getName)

// ✅ Handle nulls explicitly
Comparator.nullsFirst(Comparator.naturalOrder())

// ✅ Make Comparable consistent with equals
// ✅ Use Integer.compare, Double.compare - never use subtraction

// ✅ Use Comparator.reversed() for reverse order
employees.sort(Comparator.comparing(Employee::getSalary).reversed());
```

---

## Quick Reference

```java
// One-liner sort patterns:

// Sort by single field
list.sort(Comparator.comparing(Person::getName));

// Sort descending
list.sort(Comparator.comparing(Person::getAge).reversed());

// Sort by multiple fields
list.sort(Comparator.comparing(Person::getCity)
                    .thenComparing(Person::getName));

// Sort with null safety
list.sort(Comparator.comparing(Person::getName, 
          Comparator.nullsLast(Comparator.naturalOrder())));

// Sort with custom key extraction
list.sort(Comparator.comparing(p -> p.getName().toLowerCase()));

// Min/Max using Comparator
Employee youngest = Collections.min(employees, 
                    Comparator.comparingInt(Employee::getAge));
```

---

## Interview Tips
- Comparable = "I can compare myself" (natural ordering, single way)
- Comparator = "Someone else compares two objects" (external, multiple ways)
- Know the integer overflow bug with subtraction-based comparison
- Demonstrate Comparator chaining with `thenComparing()`
- Know `nullsFirst`/`nullsLast` for null handling
- Comparable.compareTo should be consistent with equals (especially for TreeSet)
- Use `Comparator.comparing()` factory methods over raw lambdas

---

[Prev: Q12 - Fail-Fast vs Fail-Safe](./q12-fail-fast-fail-safe.md) | [Next: Q14 - Collections Utility](./q14-collections-utility.md)
