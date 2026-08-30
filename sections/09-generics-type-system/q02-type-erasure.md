# Q2: What Is Type Erasure?

[Home](../../README.md) > [Generics & Type System](./README.md) > Type Erasure

---

## 🟡 Difficulty: Intermediate

## Question
> What is type erasure in Java? How does it affect generic code at runtime?

---

## Quick Answer
**Type erasure** is the process by which the Java compiler removes all generic type information during compilation. Unbounded type parameters are replaced with `Object`, bounded type parameters are replaced with their first bound, and casts are inserted where necessary. At runtime, a `List<String>` and a `List<Integer>` are both just `List` — there is no way to distinguish them.

---

## Detailed Explanation

### How Type Erasure Works

```java
// What you write:
public class Box<T> {
    private T content;
    public T getContent() { return content; }
    public void setContent(T content) { this.content = content; }
}

// After erasure (what the JVM sees):
public class Box {
    private Object content;
    public Object getContent() { return content; }
    public void setContent(Object content) { this.content = content; }
}
```

### Erasure Rules

| Declaration | Erased To |
|-------------|-----------|
| `<T>` (unbounded) | `Object` |
| `<T extends Number>` | `Number` |
| `<T extends Comparable<T>>` | `Comparable` |
| `<T extends Comparable & Serializable>` | `Comparable` (first bound) |

### Erasure of Bounded Type Parameters

```java
// What you write:
public class NumberBox<T extends Number> {
    private T value;

    public double doubleValue() {
        return value.doubleValue();  // Legal — T is bounded by Number
    }
}

// After erasure:
public class NumberBox {
    private Number value;           // Erased to first bound

    public double doubleValue() {
        return value.doubleValue();  // Still valid on Number
    }
}
```

### Compiler-Inserted Casts

```java
// What you write:
Box<String> box = new Box<>();
box.setContent("Hello");
String s = box.getContent();

// What the compiler generates (after erasure):
Box box = new Box();
box.setContent("Hello");
String s = (String) box.getContent();  // Cast inserted by compiler
```

### Runtime Type Queries

```java
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();

// Both are the same class at runtime
System.out.println(strings.getClass() == integers.getClass());  // true
System.out.println(strings.getClass());  // class java.util.ArrayList

// ❌ Cannot use instanceof with parameterized type
// if (strings instanceof List<String>) { }  // COMPILE ERROR

// ✅ Can only check raw type
if (strings instanceof List<?>) {         // OK — unbounded wildcard
    System.out.println("It's a List");
}

// ✅ Can check element types individually
Object obj = strings.get(0);
if (obj instanceof String) {
    System.out.println("Element is a String");
}
```

### Reflective Access to Generic Type Information

Even though type erasure removes type info from instances, the **class metadata** retains generic signatures for fields, methods, and superclass declarations:

```java
import java.lang.reflect.*;

// Generic type info is preserved in class metadata (not instances)
public class StringList extends ArrayList<String> { }

public class ReflectionDemo {
    // Field-level generic type is preserved
    private List<String> names;
    private Map<String, List<Integer>> data;

    public static void main(String[] args) throws Exception {
        // Superclass generic type
        Type superclass = StringList.class.getGenericSuperclass();
        if (superclass instanceof ParameterizedType pt) {
            System.out.println(pt.getActualTypeArguments()[0]);
            // Output: class java.lang.String
        }

        // Field-level generic type
        Field field = ReflectionDemo.class.getDeclaredField("names");
        Type genericType = field.getGenericType();
        if (genericType instanceof ParameterizedType pt) {
            System.out.println(pt.getRawType());              // interface java.util.List
            System.out.println(pt.getActualTypeArguments()[0]); // class java.lang.String
        }

        // Nested generic type
        Field dataField = ReflectionDemo.class.getDeclaredField("data");
        ParameterizedType mapType = (ParameterizedType) dataField.getGenericType();
        Type valueType = mapType.getActualTypeArguments()[1];  // List<Integer>
        if (valueType instanceof ParameterizedType listType) {
            System.out.println(listType.getActualTypeArguments()[0]); // class java.lang.Integer
        }
    }
}
```

### Method Generic Signature Preservation

```java
public class Service {
    public <T extends Comparable<T>> List<T> sort(List<T> input) {
        Collections.sort(input);
        return input;
    }
}

// Retrieve method generic info via reflection
Method method = Service.class.getMethod("sort", List.class);
TypeVariable<?>[] typeParams = method.getTypeParameters();
System.out.println(typeParams[0].getName());    // T
System.out.println(typeParams[0].getBounds()[0]); // java.lang.Comparable
```

### Erasure and Overloading Conflict

```java
public class Processor {
    // ❌ COMPILE ERROR — both erase to process(List)
    // public void process(List<String> strings) { }
    // public void process(List<Integer> integers) { }

    // ✅ Workaround: use different method names
    public void processStrings(List<String> strings) { }
    public void processIntegers(List<Integer> integers) { }

    // ✅ Another workaround: generic method
    public <T> void process(List<T> items, Class<T> type) { }
}
```

### Erasure and Exceptions

```java
// ❌ Generic class cannot extend Throwable
// public class GenericException<T> extends Exception { }  // COMPILE ERROR

// ❌ Cannot catch a type parameter
// public <T extends Exception> void handle() {
//     try { /* ... */ } catch (T e) { }  // COMPILE ERROR
// }

// ✅ Can declare throws with type parameter
public <T extends Exception> void riskyMethod(T exception) throws T {
    throw exception;
}
```

### Viewing Erased Bytecode with javap

```bash
# Compile a generic class
javac Box.java

# View erased bytecode
javap -c -p Box.class

# Output shows Object instead of T:
#   private java.lang.Object content;
#   public java.lang.Object getContent();
#   public void setContent(java.lang.Object);

# View generic signature metadata
javap -v Box.class
# Shows Signature attribute: <T:Ljava/lang/Object;>Ljava/lang/Object;
```

---

## Anti-Patterns

```java
// ❌ Trying to create instances of type parameters
public class Factory<T> {
    // public T create() { return new T(); }  // COMPILE ERROR — T is erased
}

// ❌ Relying on runtime generic type checks
public <T> boolean isList(Object obj) {
    // return obj instanceof List<T>;  // COMPILE ERROR
    return obj instanceof List<?>;     // Only this works
}

// ❌ Assuming generic types differ at runtime
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
// a.getClass() == b.getClass() is TRUE — they are the same class

// ❌ Overloading methods that differ only in type parameter
// void handle(List<String> s) { }
// void handle(List<Integer> i) { }  // Erasure collision
```

## Best Practices

```java
// ✅ Pass Class<T> token when runtime type info is needed
public class Factory<T> {
    private final Class<T> type;

    public Factory(Class<T> type) {
        this.type = type;
    }

    public T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}

// ✅ Use anonymous subclass trick to capture generic types
abstract class TypeReference<T> {
    private final Type type;

    protected TypeReference() {
        Type superclass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superclass).getActualTypeArguments()[0];
    }

    public Type getType() { return type; }
}

// Captures List<String> at runtime
TypeReference<List<String>> ref = new TypeReference<List<String>>() {};
System.out.println(ref.getType());  // java.util.List<java.lang.String>

// ✅ Understand what reflection can and cannot reveal
// CAN:  generic signatures on classes, fields, methods, superclasses
// CANNOT: generic type of a local variable or specific instance
```

---

## Interview Tips
- Type erasure exists for **backward compatibility** with pre-Java 5 code
- Generic type info is erased from **instances** but preserved in **class metadata**
- `List<String>.class` does not exist — only `List.class`
- Reflection via `ParameterizedType` can recover declared generic signatures
- Bridge methods are generated to preserve polymorphism after erasure (see Q10)

---

[← Q1: Generic Classes & Methods](./q01-generic-classes-methods.md) | [Next: Q3 - Bounded Type Parameters →](./q03-bounded-type-parameters.md)
