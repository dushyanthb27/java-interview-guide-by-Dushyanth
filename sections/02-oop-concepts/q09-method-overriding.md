# Q9: How Does Method Overriding Work?

## Question
What is method overriding in Java? Explain the rules, `@Override` annotation, access modifier and exception rules, covariant return types, and bridge methods.

## Difficulty: 🟡 Intermediate

---

## Answer

**Method overriding** occurs when a subclass provides a **specific implementation** for a method already defined in its superclass. It is the mechanism behind **runtime polymorphism**.

### Basic Overriding

```java
public class Animal {
    public String speak() {
        return "Some generic sound";
    }
}

public class Dog extends Animal {
    @Override  // ✅ Always use @Override annotation
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

### Rules for Valid Overriding

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

### @Override Annotation

```java
public class Parent {
    public void process(String data) {
        System.out.println("Parent processing: " + data);
    }
}

// ✅ @Override catches mistakes at compile time
public class Child extends Parent {
    @Override
    public void process(String data) {  // Correct override
        System.out.println("Child processing: " + data);
    }

    // ❌ Without @Override, this silent bug compiles fine:
    // public void process(Object data) { }  // This is OVERLOADING, not overriding!
    // @Override catches: "method does not override or implement a method from a supertype"
}
```

### Access Modifier Rules

```java
public class Base {
    protected void doWork() {
        System.out.println("Base work");
    }
}

// ✅ Less restrictive (protected → public) is allowed
public class LessRestrictive extends Base {
    @Override
    public void doWork() {  // protected → public ✅
        System.out.println("Public work");
    }
}

// ❌ More restrictive is NOT allowed
public class MoreRestrictive extends Base {
    // @Override
    // private void doWork() { }  // COMPILE ERROR: protected → private ❌
}
```

Access modifier hierarchy (less → more restrictive):
```
public → protected → package-private → private
  ✅         ✅            ⚠️              ❌
(can widen access, cannot narrow it)
```

### Exception Rules

```java
public class FileProcessor {
    public void process() throws IOException {
        // Base implementation
    }
}

// ✅ Subtype exception (narrower)
public class SafeProcessor extends FileProcessor {
    @Override
    public void process() throws FileNotFoundException { // Subtype of IOException ✅
    }
}

// ✅ No exception (narrower)
public class NoExceptionProcessor extends FileProcessor {
    @Override
    public void process() { // No checked exception ✅
    }
}

// ❌ Broader exception
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

### Covariant Return Types

```java
public class AnimalShelter {
    public Animal adopt() {
        return new Animal();
    }
}

public class DogShelter extends AnimalShelter {
    @Override
    public Dog adopt() {  // ✅ Covariant return: Dog IS-A Animal
        return new Dog("Buddy", 3, "Lab");
    }
}

// Benefit: callers using DogShelter directly get Dog without casting
DogShelter shelter = new DogShelter();
Dog dog = shelter.adopt(); // No cast needed
```

### Cannot Override: static, final, private

```java
public class Parent {
    // ❌ Static methods are HIDDEN, not overridden
    public static void staticMethod() {
        System.out.println("Parent static");
    }

    // ❌ Final methods CANNOT be overridden
    public final void finalMethod() {
        System.out.println("Cannot change me");
    }

    // ❌ Private methods are NOT inherited, so cannot be overridden
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

```java
// Method hiding vs overriding
Parent ref = new Child();
ref.staticMethod();   // "Parent static" → resolved by reference type (hiding)
ref.toString();       // Resolved by actual object type (overriding)
```

### Bridge Methods with Generics

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

### Real-World Override Example

```java
public abstract class CacheStrategy {
    public abstract <K, V> V get(K key);
    public abstract <K, V> void put(K key, V value);
    public abstract void evict();

    // Template method with hook
    public void clear() {
        evict();
        System.out.println("Cache cleared");
    }
}

public class LRUCache extends CacheStrategy {
    private final LinkedHashMap<Object, Object> cache;

    public LRUCache(int capacity) {
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
                return size() > capacity;
            }
        };
    }

    @Override
    @SuppressWarnings("unchecked")
    public <K, V> V get(K key) { return (V) cache.get(key); }

    @Override
    public <K, V> void put(K key, V value) { cache.put(key, value); }

    @Override
    public void evict() { cache.clear(); }
}
```

### Common Mistake

```java
// ❌ Accidentally overloading instead of overriding
public class Parent {
    public boolean equals(Object obj) { /* ... */ return false; }
}

public class Child extends Parent {
    // This is OVERLOADING (parameter type is Child, not Object)
    public boolean equals(Child other) {
        return true; // This won't be called by collections!
    }
}

// ✅ Always use @Override to catch this
public class Child extends Parent {
    @Override
    public boolean equals(Object obj) { // Correct: parameter is Object
        if (this == obj) return true;
        if (!(obj instanceof Child other)) return false;
        return true;
    }
}
```

### Performance Tip
Method overriding uses virtual method dispatch (vtable lookup). The JIT compiler can devirtualize and inline monomorphic call sites (where only one implementation is ever used), achieving near-zero overhead.

---

## Interviewer Insight
Interviewers look for precise knowledge of overriding rules, especially around access modifiers and exceptions. The `@Override` annotation question is a favorite. Understanding bridge methods shows deep knowledge of generics and type erasure.

## Follow-up Questions
1. What is the difference between method overriding and method hiding?
2. Can you override a method to be synchronized even if the parent method is not?
3. How does the JVM resolve overridden methods at runtime?

## Related Topics
- [Method Overloading](./q08-method-overloading.md)
- [Polymorphism](./q04-polymorphism.md)
- [Super Keyword](./q10-super-keyword.md)
