# Q5: What Is Externalizable?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
What is the `Externalizable` interface? How does it differ from `Serializable`, and when should you choose one over the other?

---

## Answer

`Externalizable` extends `Serializable` and hands over **complete, manual control** over the serialization and deserialization process to the developer. Unlike `Serializable`, which uses reflection to automatically serialize all non-transient fields, `Externalizable` requires explicit invocation of write and read methods for every piece of state.

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
    private transient String internalCode;  // transient is ignored; you decide what is written

    // ✅ REQUIRED: Public no-arg constructor (Externalizable mandates this)
    public Product() {}

    public Product(String name, double price, int quantity, String internalCode) {
        this.name = name;
        this.price = price;
        this.quantity = quantity;
        this.internalCode = internalCode;
    }

    // ✅ Full control: write only specified fields in any custom sequence
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        out.writeDouble(price);
        out.writeInt(quantity);
        // internalCode intentionally omitted
    }

    // ✅ Read fields back in the EXACT same order they were written
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
| Control | Automatic (reflection over non-transient fields) | Manual (explicit `writeExternal`/`readExternal`) |
| No-arg Constructor | Not required | **Strictly Required** (must be `public`) |
| `transient` Keyword | Respected (excluded from stream) | Irrelevant (you dictate stream content) |
| Performance | Slower (runtime metadata inspection) | Faster (direct binary stream operations) |
| Boilerplate | Minimal | High (requires manual stream handling) |
| Default Behavior | Serializes all non-transient fields | Serializes nothing (empty default methods) |
| Superclass Fields | Automatic (if superclass implements `Serializable`) | Manual (must explicitly chain `super.writeExternal()`) |
| Schema Versioning | `serialVersionUID` | Managed entirely by custom read logic |

---

## Handling Inheritance

When a class extends another `Externalizable` class, parent state must be explicitly delegated.

```java
public class BaseEntity implements Externalizable {
    private static final long serialVersionUID = 1L;

    private long id;
    private String createdBy;

    public BaseEntity() {}  // Required public no-arg constructor

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

    // ✅ Must invoke super to persist parent fields correctly
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);       
        out.writeUTF(orderNumber);
        out.writeDouble(total);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        super.readExternal(in);         
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
    private boolean sslEnabled;  // field introduced in version 2

    public Config() {}

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(CURRENT_VERSION);   // Write stream version prefix
        out.writeUTF(hostname);
        out.writeInt(port);
        out.writeBoolean(sslEnabled);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        int version = in.readInt();      // Inspect stream version
        this.hostname = in.readUTF();
        this.port = in.readInt();
        
        if (version >= 2) {
            this.sslEnabled = in.readBoolean();
        } else {
            this.sslEnabled = false;     // Safe fallback for legacy version 1 payloads
        }
    }
}
```

---

## ⚠️ Gotchas

1. **Mandatory Public No-Arg Constructor**: Deserialization of an `Externalizable` object always instantiates the target class via its public zero-argument constructor *before* calling `readExternal()`. If the class lacks a public no-arg constructor, `ObjectInputStream` throws an `InvalidClassException` or `InstantiationException` at runtime.
2. **Order Mismatch Corruption**: Because `Externalizable` bypasses metadata field mapping, fields are written and read sequentially positionally. If `writeExternal` writes `String` then `int`, but `readExternal` reads `int` then `String`, the stream corrupts silently or throws `StreamCorruptedException` / `ClassCastException`.
3. **Forgotten Superclass State Delegation**: If a subclass implements `Externalizable` but extends a base class (whether `Serializable` or `Externalizable`), superclass fields are **not** automatically persisted. Unless `super.writeExternal()` and `super.readExternal()` are explicitly invoked, parent fields deserialize as `null` or default primitive values.
4. **Transient Modifier Irrelevance**: The `transient` keyword has zero effect on `Externalizable` classes because reflection is bypassed entirely. Developers must manually decide which fields to omit inside `writeExternal`, otherwise marking a field transient while writing it anyway results in confusion.
5. **Security and Object Instantiation Pitfalls**: Because `Externalizable` objects are instantiated via their public no-arg constructor during deserialization, malicious actors can invoke deserialization on classes with side-effecting constructors before any validation check in `readExternal` executes.

---

[← Previous: Q4 - Serialization](./q04-serialization.md) | [Next: Q6 - NIO Channels →](./q06-nio-channels.md)
