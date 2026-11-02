# Inheritance

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Inheritance

---

## Q6 🟢 How Does Inheritance Work in Java?

**Question** — How does inheritance work in Java? Explain the types of inheritance, constructor chaining with `super()`, and why Java does not support multiple class inheritance.

### Answer

**Inheritance** is the mechanism by which one class (child/subclass) acquires the fields and methods of another class (parent/superclass). It establishes an **IS-A relationship** and promotes code reuse.

**Basic Inheritance with `extends`**

```java
// Parent (superclass)
public class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Child (subclass) - Dog IS-A Animal
public class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age); // Must be FIRST statement
        this.breed = breed;
    }

    // New behavior specific to Dog
    public void fetch() {
        System.out.println(name + " is fetching the ball");
    }

    // Overriding parent behavior
    @Override
    public void eat() {
        System.out.println(name + " the " + breed + " is eating dog food");
    }
}
```

**Types of Inheritance**

Java supports single, multilevel, and hierarchical inheritance — but NOT multiple class inheritance.

```
1. SINGLE                2. MULTILEVEL            3. HIERARCHICAL
   Animal                   Animal                   Animal
     ▲                        ▲                      ▲    ▲
     │                        │                      │    │
    Dog                     Mammal                  Dog  Cat
                              ▲
                              │
                             Dog
```

```java
// Single Inheritance
class Dog extends Animal { }

// Multilevel Inheritance
class Mammal extends Animal { }
class Dog extends Mammal { }

// Hierarchical Inheritance
class Dog extends Animal { }
class Cat extends Animal { }

// Multiple Class Inheritance - NOT allowed in Java
// class Hybrid extends Dog, Cat { }  // COMPILE ERROR
```

**Constructor Chaining with `super()`**

Constructors are NOT inherited, but the parent constructor must always be called. If you don't call `super()` explicitly, the compiler inserts `super()` (no-arg) automatically. This can cause compile errors if the parent has no no-arg constructor.

```java
public class Vehicle {
    private String make;

    public Vehicle(String make) {
        this.make = make;
        System.out.println("Vehicle constructor: " + make);
    }
}

public class Car extends Vehicle {
    private int doors;

    public Car(String make, int doors) {
        super(make); // Explicit call to parent constructor (FIRST statement)
        this.doors = doors;
        System.out.println("Car constructor: " + doors + " doors");
    }
}

public class ElectricCar extends Car {
    private int batteryCapacity;

    public ElectricCar(String make, int doors, int batteryCapacity) {
        super(make, doors); // Calls Car constructor, which calls Vehicle constructor
        this.batteryCapacity = batteryCapacity;
        System.out.println("ElectricCar constructor: " + batteryCapacity + "kWh");
    }
}
```

```java
// Output when creating: new ElectricCar("Tesla", 4, 100)
// Vehicle constructor: Tesla
// Car constructor: 4 doors
// ElectricCar constructor: 100kWh
```

**Object Class as Root**

Every class in Java implicitly extends `java.lang.Object`. This means all classes inherit methods like `toString()`, `equals()`, `hashCode()`, `getClass()`, `clone()`, `finalize()`, and the thread coordination methods.

**Method Inheritance and Access**

Public, protected, and package-private methods are inherited. Private methods are NOT inherited. Static methods are inherited but hidden (not overridden). Final methods are inherited but cannot be overridden.

**Why No Multiple Class Inheritance?**

Java avoids the diamond problem by supporting single class inheritance with multiple interface implementation. The diamond problem occurs when a class inherits from two types that both provide an implementation of the same method, creating ambiguity.

```java
// Hypothetical problem if Java allowed multiple class inheritance:
class A {
    public void greet() { System.out.println("Hello from A"); }
}

class B {
    public void greet() { System.out.println("Hello from B"); }
}

// If this were allowed, which greet() does C inherit?
// class C extends A, B { }  // Ambiguity! (The Diamond Problem)

// Java solution: use interfaces for multiple inheritance of type
interface Greetable {
    default void greet() { System.out.println("Hello from Greetable"); }
}

interface Farewell {
    default void greet() { System.out.println("Goodbye from Farewell"); }
}

// Must explicitly resolve the conflict
class Polite implements Greetable, Farewell {
    @Override
    public void greet() {
        Greetable.super.greet(); // Explicitly choose
    }
}
```

**Sealed Classes (Java 17+)**

Sealed classes control which classes can extend them, providing a more secure inheritance model.

```java
// Control which classes can extend your class
public sealed class Shape permits Circle, Rectangle, Triangle {
    public abstract double area();
}

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    @Override public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double w, double h) { this.width = w; this.height = h; }
    @Override public double area() { return width * height; }
}

public non-sealed class Triangle extends Shape {
    private final double base, height;
    public Triangle(double b, double h) { this.base = b; this.height = h; }
    @Override public double area() { return 0.5 * base * height; }
}
```

> **⚠️ Common Mistake** — Forgetting that `super()` is implicitly called if not specified. If the parent has no no-arg constructor, this causes a compile error.

```java
// Forgetting that super() is implicitly called if not specified
class Parent {
    Parent(int x) { } // No no-arg constructor
}

class Child extends Parent {
    Child() {
        // Implicit super() call fails - Parent has no no-arg constructor!
        // COMPILE ERROR: constructor Parent() not found
    }
}

// Fix: explicitly call the available parent constructor
class Child extends Parent {
    Child() {
        super(0); // Explicitly call the parameterized constructor
    }
}
```

> **🔧 Performance Tip** — Deep inheritance hierarchies (more than 3-4 levels) become hard to maintain and debug. Prefer **composition over inheritance** for flexible designs. Use inheritance only for genuine IS-A relationships.

> **🎯 Interviewer Insight** — Interviewers look for a clear understanding of the IS-A relationship, constructor chaining, and the rationale behind Java's single-inheritance restriction. Mentioning sealed classes (Java 17+) shows awareness of modern Java.

> **⚡ Follow-up** — Can a subclass inherit private members of its parent?

> **⚡ Follow-up** — What is the difference between `extends` and `implements`?

> **⚡ Follow-up** — How do sealed classes improve upon traditional inheritance?

> **📚 Related** — [Polymorphism](./04-polymorphism.md), [Super Keyword](./10-super-keyword.md), [Composition vs Inheritance](./12-object-composition.md)

---

[← Previous: Constructors](./02-constructors.md) · [Next: Polymorphism →](./04-polymorphism.md)
