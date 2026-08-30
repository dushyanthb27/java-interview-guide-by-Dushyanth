# Q2: Explain Dependency Injection in Depth 🟠

[Home](../../README.md) > [Framework Integration](./README.md) > Dependency Injection

---

## Overview

**Dependency Injection (DI)** is a design pattern where an object's dependencies are provided externally rather than created internally. It is the primary mechanism Spring uses to achieve Inversion of Control.

---

## The Three Types of Injection

### 1. Constructor Injection (Preferred)

```java
// ✅ Best practice — immutable, testable, clear dependencies
@Service
public class OrderService {

    private final OrderRepository repository;
    private final NotificationService notifier;

    // @Autowired is optional with a single constructor (Spring 4.3+)
    public OrderService(OrderRepository repository,
                        NotificationService notifier) {
        this.repository = repository;
        this.notifier = notifier;
    }
}
```

**Why constructor injection is preferred:**
- Fields can be `final` → immutable, thread-safe
- Dependencies are explicit — you see them in the constructor signature
- Impossible to create an object in an invalid state
- Easy to unit test without any Spring context

### 2. Setter Injection

```java
// 🟡 Acceptable for optional dependencies
@Service
public class ReportService {

    private ReportFormatter formatter;

    @Autowired(required = false)
    public void setFormatter(ReportFormatter formatter) {
        this.formatter = formatter;
    }
}
```

### 3. Field Injection

```java
// ❌ Anti-pattern — avoid in production code
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;   // not final, hidden dependency

    @Autowired
    private EmailService emailService;       // hard to test without Spring
}
```

**Why field injection is bad:**
- Cannot mark fields `final`
- Hides dependencies — class API doesn't reveal them
- Requires reflection to set in tests → tightly coupled to Spring
- Makes circular dependencies harder to detect

---

## Injection Comparison

```
┌───────────────────┬──────────────┬──────────────┬──────────────┐
│     Aspect        │ Constructor  │    Setter    │    Field     │
├───────────────────┼──────────────┼──────────────┼──────────────┤
│ Immutability      │ ✅ final     │ ❌ mutable   │ ❌ mutable   │
│ Required deps     │ ✅ enforced  │ ❌ optional   │ ❌ optional   │
│ Testability       │ ✅ easy      │ 🟡 moderate  │ ❌ hard      │
│ Circular detect   │ ✅ fails fast│ ❌ silent     │ ❌ silent     │
│ Readability       │ ✅ clear     │ 🟡 scattered │ ❌ hidden    │
└───────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## @Autowired vs @Inject vs @Resource

```java
// Spring's @Autowired — by type
@Autowired
private UserRepository repo;

// JSR-330 @Inject — by type (standard Java, not Spring-specific)
@Inject
private UserRepository repo;

// JSR-250 @Resource — by name
@Resource(name = "jdbcUserRepository")
private UserRepository repo;
```

| Annotation | Source | Resolution | Required |
|------------|--------|------------|----------|
| `@Autowired` | Spring | By type | Yes (configurable) |
| `@Inject` | JSR-330 | By type | Always required |
| `@Resource` | JSR-250 | By name | Yes |

---

## @Qualifier — Resolving Ambiguity

```java
public interface PaymentGateway {
    void charge(BigDecimal amount);
}

@Component("stripe")
public class StripeGateway implements PaymentGateway { ... }

@Component("paypal")
public class PayPalGateway implements PaymentGateway { ... }

// ✅ Use @Qualifier to pick the right implementation
@Service
public class CheckoutService {

    private final PaymentGateway gateway;

    public CheckoutService(@Qualifier("stripe") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

```java
// ✅ Alternative: custom qualifier annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PrimaryGateway { }

@Component
@PrimaryGateway
public class StripeGateway implements PaymentGateway { ... }

@Service
public class CheckoutService {
    public CheckoutService(@PrimaryGateway PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

---

## Bean Scopes

```java
@Component
@Scope("prototype")
public class ShoppingCart { ... }
```

| Scope | Description | Default? |
|-------|-------------|----------|
| `singleton` | One instance per IoC container | ✅ Yes |
| `prototype` | New instance every time requested | |
| `request` | One instance per HTTP request (web) | |
| `session` | One instance per HTTP session (web) | |
| `application` | One instance per ServletContext (web) | |

```java
// ❌ Anti-pattern: injecting prototype into singleton
@Component  // singleton by default
public class OrderProcessor {
    @Autowired
    private ShoppingCart cart;  // prototype — but SAME instance always!
}

// ✅ Fix: use ObjectFactory or Provider
@Component
public class OrderProcessor {
    private final ObjectFactory<ShoppingCart> cartFactory;

    public OrderProcessor(ObjectFactory<ShoppingCart> cartFactory) {
        this.cartFactory = cartFactory;
    }

    public void process() {
        ShoppingCart cart = cartFactory.getObject();  // fresh instance
    }
}
```

---

## Circular Dependencies

```java
// ❌ Circular dependency — fails with constructor injection
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { }   // needs B
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { }   // needs A → deadlock!
}
```

**Solutions:**

```java
// ✅ Solution 1: Redesign — extract shared logic
@Service
public class SharedLogic { ... }

@Service
public class ServiceA {
    public ServiceA(SharedLogic shared) { }
}

@Service
public class ServiceB {
    public ServiceB(SharedLogic shared) { }
}

// 🟡 Solution 2: @Lazy (last resort)
@Service
public class ServiceA {
    public ServiceA(@Lazy ServiceB b) { }
}
```

---

## DI Without Spring (Manual / Pure Java)

```java
// ✅ DI is a pattern, not a framework feature
public class Application {

    public static void main(String[] args) {
        // Manual wiring — the "poor man's DI"
        UserRepository repo = new JdbcUserRepository();
        EmailService email = new SmtpEmailService();
        UserService service = new UserServiceImpl(repo, email);

        service.registerUser("alice@example.com");
    }
}
```

This proves DI is about **design**, not about Spring. Spring just automates the wiring.

---

## Benefits of DI for Testing

```java
// ✅ Constructor injection makes unit testing trivial
class OrderServiceTest {

    @Test
    void shouldPlaceOrder() {
        // Arrange — inject test doubles manually, no Spring needed
        OrderRepository mockRepo = mock(OrderRepository.class);
        NotificationService mockNotifier = mock(NotificationService.class);
        OrderService service = new OrderService(mockRepo, mockNotifier);

        // Act
        service.placeOrder(new Order("item-1", 2));

        // Assert
        verify(mockRepo).save(any(Order.class));
        verify(mockNotifier).send(anyString());
    }
}
```

```java
// ❌ Field injection forces you to use reflection in tests
class UserServiceTest {
    @Test
    void testWithFieldInjection() throws Exception {
        UserService service = new UserService();
        // Ugly: must use reflection to set private fields
        Field repoField = UserService.class.getDeclaredField("userRepository");
        repoField.setAccessible(true);
        repoField.set(service, mock(UserRepository.class));
    }
}
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| Preferred injection | Constructor — always explain why (immutability, testability) |
| @Autowired optional | Since Spring 4.3, single-constructor beans don't need @Autowired |
| Scope pitfall | Prototype inside singleton gives the same instance — use Provider/ObjectFactory |
| Circular deps | Sign of design smell — redesign first, @Lazy as last resort |
| DI without Spring | Shows you understand the pattern, not just the framework |

---

[← Previous: Q1 - Spring Basics](./q01-spring-basics.md) | [Next: Q3 - Hibernate/JPA →](./q03-hibernate-jpa.md)
