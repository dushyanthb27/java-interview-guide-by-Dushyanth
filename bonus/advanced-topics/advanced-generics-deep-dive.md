# Advanced Generics Deep-Dive

## 📋 Question
Explain advanced generics concepts including covariance and contravariance, F-bounded polymorphism, type tokens, and generic type erasure. How do you work around type erasure limitations? Provide real-world examples and best practices.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Generics in Java are a powerful tool for type safety, but they come with several advanced concepts and limitations due to type erasure. Understanding these deeply is essential for writing robust, type-safe libraries and frameworks.

### Covariance and Contravariance

Java generics are **invariant** by default, meaning `List<String>` is not a subtype of `List<Object>`. However, you can declare **covariant** (extends) and **contravariant** (super) type parameters.

```java
// ❌ Invariant by default — compile error
List<String> strings = new ArrayList<>();
List<Object> objects = strings;  // Compile error!

// ✅ Covariant — accepts String and subtypes
List<? extends String> covariant = new ArrayList<String>();
covariant = new ArrayList<>();  // OK
// covariant.add("hello");       // Compile error! Cannot add (null only)
String s = covariant.get(0);     // OK — returns String

// ✅ Contravariant — accepts String and supertypes
List<? super String> contravariant = new ArrayList<Object>();
contravariant.add("hello");      // OK — can add String
// Object o = contravariant.get(0); // Compile error! Returns Object only
```

**The PECS Principle**: "Producer Extends, Consumer Super"

```java
// ✅ Producer — extends (read-only)
public static <T> T copyFirst(List<? extends T> source, List<T> dest) {
    T first = source.get(0);  // Can read as T
    dest.add(first);          // Can add to dest
    return first;
}

// ✅ Consumer — super (write-only)
public static void addAll(List<? super Integer> dest) {
    dest.add(Integer.valueOf(1));  // Can add Integer
    dest.add(Integer.valueOf(2));
}
```

### F-Bounded Polymorphism

F-bounded polymorphism allows a generic type parameter to be bounded by the type itself. This is commonly used for fluent APIs and comparable interfaces.

```java
// ✅ F-bounded type parameter
public interface Comparable<T> {
    int compareTo(T other);
}

// Self-bounded generic
public abstract class Animal<T extends Animal<T>> {
    public abstract T breed();
    
    public T giveBirth() {
        System.out.println("Giving birth...");
        return breed();
    }
}

public class Dog extends Animal<Dog> {
    @Override
    public Dog breed() {
        return new Dog();
    }
    
    public void bark() {
        System.out.println("Woof!");
    }
}

// Usage — type-safe fluent chain
Dog dog = new Dog();
Dog puppy = dog.giveBirth();  // Returns Dog, not Animal
puppy.bark();  // Dog-specific method available
```

**Real-world example: Builder pattern with F-bounding**

```java
public abstract class Pizza<T extends Pizza<T>> {
    private List<String> toppings = new ArrayList<>();
    
    @SuppressWarnings("unchecked")
    public T addTopping(String topping) {
        toppings.add(topping);
        return (T) this;  // Safe due to F-bounding
    }
    
    public void bake() {
        System.out.println("Baking pizza with: " + toppings);
    }
}

public class MargheritaPizza extends Pizza<MargheritaPizza> {
    public MargheritaPizza addExtraCheese() {
        addTopping("extra cheese");
        return this;
    }
}

// Usage — fluent chain returns correct type
MargheritaPizza pizza = new MargheritaPizza()
    .addTopping("tomato")
    .addTopping("mozzarella")
    .addExtraCheese();  // Returns MargheritaPizza, not Pizza
pizza.bake();
```

### Type Tokens

Due to type erasure, generic type information is lost at runtime. Type tokens (Class objects) are used to preserve type information.

```java
// ❌ Type erasure — cannot get generic type at runtime
List<String> list = new ArrayList<>();
// list.getClass().getTypeParameters()[0]  // Returns "E", not "String"

// ✅ Type token pattern
public class TypeReference<T> {
    private final Type type;
    
    protected TypeReference() {
        Type superClass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }
    
    public Type getType() {
        return type;
    }
}

// Usage
TypeReference<List<String>> token = new TypeReference<List<String>>() {};
System.out.println(token.getType());  // java.util.List<java.lang.String>
```

**Real-world example: Generic DAO with type tokens**

```java
public abstract class GenericDAO<T, ID> {
    private final Class<T> entityClass;
    
    @SuppressWarnings("unchecked")
    protected GenericDAO() {
        Type superClass = getClass().getGenericSuperclass();
        this.entityClass = (Class<T>) ((ParameterizedType) superClass)
            .getActualTypeArguments()[0];
    }
    
    public T findById(ID id) {
        // Can use entityClass for reflection, Hibernate, etc.
        System.out.println("Finding " + entityClass.getSimpleName() + " with id: " + id);
        return null;  // Simplified
    }
    
    public List<T> findAll() {
        System.out.println("Finding all " + entityClass.getSimpleName());
        return new ArrayList<>();
    }
}

public class UserDAO extends GenericDAO<User, Long> {
    // entityClass automatically set to User.class
}

public class ProductDAO extends GenericDAO<Product, Long> {
    // entityClass automatically set to Product.class
}

// Usage
UserDAO userDAO = new UserDAO();
userDAO.findById(1L);  // Prints: Finding User with id: 1

ProductDAO productDAO = new ProductDAO();
productDAO.findAll();  // Prints: Finding all Product
```

### Type Erasure and Workarounds

Type erasure means generic type parameters are replaced with their bounds (or Object) at compile time.

```java
// Source code
public class Box<T> {
    private T value;
    
    public void set(T value) {
        this.value = value;
    }
    
    public T get() {
        return value;
    }
}

// After type erasure (what the JVM sees)
public class Box {
    private Object value;
    
    public void set(Object value) {
        this.value = value;
    }
    
    public Object get() {
        return value;
    }
}
```

**Workaround 1: Store type explicitly**

```java
public class TypedBox<T> {
    private final Class<T> type;
    private T value;
    
    public TypedBox(Class<T> type) {
        this.type = type;
    }
    
    public void set(Object value) {
        if (value != null && !type.isInstance(value)) {
            throw new ClassCastException("Expected " + type + ", got " + value.getClass());
        }
        this.value = type.cast(value);
    }
    
    public T get() {
        return value;
    }
}

// Usage
TypedBox<String> box = new TypedBox<>(String.class);
box.set("hello");  // OK
// box.set(123);    // Runtime ClassCastException
```

**Workaround 2: Use TypeReference for complex generics**

```java
public class JsonSerializer {
    public static <T> String serialize(T object) {
        // Simplified JSON serialization
        return "{\"value\":\"" + object + "\"}";
    }
    
    @SuppressWarnings("unchecked")
    public static <T> T deserialize(String json, TypeReference<T> typeRef) {
        // In real implementation, use Gson/Jackson with TypeToken
        Type type = typeRef.getType();
        System.out.println("Deserializing to: " + type);
        return null;  // Simplified
    }
}

// Usage
TypeReference<Map<String, List<Integer>>> typeRef = 
    new TypeReference<Map<String, List<Integer>>>() {};
JsonSerializer.deserialize("{}", typeRef);
```

### Multiple Type Bounds

A type parameter can have multiple bounds using `&`.

```java
// ✅ Multiple bounds — must be a class first, then interfaces
public class SmartList<T extends Comparable<T> & Serializable & Cloneable> {
    private List<T> list = new ArrayList<>();
    
    public void add(T item) {
        list.add(item);
    }
    
    public T findMax() {
        return list.stream().max(Comparator.naturalOrder()).orElse(null);
    }
    
    public T cloneItem(T item) throws CloneNotSupportedException {
        return (T) item.clone();  // Safe because T extends Cloneable
    }
}

// Usage
public class User implements Comparable<User>, Serializable, Cloneable {
    private String name;
    
    public User(String name) {
        this.name = name;
    }
    
    @Override
    public int compareTo(User other) {
        return this.name.compareTo(other.name);
    }
    
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}

SmartList<User> users = new SmartList<>();
users.add(new User("Alice"));
users.add(new User("Bob"));
User max = users.findMax();  // Uses Comparable
```

### Generic Methods vs Generic Types

```java
// ✅ Generic method — type parameter on method level
public class Utils {
    public static <T> T first(List<T> list) {
        return list.isEmpty() ? null : list.get(0);
    }
    
    // Generic method with bounds
    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) > 0 ? a : b;
    }
    
    // Generic method with wildcards
    public static void printAll(List<?> list) {
        for (Object item : list) {
            System.out.println(item);
        }
    }
}

// Usage
String first = Utils.first(Arrays.asList("a", "b", "c"));
Integer max = Utils.max(5, 10);
Utils.printAll(Arrays.asList(1, "hello", 3.14));
```

### Generic Arrays

Creating generic arrays is problematic due to type erasure.

```java
// ❌ Cannot create generic array directly
public class Container<T> {
    private T[] array;  // Declaration is OK
    // private T[] array = new T[10];  // Compile error!
    
    @SuppressWarnings("unchecked")
    public Container(int size) {
        // Workaround: cast Object[] to T[]
        this.array = (T[]) new Object[size];
    }
    
    public void set(int index, T value) {
        array[index] = value;
    }
    
    public T get(int index) {
        return array[index];
    }
}

// ⚠️ This workaround has limitations — can be unsafe
Container<String> container = new Container<>(10);
container.set(0, "hello");
String s = container.get(0);  // OK
// Object[] objArray = container.array;  // Would break type safety
```

**Better approach: Use List instead of arrays**

```java
public class SafeContainer<T> {
    private final List<T> list = new ArrayList<>();
    
    public void add(T item) {
        list.add(item);
    }
    
    public T get(int index) {
        return list.get(index);
    }
}
```

### Recursive Type Bounds

```java
// ✅ Recursive type bound — T must be comparable to itself
public static <T extends Comparable<? super T>> T max(List<? extends T> list) {
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}

// Usage
List<Integer> numbers = Arrays.asList(1, 5, 3, 9, 2);
Integer maxNum = max(numbers);  // Works because Integer implements Comparable<Integer>

List<String> strings = Arrays.asList("apple", "banana", "cherry");
String maxStr = max(strings);  // Works because String implements Comparable<String>
```

### Wildcard Capture

```java
// ✅ Wildcard capture helper method
public static void reverse(List<?> list) {
    reverseHelper(list);
}

private static <T> void reverseHelper(List<T> list) {
    List<T> temp = new ArrayList<>(list);
    for (int i = 0; i < list.size(); i++) {
        list.set(i, temp.get(list.size() - 1 - i));
    }
}

// Usage
List<String> strings = new ArrayList<>(Arrays.asList("a", "b", "c"));
reverse(strings);  // Works due to wildcard capture
System.out.println(strings);  // [c, b, a]
```

### Code Example: Complete Advanced Generics Demo

```java
package com.interview.advanced.generics;

import java.util.*;
import java.lang.reflect.*;

// === COVARIANCE AND CONTRAVARIANCE ===
class VarianceDemo {
    public static void main(String[] args) {
        // Producer Extends
        List<Integer> integers = Arrays.asList(1, 2, 3);
        Number number = sum(integers);  // Works with List<? extends Number>
        System.out.println("Sum: " + number);
        
        // Consumer Super
        List<Object> objects = new ArrayList<>();
        addNumbers(objects);  // Works with List<? super Integer>
        System.out.println("Objects: " + objects);
    }
    
    // Producer — extends (read-only)
    public static Number sum(List<? extends Number> list) {
        double sum = 0;
        for (Number n : list) {
            sum += n.doubleValue();
        }
        return sum;
    }
    
    // Consumer — super (write-only)
    public static void addNumbers(List<? super Integer> list) {
        list.add(1);
        list.add(2);
        list.add(3);
    }
}

// === F-BOUNDED POLYMORPHISM ===
abstract class Node<T extends Node<T>> implements Comparable<T> {
    private String name;
    
    public Node(String name) {
        this.name = name;
    }
    
    @Override
    public int compareTo(T other) {
        return this.name.compareTo(other.name);
    }
    
    public T withName(String name) {
        this.name = name;
        return (T) this;
    }
    
    public String getName() {
        return name;
    }
}

class FileNode extends Node<FileNode> {
    private long size;
    
    public FileNode(String name, long size) {
        super(name);
        this.size = size;
    }
    
    public FileNode withSize(long size) {
        this.size = size;
        return this;
    }
    
    @Override
    public String toString() {
        return getName() + " (" + size + " bytes)";
    }
}

class FBoundDemo {
    public static void main(String[] args) {
        FileNode file = new FileNode("document.txt", 1024)
            .withName("report.pdf")
            .withSize(2048);
        System.out.println(file);  // report.pdf (2048 bytes)
    }
}

// === TYPE TOKEN PATTERN ===
abstract class TypeSafeRepository<T, ID> {
    private final Class<T> entityClass;
    private final Map<ID, T> storage = new HashMap<>();
    
    @SuppressWarnings("unchecked")
    protected TypeSafeRepository() {
        Type superClass = getClass().getGenericSuperclass();
        this.entityClass = (Class<T>) ((ParameterizedType) superClass)
            .getActualTypeArguments()[0];
    }
    
    public void save(ID id, T entity) {
        if (!entityClass.isInstance(entity)) {
            throw new IllegalArgumentException(
                "Expected " + entityClass + ", got " + entity.getClass()
            );
        }
        storage.put(id, entity);
    }
    
    public T findById(ID id) {
        return storage.get(id);
    }
    
    public Class<T> getEntityClass() {
        return entityClass;
    }
}

class UserRepository extends TypeSafeRepository<User, Long> {
    // entityClass automatically User.class
}

class User {
    private String name;
    
    public User(String name) {
        this.name = name;
    }
    
    @Override
    public String toString() {
        return "User{" + name + "}";
    }
}

class TypeTokenDemo {
    public static void main(String[] args) {
        UserRepository repo = new UserRepository();
        System.out.println("Entity class: " + repo.getEntityClass());
        
        repo.save(1L, new User("Alice"));
        repo.save(2L, new User("Bob"));
        
        User user = repo.findById(1L);
        System.out.println("Found: " + user);
    }
}

// === MULTIPLE BOUNDS ===
interface SerializableCloneable extends Serializable, Cloneable {
}

class SmartContainer<T extends Comparable<T> & SerializableCloneable> {
    private final List<T> items = new ArrayList<>();
    
    public void add(T item) {
        items.add(item);
    }
    
    public T getMax() {
        return items.stream().max(Comparator.naturalOrder()).orElse(null);
    }
    
    @SuppressWarnings("unchecked")
    public T cloneMax() {
        T max = getMax();
        if (max != null) {
            try {
                return (T) max.clone();
            } catch (CloneNotSupportedException e) {
                throw new RuntimeException(e);
            }
        }
        return null;
    }
}

class SmartItem implements Comparable<SmartItem>, SerializableCloneable {
    private final int value;
    
    public SmartItem(int value) {
        this.value = value;
    }
    
    @Override
    public int compareTo(SmartItem other) {
        return Integer.compare(this.value, other.value);
    }
    
    @Override
    public Object clone() {
        return new SmartItem(this.value);
    }
    
    @Override
    public String toString() {
        return "Item(" + value + ")";
    }
}

class MultipleBoundsDemo {
    public static void main(String[] args) {
        SmartContainer<SmartItem> container = new SmartContainer<>();
        container.add(new SmartItem(10));
        container.add(new SmartItem(30));
        container.add(new SmartItem(20));
        
        System.out.println("Max: " + container.getMax());
        System.out.println("Cloned max: " + container.cloneMax());
    }
}
```

### ⚠️ Common Mistakes

**1. Using raw types**

```java
// ❌ Raw type — loses type safety
List rawList = new ArrayList<String>();
rawList.add(123);  // No compile error!
String s = (String) rawList.get(0);  // ClassCastException at runtime

// ✅ Use generics
List<String> typedList = new ArrayList<>();
// typedList.add(123);  // Compile error!
```

**2. Confusing extends and super**

```java
// ❌ Wrong — cannot add to extends
List<? extends Number> list = new ArrayList<Integer>();
list.add(Integer.valueOf(1));  // Compile error!

// ✅ Correct — use super for adding
List<? super Integer> list2 = new ArrayList<Number>();
list2.add(Integer.valueOf(1));  // OK
```

**3. Creating generic arrays unsafely**

```java
// ❌ Unsafe cast
public <T> T[] createArray(int size) {
    return (T[]) new Object[size];  // ClassCastException risk
}

// ✅ Use Array.newInstance or List
@SuppressWarnings("unchecked")
public <T> T[] createArray(Class<T> type, int size) {
    return (T[]) java.lang.reflect.Array.newInstance(type, size);
}
```

### ⚡ Performance Tip

Generics have **no runtime performance overhead** due to type erasure. The JVM sees the same bytecode for `List<String>` and `List<Integer>`. However, excessive use of wildcards and bounds can increase compilation time and make code harder to read. Use the simplest generic signature that meets your needs.

### 🔧 JVM Internals

Type erasure happens at **compile time**. The compiler:
1. Replaces all type parameters with their bounds (or Object)
2. Inserts type casts where necessary
3. Generates bridge methods for covariant returns

```java
// Source
public class Box<T> {
    public T get() { return value; }
}

// After erasure
public class Box {
    public Object get() { return value; }
    
    // Bridge method for subclasses
    public Object get() { return get(); }
}
```

---

## 🎯 Interviewer Insight

Advanced generics questions separate senior developers from juniors. Interviewers want to see that you understand:
- The PECS principle (Producer Extends, Consumer Super)
- Why type erasure exists (backward compatibility with Java 1.4)
- How to work around erasure limitations (type tokens, Class objects)
- F-bounded polymorphism for fluent APIs
- The difference between covariance and contravariance

Strong candidates can explain why `List<String>` is not a subtype of `List<Object>` and demonstrate the type token pattern used in frameworks like Hibernate and Gson.

## ⚡ Follow-up Questions

1. Why does Java use type erasure instead of reified generics like C#?
2. How does the `Collections.max()` method use wildcards to work with any Comparable type?
3. What is a bridge method and when does the compiler generate one?

## 📚 Related Topics

- [Generics Type System](../../sections/09-generics-type-system/README.md)
- [Collections Framework](../../sections/03-collections-framework/README.md)
- [Reflection](../../sections/07-jvm-internals/q02-class-loading-mechanism.md)
