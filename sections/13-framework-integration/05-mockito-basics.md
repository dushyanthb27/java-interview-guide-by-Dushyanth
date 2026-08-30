# Q5: Mockito Fundamentals 🟡

[Home](../../README.md) > [Framework Integration](./README.md) > Mockito Basics

---

## Overview

**Mockito** is the most popular mocking framework for Java unit tests. It lets you create test doubles (mocks, stubs, spies) so you can test a class in complete isolation from its dependencies.

---

## Core Concepts

```
┌──────────────────────────────────────────────┐
│              Test Doubles                    │
├──────────┬──────────┬────────────────────────┤
│   Mock   │   Stub   │         Spy            │
│ Verifies │ Returns  │ Real object with       │
│ behavior │ canned   │ selective overrides     │
│          │ values   │                        │
└──────────┴──────────┴────────────────────────┘
```

---

## Creating Mocks

### Method 1: `mock()` — Manual

```java
import static org.mockito.Mockito.*;

class OrderServiceTest {

    @Test
    void shouldPlaceOrder() {
        // Create mocks manually
        OrderRepository mockRepo = mock(OrderRepository.class);
        NotificationService mockNotifier = mock(NotificationService.class);

        // Inject mocks via constructor
        OrderService service = new OrderService(mockRepo, mockNotifier);

        // Stub behavior
        when(mockRepo.save(any(Order.class)))
            .thenReturn(new Order(1L, "item-1", 2));

        // Act
        Order result = service.placeOrder("item-1", 2);

        // Assert
        assertNotNull(result.getId());

        // Verify interactions
        verify(mockRepo).save(any(Order.class));
        verify(mockNotifier).send(contains("Order placed"));
    }
}
```

### Method 2: Annotations (Preferred)

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository repository;

    @Mock
    private NotificationService notifier;

    @InjectMocks   // creates OrderService and injects the @Mocks above
    private OrderService service;

    @Test
    void shouldCalculateTotal() {
        when(repository.findById(1L))
            .thenReturn(Optional.of(new Order(1L, "item-1", 3)));

        BigDecimal total = service.calculateTotal(1L);

        assertEquals(new BigDecimal("29.97"), total);
    }
}
```

---

## Stubbing: `when().thenReturn()`

```java
// ✅ Return a value
when(userRepo.findById(1L)).thenReturn(Optional.of(user));

// ✅ Return different values on consecutive calls
when(idGenerator.next())
    .thenReturn(1L)
    .thenReturn(2L)
    .thenReturn(3L);

// ✅ Throw an exception
when(userRepo.findById(999L))
    .thenThrow(new UserNotFoundException("Not found"));

// ✅ Answer with dynamic logic
when(userRepo.save(any(User.class)))
    .thenAnswer(invocation -> {
        User u = invocation.getArgument(0);
        u.setId(42L);
        return u;
    });
```

### `doReturn` / `doThrow` — For Void Methods & Spies

```java
// ✅ Stubbing void methods
doNothing().when(emailService).sendWelcomeEmail(anyString());
doThrow(new MailException("SMTP down"))
    .when(emailService).sendWelcomeEmail("bad@test.com");

// ✅ Required for spies (avoids calling the real method during stubbing)
List<String> spyList = spy(new ArrayList<>());
doReturn(100).when(spyList).size();  // ✅ safe
// when(spyList.size()).thenReturn(100);  // ❌ calls real size() first!
```

---

## Verification

```java
@Test
void shouldVerifyInteractions() {
    service.registerUser("Alice", "alice@test.com");

    // ✅ Was this method called?
    verify(repository).save(any(User.class));

    // ✅ Called exactly N times?
    verify(emailService, times(1)).sendWelcomeEmail("alice@test.com");

    // ✅ Never called?
    verify(emailService, never()).sendWelcomeEmail("admin@test.com");

    // ✅ Called at least / at most N times?
    verify(repository, atLeastOnce()).save(any());
    verify(repository, atMost(3)).save(any());

    // ✅ No more interactions beyond what was verified
    verifyNoMoreInteractions(repository);

    // ✅ Zero interactions on a mock
    verifyNoInteractions(auditService);
}
```

---

## ArgumentCaptor

```java
@Test
void shouldCaptureArguments() {
    service.registerUser("Alice", "alice@test.com");

    // ✅ Capture what was actually passed to the mock
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    verify(repository).save(captor.capture());

    User savedUser = captor.getValue();
    assertEquals("Alice", savedUser.getName());
    assertEquals("alice@test.com", savedUser.getEmail());
    assertNotNull(savedUser.getCreatedAt());
}

// ✅ With annotation
@Captor
private ArgumentCaptor<User> userCaptor;

@Test
void shouldCaptureWithAnnotation() {
    service.registerUser("Bob", "bob@test.com");

    verify(repository).save(userCaptor.capture());
    assertEquals("Bob", userCaptor.getValue().getName());
}
```

---

## Argument Matchers

```java
// ✅ Built-in matchers
when(repo.findByName(anyString())).thenReturn(user);
when(repo.findById(anyLong())).thenReturn(Optional.of(user));
when(repo.findAll(any(Pageable.class))).thenReturn(page);

// ✅ Custom matcher with argThat
when(repo.save(argThat(user ->
    user.getName().equals("Alice") && user.getEmail().contains("@")
))).thenReturn(savedUser);

// ❌ Anti-pattern: mixing matchers and raw values
verify(service).process(eq("Alice"), anyInt());  // ✅ use eq() for literals
// verify(service).process("Alice", anyInt());   // ❌ compilation error
```

---

## Spy — Partial Mock

```java
@Test
void testSpy() {
    List<String> realList = new ArrayList<>();
    List<String> spyList = spy(realList);

    // Real methods are called by default
    spyList.add("one");
    spyList.add("two");
    assertEquals(2, spyList.size());   // real behavior

    // Override specific methods
    doReturn(100).when(spyList).size();
    assertEquals(100, spyList.size()); // stubbed behavior

    // Verify real method was called
    verify(spyList).add("one");
}
```

```java
// ✅ Spy use case: test a method that calls another method in the same class
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Spy
    @InjectMocks
    private PaymentService service;

    @Mock
    private PaymentGateway gateway;

    @Test
    void shouldApplyDiscount() {
        doReturn(new BigDecimal("0.10")).when(service).getDiscountRate(anyLong());

        BigDecimal total = service.calculateTotal(1L, new BigDecimal("100"));
        assertEquals(new BigDecimal("90.00"), total);
    }
}
```

---

## BDD Style — given/when/then

```java
import static org.mockito.BDDMockito.*;

@Test
void shouldProcessPayment_BDDStyle() {
    // Given
    given(paymentGateway.charge(any(BigDecimal.class)))
        .willReturn(new Receipt("txn-123"));

    // When
    Receipt receipt = paymentService.processPayment(new BigDecimal("50.00"));

    // Then
    then(paymentGateway).should().charge(new BigDecimal("50.00"));
    assertThat(receipt.getTransactionId()).isEqualTo("txn-123");
}
```

---

## Anti-Patterns vs Best Practices

```java
// ❌ Anti-pattern: mocking everything — test becomes meaningless
@Test
void overMocked() {
    when(mock.doA()).thenReturn(x);
    when(mock.doB()).thenReturn(y);
    when(mock.doC()).thenReturn(z);
    // You're testing your stubbing setup, not real code
}

// ✅ Best practice: only mock external dependencies
// Let the class under test run its real logic

// ❌ Anti-pattern: mocking value objects / DTOs
User mockUser = mock(User.class);
when(mockUser.getName()).thenReturn("Alice");
// ✅ Just create a real object
User user = new User("Alice", "alice@test.com");

// ❌ Anti-pattern: verifying getters (implementation detail)
verify(user).getName();
// ✅ Verify meaningful side effects
verify(repository).save(any(User.class));
verify(emailService).sendWelcomeEmail(anyString());
```

---

## Mockito vs Other Mocking Frameworks

| Feature | Mockito | EasyMock | PowerMock | JMockit |
|---------|---------|----------|-----------|---------|
| API style | Clean, fluent | Record/replay | Extends Mockito | Annotation-based |
| Static mocking | ✅ (3.4+) | ❌ | ✅ | ✅ |
| Final classes | ✅ (2.1+) | ❌ | ✅ | ✅ |
| Learning curve | Low | Medium | Medium | Medium |
| Popularity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| mock vs spy | Mock = all stubbed; Spy = real object with selective overrides |
| @Mock vs @InjectMocks | @Mock creates the mock; @InjectMocks creates the real class and injects mocks |
| doReturn vs when | Use `doReturn` for spies and void methods |
| ArgumentCaptor | Proves what data was actually passed to a dependency |
| BDD style | Shows test readability awareness — given/when/then maps to Arrange/Act/Assert |

---

[← Previous: Q4 - JUnit Testing](./q04-junit-testing.md) | [Next: Q6 - Maven & Gradle →](./q06-maven-gradle.md)
