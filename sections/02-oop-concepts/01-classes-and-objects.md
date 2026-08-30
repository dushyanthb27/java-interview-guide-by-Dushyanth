# Q1: What Are Classes and Objects in Java?

## Question
What are classes and objects in Java? Explain the relationship between them, how objects are created in memory, and the key methods inherited from the Object class.

## Difficulty: 🟢 Basic

---

## Answer

A **class** is a blueprint or template that defines the structure (fields) and behavior (methods) of a type. An **object** is a concrete instance of a class that occupies memory at runtime.

### Class as Blueprint

```java
// Class definition - the blueprint
public class Employee {
    // State (fields)
    private String name;
    private int age;
    private double salary;

    // Behavior (methods)
    public String getName() {
        return name;
    }

    public void work() {
        System.out.println(name + " is working");
    }
}
```

### Object Creation Process

When you write `new Employee()`, several things happen in order:

1. **Memory allocation** - JVM allocates space on the **heap** for the object
2. **Field initialization** - Fields are set to default values (`null`, `0`, `false`)
3. **Constructor execution** - The constructor body runs to initialize state
4. **Reference returned** - A reference (pointer) to the heap object is stored on the **stack**

```java
public class ObjectCreationDemo {
    public static void main(String[] args) {
        // 'emp' reference lives on the STACK
        // The actual Employee object lives on the HEAP
        Employee emp = new Employee();

        // Multiple references can point to the same heap object
        Employee emp2 = emp; // Both point to the same object

        // Verify same object
        System.out.println(emp == emp2); // true (same reference)
    }
}
```

### Stack vs Heap Memory

```
 STACK (per thread)          HEAP (shared)
┌─────────────────┐     ┌──────────────────────────┐
│ emp  ──────────────────▶ Employee Object          │
│ emp2 ──────────────────▶ name: null               │
│                 │     │  age: 0                    │
│                 │     │  salary: 0.0               │
└─────────────────┘     └──────────────────────────┘
```

- **Stack**: stores local variables and object references (fast, thread-local)
- **Heap**: stores actual objects (shared across threads, garbage-collected)

### Object Class Methods

Every Java class implicitly extends `java.lang.Object`. Key methods:

```java
public class Product {
    private String id;
    private String name;
    private double price;

    public Product(String id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    // ✅ Override equals() for logical equality
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Product product = (Product) o;
        return Objects.equals(id, product.id);
    }

    // ✅ Always override hashCode() when overriding equals()
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    // ✅ Override toString() for meaningful output
    @Override
    public String toString() {
        return "Product{id='%s', name='%s', price=%.2f}".formatted(id, name, price);
    }

    // clone() - creates a shallow copy (requires Cloneable interface)
    // finalize() - called before GC (deprecated since Java 9, removed in Java 18)
    // getClass() - returns runtime class
    // wait(), notify(), notifyAll() - thread coordination
}
```

### Modern Alternative: Records (Java 16+)

```java
// ✅ Records auto-generate constructor, equals, hashCode, toString
public record ProductRecord(String id, String name, double price) {
    // Compact constructor for validation
    public ProductRecord {
        if (price < 0) throw new IllegalArgumentException("Price cannot be negative");
    }
}
```

```java
public class RecordDemo {
    public static void main(String[] args) {
        var p1 = new ProductRecord("A1", "Laptop", 999.99);
        var p2 = new ProductRecord("A1", "Laptop", 999.99);

        System.out.println(p1);            // ProductRecord[id=A1, name=Laptop, price=999.99]
        System.out.println(p1.equals(p2)); // true (auto-generated)
        System.out.println(p1.hashCode() == p2.hashCode()); // true
    }
}
```

### Common Mistake

```java
// ❌ Comparing objects with == (checks reference, not content)
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);      // false (different references)

// ✅ Use equals() for logical equality
System.out.println(s1.equals(s2)); // true (same content)
```

### Performance Tip
Use `var` (Java 10+) for local variable type inference when the type is obvious from the right-hand side. It reduces boilerplate without sacrificing readability:
```java
var employees = new ArrayList<Employee>(); // Type is inferred
```

---

## Interviewer Insight
Interviewers want to see that you understand the **memory model** (stack vs heap), the **contract between equals() and hashCode()**, and how modern Java features like **records** simplify class definitions. Mentioning the Object class hierarchy shows depth.

## Follow-up Questions
1. What happens if you override `equals()` but not `hashCode()`?
2. How does garbage collection decide when to reclaim an object?
3. What is the difference between shallow copy and deep copy?

## Related Topics
- [Constructors](./q02-constructors.md)
- [Encapsulation](./q05-encapsulation.md)
- [Inheritance](./q03-inheritance.md)
