# Q2: Explain Constructors and Their Types

## Question
What are constructors in Java? Explain the different types of constructors, constructor chaining, and how constructors differ from methods.

## Difficulty: 🟢 Basic

---

## Answer

A **constructor** is a special block of code that initializes a newly created object. It has the same name as the class, no return type (not even `void`), and is invoked automatically when `new` is called.

### Types of Constructors

#### 1. Default Constructor (No-Arg)

```java
public class User {
    private String name;
    private int age;

    // ✅ Default constructor - provided by compiler if no constructor is defined
    public User() {
        // Fields get default values: name = null, age = 0
    }
}
```

> **Note**: If you define ANY constructor, the compiler does NOT generate a default one.

#### 2. Parameterized Constructor

```java
public class User {
    private String name;
    private int age;

    // ✅ Parameterized constructor
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

#### 3. Copy Constructor

```java
public class User {
    private String name;
    private int age;
    private List<String> roles;

    public User(String name, int age, List<String> roles) {
        this.name = name;
        this.age = age;
        this.roles = roles;
    }

    // ✅ Copy constructor - creates a deep copy
    public User(User other) {
        this.name = other.name;
        this.age = other.age;
        this.roles = new ArrayList<>(other.roles); // Deep copy of mutable field
    }
}
```

### Constructor Chaining with `this()`

```java
public class Employee {
    private String name;
    private int age;
    private String department;
    private double salary;

    // Primary constructor
    public Employee(String name, int age, String department, double salary) {
        this.name = name;
        this.age = age;
        this.department = department;
        this.salary = salary;
    }

    // ✅ Chains to the primary constructor using this()
    public Employee(String name, int age) {
        this(name, age, "General", 50000.0); // Must be FIRST statement
    }

    // ✅ Chains to the two-arg constructor
    public Employee(String name) {
        this(name, 0);
    }

    // ✅ No-arg chains to the one-arg constructor
    public Employee() {
        this("Unknown");
    }
}
```

### Constructor vs Method

| Feature | Constructor | Method |
|---------|------------|--------|
| Name | Must match class name | Any valid identifier |
| Return type | None (not even void) | Required |
| Invocation | Automatically via `new` | Explicitly called |
| Inheritance | NOT inherited | Inherited |
| `this()`/`super()` | First statement only | Not applicable |
| Purpose | Initialize object | Define behavior |

### Private Constructors

#### Singleton Pattern

```java
public class AppConfig {
    private static final AppConfig INSTANCE = new AppConfig();

    // ✅ Private constructor prevents external instantiation
    private AppConfig() {
        // Load configuration
    }

    public static AppConfig getInstance() {
        return INSTANCE;
    }
}
```

#### Utility Class

```java
public final class MathUtils {
    // ✅ Private constructor + final class = no instantiation, no subclassing
    private MathUtils() {
        throw new AssertionError("Utility class - do not instantiate");
    }

    public static double circleArea(double radius) {
        return Math.PI * radius * radius;
    }
}
```

### Constructor Overloading

```java
public class Connection {
    private String host;
    private int port;
    private int timeout;
    private boolean ssl;

    public Connection(String host) {
        this(host, 8080);
    }

    public Connection(String host, int port) {
        this(host, port, 30000);
    }

    public Connection(String host, int port, int timeout) {
        this(host, port, timeout, false);
    }

    public Connection(String host, int port, int timeout, boolean ssl) {
        this.host = host;
        this.port = port;
        this.timeout = timeout;
        this.ssl = ssl;
    }
}
```

> **Tip**: When you have many optional parameters, prefer the **Builder pattern** over telescoping constructors.

### Common Mistake

```java
// ❌ Calling this() and super() in the same constructor
public class Child extends Parent {
    public Child() {
        super();   // First statement
        this(10);  // COMPILE ERROR - cannot have both
    }
}

// ❌ Defining a return type makes it a method, not a constructor
public class Foo {
    void Foo() {  // This is a METHOD named Foo, NOT a constructor!
        System.out.println("This is a method, not a constructor");
    }
}
```

### Performance Tip
Avoid heavy computation or I/O in constructors. Constructors should focus on initialization. Use factory methods or `@PostConstruct` (in Spring) for complex setup logic.

---

## Interviewer Insight
Interviewers check if you know the difference between constructors and methods, understand chaining with `this()` and `super()`, and can explain private constructors for design patterns like Singleton and utility classes.

## Follow-up Questions
1. Can a constructor be `final`, `abstract`, or `static`?
2. What happens if you create a circular constructor chain with `this()`?
3. How does the Builder pattern solve the telescoping constructor problem?

## Related Topics
- [Classes and Objects](./q01-classes-and-objects.md)
- [Inheritance](./q03-inheritance.md)
- [Super Keyword](./q10-super-keyword.md)
