# ORM Patterns

## Overview

Object-Relational Mapping (ORM) bridges the gap between object-oriented programming and relational databases. JPA (Java Persistence API) with Hibernate is the standard ORM framework in Java, but understanding patterns and best practices is crucial for effective usage.

## JPA vs JDBC

### JDBC - Low-Level Control

```java
// JDBC - Manual mapping, verbose but explicit
public User getUserById(Long id) throws SQLException {
    String sql = "SELECT id, name, email, created_at FROM users WHERE id = ?";
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement(sql)) {
        stmt.setLong(1, id);
        ResultSet rs = stmt.executeQuery();
        if (rs.next()) {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setName(rs.getString("name"));
            user.setEmail(rs.getString("email"));
            user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
            return user;
        }
        return null;
    }
}
```

### JPA - High-Level Abstraction

```java
// JPA - Automatic mapping, concise
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // No implementation needed!
    Optional<User> findById(Long id);
}

// Usage
User user = userRepository.findById(id).orElse(null);
```

## Entity Mapping Patterns

### Basic Entity Mapping

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    private String passwordHash;
    
    @Column(name = "first_name", length = 50)
    private String firstName;
    
    @Column(name = "last_name", length = 50)
    private String lastName;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at", nullable = false, updatable = false)
    private Date createdAt;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "updated_at")
    private Date updatedAt;
    
    @Version
    private Long version; // Optimistic locking
    
    // Constructors, getters, setters
    @PrePersist
    protected void onCreate() {
        createdAt = new Date();
        updatedAt = new Date();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = new Date();
    }
}
```

### Embedded Objects

```java
@Embeddable
public class Address {
    @Column(name = "street", length = 100)
    private String street;
    
    @Column(name = "city", length = 50)
    private String city;
    
    @Column(name = "state", length = 50)
    private String state;
    
    @Column(name = "zip_code", length = 10)
    private String zipCode;
    
    @Column(name = "country", length = 50)
    private String country;
    
    // Getters, setters
}

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Embedded
    private Address homeAddress;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "work_street")),
        @AttributeOverride(name = "city", column = @Column(name = "work_city"))
    })
    private Address workAddress;
}
```

### Enum Mapping

```java
public enum UserStatus {
    ACTIVE,
    INACTIVE,
    SUSPENDED,
    DELETED
}

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // Default: ordinal (0, 1, 2, ...) - NOT recommended
    @Enumerated(EnumType.ORDINAL)
    private UserStatus statusOrdinal;
    
    // Recommended: string (ACTIVE, INACTIVE, ...)
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private UserStatus status;
}
```

## Relationship Mapping Patterns

### One-to-One

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id", referencedColumnName = "id")
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToOne(mappedBy = "profile", fetch = FetchType.LAZY)
    private User user;
    
    @Column(name = "bio", length = 500)
    private String bio;
    
    @Column(name = "avatar_url", length = 255)
    private String avatarUrl;
}
```

### One-to-Many

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY,
        mappedBy = "user"
    )
    private List<Order> orders = new ArrayList<>();
    
    // Helper methods for bidirectional relationship
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
}

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(name = "order_date", nullable = false)
    private LocalDateTime orderDate;
    
    @Column(name = "total", nullable = false, precision = 10, scale = 2)
    private BigDecimal total;
}
```

### Many-to-Many

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToMany(
        cascade = {CascadeType.PERSIST, CascadeType.MERGE},
        fetch = FetchType.LAZY
    )
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    public void addRole(Role role) {
        roles.add(role);
        role.getUsers().add(this);
    }
    
    public void removeRole(Role role) {
        roles.remove(role);
        role.getUsers().remove(this);
    }
}

@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, unique = true, length = 50)
    private String name;
    
    @ManyToMany(mappedBy = "roles", fetch = FetchType.LAZY)
    private Set<User> users = new HashSet<>();
}
```

## Inheritance Mapping Strategies

### Single Table (Default)

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "amount", nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;
    
    @Column(name = "payment_date", nullable = null)
    private LocalDateTime paymentDate;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    @Column(name = "card_number", nullable = false, length = 19)
    private String cardNumber;
    
    @Column(name = "expiry_date", nullable = false, length = 5)
    private String expiryDate;
    
    @Column(name = "cvv", nullable = false, length = 3)
    private String cvv;
}

@Entity
@DiscriminatorValue("PAYPAL")
public class PayPalPayment extends Payment {
    @Column(name = "paypal_email", nullable = false, length = 100)
    private String paypalEmail;
    
    @Column(name = "transaction_id", nullable = false, length = 50)
    private String transactionId;
}
```

### Joined

```java
@Entity
@Table(name = "employees")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "hire_date", nullable = false)
    private LocalDate hireDate;
}

@Entity
@Table(name = "full_time_employees")
@PrimaryKeyJoinColumn(name = "employee_id")
public class FullTimeEmployee extends Employee {
    @Column(name = "salary", nullable = false, precision = 10, scale = 2)
    private BigDecimal salary;
    
    @Column(name = "bonus", precision = 10, scale = 2)
    private BigDecimal bonus;
}

@Entity
@Table(name = "part_time_employees")
@PrimaryKeyJoinColumn(name = "employee_id")
public class PartTimeEmployee extends Employee {
    @Column(name = "hourly_rate", nullable = false, precision = 10, scale = 2)
    private BigDecimal hourlyRate;
    
    @Column(name = "hours_per_week", nullable = false)
    private Integer hoursPerWeek;
}
```

### Table Per Class

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE)
    private Long id;
    
    @Column(name = "make", nullable = false, length = 50)
    private String make;
    
    @Column(name = "model", nullable = false, length = 50)
    private String model;
    
    @Column(name = "year", nullable = false)
    private Integer year;
}

@Entity
@Table(name = "cars")
public class Car extends Vehicle {
    @Column(name = "num_doors", nullable = false)
    private Integer numDoors;
    
    @Column(name = "convertible", nullable = false)
    private Boolean convertible;
}

@Entity
@Table(name = "trucks")
public class Truck extends Vehicle {
    @Column(name = "cargo_capacity", nullable = false, precision = 10, scale = 2)
    private BigDecimal cargoCapacity;
    
    @Column(name = "four_wheel_drive", nullable = false)
    private Boolean fourWheelDrive;
}
```

## Loading Strategies

### Lazy Loading (Default for Relationships)

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<Order> orders;
    
    // Orders loaded only when accessed
    public List<Order> getOrders() {
        // SQL query executed here
        return orders;
    }
}

// Usage
User user = userRepository.findById(1L).get();
// No query for orders yet
List<Order> orders = user.getOrders(); // Query executed now
```

### Eager Loading

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(fetch = FetchType.EAGER, mappedBy = "user")
    private List<Order> orders;
    
    // Orders loaded with user
}

// Usage
User user = userRepository.findById(1L).get();
// Orders already loaded (JOIN query executed)
List<Order> orders = user.getOrders(); // No additional query
```

### Fetch Strategies Comparison

| Strategy | Performance | Memory Usage | Use Case |
|----------|-------------|--------------|----------|
| Lazy | Better (initial load) | Lower | Large collections, rarely accessed |
| Eager | Worse (initial load) | Higher | Small collections, always accessed |

## N+1 Query Problem

### The Problem

```java
// ❌ N+1 query problem
List<User> users = userRepository.findAll();
for (User user : users) {
    List<Order> orders = user.getOrders(); // Query per user!
}
// Result: 1 query for users + N queries for orders = N+1 queries
```

### Solution 1: Entity Graph

```java
@Entity
@Table(name = "users")
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<Order> orders;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = {"orders"})
    List<User> findAllWithOrders();
}

// Usage
List<User> users = userRepository.findAllWithOrders();
// Single query with JOIN
```

### Solution 2: JOIN FETCH in JPQL

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
    List<User> findAllWithOrders();
    
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
}
```

### Solution 3: @BatchSize

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    @BatchSize(size = 50) // Load in batches of 50
    private List<Order> orders;
}

// Now instead of N queries, you get N/50 queries
```

## Caching Strategies

### First-Level Cache (Session)

```java
// First-level cache is enabled by default (session-scoped)
@Transactional
public void firstLevelCacheExample() {
    User user1 = userRepository.findById(1L).get(); // Query executed
    User user2 = userRepository.findById(1L).get(); // No query - from cache
    // user1 == user2 (same object)
}
```

### Second-Level Cache (SessionFactory)

```java
@Entity
@Table(name = "users")
@Cacheable
@org.hibernate.annotations.Cache(
    usage = CacheConcurrencyStrategy.READ_WRITE,
    region = "users"
)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true)
    private String username;
}

// Configuration in application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
        javax:
          cache:
            provider: org.hibernate.cache.ehcache.EhCacheProvider
```

### Query Cache

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.status = :status")
    @org.hibernate.annotations.Cacheable
    List<User> findByStatus(@Param("status") String status);
}
```

## Optimistic Locking

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "stock", nullable = false)
    private Integer stock;
    
    @Version
    private Long version; // Optimistic locking field
}

@Service
public class ProductService {
    
    @Transactional
    public void updateStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        
        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
        
        // If version changed between read and write:
        // OptimisticLockException thrown
    }
}
```

## Pessimistic Locking

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Lock for update (pessimistic write)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT u FROM User u WHERE u.id = :id")
    Optional<User> findByIdForUpdate(@Param("id") Long id);
    
    // Lock for read (pessimistic read)
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT u FROM User u WHERE u.id = :id")
    Optional<User> findByIdForRead(@Param("id") Long id);
}

@Service
public class UserService {
    
    @Transactional
    public void updateUserWithLock(Long userId, String newName) {
        User user = userRepository.findByIdForUpdate(userId)
            .orElseThrow();
        
        // Row is locked - other transactions wait
        user.setName(newName);
        userRepository.save(user);
        
        // Lock released on commit
    }
}
```

## Custom Repository Methods

### Custom Query Methods

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Method name query
    List<User> findByUsername(String username);
    
    List<User> findByEmailContaining(String email);
    
    List<User> findByCreatedAtAfter(LocalDateTime date);
    
    List<User> findByStatusAndCreatedAtBetween(
        String status, 
        LocalDateTime start, 
        LocalDateTime end
    );
    
    // Count queries
    long countByStatus(String status);
    
    boolean existsByEmail(String email);
    
    // Distinct
    List<User> findDistinctByStatus(String status);
    
    // Limiting
    List<User> findTop10ByOrderByCreatedAtDesc();
    
    List<User> findFirst5ByStatusOrderByCreatedAtAsc(String status);
}
```

### Custom JPQL Queries

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query("SELECT o FROM Order o WHERE o.user.id = :userId AND o.status = :status")
    List<Order> findByUserAndStatus(@Param("userId") Long userId, 
                                     @Param("status") String status);
    
    @Query("SELECT new com.dto.OrderSummary(o.id, o.total, o.orderDate) " +
           "FROM Order o WHERE o.user.id = :userId")
    List<OrderSummary> findOrderSummariesByUser(@Param("userId") Long userId);
    
    @Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);
    
    @Query("SELECT COUNT(o) FROM Order o WHERE o.user.id = :userId")
    long countByUserId(@Param("userId") Long userId);
    
    @Query(value = "SELECT * FROM orders WHERE created_at > :date", 
           nativeQuery = true)
    List<Order> findOrdersAfterDateNative(@Param("date") LocalDateTime date);
}
```

### Custom Implementation

```java
public interface UserRepositoryCustom {
    List<User> findUsersWithComplexCriteria(String criteria);
}

@Repository
public class UserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<User> findUsersWithComplexCriteria(String criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> user = query.from(User.class);
        
        // Build complex criteria
        Predicate predicate = cb.conjunction();
        
        if (criteria != null && !criteria.isEmpty()) {
            predicate = cb.and(predicate,
                cb.or(
                    cb.like(user.get("username"), "%" + criteria + "%"),
                    cb.like(user.get("email"), "%" + criteria + "%")
                )
            );
        }
        
        query.where(predicate);
        query.orderBy(cb.desc(user.get("createdAt")));
        
        return entityManager.createQuery(query).getResultList();
    }
}

public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
    // Inherits custom methods
}
```

## Common Pitfalls

### 1. LazyInitializationException

```java
// ❌ Problem: Accessing lazy collection outside transaction
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}

// Later, outside transaction
User user = userService.getUser(1L);
List<Order> orders = user.getOrders(); // LazyInitializationException!

// ✅ Solution: Keep transaction open or use DTO projection
@Transactional(readOnly = true)
public UserDTO getUserWithOrders(Long id) {
    User user = userRepository.findByIdWithOrders(id).orElse(null);
    return UserDTO.fromEntity(user);
}
```

### 2. Over-Eager Loading

```java
// ❌ Problem: Eager loading large collections
@OneToMany(fetch = FetchType.EAGER)
private List<Order> orders; // Loads ALL orders for every user query

// ✅ Solution: Use lazy loading with fetch when needed
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

### 3. Missing Equals/HashCode

```java
// ❌ Problem: Entity in Set without proper equals/hashCode
@Entity
public class User {
    @Id
    private Long id;
    
    // No equals/hashCode - issues in Sets
}

// ✅ Solution: Implement equals/hashCode using business key
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return email != null && email.equals(user.email);
    }
    
    @Override
    public int hashCode() {
        return email != null ? email.hashCode() : 0;
    }
}
```

### 4. Large Transactions

```java
// ❌ Problem: Processing large dataset in single transaction
@Transactional
public void processAllUsers() {
    List<User> users = userRepository.findAll(); // 1M users
    for (User user : users) {
        // Process each user
        userRepository.save(user);
    }
    // Transaction too large, memory issues
}

// ✅ Solution: Batch processing
public void processAllUsers() {
    int pageSize = 1000;
    int page = 0;
    
    Page<User> userPage;
    do {
        Pageable pageable = PageRequest.of(page, pageSize);
        userPage = userRepository.findAll(pageable);
        
        transactionTemplate.executeWithoutResult(status -> {
            for (User user : userPage.getContent()) {
                // Process batch
                userRepository.save(user);
            }
        });
        
        page++;
    } while (userPage.hasNext());
}
```

## Best Practices

1. **Use lazy loading** for relationships by default
2. **Avoid N+1 queries** with EntityGraph or JOIN FETCH
3. **Implement equals/hashCode** using business keys
4. **Use DTO projections** for read operations
5. **Enable second-level cache** for reference data
6. **Use optimistic locking** for concurrency control
7. **Batch operations** for large datasets
8. **Validate entities** before persistence
9. **Use cascade carefully** - understand implications
10. **Monitor query performance** regularly

## Performance Tips

```java
// ✅ Use batch inserts
@Modifying
@Query("INSERT INTO User (name, email) VALUES (:name, :email)")
void batchInsert(@Param("name") String name, @Param("email") String email);

// ✅ Use stateless session for bulk operations
Session session = entityManager.unwrap(Session.class);
StatelessSession statelessSession = session.getSessionFactory().openStatelessSession();

// ✅ Use read-only transactions for queries
@Transactional(readOnly = true)
public List<User> getAllUsers() {
    return userRepository.findAll();
}

// ✅ Use pagination for large result sets
Page<User> users = userRepository.findAll(PageRequest.of(0, 50));
```

## Interview Questions

1. What is the N+1 query problem and how do you solve it?
2. Explain lazy vs eager loading with examples
3. What are the different inheritance mapping strategies?
4. How does optimistic locking work in JPA?
5. What is the difference between first-level and second-level cache?
6. When would you use pessimistic locking?
7. How do you handle large batch operations in JPA?
8. What are the cascade types and when to use each?

## Related Topics

- [Transaction Management](./transaction-management.md)
- [Query Optimization](./query-optimization.md)
- [Connection Pooling](./connection-pooling-patterns.md)
