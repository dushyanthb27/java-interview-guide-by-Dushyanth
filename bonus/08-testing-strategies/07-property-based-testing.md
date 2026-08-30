# Property-Based Testing

## 📋 Overview

Property-based testing (PBT) is a testing approach where you specify properties that your code should satisfy, and the testing framework generates hundreds of random inputs to verify those properties. This guide covers the jqwik framework for Java, property definition patterns, generators and shrinking, and stateful property testing.

## 🎯 Why Property-Based Testing?

Traditional example-based tests check specific inputs and expected outputs. Property-based testing:
- **Discovers Edge Cases**: Finds bugs you wouldn't think to test
- **Generates Test Data**: Automatically creates diverse test inputs
- **Shrinks Failures**: Reduces failing inputs to minimal counterexamples
- **Documents Invariants**: Encodes business rules as properties
- **Increases Confidence**: Tests hundreds of scenarios automatically

## 🔧 jqwik Framework Setup

### Maven Configuration

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.4</version>
    <scope>test</scope>
</dependency>
```

### Gradle Configuration

```groovy
testImplementation 'net.jqwik:jqwik:1.8.4'
```

## 📝 Basic Property Tests

### Simple Property Example

```java
import net.jqwik.api.*;
import net.jqwik.api.constraints.*;

@Property
@DisplayName("String Reversal Properties")
class StringReversalProperties {
    
    @Property
    @DisplayName("Reversing a string twice returns the original")
    void doubleReversalReturnsOriginal(@ForAll String input) {
        String reversed = new StringBuilder(input).reverse().toString();
        String doubleReversed = new StringBuilder(reversed).reverse().toString();
        
        Assertions.assertEquals(input, doubleReversed);
    }
    
    @Property
    @DisplayName("Reversed string has same length")
    void reversedStringHasSameLength(@ForAll String input) {
        String reversed = new StringBuilder(input).reverse().toString();
        Assertions.assertEquals(input.length(), reversed.length());
    }
    
    @Property
    @DisplayName("Reversing empty string returns empty string")
    void reversingEmptyStringReturnsEmpty() {
        String empty = "";
        String reversed = new StringBuilder(empty).reverse().toString();
        Assertions.assertEquals("", reversed);
    }
}
```

### Numeric Properties

```java
@Property
@DisplayName("Mathematical Properties")
class MathProperties {
    
    @Property
    @DisplayName("Addition is commutative")
    void additionIsCommutative(
        @ForAll @IntRange(min = -1000, max = 1000) int a,
        @ForAll @IntRange(min = -1000, max = 1000) int b
    ) {
        Assertions.assertEquals(a + b, b + a);
    }
    
    @Property
    @DisplayName("Addition is associative")
    void additionIsAssociative(
        @ForAll @IntRange(min = -100, max = 100) int a,
        @ForAll @IntRange(min = -100, max = 100) int b,
        @ForAll @IntRange(min = -100, max = 100) int c
    ) {
        Assertions.assertEquals((a + b) + c, a + (b + c));
    }
    
    @Property
    @DisplayName("Multiplication distributes over addition")
    void multiplicationDistributesOverAddition(
        @ForAll @IntRange(min = -50, max = 50) int a,
        @ForAll @IntRange(min = -50, max = 50) int b,
        @ForAll @IntRange(min = -50, max = 50) int c
    ) {
        Assertions.assertEquals(a * (b + c), (a * b) + (a * c));
    }
    
    @Property
    @DisplayName("Absolute value is always non-negative")
    void absoluteValueIsNonNegative(@ForAll int value) {
        Assertions.assertTrue(Math.abs(value) >= 0);
    }
}
```

## 🎯 Custom Generators

### Domain-Specific Generators

```java
@Property
@DisplayName("Custom Generators")
class CustomGeneratorProperties {
    
    @Provide
    Arbitrary<User> validUsers() {
        Arbitrary<String> emails = Arbitraries.strings()
            .withCharRange('a', 'z')
            .ofMinLength(5)
            .ofMaxLength(20)
            .map(s -> s + "@example.com");
            
        Arbitrary<String> names = Arbitraries.strings()
            .withCharRange('A', 'Z')
            .ofMinLength(2)
            .ofMaxLength(30);
            
        return Combinators.combine(emails, names)
            .as((email, name) -> User.builder()
                .email(email)
                .name(name)
                .active(true)
                .build());
    }
    
    @Property
    @DisplayName("User email contains @ symbol")
    void userEmailContainsAtSymbol(@ForAll("validUsers") User user) {
        Assertions.assertTrue(user.getEmail().contains("@"));
    }
    
    @Property
    @DisplayName("User name is not empty")
    void userNameIsNotEmpty(@ForAll("validUsers") User user) {
        Assertions.assertFalse(user.getName().isEmpty());
    }
}
```

### Complex Object Generators

```java
@Provide
Arbitrary<Order> orders() {
    Arbitrary<Long> productIds = Arbitraries.longs()
        .between(1L, 1000L);
        
    Arbitrary<Integer> quantities = Arbitraries.integers()
        .between(1, 10);
        
    Arbitrary<BigDecimal> prices = Arbitraries.bigDecimals()
        .between(new BigDecimal("1.00"), new BigDecimal("1000.00"))
        .ofScale(2);
        
    Arbitrary<OrderItem> items = Combinators.combine(
        productIds, quantities, prices
    ).as((id, qty, price) -> OrderItem.builder()
        .productId(id)
        .quantity(qty)
        .price(price)
        .build());
        
    return Arbitraries.listsOf(items)
        .ofMinSize(1)
        .ofMaxSize(10)
        .map(itemList -> Order.builder()
            .items(itemList)
            .status(OrderStatus.PENDING)
            .build());
}
```

## 🔄 Shrinking Examples

### Automatic Shrinking

```java
@Property
@DisplayName("Shrinking Example")
class ShrinkingProperties {
    
    @Property(tries = 1000)
    @DisplayName("List should not contain duplicates after deduplication")
    void listShouldNotContainDuplicatesAfterDeduplication(
        @ForAll @Size(min = 1, max = 100) List<Integer> list
    ) {
        List<Integer> deduplicated = list.stream()
            .distinct()
            .collect(Collectors.toList());
            
        // Check if deduplicated list has no duplicates
        Set<Integer> set = new HashSet<>(deduplicated);
        Assertions.assertEquals(deduplicated.size(), set.size());
        
        // Property: deduplicated list should be <= original size
        Assertions.assertTrue(deduplicated.size() <= list.size());
    }
    
    @Property
    @DisplayName("String length after trimming should be <= original")
    void trimmedStringLengthIsLessOrEqual(@ForAll String input) {
        String trimmed = input.trim();
        Assertions.assertTrue(trimmed.length() <= input.length());
    }
}
```

### Custom Shrinking

```java
@Property
@DisplayName("Custom Shrinking")
class CustomShrinkingProperties {
    
    @Property
    @DisplayName("Division by zero should be handled")
    void divisionByZeroShouldBeHandled(
        @ForAll @IntRange(min = -100, max = 100) int numerator,
        @ForAll @IntRange(min = -100, max = 100) int denominator
    ) {
        if (denominator == 0) {
            Assertions.assertThrows(ArithmeticException.class, () -> {
                int result = numerator / denominator;
            });
        } else {
            int result = numerator / denominator;
            Assertions.assertEquals(numerator, result * denominator);
        }
    }
}
```

## 🎯 Stateful Property Testing

### State Machine Testing

```java
@Property
@DisplayName("Stateful Counter Testing")
class CounterStatefulTest {
    
    @Property
    @DisplayName("Counter operations maintain consistency")
    void counterOperationsMaintainConsistency(
        @ForAll("counterActions") List<Action> actions
    ) {
        Counter counter = new Counter();
        int expectedValue = 0;
        
        for (Action action : actions) {
            switch (action.type) {
                case INCREMENT:
                    counter.increment();
                    expectedValue++;
                    break;
                case DECREMENT:
                    counter.decrement();
                    expectedValue--;
                    break;
                case RESET:
                    counter.reset();
                    expectedValue = 0;
                    break;
            }
            
            Assertions.assertEquals(expectedValue, counter.getValue());
        }
    }
    
    @Provide
    Arbitrary<List<Action>> counterActions() {
        Arbitrary<ActionType> types = Arbitraries.of(ActionType.class);
        return Arbitraries.listsOf(types.map(Action::new))
            .ofMinSize(0)
            .ofMaxSize(50);
    }
    
    static class Action {
        final ActionType type;
        
        Action(ActionType type) {
            this.type = type;
        }
    }
    
    enum ActionType {
        INCREMENT, DECREMENT, RESET
    }
}
```

### Stack Stateful Testing

```java
@Property
@DisplayName("Stack Stateful Testing")
class StackStatefulTest {
    
    @Property
    @DisplayName("Stack LIFO property")
    void stackLifoProperty(@ForAll("stackActions") List<StackAction> actions) {
        Stack<Integer> stack = new Stack<>();
        List<Integer> pushedValues = new ArrayList<>();
        
        for (StackAction action : actions) {
            switch (action.type) {
                case PUSH:
                    stack.push(action.value);
                    pushedValues.add(action.value);
                    break;
                case POP:
                    if (!pushedValues.isEmpty()) {
                        Integer popped = stack.pop();
                        Integer expected = pushedValues.remove(pushedValues.size() - 1);
                        Assertions.assertEquals(expected, popped);
                    } else {
                        Assertions.assertTrue(stack.isEmpty());
                    }
                    break;
                case PEEK:
                    if (!pushedValues.isEmpty()) {
                        Integer peeked = stack.peek();
                        Integer expected = pushedValues.get(pushedValues.size() - 1);
                        Assertions.assertEquals(expected, peeked);
                    }
                    break;
            }
        }
    }
    
    @Provide
    Arbitrary<List<StackAction>> stackActions() {
        Arbitrary<Integer> values = Arbitraries.integers().between(0, 100);
        Arbitrary<StackActionType> types = Arbitraries.of(StackActionType.class);
        
        return Arbitraries.frequencyOf(
            Tuple.of(5, types.map(type -> new StackAction(type, 0))), // PEEK
            Tuple.of(3, values.map(v -> new StackAction(StackActionType.PUSH, v))), // PUSH
            Tuple.of(2, new StackAction(StackActionType.POP, 0)) // POP
        ).list().ofMinSize(0).ofMaxSize(50);
    }
    
    static class StackAction {
        final StackActionType type;
        final int value;
        
        StackAction(StackActionType type, int value) {
            this.type = type;
            this.value = value;
        }
    }
    
    enum StackActionType {
        PUSH, POP, PEEK
    }
}
```

## 🎯 Business Logic Properties

### Discount Calculation Properties

```java
@Property
@DisplayName("Discount Calculation Properties")
class DiscountProperties {
    
    @Property
    @DisplayName("Discount should never exceed total amount")
    void discountNeverExceedsTotal(
        @ForAll @Positive BigDecimal amount,
        @ForAll @DoubleRange(min = 0.0, max = 1.0) double discountRate
    ) {
        DiscountCalculator calculator = new DiscountCalculator();
        BigDecimal discount = calculator.calculateDiscount(amount, discountRate);
        
        Assertions.assertTrue(discount.compareTo(BigDecimal.ZERO) >= 0);
        Assertions.assertTrue(discount.compareTo(amount) <= 0);
    }
    
    @Property
    @DisplayName("Zero discount rate should result in zero discount")
    void zeroDiscountRateResultsInZeroDiscount(
        @ForAll @Positive BigDecimal amount
    ) {
        DiscountCalculator calculator = new DiscountCalculator();
        BigDecimal discount = calculator.calculateDiscount(amount, 0.0);
        
        Assertions.assertEquals(BigDecimal.ZERO, discount);
    }
    
    @Property
    @DisplayName("100% discount rate should equal total amount")
    void fullDiscountRateEqualsTotalAmount(
        @ForAll @Positive BigDecimal amount
    ) {
        DiscountCalculator calculator = new DiscountCalculator();
        BigDecimal discount = calculator.calculateDiscount(amount, 1.0);
        
        Assertions.assertEquals(amount, discount);
    }
    
    @Property
    @DisplayName("Discount is monotonic with rate")
    void discountIsMonotonicWithRate(
        @ForAll @Positive BigDecimal amount,
        @ForAll @DoubleRange(min = 0.0, max = 0.5) double rate1,
        @ForAll @DoubleRange(min = 0.5, max = 1.0) double rate2
    ) {
        DiscountCalculator calculator = new DiscountCalculator();
        BigDecimal discount1 = calculator.calculateDiscount(amount, rate1);
        BigDecimal discount2 = calculator.calculateDiscount(amount, rate2);
        
        Assertions.assertTrue(discount1.compareTo(discount2) <= 0);
    }
}
```

### String Processing Properties

```java
@Property
@DisplayName("String Processing Properties")
class StringProcessingProperties {
    
    @Property
    @DisplayName("Substring length is <= original string length")
    void substringLengthIsLessOrEqual(
        @ForAll @StringLength(min = 5, max = 100) String input,
        @ForAll @IntRange(min = 0, max = 50) int start,
        @ForAll @IntRange(min = 0, max = 50) int end
    ) {
        if (start >= 0 && end <= input.length() && start <= end) {
            String substring = input.substring(start, end);
            Assertions.assertTrue(substring.length() <= input.length());
        }
    }
    
    @Property
    @DisplayName("Uppercase then lowercase returns original")
    void uppercaseThenLowercaseReturnsOriginal(@ForAll String input) {
        String result = input.toUpperCase().toLowerCase();
        Assertions.assertEquals(input.toLowerCase(), result);
    }
    
    @Property
    @DisplayName("String concatenation length equals sum of lengths")
    void concatenationLengthEqualsSumOfLengths(
        @ForAll String str1,
        @ForAll String str2
    ) {
        String concatenated = str1 + str2;
        Assertions.assertEquals(
            str1.length() + str2.length(),
            concatenated.length()
        );
    }
}
```

## 🎯 Collection Properties

### List Properties

```java
@Property
@DisplayName("Collection Properties")
class CollectionProperties {
    
    @Property
    @DisplayName("Adding element increases size by 1")
    void addingElementIncreasesSize(
        @ForAll List<Integer> list,
        @ForAll Integer element
    ) {
        int originalSize = list.size();
        list.add(element);
        Assertions.assertEquals(originalSize + 1, list.size());
    }
    
    @Property
    @DisplayName("Removing element decreases size or stays same")
    void removingElementDecreasesSizeOrStaysSame(
        @ForAll List<Integer> list,
        @ForAll Integer element
    ) {
        int originalSize = list.size();
        list.remove(element);
        Assertions.assertTrue(list.size() <= originalSize);
    }
    
    @Property
    @DisplayName("Clearing list results in empty list")
    void clearingListResultsInEmpty(@ForAll List<Integer> list) {
        list.clear();
        Assertions.assertTrue(list.isEmpty());
        Assertions.assertEquals(0, list.size());
    }
    
    @Property
    @DisplayName("Sorting preserves elements")
    void sortingPreservesElements(@ForAll List<Integer> list) {
        List<Integer> original = new ArrayList<>(list);
        Collections.sort(list);
        
        Assertions.assertEquals(original.size(), list.size());
        original.forEach(element -> 
            Assertions.assertTrue(list.contains(element))
        );
    }
}
```

### Map Properties

```java
@Property
@DisplayName("Map Properties")
class MapProperties {
    
    @Property
    @DisplayName("Put operation increases size if key is new")
    void putIncreasesSizeIfKeyIsNew(
        @ForAll Map<String, Integer> map,
        @ForAll @AlphaChars @StringLength(min = 1, max = 10) String key,
        @ForAll Integer value
    ) {
        int originalSize = map.size();
        map.put(key, value);
        
        if (!map.containsKey(key)) {
            Assertions.assertEquals(originalSize + 1, map.size());
        }
    }
    
    @Property
    @DisplayName("Get returns value that was put")
    void getReturnsValueThatWasPut(
        @ForAll Map<String, Integer> map,
        @ForAll @AlphaChars @StringLength(min = 1, max = 10) String key,
        @ForAll Integer value
    ) {
        map.put(key, value);
        Assertions.assertEquals(value, map.get(key));
    }
    
    @Property
    @DisplayName("KeySet contains all keys")
    void keySetContainsAllKeys(@ForAll Map<String, Integer> map) {
        Set<String> keySet = map.keySet();
        map.keySet().forEach(key -> 
            Assertions.assertTrue(keySet.contains(key))
        );
    }
}
```

## ⚡ Performance Considerations

### Controlling Test Runs

```java
@Property(tries = 1000) // Run 1000 times
@DisplayName("Property with custom tries")
void propertyWithCustomTries(@ForAll String input) {
    // Test logic
}

@Property(maxDiscardRatio = 10) // Allow up to 10:1 discard ratio
@DisplayName("Property with custom discard ratio")
void propertyWithCustomDiscardRatio(@ForAll @StringLength(min = 5) String input) {
    // Test logic
}
```

### Filtering with Assume

```java
@Property
@DisplayName("Property with assumptions")
void propertyWithAssumptions(
    @ForAll @IntRange(min = -100, max = 100) int value
) {
    Assume.that(value > 0); // Only test positive values
    
    int result = Math.sqrt(value * value);
    Assertions.assertEquals(value, result);
}
```

## ⚠️ Common Pitfalls

### 1. Overly Complex Properties

```java
// Bad: Too complex to understand
@Property
void complexProperty(@ForAll Object a, @ForAll Object b, @ForAll Object c) {
    // Complex logic that's hard to reason about
}

// Good: Simple, clear properties
@Property
void simpleProperty(@ForAll int a, @ForAll int b) {
    Assertions.assertEquals(a + b, b + a);
}
```

### 2. Not Handling Edge Cases

```java
// Bad: Doesn't handle null or empty
@Property
void propertyWithoutEdgeCases(@ForAll String input) {
    // Assumes input is never null or empty
}

// Good: Explicitly handle edge cases
@Property
void propertyWithEdgeCases(@ForAll @Nullable String input) {
    if (input == null || input.isEmpty()) {
        // Handle edge case
    } else {
        // Normal case
    }
}
```

### 3. Ignoring Shrinking

```java
// Bad: Custom generators without shrinking
@Provide
Arbitrary<String> badGenerator() {
    return Arbitraries.of("a", "b", "c");
}

// Good: Generators with built-in shrinking
@Provide
Arbitrary<String> goodGenerator() {
    return Arbitraries.strings()
        .withCharRange('a', 'z')
        .ofMinLength(1)
        .ofMaxLength(10);
}
```

## 🎓 Interview Discussion Points

When discussing property-based testing in interviews:

1. **vs Example-Based Testing**: Explain when to use PBT vs traditional tests
2. **Property Discovery**: Describe how you identify good properties to test
3. **Generators**: Discuss creating custom generators for domain objects
4. **Shrinking**: Explain the importance of shrinking for debugging
5. **Stateful Testing**: Share experience with state machine testing
6. **Integration**: Describe how PBT fits into your testing strategy

## 📚 Related Topics

- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Mutation Testing](./mutation-testing.md)
- [Test-Driven Development](./test-driven-development.md)

---

**Key Takeaway**: Property-based testing complements example-based testing by generating hundreds of test cases automatically. It's particularly valuable for finding edge cases and encoding business invariants as verifiable properties.
