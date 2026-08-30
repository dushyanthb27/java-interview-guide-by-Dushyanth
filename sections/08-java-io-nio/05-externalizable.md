# Q5: What Is Externalizable?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
What is the `Externalizable` interface? How does it differ from `Serializable`?

---

## Answer

`Externalizable` extends `Serializable` and gives you **full control** over the serialization process. Unlike `Serializable`, which automatically serializes all non-transient fields, `Externalizable` requires you to explicitly write and read every field.

---

## Interface Definition

```java
public interface Externalizable extends Serializable {
    void writeExternal(ObjectOutput out) throws IOException;
    void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
}
```

---

## Basic Example

```java
public class Product implements Externalizable {
    private static final long serialVersionUID = 1L;

    private String name;
    private double price;
    private int quantity;
    private transient String internalCode;  // transient is optional — you control everything

    // ✅ REQUIRED: Public no-arg constructor (Externalizable mandates this)
    public Product() {}

    public Product(String name, double price, int quantity, String internalCode) {
        this.name = name;
        this.price = price;
        this.quantity = quantity;
        this.internalCode = internalCode;
    }

    // ✅ Full control: write only what you need, in any order
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        out.writeDouble(price);
        out.writeInt(quantity);
        // internalCode intentionally not written
    }

    // ✅ Read in the SAME order as written
    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        this.name = in.readUTF();
        this.price = in.readDouble();
        this.quantity = in.readInt();
    }

    @Override
    public String toString() {
        return "Product{name='%s', price=%.2f, qty=%d, code='%s'}"
            .formatted(name, price, quantity, internalCode);
    }
}
```

### Usage

```java
// Serialize
Product product = new Product("Laptop", 999.99, 5, "INT-001");
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("product.ser"))) {
    oos.writeObject(product);
}

// Deserialize
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("product.ser"))) {
    Product restored = (Product) ois.readObject();
    System.out.println(restored);
    // Product{name='Laptop', price=999.99, qty=5, code='null'}
}
```

---

## Externalizable vs Serializable

| Feature | Serializable | Externalizable |
|---------|-------------|----------------|
| Control | Automatic (all non-transient fields) | Manual (you write every field) |
| No-arg constructor | Not required | **Required** (public) |
| `transient` keyword | Respected | Irrelevant (you control what's written) |
| Performance | Slower (reflection-based) | Faster (explicit read/write) |
| Complexity | Simple | More code |
| Default behavior | Serializes everything | Serializes nothing (empty methods) |
| Superclass fields | Automatic (if superclass is Serializable) | Manual (must handle explicitly) |
| `serialVersionUID` | Recommended | Recommended |

---

## Handling Inheritance

```java
public class BaseEntity implements Externalizable {
    private static final long serialVersionUID = 1L;

    private long id;
    private String createdBy;

    public BaseEntity() {}

    public BaseEntity(long id, String createdBy) {
        this.id = id;
        this.createdBy = createdBy;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeLong(id);
        out.writeUTF(createdBy);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        this.id = in.readLong();
        this.createdBy = in.readUTF();
    }
}

public class Order extends BaseEntity {
    private static final long serialVersionUID = 1L;

    private String orderNumber;
    private double total;

    public Order() { super(); }

    public Order(long id, String createdBy, String orderNumber, double total) {
        super(id, createdBy);
        this.orderNumber = orderNumber;
        this.total = total;
    }

    // ✅ Must call super to serialize parent fields
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);       // serialize parent fields first
        out.writeUTF(orderNumber);
        out.writeDouble(total);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        super.readExternal(in);         // deserialize parent fields first
        this.orderNumber = in.readUTF();
        this.total = in.readDouble();
    }
}
```

---

## Advanced: Version-Tolerant Externalizable

```java
public class Config implements Externalizable {
    private static final long serialVersionUID = 1L;
    private static final int CURRENT_VERSION = 2;

    private String hostname;
    private int port;
    private boolean sslEnabled;  // added in version 2

    public Config() {}

    // ✅ Write a version marker for forward/backward compatibility
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(CURRENT_VERSION);   // version marker
        out.writeUTF(hostname);
        out.writeInt(port);
        // Version 2 fields
        out.writeBoolean(sslEnabled);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        int version = in.readInt();      // read version marker
        this.hostname = in.readUTF();
        this.port = in.readInt();
        // Handle version differences gracefully
        if (version >= 2) {
            this.sslEnabled = in.readBoolean();
        } else {
            this.sslEnabled = false;     // default for older versions
        }
    }
}
```

---

## Performance Comparison

```java
public class SerializationBenchmark {

    // Serializable version
    static class UserS implements Serializable {
        private static final long serialVersionUID = 1L;
        String name;
        int age;
        double score;
        UserS(String n, int a, double s) { name = n; age = a; score = s; }
    }

    // Externalizable version
    static class UserE implements Externalizable {
        private static final long serialVersionUID = 1L;
        String name;
        int age;
        double score;
        public UserE() {}
        UserE(String n, int a, double s) { name = n; age = a; score = s; }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeUTF(name);
            out.writeInt(age);
            out.writeDouble(score);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException {
            name = in.readUTF();
            age = in.readInt();
            score = in.readDouble();
        }
    }

    public static void main(String[] args) throws Exception {
        int iterations = 100_000;

        // Benchmark Serializable
        long start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
                oos.writeObject(new UserS("Alice", 30, 95.5));
            }
        }
        System.out.printf("Serializable:   %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        // Benchmark Externalizable
        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
                oos.writeObject(new UserE("Alice", 30, 95.5));
            }
        }
        System.out.printf("Externalizable: %d ms%n",
            (System.nanoTime() - start) / 1_000_000);
    }
}
// Typical results: Externalizable is 20-40% faster
```

---

## Common Anti-Patterns

```java
// ❌ Missing public no-arg constructor
public class BadExtern implements Externalizable {
    private String data;
    // No public no-arg constructor — throws InvalidClassException!
    public BadExtern(String data) { this.data = data; }
    // ...
}

// ❌ Reading fields in wrong order
@Override
public void readExternal(ObjectInput in) throws IOException {
    this.price = in.readDouble();   // WRONG — was written as second field
    this.name = in.readUTF();       // WRONG — was written as first field
}

// ❌ Forgetting to serialize parent class fields
@Override
public void writeExternal(ObjectOutput out) throws IOException {
    // super.writeExternal(out);  — forgot this!
    out.writeUTF(childField);
}
```

---

## When to Use Externalizable

| Scenario | Recommendation |
|----------|---------------|
| Simple POJO serialization | Use `Serializable` |
| Performance-critical serialization | Consider `Externalizable` |
| Fine-grained control over what's serialized | Use `Externalizable` |
| Complex inheritance hierarchy | Use `Externalizable` with explicit parent calls |
| Version-tolerant serialization | Use `Externalizable` with version markers |
| Cross-language interoperability | Use **JSON** or **Protobuf** instead |
| Modern Java applications | Use **JSON** or **Records** — avoid both |

---

## Key Takeaways

1. **`Externalizable`** provides full manual control over serialization — you write every field explicitly
2. A **public no-arg constructor** is mandatory for `Externalizable` classes
3. **Read order must match write order** — there is no field-name mapping
4. **Faster than `Serializable`** because it avoids reflection overhead
5. **Handle inheritance manually** by calling `super.writeExternal()` / `super.readExternal()`
6. **Use version markers** for forward/backward compatibility
7. **Modern alternative**: prefer JSON, Protobuf, or Records over both `Serializable` and `Externalizable`

---

[← Previous: Q4 - Serialization](./q04-serialization.md) | [Next: Q6 - NIO Channels →](./q06-nio-channels.md)
