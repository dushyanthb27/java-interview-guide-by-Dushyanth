# Q6: What Is Abstraction and How Is It Achieved?

## Question
What is abstraction in Java? Explain the different ways to achieve it, abstract methods, and when to use abstraction in API design.

## Difficulty: 🟢 Basic

---

## Answer

**Abstraction** is the process of hiding implementation details and exposing only the essential features. It lets you focus on **what** an object does rather than **how** it does it.

### Two Ways to Achieve Abstraction

| Mechanism | Abstraction Level | Can Have State | Can Have Constructors |
|-----------|-------------------|----------------|----------------------|
| Abstract Classes | Partial (0-100%) | ✅ Yes | ✅ Yes |
| Interfaces | Full (traditionally 100%) | ❌ Only constants | ❌ No |

### Abstraction with Abstract Classes

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

    // ✅ Abstract method - WHAT to do (no implementation)
    public abstract void startEngine();

    // ✅ Abstract method - subclass decides HOW
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

public class GasCar extends Vehicle {
    private double engineSize;

    public GasCar(String make, String model, int year, double engineSize) {
        super(make, model, year);
        this.engineSize = engineSize;
    }

    @Override
    public void startEngine() {
        System.out.println("Gas engine roars to life");
    }

    @Override
    public double fuelEfficiency() {
        return 50.0 / engineSize; // simplified MPG calculation
    }

    @Override
    protected void specificMaintenance() {
        System.out.println("Changing spark plugs...");
    }
}
```

### Abstraction with Interfaces

```java
// ✅ Interface defines the contract (pure abstraction)
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
    // ... other methods
}

// Implementation B - completely different storage mechanism
public class InMemoryRepository<T> implements DataRepository<T> {
    private final Map<String, T> store = new ConcurrentHashMap<>();

    @Override
    public T findById(String id) {
        return store.get(id); // No SQL, no database
    }
    // ... other methods
}
```

### Levels of Abstraction

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

public class SmsSender extends AbstractMessageSender {
    @Override
    protected void doSend(String message, String recipient) {
        // SMS gateway-specific implementation
        System.out.println("Sending SMS to " + recipient);
    }
}
```

### Real-World Analogy

```
   ABSTRACTION                    IMPLEMENTATION
┌──────────────────┐         ┌──────────────────────┐
│  Steering Wheel  │ ───▶    │  Rack and pinion     │
│  (what you see)  │         │  Power steering pump  │
│                  │         │  Hydraulic fluid       │
│  Gas Pedal       │ ───▶    │  Fuel injection       │
│  (what you use)  │         │  Throttle body         │
│                  │         │  Engine control unit   │
└──────────────────┘         └──────────────────────┘
   Driver's view                Engineer's view
```

### Abstraction in API Design

```java
// ✅ Good API design: abstract away complexity
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

### Common Mistake

```java
// ❌ Abstract class with no abstract methods (usually a design smell)
public abstract class Config {
    public String getHost() { return "localhost"; }
    public int getPort() { return 8080; }
    // No abstract methods - why is this abstract?
}

// ✅ Better: use a concrete class or an interface
public class Config {
    public String getHost() { return "localhost"; }
    public int getPort() { return 8080; }
}

// ❌ Instantiating abstract classes directly
// Vehicle v = new Vehicle("Toyota", "Camry", 2024); // COMPILE ERROR
```

### Performance Tip
Abstraction has zero runtime cost when using abstract classes. Interface dispatch (via `invokeinterface`) can be marginally slower than virtual dispatch (via `invokevirtual`) but the JIT compiler optimizes both effectively.

---

## Interviewer Insight
Interviewers look for understanding of abstraction as a **design concept**, not just syntax. Show you can explain WHY to abstract, not just HOW. Demonstrate awareness of abstraction levels and real-world API design.

## Follow-up Questions
1. Can an abstract class have a constructor? Why?
2. What happens if a subclass doesn't implement all abstract methods?
3. How does abstraction support the Dependency Inversion Principle?

## Related Topics
- [Interfaces vs Abstract Classes](./q07-interfaces-vs-abstract-classes.md)
- [Encapsulation](./q05-encapsulation.md)
- [Composition vs Inheritance](./q12-object-composition.md)
