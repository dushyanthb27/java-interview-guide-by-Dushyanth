# Q13: What Are Inner Classes and Their Types?

## Question
What are inner classes in Java? Explain the four types of inner classes, when to use each, memory leak concerns, and how they compare to lambdas.

## Difficulty: 🟡 Intermediate

---

## Answer

An **inner class** is a class defined within another class. Java supports four types of nested classes, each serving different purposes.

### Four Types of Nested Classes

| Type | Static? | Access to Outer | Instance of Outer Required? |
|------|---------|----------------|----------------------------|
| Static nested class | Yes | Only static members | No |
| Non-static inner class | No | All members (including private) | Yes |
| Local class | No | All members + effectively final locals | Yes |
| Anonymous class | No | All members + effectively final locals | Yes |

### 1. Static Nested Class

```java
public class LinkedList<E> {
    private Node<E> head;

    // ✅ Static nested class - doesn't need outer instance
    // Best for helper classes that don't need outer class state
    private static class Node<E> {
        E data;
        Node<E> next;

        Node(E data) {
            this.data = data;
        }
    }

    public void add(E element) {
        var newNode = new Node<>(element); // No outer reference needed
        newNode.next = head;
        head = newNode;
    }
}

// Creating from outside (if public):
// LinkedList.Node<String> node = new LinkedList.Node<>("hello");
```

### 2. Non-Static Inner Class (Member Class)

```java
public class University {
    private String name;
    private List<Department> departments = new ArrayList<>();

    public University(String name) {
        this.name = name;
    }

    // ✅ Inner class - has implicit reference to outer instance
    public class Department {
        private String deptName;

        public Department(String deptName) {
            this.deptName = deptName;
        }

        public String getFullName() {
            // ✅ Can access outer class's private members
            return University.this.name + " - " + deptName;
        }
    }

    public Department addDepartment(String name) {
        var dept = new Department(name); // 'this' is implicitly passed
        departments.add(dept);
        return dept;
    }
}

// Creating inner class instance requires outer instance:
University uni = new University("MIT");
University.Department dept = uni.new Department("CS");
```

### 3. Local Class

```java
public class ReportGenerator {
    public List<String> generateReport(List<Map<String, Object>> data, String format) {

        // ✅ Local class - defined inside a method
        class Formatter {
            private final String prefix;

            Formatter(String prefix) {
                this.prefix = prefix;
            }

            String formatRow(Map<String, Object> row) {
                // Can access 'format' (effectively final local variable)
                return switch (format) {
                    case "CSV" -> row.values().stream()
                            .map(Object::toString)
                            .collect(Collectors.joining(","));
                    case "JSON" -> row.toString();
                    default -> prefix + row.toString();
                };
            }
        }

        var formatter = new Formatter("[REPORT] ");
        return data.stream()
                   .map(formatter::formatRow)
                   .toList();
    }
}
```

### 4. Anonymous Class

```java
public class EventSystem {
    public void setupListeners() {
        var button = new Button("Click Me");

        // ✅ Anonymous class - implements an interface inline
        button.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                System.out.println("Button clicked!");
            }
        });

        // ✅ Anonymous class extending a class
        var specialComparator = new Comparator<String>() {
            @Override
            public int compare(String s1, String s2) {
                return s1.length() - s2.length();
            }
        };
    }
}
```

### Memory Leak with Inner Classes

Non-static inner classes hold an implicit reference to the outer class, which can prevent garbage collection:

```java
// ❌ Memory leak: inner class keeps outer alive
public class DataManager {
    private byte[] largeData = new byte[10_000_000]; // 10 MB

    public Runnable createTask() {
        // This anonymous class holds a reference to DataManager
        return new Runnable() {
            @Override
            public void run() {
                System.out.println("Running task");
                // Even though largeData is not used, it won't be GC'd
                // because 'this' has an implicit reference to DataManager
            }
        };
    }
}

// ✅ Fix 1: Use a static nested class
public class DataManager {
    private byte[] largeData = new byte[10_000_000];

    private static class Task implements Runnable {
        @Override
        public void run() {
            System.out.println("Running task");
            // No reference to DataManager
        }
    }

    public Runnable createTask() {
        return new Task();
    }
}

// ✅ Fix 2: Use a lambda (lambdas don't capture 'this' unless needed)
public class DataManager {
    private byte[] largeData = new byte[10_000_000];

    public Runnable createTask() {
        return () -> System.out.println("Running task");
        // Lambda captures nothing - no memory leak
    }
}
```

### Lambda vs Anonymous Class

```java
public class LambdaVsAnonymous {
    private String name = "Outer";

    public void demonstrate() {
        // Anonymous class: 'this' refers to the anonymous class instance
        Runnable anon = new Runnable() {
            @Override
            public void run() {
                System.out.println(this.getClass().getSimpleName()); // "" (anonymous)
            }
        };

        // Lambda: 'this' refers to the ENCLOSING class instance
        Runnable lambda = () -> {
            System.out.println(this.getClass().getSimpleName()); // "LambdaVsAnonymous"
            System.out.println(this.name); // Accessible!
        };

        anon.run();
        lambda.run();
    }
}
```

| Feature | Anonymous Class | Lambda |
|---------|----------------|--------|
| Can implement any interface | ✅ (even multi-method) | ❌ Only functional interfaces |
| Can extend a class | ✅ | ❌ |
| Has its own `this` | ✅ | ❌ (uses enclosing `this`) |
| Can have state (fields) | ✅ | ❌ |
| Serialization | ⚠️ Complex | ⚠️ Possible but fragile |
| Creates a `.class` file | ✅ (OuterClass$1.class) | ❌ (uses invokedynamic) |

### When to Use Each Type

```java
// ✅ Static nested: helper/utility class tightly coupled to outer
public class Map<K, V> {
    static class Entry<K, V> { } // No outer instance needed
}

// ✅ Non-static inner: needs access to outer instance state
public class Iterator<E> {
    class ListIterator { } // Needs List's internal state
}

// ✅ Local class: complex logic needed only in one method
// (rare - usually replaced by lambdas or extracted classes)

// ✅ Anonymous class: one-off implementation of multi-method interface
// ✅ Lambda: one-off implementation of functional interface (preferred)
```

### Common Mistake

```java
// ❌ Using non-static inner class when static would suffice
public class Config {
    public class Entry { // Unnecessarily holds reference to Config
        String key, value;
    }
}

// ✅ Use static nested class when outer reference is not needed
public class Config {
    public static class Entry { // No overhead of outer reference
        String key, value;
    }
}
```

### Performance Tip
Prefer **static nested classes** over non-static inner classes when you don't need access to the outer instance. Each non-static inner class instance carries an extra reference (typically 4-8 bytes) to the outer class, and this reference prevents the outer class from being garbage collected.

---

## Interviewer Insight
Interviewers focus on memory leak scenarios with inner classes, the difference between `this` in lambdas vs anonymous classes, and when to use static vs non-static nested classes. Awareness of serialization pitfalls with inner classes is a bonus.

## Follow-up Questions
1. Why does Android Lint warn about non-static inner classes in Activities?
2. Can a local class access non-final local variables?
3. What bytecode instruction does the JVM use for lambda creation?

## Related Topics
- [Lambdas and OOP](./q14-lambdas-and-oop.md)
- [This Keyword](./q11-this-keyword.md)
- [Encapsulation](./q05-encapsulation.md)
