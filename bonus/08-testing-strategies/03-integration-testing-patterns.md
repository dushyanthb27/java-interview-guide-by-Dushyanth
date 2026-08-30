# Integration Testing Patterns

## 📋 Overview

Integration testing verifies that different components or services work together correctly. This guide covers Spring Boot test slices, Testcontainers usage, database testing strategies, and patterns for writing effective integration tests that are reliable and maintainable.

## 🎯 When to Use Integration Tests

Integration tests are appropriate when you need to verify:
- **Database interactions**: JPA repositories, custom queries, transactions
- **External service integration**: REST APIs, message queues, caching
- **Spring configuration**: Bean wiring, profiles, property binding
- **Multiple component interactions**: Service + repository + external API
- **Infrastructure concerns**: Connection pooling, serialization, timeouts

## 🔧 Spring Boot Test Slices

Spring Boot provides test slice annotations that load only the relevant parts of the application context, making tests faster and more focused.

### @WebMvcTest for Controller Testing

```java
@WebMvcTest(UserController.class)
@DisplayName("UserController Integration Tests")
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    @DisplayName("Should return user by ID")
    void shouldReturnUserById() throws Exception {
        User user = User.builder()
            .id(1L)
            .email("test@example.com")
            .name("John Doe")
            .build();
            
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("test@example.com"))
            .andExpect(jsonPath("$.name").value("John Doe"));
    }
    
    @Test
    @DisplayName("Should return 404 when user not found")
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.findById(999L)).thenReturn(Optional.empty());
        
        mockMvc.perform(get("/api/users/999")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotFound());
    }
    
    @Test
    @DisplayName("Should create new user")
    void shouldCreateNewUser() throws Exception {
        CreateUserRequest request = CreateUserRequest.builder()
            .email("new@example.com")
            .name("New User")
            .build();
            
        User createdUser = User.builder()
            .id(1L)
            .email("new@example.com")
            .name("New User")
            .build();
            
        when(userService.createUser(any())).thenReturn(createdUser);
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("new@example.com"));
    }
}
```

### @DataJpaTest for Repository Testing

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
@DisplayName("UserRepository Integration Tests")
class UserRepositoryTest {
    
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @DynamicPropertySource
    static void postgresProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Test
    @DisplayName("Should find user by email")
    void shouldFindUserByEmail() {
        User user = User.builder()
            .email("test@example.com")
            .name("Test User")
            .build();
            
        entityManager.persist(user);
        entityManager.flush();
        
        Optional<User> found = userRepository.findByEmail("test@example.com");
        
        assertTrue(found.isPresent());
        assertThat(found.get().getEmail()).isEqualTo("test@example.com");
    }
    
    @Test
    @DisplayName("Should check if email exists")
    void shouldCheckIfEmailExists() {
        User user = User.builder()
            .email("existing@example.com")
            .name("Existing User")
            .build();
            
        entityManager.persist(user);
        entityManager.flush();
        
        assertTrue(userRepository.existsByEmail("existing@example.com"));
        assertFalse(userRepository.existsByEmail("nonexistent@example.com"));
    }
    
    @Test
    @DisplayName("Should handle unique constraint violation")
    void shouldHandleUniqueConstraintViolation() {
        User user1 = User.builder()
            .email("duplicate@example.com")
            .name("User 1")
            .build();
            
        User user2 = User.builder()
            .email("duplicate@example.com")
            .name("User 2")
            .build();
            
        entityManager.persist(user1);
        entityManager.flush();
        
        assertThatThrownBy(() -> {
            entityManager.persist(user2);
            entityManager.flush();
        }).isInstanceOf(PersistenceException.class)
          .hasCauseInstanceOf(ConstraintViolationException.class);
    }
}
```

### @JsonTest for JSON Serialization

```java
@JsonTest
@DisplayName("User JSON Serialization Tests")
class UserJsonTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    @DisplayName("Should serialize user correctly")
    void shouldSerializeUser() throws JsonProcessingException {
        User user = User.builder()
            .id(1L)
            .email("test@example.com")
            .name("John Doe")
            .build();
            
        JsonContent<User> jsonContent = json.write(user);
        
        assertThat(jsonContent).extractingJsonPathNumberValue("$.id")
            .isEqualTo(1);
        assertThat(jsonContent).extractingJsonPathStringValue("$.email")
            .isEqualTo("test@example.com");
    }
    
    @Test
    @DisplayName("Should deserialize user correctly")
    void shouldDeserializeUser() throws IOException {
        String jsonString = """
            {
                "id": 1,
                "email": "test@example.com",
                "name": "John Doe"
            }
            """;
            
        User user = json.parseObject(jsonString);
        
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getEmail()).isEqualTo("test@example.com");
    }
}
```

## 🐳 Testcontainers Integration

Testcontainers provides lightweight, throwaway instances of databases, message brokers, and other services for integration testing.

### PostgreSQL Container

```java
@SpringBootTest
@Testcontainers
@DisplayName("UserService Integration Tests with PostgreSQL")
class UserServiceIntegrationTest {
    
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("schema.sql");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should create and retrieve user")
    @Transactional
    void shouldCreateAndRetrieveUser() {
        CreateUserRequest request = CreateUserRequest.builder()
            .email("integration@example.com")
            .name("Integration Test User")
            .build();
            
        User created = userService.createUser(request);
        
        assertThat(created.getId()).isNotNull();
        
        Optional<User> found = userService.findById(created.getId());
        assertTrue(found.isPresent());
        assertThat(found.get().getEmail()).isEqualTo("integration@example.com");
    }
    
    @Test
    @DisplayName("Should handle transaction rollback on error")
    @Transactional
    void shouldHandleTransactionRollbackOnError() {
        User user1 = userService.createUser(
            CreateUserRequest.builder()
                .email("user1@example.com")
                .name("User 1")
                .build()
        );
        
        assertThatThrownBy(() -> {
            userService.createUser(
                CreateUserRequest.builder()
                    .email("user1@example.com") // Duplicate email
                    .name("User 2")
                    .build()
            );
        }).isInstanceOf(UserAlreadyExistsException.class);
        
        // Verify user1 was rolled back
        assertThat(userRepository.count()).isEqualTo(0);
    }
}
```

### Redis Container

```java
@SpringBootTest
@Testcontainers
@DisplayName("Cache Integration Tests with Redis")
class CacheIntegrationTest {
    
    @Container
    private static final GenericContainer<?> redis = 
        new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should cache user lookup")
    void shouldCacheUserLookup() {
        User user = userRepository.save(
            User.builder()
                .email("cached@example.com")
                .name("Cached User")
                .build()
        );
        
        // First call - hits database
        User result1 = userService.findById(user.getId()).orElseThrow();
        
        // Second call - should hit cache
        User result2 = userService.findById(user.getId()).orElseThrow();
        
        assertThat(result1).isEqualTo(result2);
    }
}
```

### Kafka Container

```java
@SpringBootTest
@Testcontainers
@DisplayName("Kafka Integration Tests")
class KafkaIntegrationTest {
    
    @Container
    private static final KafkaContainer kafka = 
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private KafkaConsumer<String, String> kafkaConsumer;
    
    @Test
    @DisplayName("Should produce and consume message")
    void shouldProduceAndConsumeMessage() throws Exception {
        String topic = "test-topic";
        String message = "Test message";
        
        kafkaTemplate.send(topic, message);
        
        CompletableFuture<String> consumed = new CompletableFuture<>();
        kafkaConsumer.subscribe(Collections.singletonList(topic));
        
        ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(5));
        
        assertThat(records.count()).isGreaterThan(0);
        assertThat(records.iterator().next().value()).isEqualTo(message);
    }
}
```

## 🗄️ Database Testing Strategies

### Flyway Migration Testing

```java
@SpringBootTest
@Testcontainers
@DisplayName("Database Migration Tests")
class DatabaseMigrationTest {
    
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private Flyway flyway;
    
    @Test
    @DisplayName("Should validate migrations")
    void shouldValidateMigrations() {
        // This will fail if migrations are invalid
        flyway.validate();
    }
    
    @Test
    @DisplayName("Should apply migrations successfully")
    void shouldApplyMigrationsSuccessfully() {
        MigrationInfoService info = flyway.info();
        
        assertThat(info.pending()).isEmpty();
        assertThat(info.applied()).isNotEmpty();
    }
}
```

### Transactional Test Rollback

```java
@SpringBootTest
@Transactional
@DisplayName("Transactional Integration Tests")
class TransactionalIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private UserService userService;
    
    @Test
    @DisplayName("Should rollback changes after test")
    void shouldRollbackChangesAfterTest() {
        long initialCount = userRepository.count();
        
        userRepository.save(
            User.builder()
                .email("rollback@example.com")
                .name("Rollback User")
                .build()
        );
        
        assertThat(userRepository.count()).isEqualTo(initialCount + 1);
    }
    
    // Transaction is rolled back automatically
    @Test
    @DisplayName("Should have clean state for each test")
    void shouldHaveCleanStateForEachTest() {
        long count = userRepository.count();
        // Count should be 0 due to rollback from previous test
        assertThat(count).isEqualTo(0);
    }
}
```

## 🌐 REST API Testing

### RestTemplate Integration Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@DisplayName("REST API Integration Tests")
class RestApiIntegrationTest {
    
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @LocalServerPort
    private int port;
    
    @Test
    @DisplayName("Should create user via REST API")
    void shouldCreateUserViaRestApi() {
        String url = "http://localhost:" + port + "/api/users";
        
        CreateUserRequest request = CreateUserRequest.builder()
            .email("api@example.com")
            .name("API User")
            .build();
            
        ResponseEntity<User> response = restTemplate.postForEntity(
            url, request, User.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getEmail()).isEqualTo("api@example.com");
    }
    
    @Test
    @DisplayName("Should handle validation errors")
    void shouldHandleValidationErrors() {
        String url = "http://localhost:" + port + "/api/users";
        
        CreateUserRequest request = CreateUserRequest.builder()
            .email("invalid-email")
            .name("")
            .build();
            
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            url, request, ErrorResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody().getErrors()).isNotEmpty();
    }
}
```

## 📊 Test Data Management

### Test Data Builders for Integration Tests

```java
public class TestDataBuilder {
    
    public static UserBuilder user() {
        return new UserBuilder();
    }
    
    public static OrderBuilder order() {
        return new OrderBuilder();
    }
    
    public static class UserBuilder {
        private String email = "test@example.com";
        private String name = "Test User";
        private boolean active = true;
        
        public UserBuilder withEmail(String email) {
            this.email = email;
            return this;
        }
        
        public UserBuilder withName(String name) {
            this.name = name;
            return this;
        }
        
        public UserBuilder withActive(boolean active) {
            this.active = active;
            return this;
        }
        
        public User build() {
            return User.builder()
                .email(email)
                .name(name)
                .active(active)
                .build();
        }
    }
}

// Usage in integration tests
@Test
void shouldProcessMultipleUsers() {
    List<User> users = List.of(
        TestDataBuilder.user().withEmail("user1@example.com").build(),
        TestDataBuilder.user().withEmail("user2@example.com").build(),
        TestDataBuilder.user().withEmail("user3@example.com").build()
    );
    
    users.forEach(userRepository::save);
    
    List<User> result = userService.getActiveUsers();
    assertThat(result).hasSize(3);
}
```

## ⚡ Performance Considerations

### Parallel Test Execution

```java
// Configure parallel test execution in junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.config.strategy = fixed
junit.jupiter.execution.parallel.config.fixed.parallelism = 4
```

### Test Context Caching

```java
@SpringBootTest
@ContextConfiguration(initializers = TestContainerInitializer.class)
class ContextCachingTest {
    
    // Spring caches the application context between tests
    // Use @DirtiesContext only when necessary
    @Test
    void test1() { }
    
    @Test
    void test2() { }
}
```

## ⚠️ Common Integration Testing Pitfalls

### 1. Shared State Between Tests

```java
// Bad: Tests depend on each other
@Test
void test1() {
    userRepository.save(user);
}

@Test
void test2() {
    // Assumes user exists from test1
    User user = userRepository.findById(1L).orElseThrow();
}

// Good: Each test is independent
@Test
void test1() {
    User user = userRepository.save(TestDataBuilder.user().build());
    // Test logic
}

@Test
void test2() {
    User user = userRepository.save(TestDataBuilder.user().build());
    // Test logic
}
```

### 2. Slow Tests Due to Full Context Loading

```java
// Bad: Loading full application context for simple tests
@SpringBootTest
class SimpleTest {
    // Loads entire application context
}

// Good: Use test slices
@WebMvcTest(Controller.class)
class ControllerTest {
    // Loads only web layer
}

@DataJpaTest
class RepositoryTest {
    // Loads only JPA layer
}
```

### 3. Hardcoded External Dependencies

```java
// Bad: Depends on external database
@Test
void testWithExternalDb() {
    // Connects to development database
}

// Good: Use Testcontainers
@Testcontainers
class TestWithContainer {
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine");
}
```

## 🎓 Interview Discussion Points

When discussing integration testing in interviews:

1. **Test Slice Strategy**: Explain how you choose between @WebMvcTest, @DataJpaTest, and @SpringBootTest
2. **Testcontainers Usage**: Describe how you use Testcontainers for reliable integration tests
3. **Transaction Management**: Discuss when to use @Transactional and when to commit
4. **Test Performance**: Share strategies for keeping integration tests fast
5. **Data Management**: Explain how you manage test data and ensure isolation
6. **CI/CD Integration**: Describe how integration tests fit into your pipeline

## 📚 Related Topics

- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Contract Testing](./contract-testing.md)
- [Test Data Management](./test-data-management.md)
- [Load Testing Strategies](./load-testing-strategies.md)

---

**Key Takeaway**: Integration tests provide confidence that components work together correctly. Use test slices and Testcontainers to keep them fast, reliable, and maintainable.
