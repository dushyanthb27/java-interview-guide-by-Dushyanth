# Contract Testing

## 📋 Overview

Contract testing ensures that services (consumers and providers) adhere to a shared contract, enabling independent development and deployment while maintaining compatibility. This guide covers consumer-driven contract testing using Pact and Spring Cloud Contract, patterns for contract evolution, and best practices for microservices architectures.

## 🎯 Why Contract Testing?

Contract testing addresses the challenges of distributed systems:
- **Independent Deployment**: Services can be deployed independently without breaking integrations
- **Fast Feedback**: Detect contract violations early in the development cycle
- **Documentation**: Contracts serve as living documentation of API expectations
- **Confidence**: Deploy with confidence that integrations will work in production
- **Reduced Integration Issues**: Catch mismatches before they reach production

## 🔧 Consumer-Driven Contract Testing

Consumer-driven contract testing (CDCT) allows consumers to define the contract they expect from providers. This approach puts the consumer in control, ensuring the provider delivers what's actually needed.

### Pact Framework Integration

#### Consumer Side Testing

```java
@ExtendWith(PactConsumerTestExt.class)
@Provider("user-service")
@PactTestFor(pactMethod = "userByIdContract")
@DisplayName("User Service Consumer Contract Tests")
class UserServiceConsumerTest {
    
    @Test
    @Pact(consumer = "order-service")
    RequestResponsePact userByIdContract(PactDslWithProvider builder) {
        return builder
            .given("User with ID 1 exists")
            .uponReceiving("GET request for user by ID")
            .path("/api/users/1")
            .method("GET")
            .willRespondWith()
            .status(200)
            .header("Content-Type", "application/json")
            .body(
                new PactDslJsonBody()
                    .integerType("id", 1)
                    .stringType("email", "test@example.com")
                    .stringType("name", "John Doe")
                    .booleanType("active", true)
                    .stringMatcher("createdAt", ".*") // Regex pattern
            )
            .toPact();
    }
    
    @Test
    void testUserById(MockServer mockServer) {
        // When: Consumer makes request to mock server
        ResponseEntity<User> response = new RestTemplate().exchange(
            mockServer.getUrl() + "/api/users/1",
            HttpMethod.GET,
            null,
            User.class
        );
        
        // Then: Verify response matches contract
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getId()).isEqualTo(1L);
        assertThat(response.getBody().getEmail()).isEqualTo("test@example.com");
    }
    
    @Test
    @Pact(consumer = "order-service")
    RequestResponsePact userNotFoundContract(PactDslWithProvider builder) {
        return builder
            .given("User with ID 999 does not exist")
            .uponReceiving("GET request for non-existent user")
            .path("/api/users/999")
            .method("GET")
            .willRespondWith()
            .status(404)
            .header("Content-Type", "application/json")
            .body(
                new PactDslJsonBody()
                    .stringType("error", "User not found")
                    .integerType("status", 404)
            )
            .toPact();
    }
}
```

#### Provider Side Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("user-service")
@PactBroker(url = "http://localhost:9292")
@DisplayName("User Service Provider Contract Tests")
class UserServiceProviderTest {
    
    @LocalServerPort
    private int port;
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.setTarget(
            new HttpTestTarget("localhost", port, "/")
        );
    }
    
    @State("User with ID 1 exists")
    void userWithId1Exists() {
        // Setup: Create test data for this state
        User user = User.builder()
            .id(1L)
            .email("test@example.com")
            .name("John Doe")
            .active(true)
            .createdAt(LocalDateTime.now())
            .build();
            
        userRepository.save(user);
    }
    
    @State("User with ID 999 does not exist")
    void userWithId999DoesNotExist() {
        // Setup: Ensure user doesn't exist
        userRepository.deleteById(999L);
    }
    
    @BeforeEach
    void setUp(PactVerificationContext context) {
        // Clean database before each test
        userRepository.deleteAll();
    }
}
```

### Spring Cloud Contract

#### Consumer Side (Stub Runner)

```java
@SpringBootTest
@AutoConfigureStubRunner(
    stubsMode = StubRunnerProperties.StubsMode.LOCAL,
    ids = "com.example:user-service:+:stubs:8080"
)
@DisplayName("Order Service Consumer Tests with Spring Cloud Contract")
class OrderServiceConsumerTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Test
    @DisplayName("Should create order with user from stub")
    void shouldCreateOrderWithUserFromStub() {
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId(1L)
            .items(List.of(
                OrderItem.builder()
                    .productId(101L)
                    .quantity(2)
                    .price(new BigDecimal("29.99"))
                    .build()
            ))
            .build();
            
        Order order = orderService.createOrder(request);
        
        assertThat(order.getId()).isNotNull();
        assertThat(order.getUserId()).isEqualTo(1L);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

#### Provider Side (Contract Definition)

```groovy
// src/test/resources/contracts/userById.groovy
org.springframework.cloud.contract.spec.Contract.make {
    description "Should return user by ID"
    request {
        method 'GET'
        url '/api/users/1'
        headers {
            header('Content-Type', 'application/json')
        }
    }
    response {
        status 200
        headers {
            header('Content-Type': value(
                server: 'application/json',
                client: 'application/json'
            ))
        }
        body(
            id: 1,
            email: 'test@example.com',
            name: 'John Doe',
            active: true,
            createdAt: $(consumer: '2023-01-01T00:00:00', server: regex('.*'))
        )
    }
}
```

```groovy
// src/test/resources/contracts/userNotFound.groovy
org.springframework.cloud.contract.spec.Contract.make {
    description "Should return 404 when user not found"
    request {
        method 'GET'
        url '/api/users/999'
    }
    response {
        status 404
        body(
            error: 'User not found',
            status: 404
        )
    }
}
```

#### Provider Side Test Generation

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@DisplayName("Generated Contract Verification Tests")
class ContractVerificationTest extends BaseTest {
    
    // Tests are automatically generated from contract definitions
    // Run with: mvn contract:test
}
```

## 🔄 Contract Evolution Strategies

### Versioning Contracts

```java
// Consumer side - versioned contract
@Pact(consumer = "order-service", version = "2.0.0")
RequestResponsePact userByIdContractV2(PactDslWithProvider builder) {
    return builder
        .given("User with ID 1 exists")
        .uponReceiving("GET request for user by ID")
        .path("/api/users/1")
        .method("GET")
        .willRespondWith()
        .status(200)
        .header("Content-Type", "application/json")
        .body(
            new PactDslJsonBody()
                .integerType("id", 1)
                .stringType("email", "test@example.com")
                .stringType("name", "John Doe")
                .booleanType("active", true)
                .stringMatcher("createdAt", ".*")
                .object("address") // New field in v2
                    .stringType("street", "123 Main St")
                    .stringType("city", "New York")
                    .stringType("zip", "10001")
                .closeObject()
        )
        .toPact();
}
```

### Backward Compatibility

```java
// Provider side - maintain backward compatibility
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        User user = userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
            
        UserResponse response = UserResponse.builder()
            .id(user.getId())
            .email(user.getEmail())
            .name(user.getName())
            .active(user.isActive())
            .createdAt(user.getCreatedAt())
            .address(user.getAddress()) // New field
            .build();
            
        return ResponseEntity.ok(response);
    }
}
```

### Contract Tags for Environments

```java
@SpringBootTest
@Provider("user-service")
@PactBroker(
    url = "http://localhost:9292",
    tags = {"prod", "staging"} // Only test contracts tagged with these
)
@DisplayName("Environment-specific Contract Tests")
class EnvironmentContractTest {
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port, "/"));
    }
}
```

## 📊 Contract Management

### Pact Broker Integration

```yaml
# application.yml
pactbroker:
  url: http://localhost:9292
  auth:
    username: pact
    password: pact
  tags:
    - ${spring.profiles.active}
```

```java
@SpringBootTest
@PactBroker(
    url = "${pactbroker.url}",
    authentication = @PactBrokerAuth(
        username = "${pactbroker.auth.username}",
        password = "${pactbroker.auth.password}"
    ),
    tags = "${pactbroker.tags}"
)
class PactBrokerIntegrationTest {
    // Contract verification against Pact Broker
}
```

### Contract Publishing

```java
@Test
@Pact(consumer = "order-service")
RequestResponsePact createOrderContract(PactDslWithProvider builder) {
    return builder
        .given("User with ID 1 exists")
        .uponReceiving("POST request to create order")
        .path("/api/orders")
        .method("POST")
        .body(
            new PactDslJsonBody()
                .integerType("userId", 1)
                .array("items")
                    .object()
                        .integerType("productId", 101)
                        .integerType("quantity", 2)
                        .decimalType("price", 29.99)
                    .closeObject()
                .closeArray()
        )
        .willRespondWith()
        .status(201)
        .header("Location", "http://localhost/api/orders/1")
        .body(
            new PactDslJsonBody()
                .integerType("id", 1)
                .integerType("userId", 1)
                .stringType("status", "PENDING")
                .decimalType("totalAmount", 59.98)
        )
        .toPact();
}
```

## 🎯 Contract Testing Patterns

### Provider State Management

```java
@SpringBootTest
@Provider("user-service")
class ProviderStateManagementTest {
    
    @State("User with ID 1 exists")
    Map<String, Object> userWithId1Exists() {
        // Create test data
        User user = User.builder()
            .id(1L)
            .email("test@example.com")
            .name("John Doe")
            .build();
        userRepository.save(user);
        
        // Return parameters that might be needed
        return Map.of("userId", 1);
    }
    
    @State("Multiple users exist")
    Map<String, Object> multipleUsersExist() {
        List<User> users = List.of(
            User.builder().id(1L).email("user1@example.com").name("User 1").build(),
            User.builder().id(2L).email("user2@example.com").name("User 2").build(),
            User.builder().id(3L).email("user3@example.com").name("User 3").build()
        );
        userRepository.saveAll(users);
        
        return Map.of("userCount", 3);
    }
    
    @State("Database is empty")
    void databaseIsEmpty() {
        userRepository.deleteAll();
    }
}
```

### Dynamic Contract Values

```java
@Pact(consumer = "order-service")
RequestResponsePact dynamicValuesContract(PactDslWithProvider builder) {
    return builder
        .given("Order exists")
        .uponReceiving("GET order by ID")
        .path("/api/orders/1")
        .method("GET")
        .willRespondWith()
        .status(200)
        .body(
            new PactDslJsonBody()
                .integerType("id", 1)
                .stringMatcher("orderNumber", "ORD-[0-9]+") // Regex pattern
                .decimalType("totalAmount", 100.00)
                .stringMatcher("status", "PENDING|PROCESSING|COMPLETED")
                .stringMatcher("createdAt", ".*") // Any timestamp
        )
        .toPact();
}
```

## ⚡ CI/CD Integration

### Consumer Pipeline

```yaml
# .github/workflows/consumer-contract-tests.yml
name: Consumer Contract Tests

on: [push, pull_request]

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          
      - name: Run Consumer Tests
        run: mvn test -Dtest=**/*ConsumerTest
        
      - name: Publish Contracts to Pact Broker
        run: mvn pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_USERNAME: ${{ secrets.PACT_BROKER_USERNAME }}
          PACT_BROKER_PASSWORD: ${{ secrets.PACT_BROKER_PASSWORD }}
```

### Provider Pipeline

```yaml
# .github/workflows/provider-contract-tests.yml
name: Provider Contract Tests

on: [push, pull_request]

jobs:
  provider-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          
      - name: Run Provider Contract Tests
        run: mvn pact:verify
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_USERNAME: ${{ secrets.PACT_BROKER_USERNAME }}
          PACT_BROKER_PASSWORD: ${{ secrets.PACT_BROKER_PASSWORD }}
```

## ⚠️ Common Contract Testing Pitfalls

### 1. Over-Specific Contracts

```java
// Bad: Too specific, brittle contract
.body(
    new PactDslJsonBody()
        .integerType("id", 1)
        .stringType("email", "test@example.com") // Exact match
        .stringType("name", "John Doe") // Exact match
)

// Good: Flexible contract with matchers
.body(
    new PactDslJsonBody()
        .integerType("id", 1)
        .stringType("email") // Any string
        .stringType("name") // Any string
        .stringMatcher("email", ".+@.+\\..+") // Email pattern
)
```

### 2. Ignoring Provider States

```java
// Bad: No provider state setup
@Pact(consumer = "order-service")
RequestResponsePact contract(PactDslWithProvider builder) {
    return builder
        .uponReceiving("GET request")
        .path("/api/users/1")
        // No given() clause
}

// Good: Clear provider state
@Pact(consumer = "order-service")
RequestResponsePact contract(PactDslWithProvider builder) {
    return builder
        .given("User with ID 1 exists")
        .uponReceiving("GET request")
        .path("/api/users/1")
}
```

### 3. Not Publishing Contracts

```java
// Bad: Contracts only run locally, not published
@Test
void localContractTest() {
    // Contract test that never gets published
}

// Good: Publish contracts to broker
@Pact(consumer = "order-service")
RequestResponsePact contract(PactDslWithProvider builder) {
    // Contract that gets published to Pact Broker
}
```

## 🎓 Interview Discussion Points

When discussing contract testing in interviews:

1. **Consumer-Driven vs Provider-Driven**: Explain the benefits of consumer-driven contracts
2. **Contract Evolution**: Describe how you handle breaking changes and versioning
3. **Pact vs Spring Cloud Contract**: Compare the two approaches and when to use each
4. **CI/CD Integration**: Explain how contract tests fit into your deployment pipeline
5. **Team Collaboration**: Discuss how contract testing improves team communication
6. **Testing Strategy**: Explain where contract testing fits in your overall testing pyramid

## 📚 Related Topics

- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Load Testing Strategies](./load-testing-strategies.md)

---

**Key Takeaway**: Contract testing enables independent service development while ensuring compatibility. Use consumer-driven contracts to define clear expectations and catch integration issues early in the development cycle.
