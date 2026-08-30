# Test-Driven Development

## 📋 Overview

Test-Driven Development (TDD) is a software development process where tests are written before production code. This guide covers the Red-Green-Refactor cycle, TDD patterns and practices, testable design principles, and common pitfalls to avoid when practicing TDD in Java.

## 🎯 The TDD Philosophy

TDD is not just about testing—it's about design. Writing tests first forces you to:
- **Think before coding**: Clarify requirements and design before implementation
- **Write testable code**: Design code that's easy to test
- **Document behavior**: Tests serve as executable documentation
- **Enable refactoring**: Confidence to improve code without breaking functionality
- **Reduce debugging**: Catch bugs immediately when they're introduced

## 🔄 The Red-Green-Refactor Cycle

### Step 1: Red - Write a Failing Test

```java
@Test
@DisplayName("Should calculate total price with tax")
void shouldCalculateTotalPriceWithTax() {
    // Arrange
    Product product = new Product("Laptop", new BigDecimal("999.99"));
    TaxCalculator calculator = new TaxCalculator();
    
    // Act
    BigDecimal total = calculator.calculateTotal(product, 0.08);
    
    // Assert
    assertEquals(new BigDecimal("1079.99"), total);
}
```

### Step 2: Green - Make the Test Pass

```java
public class TaxCalculator {
    public BigDecimal calculateTotal(Product product, double taxRate) {
        // Simplest implementation to make test pass
        return new BigDecimal("1079.99");
    }
}
```

### Step 3: Refactor - Improve the Code

```java
public class TaxCalculator {
    public BigDecimal calculateTotal(Product product, double taxRate) {
        BigDecimal price = product.getPrice();
        BigDecimal tax = price.multiply(BigDecimal.valueOf(taxRate));
        return price.add(tax).setScale(2, RoundingMode.HALF_UP);
    }
}
```

## 🏗️ TDD Patterns

### Outside-In TDD

Start from the outside (acceptance criteria) and work inward.

```java
// 1. Write acceptance test
@SpringBootTest
@DisplayName("Order Processing Acceptance Test")
class OrderProcessingAcceptanceTest {
    
    @Test
    @DisplayName("Should process order and send confirmation")
    void shouldProcessOrderAndSendConfirmation() {
        // Given
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId(1L)
            .items(List.of(
                OrderItem.builder().productId(101L).quantity(2).build()
            ))
            .build();
        
        // When
        OrderResponse response = orderService.processOrder(request);
        
        // Then
        assertThat(response.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(response.getConfirmationNumber()).isNotNull();
        verify(emailService).sendConfirmation(any());
    }
}

// 2. Write service test
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Test
    @DisplayName("Should create order with correct total")
    void shouldCreateOrderWithCorrectTotal() {
        // Implementation details...
    }
}

// 3. Write repository test
@DataJpaTest
class OrderRepositoryTest {
    
    @Test
    @DisplayName("Should save order with items")
    void shouldSaveOrderWithItems() {
        // Implementation details...
    }
}
```

### Inside-Out TDD

Start from the core domain and work outward.

```java
// 1. Write domain model test
class MoneyTest {
    
    @Test
    @DisplayName("Should add money correctly")
    void shouldAddMoneyCorrectly() {
        Money money1 = new Money("100.00", "USD");
        Money money2 = new Money("50.00", "USD");
        
        Money total = money1.add(money2);
        
        assertEquals(new Money("150.00", "USD"), total);
    }
    
    @Test
    @DisplayName("Should throw exception when adding different currencies")
    void shouldThrowExceptionWhenAddingDifferentCurrencies() {
        Money usd = new Money("100.00", "USD");
        Money eur = new Money("100.00", "EUR");
        
        assertThatThrownBy(() -> usd.add(eur))
            .isInstanceOf(CurrencyMismatchException.class);
    }
}

// 2. Write service test using domain
class PaymentServiceTest {
    
    @Test
    @DisplayName("Should process payment with correct amount")
    void shouldProcessPaymentWithCorrectAmount() {
        // Uses Money domain model
    }
}

// 3. Write controller test using service
class PaymentControllerTest {
    
    @Test
    @DisplayName("Should return payment confirmation")
    void shouldReturnPaymentConfirmation() {
        // Uses PaymentService
    }
}
```

### Triangulation

Use multiple examples to generalize behavior.

```java
// First test
@Test
void shouldCalculateTaxFor10Percent() {
    assertEquals(new BigDecimal("110.00"), 
        calculator.calculateTax(new BigDecimal("100.00"), 0.10));
}

// Simplest implementation
public BigDecimal calculateTax(BigDecimal amount, double rate) {
    return new BigDecimal("110.00");
}

// Second test - triangulation
@Test
void shouldCalculateTaxFor8Percent() {
    assertEquals(new BigDecimal("108.00"), 
        calculator.calculateTax(new BigDecimal("100.00"), 0.08));
}

// Generalized implementation
public BigDecimal calculateTax(BigDecimal amount, double rate) {
    return amount.multiply(BigDecimal.valueOf(rate))
                 .add(amount)
                 .setScale(2, RoundingMode.HALF_UP);
}
```

### Fake It Till You Make It

Return hardcoded values to get the test passing, then generalize.

```java
// Test
@Test
void shouldReturnUserById() {
    User user = userService.findById(1L);
    assertEquals("John Doe", user.getName());
}

// Fake implementation
public User findById(Long id) {
    return new User(1L, "john@example.com", "John Doe");
}

// Real implementation (after more tests)
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

## 🎯 Testable Design Principles

### Dependency Injection

```java
// Bad: Hard to test
public class UserService {
    private UserRepository userRepository = new UserRepository();
    
    public User findById(Long id) {
        return userRepository.findById(id);
    }
}

// Good: Easy to test with DI
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User findById(Long id) {
        return userRepository.findById(id);
    }
}

// Test
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void shouldFindUserById() {
        when(userRepository.findById(1L))
            .thenReturn(Optional.of(user));
        
        User result = userService.findById(1L);
        
        assertThat(result).isNotNull();
    }
}
```

### Single Responsibility Principle

```java
// Bad: Multiple responsibilities
public class OrderService {
    public Order processOrder(Order order) {
        // Validation
        if (order.getItems().isEmpty()) {
            throw new InvalidOrderException();
        }
        
        // Payment processing
        paymentService.charge(order.getTotal());
        
        // Inventory update
        inventoryService.updateStock(order.getItems());
        
        // Email notification
        emailService.sendConfirmation(order);
        
        return order;
    }
}

// Good: Single responsibility per class
public class OrderService {
    private final OrderValidator validator;
    private final PaymentProcessor paymentProcessor;
    private final InventoryManager inventoryManager;
    private final NotificationService notificationService;
    
    public Order processOrder(Order order) {
        validator.validate(order);
        paymentProcessor.process(order);
        inventoryManager.update(order);
        notificationService.notify(order);
        return order;
    }
}
```

### Interface Segregation

```java
// Bad: Fat interface
public interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void delete(Long id);
    List<User> search(String query);
    List<User> findByRole(String role);
    // ... many more methods
}

// Good: Segregated interfaces
public interface UserReader {
    Optional<User> findById(Long id);
    List<User> findAll();
}

public interface UserWriter {
    User save(User user);
    void delete(Long id);
}

public class UserService {
    private final UserReader userReader;
    private final UserWriter userWriter;
    
    public UserService(UserReader userReader, UserWriter userWriter) {
        this.userReader = userReader;
        this.userWriter = userWriter;
    }
}
```

## 📝 TDD Example: Building a Feature

Let's build a discount feature using TDD.

### Step 1: Write First Test

```java
@Test
@DisplayName("Should apply 10% discount for orders over $100")
void shouldApplyDiscountForOrdersOver100() {
    Order order = Order.builder()
        .totalAmount(new BigDecimal("150.00"))
        .build();
    
    DiscountService discountService = new DiscountService();
    BigDecimal discount = discountService.calculateDiscount(order);
    
    assertEquals(new BigDecimal("15.00"), discount);
}
```

### Step 2: Make It Pass

```java
public class DiscountService {
    public BigDecimal calculateDiscount(Order order) {
        return new BigDecimal("15.00");
    }
}
```

### Step 3: Add Second Test

```java
@Test
@DisplayName("Should not apply discount for orders under $100")
void shouldNotApplyDiscountForOrdersUnder100() {
    Order order = Order.builder()
        .totalAmount(new BigDecimal("50.00"))
        .build();
    
    DiscountService discountService = new DiscountService();
    BigDecimal discount = discountService.calculateDiscount(order);
    
    assertEquals(BigDecimal.ZERO, discount);
}
```

### Step 4: Refactor Implementation

```java
public class DiscountService {
    private static final BigDecimal DISCOUNT_THRESHOLD = new BigDecimal("100.00");
    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.10");
    
    public BigDecimal calculateDiscount(Order order) {
        if (order.getTotalAmount().compareTo(DISCOUNT_THRESHOLD) > 0) {
            return order.getTotalAmount()
                .multiply(DISCOUNT_RATE)
                .setScale(2, RoundingMode.HALF_UP);
        }
        return BigDecimal.ZERO;
    }
}
```

### Step 5: Add Edge Case Test

```java
@Test
@DisplayName("Should not apply discount for orders exactly $100")
void shouldNotApplyDiscountForOrdersExactly100() {
    Order order = Order.builder()
        .totalAmount(new BigDecimal("100.00"))
        .build();
    
    DiscountService discountService = new DiscountService();
    BigDecimal discount = discountService.calculateDiscount(order);
    
    assertEquals(BigDecimal.ZERO, discount);
}
```

### Step 6: Add Customer Loyalty Test

```java
@Test
@DisplayName("Should apply additional 5% discount for premium customers")
void shouldApplyAdditionalDiscountForPremiumCustomers() {
    Order order = Order.builder()
        .totalAmount(new BigDecimal("150.00"))
        .customer(Customer.builder()
            .type(CustomerType.PREMIUM)
            .build())
        .build();
    
    DiscountService discountService = new DiscountService();
    BigDecimal discount = discountService.calculateDiscount(order);
    
    assertEquals(new BigDecimal("22.50"), discount); // 15 + 7.50
}
```

### Step 7: Refactor with Strategy Pattern

```java
public interface DiscountStrategy {
    BigDecimal calculateDiscount(Order order);
}

public class VolumeDiscountStrategy implements DiscountStrategy {
    private static final BigDecimal THRESHOLD = new BigDecimal("100.00");
    private static final BigDecimal RATE = new BigDecimal("0.10");
    
    @Override
    public BigDecimal calculateDiscount(Order order) {
        if (order.getTotalAmount().compareTo(THRESHOLD) > 0) {
            return order.getTotalAmount().multiply(RATE)
                .setScale(2, RoundingMode.HALF_UP);
        }
        return BigDecimal.ZERO;
    }
}

public class LoyaltyDiscountStrategy implements DiscountStrategy {
    private static final BigDecimal PREMIUM_RATE = new BigDecimal("0.05");
    
    @Override
    public BigDecimal calculateDiscount(Order order) {
        if (order.getCustomer().getType() == CustomerType.PREMIUM) {
            return order.getTotalAmount().multiply(PREMIUM_RATE)
                .setScale(2, RoundingMode.HALF_UP);
        }
        return BigDecimal.ZERO;
    }
}

public class DiscountService {
    private final List<DiscountStrategy> strategies;
    
    public DiscountService(List<DiscountStrategy> strategies) {
        this.strategies = strategies;
    }
    
    public BigDecimal calculateDiscount(Order order) {
        return strategies.stream()
            .map(strategy -> strategy.calculateDiscount(order))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

## ⚠️ Common TDD Pitfalls

### 1. Writing Too Many Tests at Once

```java
// Bad: Writing multiple tests before implementation
@Test
void test1() { }
@Test
void test2() { }
@Test
void test3() { }
// Now implement all at once - overwhelming

// Good: One test at a time
@Test
void shouldCalculateBasicDiscount() { }
// Implement
// Then next test
```

### 2. Skipping the Refactor Step

```java
// Bad: Only Red-Green, no Refactor
@Test
void shouldCalculateDiscount() {
    // Test passes with messy implementation
}

// Good: Always refactor after Green
@Test
void shouldCalculateDiscount() {
    // Test passes with clean, refactored implementation
}
```

### 3. Testing Implementation Details

```java
// Bad: Testing internal implementation
@Test
void shouldCallRepositoryThreeTimes() {
    userService.processUsers();
    verify(repository, times(3)).save(any());
}

// Good: Testing observable behavior
@Test
void shouldProcessAllUsersSuccessfully() {
    List<User> users = userService.processUsers();
    assertThat(users).allMatch(User::isActive);
}
```

### 4. Not Writing Tests for Edge Cases

```java
// Bad: Only happy path
@Test
void shouldCalculateDiscount() {
    // Normal case only
}

// Good: Include edge cases
@Test
void shouldCalculateDiscount() { }
@Test
void shouldHandleZeroAmount() { }
@Test
void shouldHandleNegativeAmount() { }
@Test
void shouldHandleNullOrder() { }
```

## 🎯 TDD in Legacy Code

### Characterization Tests

Before refactoring legacy code, write characterization tests to document current behavior.

```java
@Test
void characterizeLegacyBehavior() {
    // This test documents what the code currently does
    // even if the behavior seems wrong
    LegacyService service = new LegacyService();
    String result = service.processInput("test");
    
    assertEquals("EXPECTED_OUTPUT", result); // Document current behavior
}
```

### The Golden Master

Capture output of legacy code, then use it as expected result in tests.

```java
@Test
void goldenMasterTest() throws Exception {
    // Run legacy code and capture output
    String actualOutput = captureLegacyOutput("test-input.txt");
    
    // Compare with golden master
    String expectedOutput = Files.readString(
        Path.of("src/test/resources/golden-master/test-input-expected.txt")
    );
    
    assertEquals(expectedOutput, actualOutput);
}
```

## 🎓 Interview Discussion Points

When discussing TDD in interviews:

1. **Benefits**: Explain how TDD improves design and reduces bugs
2. **Challenges**: Discuss the learning curve and initial slowdown
3. **Adoption**: Share strategies for introducing TDD to teams
4. **Legacy Code**: Explain how to apply TDD to existing codebases
5. **Red-Green-Refactor**: Describe your experience with the cycle
6. **Testable Design**: Discuss how TDD influences design decisions

## 📚 Related Topics

- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Mutation Testing](./mutation-testing.md)

---

**Key Takeaway**: TDD is a design practice that happens to result in tests. The primary benefit is better design and cleaner code, with comprehensive test coverage being a valuable side effect.
