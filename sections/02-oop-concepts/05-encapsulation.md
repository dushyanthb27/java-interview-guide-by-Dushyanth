# Q5: What Is Encapsulation and Why Is It Important?

## Question
What is encapsulation in Java? Explain how it is achieved, the role of access modifiers, and how it differs from abstraction.

## Difficulty: 🟢 Basic

---

## Answer

**Encapsulation** is the bundling of data (fields) and the methods that operate on that data into a single unit (class), while **restricting direct access** to the internal state. It is often called "data hiding."

### Core Principles

1. **Declare fields as `private`** - hide internal state
2. **Provide public getters/setters** - controlled access
3. **Add validation** in setters - enforce invariants

### Basic Encapsulation

```java
public class BankAccount {
    // ✅ Private fields - hidden from outside
    private String accountNumber;
    private double balance;
    private boolean active;

    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        setBalance(initialBalance); // Use setter for validation
        this.active = true;
    }

    // ✅ Getter - controlled read access
    public String getAccountNumber() {
        return accountNumber;
    }

    public double getBalance() {
        return balance;
    }

    public boolean isActive() { // boolean getter uses 'is' prefix
        return active;
    }

    // ✅ Setter with validation - controlled write access
    private void setBalance(double balance) {
        if (balance < 0) {
            throw new IllegalArgumentException("Balance cannot be negative");
        }
        this.balance = balance;
    }

    // ✅ Business methods that enforce rules
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit must be positive");
        if (!active) throw new IllegalStateException("Account is inactive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Withdrawal must be positive");
        if (amount > balance) throw new IllegalArgumentException("Insufficient funds");
        if (!active) throw new IllegalStateException("Account is inactive");
        this.balance -= amount;
    }
}
```

### Access Modifiers Role

| Modifier | Class | Package | Subclass | World |
|----------|-------|---------|----------|-------|
| `private` | ✅ | ❌ | ❌ | ❌ |
| default (package-private) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
public class Employee {
    private String ssn;           // Only this class
    String department;             // Same package only
    protected double salary;       // Same package + subclasses
    public String name;            // Everyone (avoid for fields!)
}
```

### Benefits of Encapsulation

```java
// ✅ 1. Controlled Access - prevent invalid state
public class Temperature {
    private double celsius;

    public void setCelsius(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException("Below absolute zero");
        }
        this.celsius = celsius;
    }

    // ✅ 2. Flexibility - change implementation without breaking clients
    public double getFahrenheit() {
        return celsius * 9.0 / 5.0 + 32; // Derived from stored value
    }

    // ✅ 3. Read-only or write-only access
    public double getCelsius() {
        return celsius; // Read-only: no public setter for direct celsius setting
    }
}
```

### Encapsulation vs Abstraction

| Aspect | Encapsulation | Abstraction |
|--------|--------------|-------------|
| Focus | **How** to hide data | **What** to expose |
| Goal | Protect internal state | Simplify complexity |
| Mechanism | Access modifiers, getters/setters | Interfaces, abstract classes |
| Level | Implementation level | Design level |

```java
// Abstraction: User only sees the interface (what)
public interface PaymentProcessor {
    boolean processPayment(double amount);
}

// Encapsulation: Implementation hides details (how)
public class StripePaymentProcessor implements PaymentProcessor {
    private final String apiKey;         // Hidden
    private final HttpClient client;     // Hidden

    public StripePaymentProcessor(String apiKey) {
        this.apiKey = apiKey;
        this.client = HttpClient.newHttpClient();
    }

    @Override
    public boolean processPayment(double amount) {
        // Internal details are encapsulated
        return sendToStripeApi(amount);
    }

    private boolean sendToStripeApi(double amount) {
        // Hidden implementation detail
        return true;
    }
}
```

### JavaBeans Convention

```java
// ✅ Standard JavaBean pattern
public class UserBean {
    private String firstName;
    private String lastName;
    private int age;
    private boolean active;

    // No-arg constructor
    public UserBean() { }

    // Getters follow getXxx() naming
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }

    // Boolean getter uses isXxx()
    public boolean isActive() { return active; }

    // Setters follow setXxx() naming
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    public void setAge(int age) { this.age = age; }
    public void setActive(boolean active) { this.active = active; }
}
```

### Records and Encapsulation (Java 16+)

Records provide **immutable encapsulation** by default:

```java
// ✅ Immutable, automatically encapsulated
public record Account(String id, String owner, double balance) {
    // Compact constructor for validation
    public Account {
        if (balance < 0) throw new IllegalArgumentException("Negative balance");
        if (id == null || id.isBlank()) throw new IllegalArgumentException("ID required");
    }

    // Accessor methods auto-generated: id(), owner(), balance()
    // No setters - records are immutable
}
```

```java
var account = new Account("A001", "Alice", 1000.0);
System.out.println(account.owner());   // "Alice" - accessor method (not getOwner)
// account.balance = 500;  // ❌ COMPILE ERROR - fields are final
```

### Defensive Copying for Mutable Fields

```java
public class Course {
    private final String name;
    private final List<String> students;

    public Course(String name, List<String> students) {
        this.name = name;
        // ✅ Defensive copy on input
        this.students = new ArrayList<>(students);
    }

    public List<String> getStudents() {
        // ✅ Defensive copy on output (or use unmodifiable)
        return Collections.unmodifiableList(students);
    }
}
```

### Common Mistake

```java
// ❌ Exposing mutable internal state
public class Team {
    private List<String> members = new ArrayList<>();

    public List<String> getMembers() {
        return members; // Caller can modify the internal list!
    }
}

// External code can break encapsulation:
team.getMembers().clear(); // Oops! Internal state corrupted

// ✅ Fix: return an unmodifiable view or a copy
public List<String> getMembers() {
    return List.copyOf(members);
}
```

### Performance Tip
The JIT compiler inlines trivial getters and setters, so there is no runtime cost compared to direct field access. Always use encapsulation without worrying about performance impact.

---

## Interviewer Insight
Interviewers check that you understand why fields should be private, how validation in setters maintains invariants, and the difference between encapsulation and abstraction. Mentioning defensive copying shows advanced understanding.

## Follow-up Questions
1. How do Java records handle encapsulation differently from traditional classes?
2. What is the Principle of Least Privilege in the context of access modifiers?
3. How does encapsulation support the Open/Closed Principle?

## Related Topics
- [Abstraction](./q06-abstraction.md)
- [Classes and Objects](./q01-classes-and-objects.md)
- [This Keyword](./q11-this-keyword.md)
