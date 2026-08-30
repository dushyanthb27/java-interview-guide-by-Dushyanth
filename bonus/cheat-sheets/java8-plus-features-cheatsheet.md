# Java 8+ Features - Cheat Sheet

Quick reference for modern Java features from Java 8 to Java 21.

## 🔧 Java 8 Features

### Lambda Expressions
```java
// Traditional anonymous class
button.setOnClickListener(new OnClickListener() {
    public void onClick(View v) {
        System.out.println("Clicked");
    }
});

// Lambda expression
button.setOnClickListener(v -> System.out.println("Clicked"));

// Lambda with multiple parameters
Comparator<String> comp = (a, b) -> a.compareTo(b);

// Lambda with multiple statements
Runnable r = () -> {
    System.out.println("Line 1");
    System.out.println("Line 2");
};

// Lambda with explicit types
BiFunction<Integer, Integer, Integer> add = (Integer a, Integer b) -> a + b;

// Syntax rules:
// - Single parameter: x -> x * 2
// - Multiple parameters: (x, y) -> x + y
// - No parameters: () -> value
// - Multiple statements: { statement1; statement2; }
```

### Functional Interfaces
```java
// Functional interface - single abstract method
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);
}

// Implementation with lambda
Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;

// Built-in functional interfaces
Function<Integer, Integer> square = x -> x * x;
Predicate<Integer> isPositive = x -> x > 0;
Consumer<String> print = System.out::println;
Supplier<String> greeting = () -> "Hello";
BiFunction<Integer, Integer, Integer> sum = (a, b) -> a + b;

// Custom functional interface
@FunctionalInterface
public interface StringProcessor {
    String process(String input);
    
    // Can have default and static methods
    default String processWithPrefix(String input) {
        return "PREFIX: " + process(input);
    }
}
```

### Method References
```java
// Reference to static method
Function<String, Integer> parseInt = Integer::parseInt;

// Reference to instance method
String str = "hello";
Supplier<Integer> length = str::length;

// Reference to constructor
Function<String, Integer> intConstructor = Integer::new;
BiFunction<String, Integer, String> substring = String::substring;

// Reference to instance method of parameter
Function<String, String> toUpperCase = String::toUpperCase;

// Syntax:
// - Static method: ClassName::methodName
// - Instance method: instance::methodName
// - Constructor: ClassName::new
```

### Stream API (See stream-api-cheatsheet.md)
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Filter, map, collect
List<Integer> doubled = numbers.stream()
    .filter(n -> n > 2)
    .map(n -> n * 2)
    .collect(Collectors.toList());

// Reduce
int sum = numbers.stream()
    .reduce(0, Integer::sum);

// Group by
Map<Integer, List<Integer>> grouped = numbers.stream()
    .collect(Collectors.groupingBy(n -> n % 2));
```

### Default Methods in Interfaces
```java
// Interface with default method
public interface Animal {
    void eat();
    
    default void sleep() {
        System.out.println("Zzz...");
    }
}

// Implementation
public class Dog implements Animal {
    @Override
    public void eat() {
        System.out.println("Eating");
    }
    // sleep() inherited from interface
}

// Multiple inheritance of default methods
public interface Swimmer {
    default void swim() {
        System.out.println("Swimming");
    }
}

public class Duck implements Animal, Swimmer {
    @Override
    public void eat() { }
    
    // Must override if both interfaces have same default method
    @Override
    public void swim() {
        Swimmer.super.swim();  // Call specific interface
    }
}
```

### Optional
```java
// Create Optional
Optional<String> empty = Optional.empty();
Optional<String> value = Optional.of("hello");
Optional<String> nullable = Optional.ofNullable(null);

// Check and get
if (value.isPresent()) {
    System.out.println(value.get());
}

// Functional operations
value.ifPresent(System.out::println);
value.ifPresentOrElse(
    System.out::println,
    () -> System.out.println("Empty")
);

// Transform
Optional<Integer> length = value.map(String::length);

// Filter
Optional<String> filtered = value.filter(s -> s.length() > 2);

// Flat map
Optional<Integer> result = value.flatMap(s -> Optional.of(s.length()));

// Get with default
String result = value.orElse("default");
String result2 = value.orElseGet(() -> "default");
String result3 = value.orElseThrow(() -> new Exception("Not found"));
```

## 📦 Java 9 Features

### Module System
```java
// module-info.java
module com.example.app {
    requires java.base;
    requires java.logging;
    
    exports com.example.api;
    exports com.example.utils;
    
    opens com.example.internal;
}

// Run with modules
java --module-path mods -m com.example.app/com.example.Main
```

### Private Interface Methods
```java
public interface Processor {
    void process();
    
    // Private method (Java 9+)
    private void validate() {
        System.out.println("Validating");
    }
    
    default void processWithValidation() {
        validate();
        process();
    }
}
```

### Try-With-Resources Enhancement
```java
// Java 7-8
try (FileReader reader = new FileReader("file.txt")) {
    // Use reader
}

// Java 9+ - can reference external variable
FileReader reader = new FileReader("file.txt");
try (reader) {
    // Use reader
}
```

### Diamond Operator with Anonymous Classes
```java
// Java 8 - not allowed
// List<String> list = new ArrayList<>() { };

// Java 9+ - allowed
List<String> list = new ArrayList<>() {
    @Override
    public String toString() {
        return "Custom list";
    }
};
```

## 🎁 Java 10 Features

### Local Variable Type Inference
```java
// Traditional
List<String> list = new ArrayList<>();

// Java 10+ - var keyword
var list = new ArrayList<String>();
var name = "John";
var age = 25;

// Works with streams
var numbers = Arrays.asList(1, 2, 3);
var doubled = numbers.stream()
    .map(n -> n * 2)
    .collect(Collectors.toList());

// Restrictions:
// - Only for local variables
// - Must have initializer
// - Cannot be used in method parameters or return types
```

## 📝 Java 11 Features

### String Methods
```java
String str = "  hello  ";

// Strip whitespace
str.strip();           // "hello"
str.stripLeading();    // "hello  "
str.stripTrailing();   // "  hello"

// Repeat
"ab".repeat(3);        // "ababab"

// Is blank
"   ".isBlank();       // true
"hello".isBlank();     // false

// Lines
String multiline = "line1\nline2\nline3";
multiline.lines().forEach(System.out::println);
```

### File Operations
```java
// Read all lines
List<String> lines = Files.readAllLines(Paths.get("file.txt"));

// Read string
String content = Files.readString(Paths.get("file.txt"));

// Write string
Files.writeString(Paths.get("file.txt"), "content");
```

### Predicate.not()
```java
List<String> list = Arrays.asList("a", "b", "c");

// Java 10
list.stream()
    .filter(s -> !s.equals("a"))
    .forEach(System.out::println);

// Java 11
list.stream()
    .filter(Predicate.not(s -> s.equals("a")))
    .forEach(System.out::println);
```

## 🎯 Java 14 Features

### Records
```java
// Traditional class
public class Person {
    private final String name;
    private final int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    @Override
    public String toString() { /* ... */ }
    
    @Override
    public boolean equals(Object o) { /* ... */ }
    
    @Override
    public int hashCode() { /* ... */ }
}

// Record (Java 14+)
public record Person(String name, int age) { }

// Usage
Person p = new Person("John", 30);
System.out.println(p.name());
System.out.println(p.age());

// With custom methods
public record Person(String name, int age) {
    public Person {
        if (age < 0) throw new IllegalArgumentException("Age must be positive");
    }
    
    public String fullInfo() {
        return name + " (" + age + ")";
    }
}
```

## 🔒 Java 15 Features

### Sealed Classes
```java
// Sealed class - can only be extended by specific classes
public sealed class Animal permits Dog, Cat {
    public abstract void sound();
}

// Permitted subclass
public final class Dog extends Animal {
    @Override
    public void sound() {
        System.out.println("Woof");
    }
}

// Permitted subclass
public final class Cat extends Animal {
    @Override
    public void sound() {
        System.out.println("Meow");
    }
}

// Not permitted - compilation error
// public class Bird extends Animal { }
```

### Text Blocks
```java
// Traditional string
String json = "{\"name\": \"John\", \"age\": 30}";

// Text block (Java 13+)
String json = """
    {
        "name": "John",
        "age": 30
    }
    """;

// With expressions (Java 21+)
String message = """
    Hello, %s!
    """.formatted(name);
```

## 🚀 Java 16+ Features

### Pattern Matching for instanceof
```java
// Traditional
if (obj instanceof String) {
    String str = (String) obj;
    System.out.println(str.length());
}

// Java 16+
if (obj instanceof String str) {
    System.out.println(str.length());
}

// With pattern matching
if (obj instanceof String str && str.length() > 5) {
    System.out.println("Long string: " + str);
}
```

### Records with Sealed Classes
```java
// Sealed interface with record implementations
public sealed interface Shape permits Circle, Rectangle {
    double area();
}

public record Circle(double radius) implements Shape {
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public record Rectangle(double width, double height) implements Shape {
    @Override
    public double area() {
        return width * height;
    }
}
```

## 📊 Feature Comparison Table

| Feature | Java Version | Use Case |
|---------|--------------|----------|
| Lambda | 8 | Functional programming |
| Streams | 8 | Data processing |
| Optional | 8 | Null handling |
| Modules | 9 | Large applications |
| var | 10 | Local variables |
| Records | 14 | Data classes |
| Sealed classes | 15 | Type hierarchies |
| Text blocks | 13 | Multi-line strings |
| Pattern matching | 16 | Type checking |

## 🎓 Best Practices

### Use var Wisely
```java
// ✅ Good - type is obvious
var list = new ArrayList<String>();

// ❌ Bad - type is not obvious
var result = processData();

// ✅ Good - clear from context
var numbers = Arrays.asList(1, 2, 3);
```

### Prefer Records for Data Classes
```java
// ❌ Old way
public class Point {
    private final int x;
    private final int y;
    // ... boilerplate
}

// ✅ Modern way
public record Point(int x, int y) { }
```

### Use Sealed Classes for Type Safety
```java
// ✅ Good - clear hierarchy
public sealed class Result permits Success, Failure { }

// ❌ Bad - open to extension
public class Result { }
```

### Leverage Pattern Matching
```java
// ✅ Modern
if (obj instanceof String str) {
    System.out.println(str.length());
}

// ❌ Old way
if (obj instanceof String) {
    String str = (String) obj;
    System.out.println(str.length());
}
```

## ⚠️ Common Pitfalls

1. **Overusing var**
   ```java
   // ❌ Wrong - unclear type
   var x = someComplexMethod();
   
   // ✅ Correct - clear type
   var list = new ArrayList<String>();
   ```

2. **Records with mutable fields**
   ```java
   // ❌ Wrong - records should be immutable
   public record Person(String name, List<String> hobbies) { }
   
   // ✅ Correct - use immutable collections
   public record Person(String name, List<String> hobbies) { }
   ```

3. **Sealed classes too restrictive**
   ```java
   // ❌ Wrong - too many permitted classes
   public sealed class Base permits A, B, C, D, E, F { }
   
   // ✅ Correct - reasonable number
   public sealed class Base permits A, B, C { }
   ```

## 🎓 Interview Tips

- **Know the timeline**: Java 8 (lambda), 11 (var), 14 (records), 15 (sealed)
- **Understand trade-offs**: Conciseness vs readability
- **Know when to use**: Records for data, sealed for type safety
- **Pattern matching**: Simplifies type checking and casting
- **Modern Java style**: Use var, records, and sealed classes appropriately

---

**Related**: See Stream API cheat sheet for functional operations
