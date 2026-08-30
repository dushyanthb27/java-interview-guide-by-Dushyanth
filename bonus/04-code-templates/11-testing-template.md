# Testing Template

## Overview
JUnit 5 and Mockito testing patterns for unit, integration, and parameterized tests.

## Use Cases
- Unit testing business logic
- Mocking external dependencies
- Integration testing
- Parameterized testing
- Test fixtures and setup
- Assertion patterns

## Implementation

Complete testing patterns and examples.

## Code

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
import org.mockito.Mock;
import org.mockito.InjectMocks;
import org.mockito.MockitoAnnotations;
import org.mockito.ArgumentCaptor;
import org.mockito.Spy;

/**
 * SIMPLE VERSION: Basic Unit Tests
 * - Simple assertions
 * - Setup and teardown
 * - Basic test structure
 */
@DisplayName("User Service Tests")
public class SimpleUserServiceTest {
    private UserService userService;

    @BeforeEach
    public void setUp() {
        userService = new UserService();
    }

    @Test
    @DisplayName("Should create user successfully")
    public void testCreateUser() {
        User user = userService.createUser("john", "john@example.com");

        assertNotNull(user);
        assertEquals("john", user.getName());
        assertEquals("john@example.com", user.getEmail());
    }

    @Test
    @DisplayName("Should throw exception for invalid email")
    public void testCreateUserWithInvalidEmail() {
        assertThrows(IllegalArgumentException.class, () -> {
            userService.createUser("john", "invalid-email");
        });
    }

    @Test
    @DisplayName("Should find user by id")
    public void testFindUserById() {
        User user = userService.createUser("john", "john@example.com");
        User found = userService.findUserById(user.getId());

        assertNotNull(found);
        assertEquals(user.getId(), found.getId());
    }

    @Test
    @DisplayName("Should return null for non-existent user")
    public void testFindNonExistentUser() {
        User found = userService.findUserById("non-existent");
        assertNull(found);
    }
}

/**
 * PRODUCTION VERSION: Advanced Testing with Mocks
 * - Mockito for dependencies
 * - Argument captors
 * - Verification
 * - Spy objects
 */
@DisplayName("User Repository Tests")
public class ProductionUserRepositoryTest {
    @Mock
    private DatabaseConnection databaseConnection;

    @InjectMocks
    private UserRepository userRepository;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
    @DisplayName("Should save user to database")
    public void testSaveUser() {
        User user = new User("john", "john@example.com");

        userRepository.save(user);

        // Verify database connection was called
        verify(databaseConnection).execute(anyString());
    }

    @Test
    @DisplayName("Should retrieve user from database")
    public void testFindUser() {
        User expectedUser = new User("john", "john@example.com");
        when(databaseConnection.query(anyString()))
            .thenReturn(expectedUser);

        User result = userRepository.findById("123");

        assertEquals(expectedUser, result);
        verify(databaseConnection).query(contains("SELECT"));
    }

    @Test
    @DisplayName("Should handle database errors")
    public void testFindUserWithDatabaseError() {
        when(databaseConnection.query(anyString()))
            .thenThrow(new RuntimeException("Database error"));

        assertThrows(RuntimeException.class, () -> {
            userRepository.findById("123");
        });
    }

    @Test
    @DisplayName("Should capture arguments")
    public void testSaveUserWithArgumentCaptor() {
        User user = new User("john", "john@example.com");
        ArgumentCaptor<String> sqlCaptor = ArgumentCaptor.forClass(String.class);

        userRepository.save(user);

        verify(databaseConnection).execute(sqlCaptor.capture());
        String sql = sqlCaptor.getValue();
        assertTrue(sql.contains("INSERT"));
        assertTrue(sql.contains("john"));
    }

    @Test
    @DisplayName("Should verify method call count")
    public void testMultipleCalls() {
        userRepository.findById("1");
        userRepository.findById("2");
        userRepository.findById("3");

        verify(databaseConnection, times(3)).query(anyString());
    }

    @Test
    @DisplayName("Should verify no interactions")
    public void testNoInteractions() {
        // Some operation that shouldn't call database
        User user = new User("john", "john@example.com");
        user.getName();

        verifyNoInteractions(databaseConnection);
    }
}

/**
 * PARAMETERIZED TESTS: Test multiple inputs
 */
@DisplayName("Email Validation Tests")
public class ParameterizedEmailValidationTest {
    private EmailValidator validator;

    @BeforeEach
    public void setUp() {
        validator = new EmailValidator();
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "john@example.com",
        "jane.doe@company.co.uk",
        "user+tag@domain.org"
    })
    @DisplayName("Should validate correct emails")
    public void testValidEmails(String email) {
        assertTrue(validator.isValid(email));
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "invalid-email",
        "user@",
        "@example.com",
        "user@.com"
    })
    @DisplayName("Should reject invalid emails")
    public void testInvalidEmails(String email) {
        assertFalse(validator.isValid(email));
    }

    @ParameterizedTest
    @CsvSource({
        "john@example.com, true",
        "invalid-email, false",
        "jane@company.co.uk, true",
        "user@, false"
    })
    @DisplayName("Should validate emails from CSV")
    public void testEmailsFromCsv(String email, boolean expected) {
        assertEquals(expected, validator.isValid(email));
    }

    @ParameterizedTest
    @MethodSource("provideEmails")
    @DisplayName("Should validate emails from method source")
    public void testEmailsFromMethod(String email, boolean expected) {
        assertEquals(expected, validator.isValid(email));
    }

    static java.util.stream.Stream<org.junit.jupiter.params.provider.Arguments> 
    provideEmails() {
        return java.util.stream.Stream.of(
            org.junit.jupiter.params.provider.Arguments.of(
                "john@example.com", true),
            org.junit.jupiter.params.provider.Arguments.of(
                "invalid-email", false),
            org.junit.jupiter.params.provider.Arguments.of(
                "jane@company.co.uk", true)
        );
    }
}

/**
 * NESTED TESTS: Organize related tests
 */
@DisplayName("Payment Service Tests")
public class NestedPaymentServiceTest {
    private PaymentService paymentService;

    @BeforeEach
    public void setUp() {
        paymentService = new PaymentService();
    }

    @Nested
    @DisplayName("Successful Payment Tests")
    class SuccessfulPaymentTests {
        @Test
        @DisplayName("Should process payment successfully")
        public void testProcessPayment() {
            Payment payment = paymentService.process(100.0);
            assertEquals(PaymentStatus.SUCCESS, payment.getStatus());
        }

        @Test
        @DisplayName("Should generate transaction ID")
        public void testGenerateTransactionId() {
            Payment payment = paymentService.process(50.0);
            assertNotNull(payment.getTransactionId());
        }
    }

    @Nested
    @DisplayName("Failed Payment Tests")
    class FailedPaymentTests {
        @Test
        @DisplayName("Should fail for negative amount")
        public void testNegativeAmount() {
            assertThrows(IllegalArgumentException.class, () -> {
                paymentService.process(-100.0);
            });
        }

        @Test
        @DisplayName("Should fail for zero amount")
        public void testZeroAmount() {
            assertThrows(IllegalArgumentException.class, () -> {
                paymentService.process(0.0);
            });
        }
    }

    @Nested
    @DisplayName("Refund Tests")
    class RefundTests {
        @Test
        @DisplayName("Should refund successful payment")
        public void testRefund() {
            Payment payment = paymentService.process(100.0);
            Payment refund = paymentService.refund(payment.getTransactionId());

            assertEquals(PaymentStatus.REFUNDED, refund.getStatus());
        }
    }
}

/**
 * INTEGRATION TESTS: Test multiple components
 */
@DisplayName("User Service Integration Tests")
public class UserServiceIntegrationTest {
    private UserService userService;
    private UserRepository userRepository;
    private EmailService emailService;

    @BeforeEach
    public void setUp() {
        userRepository = new UserRepository();
        emailService = mock(EmailService.class);
        userService = new UserService(userRepository, emailService);
    }

    @Test
    @DisplayName("Should create user and send welcome email")
    public void testCreateUserWithEmail() {
        User user = userService.createUser("john", "john@example.com");

        assertNotNull(user);
        verify(emailService).sendWelcomeEmail("john@example.com");
    }

    @Test
    @DisplayName("Should handle email service failure gracefully")
    public void testCreateUserWithEmailFailure() {
        doThrow(new RuntimeException("Email service down"))
            .when(emailService).sendWelcomeEmail(anyString());

        assertThrows(RuntimeException.class, () -> {
            userService.createUser("john", "john@example.com");
        });
    }
}

/**
 * TEST FIXTURES AND BUILDERS
 */
public class TestFixtures {
    /**
     * User builder for tests
     */
    public static class UserBuilder {
        private String name = "John Doe";
        private String email = "john@example.com";
        private String phone = "555-1234";

        public UserBuilder withName(String name) {
            this.name = name;
            return this;
        }

        public UserBuilder withEmail(String email) {
            this.email = email;
            return this;
        }

        public UserBuilder withPhone(String phone) {
            this.phone = phone;
            return this;
        }

        public User build() {
            return new User(name, email, phone);
        }
    }

    /**
     * Test data factory
     */
    public static class TestDataFactory {
        public static User createDefaultUser() {
            return new UserBuilder().build();
        }

        public static User createUser(String name, String email) {
            return new UserBuilder()
                .withName(name)
                .withEmail(email)
                .build();
        }

        public static java.util.List<User> createUsers(int count) {
            java.util.List<User> users = new java.util.ArrayList<>();
            for (int i = 0; i < count; i++) {
                users.add(new UserBuilder()
                    .withName("User" + i)
                    .withEmail("user" + i + "@example.com")
                    .build());
            }
            return users;
        }
    }
}

/**
 * ASSERTION PATTERNS
 */
public class AssertionPatterns {
    @Test
    public void testAssertions() {
        // Basic assertions
        assertTrue(true);
        assertFalse(false);
        assertNull(null);
        assertNotNull("value");
        assertEquals(5, 5);
        assertNotEquals(5, 6);

        // String assertions
        String text = "Hello World";
        assertTrue(text.contains("World"));
        assertTrue(text.startsWith("Hello"));

        // Collection assertions
        java.util.List<Integer> list = java.util.Arrays.asList(1, 2, 3);
        assertEquals(3, list.size());
        assertTrue(list.contains(2));

        // Exception assertions
        assertThrows(IllegalArgumentException.class, () -> {
            throw new IllegalArgumentException("Error");
        });

        // All assertions
        assertAll(
            () -> assertEquals(5, 5),
            () -> assertTrue(true),
            () -> assertNotNull("value")
        );
    }
}

/**
 * MOCK PATTERNS
 */
public class MockPatterns {
    @Test
    public void testMockingPatterns() {
        // Create mock
        java.util.List<String> mockList = mock(java.util.List.class);

        // Setup behavior
        when(mockList.get(0)).thenReturn("first");
        when(mockList.size()).thenReturn(3);

        // Use mock
        assertEquals("first", mockList.get(0));
        assertEquals(3, mockList.size());

        // Verify
        verify(mockList).get(0);
        verify(mockList, times(1)).size();
    }

    @Test
    public void testSpyPatterns() {
        // Create spy (partial mock)
        java.util.List<String> list = new java.util.ArrayList<>();
        java.util.List<String> spyList = spy(list);

        // Spy wraps real object
        spyList.add("item1");
        assertEquals(1, spyList.size());

        // Can override specific methods
        when(spyList.get(0)).thenReturn("mocked");
        assertEquals("mocked", spyList.get(0));
    }
}

/**
 * EXAMPLE CLASSES FOR TESTING
 */
class User {
    private String id;
    private String name;
    private String email;
    private String phone;

    public User(String name, String email) {
        this.id = java.util.UUID.randomUUID().toString();
        this.name = name;
        this.email = email;
    }

    public User(String name, String email, String phone) {
        this(name, email);
        this.phone = phone;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
}

class UserService {
    private UserRepository repository;
    private EmailService emailService;

    public UserService() {
        this.repository = new UserRepository();
    }

    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    public User createUser(String name, String email) {
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        User user = new User(name, email);
        repository.save(user);
        if (emailService != null) {
            emailService.sendWelcomeEmail(email);
        }
        return user;
    }

    public User findUserById(String id) {
        return repository.findById(id);
    }
}

class UserRepository {
    private java.util.Map<String, User> users = new java.util.HashMap<>();

    public void save(User user) {
        users.put(user.getId(), user);
    }

    public User findById(String id) {
        return users.get(id);
    }
}

class DatabaseConnection {
    public Object query(String sql) {
        return null;
    }

    public void execute(String sql) {
    }
}

class EmailValidator {
    public boolean isValid(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
}

class EmailService {
    public void sendWelcomeEmail(String email) {
    }
}

class Payment {
    private String transactionId;
    private PaymentStatus status;

    public Payment(String transactionId, PaymentStatus status) {
        this.transactionId = transactionId;
        this.status = status;
    }

    public String getTransactionId() { return transactionId; }
    public PaymentStatus getStatus() { return status; }
}

enum PaymentStatus {
    SUCCESS, FAILED, REFUNDED
}

class PaymentService {
    public Payment process(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        return new Payment(java.util.UUID.randomUUID().toString(), 
                          PaymentStatus.SUCCESS);
    }

    public Payment refund(String transactionId) {
        return new Payment(transactionId, PaymentStatus.REFUNDED);
    }
}
```

## Testing Best Practices

```java
/**
 * TESTING BEST PRACTICES
 */
public class TestingBestPractices {
    /**
     * 1. Test one thing per test
     * 2. Use descriptive test names
     * 3. Follow AAA pattern: Arrange, Act, Assert
     * 4. Use test fixtures and builders
     * 5. Mock external dependencies
     * 6. Verify behavior, not implementation
     * 7. Test edge cases and error conditions
     * 8. Keep tests independent
     * 9. Use parameterized tests for multiple inputs
     * 10. Maintain test code quality
     */

    @Test
    @DisplayName("Should calculate total price with tax")
    public void testCalculateTotalPrice() {
        // Arrange
        double basePrice = 100.0;
        double taxRate = 0.1;
        PriceCalculator calculator = new PriceCalculator();

        // Act
        double total = calculator.calculateTotal(basePrice, taxRate);

        // Assert
        assertEquals(110.0, total);
    }
}
```

## Performance Considerations

### Test Execution Time
- **Unit tests**: < 1ms per test
- **Integration tests**: < 100ms per test
- **Total test suite**: < 10 seconds

### Optimization Tips
1. **Use mocks**: Avoid slow I/O operations
2. **Parallel execution**: Run independent tests in parallel
3. **Test fixtures**: Reuse setup across tests
4. **Avoid sleep()**: Use proper synchronization
5. **Minimal database**: Use in-memory databases for tests

## Common Pitfalls

1. **Testing implementation details**: Test behavior instead
2. **Brittle tests**: Too specific, break on minor changes
3. **Slow tests**: Using real databases, network calls
4. **Flaky tests**: Non-deterministic, timing-dependent
5. **Poor naming**: Unclear what test is testing
6. **No assertions**: Tests that don't verify anything
7. **Test interdependence**: Tests affecting each other
8. **Over-mocking**: Mocking too much, missing real bugs

## Test Coverage Goals

- **Unit tests**: 80%+ coverage
- **Integration tests**: Critical paths
- **E2E tests**: Happy path and error scenarios
- **Overall**: Aim for meaningful coverage, not 100%
