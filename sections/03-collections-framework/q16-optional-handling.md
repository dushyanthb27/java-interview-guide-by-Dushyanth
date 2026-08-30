# Q16: Optional Handling and Best Practices

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> How does Optional work in Java? What are the best practices for using it?

---

## Answer

### What is Optional?

`Optional<T>` is a container that may or may not contain a non-null value. It was introduced in Java 8 to provide a better alternative to returning `null`.

```java
// Optional is a container: either contains a value or is empty
// It forces the caller to think about the absent-value case

// Without Optional
String name = findUserName(id);  // Could be null!
int length = name.length();      // NullPointerException risk!

// With Optional
Optional<String> name = findUserName(id);  // Explicitly might be absent
int length = name.map(String::length).orElse(0);  // Handled safely
```

### Creating Optional Instances

```java
// ✅ of() - value MUST be non-null
Optional<String> opt = Optional.of("Hello");
// Optional.of(null);  // ❌ NullPointerException!

// ✅ ofNullable() - value CAN be null
Optional<String> opt = Optional.ofNullable(getValue());  // Safe even if null
Optional<String> empty = Optional.ofNullable(null);       // Creates empty Optional

// ✅ empty() - explicitly empty
Optional<String> empty = Optional.empty();

// Rule of thumb:
// - Use of() when you're CERTAIN the value is non-null
// - Use ofNullable() when the value might be null
// - Use empty() when you want to explicitly return "no value"
```

### Retrieving Values

```java
Optional<String> name = Optional.of("Alice");
Optional<String> empty = Optional.empty();

// ✅ orElse - provide default value (always evaluated)
String result = name.orElse("Unknown");        // "Alice"
String result2 = empty.orElse("Unknown");      // "Unknown"

// ✅ orElseGet - provide default via Supplier (lazy evaluation)
String result = empty.orElseGet(() -> computeDefault());  // Only called if empty

// ✅ orElseThrow - throw exception if empty
String result = name.orElseThrow();  // "Alice" (Java 10+, throws NoSuchElementException)
String result = name.orElseThrow(() -> new UserNotFoundException("Not found"));

// ✅ or() - provide alternative Optional (Java 9+)
Optional<String> result = empty.or(() -> Optional.of("fallback"));

// ⚠️ get() - avoid! Throws NoSuchElementException if empty
String risky = name.get();    // Works, but no safer than null check
String crash = empty.get();   // ❌ NoSuchElementException!

// ✅ isPresent / isEmpty
if (name.isPresent()) {
    System.out.println(name.get());  // Safe after isPresent check
}
if (empty.isEmpty()) {  // Java 11+
    System.out.println("No value");
}

// ✅ ifPresent - execute action if value exists
name.ifPresent(n -> System.out.println("Name: " + n));

// ✅ ifPresentOrElse (Java 9+)
name.ifPresentOrElse(
    n -> System.out.println("Found: " + n),
    () -> System.out.println("Not found")
);
```

### Transforming Values

```java
Optional<String> name = Optional.of("  Alice  ");

// ✅ map - transform value if present
Optional<String> trimmed = name.map(String::trim);           // Optional["Alice"]
Optional<Integer> length = name.map(String::length);         // Optional[9]
Optional<String> upper = name.map(String::trim).map(String::toUpperCase); // Optional["ALICE"]

// If Optional is empty, map returns empty
Optional<String> empty = Optional.<String>empty();
Optional<Integer> len = empty.map(String::length);           // Optional.empty()

// ✅ flatMap - when transformation returns Optional (avoid Optional<Optional<T>>)
public Optional<Address> getAddress(User user) { ... }
public Optional<String> getCity(Address address) { ... }

// ❌ map creates Optional<Optional<String>>
Optional<Optional<String>> nested = user.map(u -> getAddress(u).flatMap(a -> getCity(a)));

// ✅ flatMap keeps it flat
Optional<String> city = findUser(id)
    .flatMap(this::getAddress)
    .flatMap(this::getCity);

// ✅ filter - keep value only if it matches predicate
Optional<String> long_name = name.filter(n -> n.trim().length() > 3);  // Optional["  Alice  "]
Optional<String> short_name = name.filter(n -> n.trim().length() > 10); // Optional.empty()
```

### orElse vs orElseGet

```java
// ⚠️ IMPORTANT difference: orElse ALWAYS evaluates the default

// orElse - default is ALWAYS computed (even when value is present!)
String result = Optional.of("Alice")
    .orElse(expensiveOperation());  // expensiveOperation() STILL RUNS!

// orElseGet - default is ONLY computed when empty (lazy)
String result = Optional.of("Alice")
    .orElseGet(() -> expensiveOperation());  // NOT called - Optional has value

// Rule: Use orElseGet when the default is expensive to compute
// Use orElse only for simple constants

// ✅ orElse with constant - fine
String name = opt.orElse("default");

// ✅ orElseGet with computation - correct
String name = opt.orElseGet(() -> database.getDefaultName());
```

### Practical Patterns

```java
// ✅ Pattern 1: Safe method chaining
String cityName = findUser(userId)
    .flatMap(User::getAddress)
    .flatMap(Address::getCity)
    .map(City::getName)
    .orElse("Unknown");

// ✅ Pattern 2: Converting null-returning APIs
Optional<String> prop = Optional.ofNullable(System.getProperty("my.config"));
String value = prop.orElse("default-value");

// ✅ Pattern 3: Conditional execution
findUser(userId).ifPresent(user -> {
    sendWelcomeEmail(user);
    logUserLogin(user);
});

// ✅ Pattern 4: Stream integration
List<String> validNames = users.stream()
    .map(User::getMiddleName)           // Stream<Optional<String>>
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());

// Better with flatMap (Java 9+ has Optional.stream())
List<String> validNames = users.stream()
    .map(User::getMiddleName)
    .flatMap(Optional::stream)          // Java 9+
    .collect(Collectors.toList());

// ✅ Pattern 5: First non-empty from multiple sources
Optional<String> config = getFromEnv()
    .or(() -> getFromFile())         // Java 9+
    .or(() -> getFromDefaults());
```

### When NOT to Use Optional

```java
// ❌ Don't use Optional for fields
public class User {
    private Optional<String> middleName;  // ❌ Not serializable, wasteful
    private String middleName;             // ✅ Use null (or "" for strings)
}

// ❌ Don't use Optional for method parameters
public void processUser(Optional<String> name) {  // ❌ Awkward API
    // Caller must wrap: processUser(Optional.of("Alice"))
}
public void processUser(String name) {              // ✅ Accept nullable
    String resolved = name != null ? name : "default";
}

// ❌ Don't use Optional in collections
List<Optional<String>> names;   // ❌ Just filter out nulls
List<String> names;              // ✅ Use null-free collections

// ❌ Don't use Optional for primitive types (boxing overhead)
Optional<Integer> age;           // ❌ Boxing
OptionalInt age;                 // ✅ Use primitive Optional variants

// ❌ Don't use Optional.get() without checking
String name = optional.get();    // ❌ Defeats the purpose

// ❌ Don't use isPresent() + get() pattern
if (optional.isPresent()) {      // ❌ This is just a fancy null check
    return optional.get();
}
return "default";
// ✅ Better:
return optional.orElse("default");

// ❌ Don't wrap already-not-null values
Optional.of(nonNullValue).orElse(default);  // ❌ Pointless wrapping
```

### Optional for Method Return Types

```java
// ✅ DO use Optional as return type to indicate possible absence
public Optional<User> findById(long id) {
    User user = database.query(id);
    return Optional.ofNullable(user);
}

// ✅ Good API design
public interface UserRepository {
    Optional<User> findById(long id);           // Might not exist
    Optional<User> findByEmail(String email);   // Might not exist
    List<User> findByDepartment(String dept);   // Empty list if none
    User save(User user);                        // Always returns (or throws)
}

// Usage
User user = userRepo.findById(42)
    .orElseThrow(() -> new UserNotFoundException("ID: 42"));
```

### Primitive Optional Variants

```java
// Avoid boxing with primitive-specific Optionals
OptionalInt optInt = OptionalInt.of(42);
OptionalLong optLong = OptionalLong.of(100L);
OptionalDouble optDouble = OptionalDouble.of(3.14);

// From streams
OptionalInt max = IntStream.of(1, 2, 3).max();
OptionalDouble avg = IntStream.of(1, 2, 3).average();

int value = max.orElse(0);
double average = avg.orElse(0.0);

// ⚠️ Primitive Optionals have fewer methods
// No map(), flatMap(), or filter()
// Use mapToObj() if you need to transform
```

### Common Anti-Patterns

```java
// ❌ Using Optional.of() with potentially null value
Optional<String> opt = Optional.of(mayBeNull);  // NullPointerException!

// ❌ Returning null instead of Optional.empty()
public Optional<User> findUser() {
    return null;  // ❌ Defeats the entire purpose!
}

// ❌ Nesting Optionals
Optional<Optional<String>> nested;  // ❌ Use flatMap instead

// ❌ Using Optional in constructors/setters
public User(Optional<String> name) { ... }  // ❌ Awkward

// ❌ Using Optional for control flow that should be an exception
Optional<User> user = findUser(id);
user.orElseThrow();  // ✅ OK for "must exist" scenarios
// But don't use Optional when absence is truly exceptional
```

### Best Practices

```java
// ✅ Use as method return type for potentially absent values
public Optional<String> findEmail(long userId) { ... }

// ✅ Use map/flatMap/filter for transformation chains
// ✅ Use orElse for simple defaults, orElseGet for expensive ones
// ✅ Use orElseThrow when absence is an error
// ✅ Use ifPresent for side-effect actions
// ✅ Use Optional.stream() (Java 9+) for stream integration
// ✅ Use primitive variants (OptionalInt, etc.) to avoid boxing
// ✅ Never return null from a method declared to return Optional
```

---

## Quick Reference

| Method | Returns | When Empty |
|--------|---------|------------|
| `get()` | Value | NoSuchElementException |
| `orElse(default)` | Value | default (always evaluated) |
| `orElseGet(supplier)` | Value | supplier.get() (lazy) |
| `orElseThrow()` | Value | NoSuchElementException |
| `orElseThrow(supplier)` | Value | supplier.get() thrown |
| `map(function)` | Optional | Optional.empty() |
| `flatMap(function)` | Optional | Optional.empty() |
| `filter(predicate)` | Optional | Optional.empty() |
| `ifPresent(consumer)` | void | No-op |
| `or(supplier)` | Optional | supplier.get() (Java 9+) |
| `stream()` | Stream | Empty stream (Java 9+) |

---

## Interview Tips
- Optional is for return types, NOT fields, parameters, or collections
- Know the critical difference between `orElse` and `orElseGet` (eager vs lazy)
- Demonstrate fluent chaining: map → flatMap → filter → orElse
- Explain why `get()` without `isPresent()` is an anti-pattern
- Mention `Optional.stream()` for Java 9+ stream integration
- Optional is NOT serializable - don't use for entity fields

---

[Prev: Q15 - Streams API](./q15-streams-api-collections.md) | [Next: Q17 - Immutable Collections](./q17-immutable-collections.md)
