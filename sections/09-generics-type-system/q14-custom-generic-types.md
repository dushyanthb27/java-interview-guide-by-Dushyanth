# Q14: How to Design Custom Generic Types?

[Home](../../README.md) > [Generics & Type System](./README.md) > Custom Generic Types

---

## 🟠 Difficulty: Advanced

## Question
> How do you design custom generic classes and interfaces? What patterns enable type-safe, reusable APIs?

---

## Quick Answer
Designing custom generic types involves choosing appropriate type parameters, applying bounds for safety, implementing generic interfaces correctly, and using patterns like **type-safe heterogeneous containers**, **generic builders**, and **generic factory methods**. Good generic design maximizes reusability while maintaining compile-time type safety. The key is to genericize only what varies and keep the API intuitive.

---

## Detailed Explanation

### Generic Result/Response Wrapper

```java
// A type-safe API response wrapper
public class Result<T> {
    private final T data;
    private final String error;
    private final boolean success;

    private Result(T data, String error, boolean success) {
        this.data = data;
        this.error = error;
        this.success = success;
    }

    // Factory methods — clear intent
    public static <T> Result<T> success(T data) {
        return new Result<>(data, null, true);
    }

    public static <T> Result<T> failure(String error) {
        return new Result<>(null, error, false);
    }

    // Safe accessors
    public T getData() {
        if (!success) throw new IllegalStateException("No data in failed result");
        return data;
    }

    public String getError() {
        if (success) throw new IllegalStateException("No error in successful result");
        return error;
    }

    public boolean isSuccess() { return success; }

    // Monadic operations
    public <R> Result<R> map(Function<T, R> mapper) {
        if (success) {
            return Result.success(mapper.apply(data));
        }
        return Result.failure(error);
    }

    public <R> Result<R> flatMap(Function<T, Result<R>> mapper) {
        if (success) {
            return mapper.apply(data);
        }
        return Result.failure(error);
    }

    public T orElse(T defaultValue) {
        return success ? data : defaultValue;
    }

    public T orElseThrow(Supplier<? extends RuntimeException> exSupplier) {
        if (success) return data;
        throw exSupplier.get();
    }
}

// Usage — fully type-safe
Result<User> userResult = userService.findById(1L);
String name = userResult.map(User::getName).orElse("Unknown");

Result<Order> orderResult = userResult
    .flatMap(user -> orderService.getLatestOrder(user.getId()));
```

### Generic Builder Pattern

```java
public class GenericBuilder<T> {
    private final Supplier<T> constructor;
    private final List<Consumer<T>> modifiers = new ArrayList<>();

    public GenericBuilder(Supplier<T> constructor) {
        this.constructor = constructor;
    }

    public static <T> GenericBuilder<T> of(Supplier<T> constructor) {
        return new GenericBuilder<>(constructor);
    }

    public <V> GenericBuilder<T> with(BiConsumer<T, V> setter, V value) {
        modifiers.add(instance -> setter.accept(instance, value));
        return this;
    }

    public T build() {
        T instance = constructor.get();
        modifiers.forEach(modifier -> modifier.accept(instance));
        return instance;
    }
}

// Usage — works for ANY class with setters
User user = GenericBuilder.of(User::new)
    .with(User::setName, "Alice")
    .with(User::setAge, 30)
    .with(User::setEmail, "alice@test.com")
    .build();

Order order = GenericBuilder.of(Order::new)
    .with(Order::setProduct, "Widget")
    .with(Order::setQuantity, 5)
    .build();
```

### Generic Repository Interface

```java
// Base repository with generic CRUD operations
public interface Repository<T, ID> {

    Optional<T> findById(ID id);
    List<T> findAll();
    List<T> findAll(Predicate<T> filter);
    T save(T entity);
    void deleteById(ID id);
    long count();
    boolean existsById(ID id);
}

// Abstract implementation with common logic
public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
    protected final Map<ID, T> store = new ConcurrentHashMap<>();
    protected abstract ID getId(T entity);

    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<T> findAll() {
        return new ArrayList<>(store.values());
    }

    @Override
    public List<T> findAll(Predicate<T> filter) {
        return store.values().stream()
            .filter(filter)
            .collect(Collectors.toList());
    }

    @Override
    public T save(T entity) {
        store.put(getId(entity), entity);
        return entity;
    }

    @Override
    public void deleteById(ID id) {
        store.remove(id);
    }

    @Override
    public long count() { return store.size(); }

    @Override
    public boolean existsById(ID id) { return store.containsKey(id); }
}

// Concrete implementation — type parameters fixed
public class UserRepository extends AbstractRepository<User, Long> {
    @Override
    protected Long getId(User user) {
        return user.getId();
    }
}
```

### Type-Safe Heterogeneous Container

```java
// Stores different types safely — using Class<T> as the key
public class TypeSafeRegistry {
    private final Map<Class<?>, Object> registry = new HashMap<>();

    public <T> void register(Class<T> type, T instance) {
        registry.put(type, type.cast(instance));  // Runtime type check
    }

    public <T> T get(Class<T> type) {
        return type.cast(registry.get(type));  // Type-safe retrieval
    }

    public <T> Optional<T> getOptional(Class<T> type) {
        return Optional.ofNullable(registry.get(type)).map(type::cast);
    }

    public boolean contains(Class<?> type) {
        return registry.containsKey(type);
    }
}

// Usage — each entry has its own type, all type-safe
TypeSafeRegistry config = new TypeSafeRegistry();
config.register(String.class, "application-name");
config.register(Integer.class, 8080);
config.register(Duration.class, Duration.ofSeconds(30));

String appName = config.get(String.class);    // ✅ Returns String
Integer port = config.get(Integer.class);     // ✅ Returns Integer
Duration timeout = config.get(Duration.class); // ✅ Returns Duration
// No casting needed — compile-time safe!
```

### Generic Event System

```java
// Type-safe event bus using generics
public class EventBus {
    private final Map<Class<?>, List<Consumer<?>>> listeners = new ConcurrentHashMap<>();

    public <E> void subscribe(Class<E> eventType, Consumer<E> listener) {
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                .add(listener);
    }

    @SuppressWarnings("unchecked")
    public <E> void publish(E event) {
        List<Consumer<?>> handlers = listeners.get(event.getClass());
        if (handlers != null) {
            for (Consumer<?> handler : handlers) {
                ((Consumer<E>) handler).accept(event);
            }
        }
    }
}

// Type-safe events
record UserCreated(String username) {}
record OrderPlaced(long orderId, double amount) {}

// Usage
EventBus bus = new EventBus();
bus.subscribe(UserCreated.class, e -> log("User: " + e.username()));
bus.subscribe(OrderPlaced.class, e -> log("Order: " + e.orderId()));

bus.publish(new UserCreated("alice"));     // Triggers UserCreated handler
bus.publish(new OrderPlaced(1L, 99.99));   // Triggers OrderPlaced handler
```

### Generic Converter/Mapper

```java
// Type-safe converter registry
public class ConverterRegistry {
    private final Map<ConversionKey<?, ?>, Function<?, ?>> converters = new HashMap<>();

    public <S, T> void register(Class<S> source, Class<T> target,
                                 Function<S, T> converter) {
        converters.put(new ConversionKey<>(source, target), converter);
    }

    @SuppressWarnings("unchecked")
    public <S, T> T convert(S source, Class<T> targetType) {
        ConversionKey<S, T> key = new ConversionKey<>(
            (Class<S>) source.getClass(), targetType
        );
        Function<S, T> converter = (Function<S, T>) converters.get(key);
        if (converter == null) {
            throw new IllegalArgumentException(
                "No converter: " + source.getClass() + " → " + targetType
            );
        }
        return converter.apply(source);
    }

    private record ConversionKey<S, T>(Class<S> source, Class<T> target) {}
}

// Usage
ConverterRegistry registry = new ConverterRegistry();
registry.register(String.class, Integer.class, Integer::parseInt);
registry.register(String.class, LocalDate.class, LocalDate::parse);
registry.register(Integer.class, String.class, String::valueOf);

Integer num = registry.convert("42", Integer.class);        // 42
String text = registry.convert(42, String.class);           // "42"
LocalDate date = registry.convert("2024-01-01", LocalDate.class);
```

### Generic Pair and Tuple Types

```java
// Immutable generic Pair
public record Pair<A, B>(A first, B second) {

    public <C> Pair<C, B> mapFirst(Function<A, C> mapper) {
        return new Pair<>(mapper.apply(first), second);
    }

    public <C> Pair<A, C> mapSecond(Function<B, C> mapper) {
        return new Pair<>(first, mapper.apply(second));
    }

    public <R> R combine(BiFunction<A, B, R> combiner) {
        return combiner.apply(first, second);
    }
}

// Usage
Pair<String, Integer> entry = new Pair<>("Alice", 30);
Pair<String, String> mapped = entry.mapSecond(String::valueOf);  // ("Alice", "30")
String result = entry.combine((name, age) -> name + " is " + age);
```

### Fluent API with Self-Type Pattern

```java
// Abstract base with recursive generic bound
public abstract class AbstractQuery<T, SELF extends AbstractQuery<T, SELF>> {
    protected Class<T> entityType;
    protected String whereClause;
    protected String orderBy;
    protected int limit = -1;

    @SuppressWarnings("unchecked")
    protected SELF self() { return (SELF) this; }

    public SELF where(String clause) {
        this.whereClause = clause;
        return self();
    }

    public SELF orderBy(String field) {
        this.orderBy = field;
        return self();
    }

    public SELF limit(int limit) {
        this.limit = limit;
        return self();
    }

    public abstract List<T> execute();
}

// Concrete subclass with additional methods
public class UserQuery extends AbstractQuery<User, UserQuery> {
    public UserQuery byRole(String role) {
        return where("role = '" + role + "'");
    }

    @Override
    public List<User> execute() {
        // Build and execute query...
        return List.of();
    }
}

// Fluent usage — all methods return UserQuery, not AbstractQuery
List<User> admins = new UserQuery()
    .byRole("ADMIN")        // Returns UserQuery
    .orderBy("name")        // Returns UserQuery (not AbstractQuery)
    .limit(10)              // Returns UserQuery
    .execute();
```

---

## Anti-Patterns

```java
// ❌ Over-genericizing — adding type parameters that aren't needed
public class Logger<T> {          // T is never used meaningfully
    public void log(String message) { System.out.println(message); }
}

// ❌ Returning raw collections from generic methods
public List getItems() { return items; }  // Should return List<T>

// ❌ Using Object where generics should be used
public class Container {
    private Object value;               // No type safety for callers
    public Object getValue() { return value; }
}

// ❌ Overly complex generic signatures that are hard to read
public <T extends Comparable<? super T> & Serializable,
        U extends Collection<? extends T>,
        V extends Function<? super T, ? extends U>> V process(U input) { ... }
```

## Best Practices

```java
// ✅ Genericize what varies, keep the rest concrete
public class Cache<K, V> {        // K and V vary per usage
    private final Duration ttl;    // Duration is always Duration
}

// ✅ Use factory methods for clean type inference
public static <T> Result<T> success(T data) {
    return new Result<>(data, null, true);
}
// Caller: Result<User> r = Result.success(user);  // T inferred

// ✅ Apply PECS to your API parameters
public <T> void process(
    List<? extends T> source,     // Read from
    Consumer<? super T> handler   // Write to
) { ... }

// ✅ Use Class<T> tokens for runtime type operations
public <T> T getInstance(Class<T> type) {
    return type.cast(registry.get(type));
}

// ✅ Document type parameters in Javadoc
/**
 * @param <T> the type of entity managed by this repository
 * @param <ID> the type of the entity's primary key
 */
public interface Repository<T, ID> { }
```

---

## Interview Tips
- Good generic design: genericize what **varies**, keep the rest concrete
- Use **factory methods** (`Result.success(data)`) for clean type inference
- **Type-safe heterogeneous containers** use `Class<T>` as keys for mixed-type storage
- The **self-type pattern** (recursive bounds) enables fluent APIs with inheritance
- Avoid over-genericizing — if a type parameter is used once, a wildcard may suffice
- Always apply **PECS** to method parameters in your generic APIs

---

[← Q13: Covariance & Contravariance](./q13-covariance-contravariance.md) | [Next: Q15 - Advanced Generics Patterns →](./q15-advanced-generics-patterns.md)
