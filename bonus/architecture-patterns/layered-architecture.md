# Layered Architecture

[Home](../../README.md) > Bonus > Architecture Patterns > Layered Architecture

---

## Overview
Layered architecture is one of the most common and fundamental architectural patterns. It organizes the application into distinct layers, each with specific responsibilities. This pattern promotes separation of concerns, making the codebase more maintainable and testable.

## Pattern Description

Layered architecture divides the application into horizontal layers, where each layer sits on top of the layer below it. Upper layers can use services from lower layers, but lower layers should not depend on upper layers. This creates a clear dependency hierarchy and enforces unidirectional flow of control.

### Typical Layers

```
┌─────────────────────────────────────┐
│     Presentation Layer (UI/API)     │  ← Handles user interaction
├─────────────────────────────────────┤
│      Application Layer (Service)    │  ← Business logic orchestration
├─────────────────────────────────────┤
│        Domain Layer (Model)         │  ← Core business entities
├─────────────────────────────────────┤
│    Infrastructure Layer (Data)      │  ← External systems, DB, APIs
└─────────────────────────────────────┘
```

### Layer Responsibilities

**Presentation Layer**
- Handles HTTP requests/responses
- Input validation and formatting
- View rendering (for web apps)
- API documentation
- Authentication/authorization at edge

**Application Layer**
- Orchestrates business operations
- Transaction management
- Use case implementation
- Coordinates domain objects
- Handles cross-cutting concerns

**Domain Layer**
- Core business entities
- Business rules and logic
- Domain services
- Value objects
- Aggregates and entities

**Infrastructure Layer**
- Database operations
- External API calls
- File system operations
- Message queue integration
- Caching mechanisms

## When to Use Layered Architecture

### Ideal Scenarios
✅ Traditional monolithic applications
✅ Teams familiar with n-tier architecture
✅ Applications with clear separation of concerns
✅ When you need predictable structure
✅ Medium complexity systems
✅ Enterprise applications with standard requirements

### When to Avoid
❌ Highly dynamic domains requiring flexibility
❌ Microservices with independent deployment
❌ When layers become anemic (pass-through)
❌ Real-time systems with strict latency requirements
❌ Simple CRUD applications (overkill)

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── ecommerce/
│   │               ├── EcommerceApplication.java
│   │               ├── presentation/
│   │               │   ├── controller/
│   │               │   │   └── ProductController.java
│   │               │   └── dto/
│   │               │       ├── ProductRequest.java
│   │               │       └── ProductResponse.java
│   │               ├── application/
│   │               │   ├── service/
│   │               │   │   └── ProductService.java
│   │               │   └── mapper/
│   │               │       └── ProductMapper.java
│   │               ├── domain/
│   │               │   ├── model/
│   │               │   │   └── Product.java
│   │               │   └── repository/
│   │               │       └── ProductRepository.java
│   │               └── infrastructure/
│   │                   ├── persistence/
│   │                   │   ├── JpaProductRepository.java
│   │                   │   └── ProductEntity.java
│   │                   └── config/
│   │                       └── DatabaseConfig.java
```

### Domain Layer
```java
// Domain Entity
package com.example.ecommerce.domain.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class Product {
    private final UUID id;
    private String name;
    private String description;
    private BigDecimal price;
    private int stockQuantity;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public Product(UUID id, String name, String description, 
                   BigDecimal price, int stockQuantity) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.price = price;
        this.stockQuantity = stockQuantity;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        validate();
    }

    public void updatePrice(BigDecimal newPrice) {
        if (newPrice.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        this.price = newPrice;
        this.updatedAt = LocalDateTime.now();
    }

    public void reduceStock(int quantity) {
        if (quantity > stockQuantity) {
            throw new IllegalStateException("Insufficient stock");
        }
        this.stockQuantity -= quantity;
        this.updatedAt = LocalDateTime.now();
    }

    public void increaseStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.stockQuantity += quantity;
        this.updatedAt = LocalDateTime.now();
    }

    private void validate() {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Product name is required");
        }
        if (price == null || price.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        if (stockQuantity < 0) {
            throw new IllegalArgumentException("Stock cannot be negative");
        }
    }

    // Getters
    public UUID getId() { return id; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public BigDecimal getPrice() { return price; }
    public int getStockQuantity() { return stockQuantity; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Domain Repository Interface
package com.example.ecommerce.domain.repository;

import com.example.ecommerce.domain.model.Product;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface ProductRepository {
    Product save(Product product);
    Optional<Product> findById(UUID id);
    List<Product> findAll();
    void delete(UUID id);
    boolean existsByName(String name);
}
```

### Infrastructure Layer
```java
// JPA Entity
package com.example.ecommerce.infrastructure.persistence;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "products")
public class ProductEntity {
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

    // Default constructor for JPA
    protected ProductEntity() {}

    public ProductEntity(UUID id, String name, String description,
                        BigDecimal price, int stockQuantity,
                        LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.price = price;
        this.stockQuantity = stockQuantity;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    // Getters and setters
    public UUID getId() { return id; }
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

// JPA Repository Implementation
package com.example.ecommerce.infrastructure.persistence;

import com.example.ecommerce.domain.model.Product;
import com.example.ecommerce.domain.repository.ProductRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.stream.Collectors;

@Repository
@Transactional
public class JpaProductRepository implements ProductRepository {
    
    private final SpringDataProductRepository springDataRepo;
    private final ProductMapper mapper;

    public JpaProductRepository(SpringDataProductRepository springDataRepo,
                                ProductMapper mapper) {
        this.springDataRepo = springDataRepo;
        this.mapper = mapper;
    }

    @Override
    public Product save(Product product) {
        ProductEntity entity = mapper.toEntity(product);
        ProductEntity saved = springDataRepo.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Product> findById(UUID id) {
        return springDataRepo.findById(id)
                .map(mapper::toDomain);
    }

    @Override
    public List<Product> findAll() {
        return springDataRepo.findAll().stream()
                .map(mapper::toDomain)
                .collect(Collectors.toList());
    }

    @Override
    public void delete(UUID id) {
        springDataRepo.deleteById(id);
    }

    @Override
    public boolean existsByName(String name) {
        return springDataRepo.existsByName(name);
    }
}

// Spring Data JPA Repository
package com.example.ecommerce.infrastructure.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.UUID;

@Repository
interface SpringDataProductRepository extends JpaRepository<ProductEntity, UUID> {
    boolean existsByName(String name);
}

// Mapper
package com.example.ecommerce.infrastructure.persistence;

import com.example.ecommerce.domain.model.Product;
import org.springframework.stereotype.Component;

@Component
public class ProductMapper {
    
    public ProductEntity toEntity(Product product) {
        return new ProductEntity(
            product.getId(),
            product.getName(),
            product.getDescription(),
            product.getPrice(),
            product.getStockQuantity(),
            product.getCreatedAt(),
            product.getUpdatedAt()
        );
    }

    public Product toDomain(ProductEntity entity) {
        return new Product(
            entity.getId(),
            entity.getName(),
            entity.getDescription(),
            entity.getPrice(),
            entity.getStockQuantity()
        );
    }
}
```

### Application Layer
```java
// Service
package com.example.ecommerce.application.service;

import com.example.ecommerce.application.dto.ProductRequest;
import com.example.ecommerce.application.dto.ProductResponse;
import com.example.ecommerce.domain.model.Product;
import com.example.ecommerce.domain.repository.ProductRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@Service
@Transactional
public class ProductService {
    
    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    public ProductService(ProductRepository productRepository,
                         ProductMapper productMapper) {
        this.productRepository = productRepository;
        this.productMapper = productMapper;
    }

    public ProductResponse createProduct(ProductRequest request) {
        if (productRepository.existsByName(request.getName())) {
            throw new IllegalArgumentException(
                "Product with name '" + request.getName() + "' already exists"
            );
        }

        Product product = new Product(
            UUID.randomUUID(),
            request.getName(),
            request.getDescription(),
            request.getPrice(),
            request.getStockQuantity()
        );

        Product saved = productRepository.save(product);
        return productMapper.toResponse(saved);
    }

    public ProductResponse getProduct(UUID id) {
        return productRepository.findById(id)
                .map(productMapper::toResponse)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Product not found: " + id
                ));
    }

    public List<ProductResponse> getAllProducts() {
        return productRepository.findAll().stream()
                .map(productMapper::toResponse)
                .collect(Collectors.toList());
    }

    public ProductResponse updatePrice(UUID id, java.math.BigDecimal newPrice) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Product not found: " + id
                ));
        
        product.updatePrice(newPrice);
        Product updated = productRepository.save(product);
        return productMapper.toResponse(updated);
    }

    public void deleteProduct(UUID id) {
        if (!productRepository.findById(id).isPresent()) {
            throw new IllegalArgumentException("Product not found: " + id);
        }
        productRepository.delete(id);
    }
}

// Mapper
package com.example.ecommerce.application.mapper;

import com.example.ecommerce.application.dto.ProductRequest;
import com.example.ecommerce.application.dto.ProductResponse;
import com.example.ecommerce.domain.model.Product;
import org.springframework.stereotype.Component;

@Component
public class ProductMapper {
    
    public ProductResponse toResponse(Product product) {
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getDescription(),
            product.getPrice(),
            product.getStockQuantity(),
            product.getCreatedAt(),
            product.getUpdatedAt()
        );
    }
}
```

### Presentation Layer
```java
// DTOs
package com.example.ecommerce.presentation.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class ProductRequest {
    private String name;
    private String description;
    private BigDecimal price;
    private int stockQuantity;

    // Constructors, getters, setters
    public ProductRequest() {}

    public ProductRequest(String name, String description, 
                         BigDecimal price, int stockQuantity) {
        this.name = name;
        this.description = description;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public int getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(int stockQuantity) { this.stockQuantity = stockQuantity; }
}

public class ProductResponse {
    private UUID id;
    private String name;
    private String description;
    private BigDecimal price;
    private int stockQuantity;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Constructor, getters
    public ProductResponse(UUID id, String name, String description,
                          BigDecimal price, int stockQuantity,
                          LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.price = price;
        this.stockQuantity = stockQuantity;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    public UUID getId() { return id; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public BigDecimal getPrice() { return price; }
    public int getStockQuantity() { return stockQuantity; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Controller
package com.example.ecommerce.presentation.controller;

import com.example.ecommerce.application.dto.ProductRequest;
import com.example.ecommerce.application.dto.ProductResponse;
import com.example.ecommerce.application.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody ProductRequest request) {
        ProductResponse response = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable UUID id) {
        ProductResponse response = productService.getProduct(id);
        return ResponseEntity.ok(response);
    }

    @GetMapping
    public ResponseEntity<List<ProductResponse>> getAllProducts() {
        List<ProductResponse> responses = productService.getAllProducts();
        return ResponseEntity.ok(responses);
    }

    @PatchMapping("/{id}/price")
    public ResponseEntity<ProductResponse> updatePrice(
            @PathVariable UUID id,
            @RequestBody BigDecimal newPrice) {
        ProductResponse response = productService.updatePrice(id, newPrice);
        return ResponseEntity.ok(response);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable UUID id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

## Pros and Cons

### Advantages
✅ **Clear Separation of Concerns**: Each layer has a single, well-defined responsibility
✅ **Maintainability**: Easy to locate and modify code in specific layers
✅ **Testability**: Layers can be tested independently with mocks
✅ **Reusability**: Lower layers can be reused by different upper layers
✅ **Parallel Development**: Teams can work on different layers simultaneously
✅ **Standardized Structure**: Predictable organization helps new team members
✅ **Technology Independence**: Can swap implementation in lower layers

### Disadvantages
❌ **Performance Overhead**: Multiple layers add latency
❌ **Code Duplication**: DTOs and mappers can create boilerplate
❌ **Rigid Structure**: Can be difficult to modify once established
❌ **Anemic Layers**: Layers may become pass-through with no logic
❌ **Complexity**: Overkill for simple applications
❌ **Tight Coupling**: Sometimes layers become tightly coupled
❌ **Learning Curve**: New developers must understand layer boundaries

## Real-World Use Cases

### E-commerce Platform
- **Presentation**: REST APIs for web and mobile
- **Application**: Order processing, payment orchestration
- **Domain**: Product, Order, Customer entities
- **Infrastructure**: PostgreSQL, Redis, Stripe API

### Banking System
- **Presentation**: Web portal, mobile app APIs
- **Application**: Transaction processing, account management
- **Domain**: Account, Transaction, Customer entities
- **Infrastructure**: Core banking system integration

### Content Management System
- **Presentation**: Admin UI, public API
- **Application**: Content workflow, user management
- **Domain**: Article, User, Category entities
- **Infrastructure**: File storage, search engine

## Interview Discussion Points

### Common Questions

**Q: What's the difference between layered and n-tier architecture?**
A: Layered architecture is a logical separation within a single application, while n-tier architecture involves physical separation across different machines/tiers. Layered is about code organization; n-tier is about deployment.

**Q: How do you handle validation in layered architecture?**
A: Validation occurs at multiple layers:
- Presentation: Input format validation
- Application: Business rule validation
- Domain: Entity invariant validation
Each layer validates what it's responsible for.

**Q: What's the role of DTOs in layered architecture?**
A: DTOs (Data Transfer Objects) separate layer boundaries:
- Prevent domain entities from leaking to presentation
- Allow different representations for different layers
- Enable selective field exposure
- Support versioning of APIs

**Q: How do you handle transactions in layered architecture?**
A: Transactions are typically managed at the application layer:
- Use @Transactional in Spring services
- Define transaction boundaries around use cases
- Keep transactions short to avoid locking
- Consider compensation for distributed transactions

**Q: When would you move away from layered architecture?**
A: Consider alternatives when:
- Layers become anemic with no real logic
- Performance requirements demand fewer layers
- Domain complexity requires different modeling
- Microservices architecture is needed
- Real-time processing with strict latency

### Advanced Topics

**Cross-Cutting Concerns**
- Logging, monitoring, security
- Use AOP (Aspect-Oriented Programming)
- Implement as interceptors/filters
- Keep concerns separate from business logic

**Layer Violations**
- Avoid direct database access from controllers
- Don't pass domain entities to presentation
- Keep infrastructure out of domain layer
- Use dependency injection to enforce boundaries

**Testing Strategy**
- Unit test each layer independently
- Use mocks for dependencies
- Integration tests for layer interactions
- End-to-end tests for complete flows

## Best Practices

### Layer Design
1. **Keep layers thin**: Each layer should have a clear purpose
2. **Avoid pass-through layers**: If a layer has no logic, consider removing it
3. **Use interfaces**: Define contracts between layers
4. **Follow dependency rule**: Dependencies point inward
5. **Minimize layer coupling**: Use DTOs and mappers

### Code Organization
1. **Package by layer**: Clear package structure
2. **Package by feature**: Alternative for large applications
3. **Consistent naming**: Follow conventions
4. **Clear boundaries**: Enforce with architecture tests

### Technology Choices
1. **Framework support**: Spring Boot works well with layered architecture
2. **Database abstraction**: Use repositories to hide JPA details
3. **API documentation**: OpenAPI/Swagger for presentation layer
4. **Validation**: Bean Validation for input validation

## Related Patterns

- **Hexagonal Architecture**: Evolution of layered with better isolation
- **Onion Architecture**: Similar to hexagonal, emphasizes domain
- **Clean Architecture**: Uncle Bob's take on layered architecture
- **MVC Pattern**: Often used within presentation layer

## Migration Strategies

### From Monolith to Layered
1. Identify existing code responsibilities
2. Create layer packages
3. Move code to appropriate layers
4. Add interfaces between layers
5. Refactor gradually

### From Layered to Hexagonal
1. Extract domain logic from application layer
2. Define ports (interfaces) for external interactions
3. Implement adapters for each port
4. Remove direct dependencies on frameworks
5. Test domain logic independently

---

**Next**: Learn about [Hexagonal Architecture](./hexagonal-architecture.md) for better isolation
