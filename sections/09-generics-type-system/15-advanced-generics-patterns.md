# Q15: Advanced Generics Patterns

[Home](../../README.md) > [Generics & Type System](./README.md) > Advanced Generics Patterns

---

## 🔴 Difficulty: Expert

## Question
> What are the advanced generics patterns used in Java frameworks? Explain recursive type bounds, F-bounded polymorphism, type tokens, super type tokens, and the generic singleton factory.

---

## Quick Answer
Advanced generics patterns push Java's type system to its limits. **Recursive type bounds** enable fluent APIs and self-referential types. **F-bounded polymorphism** constrains a type to be parameterized by itself. **Type tokens** (`Class<T>`) capture type info at runtime. **Super type tokens** (anonymous subclass trick) capture complex generic types like `List<String>`. The **generic singleton factory** pattern creates a single instance that serves all type parameterizations safely.

---

## Detailed Explanation

### Recursive Type Bounds

```java
// Pattern: <T extends SomeInterface<T>>
// T is bounded by an interface that is itself parameterized by T

// Classic example: Comparable
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

// Why recursive? T must be comparable WITH ITSELF
// Integer extends Comparable<Integer>     ✅
// String extends Comparable<String>       ✅
// Object does NOT extend Comparable<Object> ❌

// More flexible version (supports inherited Comparable):
public static <T extends Comparable<? super T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

### F-Bounded Polymorphism (Simulated Self-Types)

```java
// F-bounded: a type parameter that references the type being defined
// Used to return "this type" in base class methods

// The base class references its own subtype
public abstract class Shape<S extends Shape<S>> {
    private double x, y;

    @SuppressWarnings("unchecked")
    protected S self() {
        return (S) this;
    }

    public S moveTo(double x, double y) {
        this.x = x;
        this.y = y;
        return self();  // Returns the CONCRETE subtype
    }

    public S scale(double factor) {
        // Scale logic...
        return self();
    }

    public abstract double area();
}

// Concrete shapes — S is fixed to the concrete class
public class Circle extends Shape<Circle> {
    private double radius;

    public Circle(double radius) { this.radius = radius; }

    public Circle withRadius(double radius) {
        this.radius = radius;
        return this;
    }

    @Override
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle extends Shape<Rectangle> {
    private double width, height;

    public Rectangle(double w, double h) { this.width = w; this.height = h; }

    public Rectangle resize(double w, double h) {
        this.width = w;
        this.height = h;
        return this;
    }

    @Override
    public double area() { return width * height; }
}

// Fluent chaining returns the CORRECT subtype at every step
Circle c = new Circle(5)
    .moveTo(10, 20)       // Returns Circle, not Shape
    .scale(2.0)           // Returns Circle, not Shape
    .withRadius(7);       // Returns Circle — subclass method accessible

Rectangle r = new Rectangle(3, 4)
    .moveTo(0, 0)         // Returns Rectangle, not Shape
    .resize(5, 6);        // Returns Rectangle
```

### Type Tokens (Class&lt;T&gt;)

```java
// Type token: using Class<T> to carry runtime type information
// Compensates for erasure — Class objects are NOT erased

// Pattern: pass Class<T> to retain type info at runtime
public class TypeSafeMap {
    private final Map<Class<?>, Object> map = new HashMap<>();

    // Class<T> serves as both KEY and TYPE GUARANTEE
    public <T> void put(Class<T> key, T value) {
        map.put(key, key.cast(value));  // Runtime check
    }

    public <T> T get(Class<T> key) {
        return key.cast(map.get(key));  // Type-safe cast
    }
}

// Usage
TypeSafeMap favorites = new TypeSafeMap();
favorites.put(String.class, "Java");
favorites.put(Integer.class, 42);
favorites.put(Class.class, TypeSafeMap.class);

String fav = favorites.get(String.class);    // ✅ Compile-time safe
Integer num = favorites.get(Integer.class);  // ✅ No casting

// This pattern is used in:
// - java.util.ServiceLoader<S>
// - Spring's ApplicationContext.getBean(Class<T>)
// - Guice's Key<T> and TypeLiteral<T>
```

### Super Type Tokens (TypeReference)

```java
// Problem: Class<T> can't capture parameterized types
// Class<List<String>> doesn't exist — erasure removes the <String>

// Solution: Anonymous subclass trick preserves generic type info
// The generic superclass signature is stored in class metadata

public abstract class TypeReference<T> {
    private final Type type;

    protected TypeReference() {
        // Get generic superclass: TypeReference<List<String>>
        Type superclass = getClass().getGenericSuperclass();
        if (superclass instanceof ParameterizedType pt) {
            this.type = pt.getActualTypeArguments()[0];
        } else {
            throw new RuntimeException("Missing type parameter");
        }
    }

    public Type getType() { return type; }

    public Class<?> getRawType() {
        if (type instanceof ParameterizedType pt) {
            return (Class<?>) pt.getRawType();
        }
        return (Class<?>) type;
    }
}

// Usage — anonymous subclass captures the type
TypeReference<List<String>> listOfStrings = new TypeReference<>() {};
System.out.println(listOfStrings.getType());
// Output: java.util.List<java.lang.String>

TypeReference<Map<String, List<Integer>>> complexType = new TypeReference<>() {};
System.out.println(complexType.getType());
// Output: java.util.Map<java.lang.String, java.util.List<java.lang.Integer>>

// Used extensively in:
// - Jackson: new TypeReference<List<User>>() {}
// - Guava: new TypeToken<List<String>>() {}
// - Guice: new TypeLiteral<List<String>>() {}
```

### How Super Type Tokens Work

```java
// When you write: new TypeReference<List<String>>() {}
// The compiler generates an anonymous class like:

// class Anonymous$1 extends TypeReference<List<String>> { }
//
// The bytecode stores the Signature attribute:
//   Lcom/example/TypeReference<Ljava/util/List<Ljava/lang/String;>;>;
//
// This is class METADATA — not erased!
// Erasure only removes type info from INSTANCES and local variables
// Class declarations retain their generic signatures

// That's why getGenericSuperclass() returns:
//   TypeReference<List<String>>  — full type info preserved!
```

### Generic Singleton Factory

```java
// A single instance serves ALL type parameterizations
// Safe because the operation is type-agnostic

// Example: identity function
public class GenericSingletonFactory {

    // Single instance — works for any type
    private static final UnaryOperator<Object> IDENTITY = t -> t;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY;  // Safe: identity works for any type
    }
}

// Usage — same instance, different types
UnaryOperator<String> stringIdentity = GenericSingletonFactory.identityFunction();
UnaryOperator<Integer> intIdentity = GenericSingletonFactory.identityFunction();

// Both reference the SAME object but are type-safe at compile time
String s = stringIdentity.apply("hello");  // ✅ Returns String
Integer n = intIdentity.apply(42);          // ✅ Returns Integer

// Real-world example: Collections.emptyList()
// There's ONE EmptyList instance, but it serves as List<String>, List<Integer>, etc.
@SuppressWarnings("unchecked")
public static <T> List<T> emptyList() {
    return (List<T>) EMPTY_LIST;  // Single instance, any type
}

// Another example: Comparator.naturalOrder()
@SuppressWarnings("unchecked")
public static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
    return (Comparator<T>) NaturalOrderComparator.INSTANCE;
}
```

### Generics in Annotation Processors

```java
// Annotation processors use the javax.lang.model API for generics

// Processing generic types at compile time:
@SupportedAnnotationTypes("com.example.AutoBuilder")
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (Element element : roundEnv.getElementsAnnotatedWith(AutoBuilder.class)) {
            TypeElement typeElement = (TypeElement) element;

            // Get type parameters of the annotated class
            List<? extends TypeParameterElement> typeParams =
                typeElement.getTypeParameters();

            for (TypeParameterElement tp : typeParams) {
                System.out.println("Type param: " + tp.getSimpleName());
                for (TypeMirror bound : tp.getBounds()) {
                    System.out.println("  Bound: " + bound);
                }
            }
        }
        return true;
    }
}
```

### Type-Safe Builder with Phantom Types

```java
// Phantom types: type parameters used ONLY for compile-time checks
// Not stored in any field — purely for type-level state

// Marker interfaces (phantom types)
interface Unset {}
interface Set {}

public class HttpRequest<UrlState, MethodState> {
    private String url;
    private String method;
    private Map<String, String> headers = new HashMap<>();

    private HttpRequest() {}

    public static HttpRequest<Unset, Unset> builder() {
        return new HttpRequest<>();
    }

    // Setting URL changes state from Unset to Set
    public HttpRequest<Set, MethodState> url(String url) {
        this.url = url;
        @SuppressWarnings("unchecked")
        HttpRequest<Set, MethodState> self = (HttpRequest<Set, MethodState>) (Object) this;
        self.url = url;
        return self;
    }

    // Setting method changes state from Unset to Set
    public HttpRequest<UrlState, Set> method(String method) {
        this.method = method;
        @SuppressWarnings("unchecked")
        HttpRequest<UrlState, Set> self = (HttpRequest<UrlState, Set>) (Object) this;
        self.method = method;
        return self;
    }

    // Headers don't change state
    public HttpRequest<UrlState, MethodState> header(String key, String value) {
        headers.put(key, value);
        return this;
    }

    // build() ONLY available when BOTH url and method are Set
    // This is enforced at COMPILE TIME!
}

// Extension method restricted to fully-configured requests
class HttpRequestExecutor {
    public static Response execute(HttpRequest<Set, Set> request) {
        // Can only call with a fully configured request
        return null;
    }
}

// Usage:
// HttpRequestExecutor.execute(
//     HttpRequest.builder().url("https://api.example.com")  // <Set, Unset>
// );  // ❌ COMPILE ERROR — method not set!

// HttpRequestExecutor.execute(
//     HttpRequest.builder()
//         .url("https://api.example.com")    // <Set, Unset>
//         .method("GET")                      // <Set, Set>
//         .header("Accept", "application/json")
// );  // ✅ Compiles — both URL and method are Set
```

### Intersection Types and Capture

```java
// Intersection types in generics: T extends A & B
// The captured type satisfies BOTH bounds

public static <T extends Comparable<T> & Serializable> T process(T item) {
    // Can call methods from Comparable AND Serializable
    return item;
}

// Wildcard capture helper (used in JDK source code)
// When you need to work with a captured wildcard type

// Public method with wildcard
public static void reverse(List<?> list) {
    reverseHelper(list);  // Captures ? as a concrete type T
}

// Private helper captures the wildcard
private static <T> void reverseHelper(List<T> list) {
    // Now T is a concrete (captured) type — can read AND write
    for (int i = 0, j = list.size() - 1; i < j; i++, j--) {
        T temp = list.get(i);
        list.set(i, list.get(j));
        list.set(j, temp);
    }
}
```

### Token-Based Generic Registry (Framework Pattern)

```java
// Used in dependency injection frameworks (Spring, Guice)
public class DIContainer {
    private final Map<TypeKey<?>, Supplier<?>> bindings = new HashMap<>();

    public <T> void bind(Class<T> type, Supplier<T> supplier) {
        bindings.put(TypeKey.of(type), supplier);
    }

    public <T> void bind(TypeReference<T> typeRef, Supplier<T> supplier) {
        bindings.put(TypeKey.of(typeRef), supplier);
    }

    @SuppressWarnings("unchecked")
    public <T> T resolve(Class<T> type) {
        Supplier<T> supplier = (Supplier<T>) bindings.get(TypeKey.of(type));
        if (supplier == null) throw new RuntimeException("No binding for " + type);
        return supplier.get();
    }

    @SuppressWarnings("unchecked")
    public <T> T resolve(TypeReference<T> typeRef) {
        Supplier<T> supplier = (Supplier<T>) bindings.get(TypeKey.of(typeRef));
        if (supplier == null) throw new RuntimeException("No binding for " + typeRef);
        return supplier.get();
    }

    private record TypeKey<T>(Type type) {
        static <T> TypeKey<T> of(Class<T> clazz) { return new TypeKey<>(clazz); }
        static <T> TypeKey<T> of(TypeReference<T> ref) { return new TypeKey<>(ref.getType()); }
    }
}

// Usage:
DIContainer container = new DIContainer();
container.bind(UserService.class, UserServiceImpl::new);
container.bind(
    new TypeReference<List<String>>() {},
    () -> List.of("default")
);

UserService service = container.resolve(UserService.class);
List<String> defaults = container.resolve(new TypeReference<>() {});
```

---

## Anti-Patterns

```java
// ❌ F-bounded type without the self() safety net
public abstract class Base<T extends Base<T>> {
    public T doSomething() {
        return (T) this;  // Unsafe if subclass forgets to parameterize correctly
    }
}
// Nothing prevents: class Wrong extends Base<SomeOtherClass> { }

// ❌ Using TypeReference without the anonymous subclass {}
// TypeReference<List<String>> ref = new TypeReference<List<String>>();
// Missing {} — won't capture generic type!

// ❌ Overcomplicating simple problems with advanced generics
// Not everything needs recursive bounds or phantom types
// Simple <T> is often sufficient

// ❌ Creating mutable generic singleton
private static final List<Object> SHARED = new ArrayList<>();
// Mutations are visible to ALL type parameterizations — dangerous!
```

## Best Practices

```java
// ✅ Use recursive bounds for fluent builder hierarchies
public abstract class Builder<B extends Builder<B>> {
    protected abstract B self();
}

// ✅ Use super type tokens (TypeReference) for capturing complex types
new TypeReference<Map<String, List<Integer>>>() {}

// ✅ Generic singleton factory only for IMMUTABLE/STATELESS instances
private static final Comparator<Object> INSTANCE = ...;
public static <T> Comparator<T> natural() {
    return (Comparator<T>) INSTANCE;  // Safe: stateless
}

// ✅ Document complex generic patterns thoroughly
/**
 * F-bounded polymorphism pattern.
 * @param <S> the concrete subtype (must be the implementing class itself)
 */
public abstract class Shape<S extends Shape<S>> { }

// ✅ Use wildcard capture helper for List<?> manipulation
public static void sort(List<?> list) {
    sortHelper(list);
}
private static <T extends Comparable<? super T>> void sortHelper(List<T> list) {
    Collections.sort(list);
}
```

---

## Interview Tips
- **Recursive type bounds** (`T extends Comparable<T>`) — used in Enum, sorting, fluent APIs
- **F-bounded polymorphism** — simulates self-types for builder pattern inheritance
- **Type tokens** (`Class<T>`) — compensate for erasure at runtime (Spring's `getBean(Class<T>)`)
- **Super type tokens** — anonymous subclass trick to capture `List<String>` at runtime (Jackson's `TypeReference`)
- **Generic singleton factory** — one instance, many types (only for stateless operations)
- **Phantom types** — compile-time state tracking without runtime overhead
- **Wildcard capture** — private helper method to convert `?` into a named type parameter

---

[← Q14: Custom Generic Types](./q14-custom-generic-types.md) | [Back to Section Index](./README.md)
