# Advanced Observability Deep-Dive

## 📋 Question
Explain advanced observability patterns including distributed tracing, metrics collection, structured logging, and monitoring strategies for microservices. How do you implement end-to-end tracing across service boundaries? Provide real-world implementation examples.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Observability is crucial for understanding system behavior in production. Advanced observability combines distributed tracing, metrics, and structured logging to provide insights into complex distributed systems.

### Distributed Tracing with OpenTelemetry

Distributed tracing tracks requests as they flow through multiple services, providing end-to-end visibility.

**OpenTelemetry Configuration:**

```java
import io.opentelemetry.api.*;
import io.opentelemetry.api.trace.*;
import io.opentelemetry.context.*;
import io.opentelemetry.extension.trace.annotations.*;
import io.opentelemetry.instrumentation.resources.*;

@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault()
            .merge(Resource.create(
                Attributes.of(
                    AttributeKey.stringKey("service.name"), "order-service",
                    AttributeKey.stringKey("service.version"), "1.0.0"
                )
            ));
        
        return OpenTelemetrySetup.builder()
            .setResource(resource)
            .build()
            .getOpenTelemetry();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("order-service", "1.0.0");
    }
}
```

**Manual Tracing:**

```java
@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private PaymentServiceClient paymentClient;
    
    public Order createOrder(OrderRequest request) {
        // Create a span
        Span span = tracer.spanBuilder("createOrder")
            .setAttribute("user.id", request.getUserId())
            .setAttribute("order.items", request.getItems().size())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Business logic
            Order order = new Order(request.getUserId(), request.getItems());
            order = orderRepository.save(order);
            
            // Add event to span
            span.addEvent("order-created", 
                Attributes.of(AttributeKey.longKey("order.id"), order.getId()));
            
            // Call downstream service (trace propagates automatically)
            PaymentResult payment = paymentClient.processPayment(
                new PaymentRequest(order.getId(), order.getTotal())
            );
            
            span.setAttribute("payment.success", payment.isSuccess());
            
            return order;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Automatic Instrumentation with @WithSpan:**

```java
@Service
public class PaymentService {
    
    @WithSpan
    public PaymentResult processPayment(PaymentRequest request) {
        // Span is automatically created
        Span.current().setAttribute("payment.amount", request.getAmount());
        
        // Business logic
        PaymentResult result = paymentGateway.charge(request);
        
        Span.current().setAttribute("payment.success", result.isSuccess());
        
        return result;
    }
    
    @WithSpan("validatePayment")
    public boolean validatePayment(PaymentRequest request) {
        // Custom span name
        return request.getAmount() > 0 && request.getCurrency() != null;
    }
}
```

**Trace Context Propagation:**

```java
@Component
public class HttpClientInterceptor implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(
        HttpRequest request,
        byte[] body,
        ClientHttpRequestExecution execution
    ) throws IOException {
        
        // Inject trace context into HTTP headers
        SpanContext spanContext = Span.current().getSpanContext();
        
        request.getHeaders().add("traceparent", 
            String.format("00-%s-%s-%02d",
                spanContext.getTraceId(),
                spanContext.getSpanId(),
                spanContext.getTraceFlags()
            ));
        
        return execution.execute(request, body);
    }
}
```

### Metrics with Micrometer

Micrometer provides a metrics facade that works with multiple monitoring systems (Prometheus, Datadog, New Relic, etc.).

**Custom Metrics:**

```java
@Service
public class OrderMetrics {
    
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderAmountSummary;
    private final Gauge activeOrdersGauge;
    
    public OrderMetrics(MeterRegistry registry) {
        // Counter - monotonically increasing
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(registry);
        
        // Timer - measures duration
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time taken to process orders")
            .tag("service", "order-service")
            .register(registry);
        
        // Distribution summary - distribution of values
        this.orderAmountSummary = DistributionSummary.builder("orders.amount")
            .description("Distribution of order amounts")
            .tag("service", "order-service")
            .register(registry);
        
        // Gauge - instantaneous value
        this.activeOrdersGauge = Gauge.builder("orders.active", 
            orderRepository, OrderRepository::countActive)
            .description("Number of active orders")
            .tag("service", "order-service")
            .register(registry);
    }
    
    public void recordOrderCreated(Order order) {
        orderCreatedCounter.increment(
            Tags.of("user.type", getUserType(order.getUserId()))
        );
        
        orderAmountSummary.record(order.getTotal());
    }
    
    public Timer.Sample startOrderProcessing() {
        return Timer.start();
    }
    
    public void stopOrderProcessing(Timer.Sample sample) {
        sample.stop(orderProcessingTimer);
    }
}
```

**Using Metrics in Service:**

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderMetrics metrics;
    
    public Order createOrder(OrderRequest request) {
        Timer.Sample timer = metrics.startOrderProcessing();
        
        try {
            Order order = new Order(request.getUserId(), request.getItems());
            order = orderRepository.save(order);
            
            metrics.recordOrderCreated(order);
            
            return order;
        } finally {
            metrics.stopOrderProcessing(timer);
        }
    }
}
```

**Prometheus Configuration:**

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health, metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

**Custom Histogram Buckets:**

```java
@Bean
public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
    return registry -> registry.config().commonTags(
        "application", "order-service",
        "region", System.getenv().getOrDefault("REGION", "us-east-1")
    );
}

@Bean
public MeterRegistryCustomizer<MeterRegistry> histogramConfig() {
    return registry -> registry.config().meterFilter(
        new MeterFilter() {
            @Override
            public DistributionStatisticConfig configure(
                Histogram.Builder histogram,
                DistributionStatisticConfig config
            ) {
                // Custom buckets for latency
                return DistributionStatisticConfig.builder()
                    .percentiles(0.5, 0.95, 0.99)
                    .percentilesHistogram(true)
                    .sla(10, 50, 100, 500, 1000)  // 10ms, 50ms, etc.
                    .build()
                    .merge(config);
            }
        }
    );
}
```

### Structured Logging

Structured logging with JSON format enables better log aggregation and analysis.

**Logback Configuration with JSON:**

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <level>level</level>
                <logger>logger</logger>
                <message>message</message>
                <stackTrace>stack_trace</stackTrace>
            </fieldNames>
            <includeContext>true</includeContext>
            <includeMdc>true</includeMdc>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON" />
    </root>
</configuration>
```

**Structured Logging with MDC:**

```java
@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(OrderRequest request) {
        // Add context to MDC (included in all log statements)
        MDC.put("user.id", request.getUserId());
        MDC.put("request.id", UUID.randomUUID().toString());
        MDC.put("trace.id", Span.current().getSpanContext().getTraceId());
        
        try {
            logger.info("Creating order");
            
            Order order = new Order(request.getUserId(), request.getItems());
            order = orderRepository.save(order);
            
            MDC.put("order.id", order.getId());
            logger.info("Order created successfully");
            
            return order;
            
        } catch (Exception e) {
            logger.error("Failed to create order", e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

**Output:**
```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created successfully",
  "user.id": "12345",
  "request.id": "abc-123-def",
  "trace.id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "order.id": "98765"
}
```

### Correlation ID Pattern

Correlation IDs track requests across multiple services.

**Correlation ID Filter:**

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String CORRELATION_ID_MDC_KEY = "correlation.id";
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY);
        }
    }
}
```

**Propagating Correlation ID to Downstream Services:**

```java
@Component
public class CorrelationIdInterceptor implements ClientHttpRequestInterceptor {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    
    @Override
    public ClientHttpResponse intercept(
        HttpRequest request,
        byte[] body,
        ClientHttpRequestExecution execution
    ) throws IOException {
        
        String correlationId = MDC.get("correlation.id");
        if (correlationId != null) {
            request.getHeaders().add(CORRELATION_ID_HEADER, correlationId);
        }
        
        return execution.execute(request, body);
    }
}
```

### Health Checks and Probes

**Custom Health Indicators:**

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(1)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("validation", "passed")
                    .build();
            }
            return Health.down()
                .withDetail("error", "Connection validation failed")
                .build();
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    @Autowired
    private PaymentServiceClient paymentClient;
    
    @Override
    public Health health() {
        try {
            boolean isHealthy = paymentClient.healthCheck();
            if (isHealthy) {
                return Health.up()
                    .withDetail("service", "payment-service")
                    .build();
            }
            return Health.down()
                .withDetail("service", "payment-service")
                .withDetail("error", "Service unhealthy")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

**Kubernetes Probes:**

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

---
# deployment.yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
```

### Code Example: Complete Observability Demo

```java
package com.interview.advanced.observability;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.context.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.*;
import io.micrometer.core.instrument.*;
import io.opentelemetry.api.trace.*;
import org.slf4j.*;
import org.slf4j.MDC;
import java.util.*;

// === OBSERVABLE SERVICE ===
@Service
public class ObservableOrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(ObservableOrderService.class);
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private OrderRepository orderRepository;
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    
    public ObservableOrderService(MeterRegistry meterRegistry) {
        this.orderCounter = Counter.builder("orders.created")
            .tag("service", "order-service")
            .register(meterRegistry);
        
        this.orderTimer = Timer.builder("orders.processing.time")
            .tag("service", "order-service")
            .register(meterRegistry);
    }
    
    @NewSpan("createOrder")
    public Order createOrder(OrderRequest request) {
        Timer.Sample timer = Timer.start();
        
        // Add trace attributes
        Span.current().setAttribute("user.id", request.getUserId());
        Span.current().setAttribute("items.count", request.getItems().size());
        
        // Add MDC context
        MDC.put("user.id", request.getUserId());
        MDC.put("trace.id", Span.current().getSpanContext().getTraceId());
        
        try {
            logger.info("Creating order");
            
            Order order = new Order(request.getUserId(), request.getItems());
            order = orderRepository.save(order);
            
            // Record metrics
            orderCounter.increment(
                Tags.of("user.type", getUserType(request.getUserId()))
            );
            
            // Add event to span
            Span.current().addEvent("order-persisted",
                Attributes.of(AttributeKey.longKey("order.id"), order.getId())
            );
            
            MDC.put("order.id", order.getId());
            logger.info("Order created successfully");
            
            return order;
            
        } catch (Exception e) {
            Span.current().recordException(e);
            Span.current().setStatus(StatusCode.ERROR, e.getMessage());
            logger.error("Failed to create order", e);
            throw e;
        } finally {
            timer.stop(orderTimer);
            MDC.clear();
        }
    }
    
    private String getUserType(Long userId) {
        // Determine user type for tagging
        return userId % 2 == 0 ? "premium" : "standard";
    }
}

// === CORRELATION ID FILTER ===
@Component
public class CorrelationFilter extends OncePerRequestFilter {
    
    private static final String CORRELATION_HEADER = "X-Correlation-ID";
    
    @Override
    protected void doFilterInternal(
        javax.servlet.http.HttpServletRequest request,
        javax.servlet.http.HttpServletResponse response,
        javax.servlet.FilterChain filterChain
    ) throws java.io.IOException, javax.servlet.ServletException {
        
        String correlationId = request.getHeader(CORRELATION_HEADER);
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        
        MDC.put("correlation.id", correlationId);
        response.setHeader(CORRELATION_HEADER, correlationId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove("correlation.id");
        }
    }
}

// === CUSTOM HEALTH INDICATOR ===
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Override
    public Health health() {
        try {
            long activeOrders = orderRepository.countActive();
            
            if (activeOrders < 10000) {
                return Health.up()
                    .withDetail("activeOrders", activeOrders)
                    .withDetail("status", "healthy")
                    .build();
            } else {
                return Health.down()
                    .withDetail("activeOrders", activeOrders)
                    .withDetail("status", "overloaded")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// === REST CONTROLLER WITH OBSERVABILITY ===
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @Autowired
    private ObservableOrderService orderService;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        Order order = orderService.createOrder(request);
        return ResponseEntity.ok(order);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }
}

// === CONFIGURATION ===
@Configuration
public class ObservabilityConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
            "application", "order-service",
            "environment", System.getenv().getOrDefault("ENV", "dev")
        );
    }
}
```

### ⚠️ Common Mistakes

**1. Not propagating trace context**

```java
// ❌ Trace context lost
public void callExternalService() {
    restTemplate.getForObject("http://service/api", Response.class);
}

// ✅ Propagate trace context
public void callExternalService() {
    Span.current().addEvent("calling-external-service");
    restTemplate.getForObject("http://service/api", Response.class);
}
```

**2. Too many metrics**

```java
// ❌ High cardinality metrics
counter.increment(Tags.of("user.id", userId));  // Too many unique values

// ✅ Low cardinality metrics
counter.increment(Tags.of("user.type", userType));  // Limited values
```

### ⚡ Performance Tip

**Sampling** is essential for high-traffic systems. Configure trace sampling to avoid overwhelming your tracing backend:

```java
@Bean
public Sampler sampler() {
    return Sampler.traceIdRatioBased(0.1);  // Sample 10% of traces
}
```

**Asynchronous logging** prevents I/O from blocking application threads:

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="JSON" />
    <queueSize>10000</queueSize>
    <discardingThreshold>0</discardingThreshold>
</appender>
```

### 🔧 Observability Internals

OpenTelemetry uses **W3C trace context** format (`traceparent` header) to propagate trace information across services. The trace context includes:
- `trace-id`: Unique identifier for the entire trace
- `span-id`: Unique identifier for the current span
- `trace-flags`: Flags including sampling decision

Micrometer instruments code by collecting metrics at runtime and pushing them to configured backends. The `MeterRegistry` is the central component for metric collection.

---

## 🎯 Interviewer Insight

Observability questions test your understanding of production monitoring. Interviewers want to see that you:
- Understand distributed tracing and context propagation
- Can implement custom metrics with proper tagging
- Know how to structure logs for aggregation
- Understand correlation IDs for request tracking
- Can design health checks for Kubernetes

Strong candidates can explain the W3C trace context format and demonstrate how to implement end-to-end tracing across services.

## ⚡ Follow-up Questions

1. How do you handle high cardinality in metrics?
2. What is the difference between logs, metrics, and traces?
3. How do you implement distributed tracing in a polyglot environment?

## 📚 Related Topics

- [Spring Boot Actuator](../../sections/13-framework-integration/README.md)
- [Monitoring JVM](../../sections/07-jvm-internals/q13-monitoring-jvm.md)
- [System Design](../../sections/14-system-design/README.md)
