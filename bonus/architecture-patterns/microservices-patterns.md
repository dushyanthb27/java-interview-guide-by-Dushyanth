# Microservices Patterns

[Home](../../README.md) > Bonus > Architecture Patterns > Microservices Patterns

---

## Overview
Microservices architecture structures an application as a collection of loosely coupled, independently deployable services. Each service runs in its own process and communicates through well-defined APIs. This pattern enables independent scaling, faster development cycles, and technology diversity, but introduces complexity in distributed systems.

## Pattern Description

Microservices break a monolithic application into small, focused services that each handle a specific business capability. Services communicate through APIs (REST, gRPC, messaging) and can be developed, deployed, and scaled independently.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│                  (Routing, Auth, Rate Limiting)               │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Service A  │ │  Service B  │ │  Service C  │ │  Service D  │
│  (Product)  │ │  (Order)    │ │  (Customer) │ │  (Payment)  │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │               │
       ▼               ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Database A │ │  Database B │ │  Database C │ │  Database D │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

        │              │              │              │
        └──────────────┼──────────────┼──────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Message     │ │ Config      │ │ Service     │
│ Broker      │ │ Server     │ │ Discovery   │
│ (Kafka)     │ │ (Consul)   │ │ (Eureka)    │
└─────────────┘ └─────────────┘ └─────────────┘

Communication Patterns:

1. Synchronous (REST/gRPC)
   Service A ──────▶ Service B
   Response ◀──────

2. Asynchronous (Message Queue)
   Service A ──────▶ Message Broker ──────▶ Service B

3. Event-Driven
   Service A ──────▶ Event Broker ──────▶ Service B
                          │
                          └─────────────▶ Service C
```

### Key Concepts

**Service**
- Single business capability
- Owns its data
- Independent deployment
- Technology agnostic

**API Gateway**
- Single entry point
- Request routing
- Authentication/authorization
- Rate limiting
- Response aggregation

**Service Discovery**
- Dynamic service registration
- Service location lookup
- Load balancing
- Health checking

**Configuration Management**
- Centralized configuration
- Environment-specific configs
- Dynamic configuration updates
- Configuration versioning

**Circuit Breaker**
- Prevents cascading failures
- Automatic fallback
- Automatic recovery
- Monitoring and alerting

**Bulkhead Pattern**
- Resource isolation
- Prevents resource exhaustion
- Separate thread pools
- Separate connection pools

## When to Use Microservices

### Ideal Scenarios
✅ Large-scale applications requiring independent scaling
✅ Multiple teams working on different domains
✅ When technology diversity is beneficial
✅ Systems requiring high availability
✅ Complex business domains with clear boundaries
✅ Rapid development and deployment cycles
✅ Organizations with DevOps maturity

### When to Avoid
❌ Small applications (overkill)
❌ Teams lacking distributed systems experience
❌ When development time is constrained
❌ Simple domains with clear boundaries
❌ When strong consistency is required
❌ Organizations without DevOps capabilities

## Java Implementation

### Project Structure
```
microservices/
├── api-gateway/
│   ├── src/main/java/com/example/gateway/
│   │   ├── ApiGatewayApplication.java
│   │   ├── config/
│   │   │   ├── GatewayConfig.java
│   │   │   └── SecurityConfig.java
│   │   └── filter/
│   │       └── AuthenticationFilter.java
├── product-service/
│   ├── src/main/java/com/example/product/
│   │   ├── ProductApplication.java
│   │   ├── controller/
│   │   │   └── ProductController.java
│   │   ├── service/
│   │   │   └── ProductService.java
│   │   ├── repository/
│   │   │   └── ProductRepository.java
│   │   └── model/
│   │       └── Product.java
├── order-service/
│   ├── src/main/java/com/example/order/
│   │   ├── OrderApplication.java
│   │   ├── controller/
│   │   │   └── OrderController.java
│   │   ├── service/
│   │   │   └── OrderService.java
│   │   ├── repository/
│   │   │   └── OrderRepository.java
│   │   └── model/
│   │       └── Order.java
├── config-server/
│   ├── src/main/java/com/example/config/
│   │   ├── ConfigServerApplication.java
│   │   └── config/
│   │       └── ConfigConfig.java
├── service-discovery/
│   ├── src/main/java/com/example/discovery/
│   │   ├── DiscoveryApplication.java
│   │   └── config/
│   │       └── EurekaConfig.java
└── docker-compose.yml
```

### API Gateway
```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("product-service", r -> r
                        .path("/api/products/**")
                        .filters(f -> f
                                .filter(new AuthenticationFilter())
                                .circuitBreaker(c -> c
                                        .setName("productCircuitBreaker")
                                        .setFallbackUri("forward:/fallback/product")))
                        .uri("lb://PRODUCT-SERVICE"))
                .route("order-service", r -> r
                        .path("/api/orders/**")
                        .filters(f -> f
                                .filter(new AuthenticationFilter())
                                .circuitBreaker(c -> c
                                        .setName("orderCircuitBreaker")
                                        .setFallbackUri("forward:/fallback/order")))
                        .uri("lb://ORDER-SERVICE"))
                .build();
    }
}

package com.example.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class AuthenticationFilter implements GatewayFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Validate token (simplified)
        String token = authHeader.substring(7);
        if (!isValidToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Add user info to headers for downstream services
        exchange.getRequest().mutate()
                .header("X-User-Id", extractUserId(token))
                .build();

        return chain.filter(exchange);
    }

    private boolean isValidToken(String token) {
        // Simplified token validation
        return token != null && !token.isEmpty();
    }

    private String extractUserId(String token) {
        // Simplified user extraction
        return "user-123";
    }
}
```

### Product Service
```java
package com.example.product;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class ProductApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }
}

package com.example.product.controller;

import com.example.product.model.Product;
import com.example.product.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;

    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Product product) {
        Product created = productService.createProduct(product);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable UUID id) {
        return productService.getProduct(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping
    public ResponseEntity<List<Product>> getAllProducts() {
        List<Product> products = productService.getAllProducts();
        return ResponseEntity.ok(products);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(
            @PathVariable UUID id, 
            @RequestBody Product product) {
        return productService.updateProduct(id, product)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable UUID id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}

package com.example.product.service;

import com.example.product.model.Product;
import com.example.product.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Service
@Transactional
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;

    public Product createProduct(Product product) {
        product.setId(UUID.randomUUID());
        return productRepository.save(product);
    }

    public Optional<Product> getProduct(UUID id) {
        return productRepository.findById(id);
    }

    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    public Optional<Product> updateProduct(UUID id, Product product) {
        return productRepository.findById(id)
                .map(existing -> {
                    existing.setName(product.getName());
                    existing.setDescription(product.getDescription());
                    existing.setPrice(product.getPrice());
                    existing.setStockQuantity(product.getStockQuantity());
                    return productRepository.save(existing);
                });
    }

    public void deleteProduct(UUID id) {
        productRepository.deleteById(id);
    }
}

package com.example.product.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;
    
    @Column(nullable = false)
    private int stockQuantity;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    // Constructors, getters, setters
    public Product() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public int getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(int stockQuantity) { this.stockQuantity = stockQuantity; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}

package com.example.product.repository;

import com.example.product.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.UUID;

@Repository
public interface ProductRepository extends JpaRepository<Product, UUID> {
}
```

### Order Service with Circuit Breaker
```java
package com.example.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
@EnableFeignClients
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}

package com.example.order.client;

import com.example.order.dto.ProductDTO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.UUID;

@FeignClient(name = "PRODUCT-SERVICE", fallback = ProductClientFallback.class)
public interface ProductClient {
    
    @GetMapping("/api/products/{id}")
    ProductDTO getProduct(@PathVariable UUID id);
}

package com.example.order.client;

import com.example.order.dto.ProductDTO;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Component
public class ProductClientFallback implements ProductClient {
    
    @Override
    public ProductDTO getProduct(UUID id) {
        // Fallback when product service is unavailable
        return new ProductDTO(
            id,
            "Unknown Product",
            "Product service unavailable",
            BigDecimal.ZERO,
            0,
            LocalDateTime.now(),
            LocalDateTime.now()
        );
    }
}

package com.example.order.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record ProductDTO(
    UUID id,
    String name,
    String description,
    BigDecimal price,
    int stockQuantity,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

### Service Discovery
```java
package com.example.discovery;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class DiscoveryApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryApplication.class, args);
    }
}
```

### Configuration Server
```java
package com.example.config;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Docker Compose
```yaml
version: '3.8'

services:
  discovery:
    build: ./service-discovery
    ports:
      - "8761:8761"
    environment:
      - SPRING_PROFILES_ACTIVE=docker

  config:
    build: ./config-server
    ports:
      - "8888:8888"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - discovery

  product-service:
    build: ./product-service
    ports:
      - "8081:8081"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://discovery:8761/eureka
      - SPRING_CLOUD_CONFIG_URI=http://config:8888
    depends_on:
      - discovery
      - config

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://discovery:8761/eureka
      - SPRING_CLOUD_CONFIG_URI=http://config:8888
    depends_on:
      - discovery
      - config

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://discovery:8761/eureka
    depends_on:
      - discovery
      - product-service
      - order-service
```

## Pros and Cons

### Advantages
✅ **Independent Scaling**: Scale services based on demand
✅ **Technology Diversity**: Use different technologies per service
✅ **Faster Development**: Smaller codebases, faster builds
✅ **Fault Isolation**: Failure in one service doesn't affect others
✅ **Team Autonomy**: Teams work independently
✅ **Easy Deployment**: Deploy individual services
✅ **Better Resilience**: Circuit breakers prevent cascading failures

### Disadvantages
❌ **Complexity**: More complex than monolith
❌ **Network Latency**: Inter-service communication overhead
❌ **Data Consistency**: Distributed transactions are complex
❌ **Operational Overhead**: More services to monitor
❌ **Testing Complexity**: Integration testing is harder
❌ **Learning Curve**: Requires distributed systems knowledge
❌ **Debugging**: Harder to trace requests across services

## Real-World Use Cases

### E-commerce Platform
- **Services**: Product, Order, Payment, Inventory, Shipping
- **Benefits**: Independent scaling, team autonomy
- **Scale**: High traffic during sales events

### Streaming Platform
- **Services**: User, Content, Recommendation, Billing, Analytics
- **Benefits**: Technology diversity, rapid iteration
- **Scale**: Millions of concurrent users

### Financial Platform
- **Services**: Account, Transaction, Risk, Compliance, Reporting
- **Benefits**: Regulatory compliance, fault isolation
- **Scale**: High transaction volume

## Interview Discussion Points

### Common Questions

**Q: How do you handle distributed transactions in microservices?**
A: Distributed transaction strategies:
- Saga pattern: Compensating transactions
- Eventual consistency: Accept temporary inconsistency
- Two-phase commit: For critical operations
- Idempotent operations: Handle retries safely
- Event-driven architecture: Async communication

**Q: How do you handle service discovery?**
A: Service discovery approaches:
- Client-side discovery: Clients query registry
- Server-side discovery: Load balancer queries registry
- Service registry: Eureka, Consul, Zookeeper
- DNS-based discovery: Simpler but less dynamic
- Kubernetes service discovery: Native for K8s

**Q: How do you handle inter-service communication?**
A: Communication patterns:
- Synchronous: REST, gRPC for request/response
- Asynchronous: Message queues for event-driven
- Hybrid: Both sync and async as needed
- API Gateway: Centralized entry point
- Service mesh: Advanced communication management

**Q: How do you handle data consistency across services?**
A: Consistency strategies:
- Eventual consistency: Accept temporary inconsistency
- Saga pattern: Compensating transactions
- CQRS: Separate read and write models
- Event sourcing: Event-based state management
- Distributed transactions: Two-phase commit (rare)

**Q: How do you monitor microservices?**
A: Monitoring strategies:
- Distributed tracing: Jaeger, Zipkin
- Metrics: Prometheus, Grafana
- Logging: Centralized logging (ELK stack)
- Health checks: Service health endpoints
- Alerting: PagerDuty, OpsGenie

### Advanced Topics

**Service Mesh**
- Istio, Linkerd for service-to-service communication
- Traffic management, security, observability
- Sidecar proxy pattern
- Policy enforcement

**Container Orchestration**
- Kubernetes for deployment and scaling
- Docker for containerization
- Helm for package management
- Service discovery in Kubernetes

**API Design**
- REST vs GraphQL vs gRPC
- API versioning strategies
- API documentation (OpenAPI/Swagger)
- Contract testing (Pact)

## Best Practices

### Service Design
1. **Single responsibility**: One business capability per service
2. **Database per service**: Service owns its data
3. **API design**: Clear, versioned APIs
4. **Idempotency**: Handle retries safely
5. **Failure handling**: Graceful degradation

### Communication
1. **Async first**: Prefer async over sync
2. **Circuit breakers**: Prevent cascading failures
3. **Bulkheads**: Resource isolation
4. **Retries**: With exponential backoff
5. **Timeouts**: Always set timeouts

### Deployment
1. **Independent deployment**: Deploy services independently
2. **Blue-green deployment**: Zero-downtime deployments
3. **Canary releases**: Gradual rollout
4. **Feature flags**: Toggle features without deployment
5. **Rollback strategy**: Quick rollback capability

## Related Patterns

- **API Gateway Pattern**: Single entry point
- **Service Discovery Pattern**: Dynamic service location
- **Circuit Breaker Pattern**: Failure handling
- **Saga Pattern**: Distributed transactions
- **Event-Driven Architecture**: Async communication

## Technology Choices

### Frameworks
- **Spring Cloud**: Comprehensive microservices framework
- **Micronaut**: Lightweight, fast startup
- **Quarkus**: Cloud-native, Kubernetes-optimized
- **Helidon**: Oracle's microservices framework

### Service Discovery
- **Eureka**: Netflix service discovery
- **Consul**: HashiCorp service discovery
- **Zookeeper**: Apache coordination service
- **Kubernetes**: Native service discovery

### API Gateway
- **Spring Cloud Gateway**: Reactive gateway
- **Kong**: Open-source gateway
- **Ambassador**: Kubernetes-native gateway
- **AWS API Gateway**: Managed service

---

**Next**: Learn about [Serverless Architecture](./serverless-architecture.md) for cloud-native patterns
