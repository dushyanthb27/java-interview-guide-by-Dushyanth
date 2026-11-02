# Super Keyword

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Super Keyword

---

## Q13 🟢 Explain the Super Keyword

**Question** — What is the `super` keyword in Java? Explain its uses for constructor chaining, method access, field access, and how it works with interfaces since Java 8.

### Answer

The `super` keyword in Java is a reference to the **immediate parent class**. It serves three primary purposes — invoking parent constructors, calling overridden methods, and accessing hidden parent fields.

**`super()` for Constructor Chaining**

Use `super()` to call a parent constructor. It must be the **first statement** in the constructor body. If you don't call `super()` explicitly, the compiler inserts `super()` (no-arg) automatically.

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        System.out.println("Person constructor called");
    }

    public String getName() { return name; }
    public int getAge() { return age; }
}

public class Student extends Person {
    private String university;
    private double gpa;

    public Student(String name, int age, String university, double gpa) {
        super(name, age); // Must be FIRST statement in constructor
        this.university = university;
        this.gpa = gpa;
        System.out.println("Student constructor called");
    }

    // Chaining: no-arg constructor calls parameterized
    public Student() {
        this("Unknown", 0, "Unassigned", 0.0); // Calls the 4-arg constructor
    }
}
```

```java
// Output when creating: new Student("Alice", 20, "MIT", 3.9)
// Person constructor called
// Student constructor called
```

**Important Rules** — `super()` must be the **first statement** in the constructor. If you don't call `super()` explicitly, the compiler inserts `super()` (no-arg) automatically. You **cannot** use both `this()` and `super()` in the same constructor.

```java
// Compiler error - no-arg parent constructor doesn't exist
public class Employee extends Person {
    public Employee() {
        // Implicit super() fails because Person has no no-arg constructor
        // COMPILE ERROR: constructor Person() not found
    }
}

// Fix: explicitly call the available parent constructor
public class Employee extends Person {
    public Employee() {
        super("Default", 0); // Explicit call
    }
}
```

**`super.method()` for Accessing Overridden Methods**

Call the parent's version of an overridden method using `super.methodName()`. This is useful when you want to extend rather than replace the parent's behavior.

```java
public class Logger {
    public void log(String message) {
        System.out.println("[LOG] " + message);
    }
}

public class TimestampLogger extends Logger {
    @Override
    public void log(String message) {
        String timestamp = java.time.LocalDateTime.now().toString();
        // Call the parent's version, then add behavior
        super.log("[" + timestamp + "] " + message);
    }
}

public class FileLogger extends TimestampLogger {
    @Override
    public void log(String message) {
        super.log(message); // Calls TimestampLogger.log()
        writeToFile(message);
    }

    private void writeToFile(String message) {
        // Write to file implementation
    }
}
```

**`super.field` for Accessing Hidden Fields**

When a subclass declares a field with the same name as a parent field, the subclass field **hides** the parent field. Use `super.fieldName` to access the hidden parent field.

```java
public class Shape {
    protected String color = "White";

    public void printColor() {
        System.out.println("Shape color: " + color);
    }
}

public class Circle extends Shape {
    protected String color = "Red"; // Hides parent's color field

    public void printColors() {
        System.out.println("Circle color: " + this.color);   // "Red"
        System.out.println("Shape color: " + super.color);    // "White"
    }
}
```

Field hiding is generally a bad practice. Prefer different names or encapsulation.

**`super` with Interfaces (Java 8+ Default Methods)**

When implementing multiple interfaces with conflicting default methods, use `InterfaceName.super.method()` to explicitly choose which implementation to call.

```java
public interface Greetable {
    default String greet() {
        return "Hello!";
    }
}

public interface Farewell {
    default String greet() {
        return "Goodbye!";
    }
}

// When implementing multiple interfaces with conflicting defaults:
public class Polite implements Greetable, Farewell {
    @Override
    public String greet() {
        // Use InterfaceName.super.method() to choose
        String greeting = Greetable.super.greet();
        String farewell = Farewell.super.greet();
        return greeting + " and " + farewell;
    }
}
```

**Constructor Chaining Order**

Constructors execute from the topmost parent down to the current class. This ensures the parent is fully initialized before the child.

```java
public class A {
    public A() { System.out.println("A constructor"); }
}

public class B extends A {
    public B() {
        super(); // Calls A()
        System.out.println("B constructor");
    }
}

public class C extends B {
    public C() {
        super(); // Calls B()
        System.out.println("C constructor");
    }
}

// new C() outputs:
// A constructor   ← topmost parent first
// B constructor
// C constructor   ← current class last
```

**Real-World Example — Spring-Style Component**

```java
public abstract class AbstractService {
    private boolean initialized = false;

    public void init() {
        System.out.println("Base initialization: loading config");
        this.initialized = true;
    }

    public boolean isInitialized() { return initialized; }
}

public class UserService extends AbstractService {
    private UserRepository repository;

    @Override
    public void init() {
        super.init(); // Ensure parent initialization runs first
        this.repository = new UserRepository();
        System.out.println("UserService: repository initialized");
    }
}

public class AdminService extends UserService {
    @Override
    public void init() {
        super.init(); // Chains up: AdminService → UserService → AbstractService
        System.out.println("AdminService: admin permissions loaded");
    }
}
```

> **⚠️ Common Mistake** — Trying to use `super` in a static context, trying to skip a level in the hierarchy with `super.super`, or forgetting that implicit `super()` calls the no-arg constructor.

```java
// Trying to use super in a static context
public class Child extends Parent {
    public static void staticMethod() {
        // super.someMethod();  // COMPILE ERROR: super cannot be used in static context
    }
}

// Trying to skip a level in the hierarchy
public class Grandchild extends Child {
    public void method() {
        // super.super.method();  // COMPILE ERROR: cannot chain super
        super.method(); // Can only access immediate parent
    }
}

// Forgetting super() in constructor causes implicit no-arg call
public class Derived extends Base {
    public Derived(int x) {
        // If Base has no no-arg constructor, this fails
        // Always explicitly call super(...) when parent has only parameterized constructors
    }
}
```

> **🔧 Performance Tip** — Calls through `super` are resolved at compile time (static dispatch), so `super.method()` is slightly faster than a regular virtual method call. However, this difference is negligible and should not influence design decisions.

> **🎯 Interviewer Insight** — Interviewers test whether you understand the constructor chain from child to parent, the difference between `this()` and `super()`, and the restriction that both cannot coexist. The Java 8+ `InterfaceName.super.method()` syntax is a frequent advanced question.

> **⚡ Follow-up** — What happens if neither `this()` nor `super()` is used in a constructor?

> **⚡ Follow-up** — Can `super` be used in a static method? Why not?

> **⚡ Follow-up** — How does `super` work in the context of the Diamond Problem?

> **📚 Related** — [This Keyword](./11-this-keyword.md), [Constructors](./02-constructors.md), [Diamond Problem](./18-diamond-problem.md)

---

[← Previous: Method Overriding](./09-method-overriding.md) · [Next: This Keyword →](./11-this-keyword.md)
