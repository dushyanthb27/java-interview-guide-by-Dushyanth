# Test Data Management

## 📋 Overview

Effective test data management is crucial for writing maintainable, reliable tests. This guide covers test data builders, factory patterns, database seeding strategies, data privacy considerations in tests, and test data versioning approaches for Java applications.

## 🎯 Why Test Data Management Matters

Poor test data management leads to:
- **Brittle Tests**: Tests that break when data changes
- **Slow Tests**: Expensive data setup and teardown
- **Data Leaks**: Sensitive data in test suites
- **Maintenance Burden**: Hard to understand and update test data
- **Flaky Tests**: Tests that depend on shared state

## 🏗️ Test Data Builders

### Basic Builder Pattern

```java
public class UserBuilder {
    private Long id = 1L;
    private String email = "test@example.com";
    private String name = "Test User";
    private String phone = "555-1234";
    private boolean active = true;
    private LocalDateTime createdAt = LocalDateTime.now();
    private LocalDateTime updatedAt = LocalDateTime.now();
    
    public static UserBuilder aUser() {
        return new UserBuilder();
    }
    
    public UserBuilder withId(Long id) {
        this.id = id;
        return this;
    }
    
    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }
    
    public UserBuilder withName(String name) {
        this.name = name;
        return this;
    }
    
    public UserBuilder withPhone(String phone) {
        this.phone = phone;
        return this;
    }
    
    public UserBuilder withActive(boolean active) {
        this.active = active;
        return this;
    }
    
    public UserBuilder withCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
        return this;
    }
    
    public UserBuilder withUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
        return this;
    }
    
    public User build() {
        return new User(id, email, name, phone, active, createdAt, updatedAt);
    }
}

// Usage in tests
@Test
void shouldCreateUser() {
    User user = UserBuilder.aUser()
        .withEmail("john@example.com")
        .withName("John Doe")
        .withActive(true)
        .build();
    
    userRepository.save(user);
    
    assertThat(userRepository.findById(user.getId())).isPresent();
}
```

### Nested Builders

```java
public class OrderBuilder {
    private Long id = 1L;
    private User customer;
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.PENDING;
    private BigDecimal totalAmount = BigDecimal.ZERO;
    private LocalDateTime createdAt = LocalDateTime.now();
    
    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }
    
    public OrderBuilder withCustomer(User customer) {
        this.customer = customer;
        return this;
    }
    
    public OrderBuilder withItems(List<OrderItem> items) {
        this.items = items;
        return this;
    }
    
    public OrderBuilder withItem(OrderItem item) {
        this.items.add(item);
        return this;
    }
    
    public OrderBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }
    
    public OrderBuilder withTotalAmount(BigDecimal totalAmount) {
        this.totalAmount = totalAmount;
        return this;
    }
    
    public Order build() {
        return new Order(id, customer, items, status, totalAmount, createdAt);
    }
}

// Usage with nested builders
@Test
void shouldCreateOrderWithItems() {
    User customer = UserBuilder.aUser()
        .withEmail("customer@example.com")
        .build();
    
    Order order = OrderBuilder.anOrder()
        .withCustomer(customer)
        .withItem(OrderItemBuilder.anItem()
            .withProductId(101L)
            .withQuantity(2)
            .withPrice(new BigDecimal("29.99"))
            .build())
        .withItem(OrderItemBuilder.anItem()
            .withProductId(102L)
            .withQuantity(1)
            .withPrice(new BigDecimal("49.99"))
            .build())
        .build();
    
    orderRepository.save(order);
    
    assertThat(order.getItems()).hasSize(2);
}
```

### Builder with Defaults

```java
public class TestDataDefaults {
    public static User defaultUser() {
        return UserBuilder.aUser()
            .withEmail("default@example.com")
            .withName("Default User")
            .withActive(true)
            .build();
    }
    
    public static Order defaultOrder() {
        return OrderBuilder.anOrder()
            .withCustomer(defaultUser())
            .withStatus(OrderStatus.PENDING)
            .build();
    }
    
    public static OrderItem defaultOrderItem() {
        return OrderItemBuilder.anItem()
            .withProductId(1L)
            .withQuantity(1)
            .withPrice(new BigDecimal("10.00"))
            .build();
    }
}

// Usage
@Test
void shouldProcessDefaultOrder() {
    Order order = TestDataDefaults.defaultOrder();
    orderService.process(order);
    
    assertThat(order.getStatus()).isEqualTo(OrderStatus.PROCESSED);
}
```

## 🏭 Factory Pattern

### Test Data Factory

```java
@Component
public class TestDataFactory {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    public User createUser(String email, String name) {
        User user = UserBuilder.aUser()
            .withEmail(email)
            .withName(name)
            .build();
        return userRepository.save(user);
    }
    
    public User createDefaultUser() {
        return createUser("default@example.com", "Default User");
    }
    
    public Order createOrder(User customer, List<OrderItem> items) {
        Order order = OrderBuilder.anOrder()
            .withCustomer(customer)
            .withItems(items)
            .build();
        return orderRepository.save(order);
    }
    
    public Order createOrderWithItems(User customer, int itemCount) {
        List<OrderItem> items = new ArrayList<>();
        for (int i = 0; i < itemCount; i++) {
            items.add(OrderItemBuilder.anItem()
                .withProductId((long) i)
                .withQuantity(1)
                .withPrice(new BigDecimal("10.00"))
                .build());
        }
        return createOrder(customer, items);
    }
}

// Usage in tests
@SpringBootTest
class OrderServiceTest {
    
    @Autowired
    private TestDataFactory testDataFactory;
    
    @Autowired
    private OrderService orderService;
    
    @Test
    void shouldProcessOrder() {
        User customer = testDataFactory.createDefaultUser();
        Order order = testDataFactory.createOrderWithItems(customer, 3);
        
        orderService.process(order);
        
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PROCESSED);
    }
}
```

### Dynamic Data Generation

```java
public class DynamicDataGenerator {
    private static final Random random = new Random();
    
    public static String randomEmail() {
        return "user" + random.nextInt(10000) + "@example.com";
    }
    
    public static String randomName() {
        String[] names = {"John", "Jane", "Bob", "Alice", "Charlie"};
        return names[random.nextInt(names.length)];
    }
    
    public static String randomPhone() {
        return "555-" + String.format("%04d", random.nextInt(10000));
    }
    
    public static BigDecimal randomPrice() {
        return new BigDecimal(
            random.nextDouble() * 100 + 1
        ).setScale(2, RoundingMode.HALF_UP);
    }
    
    public static LocalDateTime randomDate() {
        long minDay = LocalDate.now().minusDays(365).toEpochDay();
        long maxDay = LocalDate.now().toEpochDay();
        long randomDay = minDay + random.nextInt((int) (maxDay - minDay));
        return LocalDate.ofEpochDay(randomDay).atStartOfDay();
    }
}

// Usage
@Test
void shouldHandleRandomData() {
    User user = UserBuilder.aUser()
        .withEmail(DynamicDataGenerator.randomEmail())
        .withName(DynamicDataGenerator.randomName())
        .withPhone(DynamicDataGenerator.randomPhone())
        .build();
    
    userRepository.save(user);
    
    assertThat(userRepository.findById(user.getId())).isPresent();
}
```

## 🗄️ Database Seeding Strategies

### SQL Script Seeding

```sql
-- src/test/resources/data/test-users.sql
INSERT INTO users (id, email, name, phone, active, created_at, updated_at)
VALUES 
    (1, 'user1@example.com', 'User One', '555-0001', true, NOW(), NOW()),
    (2, 'user2@example.com', 'User Two', '555-0002', true, NOW(), NOW()),
    (3, 'user3@example.com', 'User Three', '555-0003', false, NOW(), NOW());

-- src/test/resources/data/test-orders.sql
INSERT INTO orders (id, customer_id, status, total_amount, created_at)
VALUES
    (1, 1, 'PENDING', 100.00, NOW()),
    (2, 1, 'COMPLETED', 150.00, NOW()),
    (3, 2, 'PROCESSING', 200.00, NOW());
```

```java
@SpringBootTest
@Sql(scripts = {
    "/data/test-users.sql",
    "/data/test-orders.sql"
}, executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@DisplayName("Database Seeding Tests")
class DatabaseSeedingTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldHaveSeededData() {
        assertThat(userRepository.count()).isEqualTo(3);
        assertThat(orderRepository.count()).isEqualTo(3);
    }
    
    @Test
    void shouldFindSeededUser() {
        Optional<User> user = userRepository.findByEmail("user1@example.com");
        assertThat(user).isPresent();
        assertThat(user.get().getName()).isEqualTo("User One");
    }
}
```

### Flyway Test Migrations

```sql
-- src/test/resources/db/migration/test/V1__test_data.sql
INSERT INTO users (email, name, active) VALUES
    ('test1@example.com', 'Test User 1', true),
    ('test2@example.com', 'Test User 2', true),
    ('test3@example.com', 'Test User 3', false);
```

```java
@SpringBootTest
@DisplayName("Flyway Test Data Tests")
class FlywayTestDataTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldLoadTestData() {
        assertThat(userRepository.count()).isEqualTo(3);
    }
}
```

### Programmatic Seeding

```java
@Component
public class TestDataSeeder {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @PostConstruct
    public void seedTestData() {
        if (shouldSeed()) {
            seedUsers();
            seedProducts();
            seedOrders();
        }
    }
    
    private boolean shouldSeed() {
        return "test".equals(System.getProperty("spring.profiles.active"));
    }
    
    private void seedUsers() {
        if (userRepository.count() == 0) {
            for (int i = 1; i <= 10; i++) {
                userRepository.save(UserBuilder.aUser()
                    .withEmail("user" + i + "@example.com")
                    .withName("User " + i)
                    .build());
            }
        }
    }
    
    private void seedProducts() {
        if (productRepository.count() == 0) {
            for (int i = 1; i <= 20; i++) {
                productRepository.save(ProductBuilder.aProduct()
                    .withName("Product " + i)
                    .withPrice(new BigDecimal(i * 10.00))
                    .build());
            }
        }
    }
    
    private void seedOrders() {
        if (orderRepository.count() == 0) {
            List<User> users = userRepository.findAll();
            List<Product> products = productRepository.findAll();
            
            for (int i = 0; i < 5; i++) {
                User customer = users.get(i % users.size());
                Order order = OrderBuilder.anOrder()
                    .withCustomer(customer)
                    .withItems(List.of(
                        OrderItemBuilder.anItem()
                            .withProduct(products.get(i))
                            .withQuantity(2)
                            .build()
                    ))
                    .build();
                orderRepository.save(order);
            }
        }
    }
}
```

## 🔒 Data Privacy in Tests

### Data Anonymization

```java
public class TestDataAnonymizer {
    
    public static String anonymizeEmail(String email) {
        if (email == null || email.isEmpty()) {
            return "anonymous@example.com";
        }
        String[] parts = email.split("@");
        return parts[0].substring(0, Math.min(3, parts[0].length())) + 
               "***@" + parts[1];
    }
    
    public static String anonymizeName(String name) {
        if (name == null || name.isEmpty()) {
            return "Anonymous";
        }
        return name.charAt(0) + "***";
    }
    
    public static String anonymizePhone(String phone) {
        if (phone == null || phone.isEmpty()) {
            return "555-****";
        }
        return phone.substring(0, phone.length() - 4) + "****";
    }
    
    public static String anonymizeCreditCard(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) {
            return "****-****-****-****";
        }
        return "****-****-****-" + cardNumber.substring(cardNumber.length() - 4);
    }
}

// Usage
@Test
void shouldAnonymizeSensitiveData() {
    User user = UserBuilder.aUser()
        .withEmail("real.user@company.com")
        .withName("Real User")
        .withPhone("555-1234")
        .build();
    
    String anonymizedEmail = TestDataAnonymizer.anonymizeEmail(user.getEmail());
    String anonymizedName = TestDataAnonymizer.anonymizeName(user.getName());
    
    assertThat(anonymizedEmail).isEqualTo("rea***@company.com");
    assertThat(anonymizedName).isEqualTo("R***");
}
```

### Synthetic Data Generation

```java
public class SyntheticDataGenerator {
    
    public static User generateSyntheticUser() {
        return UserBuilder.aUser()
            .withEmail(generateSyntheticEmail())
            .withName(generateSyntheticName())
            .withPhone(generateSyntheticPhone())
            .build();
    }
    
    private static String generateSyntheticEmail() {
        String[] domains = {"example.com", "test.org", "demo.net"};
        String username = "user" + UUID.randomUUID().toString().substring(0, 8);
        return username + "@" + domains[new Random().nextInt(domains.length)];
    }
    
    private static String generateSyntheticName() {
        String[] firstNames = {"John", "Jane", "Bob", "Alice", "Charlie"};
        String[] lastNames = {"Smith", "Johnson", "Williams", "Brown", "Jones"};
        return firstNames[new Random().nextInt(firstNames.length)] + " " +
               lastNames[new Random().nextInt(lastNames.length)];
    }
    
    private static String generateSyntheticPhone() {
        return "555-" + String.format("%04d", new Random().nextInt(10000));
    }
}
```

### Test Data Exclusion

```java
@SpringBootTest
@DisplayName("Data Privacy Tests")
class DataPrivacyTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should not include production data in tests")
    void shouldNotIncludeProductionData() {
        // Verify test database doesn't contain production data
        List<User> users = userRepository.findAll();
        
        users.forEach(user -> {
            assertThat(user.getEmail()).endsWith("@example.com");
            assertThat(user.getEmail()).doesNotContain("@company.com");
        });
    }
    
    @Test
    @DisplayName("Should use test-specific data")
    void shouldUseTestSpecificData() {
        User user = userRepository.findByEmail("test@example.com");
        assertThat(user).isPresent();
        assertThat(user.get()).extracting("name").isEqualTo("Test User");
    }
}
```

## 📊 Test Data Versioning

### Data Version Control

```java
public class TestDataVersion {
    private static final String CURRENT_VERSION = "1.0";
    
    public static String getVersion() {
        return CURRENT_VERSION;
    }
    
    public static boolean isCompatible(String version) {
        return CURRENT_VERSION.equals(version);
    }
}

// Usage in tests
@Test
void shouldUseCorrectDataVersion() {
    assertThat(TestDataVersion.getVersion()).isEqualTo("1.0");
}
```

### Migration Scripts

```sql
-- src/test/resources/db/migration/test/V1.1__update_test_data.sql
-- Update test data for version 1.1
UPDATE users SET active = true WHERE email LIKE 'user%@example.com';
```

### Data Snapshots

```java
@Component
public class TestDataSnapshot {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    public void takeSnapshot(String snapshotName) {
        Map<String, Object> snapshot = new HashMap<>();
        snapshot.put("users", userRepository.findAll());
        snapshot.put("orders", orderRepository.findAll());
        snapshot.put("timestamp", LocalDateTime.now());
        
        // Save snapshot to file or database
        saveSnapshot(snapshotName, snapshot);
    }
    
    public void restoreSnapshot(String snapshotName) {
        Map<String, Object> snapshot = loadSnapshot(snapshotName);
        
        // Clear existing data
        userRepository.deleteAll();
        orderRepository.deleteAll();
        
        // Restore data
        @SuppressWarnings("unchecked")
        List<User> users = (List<User>) snapshot.get("users");
        userRepository.saveAll(users);
        
        @SuppressWarnings("unchecked")
        List<Order> orders = (List<Order>) snapshot.get("orders");
        orderRepository.saveAll(orders);
    }
}
```

## 🎯 Test Data Cleanup

### Automatic Cleanup

```java
@SpringBootTest
@Transactional
@DisplayName("Automatic Cleanup Tests")
class AutomaticCleanupTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldCleanupAfterTest() {
        userRepository.save(UserBuilder.aUser().build());
        assertThat(userRepository.count()).isEqualTo(1);
    }
    
    // Transaction is rolled back automatically
    @Test
    void shouldStartWithCleanState() {
        assertThat(userRepository.count()).isEqualTo(0);
    }
}
```

### Manual Cleanup

```java
@SpringBootTest
@DisplayName("Manual Cleanup Tests")
class ManualCleanupTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @BeforeEach
    void setUp() {
        // Clean up before each test
        orderRepository.deleteAll();
        userRepository.deleteAll();
    }
    
    @AfterEach
    void tearDown() {
        // Clean up after each test
        orderRepository.deleteAll();
        userRepository.deleteAll();
    }
    
    @Test
    void shouldStartWithCleanDatabase() {
        assertThat(userRepository.count()).isEqualTo(0);
        assertThat(orderRepository.count()).isEqualTo(0);
    }
}
```

### Cleanup with @DirtiesContext

```java
@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
@DisplayName("Context Cleanup Tests")
class ContextCleanupTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldReloadContext() {
        userRepository.save(UserBuilder.aUser().build());
    }
    
    // Context is reloaded, database is clean
    @Test
    void shouldHaveCleanDatabase() {
        assertThat(userRepository.count()).isEqualTo(0);
    }
}
```

## 🎯 Test Data Management Best Practices

### DO ✅

- Use builders for complex test data
- Keep test data close to tests
- Use synthetic data, not production data
- Clean up test data after tests
- Version test data with migrations
- Anonymize sensitive data
- Use factories for common test data
- Document test data structure

### DON'T ❌

- Hardcode test data in multiple places
- Use production data in tests
- Share test data between tests
- Forget to clean up test data
- Embed sensitive data in tests
- Duplicate test data setup code
- Use magic numbers in test data
- Ignore test data versioning

## ⚠️ Common Pitfalls

### 1. Hardcoded Test Data

```java
// Bad: Hardcoded data scattered in tests
@Test
void test1() {
    User user = new User(1L, "test@example.com", "Test User");
}

@Test
void test2() {
    User user = new User(1L, "test@example.com", "Test User");
}

// Good: Use builders
@Test
void test1() {
    User user = UserBuilder.aUser().build();
}

@Test
void test2() {
    User user = UserBuilder.aUser().build();
}
```

### 2. Shared Test Data

```java
// Bad: Tests depend on shared data
@SpringBootTest
class SharedDataTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void test1() {
        userRepository.save(new User("user@example.com"));
    }
    
    @Test
    void test2() {
        // Assumes user exists from test1
        User user = userRepository.findByEmail("user@example.com");
    }
}

// Good: Each test is independent
@SpringBootTest
class IndependentDataTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void test1() {
        User user = userRepository.save(UserBuilder.aUser().build());
        // Test logic
    }
    
    @Test
    void test2() {
        User user = userRepository.save(UserBuilder.aUser().build());
        // Test logic
    }
}
```

### 3. Sensitive Data in Tests

```java
// Bad: Real sensitive data in tests
@Test
void shouldProcessPayment() {
    CreditCard card = new CreditCard("4111-1111-1111-1111", "12/25", "123");
}

// Good: Synthetic test data
@Test
void shouldProcessPayment() {
    CreditCard card = new CreditCard("4111-1111-1111-1111", "12/25", "123");
    // Use test-only payment gateway
}
```

## 🎓 Interview Discussion Points

When discussing test data management in interviews:

1. **Builder Pattern**: Explain how builders improve test readability
2. **Data Privacy**: Describe strategies for handling sensitive data
3. **Cleanup**: Discuss approaches for test data cleanup
4. **Performance**: Explain how data management affects test performance
5. **Maintenance**: Share strategies for keeping test data maintainable
6. **Seeding**: Compare different approaches to database seeding

## 📚 Related Topics

- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Test-Driven Development](./test-driven-development.md)

---

**Key Takeaway**: Effective test data management is essential for maintainable, reliable tests. Use builders, factories, and synthetic data to create clear, isolated test scenarios while protecting sensitive information and ensuring proper cleanup.
