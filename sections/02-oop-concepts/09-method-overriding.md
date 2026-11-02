# Method Overriding

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Method Overriding

---

## Q12 🟡 How Does Method Overriding Work?

**Question** — What is method overriding in Java? Explain the rules, `@Override` annotation, access modifier and exception rules, covariant return types, and bridge methods.

### Answer

**Method overriding** occurs when a subclass provides a **specific implementation** for a method already defined in its superclass. It is the mechanism behind **runtime polymorphism** — the JVM decides which method to call based on the actual object type at runtime.

**Basic Overriding**

```java
public class Animal {
    public String speak() {
        return "Some generic sound";
    }
}

public class Dog extends Animal {
    @Override  // Always use @Override annotation
    public String speak() {
        return "Woof!";
    }
}

public class Cat extends Animal {
    @Override
    public String speak() {
        return "Meow!";
    }
}

// Runtime polymorphism in action
Animal animal = new Dog();
System.out.println(animal.speak()); // "Woof!" - determined at runtime
```

**Rules for Valid Overriding**

| Rule | Requirement |
|------|------------|
| Method name | Must be **identical** |
| Parameters | Must be **identical** (same types, order, count) |
| Return type | Same type or **covariant** (subtype) |
| Access modifier | Same or **less restrictive** |
| Exceptions (checked) | Same, subtype, or **fewer** |
| `static` methods | ❌ Cannot override (hidden instead) |
| `final` methods | ❌ Cannot override |
| `private` methods | ❌ Cannot override (not inherited) |
| Constructors | ❌ Cannot override (not inherited) |

**@Override Annotation**

Always use the `@Override` annotation to catch mistakes at compile time. It ensures you're actually overriding a method and not accidentally overloading.

```java
public class Parent {
    public void process(String data) {
        System.out.println("Parent processing: " + data);
    }
}

// @Override catches mistakes at compile time
public class Child extends Parent {
    @Override
    public void process(String data) {  // Correct override
        System.out.println("Child processing: " + data);
    }

    // Without @Override, this silent bug compiles fine:
    // public void process(Object data) { }  // This is OVERLOADING, not overriding!
    // @Override catches: "method does not override or implement a method from a supertype"
}
```

**Access Modifier Rules**

Subclasses can widen access (make methods more visible) but cannot narrow it. The hierarchy is — `public` → `protected` → package-private → `private`. You can move down this hierarchy but not up.

```java
public class Base {
    protected void doWork() {
        System.out.println("Base work");
    }
}

// Less restrictive (protected → public) is allowed
public class LessRestrictive extends Base {
    @Override
    public void doWork() {  // protected → public ✅
        System.out.println("Public work");
    }
}

// More restrictive is NOT allowed
public class MoreRestrictive extends Base {
    // @Override
    // private void doWork() { }  // COMPILE ERROR: protected → private ❌
}
```

**Exception Rules**

Subclasses can narrow the exception list (declare fewer or more specific checked exceptions) but cannot broaden it. Unchecked exceptions (RuntimeException) can always be thrown regardless of the parent method's exception declaration.

```java
public class FileProcessor {
    public void process() throws IOException {
        // Base implementation
    }
}

// Subtype exception (narrower)
public class SafeProcessor extends FileProcessor {
    @Override
    public void process() throws FileNotFoundException { // Subtype of IOException ✅
    }
}

// No exception (narrower)
public class NoExceptionProcessor extends FileProcessor {
    @Override
    public void process() { // No checked exception ✅
    }
}

// Broader exception
public class BroadProcessor extends FileProcessor {
    // @Override
    // public void process() throws Exception { } // COMPILE ERROR ❌
    // Exception is broader than IOException
}

// Note: Unchecked exceptions (RuntimeException) can always be thrown
public class RuntimeProcessor extends FileProcessor {
    @Override
    public void process() throws IOException {
        throw new RuntimeException("Always allowed"); // ✅
    }
}
```

**Covariant Return Types**

A subclass can override a method and return a **more specific type** — a subtype of the parent's return type. This eliminates the need for casting when the subclass type is known.

```java
public class AnimalShelter {
    public Animal adopt() {
        return new Animal();
    }
}

public class DogShelter extends AnimalShelter {
    @Override
    public Dog adopt() {  // Covariant return: Dog IS-A Animal
        return new Dog("Buddy", 3, "Lab");
    }
}

// Benefit: callers using DogShelter directly get Dog without casting
DogShelter shelter = new DogShelter();
Dog dog = shelter.adopt(); // No cast needed
```

**Cannot Override — static, final, private**

Static methods are **hidden**, not overridden. Final methods cannot be overridden by design. Private methods are not inherited, so they cannot be overridden.

```java
public class Parent {
    // Static methods are HIDDEN, not overridden
    public static void staticMethod() {
        System.out.println("Parent static");
    }

    // Final methods CANNOT be overridden
    public final void finalMethod() {
        System.out.println("Cannot change me");
    }

    // Private methods are NOT inherited, so cannot be overridden
    private void privateMethod() {
        System.out.println("Invisible to child");
    }
}

public class Child extends Parent {
    // This is METHOD HIDING, not overriding
    public static void staticMethod() {
        System.out.println("Child static");
    }

    // COMPILE ERROR:
    // public void finalMethod() { } // Cannot override final

    // This is a NEW method, not an override (Parent's is not visible)
    private void privateMethod() {
        System.out.println("Child's own private method");
    }
}
```

**Method hiding vs overriding** — static methods are resolved by reference type (hiding), while instance methods are resolved by actual object type (overriding).

```java
// Method hiding vs overriding
Parent ref = new Child();
ref.staticMethod();   // "Parent static" → resolved by reference type (hiding)
ref.toString();       // Resolved by actual object type (overriding)
```

**Bridge Methods with Generics**

The compiler generates synthetic "bridge methods" to ensure polymorphism works correctly after type erasure. These methods delegate to the actual overridden method.

```java
public class Container<T> {
    public T get(T item) {
        return item;
    }
}

public class StringContainer extends Container<String> {
    @Override
    public String get(String item) {
        return item.toUpperCase();
    }
}

// After type erasure, the compiler generates a BRIDGE method:
// public Object get(Object item) {        // Bridge method (synthetic)
//     return get((String) item);           // Delegates to actual override
// }
// This ensures polymorphism works correctly with erased generics
```

> **⚠️ Common Mistake** — Accidentally overloading instead of overriding by changing the parameter type. Always use `@Override` to catch this.

```java
// Accidentally overloading instead of overriding
public class Parent {
    public boolean equals(Object obj) { /* ... */ return false; }
}

public class Child extends Parent {
    // This is OVERLOADING (parameter type is Child, not Object)
    public boolean equals(Child other) {
        return true; // This won't be called by collections!
    }
}

// Always use @Override to catch this
public class Child extends Parent {
    @Override
    public boolean equals(Object obj) { // Correct: parameter is Object
        if (this == obj) return true;
        if (!(obj instanceof Child other)) return false;
        return true;
    }
}
```

> **🔧 Performance Tip** — Method overriding uses virtual method dispatch (vtable lookup). The JIT compiler can devirtualize and inline monomorphic call sites (where only one implementation is ever used), achieving near-zero overhead.

> **🎯 Interviewer Insight** — Interviewers look for precise knowledge of overriding rules, especially around access modifiers and exceptions. The `@Override` annotation question is a favorite. Understanding bridge methods shows deep knowledge of generics and type erasure.

> **⚡ Follow-up** — What is the difference between method overriding and method hiding?

> **⚡ Follow-up** — Can you override a method to be synchronized even if the parent method is not?

> **⚡ Follow-up** — How does the JVM resolve overridden methods at runtime?

> **📚 Related** — [Method Overloading](./08-method-overloading.md), [Polymorphism](./04-polymorphism.md), [Super Keyword](./10-super-keyword.md)

---

[← Previous: Method Overloading](./08-method-overloading.md) · [Next: Super Keyword →](./10-super-keyword.md)
