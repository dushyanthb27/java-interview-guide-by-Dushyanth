# Q10: What Are Bridge Methods?

[Home](../../README.md) > [Generics & Type System](./README.md) > Bridge Methods

---

## 🟠 Difficulty: Advanced

## Question

> What are bridge methods in Java? Why does the compiler generate them, how do they interact with type erasure and covariant return types, and how can they be inspected via bytecode and reflection?

---

## Quick Answer

**Bridge methods** are synthetic, compiler-generated methods created to preserve **polymorphism and virtual method dispatch** after type erasure. When a generic class or interface is extended/implemented and a method is overridden with a specific (non-erased) parameter or return type, erasure alters the method signature such that the subclass method no longer overrides the parent's erased method. The compiler inserts a bridge method with the erased signature that delegates directly to the user-written overriding method.

---

## Detailed Explanation

### The Problem: Erasure Breaks Overriding

```java
// Generic parent class
public class Box<T> {
    private T value;
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
}

// Concrete subclass with specific type parameter substitution
public class StringBox extends Box<String> {
    @Override
    public String getValue() { return super.getValue(); }

    @Override
    public void setValue(String value) { super.setValue(value); }
}
```

### After Type Erasure

```java
// Box<T> erases to:
public class Box {
    private Object value;
    public Object getValue() { return value; }
    public void setValue(Object value) { this.value = value; }
}

// StringBox erases to (without bridge methods):
public class StringBox extends Box {
    public String getValue() { return (String) super.getValue(); }
    // ⚠️ Signature is getValue() → String, NOT getValue() → Object!
    // It no longer overrides Box.getValue() → Object in the JVM!
}
```

### The Solution: Bridge Methods

```java
// The compiler generates bridge methods in StringBox:
public class StringBox extends Box {
    // 1. YOUR method (actual implementation)
    public String getValue() {
        return (String) super.getValue();
    }

    // 2. BRIDGE method (compiler-generated, ACC_BRIDGE + ACC_SYNTHETIC flags)
    @Override
    public Object getValue() {        // Matches erased parent signature
        return this.getValue();        // Delegates to your String version
    }

    // 3. YOUR method
    public void setValue(String value) {
        super.setValue(value);
    }

    // 4. BRIDGE method (compiler-generated)
    @Override
    public void setValue(Object value) {  // Matches erased parent signature
        this.setValue((String) value);    // Delegates with runtime check/cast
    }
}
```

### Viewing Bridge Methods with javap

```bash
javac Box.java StringBox.java
javap -c -p StringBox.class
```

**Bytecode output excerpts:**
```
public java.lang.Object getValue();
  Code:
     0: aload_0
     1: invokevirtual #2  // Calls StringBox.getValue():String
     4: areturn

public void setValue(java.lang.Object);
  Code:
     0: aload_0
     1: aload_1
     2: checkcast     #3  // Cast Object to String
     5: invokevirtual #4  // Calls StringBox.setValue(String)
     8: return
```

### Detecting Bridge Methods via Reflection

```java
import java.lang.reflect.Method;

public class BridgeInspector {
    public static void inspect(Class<?> clazz) {
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isBridge()) {
                System.out.printf("Bridge method found: %s%n", method.toGenericString());
            }
        }
    }
}
```

### Bridge Methods and Covariant Return Types

```java
public class Animal {
    public Animal create() { return new Animal(); }
}

public class Dog extends Animal {
    @Override
    public Dog create() { return new Dog(); }  // Covariant return type

    // Compiler generates bridge method:
    // public Animal create() { return this.create(); }
}
```
*Note*: The JVM itself does not natively support covariant return types at the virtual instruction level; bridge methods implement covariant return types under the hood.

### Framework Handling (Spring Framework)

Frameworks like Spring use `org.springframework.core.BridgeMethodResolver` to unwrap bridge methods when inspecting annotations or proxying methods:
```java
Method resolved = BridgeMethodResolver.findBridgedMethod(method);
```

---

## Gotchas

- **ClassCastException at Bridge Boundary**: When raw types are passed into generic references pointing to concrete subtypes, the bridge method's implicit `checkcast` throws a `ClassCastException` at the entry point.
- **Reflection Pollution**: Iterating over `Class.getDeclaredMethods()` without filtering `Method.isBridge()` will process both the user method and the synthetic bridge method, leading to duplicate execution or annotation processing bugs.
- **Overloading vs Overriding Trap**: Accidentally altering parameter types in subclasses can result in method overloading rather than overriding, bypassing bridge method generation and breaking polymorphic dispatch.
- **Annotation Inheritance Quirks**: Annotations placed on interface or superclass generic methods must be handled carefully when bridge methods are dispatched via reflection proxies.

---

## Anti-Patterns

```java
// ❌ Processing reflection methods without filtering out bridge/synthetic methods
for (Method m : clazz.getDeclaredMethods()) {
    if (m.isAnnotationPresent(MyAnno.class)) {
        execute(m); // Executes twice: once for user method, once for bridge method!
    }
}

// ❌ Assuming a class only has one method for a given name and arity
```

## Best Practices

```java
// ✅ Always filter bridge methods in reflection processing
for (Method method : clazz.getDeclaredMethods()) {
    if (!method.isBridge() && !method.isSynthetic()) {
        process(method);
    }
}

// ✅ Use framework utilities like Spring's BridgeMethodResolver when dealing with AOP proxies
Method realMethod = BridgeMethodResolver.findBridgedMethod(proxyMethod);

// ✅ Rely on @Override to ensure signature matching before bridge generation
```

## Interview Tips

- Bridge methods are **synthetic** and marked with `ACC_BRIDGE` and `ACC_SYNTHETIC` bytecode flags.
- They preserve **polymorphism** after type erasure when subclassing generic types.
- They power **covariant return types** in Java since the JVM instruction set relies on erased signatures.
- Use `Method.isBridge()` to identify them during reflection tasks.

---

[← Q9: Diamond Operator](./q09-diamond-operator.md) | [Next: Q11 - Generic Arrays →](./q11-generic-arrays.md)
