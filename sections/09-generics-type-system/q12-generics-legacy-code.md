# Q12: How to Mix Generics with Legacy Code?

[Home](../../README.md) > [Generics & Type System](./README.md) > Generics & Legacy Code

---

## 🟡 Difficulty: Intermediate

## Question
> How do you mix generic code with pre-Java 5 legacy code that uses raw types? What are the risks and migration strategies?

---

## Quick Answer
Java generics are designed for **backward compatibility** through erasure — generic and legacy raw-type code can coexist. Raw types (`List` instead of `List<String>`) compile but generate **unchecked warnings**. When passing generic collections to legacy code, you risk **heap pollution** — the legacy code might insert wrong types. Migration should be incremental, using `@SuppressWarnings("unchecked")` judiciously, and verifying with `Collections.checkedXxx()` wrappers where needed.

---

## Detailed Explanation

### What Are Raw Types?

```java
// Raw type: generic class used WITHOUT type parameters
List rawList = new ArrayList();          // Raw List
Map rawMap = new HashMap();              // Raw Map
Iterator rawIterator = rawList.iterator(); // Raw Iterator

// Parameterized type (correct usage)
List<String> typedList = new ArrayList<>();   // ✅
Map<String, Integer> typedMap = new HashMap<>(); // ✅

// Raw types exist ONLY for backward compatibility with pre-Java 5 code
// They should NEVER be used in new code
```

### Legacy Code Example

```java
// Pre-Java 5 legacy code (no generics)
public class LegacyService {

    // Returns raw List — no type safety
    public List getUsers() {
        List users = new ArrayList();
        users.add(new User("Alice"));
        users.add(new User("Bob"));
        return users;
    }

    // Accepts raw List — can add anything
    public void processItems(List items) {
        for (Object item : items) {
            System.out.println(item);
        }
    }

    // Dangerous legacy method — adds non-User items
    public void populateList(List items) {
        items.add("This is not a User!");  // ⚠️ Adds wrong type
        items.add(42);                     // ⚠️ Adds wrong type
    }
}
```

### Calling Legacy Code from Generic Code

```java
public class ModernService {

    private LegacyService legacy = new LegacyService();

    public void example() {
        // Receiving raw List from legacy code
        @SuppressWarnings("unchecked")  // We trust legacy returns Users
        List<User> users = legacy.getUsers();  // Unchecked assignment warning
        // Works at runtime because the list actually contains Users

        // Passing generic List to legacy code
        List<String> names = new ArrayList<>(List.of("Alice", "Bob"));
        legacy.processItems(names);  // ✅ Works — legacy reads with Object

        // ⚠️ DANGER: Passing a typed list to a method that may corrupt it
        List<User> userList = new ArrayList<>();
        userList.add(new User("Alice"));
        legacy.populateList(userList);  // 💥 Adds String and Integer!

        // Later access causes ClassCastException
        // User u = userList.get(1);  // 💥 "This is not a User!" is a String
    }
}
```

### Unchecked Warnings Explained

```java
// The compiler generates warnings when type safety cannot be guaranteed

// Warning: unchecked assignment
List<String> list = new ArrayList();  // Raw type on RHS
// Warning: "unchecked assignment: ArrayList to List<String>"

// Warning: unchecked call
List rawList = new ArrayList();
rawList.add("hello");  // Unchecked call to add(E)
// Warning: "unchecked call to add(E) as a member of raw type List"

// Warning: unchecked cast
Object obj = getList();
List<String> strings = (List<String>) obj;  // Unchecked cast
// Warning: "unchecked cast: Object to List<String>"

// Warning: unchecked conversion
List<String> typed = new ArrayList<>();
List raw = typed;            // ✅ OK — widening to raw type
List<String> back = raw;     // ⚠️ Warning — narrowing from raw to typed
```

### @SuppressWarnings("unchecked")

```java
// Proper usage: as narrow as possible, with justification

// ❌ Bad — too broad, hides all warnings in the method
@SuppressWarnings("unchecked")
public void entireMethod() {
    List<String> a = legacyMethod();
    List<Integer> b = anotherLegacyMethod();
    // ALL unchecked warnings silenced — can't see new issues
}

// ✅ Good — narrow scope with comment
public void narrowSuppress() {
    // Safe: legacyMethod always returns List of Strings (verified)
    @SuppressWarnings("unchecked")
    List<String> users = (List<String>) legacyMethod();

    // This line still produces warnings if incorrect — visible
    List<Integer> numbers = anotherMethod();
}

// ✅ Best — extract to local variable for minimal scope
public List<User> getUsers() {
    @SuppressWarnings("unchecked")  // Safe: DAO always returns User objects
    List<User> result = (List<User>) legacyDao.findAll();
    return result;
}
```

### Collections.checkedXxx() Wrappers

```java
// Checked collections enforce type safety at RUNTIME
// Useful when passing generic collections to legacy code

List<String> safeList = Collections.checkedList(
    new ArrayList<>(), String.class
);

// Any attempt to add wrong type throws ClassCastException immediately
safeList.add("hello");          // ✅ OK
// safeList.add((Object) 42);   // 💥 ClassCastException — caught immediately!

// Usage with legacy code
List<String> names = Collections.checkedList(new ArrayList<>(), String.class);
names.add("Alice");
legacyService.populateList(names);
// 💥 ClassCastException when legacy tries to add non-String

// Available checked wrappers:
Collections.checkedList(list, String.class);
Collections.checkedSet(set, Integer.class);
Collections.checkedMap(map, String.class, Integer.class);
Collections.checkedCollection(collection, User.class);
Collections.checkedQueue(queue, Task.class);  // Java 8+
```

### Migration Strategy: Incremental Generification

```java
// Step 1: Add type parameters to class signatures
// Before:
public class Repository {
    private Map cache = new HashMap();
    public Object findById(Object id) { return cache.get(id); }
}

// After Step 1: Add generics to the interface
public class Repository<T, ID> {
    @SuppressWarnings("unchecked")
    private Map<ID, T> cache = new HashMap<>();

    public T findById(ID id) {
        return cache.get(id);
    }
}

// Step 2: Update method signatures
// Step 3: Update callers one by one
// Step 4: Remove @SuppressWarnings as raw usage is eliminated
```

### Binary Compatibility

```java
// Generic and raw types are BINARY COMPATIBLE
// Compiled bytecode is identical (due to erasure)

// Library v1 (pre-generics):
// public class Box { Object value; }

// Library v2 (with generics):
// public class Box<T> { T value; }

// Client code compiled against v1 works with v2 — and vice versa
// This is WHY erasure exists — to maintain backward compatibility

// The .class files are the same structure
// Only the Signature attribute (metadata) differs
```

### Mixing Raw and Generic in Collections API

```java
// Java's own APIs evolved from raw to generic:

// Java 1.2 (raw):       List, Map, Set
// Java 5   (generic):   List<E>, Map<K,V>, Set<E>

// Old code: new ArrayList(), new HashMap()
// New code: new ArrayList<>(), new HashMap<>()

// Both work because of erasure — same bytecode at runtime
// The generics are compile-time only
```

### Common Migration Patterns

```java
// Pattern 1: Wrapping legacy return values
public List<User> getUsers() {
    List rawList = legacyService.getUsers();

    // Option A: Unchecked cast (trust legacy code)
    @SuppressWarnings("unchecked")
    List<User> users = rawList;
    return users;

    // Option B: Defensive copy with type check
    List<User> users = new ArrayList<>();
    for (Object item : rawList) {
        if (item instanceof User) {
            users.add((User) item);
        }
    }
    return users;
}

// Pattern 2: Protecting collections passed to legacy code
public void sendToLegacy(List<String> data) {
    // Pass an unmodifiable copy — legacy can't corrupt original
    List<String> safeCopy = List.copyOf(data);
    legacyService.process(safeCopy);
    // Or use checked wrapper if legacy must modify it
}

// Pattern 3: Adapter for legacy generic-unaware interface
public class LegacyAdapter<T> {
    private final LegacyStore store;  // Raw legacy class
    private final Class<T> type;

    public LegacyAdapter(LegacyStore store, Class<T> type) {
        this.store = store;
        this.type = type;
    }

    public T get(String key) {
        Object value = store.get(key);
        return type.cast(value);  // Type-safe cast
    }

    public void put(String key, T value) {
        store.put(key, value);
    }
}
```

---

## Anti-Patterns

```java
// ❌ Using raw types in new code
List list = new ArrayList();  // NO — always use List<SomeType>

// ❌ Blindly suppressing all unchecked warnings
@SuppressWarnings("unchecked")
public class EverythingSuppressed { /* entire class */ }

// ❌ Ignoring unchecked warnings without analysis
List<String> items = getLegacyData();  // Warning ignored — risk of runtime crash

// ❌ Passing typed collections to legacy code that mutates them
List<User> users = new ArrayList<>();
legacyService.addStuff(users);  // Legacy may add non-User objects
```

## Best Practices

```java
// ✅ Never use raw types in new code
List<String> names = new ArrayList<>();

// ✅ Narrow @SuppressWarnings to smallest possible scope
@SuppressWarnings("unchecked")
List<User> users = (List<User>) legacyDao.findAll();

// ✅ Use Collections.checkedXxx() when interacting with legacy
List<String> safe = Collections.checkedList(new ArrayList<>(), String.class);
legacyMethod(safe);  // Throws immediately on type violation

// ✅ Defensively copy when receiving from legacy
List<User> users = new ArrayList<>();
for (Object obj : legacyService.getAll()) {
    users.add((User) obj);  // Explicit cast — fails fast if wrong
}

// ✅ Write unit tests that cover legacy integration boundaries
@Test
void legacyServiceReturnsCorrectTypes() {
    List<User> users = service.getUsers();
    assertThat(users).allMatch(item -> item instanceof User);
}
```

---

## Interview Tips
- Raw types exist **solely for backward compatibility** — never use them in new code
- Type erasure ensures **binary compatibility** between generic and pre-generic bytecode
- `@SuppressWarnings("unchecked")` should be as narrow as possible with a justifying comment
- `Collections.checkedList/Map/Set()` adds **runtime** type enforcement
- Migration should be incremental — add generics to interfaces first, then implementations
- Heap pollution occurs when legacy code corrupts a typed collection

---

[← Q11: Generic Arrays](./q11-generic-arrays.md) | [Next: Q13 - Covariance & Contravariance →](./q13-covariance-contravariance.md)
