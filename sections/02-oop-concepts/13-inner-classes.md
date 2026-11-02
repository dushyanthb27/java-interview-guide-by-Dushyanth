# Inner Classes

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Inner Classes

---

## Q16 🟡 What Are Inner Classes and Their Types?

**Question** — What are inner classes in Java? Explain the four types of inner classes, when to use each, memory leak concerns, and how they compare to lambdas.

### Answer

An **inner class** is a class defined within another class. Java supports four types of nested classes, each serving different purposes.

**Four Types of Nested Classes**

| Type | Static? | Access to Outer | Instance of Outer Required? |
|------|---------|----------------|----------------------------|
| Static nested class | Yes | Only static members | No |
| Non-static inner class | No | All members (including private) | Yes |
| Local class | No | All members + effectively final locals | Yes |
| Anonymous class | No | All members + effectively final locals | Yes |

**Static Nested Class**

Static nested classes don't need an outer instance and are best for helper classes that don't need outer class state. They behave like top-level classes but are scoped inside another class for logical grouping.

```java
public class LinkedList<E> {
    private Node<E> head;

    // Static nested class - doesn't need outer instance
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
```

**Non-Static Inner Class (Member Class)**

Non-static inner classes have an implicit reference to the outer instance and can access all its members, including private ones. They're useful when the inner class is tightly coupled to the outer class and needs its state.

```java
public class University {
    private String name;
    private List<Department> departments = new ArrayList<>();

    public University(String name) {
        this.name = name;
    }

    // Inner class - has implicit reference to outer instance
    public class Department {
        private String deptName;

        public Department(String deptName) {
            this.deptName = deptName;
        }

        public String getFullName() {
            // Can access outer class's private members
            return University.this.name + " - " + deptName;
        }
    }

    public Department addDepartment(String name) {
        var dept = new Department(name); // 'this' is implicitly passed
        departments.add(dept);
        return dept;
    }
}
```

**Local Class**

Local classes are defined inside a method. They can access effectively final local variables from the enclosing method. They're rarely used now — lambdas or extracted classes are usually better.

```java
public class ReportGenerator {
    public List<String> generateReport(List<Map<String, Object>> data, String format) {

        // Local class - defined inside a method
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

**Anonymous Class**

Anonymous classes implement an interface or extend a class inline, without a named class. They're useful for one-off implementations, especially before Java 8 lambdas.

```java
public class EventSystem {
    public void setupListeners() {
        var button = new Button("Click Me");

        // Anonymous class - implements an interface inline
        button.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                System.out.println("Button clicked!");
            }
        });

        // Anonymous class extending a class
        var specialComparator = new Comparator<String>() {
            @Override
            public int compare(String s1, String s2) {
                return s1.length() - s2.length();
            }
        };
    }
}
```

**Memory Leak with Inner Classes**

Non-static inner classes hold an implicit reference to the outer class, which can prevent garbage collection if the inner class outlives the outer class's usefulness.

```java
// Memory leak: inner class keeps outer alive
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

// Fix 1: Use a static nested class
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

// Fix 2: Use a lambda (lambdas don't capture 'this' unless needed)
public class DataManager {
    private byte[] largeData = new byte[10_000_000];

    public Runnable createTask() {
        return () -> System.out.println("Running task");
        // Lambda captures nothing - no memory leak
    }
}
```

**Lambda vs Anonymous Class**

In lambdas, `this` refers to the enclosing class instance. In anonymous classes, `this` refers to the anonymous class instance itself. Lambdas are preferred for functional interfaces — they're more concise and don't create additional .class files.

| Feature | Anonymous Class | Lambda |
|---------|----------------|--------|
| Can implement any interface | ✅ (even multi-method) | ❌ Only functional interfaces |
| Can extend a class | ✅ | ❌ |
| Has its own `this` | ✅ | ❌ (uses enclosing `this`) |
| Can have state (fields) | ✅ | ❌ |
| Serialization | ⚠️ Complex | ⚠️ Possible but fragile |
| Creates a `.class` file | ✅ (OuterClass$1.class) | ❌ (uses invokedynamic) |

> **⚠️ Common Mistake** — Using non-static inner class when static would suffice. This adds unnecessary overhead (an extra reference to the outer class) and can prevent garbage collection.

```java
// Using non-static inner class when static would suffice
public class Config {
    public class Entry { // Unnecessarily holds reference to Config
        String key, value;
    }
}

// Use static nested class when outer reference is not needed
public class Config {
    public static class Entry { // No overhead of outer reference
        String key, value;
    }
}
```

> **🔧 Performance Tip** — Prefer **static nested classes** over non-static inner classes when you don't need access to the outer instance. Each non-static inner class instance carries an extra reference (typically 4-8 bytes) to the outer class, and this reference prevents the outer class from being garbage collected.

> **🎯 Interviewer Insight** — Interviewers focus on memory leak scenarios with inner classes, the difference between `this` in lambdas vs anonymous classes, and when to use static vs non-static nested classes. Awareness of serialization pitfalls with inner classes is a bonus.

> **⚡ Follow-up** — Why does Android Lint warn about non-static inner classes in Activities?

> **⚡ Follow-up** — Can a local class access non-final local variables?

> **⚡ Follow-up** — What bytecode instruction does the JVM use for lambda creation?

> **📚 Related** — [Lambdas and OOP](./14-lambdas-and-oop.md), [This Keyword](./11-this-keyword.md), [Encapsulation](./05-encapsulation.md)

---

[← Previous: Object Composition](./12-object-composition.md) · [Next: Lambdas and OOP →](./14-lambdas-and-oop.md)
