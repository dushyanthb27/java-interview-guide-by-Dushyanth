# Core Java Terminology

Comprehensive reference for fundamental Java language concepts, features, and standard library terms. This glossary covers 300+ essential Java terms organized alphabetically.

---

## A

### Abstract Class
**Category**: OOP Fundamentals

**Definition**: A class declared with the `abstract` keyword that cannot be instantiated directly but serves as a blueprint for subclasses. Abstract classes can contain both abstract methods (without implementation) and concrete methods (with implementation), allowing partial implementation of functionality.

**Practical Example**:
```java
abstract class Animal {
    abstract void makeSound();
    
    void sleep() {
        System.out.println("Zzz...");
    }
}

class Dog extends Animal {
    @Override
    void makeSound() {
        System.out.println("Woof!");
    }
}
```

**Related Terms**: [Interface](#interface), [Inheritance](#inheritance), [Polymorphism](#polymorphism)

**Interview Tip**: Understand the difference between abstract classes and interfaces. Abstract classes are used when you want to share code among closely related classes, while interfaces define contracts for unrelated classes.

---

## B

### Boolean
**Category**: Data Types

**Definition**: A primitive data type that can hold only two values: `true` or `false`. Booleans are used in conditional statements and logical operations.

**Practical Example**:
```java
boolean isActive = true;
boolean isEmpty = false;

if (isActive && !isEmpty) {
    System.out.println("Active and not empty");
}

boolean result = (5 > 3) || (2 < 1);  // true
```

**Related Terms**: [Data Type](#data-type), [Logical Operator](#logical-operator), [Conditional](#conditional)

**Interview Tip**: Remember that Java's boolean is not the same as integers (unlike C). You cannot use 0/1 as boolean values.

---

## C

### Class
**Category**: OOP Fundamentals

**Definition**: A blueprint or template for creating objects that defines the structure (attributes) and behavior (methods) of objects. Classes are the fundamental building blocks of object-oriented programming in Java.

**Practical Example**:
```java
public class Person {
    // Attributes
    private String name;
    private int age;
    
    // Constructor
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Methods
    public void introduce() {
        System.out.println("I am " + name);
    }
}

// Creating an object
Person person = new Person("Alice", 30);
```

**Related Terms**: [Object](#object), [Inheritance](#inheritance), [Encapsulation](#encapsulation), [Polymorphism](#polymorphism)

**Interview Tip**: Understand the difference between a class (blueprint) and an object (instance). Know about class variables, instance variables, and methods.

---

## D

### Data Type
**Category**: Core Language Features

**Definition**: A classification of data that determines what values a variable can hold and what operations can be performed on it. Java has primitive types (int, double, boolean, etc.) and reference types (classes, interfaces, arrays).

**Practical Example**:
```java
// Primitive types
int age = 25;
double salary = 50000.50;
boolean isActive = true;
char grade = 'A';

// Reference types
String name = "Alice";
List<String> items = new ArrayList<>();
Object obj = new Object();
```

**Related Terms**: [Primitive Type](#primitive-type), [Reference Type](#reference-type), [Type System](#type-system), [Type Conversion](#type-conversion)

**Interview Tip**: Know the differences between primitive and reference types. Understand how they are stored in memory (stack vs. heap).

---

## E

### Encapsulation
**Category**: OOP Principles

**Definition**: A fundamental OOP principle that bundles data (attributes) and methods that operate on that data into a single unit (class) and hides internal details from the outside world. Access is controlled through public methods.

**Practical Example**:
```java
public class BankAccount {
    private double balance;  // Hidden from outside
    
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
    
    public double getBalance() {
        return balance;
    }
}

BankAccount account = new BankAccount();
account.deposit(100);
// account.balance = -50;  // Not allowed - private
```

**Related Terms**: [Access Modifiers](#access-modifiers), [Abstraction](#abstraction), [Information Hiding](#information-hiding), [OOP](#oop)

**Interview Tip**: Explain how encapsulation improves maintainability and allows internal implementation changes without affecting external code.

---

## F

### Final
**Category**: Modifiers

**Definition**: A keyword that prevents modification of a class, method, or variable. Final classes cannot be extended, final methods cannot be overridden, and final variables cannot be reassigned after initialization.

**Practical Example**:
```java
// Final class - cannot be extended
final class ImmutableClass {
    // Final method - cannot be overridden
    final void criticalMethod() {
        // Implementation
    }
}

// Final variable - cannot be reassigned
final int MAX_SIZE = 100;
final String name = "Alice";

// Attempting to modify
// MAX_SIZE = 200;  // Compile error
```

**Related Terms**: [Constant](#constant), [Immutable](#immutable), [Inheritance](#inheritance), [Method Overriding](#method-overriding)

**Interview Tip**: Understand the performance implications of final. Know when to use final for security and optimization.

---

## G

### Garbage Collection
**Category**: Memory Management

**Definition**: An automatic memory management process where the JVM identifies and frees memory occupied by objects that are no longer referenced. Garbage collection eliminates the need for manual memory deallocation.

**Practical Example**:
```java
// Objects eligible for garbage collection
Object obj1 = new Object();
Object obj2 = new Object();

obj1 = null;  // obj1 is now eligible for GC
obj2 = new Object();  // Old obj2 is eligible for GC

// Requesting garbage collection (not guaranteed)
System.gc();
```

**Related Terms**: [Memory Management](#memory-management), [Heap](#heap), [Reference](#reference), [Finalize](#finalize)

**Interview Tip**: Understand different GC algorithms (G1GC, ZGC, etc.). Know about GC tuning and monitoring.

---

## H

### Hashcode
**Category**: Object Methods

**Definition**: An integer value computed from an object's state that represents its hash code. Objects with the same content should have the same hash code. Hash codes are used in hash-based collections like HashMap and HashSet.

**Practical Example**:
```java
class Person {
    private String name;
    private int age;
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person person = (Person) obj;
        return age == person.age && Objects.equals(name, person.name);
    }
}

Map<Person, String> map = new HashMap<>();
map.put(new Person("Alice", 30), "Engineer");
```

**Related Terms**: [Equals](#equals), [HashMap](#hashmap), [HashSet](#hashset), [Object](#object)

**Interview Tip**: Know the contract between hashCode() and equals(). Understand hash collisions and their impact on performance.

---

## I

### Immutable
**Category**: Design Principles

**Definition**: An object whose state cannot be changed after creation. Immutable objects are inherently thread-safe and can be safely shared between threads without synchronization.

**Practical Example**:
```java
final class ImmutablePerson {
    private final String name;
    private final int age;
    
    public ImmutablePerson(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;
    }
    
    // No setters - state cannot change
}

ImmutablePerson person = new ImmutablePerson("Alice", 30);
// person.age = 31;  // Not possible
```

**Related Terms**: [Final](#final), [Thread Safety](#thread-safety), [Immutability](#immutability), [String](#string)

**Interview Tip**: Discuss benefits of immutability: thread safety, caching, hashCode stability. Know how to create immutable classes.

---

## J

### Java Virtual Machine (JVM)
**Category**: Runtime Environment

**Definition**: An abstract computing machine that enables a computer to run Java programs and programs written in other languages that are compiled to Java bytecode. The JVM provides platform independence through bytecode interpretation and compilation.

**Practical Example**:
```java
// Java source code
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// Compiled to bytecode (.class file)
// JVM executes the bytecode
// java HelloWorld
```

**Related Terms**: [Bytecode](#bytecode), [Compilation](#compilation), [Platform Independence](#platform-independence), [JIT Compiler](#jit-compiler)

**Interview Tip**: Understand the role of the JVM in achieving platform independence. Know about JVM memory areas and garbage collection.

---

## K

### Keyword
**Category**: Language Features

**Definition**: A reserved word in Java that has a special meaning and cannot be used as an identifier. Keywords include control flow (if, else, for, while), type declarations (class, interface, enum), and modifiers (public, private, static, final).

**Practical Example**:
```java
// Keywords in Java
public class MyClass {  // public, class are keywords
    private int value;   // private is a keyword
    
    public void method() {  // public, void are keywords
        if (value > 0) {    // if is a keyword
            System.out.println("Positive");
        }
    }
}
```

**Related Terms**: [Identifier](#identifier), [Reserved Word](#reserved-word), [Modifier](#modifier)

**Interview Tip**: Know the complete list of Java keywords. Understand the difference between keywords and identifiers.

---

## L

### Lambda Expression
**Category**: Functional Programming

**Definition**: A concise way to represent an anonymous function (a function without a name) that can be passed around as a value. Lambda expressions enable functional programming in Java and are typically used with functional interfaces.

**Pronunciation**: LAM-duh

**Practical Example**:
```java
// Traditional anonymous class
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};

// Lambda expression (Java 8+)
Runnable task2 = () -> System.out.println("Running");

// Lambda with parameters
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
numbers.forEach(n -> System.out.println(n));

// Lambda with multiple statements
Comparator<String> comparator = (s1, s2) -> {
    System.out.println("Comparing");
    return s1.compareTo(s2);
};
```

**Related Terms**: [Functional Interface](#functional-interface), [Anonymous Class](#anonymous-class), [Method Reference](#method-reference), [Functional Programming](#functional-programming)

**Interview Tip**: Understand lambda syntax and when to use lambdas vs. anonymous classes. Know about lambda variable capture and effectively final variables.

---

## M

### Method
**Category**: OOP Fundamentals

**Definition**: A function defined inside a class that performs a specific action or behavior. Methods can accept parameters, return values, and can be overloaded (same name with different parameters).

**Practical Example**:
```java
class Calculator {
    // Method with parameters and return value
    public int add(int a, int b) {
        return a + b;
    }
    
    // Method without parameters
    public void displayInfo() {
        System.out.println("Calculator");
    }
    
    // Overloaded methods (same name, different parameters)
    public double add(double a, double b) {
        return a + b;
    }
}

Calculator calc = new Calculator();
int result = calc.add(5, 3);  // 8
```

**Related Terms**: [Function](#function), [Method Overloading](#method-overloading), [Method Overriding](#method-overriding), [Parameter](#parameter)

**Interview Tip**: Understand method signatures, return types, and parameter passing. Know the difference between pass-by-value and pass-by-reference.

---

## N

### Null
**Category**: Reference Types

**Definition**: A special value that represents the absence of an object reference. A variable can be assigned null to indicate it doesn't reference any object. Attempting to call methods on null references throws NullPointerException.

**Practical Example**:
```java
String str = null;  // null reference
System.out.println(str);  // null

// NullPointerException
// System.out.println(str.length());  // Throws NPE

// Null checking
if (str != null) {
    System.out.println(str.length());
}

// Optional (Java 8+) as alternative to null
Optional<String> optional = Optional.ofNullable(str);
optional.ifPresent(s -> System.out.println(s.length()));
```

**Related Terms**: [Reference](#reference), [NullPointerException](#nullpointerexception), [Optional](#optional), [Null Safety](#null-safety)

**Interview Tip**: Discuss null handling best practices. Know about Optional and null-safe operations. Understand the billion-dollar mistake (null references).

---

## O

### Object
**Category**: OOP Fundamentals

**Definition**: An instance of a class that contains state (data) and behavior (methods). Objects are created using the `new` keyword and are the fundamental units of object-oriented programming.

**Practical Example**:
```java
class Person {
    String name;
    int age;
    
    void introduce() {
        System.out.println("I am " + name);
    }
}

// Creating objects
Person person1 = new Person();
person1.name = "Alice";
person1.age = 30;

Person person2 = new Person();
person2.name = "Bob";
person2.age = 25;

// Each object has its own state
person1.introduce();  // "I am Alice"
person2.introduce();  // "I am Bob"
```

**Related Terms**: [Class](#class), [Instance](#instance), [State](#state), [Behavior](#behavior)

**Interview Tip**: Understand the relationship between classes and objects. Know about object identity vs. equality.

---

## P

### Polymorphism
**Category**: OOP Principles

**Definition**: The ability of objects to take on multiple forms or for methods to behave differently based on the context. Java supports compile-time (method overloading) and runtime (method overriding) polymorphism.

**Pronunciation**: puh-LIM-or-fiz-um

**Practical Example**:
```java
// Runtime polymorphism (method overriding)
abstract class Animal {
    abstract void makeSound();
}

class Dog extends Animal {
    @Override
    void makeSound() {
        System.out.println("Woof!");
    }
}

class Cat extends Animal {
    @Override
    void makeSound() {
        System.out.println("Meow!");
    }
}

// Polymorphic behavior
Animal dog = new Dog();
Animal cat = new Cat();

dog.makeSound();  // "Woof!"
cat.makeSound();  // "Meow!"

// Compile-time polymorphism (method overloading)
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
}
```

**Related Terms**: [Inheritance](#inheritance), [Method Overriding](#method-overriding), [Method Overloading](#method-overloading), [Abstraction](#abstraction)

**Interview Tip**: Explain the difference between compile-time and runtime polymorphism. Discuss how polymorphism enables flexible and extensible code.

---

## S

### Static
**Category**: Modifiers

**Definition**: A keyword that indicates a member (variable, method, or block) belongs to the class itself rather than to instances of the class. Static members are shared among all instances and can be accessed without creating an object.

**Practical Example**:
```java
class Counter {
    private static int count = 0;  // Static variable
    private int id;
    
    public Counter() {
        count++;
        id = count;
    }
    
    public static int getCount() {  // Static method
        return count;
    }
    
    static {  // Static initializer block
        System.out.println("Counter class loaded");
    }
}

Counter c1 = new Counter();
Counter c2 = new Counter();
System.out.println(Counter.getCount());  // 2
```

**Related Terms**: [Instance](#instance), [Class Variable](#class-variable), [Class Method](#class-method), [Static Initializer](#static-initializer)

**Interview Tip**: Understand the difference between static and instance members. Know about static initialization order and common pitfalls.

---

## T

### Thread
**Category**: Concurrency

**Definition**: The smallest unit of execution within a process. Java supports multithreading, allowing multiple threads to execute concurrently within the same program, sharing the same memory space but maintaining separate execution stacks.

**Practical Example**:
```java
// Creating threads
class MyThread extends Thread {
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);
        }
    }
}

MyThread thread = new MyThread();
thread.start();  // Starts the thread

// Using Runnable
Thread thread2 = new Thread(() -> {
    System.out.println("Running in thread");
});
thread2.start();

// Main thread
System.out.println("Main thread");
```

**Related Terms**: [Concurrency](#concurrency), [Multithreading](#multithreading), [Synchronization](#synchronization), [Runnable](#runnable)

**Interview Tip**: Understand thread lifecycle (new, runnable, running, waiting, terminated). Know about thread safety and synchronization.

---

## V

### Variable
**Category**: Core Language Features

**Definition**: A named container that stores a value of a specific data type. Variables can be primitive types (int, double, boolean) or reference types (objects, arrays). Variables must be declared before use.

**Practical Example**:
```java
// Primitive variable
int age = 25;
double salary = 50000.50;
boolean isActive = true;

// Reference variable
String name = "Alice";
List<String> items = new ArrayList<>();
Person person = new Person();

// Variable scope
{
    int localVar = 10;  // Local variable
    System.out.println(localVar);
}
// localVar is not accessible here

// Static variable
class MyClass {
    static int staticVar = 0;  // Shared among all instances
}

// Instance variable
class MyClass {
    int instanceVar = 0;  // Unique to each instance
}
```

**Related Terms**: [Data Type](#data-type), [Scope](#scope), [Instance Variable](#instance-variable), [Static Variable](#static-variable)

**Interview Tip**: Know the different types of variables and their scope. Understand variable initialization and default values.

---

## W

### Wrapper Class
**Category**: Type System

**Definition**: A class that wraps a primitive data type and provides object-oriented functionality. Java provides wrapper classes for each primitive type: Integer, Double, Boolean, Character, etc. They enable primitives to be used in collections and provide utility methods.

**Practical Example**:
```java
// Wrapper classes
Integer intWrapper = 5;              // Autoboxing
Double doubleWrapper = 3.14;
Boolean boolWrapper = true;

// Unboxing
int intValue = intWrapper;           // Unboxing
double doubleValue = doubleWrapper;

// Using wrapper classes
List<Integer> numbers = new ArrayList<>();
numbers.add(10);  // Autoboxing

// Utility methods
int max = Integer.max(5, 10);
String binary = Integer.toBinaryString(5);
Integer parsed = Integer.parseInt("123");

// Caching
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (cached)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (not cached)
```

**Related Terms**: [Primitive Type](#primitive-type), [Autoboxing](#autoboxing), [Unboxing](#unboxing), [Boxing](#boxing)

**Interview Tip**: Know the difference between wrapper classes and primitives. Understand autoboxing/unboxing and integer caching. Know when to use each.

---

## Additional Key Terms (Continued)

### Abstract Method
**Category**: OOP Fundamentals

**Definition**: A method declared with the `abstract` keyword in an abstract class that has no implementation body. Subclasses must provide concrete implementations of abstract methods.

**Practical Example**:
```java
abstract class Shape {
    abstract double calculateArea();
    abstract double calculatePerimeter();
}

class Circle extends Shape {
    private double radius;
    
    @Override
    double calculateArea() {
        return Math.PI * radius * radius;
    }
    
    @Override
    double calculatePerimeter() {
        return 2 * Math.PI * radius;
    }
}
```

**Related Terms**: [Abstract Class](#abstract-class), [Method Overriding](#method-overriding), [Interface](#interface)

**Interview Tip**: Be prepared to explain why abstract methods are useful for enforcing contracts.

---

### Array
**Category**: Data Structures

**Definition**: A collection of elements of the same type stored in contiguous memory locations, accessed by index. Arrays have a fixed size determined at creation time and are zero-indexed in Java.

**Practical Example**:
```java
int[] numbers = new int[5];           // Array of 5 integers
String[] names = {"Alice", "Bob"};    // Array with initial values
int[][] matrix = new int[3][3];       // 2D array

// Accessing elements
numbers[0] = 10;
String firstName = names[0];
```

**Related Terms**: [List](#list), [Collection](#collection), [ArrayList](#arraylist)

**Interview Tip**: Know the difference between arrays and dynamic collections like ArrayList.

---

### Constructor
**Category**: OOP Fundamentals

**Definition**: A special method that initializes an object when it is created. Constructors have the same name as the class, no return type, and can be overloaded.

**Practical Example**:
```java
class Person {
    private String name;
    private int age;
    
    // No-argument constructor
    public Person() {
        this("Unknown", 0);
    }
    
    // Constructor with parameters
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

Person p1 = new Person();
Person p2 = new Person("Alice", 30);
```

**Related Terms**: [Class](#class), [Object](#object), [Initialization](#initialization), [Method Overloading](#method-overloading)

**Interview Tip**: Understand constructor chaining with `this()` and `super()`.

---

### Exception
**Category**: Exception Handling

**Definition**: An event that occurs during program execution that disrupts the normal flow. Exceptions are objects that represent errors and can be caught and handled using try-catch blocks.

**Practical Example**:
```java
try {
    int result = 10 / 0;  // Throws ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero");
} catch (Exception e) {
    System.out.println("Generic exception");
}

// Throwing exceptions
if (age < 0) {
    throw new IllegalArgumentException("Age cannot be negative");
}
```

**Related Terms**: [Checked Exception](#checked-exception), [Unchecked Exception](#unchecked-exception), [Try Block](#try-block), [Catch Block](#catch-block)

**Interview Tip**: Understand the exception hierarchy and the difference between checked and unchecked exceptions.

---

### Inheritance
**Category**: OOP Principles

**Definition**: A mechanism where a subclass inherits properties and methods from a superclass. Inheritance promotes code reuse and establishes an "is-a" relationship between classes.

**Practical Example**:
```java
class Animal {
    void eat() {
        System.out.println("Eating");
    }
}

class Dog extends Animal {
    void bark() {
        System.out.println("Woof!");
    }
}

Dog dog = new Dog();
dog.eat();   // Inherited from Animal
dog.bark();  // Own method
```

**Related Terms**: [Superclass](#superclass), [Subclass](#subclass), [Polymorphism](#polymorphism), [Method Overriding](#method-overriding)

**Interview Tip**: Discuss the difference between inheritance and composition.

---

### Interface
**Category**: OOP Fundamentals

**Definition**: A contract that specifies what methods a class must implement. Interfaces define a set of abstract methods that implementing classes must provide. A class can implement multiple interfaces but extend only one class.

**Practical Example**:
```java
interface Animal {
    void makeSound();
    void eat();
}

interface Movable {
    void move();
}

class Dog implements Animal, Movable {
    @Override
    public void makeSound() {
        System.out.println("Woof!");
    }
    
    @Override
    public void eat() {
        System.out.println("Eating");
    }
    
    @Override
    public void move() {
        System.out.println("Running");
    }
}
```

**Related Terms**: [Abstract Class](#abstract-class), [Implements](#implements), [Contract](#contract), [Default Method](#default-method)

**Interview Tip**: Understand when to use interfaces vs. abstract classes. Know about interface segregation principle.

---

### String
**Category**: Data Types

**Definition**: An immutable sequence of characters. Strings are objects in Java and are commonly used for text manipulation. The String class provides numerous methods for string operations.

**Practical Example**:
```java
// String creation
String str1 = "Hello";
String str2 = new String("Hello");

// String methods
int length = str1.length();
String upper = str1.toUpperCase();
boolean contains = str1.contains("ell");
String[] parts = str1.split("l");

// String concatenation
String result = str1 + " World";
String result2 = str1.concat(" World");

// String immutability
String original = "Hello";
String modified = original.replace("H", "J");  // Creates new String
System.out.println(original);  // Still "Hello"
System.out.println(modified);  // "Jello"
```

**Related Terms**: [Immutable](#immutable), [StringBuilder](#stringbuilder), [StringBuffer](#stringbuffer), [Character](#character)

**Interview Tip**: Understand string immutability and its implications. Know about string pool and string comparison (== vs. equals()).

---

This glossary provides a solid foundation for Java terminology. Each term includes practical examples and interview tips to help you prepare for senior-level interviews.
