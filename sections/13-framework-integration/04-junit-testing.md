# Q4: JUnit 5 Testing Essentials 🟡

[Home](../../README.md) > [Framework Integration](./README.md) > JUnit Testing

---

## Overview

**JUnit 5** is the next generation testing framework for Java. It is composed of three sub-projects: **JUnit Platform** (foundation), **JUnit Jupiter** (new programming model), and **JUnit Vintage** (backward compatibility with JUnit 3/4).

```
┌──────────────────────────────────────┐
│          JUnit Platform              │  ← launcher, engine API
├──────────────┬───────────────────────┤
│ JUnit Jupiter │    JUnit Vintage     │  ← JUnit 5 API  │  JUnit 3/4 compat
└──────────────┴───────────────────────┘
```

---

## Basic Test Structure

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private Calculator calculator;

    @BeforeAll
    static void setupAll() {
        // Runs ONCE before all tests (must be static)
        System.out.println("Starting Calculator tests");
    }

    @BeforeEach
    void setup() {
        // Runs before EACH test
        calculator = new Calculator();
    }

    @Test
    @DisplayName("should add two positive numbers correctly")
    void testAddition() {
        assertEquals(5, calculator.add(2, 3));
    }

    @Test
    @DisplayName("should handle division by zero")
    void testDivisionByZero() {
        assertThrows(ArithmeticException.class,
            () -> calculator.divide(10, 0));
    }

    @AfterEach
    void tearDown() {
        // Runs after EACH test
        calculator = null;
    }

    @AfterAll
    static void tearDownAll() {
        // Runs ONCE after all tests (must be static)
        System.out.println("All Calculator tests completed");
    }
}
```

---

## Lifecycle Order

```
@BeforeAll       (once)
  ├── @BeforeEach  (before test 1)
  │     └── @Test  (test 1)
  │     └── @AfterEach  (after test 1)
  ├── @BeforeEach  (before test 2)
  │     └── @Test  (test 2)
  │     └── @AfterEach  (after test 2)
@AfterAll        (once)
```

---

## Assertions

```java
@Test
void demonstrateAssertions() {
    // Basic assertions
    assertEquals(4, calculator.add(2, 2));
    assertNotEquals(5, calculator.add(2, 2));
    assertTrue(calculator.isPositive(5));
    assertFalse(calculator.isPositive(-1));
    assertNull(calculator.getLastError());
    assertNotNull(calculator.getResult());

    // ✅ Always provide a failure message
    assertEquals(4, calculator.add(2, 2),
        "2 + 2 should equal 4");

    // ✅ Lazy message (computed only on failure — better performance)
    assertEquals(4, calculator.add(2, 2),
        () -> "Expected 4 but got " + calculator.add(2, 2));
}
```

### assertAll — Grouped Assertions

```java
@Test
@DisplayName("should create user with all fields populated")
void testUserCreation() {
    User user = userService.createUser("Alice", "alice@test.com");

    // ✅ All assertions run even if one fails — see ALL failures at once
    assertAll("user",
        () -> assertNotNull(user.getId()),
        () -> assertEquals("Alice", user.getName()),
        () -> assertEquals("alice@test.com", user.getEmail()),
        () -> assertNotNull(user.getCreatedAt())
    );
}
```

```java
// ❌ Anti-pattern: separate assertions — first failure stops execution
@Test
void testUserBad() {
    User user = userService.createUser("Alice", "alice@test.com");
    assertNotNull(user.getId());           // if this fails...
    assertEquals("Alice", user.getName()); // ...this never runs
}
```

### assertThrows

```java
@Test
void shouldThrowOnInvalidInput() {
    // ✅ Captures the exception for further assertions
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> userService.createUser(null, "test@test.com")
    );

    assertEquals("Name cannot be null", ex.getMessage());
}
```

---

## @ParameterizedTest

```java
// ✅ Run the same test with different inputs
@ParameterizedTest
@ValueSource(ints = {1, 3, 5, 7, 9})
@DisplayName("should identify odd numbers")
void testIsOdd(int number) {
    assertTrue(MathUtils.isOdd(number));
}

@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"  ", "\t", "\n"})
@DisplayName("should reject blank strings")
void testRejectBlank(String input) {
    assertThrows(IllegalArgumentException.class,
        () -> validator.validateName(input));
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, -5, 5",
    "0, 0, 0"
})
@DisplayName("should add two numbers correctly")
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideStringsForPalindrome")
void testIsPalindrome(String input, boolean expected) {
    assertEquals(expected, StringUtils.isPalindrome(input));
}

static Stream<Arguments> provideStringsForPalindrome() {
    return Stream.of(
        Arguments.of("racecar", true),
        Arguments.of("hello", false),
        Arguments.of("madam", true)
    );
}
```

---

## @Nested — Organized Test Structure

```java
@DisplayName("UserService")
class UserServiceTest {

    private UserService service;

    @BeforeEach
    void setup() {
        service = new UserService(mock(UserRepository.class));
    }

    @Nested
    @DisplayName("when creating a user")
    class CreateUser {

        @Test
        @DisplayName("should succeed with valid input")
        void validInput() {
            User user = service.createUser("Alice", "alice@test.com");
            assertNotNull(user);
        }

        @Test
        @DisplayName("should fail with null name")
        void nullName() {
            assertThrows(IllegalArgumentException.class,
                () -> service.createUser(null, "alice@test.com"));
        }
    }

    @Nested
    @DisplayName("when deleting a user")
    class DeleteUser {

        @Test
        @DisplayName("should remove existing user")
        void existingUser() {
            assertTrue(service.deleteUser(1L));
        }

        @Test
        @DisplayName("should throw for non-existent user")
        void nonExistentUser() {
            assertThrows(UserNotFoundException.class,
                () -> service.deleteUser(999L));
        }
    }
}
```

Output:
```
UserService
├── when creating a user
│   ├── ✓ should succeed with valid input
│   └── ✓ should fail with null name
└── when deleting a user
    ├── ✓ should remove existing user
    └── ✓ should throw for non-existent user
```

---

## Assumptions

```java
@Test
void testOnlyOnLinux() {
    // Skip (not fail) if condition isn't met
    assumeTrue(System.getProperty("os.name").contains("Linux"),
        "Test only runs on Linux");

    // test logic here
}

@Test
void testWithDatabaseConnection() {
    assumingThat(isDatabaseAvailable(), () -> {
        // Only runs if assumption holds; test still passes otherwise
        assertEquals(5, repository.count());
    });
}
```

---

## Extensions (Replaces JUnit 4 @Rule/@RunWith)

```java
// ✅ Custom extension for timing tests
public class TimingExtension implements BeforeTestExecutionCallback,
                                        AfterTestExecutionCallback {
    private long startTime;

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        startTime = System.currentTimeMillis();
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        long duration = System.currentTimeMillis() - startTime;
        System.out.println(context.getDisplayName() + " took " + duration + "ms");
    }
}

@ExtendWith(TimingExtension.class)
class PerformanceTest {
    @Test
    void shouldRunFast() { ... }
}
```

```java
// ✅ Spring Boot integration
@ExtendWith(SpringExtension.class)      // or simply use @SpringBootTest
@SpringBootTest
class ApplicationIntegrationTest { ... }

// ✅ Mockito integration
@ExtendWith(MockitoExtension.class)
class ServiceTest { ... }
```

---

## Anti-Patterns vs Best Practices

```java
// ❌ Anti-pattern: test method names like test1, test2
@Test
void test1() { }

// ✅ Best practice: descriptive names
@Test
@DisplayName("should return empty list when no users exist")
void shouldReturnEmptyListWhenNoUsersExist() { }

// ❌ Anti-pattern: multiple unrelated assertions without grouping
@Test
void testEverything() {
    // tests creation, deletion, update in one method
}

// ✅ Best practice: one logical concept per test
@Test
void shouldCreateUser() { }

@Test
void shouldDeleteUser() { }

// ❌ Anti-pattern: tests depend on execution order
// ✅ Best practice: each test is independent and idempotent
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| JUnit 5 vs 4 | Jupiter annotations (`@BeforeEach` vs `@Before`), Extensions vs Rules |
| @ParameterizedTest | Shows you write thorough, data-driven tests |
| assertAll | Groups assertions so all run even if one fails |
| @Nested | Organizes related tests — shows BDD-style thinking |
| Assumptions | Skip tests gracefully vs hard failure |

---

[← Previous: Q3 - Hibernate/JPA](./q03-hibernate-jpa.md) | [Next: Q5 - Mockito Basics →](./q05-mockito-basics.md)
