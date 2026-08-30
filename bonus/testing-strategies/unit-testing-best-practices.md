# Unit Testing Best Practices

## 📋 Overview

Unit testing is the foundation of a robust testing strategy. This guide covers advanced JUnit 5 patterns, Mockito best practices, and testing patterns that senior developers should master. Well-written unit tests are fast, reliable, and serve as living documentation of your code's behavior.

## 🎯 What Makes a Good Unit Test?

A good unit test should be:
- **Fast**: Runs in milliseconds, not seconds
- **Isolated**: Doesn't depend on other tests or external systems
- **Repeatable**: Produces the same result every time
- **Self-validating**: Has a clear pass/fail criteria
- **Timely**: Written alongside or before production code
- **Readable**: Clearly expresses what it's testing and why

## 🔧 JUnit 5 Advanced Patterns

### Parameterized Tests

Parameterized tests allow you to run the same test with different inputs, reducing code duplication and improving test coverage.

```java
@ParameterizedTest
@ValueSource(strings = {"racecar", "madam", "level"})
@DisplayName("Should identify palindromes correctly")
void shouldIdentifyPalindromes(String word) {
    assertTrue(PalindromeChecker.isPalindrome(word));
}

@ParameterizedTest
@CsvSource({
    "2, 4, 6",
    "3, 5, 8",
    "10, 20, 30"
})
@DisplayName("Should add two numbers correctly")
void shouldAddTwoNumbers(int a, int b, int expected) {
    Calculator calculator = new Calculator();
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideInvalidEmails")
@DisplayName("Should reject invalid email formats")
void shouldRejectInvalidEmails(String email) {
    EmailValidator validator = new EmailValidator();
    assertFalse(validator.isValid(email));
}

private static Stream<Arguments> provideInvalidEmails() {
    return Stream.of(
        Arguments.of("invalid"),
        Arguments.of("@example.com"),
        Arguments.of("user@"),
        Arguments.of("user @example.com")
    );
}
```

### Dynamic Tests

Dynamic tests allow you to generate tests at runtime, useful for testing data-driven scenarios.

```java
@TestFactory
@DisplayName("Dynamic tests for Fibonacci sequence")
Stream<DynamicTest> fibonacciTests() {
    FibonacciCalculator calculator = new FibonacciCalculator();
    List<Integer> inputs = List.of(0, 1, 2, 5, 10);
    
    return inputs.stream()
        .map(input -> DynamicTest.dynamicTest(
            "Fibonacci(" + input + ")",
            () -> {
                long result = calculator.calculate(input);
                assertNotNull(result);
                assertTrue(result >= 0);
            }
        ));
}
```

### Test Templates

Test templates allow you to define a test structure that can be invoked multiple times with different contexts.

```java
@TestTemplate
@ExtendWith(CalculatorTestExtension.class)
@DisplayName("Calculator operations test template")
void calculatorOperationsTest(CalculatorTestCase testCase) {
    Calculator calculator = new Calculator();
    double result = calculator.execute(
        testCase.getOperation(), 
        testCase.getOperand1(), 
        testCase.getOperand2()
    );
    assertEquals(testCase.getExpected(), result, 0.001);
}
```

### Nested Tests

Nested tests organize tests hierarchically, improving readability and allowing shared setup/teardown.

```java
@DisplayName("UserService Tests")
class UserServiceTest {
    
    private UserService userService;
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        userService = new UserService(userRepository);
    }
    
    @Nested
    @DisplayName("User Creation")
    class UserCreation {
        
        @Test
        @DisplayName("Should create user with valid data")
        void shouldCreateUserWithValidData() {
            CreateUserRequest request = new CreateUserRequest(
                "test@example.com", "John Doe"
            );
            
            User savedUser = new User(1L, "test@example.com", "John Doe");
            when(userRepository.save(any())).thenReturn(savedUser);
            
            User result = userService.createUser(request);
            
            assertThat(result.getId()).isEqualTo(1L);
            verify(userRepository).save(any());
        }
        
        @Nested
        @DisplayName("Validation")
        class Validation {
            
            @ParameterizedTest
            @NullSource
            @EmptySource
            @DisplayName("Should reject null or empty email")
            void shouldRejectNullOrEmptyEmail(String email) {
                CreateUserRequest request = new CreateUserRequest(email, "John Doe");
                
                assertThatThrownBy(() -> userService.createUser(request))
                    .isInstanceOf(ValidationException.class);
            }
        }
    }
    
    @Nested
    @DisplayName("User Retrieval")
    class UserRetrieval {
        
        @Test
        @DisplayName("Should find user by ID")
        void shouldFindUserById() {
            User user = new User(1L, "test@example.com", "John Doe");
            when(userRepository.findById(1L)).thenReturn(Optional.of(user));
            
            Optional<User> result = userService.findById(1L);
            
            assertTrue(result.isPresent());
            assertThat(result.get().getEmail()).isEqualTo("test@example.com");
        }
    }
}
```

## 🎭 Mockito Advanced Patterns

### Argument Matchers

Use argument matchers for flexible verification while maintaining type safety.

```java
@Test
@DisplayName("Should save user with correct properties")
void shouldSaveUserWithCorrectProperties() {
    UserService userService = new UserService(userRepository);
    
    userService.createUser("test@example.com", "John Doe");
    
    verify(userRepository).save(argThat(user -> 
        user.getEmail().equals("test@example.com") &&
        user.getName().equals("John Doe") &&
        user.getCreatedAt() != null
    ));
}

@Test
@DisplayName("Should handle any string argument")
void shouldHandleAnyStringArgument() {
    EmailService emailService = mock(EmailService.class);
    
    emailService.sendEmail(anyString(), anyString());
    
    verify(emailService).sendEmail(anyString(), eq("Expected Subject"));
}
```

### Answer Patterns

Use `Answer` for complex mock behavior that depends on input arguments.

```java
@Test
@DisplayName("Should handle repository save with ID generation")
void shouldHandleRepositorySaveWithIdGeneration() {
    UserRepository userRepository = mock(UserRepository.class);
    
    // Simulate auto-increment ID generation
    when(userRepository.save(any(User.class)))
        .thenAnswer(invocation -> {
            User user = invocation.getArgument(0);
            if (user.getId() == null) {
                user.setId(ThreadLocalRandom.current().nextLong(1, 1000));
            }
            return user;
        });
    
    User user1 = userRepository.save(new User(null, "email1@example.com", "User 1"));
    User user2 = userRepository.save(new User(null, "email2@example.com", "User 2"));
    
    assertNotNull(user1.getId());
    assertNotNull(user2.getId());
    assertNotEquals(user1.getId(), user2.getId());
}
```

### Capturing Arguments

Use `ArgumentCaptor` to verify arguments passed to mocked methods.

```java
@Test
@DisplayName("Should send email with correct content")
void shouldSendEmailWithCorrectContent() {
    EmailService emailService = mock(EmailService.class);
    UserService userService = new UserService(userRepository, emailService);
    
    userService.sendWelcomeEmail("test@example.com", "John Doe");
    
    ArgumentCaptor<Email> emailCaptor = ArgumentCaptor.forClass(Email.class);
    verify(emailService).sendEmail(emailCaptor.capture());
    
    Email capturedEmail = emailCaptor.getValue();
    assertThat(capturedEmail.getTo()).isEqualTo("test@example.com");
    assertThat(capturedEmail.getSubject()).contains("Welcome");
    assertThat(capturedEmail.getBody()).contains("John Doe");
}
```

### Spies for Partial Mocking

Use spies when you need to mock some methods but execute others normally.

```java
@Test
@DisplayName("Should use spy for partial mocking")
void shouldUseSpyForPartialMocking() {
    UserService realService = new UserService(userRepository);
    UserService spyService = spy(realService);
    
    // Mock only the email sending, keep other behavior real
    doNothing().when(spyService).sendWelcomeEmail(anyString(), anyString());
    
    spyService.createUser("test@example.com", "John Doe");
    
    verify(spyService).sendWelcomeEmail("test@example.com", "John Doe");
    // Other methods execute normally
}
```

### Verification in Order

Verify method calls in specific order when testing complex interactions.

```java
@Test
@DisplayName("Should verify method call order")
void shouldVerifyMethodCallOrder() {
    OrderService orderService = mock(OrderService.class);
    PaymentService paymentService = mock(PaymentService.class);
    NotificationService notificationService = mock(NotificationService.class);
    
    OrderProcessor processor = new OrderProcessor(
        orderService, paymentService, notificationService
    );
    
    processor.processOrder(new Order(1L, 100.0));
    
    InOrder inOrder = inOrder(orderService, paymentService, notificationService);
    inOrder.verify(orderService).validateOrder(any());
    inOrder.verify(paymentService).processPayment(any());
    inOrder.verify(notificationService).sendConfirmation(any());
}
```

## 🏗️ Test Builders Pattern

Use test builders to create complex test objects with readable, fluent APIs.

```java
public class UserBuilder {
    private Long id = 1L;
    private String email = "test@example.com";
    private String name = "Test User";
    private LocalDateTime createdAt = LocalDateTime.now();
    private boolean active = true;
    
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
    
    public UserBuilder withCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
        return this;
    }
    
    public UserBuilder withActive(boolean active) {
        this.active = active;
        return this;
    }
    
    public User build() {
        return new User(id, email, name, createdAt, active);
    }
}

// Usage in tests
@Test
@DisplayName("Should process active user")
void shouldProcessActiveUser() {
    User activeUser = UserBuilder.aUser()
        .withEmail("active@example.com")
        .withActive(true)
        .build();
    
    boolean result = userService.processUser(activeUser);
    
    assertTrue(result);
}
```

## 📊 Arrange-Act-Assert Pattern

Follow the AAA pattern for clear, readable tests.

```java
@Test
@DisplayName("Should calculate discount for premium customer")
void shouldCalculateDiscountForPremiumCustomer() {
    // Arrange
    Customer customer = CustomerBuilder.aCustomer()
        .withType(CustomerType.PREMIUM)
        .withLoyaltyPoints(1000)
        .build();
    
    Order order = OrderBuilder.anOrder()
        .withAmount(100.0)
        .withCustomer(customer)
        .build();
    
    DiscountCalculator calculator = new DiscountCalculator();
    
    // Act
    BigDecimal discount = calculator.calculateDiscount(order);
    
    // Assert
    assertThat(discount).isEqualByComparingTo("15.00"); // 10% premium + 5% loyalty
}
```

## 🎯 Test Naming Conventions

Use descriptive test names that explain the behavior being tested.

```java
// Good: Descriptive and clear
@Test
@DisplayName("Should return 404 when user ID does not exist")
void shouldReturn404WhenUserIdDoesNotExist() { }

@Test
@DisplayName("Should throw exception when email is invalid")
void shouldThrowExceptionWhenEmailIsInvalid() { }

@Test
@DisplayName("Should apply discount when order amount exceeds threshold")
void shouldApplyDiscountWhenOrderAmountExceedsThreshold() { }

// Bad: Vague and unhelpful
@Test
void testUser() { }
@Test
void testMethod() { }
@Test
void test1() { }
```

## 🔄 Test Lifecycle Management

Use lifecycle annotations for proper setup and teardown.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceLifecycleTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    private UserService userService;
    
    @BeforeAll
    static void beforeAll() {
        // Runs once before all tests - useful for expensive setup
        System.out.println("Initializing test suite");
    }
    
    @BeforeEach
    void setUp() {
        // Runs before each test
        userService = new UserService(userRepository, emailService);
    }
    
    @AfterEach
    void tearDown() {
        // Runs after each test - cleanup
        reset(userRepository, emailService);
    }
    
    @AfterAll
    static void afterAll() {
        // Runs once after all tests - cleanup resources
        System.out.println("Cleaning up test suite");
    }
    
    @Test
    void testOne() { }
    
    @Test
    void testTwo() { }
}
```

## ⚠️ Common Unit Testing Pitfalls

### 1. Testing Implementation Details

```java
// Bad: Tests internal implementation
@Test
void shouldCallRepositoryThreeTimes() {
    userService.processUsers();
    verify(userRepository, times(3)).save(any());
}

// Good: Tests observable behavior
@Test
void shouldProcessAllUsersSuccessfully() {
    List<User> users = userService.processUsers();
    assertThat(users).allMatch(User::isActive);
}
```

### 2. Over-Mocking

```java
// Bad: Mocking too much leads to brittle tests
@Test
void shouldCreateUser() {
    when(userRepository.save(any())).thenReturn(user);
    when(emailService.sendEmail(any())).thenReturn(true);
    when(auditService.logEvent(any())).thenReturn(true);
    when(cacheService.invalidate(any())).thenReturn(true);
    
    userService.createUser(request);
}

// Good: Use real implementations where possible
@Test
void shouldCreateUser() {
    when(userRepository.save(any())).thenReturn(user);
    // emailService, auditService, cacheService are real implementations
    userService.createUser(request);
}
```

### 3. Testing Private Methods

```java
// Bad: Using reflection to test private methods
@Test
void testPrivateMethod() throws Exception {
    Method method = UserService.class.getDeclaredMethod("privateMethod");
    method.setAccessible(true);
    method.invoke(userService);
}

// Good: Test public behavior that exercises the private method
@Test
void shouldBehaveCorrectly() {
    // Test the public method that uses the private method
    userService.publicMethod();
    assertThat(result).isExpected();
}
```

## 🎓 Interview Discussion Points

When discussing unit testing in interviews, emphasize:

1. **Test Quality over Quantity**: Explain why you prioritize meaningful tests over high coverage numbers
2. **Testing Strategy**: Describe how you decide what to test at the unit level vs integration level
3. **Mocking Philosophy**: Discuss when to mock vs use real dependencies
4. **Test Maintenance**: Share strategies for keeping tests maintainable as code evolves
5. **Flaky Tests**: Explain how you identify and eliminate flaky tests
6. **TDD Experience**: Discuss your experience with test-driven development

## 📚 Related Topics

- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Test-Driven Development](./test-driven-development.md)
- [Property-Based Testing](./property-based-testing.md)
- [Test Data Management](./test-data-management.md)

---

**Key Takeaway**: Good unit tests are an investment in code quality and maintainability. They should be treated with the same care as production code, following clean code principles and best practices.
