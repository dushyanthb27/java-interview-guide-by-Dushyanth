# Testing - Cheat Sheet

Complete reference for JUnit 5, Mockito, and testing best practices.

## 🧪 JUnit 5 Basics

### Test Annotations
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.AfterAll;

public class CalculatorTest {
    
    private Calculator calculator;
    
    // Runs once before all tests
    @BeforeAll
    static void setupAll() {
        System.out.println("Setup all tests");
    }
    
    // Runs before each test
    @BeforeEach
    void setup() {
        calculator = new Calculator();
    }
    
    // Runs after each test
    @AfterEach
    void tearDown() {
        calculator = null;
    }
    
    // Runs once after all tests
    @AfterAll
    static void tearDownAll() {
        System.out.println("Cleanup all tests");
    }
    
    // Test method
    @Test
    void testAddition() {
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }
}
```

### Test Lifecycle
```
@BeforeAll (static, once)
    ↓
@BeforeEach (before each test)
    ↓
@Test (test method)
    ↓
@AfterEach (after each test)
    ↓
@AfterAll (static, once)
```

### Assertions
```java
import static org.junit.jupiter.api.Assertions.*;

// Equality
assertEquals(5, calculator.add(2, 3));
assertNotEquals(6, calculator.add(2, 3));

// Boolean
assertTrue(value > 0);
assertFalse(value < 0);

// Null checks
assertNull(value);
assertNotNull(value);

// Same object reference
assertSame(obj1, obj2);
assertNotSame(obj1, obj2);

// Collections
assertIterableEquals(expected, actual);

// Exceptions
assertThrows(IllegalArgumentException.class, () -> {
    calculator.divide(10, 0);
});

// Multiple assertions
assertAll(
    () -> assertEquals(5, calculator.add(2, 3)),
    () -> assertEquals(1, calculator.subtract(3, 2)),
    () -> assertEquals(6, calculator.multiply(2, 3))
);

// Timeout
assertTimeout(Duration.ofMillis(100), () -> {
    // Code that should complete within 100ms
});

// Custom messages
assertEquals(5, result, "Addition failed");
assertEquals(5, result, () -> "Addition failed: " + result);
```

## 🎯 Test Organization

### Naming Convention
```java
// Pattern: test[UnitOfWork]_[StateUnderTest]_[ExpectedBehavior]

public class CalculatorTest {
    
    @Test
    void testAdd_PositiveNumbers_ReturnsSum() {
        assertEquals(5, calculator.add(2, 3));
    }
    
    @Test
    void testAdd_NegativeNumbers_ReturnsSum() {
        assertEquals(-5, calculator.add(-2, -3));
    }
    
    @Test
    void testDivide_DivideByZero_ThrowsException() {
        assertThrows(IllegalArgumentException.class, 
            () -> calculator.divide(10, 0));
    }
}
```

### Nested Test Classes
```java
@DisplayName("Calculator Tests")
class CalculatorTest {
    
    @Nested
    @DisplayName("Addition Tests")
    class AdditionTests {
        
        @Test
        @DisplayName("Adding two positive numbers")
        void testAddPositive() {
            assertEquals(5, calculator.add(2, 3));
        }
    }
    
    @Nested
    @DisplayName("Division Tests")
    class DivisionTests {
        
        @Test
        @DisplayName("Dividing by zero throws exception")
        void testDivideByZero() {
            assertThrows(IllegalArgumentException.class,
                () -> calculator.divide(10, 0));
        }
    }
}
```

### Display Names
```java
@DisplayName("Calculator Tests")
class CalculatorTest {
    
    @Test
    @DisplayName("Should add two positive numbers correctly")
    void testAddition() {
        assertEquals(5, calculator.add(2, 3));
    }
}
```

## 📦 Parameterized Tests

### Value Source
```java
@ParameterizedTest
@ValueSource(ints = { 1, 3, 5, -3, 15, Integer.MAX_VALUE })
void testIsOdd(int number) {
    assertTrue(isOdd(number));
}

@ParameterizedTest
@ValueSource(strings = { "hello", "world", "test" })
void testStringLength(String str) {
    assertTrue(str.length() > 0);
}
```

### CSV Source
```java
@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, 20, 30"
})
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

### Method Source
```java
@ParameterizedTest
@MethodSource("provideNumbers")
void testIsPositive(int number) {
    assertTrue(number > 0);
}

static Stream<Arguments> provideNumbers() {
    return Stream.of(
        Arguments.of(1),
        Arguments.of(2),
        Arguments.of(3)
    );
}
```

### CSV File Source
```java
@ParameterizedTest
@CsvFileSource(resources = "/test-data.csv")
void testFromCsv(String name, int age) {
    assertNotNull(name);
    assertTrue(age > 0);
}
```

## 🎭 Mocking with Mockito

### Creating Mocks
```java
import static org.mockito.Mockito.*;

// Create mock
UserRepository userRepo = mock(UserRepository.class);

// Or with annotation
@Mock
UserRepository userRepo;

// Initialize mocks
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    UserRepository userRepo;
    
    @Test
    void testGetUser() {
        // Use userRepo
    }
}
```

### Stubbing
```java
// Return value
when(userRepo.findById(1)).thenReturn(new User(1, "John"));

// Return different values on consecutive calls
when(userRepo.findById(1))
    .thenReturn(new User(1, "John"))
    .thenReturn(new User(1, "Jane"));

// Return value based on argument
when(userRepo.findById(anyInt())).thenReturn(new User(1, "John"));

// Throw exception
when(userRepo.findById(999))
    .thenThrow(new UserNotFoundException("Not found"));

// Call real method
when(userRepo.findById(1)).thenCallRealMethod();

// Answer with custom logic
when(userRepo.findById(anyInt()))
    .thenAnswer(invocation -> {
        int id = invocation.getArgument(0);
        return new User(id, "User " + id);
    });
```

### Verification
```java
// Verify method was called
verify(userRepo).findById(1);

// Verify called with specific argument
verify(userRepo).findById(eq(1));

// Verify called multiple times
verify(userRepo, times(2)).findById(1);
verify(userRepo, atLeast(1)).findById(1);
verify(userRepo, atMost(3)).findById(1);

// Verify never called
verify(userRepo, never()).delete(any());

// Verify call order
InOrder inOrder = inOrder(userRepo, userService);
inOrder.verify(userRepo).findById(1);
inOrder.verify(userService).processUser(any());

// Verify no more interactions
verifyNoMoreInteractions(userRepo);
```

### Argument Matchers
```java
// Exact match
when(userRepo.findById(1)).thenReturn(user);

// Any value
when(userRepo.findById(anyInt())).thenReturn(user);

// Any string
when(userRepo.findByName(anyString())).thenReturn(user);

// Custom matcher
when(userRepo.findById(argThat(id -> id > 0))).thenReturn(user);

// Argument captor
ArgumentCaptor<Integer> captor = ArgumentCaptor.forClass(Integer.class);
verify(userRepo).findById(captor.capture());
assertEquals(1, captor.getValue());
```

### Spies
```java
// Spy on real object
UserRepository realRepo = new UserRepository();
UserRepository spy = spy(realRepo);

// Stub specific method
when(spy.findById(1)).thenReturn(new User(1, "John"));

// Other methods call real implementation
User user = spy.findById(2);  // Calls real method

// Verify interactions
verify(spy).findById(1);
```

## 🏗️ Test Structure (AAA Pattern)

```java
@Test
void testUserCreation() {
    // Arrange - Setup test data
    String name = "John";
    int age = 30;
    
    // Act - Execute the code being tested
    User user = new User(name, age);
    
    // Assert - Verify the results
    assertEquals(name, user.getName());
    assertEquals(age, user.getAge());
}
```

## 🔄 Integration Testing

### Spring Boot Test
```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

@SpringBootTest
class UserServiceIntegrationTest {
    
    @Autowired
    UserService userService;
    
    @Autowired
    UserRepository userRepository;
    
    @Test
    void testCreateAndRetrieveUser() {
        User user = new User("John", 30);
        userService.createUser(user);
        
        User retrieved = userRepository.findByName("John");
        assertEquals("John", retrieved.getName());
    }
}
```

### Mocking Spring Beans
```java
@SpringBootTest
class UserControllerTest {
    
    @MockBean
    UserService userService;
    
    @Autowired
    MockMvc mockMvc;
    
    @Test
    void testGetUser() throws Exception {
        User user = new User(1, "John");
        when(userService.getUser(1)).thenReturn(user);
        
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

## 🌐 REST API Testing

### MockMvc
```java
@SpringBootTest
class UserControllerTest {
    
    @Autowired
    MockMvc mockMvc;
    
    @Test
    void testGetUser() throws Exception {
        mockMvc.perform(get("/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John"));
    }
    
    @Test
    void testCreateUser() throws Exception {
        String json = "{\"name\": \"John\", \"age\": 30}";
        
        mockMvc.perform(post("/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists());
    }
    
    @Test
    void testUpdateUser() throws Exception {
        String json = "{\"name\": \"Jane\", \"age\": 25}";
        
        mockMvc.perform(put("/users/1")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
            .andExpect(status().isOk());
    }
    
    @Test
    void testDeleteUser() throws Exception {
        mockMvc.perform(delete("/users/1"))
            .andExpect(status().isNoContent());
    }
}
```

## 📋 Test Coverage

### JaCoCo Configuration
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## ⚠️ Common Testing Pitfalls

1. **Testing implementation details**
   ```java
   // ❌ Wrong - testing internal state
   @Test
   void testInternalState() {
       calculator.add(2, 3);
       assertEquals(5, calculator.lastResult);  // Implementation detail
   }
   
   // ✅ Correct - testing behavior
   @Test
   void testAddition() {
       assertEquals(5, calculator.add(2, 3));
   }
   ```

2. **Not isolating tests**
   ```java
   // ❌ Wrong - tests depend on each other
   private static List<String> items = new ArrayList<>();
   
   @Test
   void testAddItem() {
       items.add("item1");
   }
   
   @Test
   void testItemCount() {
       assertEquals(1, items.size());  // Depends on testAddItem
   }
   
   // ✅ Correct - independent tests
   @Test
   void testAddItem() {
       List<String> items = new ArrayList<>();
       items.add("item1");
       assertEquals(1, items.size());
   }
   ```

3. **Over-mocking**
   ```java
   // ❌ Wrong - mocking too much
   @Test
   void testUserCreation() {
       User user = mock(User.class);
       when(user.getName()).thenReturn("John");
       assertEquals("John", user.getName());  // Testing mock, not code
   }
   
   // ✅ Correct - test real objects
   @Test
   void testUserCreation() {
       User user = new User("John", 30);
       assertEquals("John", user.getName());
   }
   ```

4. **Ignoring test failures**
   ```java
   // ❌ Wrong - disabled test
   @Disabled
   @Test
   void testImportantFeature() {
       // Test that should be fixed
   }
   
   // ✅ Correct - fix the test
   @Test
   void testImportantFeature() {
       // Fixed test
   }
   ```

## 🎓 Best Practices

1. **Test one thing per test**
   ```java
   // ✅ Good - single assertion focus
   @Test
   void testAddPositiveNumbers() {
       assertEquals(5, calculator.add(2, 3));
   }
   ```

2. **Use descriptive names**
   ```java
   // ✅ Good - clear what is being tested
   @Test
   void shouldThrowExceptionWhenDividingByZero() {
       assertThrows(IllegalArgumentException.class,
           () -> calculator.divide(10, 0));
   }
   ```

3. **Keep tests simple**
   ```java
   // ✅ Good - straightforward test
   @Test
   void testAddition() {
       assertEquals(5, calculator.add(2, 3));
   }
   ```

4. **Use test fixtures**
   ```java
   // ✅ Good - reusable setup
   @BeforeEach
   void setup() {
       calculator = new Calculator();
   }
   ```

## 📊 Test Types

| Type | Scope | Speed | Cost |
|------|-------|-------|------|
| Unit | Single class | Fast | Low |
| Integration | Multiple classes | Medium | Medium |
| E2E | Entire system | Slow | High |
| Performance | System under load | Slow | High |

---

**Related**: See Exception Handling cheat sheet for testing exception scenarios
