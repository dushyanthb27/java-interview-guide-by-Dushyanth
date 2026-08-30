# Load Testing Strategies

## 📋 Overview

Load testing evaluates system performance under expected and peak load conditions. This guide covers the Gatling framework for Java applications, performance test design patterns, load test scenarios, metrics analysis, and continuous performance testing strategies.

## 🎯 Why Load Testing?

Load testing is essential for:
- **Capacity Planning**: Determine system capacity and scaling needs
- **Performance Baselines**: Establish performance benchmarks
- **Bottleneck Identification**: Find performance bottlenecks before production
- **SLA Validation**: Verify service level agreements are met
- **Confidence in Deployments**: Ensure performance doesn't degrade with changes

## 🔧 Gatling Framework Setup

### Maven Configuration

```xml
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.9.5</version>
    <scope>test</scope>
</dependency>

<plugin>
    <groupId>io.gatling</groupId>
    <artifactId>gatling-maven-plugin</artifactId>
    <version>4.3.1</version>
</plugin>
```

### Gradle Configuration

```groovy
testImplementation 'io.gatling.highcharts:gatling-charts-highcharts:3.9.5'
```

## 📝 Basic Gatling Simulation

### Simple Load Test

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class BasicLoadTest extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .userAgentHeader("Gatling Performance Test")
  
  val scn = scenario("Basic User Scenario")
    .exec(http("Get All Users")
      .get("/api/users")
      .check(status.is(200))
      .check(jsonPath("$[*]").count.save("userCount")))
    .pause(1)
    .exec(http("Get User by ID")
      .get("/api/users/1")
      .check(status.is(200))
      .check(jsonPath("$.id").is("1")))
    .pause(1)
  
  setUp(
    scn.inject(
      rampUsers(10) during (5 seconds)
    )
  ).protocols(httpProtocol)
}
```

### Advanced Load Test with Multiple Scenarios

```scala
class AdvancedLoadTest extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling Performance Test")
  
  // Scenario 1: Read Operations
  val readScenario = scenario("Read Operations")
    .exec(http("Get Users")
      .get("/api/users")
      .check(status.is(200))
      .check(responseTimeInMillis.lte(500)))
    .pause(1, 3)
    .exec(http("Get User")
      .get("/api/users/#{userId}")
      .check(status.is(200)))
    .pause(1, 2)
  
  // Scenario 2: Write Operations
  val writeScenario = scenario("Write Operations")
    .exec(http("Create User")
      .post("/api/users")
      .body(StringBody("""{"email":"test@example.com","name":"Test User"}"""))
      .asJson
      .check(status.is(201))
      .check(jsonPath("$.id").save("newUserId")))
    .pause(2, 4)
    .exec(http("Update User")
      .put("/api/users/#{newUserId}")
      .body(StringBody("""{"email":"updated@example.com","name":"Updated User"}"""))
      .asJson
      .check(status.is(200)))
    .pause(1, 2)
    .exec(http("Delete User")
      .delete("/api/users/#{newUserId}")
      .check(status.is(204)))
    .pause(1)
  
  setUp(
    readScenario.inject(
      rampUsers(50) during (30 seconds),
      constantUsersPerSec(10) during (1 minute)
    ),
    writeScenario.inject(
      nothingFor(10 seconds),
      rampUsers(20) during (20 seconds),
      constantUsersPerSec(5) during (1 minute)
    )
  ).protocols(httpProtocol)
    .maxDuration(2 minutes)
}
```

## 🎯 Load Test Patterns

### Ramp-Up Pattern

```scala
class RampUpPattern extends Simulation {
  
  val scn = scenario("Ramp Up Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(
      rampUsers(100) during (5 minutes) // Gradual ramp-up
    )
  )
}
```

### Spike Test Pattern

```scala
class SpikeTestPattern extends Simulation {
  
  val scn = scenario("Spike Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(
      constantUsersPerSec(10) during (1 minute), // Normal load
      rampUsersPerSec(10) to (100) during (10 seconds), // Spike
      constantUsersPerSec(100) during (30 seconds), // Sustained spike
      rampUsersPerSec(100) to (10) during (10 seconds) // Recovery
    )
  )
}
```

### Soak Test Pattern

```scala
class SoakTestPattern extends Simulation {
  
  val scn = scenario("Soak Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(status.is(200)))
    .pause(2, 5)
  
  setUp(
    scn.inject(
      rampUsers(50) during (5 minutes),
      constantUsersPerSec(20) during (2 hours) // Sustained load
    )
  )
}
```

### Stress Test Pattern

```scala
class StressTestPattern extends Simulation {
  
  val scn = scenario("Stress Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(
      rampUsers(100) during (5 minutes),
      rampUsersPerSec(100) to (500) during (10 minutes), // Push to limit
      rampUsersPerSec(500) to (1000) during (5 minutes) // Beyond limit
    )
  ).assertions(
    global.responseTime.percentile3.lte(5000), // 95th percentile under 5s
    global.successfulRequests.percent.gte(95) // 95% success rate
  )
}
```

## 📊 Performance Metrics

### Key Metrics to Track

```scala
class MetricsExample extends Simulation {
  
  val scn = scenario("Metrics Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(
        status.is(200),
        responseTimeInMillis.lte(1000), // Response time check
        responseTimeInMillis.save("rt")
      ))
    .doIf(session => session("rt").as[Int] > 500) {
      exec(session => {
        println(s"Slow response: ${session("rt").as[Int]}ms")
        session
      })
    }
  
  setUp(
    scn.inject(rampUsers(50) during (1 minute))
  ).assertions(
    // Response time assertions
    global.responseTime.percentile1.lte(100), // 50th percentile
    global.responseTime.percentile2.lte(500), // 75th percentile
    global.responseTime.percentile3.lte(1000), // 95th percentile
    global.responseTime.percentile4.lte(2000), // 99th percentile
    
    // Success rate assertions
    global.successfulRequests.percent.gte(99),
    
    // Response time per request
    details("API Call" / "API Call").responseTime.percentile3.lte(1000)
  )
}
```

### Custom Metrics Collection

```scala
class CustomMetrics extends Simulation {
  
  val scn = scenario("Custom Metrics")
    .exec(http("API Call")
      .get("/api/resource")
      .check(
        status.save("status"),
        responseTimeInMillis.save("responseTime")
      ))
    .exec(session => {
      val status = session("status").as[Int]
      val responseTime = session("responseTime").as[Int]
      
      // Custom logging
      println(s"Status: $status, Response Time: ${responseTime}ms")
      
      // Custom metrics collection
      if (responseTime > 1000) {
        println("WARNING: Slow response detected")
      }
      
      session
    })
  
  setUp(
    scn.inject(rampUsers(50) during (1 minute))
  )
}
```

## 🎯 Realistic Test Scenarios

### E-commerce Scenario

```scala
class EcommerceScenario extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .userAgentHeader("Gatling E-commerce Test")
  
  val feeder = csv("products.csv").random // Load product data
  
  val browseScenario = scenario("Browse Products")
    .exec(http("Get Products")
      .get("/api/products")
      .check(status.is(200)))
    .pause(2, 5)
    .feed(feeder)
    .exec(http("Get Product Details")
      .get("/api/products/#{productId}")
      .check(status.is(200)))
    .pause(1, 3)
  
  val checkoutScenario = scenario("Checkout Process")
    .exec(http("Add to Cart")
      .post("/api/cart/items")
      .body(StringBody("""{"productId":"#{productId}","quantity":1}"""))
      .asJson
      .check(status.is(201)))
    .pause(1, 2)
    .exec(http("Get Cart")
      .get("/api/cart")
      .check(status.is(200)))
    .pause(1)
    .exec(http("Create Order")
      .post("/api/orders")
      .body(StringBody("""{"cartId":"#{cartId}"}"""))
      .asJson
      .check(status.is(201))
      .check(jsonPath("$.orderId").save("orderId")))
    .pause(2, 4)
    .exec(http("Process Payment")
      .post("/api/orders/#{orderId}/payment")
      .body(StringBody("""{"amount":100.00,"method":"credit_card"}"""))
      .asJson
      .check(status.is(200)))
    .pause(1, 2)
  
  setUp(
    browseScenario.inject(
      rampUsers(100) during (5 minutes),
      constantUsersPerSec(20) during (10 minutes)
    ),
    checkoutScenario.inject(
      nothingFor(30 seconds),
      rampUsers(50) during (3 minutes),
      constantUsersPerSec(10) during (10 minutes)
    )
  ).protocols(httpProtocol)
}
```

### API Rate Limiting Test

```scala
class RateLimitingTest extends Simulation {
  
  val scn = scenario("Rate Limiting Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(
        status.in(200, 429), // Accept 200 or 429 (rate limit)
        responseTimeInMillis.save("rt")
      ))
    .pause(100 milliseconds) // 10 requests per second
  
  setUp(
    scn.inject(
      constantUsersPerSec(50) during (2 minutes) // 50 RPS for 2 minutes
    )
  ).assertions(
    global.responseTime.percentile3.lte(1000),
    details("API Call" / "API Call").responseTime.percentile3.lte(1000)
  )
}
```

## 🔧 Test Data Management

### CSV Feeders

```scala
class CsvFeederExample extends Simulation {
  
  // users.csv: userId,email,name
  val userFeeder = csv("users.csv").circular
  
  val scn = scenario("User Operations")
    .feed(userFeeder)
    .exec(http("Get User")
      .get("/api/users/#{userId}")
      .check(status.is(200)))
    .pause(1)
    .exec(http("Update User")
      .put("/api/users/#{userId}")
      .body(StringBody("""{"email":"#{email}","name":"#{name}"}"""))
      .asJson
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(rampUsers(100) during (5 minutes))
  )
}
```

### JSON Feeders

```scala
class JsonFeederExample extends Simulation {
  
  val productFeeder = jsonFile("products.json").random
  
  val scn = scenario("Product Operations")
    .feed(productFeeder)
    .exec(http("Get Product")
      .get("/api/products/#{id}")
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(rampUsers(50) during (3 minutes))
  )
}
```

## ⚡ Performance Optimization Testing

### Database Query Performance

```scala
class DatabasePerformanceTest extends Simulation {
  
  val scn = scenario("Database Query Test")
    .exec(http("Search Users")
      .get("/api/users?search=#{searchTerm}")
      .check(
        status.is(200),
        responseTimeInMillis.lte(200) // Should be fast
      ))
    .pause(1)
  
  val searchTerms = List("john", "jane", "smith", "doe", "test")
  
  setUp(
    scn.inject(
      rampUsers(100) during (5 minutes)
    ).feeders(
      Iterator.continually(Map("searchTerm" -> 
        searchTerms(scala.util.Random.nextInt(searchTerms.length))
      ))
    )
  )
}
```

### Caching Performance Test

```scala
class CachingPerformanceTest extends Simulation {
  
  val scn = scenario("Caching Test")
    .exec(http("Get Cached Resource")
      .get("/api/resource/#{id}")
      .check(
        status.is(200),
        responseTimeInMillis.save("rt")
      ))
    .pause(1)
  
  setUp(
    scn.inject(
      rampUsers(200) during (2 minutes)
    )
  ).assertions(
    // Cached responses should be fast
    details("Get Cached Resource" / "Get Cached Resource")
      .responseTime.percentile2.lte(100) // 75th percentile under 100ms
  )
}
```

## 🔄 CI/CD Integration

### GitHub Actions

```yaml
name: Load Testing

on:
  schedule:
    - cron: '0 2 * * *' # Run daily at 2 AM
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    
    services:
      app:
        image: your-app:latest
        ports:
          - 8080:8080
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          
      - name: Run Gatling Tests
        run: mvn gatling:test
        
      - name: Upload Gatling Report
        uses: actions/upload-artifact@v3
        with:
          name: gatling-report
          path: target/gatling/
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Start Application') {
            steps {
                sh 'docker-compose up -d'
                sh 'sleep 30' // Wait for app to start
            }
        }
        
        stage('Load Test') {
            steps {
                sh 'mvn gatling:test'
            }
        }
        
        stage('Analyze Results') {
            steps {
                script {
                    // Parse Gatling report
                    def report = readJSON file: 'target/gatling/gatling-report.json'
                    def p95 = report.stats.responseTime.percentile3
                    
                    if (p95 > 1000) {
                        error("95th percentile response time ${p95}ms exceeds threshold 1000ms")
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker-compose down'
            publishHTML(target: [
                reportDir: 'target/gatling',
                reportFiles: 'index.html',
                reportName: 'Gatling Report'
            ])
        }
    }
}
```

## 🎯 Performance Baseline Establishment

```scala
class BaselineTest extends Simulation {
  
  val scn = scenario("Baseline Test")
    .exec(http("API Call")
      .get("/api/resource")
      .check(status.is(200)))
    .pause(1)
  
  setUp(
    scn.inject(
      rampUsers(50) during (5 minutes)
    )
  ).assertions(
    // Establish baseline metrics
    global.responseTime.percentile1.lte(100), // Baseline: 50th percentile
    global.responseTime.percentile2.lte(200), // Baseline: 75th percentile
    global.responseTime.percentile3.lte(500), // Baseline: 95th percentile
    global.successfulRequests.percent.gte(99.9) // Baseline: success rate
  )
}
```

## ⚠️ Common Load Testing Pitfalls

### 1. Testing Against Production

```scala
// Bad: Load testing production
val httpProtocol = http
  .baseUrl("https://production.example.com")

// Good: Test against staging or dedicated test environment
val httpProtocol = http
  .baseUrl("http://staging.example.com")
```

### 2. Unrealistic Load Patterns

```scala
// Bad: Constant high load from start
setUp(
  scn.inject(constantUsersPerSec(1000) during (10 minutes))
)

// Good: Realistic ramp-up
setUp(
  scn.inject(
    rampUsers(100) during (5 minutes),
    constantUsersPerSec(100) during (10 minutes)
  )
)
```

### 3. Ignoring Think Time

```scala
// Bad: No pauses between requests
val scn = scenario("No Think Time")
  .exec(http("API Call").get("/api/resource"))
  .exec(http("API Call").get("/api/resource"))

// Good: Realistic think time
val scn = scenario("With Think Time")
  .exec(http("API Call").get("/api/resource"))
  .pause(2, 5) // 2-5 seconds between requests
  .exec(http("API Call").get("/api/resource"))
```

### 4. Not Analyzing Results

```scala
// Bad: Running tests without assertions
setUp(scn.inject(rampUsers(100) during (5 minutes)))

// Good: With performance assertions
setUp(scn.inject(rampUsers(100) during (5 minutes)))
  .assertions(
    global.responseTime.percentile3.lte(1000),
    global.successfulRequests.percent.gte(99)
  )
```

## 🎓 Interview Discussion Points

When discussing load testing in interviews:

1. **Test Types**: Explain the difference between load, stress, spike, and soak tests
2. **Metrics**: Discuss which metrics are most important and why
3. **Tool Selection**: Compare Gatling with JMeter, Locust, and k6
4. **CI/CD Integration**: Describe how you integrate load testing into your pipeline
5. **Baseline Management**: Explain how you establish and track performance baselines
6. **Real-world Scenarios**: Share examples of performance issues you've found

## 📚 Related Topics

- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Chaos Engineering](./chaos-engineering.md)
- [Mutation Testing](./mutation-testing.md)

---

**Key Takeaway**: Load testing is essential for ensuring your application can handle expected traffic and identifying performance bottlenecks before they impact users. Use realistic scenarios and analyze results thoroughly to get meaningful insights.
