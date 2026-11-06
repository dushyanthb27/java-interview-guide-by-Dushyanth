# Q4: How Does Serialization Work?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
How does Java serialization work? What are the best practices, security concerns, and modern alternatives?

---

## Answer

**Serialization** is the process of converting an object's state into a byte stream so it can be stored in a file, sent over a network, or cached. **Deserialization** is the reverse process — reconstructing the live object graph from bytes. Java provides built-in binary serialization through the `java.io.Serializable` marker interface.

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

The `serialVersionUID` is a 64-bit hash version identifier for serialized classes.

```java
// ✅ Always declare serialVersionUID explicitly
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}

// ❌ Anti-pattern: Not declaring serialVersionUID
public class User implements Serializable {
    // JVM generates one based on class structure (SHA-1 over fields, methods, modifiers)
    // Any change to class structure (adding fields, renaming) breaks deserialization!
}
```

**What happens on mismatch:**
```
java.io.InvalidClassException:
  com.myapp.User; local class incompatible: 
  stream classdesc serialVersionUID = 7892834729384, 
  local class serialVersionUID = 1
```

---

## The `transient` Keyword

Fields marked `transient` are bypassed by default serialization.

```java
public class Session implements Serializable {
    private static final long serialVersionUID = 1L;

    private String userId;
    private transient Socket connection;       // non-serializable network state — must be transient
    private transient String authToken;        // sensitive credentials — should be transient
    private transient Logger logger;           // non-serializable utility objects — should be transient
    private transient int cachedHashCode;      // derived/cached state — can be recomputed on demand
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

    // ✅ Custom serialization: encrypt sensitive data before writing
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();                      // serialize non-transient fields
        oos.writeObject(encrypt(encryptedPin));         // manually write transient field
    }

    // ✅ Custom deserialization: decrypt sensitive data upon reading
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();                       // deserialize non-transient fields
        this.encryptedPin = decrypt((String) ois.readObject());
    }

    // ✅ Validation after deserialization
    private void readResolve() throws ObjectStreamException {
        // called after readObject — validate invariants
        if (accountId == null || accountId.isBlank()) {
            throw new InvalidObjectException("Invalid account ID");
        }
    }

    private String encrypt(String data) { /* ... */ return data; }
    private String decrypt(String data) { /* ... */ return data; }
}
```

---

## Serialization Proxy Pattern

The **serialization proxy pattern** (Effective Java, Item 90) is the gold standard for robust, secure class serialization.

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

        // Reconstruct the outer object via public constructor — fully validated!
        private Object readResolve() {
            return new Period(start, end);  
        }
    }

    // Replace outer object with proxy during serialization
    private Object writeReplace() {
        return new SerializationProxy(this);
    }

    // Prevent direct deserialization attackes on outer class
    private void readObject(ObjectInputStream ois) throws InvalidObjectException {
        throw new InvalidObjectException("Proxy required");
    }
}
```

---

## Security Concerns: Deserialization Attacks

Java native serialization is notoriously vulnerable to **Remote Code Execution (RCE)** gadgets when deserializing untrusted streams.

```java
// ❌ DANGEROUS: Deserializing untrusted stream without filtering
ObjectInputStream ois = new ObjectInputStream(untrustedSocket.getInputStream());
Object obj = ois.readObject();  // gadget chains can execute arbitrary shell commands!

// ✅ Use ObjectInputFilter (Java 9+) to restrict deserialization classes
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"));
ois.setObjectInputFilter(filterInfo -> {
    Class<?> clazz = filterInfo.serialClass();
    if (clazz != null) {
        // Whitelist allowed model classes
        if (clazz == Employee.class || clazz == String.class) {
            return ObjectInputFilter.Status.ALLOWED;
        }
        return ObjectInputFilter.Status.REJECTED;
    }
    // Set bounds against denial-of-service (DoS) depth and array bombs
    if (filterInfo.depth() > 5) return ObjectInputFilter.Status.REJECTED;
    if (filterInfo.arrayLength() > 1000) return ObjectInputFilter.Status.REJECTED;
    return ObjectInputFilter.Status.UNDECIDED;
});

// ✅ JVM-wide global filter factory (Java 17+)
ObjectInputFilter.Config.setSerialFilterFactory((current, next) ->
    ObjectInputFilter.Config.createFilter(
        "com.myapp.model.*;java.lang.*;java.util.*;!*"  // strict whitelist
    )
);
```

---

## Record Serialization (Java 16+)

Records have built-in, simplified serialization semantics that bypass traditional field-reflection pitfalls.

```java
// ✅ Records are serializable automatically when they implement Serializable
public record Point(int x, int y) implements Serializable {}

// Serialization uses the canonical constructor for reconstruction.
// No custom readObject/writeObject needed; immune to many gadget injection vectors.
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

## Alternatives to Java Serialization

```java
// ✅ JSON via Jackson
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(employee);
Employee restored = mapper.readValue(json, Employee.class);

// ✅ Google Protocol Buffers (Protobuf) for high-performance microservices
EmployeeProto.Employee proto = EmployeeProto.Employee.newBuilder()
    .setName("Alice")
    .setAge(30)
    .build();
byte[] bytes = proto.toByteArray();
```

---

## Serialization Comparison

| Feature | Java Serialization | JSON (Jackson / Gson) | Protobuf / gRPC |
|---------|-------------------|----------------------|-----------------|
| Format | Binary (custom header) | Text (JSON) | Binary (packed varint) |
| Speed | Moderate | Slower (reflection parsing) | Extremely Fast |
| Payload Size | Large | Medium | Minimal |
| Human-Readable | No | Yes | No |
| Cross-Language | No (Java-only) | Yes | Yes |
| Security | High Risk (RCE vectors) | Safe | Safe |
| Schema Evolution | `serialVersionUID` / fragile | Flexible (field ignore) | Explicit `.proto` numbering |

---

## ⚠️ Gotchas

1. **Remote Code Execution (RCE) Vulnerabilities**: Native Java deserialization accepts arbitrary object graphs. If untrusted data is read via `ObjectInputStream`, attackers can feed malicious gadget chains (found in common libraries like Apache Commons Collections) that execute shell commands during object reconstruction. Always use `ObjectInputFilter` or avoid native serialization.
2. **Missing or Implicit `serialVersionUID`**: Omitting `serialVersionUID` causes the JVM to generate one dynamically based on compiler metadata. Any minor change to compiler version, method signatures, or private helpers alters the hash, throwing `InvalidClassException` during rolling updates and breaking backward compatibility.
3. **Transient Fields and Constructor Bypass**: Deserialization does **not** invoke standard constructors (except for records or externalizable). Transient fields are quietly initialized to their default type values (`null`, `0`, `false`). If these fields are required for business logic invariants, business methods called post-deserialization can trigger `NullPointerException`.
4. **ClassLoader Inconsistencies (`ClassNotFoundException`)**: During `readObject()`, the runtime uses the context class loader to resolve class descriptors. If the stream references a class version or package path missing from the classpath or loaded by an isolated plugin classloader, deserialization fails catastrophically.
5. **Object Graph Cyclical References and Memory Bloat**: Native serialization tracks object identity across the stream graph using handles (`TC_REFERENCE`). Deeply nested or circular object graphs can cause massive memory consumption in `ObjectInputStream` buffers, rendering applications vulnerable to OutOfMemoryError and Denial of Service attacks.

---

[← Previous: Q3 - BufferedReader and BufferedWriter](./q03-buffered-reader-writer.md) | [Next: Q5 - Externalizable →](./q05-externalizable.md)
