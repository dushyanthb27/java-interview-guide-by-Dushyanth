# Mutation Testing

## 📋 Overview

Mutation testing evaluates the quality of your test suite by introducing small changes (mutations) to your production code and checking if your tests detect them. This guide covers PITest configuration, mutation operators, interpreting mutation scores, and integrating mutation testing into CI/CD pipelines.

## 🎯 Why Mutation Testing?

Traditional code coverage metrics (line, branch) are misleading—they measure how much code is executed, not how well it's tested. Mutation testing addresses this by:
- **Measuring Test Effectiveness**: Verifies that tests actually assert on behavior
- **Detecting Weak Tests**: Identifies tests that pass but don't catch bugs
- **Improving Test Quality**: Encourages writing meaningful assertions
- **Finding Dead Code**: Uncovers code that can be removed without breaking tests
- **Guiding Refactoring**: Shows which code is safely covered by tests

## 🔧 PITest Configuration

### Maven Configuration

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-maven</artifactId>
            <version>1.15.0</version>
            <configuration>
                <targetClasses>
                    <param>com.example.service.*</param>
                </targetClasses>
                <targetTests>
                    <param>com.example.service.*</param>
                </targetTests>
                <mutators>
                    <mutator>ALL</mutator>
                </mutators>
                <excludedClasses>
                    <param>com.example.dto.*</param>
                    <param>com.example.config.*</param>
                </excludedClasses>
                <coverageThreshold>80</coverageThreshold>
                <mutationThreshold>60</mutationThreshold>
                <timeoutConstant>5000</timeoutConstant>
                <jvmArgs>
                    <jvmArg>-Xmx1024m</jvmArg>
                </jvmArgs>
                <outputFormats>
                    <outputFormat>HTML</outputFormat>
                    <outputFormat>XML</outputFormat>
                </outputFormats>
                <timestampedReports>false</timestampedReports>
            </configuration>
            <executions>
                <execution>
                    <id>pitest</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>mutationCoverage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Gradle Configuration

```groovy
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

pitest {
    targetClasses = ['com.example.service.*']
    targetTests = ['com.example.service.*']
    excludedClasses = ['com.example.dto.*', 'com.example.config.*']
    mutators = ['ALL']
    coverageThreshold = 80
    mutationThreshold = 60
    timeoutConstant = 5000
    jvmArgs = ['-Xmx1024m']
    outputFormats = ['HTML', 'XML']
    timestampedReports = false
}

test.dependsOn pitest
```

## 🧬 Mutation Operators

Mutation operators are the specific changes PITest makes to your code. Understanding them helps you write better tests.

### Conditional Mutators

```java
// Original code
public boolean isEligibleForDiscount(Customer customer) {
    if (customer.getLoyaltyPoints() > 1000) {
        return true;
    }
    return false;
}

// Mutations:
// 1. Conditionals Boundary: > becomes >=
if (customer.getLoyaltyPoints() >= 1000) { }

// 2. Remove Conditional: if condition removed
if (true) { }

// 3. Negate Conditionals: > becomes <=
if (customer.getLoyaltyPoints() <= 1000) { }
```

### Arithmetic Mutators

```java
// Original code
public double calculateTotal(double price, int quantity) {
    return price * quantity;
}

// Mutations:
// 1. AOR (Arithmetic Operator Replacement)
return price + quantity;  // * becomes +
return price - quantity;  // * becomes -
return price / quantity;  // * becomes /
return price % quantity;  // * becomes %

// 2. AOD (Arithmetic Operator Deletion)
return price;  // quantity removed
return quantity;  // price removed
```

### Return Value Mutators

```java
// Original code
public User getUserById(Long id) {
    return userRepository.findById(id).orElse(null);
}

// Mutations:
// 1. Return Value Replacement
return null;  // Always return null
return new User();  // Return empty object
```

### Method Call Mutators

```java
// Original code
public void processOrder(Order order) {
    order.setStatus(OrderStatus.PROCESSING);
    emailService.sendConfirmation(order);
}

// Mutations:
// 1. Non Void Method Call
order.setStatus(null);  // Set to null instead of PROCESSING

// 2. Void Method Call Removal
// emailService.sendConfirmation(order);  // Method call removed
```

## 📊 Interpreting Mutation Scores

### Mutation Score Calculation

```
Mutation Score = (Killed Mutations / Total Mutations) × 100

Killed: Test detected the mutation (failed)
Survived: Test didn't detect the mutation (passed)
Timed Out: Test hung (indicates infinite loop)
```

### Example Report Analysis

```html
<!-- PITest HTML Report Summary -->
<div class="summary">
    <h3>Mutation Coverage Report</h3>
    <table>
        <tr>
            <td>Total Mutations:</td>
            <td>150</td>
        </tr>
        <tr>
            <td>Killed:</td>
            <td>120 (80%)</td>
        </tr>
        <tr>
            <td>Survived:</td>
            <td>25 (16.7%)</td>
        </tr>
        <tr>
            <td>Timed Out:</td>
            <td>5 (3.3%)</td>
        </tr>
        <tr>
            <td>Mutation Score:</td>
            <td>80%</td>
        </tr>
    </table>
</div>
```

### Analyzing Survived Mutations

```java
// Example: Survived mutation indicates weak test
public class UserService {
    public User createUser(String email, String name) {
        if (email == null || email.isEmpty()) {
            throw new IllegalArgumentException("Email required");
        }
        User user = new User(email, name);
        return userRepository.save(user);
    }
}

// Test that doesn't catch the mutation
@Test
void shouldCreateUser() {
    User user = userService.createUser("test@example.com", "John Doe");
    assertNotNull(user);
}

// Mutation: email == null becomes email != null
// Test still passes because it doesn't test null email case

// Improved test
@Test
void shouldCreateUser() {
    User user = userService.createUser("test@example.com", "John Doe");
    assertNotNull(user);
    assertEquals("test@example.com", user.getEmail());
}

@Test
void shouldThrowExceptionWhenEmailIsNull() {
    assertThatThrownBy(() -> 
        userService.createUser(null, "John Doe")
    ).isInstanceOf(IllegalArgumentException.class);
}
```

## 🎯 Mutation Testing Patterns

### Selective Mutation Testing

```xml
<!-- Run only specific mutators for faster feedback -->
<configuration>
    <mutators>
        <mutator>CONDITIONALS_BOUNDARY</mutator>
        <mutator>NEGATE_CONDITIONALS</mutator>
        <mutator>RETURN_VALS</mutator>
        <mutator>MATH</mutator>
    </mutators>
</configuration>
```

### Incremental Mutation Testing

```xml
<!-- Only test changed classes -->
<configuration>
    <detectInlinedCode>true</detectInlinedCode>
    <historyInputFile>target/pitHistory/pit.history</historyInputFile>
    <historyOutputFile>target/pitHistory/pit.history</historyOutputFile>
</configuration>
```

### Filtering by Package

```xml
<!-- Target specific packages -->
<configuration>
    <targetClasses>
        <param>com.example.service.*</param>
        <param>com.example.domain.*</param>
    </targetClasses>
    <excludedClasses>
        <param>com.example.dto.*</param>
        <param>com.example.config.*</param>
        <param>com.example.Application</param>
    </excludedClasses>
</configuration>
```

## 🔍 Analyzing Mutation Results

### Example: Improving Test Quality

```java
// Production code
public class DiscountCalculator {
    public BigDecimal calculateDiscount(Order order) {
        if (order.getTotalAmount().compareTo(new BigDecimal("100")) > 0) {
            return order.getTotalAmount().multiply(new BigDecimal("0.10"));
        }
        return BigDecimal.ZERO;
    }
}

// Initial test (weak)
@Test
void shouldCalculateDiscount() {
    Order order = new Order(new BigDecimal("150"));
    BigDecimal discount = calculator.calculateDiscount(order);
    assertNotNull(discount);
}

// Mutation survives: > becomes >=
// Test doesn't verify the actual discount amount

// Improved test
@Test
void shouldCalculateDiscountForLargeOrder() {
    Order order = new Order(new BigDecimal("150"));
    BigDecimal discount = calculator.calculateDiscount(order);
    assertEquals(new BigDecimal("15.00"), discount);
}

@Test
void shouldReturnZeroDiscountForSmallOrder() {
    Order order = new Order(new BigDecimal("50"));
    BigDecimal discount = calculator.calculateDiscount(order);
    assertEquals(BigDecimal.ZERO, discount);
}

@Test
void shouldHandleBoundaryCondition() {
    Order order = new Order(new BigDecimal("100"));
    BigDecimal discount = calculator.calculateDiscount(order);
    assertEquals(BigDecimal.ZERO, discount); // No discount for exactly 100
}
```

## ⚡ Performance Optimization

### Parallel Execution

```xml
<configuration>
    <threads>4</threads>
    <timeoutFactor>2</timeoutFactor>
    <timeoutConstant>5000</timeoutConstant>
</configuration>
```

### Excluding Expensive Tests

```java
// Mark expensive tests to exclude from mutation testing
@Tag("mutation-exclude")
@Test
void expensiveIntegrationTest() {
    // This test won't run during mutation testing
}
```

```xml
<configuration>
    <excludedTestGroups>
        <param>mutation-exclude</param>
    </excludedTestGroups>
</configuration>
```

## 🔄 CI/CD Integration

### GitHub Actions

```yaml
name: Mutation Testing

on: [push, pull_request]

jobs:
  mutation-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          
      - name: Run Mutation Tests
        run: mvn verify -Ppitest
        
      - name: Upload Mutation Report
        uses: actions/upload-artifact@v3
        with:
          name: pitest-report
          path: target/pit-reports/
          
      - name: Check Mutation Threshold
        run: |
          SCORE=$(mvn pitest:mutationCoverage | grep "Mutation Score" | awk '{print $4}')
          if (( $(echo "$SCORE < 60" | bc -l) )); then
            echo "Mutation score $SCORE is below threshold 60"
            exit 1
          fi
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Mutation Testing') {
            steps {
                sh 'mvn pitest:mutationCoverage'
            }
        }
        
        stage('Mutation Threshold Check') {
            steps {
                script {
                    def report = readXML file: 'target/pit-reports/mutations.xml'
                    def score = report.mutationCoverage.@mutationScore.toDouble()
                    if (score < 60) {
                        error("Mutation score ${score}% is below threshold 60%")
                    }
                }
            }
        }
    }
    
    post {
        always {
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'target/pit-reports',
                reportFiles: 'index.html',
                reportName: 'Mutation Report'
            ])
        }
    }
}
```

## 🎯 Mutation Testing Best Practices

### DO ✅

- Set realistic mutation thresholds (60-80%)
- Run mutation tests on critical business logic
- Analyze survived mutations to improve tests
- Use selective mutators for faster feedback
- Integrate into CI/CD for continuous quality
- Review mutation reports regularly
- Focus on high-risk code areas

### DON'T ❌

- Aim for 100% mutation score (impractical)
- Run mutation tests on every commit (too slow)
- Ignore survived mutations without review
- Test generated code (Lombok, Jackson, etc.)
- Run mutation tests on integration tests (unit tests only)
- Set thresholds too low (meaningless)
- Treat mutation score as the only metric

## ⚠️ Common Pitfalls

### 1. Testing Trivial Code

```java
// Don't mutation test getters/setters
public class User {
    private String name;
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
}

// Exclude from mutation testing
<excludedClasses>
    <param>com.example.dto.*</param>
</excludedClasses>
```

### 2. False Positives from Defensive Code

```java
// Defensive null checks might survive mutations
public String getDisplayName(User user) {
    if (user != null) {
        return user.getName();
    }
    return "Guest";
}

// Mutation: user != null becomes user == null
// Test might not exercise null case, but code is defensive

// Solution: Add test for null case
@Test
void shouldReturnGuestWhenUserIsNull() {
    String name = service.getDisplayName(null);
    assertEquals("Guest", name);
}
```

### 3. Infinite Loop Mutations

```java
// Some mutations cause infinite loops
public void processUntilComplete(Task task) {
    while (!task.isComplete()) {
        task.process();
    }
}

// Mutation: ! becomes no operation
// while (task.isComplete()) { } // Might never exit

// Solution: Set timeout to prevent hangs
<configuration>
    <timeoutConstant>5000</timeoutConstant>
</configuration>
```

## 🎓 Interview Discussion Points

When discussing mutation testing in interviews:

1. **Coverage vs Mutation Testing**: Explain why mutation testing is better than traditional coverage
2. **Mutation Thresholds**: Discuss how you set and adjust mutation score thresholds
3. **CI/CD Integration**: Describe how you integrate mutation testing into your pipeline
4. **Team Adoption**: Share strategies for getting teams to adopt mutation testing
5. **Performance**: Explain how you balance mutation testing with CI/CD speed
6. **Practical Experience**: Share real examples of bugs found through mutation testing

## 📚 Related Topics

- [Unit Testing Best Practices](./unit-testing-best-practices.md)
- [Test-Driven Development](./test-driven-development.md)
- [Integration Testing Patterns](./integration-testing-patterns.md)

---

**Key Takeaway**: Mutation testing is the gold standard for measuring test quality. It goes beyond coverage metrics to ensure your tests actually verify behavior, not just execute code.
