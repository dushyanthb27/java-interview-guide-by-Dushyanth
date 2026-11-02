# Abstraction

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Abstraction

---

## Q9 🟢 What Is Abstraction and How Is It Achieved?

**Question** — What is abstraction in Java? Explain the different ways to achieve it, abstract methods, and when to use abstraction in API design.

### Answer

**Abstraction** is the process of hiding implementation details and exposing only the essential features. It lets you focus on **what** an object does rather than **how** it does it.

**Two Ways to Achieve Abstraction**

Java provides two mechanisms for abstraction — abstract classes and interfaces. Abstract classes allow partial abstraction (0-100%) with state and constructors, while interfaces traditionally provide full abstraction with no state or constructors (though Java 8+ added default and static methods).

| Mechanism | Abstraction Level | Can Have State | Can Have Constructors |
|-----------|-------------------|----------------|----------------------|
| Abstract Classes | Partial (0-100%) | ✅ Yes | ✅ Yes |
| Interfaces | Full (traditionally 100%) | ❌ Only constants | ❌ No |

**Abstraction with Abstract Classes**

Abstract classes cannot be instantiated directly. They can have abstract methods (no implementation) that subclasses must implement, and concrete methods that provide shared behavior.

```java
// Abstract class - cannot be instantiated directly
public abstract class Vehicle {
    protected String make;
    protected String model;
    protected int year;

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }

    // Abstract method - WHAT to do (no implementation)
    public abstract void startEngine();

    // Abstract method - subclass decides HOW
    public abstract double fuelEfficiency();

    // Concrete method - shared behavior
    public String getInfo() {
        return "%d %s %s".formatted(year, make, model);
    }

    // Template method pattern: defines the skeleton
    public final void performMaintenance() {
        checkOil();
        checkTires();
        specificMaintenance(); // Subclass hook
        System.out.println("Maintenance complete for " + getInfo());
    }

    private void checkOil() { System.out.println("Checking oil..."); }
    private void checkTires() { System.out.println("Checking tires..."); }
    protected abstract void specificMaintenance();
}

public class ElectricCar extends Vehicle {
    private int batteryCapacity;

    public ElectricCar(String make, String model, int year, int batteryCapacity) {
        super(make, model, year);
        this.batteryCapacity = batteryCapacity;
    }

    @Override
    public void startEngine() {
        System.out.println("Electric motor silently starts");
    }

    @Override
    public double fuelEfficiency() {
        return batteryCapacity * 3.5; // miles per full charge
    }

    @Override
    protected void specificMaintenance() {
        System.out.println("Checking battery health...");
    }
}
```

**Abstraction with Interfaces**

Interfaces define a contract (pure abstraction) — what a class can do, not how it does it. They enable multiple inheritance of type without the diamond problem.

```java
// Interface defines the contract (pure abstraction)
public interface DataRepository<T> {
    T findById(String id);
    List<T> findAll();
    T save(T entity);
    void delete(String id);
    boolean exists(String id);
}

// Implementation A - hides database details
public class PostgresRepository<T> implements DataRepository<T> {
    private final DataSource dataSource;

    public PostgresRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public T findById(String id) {
        // SQL-specific implementation hidden from caller
        return executeQuery("SELECT * FROM table WHERE id = ?", id);
    }
}

// Implementation B - completely different storage mechanism
public class InMemoryRepository<T> implements DataRepository<T> {
    private final Map<String, T> store = new ConcurrentHashMap<>();

    @Override
    public T findById(String id) {
        return store.get(id); // No SQL, no database
    }
}
```

**Levels of Abstraction**

Good API design provides multiple levels of abstraction — high-level interfaces, partial abstract classes with common behavior, and concrete implementations with full details.

```java
// Level 1: High-level abstraction (interface)
public interface MessageSender {
    void send(String message, String recipient);
}

// Level 2: Partial abstraction (abstract class adds common behavior)
public abstract class AbstractMessageSender implements MessageSender {
    protected final Logger logger = LoggerFactory.getLogger(getClass());

    @Override
    public final void send(String message, String recipient) {
        validate(message, recipient);
        logger.info("Sending message to {}", recipient);
        doSend(message, recipient);
        logger.info("Message sent successfully");
    }

    private void validate(String message, String recipient) {
        if (message == null || message.isBlank()) {
            throw new IllegalArgumentException("Message cannot be empty");
        }
        if (recipient == null || recipient.isBlank()) {
            throw new IllegalArgumentException("Recipient cannot be empty");
        }
    }

    // Subclass provides the actual sending logic
    protected abstract void doSend(String message, String recipient);
}

// Level 3: Concrete implementation (full detail)
public class EmailSender extends AbstractMessageSender {
    @Override
    protected void doSend(String message, String recipient) {
        // SMTP-specific implementation
        System.out.println("Sending email to " + recipient);
    }
}
```

**Abstraction in API Design**

Good API design abstracts away complexity — HTTP calls, authentication, retry logic, serialization — so clients only see the contract.

```java
// Good API design: abstract away complexity
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
    PaymentResult refund(String transactionId, Money amount);
    PaymentStatus checkStatus(String transactionId);
}

// The client code doesn't know or care about:
// - HTTP calls to payment provider
// - Authentication token management
// - Retry logic and circuit breakers
// - JSON serialization/deserialization
// - Error mapping and translation

public class PaymentService {
    private final PaymentGateway gateway; // Depends on abstraction

    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway; // Injected - easily swappable
    }

    public PaymentResult processOrder(Order order) {
        return gateway.charge(order.getTotal(), order.getPaymentMethod());
    }
}
```

> **⚠️ Common Mistake** — Creating an abstract class with no abstract methods. This is usually a design smell — either make it a concrete class or an interface.

```java
// Abstract class with no abstract methods (usually a design smell)
public abstract class Config {
    public String getHost() { return "localhost"; }
    public int getPort() { return 8080; }
    // No abstract methods - why is this abstract?
}

// Better: use a concrete class or an interface
public class Config {
    public String getHost() { return "localhost"; }
    public int getPort() { return 8080; }
}

// Instantiating abstract classes directly
// Vehicle v = new Vehicle("Toyota", "Camry", 2024); // COMPILE ERROR
```

> **🔧 Performance Tip** — Abstraction has zero runtime cost when using abstract classes. Interface dispatch (via `invokeinterface`) can be marginally slower than virtual dispatch (via `invokevirtual`) but the JIT compiler optimizes both effectively.

> **🎯 Interviewer Insight** — Interviewers look for understanding of abstraction as a **design concept**, not just syntax. Show you can explain WHY to abstract, not just HOW. Demonstrate awareness of abstraction levels and real-world API design.

> **⚡ Follow-up** — Can an abstract class have a constructor? Why?

> **⚡ Follow-up** — What happens if a subclass doesn't implement all abstract methods?

> **⚡ Follow-up** — How does abstraction support the Dependency Inversion Principle?

> **📚 Related** — [Interfaces vs Abstract Classes](./07-interfaces-vs-abstract-classes.md), [Encapsulation](./05-encapsulation.md), [Composition vs Inheritance](./12-object-composition.md)

---

[← Previous: Encapsulation](./05-encapsulation.md) · [Next: Interfaces vs Abstract Classes →](./07-interfaces-vs-abstract-classes.md)
