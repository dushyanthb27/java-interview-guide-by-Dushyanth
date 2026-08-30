# Q4: How Does Serialization Work?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
How does Java serialization work? What are the best practices and security concerns?

---

## Answer

**Serialization** is the process of converting an object's state into a byte stream. **Deserialization** is the reverse — reconstructing the object from bytes. Java provides built-in serialization through the `Serializable` interface.

---

## Basic Serialization

```java
// ✅ Make a class serializable
public class Employee implements Serializable {
    private static final long serialVersionUID = 1L;  // version control

    private String name;
    private int age;
    private double salary;
    private transient String password;  // excluded from serialization

    public Employee(String name, int age, double salary, String password) {
        this.name = name;
        this.age = age;
        this.salary = salary;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Employee{name='%s', age=%d, salary=%.2f, password='%s'}"
            .formatted(name, age, salary, password);
    }
}
```

### Serializing and Deserializing

```java
// ✅ Serialize an object to a file
Employee emp = new Employee("Alice", 30, 85000.0, "secret123");

try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("employee.ser"))) {
    oos.writeObject(emp);
}

// ✅ Deserialize an object from a file
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("employee.ser"))) {
    Employee restored = (Employee) ois.readObject();
    System.out.println(restored);
    // Employee{name='Alice', age=30, salary=85000.00, password='null'}
    // password is null because it's transient
}
```

---

## serialVersionUID

The `serialVersionUID` is a version identifier for serialized classes.

```java
// ✅ Always declare serialVersionUID explicitly
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}

// ❌ Anti-pattern: Not declaring serialVersionUID
public class User implements Serializable {
    // JVM generates one based on class structure
    // Any change to class fields/methods breaks deserialization!
}
```

**What happens on mismatch:**
```
java.io.InvalidClassException:
  local class incompatible: stream classdesc serialVersionUID = 1,
  local class serialVersionUID = 2
```

---

## The `transient` Keyword

Fields marked `transient` are excluded from serialization.

```java
public class Session implements Serializable {
    private static final long serialVersionUID = 1L;

    private String userId;
    private transient Socket connection;       // non-serializable — must be transient
    private transient String authToken;        // sensitive data — should be transient
    private transient Logger logger;           // utility objects — should be transient
    private transient int cachedHashCode;      // derived data — can be recomputed
}
```

---

## Custom Serialization (writeObject / readObject)

Override default serialization behavior with private `writeObject` and `readObject` methods.

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;

    private String accountId;
    private transient String encryptedPin;  // custom handling needed

    // ✅ Custom serialization: encrypt sensitive data
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();                      // serialize non-transient fields
        oos.writeObject(encrypt(encryptedPin));         // manually write transient field
    }

    // ✅ Custom deserialization: decrypt sensitive data
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();                       // deserialize non-transient fields
        this.encryptedPin = decrypt((String) ois.readObject());
    }

    // ✅ Validation after deserialization
    private void readResolve() {
        // called after readObject — validate invariants
        if (accountId == null || accountId.isBlank()) {
            throw new IllegalStateException("Invalid account ID");
        }
    }

    private String encrypt(String data) { /* ... */ return data; }
    private String decrypt(String data) { /* ... */ return data; }
}
```

---

## Serialization Proxy Pattern

The **serialization proxy pattern** (Effective Java, Item 90) is the most robust approach.

```java
public final class Period implements Serializable {
    private static final long serialVersionUID = 1L;

    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        if (start.after(end)) throw new IllegalArgumentException("start > end");
        this.start = new Date(start.getTime());  // defensive copy
        this.end = new Date(end.getTime());
    }

    // ✅ Serialization proxy — inner class represents serialized form
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 1L;
        private final Date start;
        private final Date end;

        SerializationProxy(Period period) {
            this.start = period.start;
            this.end = period.end;
        }

        // Reconstruct the outer object with full validation
        private Object readResolve() {
            return new Period(start, end);  // uses public constructor — validated!
        }
    }

    // Replace the outer object with the proxy during serialization
    private Object writeReplace() {
        return new SerializationProxy(this);
    }

    // Prevent direct deserialization of the outer class
    private void readObject(ObjectInputStream ois) throws InvalidObjectException {
        throw new InvalidObjectException("Proxy required");
    }
}
```

---

## Security Concerns: Deserialization Attacks

Deserialization of untrusted data is inherently dangerous.

```java
// ❌ DANGEROUS: Deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(untrustedSocket.getInputStream());
Object obj = ois.readObject();  // can execute arbitrary code!

// ✅ Use ObjectInputFilter (Java 9+) to restrict deserialization
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"));
ois.setObjectInputFilter(filterInfo -> {
    Class<?> clazz = filterInfo.serialClass();
    if (clazz != null) {
        // Only allow specific classes
        if (clazz == Employee.class || clazz == String.class) {
            return ObjectInputFilter.Status.ALLOWED;
        }
        return ObjectInputFilter.Status.REJECTED;
    }
    // Limit depth and array size
    if (filterInfo.depth() > 5) return ObjectInputFilter.Status.REJECTED;
    if (filterInfo.arrayLength() > 1000) return ObjectInputFilter.Status.REJECTED;
    return ObjectInputFilter.Status.UNDECIDED;
});

// ✅ Global filter (Java 17+)
ObjectInputFilter.Config.setSerialFilterFactory((current, next) ->
    ObjectInputFilter.Config.createFilter(
        "com.myapp.model.*;!*"  // allow only com.myapp.model classes
    )
);
```

---

## Alternatives to Java Serialization

```java
// ✅ JSON (Jackson)
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(employee);
Employee restored = mapper.readValue(json, Employee.class);

// ✅ JSON (Gson)
Gson gson = new Gson();
String json = gson.toJson(employee);
Employee restored = gson.fromJson(json, Employee.class);

// ✅ Protocol Buffers (Protobuf)
// Define in .proto file, generate Java classes
EmployeeProto.Employee proto = EmployeeProto.Employee.newBuilder()
    .setName("Alice")
    .setAge(30)
    .build();
byte[] bytes = proto.toByteArray();

// ✅ Java Records with JSON (Java 16+)
public record EmployeeDTO(String name, int age, double salary) {}
```

---

## Record Serialization (Java 16+)

Records have simplified serialization — no `writeObject`/`readObject` needed.

```java
// ✅ Records are serializable when they implement Serializable
public record Point(int x, int y) implements Serializable {}

// Serialization uses the canonical constructor for deserialization
// No custom readObject/writeObject needed
// Immune to many deserialization attacks

try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("point.ser"))) {
    oos.writeObject(new Point(10, 20));
}

try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("point.ser"))) {
    Point p = (Point) ois.readObject();
    System.out.println(p);  // Point[x=10, y=20]
}
```

---

## Serialization Comparison

| Feature | Java Serialization | JSON (Jackson) | Protobuf |
|---------|-------------------|----------------|----------|
| Format | Binary | Text (JSON) | Binary |
| Speed | Medium | Slower | Fastest |
| Size | Large | Medium | Smallest |
| Human-readable | No | Yes | No |
| Cross-language | No | Yes | Yes |
| Security | Risky | Safer | Safer |
| Schema evolution | `serialVersionUID` | Flexible | `.proto` schema |

---

## Key Takeaways

1. **`serialVersionUID`** must always be declared explicitly to control version compatibility
2. **`transient`** excludes fields from serialization (sensitive data, non-serializable objects)
3. **Serialization proxy pattern** is the safest approach for custom serialization
4. **Never deserialize untrusted data** — use `ObjectInputFilter` or avoid Java serialization entirely
5. **Prefer JSON or Protobuf** for modern applications — safer, cross-language, and human-readable
6. **Records** provide simplified, safer serialization out of the box (Java 16+)

---

[← Previous: Q3 - BufferedReader and BufferedWriter](./q03-buffered-reader-writer.md) | [Next: Q5 - Externalizable →](./q05-externalizable.md)
