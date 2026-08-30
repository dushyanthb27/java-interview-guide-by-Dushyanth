# Q7: Project Lombok Usage 🟢

[Home](../../README.md) > [Framework Integration](./README.md) > Lombok

---

## Overview

**Project Lombok** is a Java library that uses annotation processing to generate boilerplate code at compile time — getters, setters, constructors, builders, and more. It dramatically reduces verbosity while keeping your bytecode identical to hand-written code.

---

## How Lombok Works

```
┌──────────────┐    javac     ┌────────────────────┐    compile    ┌───────────┐
│  Your Code   │────────────►│ Annotation Processor │─────────────►│  .class   │
│  (with @Data)│             │   (Lombok plugs in) │              │ (with all │
│              │             │   Modifies the AST   │              │  methods) │
└──────────────┘             └────────────────────┘              └───────────┘
```

Lombok hooks into the **Java compiler's annotation processing** phase. It modifies the Abstract Syntax Tree (AST) before bytecode generation — the generated methods exist in `.class` files but not in your source code.

---

## Core Annotations

### @Getter / @Setter

```java
// ❌ Without Lombok — 20+ lines of boilerplate
public class User {
    private String name;
    private String email;
    private int age;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}

// ✅ With Lombok — clean and concise
@Getter
@Setter
public class User {
    private String name;
    private String email;
    private int age;
}

// ✅ Field-level control
public class Account {
    @Getter
    private final String id;

    @Getter @Setter
    private String name;

    @Setter(AccessLevel.PROTECTED)   // restrict setter visibility
    private String internalCode;
}
```

---

### @ToString

```java
@ToString
public class Employee {
    private String name;
    private String department;

    @ToString.Exclude
    private String ssn;            // sensitive — excluded from toString
}

// Output: Employee(name=Alice, department=Engineering)

@ToString(callSuper = true)        // include parent fields
public class Manager extends Employee {
    private int teamSize;
}
```

---

### @EqualsAndHashCode

```java
@EqualsAndHashCode
public class Product {
    private String sku;
    private String name;

    @EqualsAndHashCode.Exclude
    private double price;          // not part of identity
}

// ✅ For JPA entities, use business key only
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @EqualsAndHashCode.Include
    private String orderNumber;    // natural business key

    private BigDecimal total;
}
```

---

### @NoArgsConstructor / @AllArgsConstructor / @RequiredArgsConstructor

```java
@NoArgsConstructor                   // public User() { }
@AllArgsConstructor                  // public User(String name, String email, int age) { }
public class User {
    private String name;
    private String email;
    private int age;
}

// ✅ @RequiredArgsConstructor — constructor for final fields only
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;   // included
    private final NotificationService notifier; // included
    private int retryCount;                     // excluded (not final)
}
// Generates: OrderService(OrderRepository repository, NotificationService notifier)
```

```java
// ✅ Perfect for Spring constructor injection
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    // No need for @Autowired — Spring uses the single generated constructor
}
```

---

### @Data

```java
// ✅ @Data = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@Data
public class UserDto {
    private final String id;        // final → in RequiredArgs constructor
    private String name;
    private String email;
}
```

```java
// ❌ Anti-pattern: @Data on JPA entities
@Data
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    // Problem 1: equals/hashCode uses all fields including generated ID
    // Problem 2: toString may trigger lazy loading
    // Problem 3: setters on @Id field
}

// ✅ Best practice: manual control for entities
@Entity
@Getter @Setter
@ToString(exclude = "orders")
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @Id @GeneratedValue
    private Long id;

    @EqualsAndHashCode.Include
    private String email;

    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

---

### @Builder

```java
@Builder
public class Email {
    private String from;
    private String to;
    private String subject;
    private String body;
    @Builder.Default
    private boolean html = false;
    @Singular
    private List<String> attachments;
}

// ✅ Fluent, readable object construction
Email email = Email.builder()
    .from("noreply@app.com")
    .to("alice@example.com")
    .subject("Welcome!")
    .body("<h1>Hello</h1>")
    .html(true)
    .attachment("welcome.pdf")
    .attachment("terms.pdf")
    .build();
```

```java
// ✅ Combine with @AllArgsConstructor for flexibility
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Data
public class CreateUserRequest {
    private String name;
    private String email;
    private String role;
}
```

---

### @Slf4j — Logging

```java
// ❌ Without Lombok
public class PaymentService {
    private static final Logger log =
        LoggerFactory.getLogger(PaymentService.class);
}

// ✅ With Lombok
@Slf4j
@Service
public class PaymentService {
    public void processPayment(BigDecimal amount) {
        log.info("Processing payment of {}", amount);
    }
}

// Other logging annotations:
// @Log          → java.util.logging.Logger
// @Log4j2       → org.apache.logging.log4j.Logger
// @CommonsLog   → org.apache.commons.logging.Log
```

---

### @Value — Immutable Classes

```java
// ✅ @Value = immutable version of @Data
//    All fields private final, no setters, class is final
@Value
public class Money {
    BigDecimal amount;
    String currency;
}

// Equivalent to:
// @Getter @ToString @EqualsAndHashCode
// @AllArgsConstructor @FieldDefaults(makeFinal=true, level=AccessLevel.PRIVATE)
// public final class Money { ... }
```

---

## Delombok

```bash
# ✅ Delombok generates the source code Lombok would produce
java -jar lombok.jar delombok src/main/java -d delomboked

# Useful for:
# - Understanding what Lombok generates
# - Source code analysis tools that don't support Lombok
# - Debugging annotation processing issues
```

---

## IDE Support

| IDE | Support | Setup |
|-----|---------|-------|
| IntelliJ IDEA | ✅ Built-in (2020.3+) | Enable annotation processing |
| Eclipse | ✅ Plugin required | Run `lombok.jar` installer |
| VS Code | ✅ Extension | Install Java Extension Pack |
| NetBeans | ✅ Plugin | Install via plugin manager |

**IntelliJ Setup:**
```
Settings → Build → Compiler → Annotation Processors → Enable
```

---

## Pros and Cons

| Pros ✅ | Cons ❌ |
|---------|---------|
| Eliminates boilerplate | Magic — code not visible in source |
| Cleaner, more readable classes | IDE plugin dependency |
| Less merge conflicts | Debugging can be confusing |
| Consistent code generation | Tight coupling to Lombok |
| Reduces human error | New team members need onboarding |
| Active maintenance | Relies on internal compiler APIs |

---

## Anti-Patterns vs Best Practices

```java
// ❌ Anti-pattern: @Data on everything
@Data   // generates setters → mutable where immutability is desired
public class Config { ... }

// ✅ Best practice: choose the right annotation
@Value  // immutable DTO
public class Config { ... }

// ❌ Anti-pattern: @Builder without @NoArgsConstructor for frameworks
@Builder
public class UserDto { ... }  // Jackson/JPA can't instantiate

// ✅ Best practice: combine annotations for framework compatibility
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDto { ... }

// ❌ Anti-pattern: @EqualsAndHashCode with JPA generated IDs
// ✅ Best practice: use business keys with onlyExplicitlyIncluded
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| How it works | Annotation processing at compile time — modifies the AST |
| @Data vs @Value | @Data = mutable; @Value = immutable (final fields, no setters) |
| @Data on entities | Bad — use @Getter/@Setter with explicit @EqualsAndHashCode |
| @RequiredArgsConstructor | Perfect for Spring DI — generates constructor for final fields |
| Delombok | Tool to see generated code — useful for debugging |

---

[← Previous: Q6 - Maven & Gradle](./q06-maven-gradle.md) | [Next: Q8 - SLF4J Logging →](./q08-logging-slf4j.md)
