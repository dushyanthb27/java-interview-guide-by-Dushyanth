# Q3: Hibernate/JPA Fundamentals 🟡

[Home](../../README.md) > [Framework Integration](./README.md) > Hibernate/JPA

---

## Overview

**JPA (Jakarta Persistence API)** is a specification that defines how Java objects map to relational database tables. **Hibernate** is the most popular JPA implementation. Understanding both the standard and its implementation details is crucial.

---

## JPA Core Annotations

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name", nullable = false, length = 50)
    private String firstName;

    @Column(name = "last_name", nullable = false)
    private String lastName;

    @Column(unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    private Status status;

    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt;

    // constructors, getters, setters
}
```

---

## Relationship Mappings

### @OneToMany / @ManyToOne (Most Common)

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // ✅ mappedBy avoids extra join table
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();

    // ✅ Helper methods to maintain both sides
    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this);
    }

    public void removeEmployee(Employee emp) {
        employees.remove(emp);
        emp.setDepartment(null);
    }
}

@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)   // ✅ LAZY for @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```

### @ManyToMany

```java
@Entity
public class Student {

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}
```

---

## EntityManager

```java
@Repository
@Transactional
public class EmployeeDao {

    @PersistenceContext
    private EntityManager em;

    public void save(Employee emp) {
        em.persist(emp);          // INSERT — entity becomes managed
    }

    public Employee findById(Long id) {
        return em.find(Employee.class, id);  // SELECT by PK
    }

    public Employee update(Employee emp) {
        return em.merge(emp);     // UPDATE — returns managed copy
    }

    public void delete(Employee emp) {
        em.remove(em.merge(emp)); // DELETE — must be managed first
    }
}
```

**Entity States:**
```
┌──────────┐  persist()  ┌──────────┐  detach()/close()  ┌──────────┐
│   New    │────────────►│ Managed  │──────────────────►│ Detached │
│(Transient)│             │          │                    │          │
└──────────┘              └────┬─────┘                    └────┬─────┘
                               │ remove()                      │ merge()
                               ▼                               │
                          ┌──────────┐                         │
                          │ Removed  │◄────────────────────────┘
                          └──────────┘
```

---

## JPQL (Java Persistence Query Language)

```java
// ✅ JPQL operates on entities, not tables
@Repository
public class EmployeeDao {

    @PersistenceContext
    private EntityManager em;

    public List<Employee> findByDepartment(String deptName) {
        return em.createQuery(
            "SELECT e FROM Employee e WHERE e.department.name = :dept",
            Employee.class)
            .setParameter("dept", deptName)
            .getResultList();
    }

    // ✅ Named query (defined on entity)
    // @NamedQuery(name = "Employee.findActive",
    //     query = "SELECT e FROM Employee e WHERE e.status = 'ACTIVE'")
    public List<Employee> findActive() {
        return em.createNamedQuery("Employee.findActive", Employee.class)
                 .getResultList();
    }
}
```

```java
// ❌ Anti-pattern: string concatenation → SQL injection risk
String query = "SELECT e FROM Employee e WHERE e.name = '" + name + "'";

// ✅ Best practice: parameterized queries
em.createQuery("SELECT e FROM Employee e WHERE e.name = :name")
  .setParameter("name", name);
```

---

## Criteria API (Type-Safe Queries)

```java
public List<Employee> searchEmployees(String name, String dept) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
    Root<Employee> root = cq.from(Employee.class);

    List<Predicate> predicates = new ArrayList<>();

    if (name != null) {
        predicates.add(cb.like(root.get("firstName"), "%" + name + "%"));
    }
    if (dept != null) {
        predicates.add(cb.equal(root.get("department").get("name"), dept));
    }

    cq.where(predicates.toArray(new Predicate[0]));
    return em.createQuery(cq).getResultList();
}
```

---

## Lazy vs Eager Loading

| Fetch Type | Behavior | Default For |
|------------|----------|-------------|
| `LAZY` | Load on first access | `@OneToMany`, `@ManyToMany` |
| `EAGER` | Load immediately with parent | `@ManyToOne`, `@OneToOne` |

```java
// ❌ Anti-pattern: EAGER on collections — loads ALL data every time
@OneToMany(fetch = FetchType.EAGER)
private List<Order> orders;  // 10,000 orders loaded even for simple user lookup

// ✅ Best practice: always use LAZY, fetch explicitly when needed
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

---

## The N+1 Problem

```java
// ❌ N+1 problem: 1 query for departments + N queries for employees
List<Department> depts = em.createQuery(
    "SELECT d FROM Department d", Department.class).getResultList();

for (Department d : depts) {
    System.out.println(d.getEmployees().size());  // triggers N queries!
}

// ✅ Solution 1: JOIN FETCH
List<Department> depts = em.createQuery(
    "SELECT d FROM Department d JOIN FETCH d.employees", Department.class)
    .getResultList();

// ✅ Solution 2: @EntityGraph
@EntityGraph(attributePaths = {"employees"})
@Query("SELECT d FROM Department d")
List<Department> findAllWithEmployees();

// ✅ Solution 3: @BatchSize (Hibernate-specific)
@OneToMany(mappedBy = "department")
@BatchSize(size = 25)
private List<Employee> employees;
```

---

## Transaction Management

```java
// ✅ Declarative transactions with Spring
@Service
@Transactional(readOnly = true)  // default for the class
public class EmployeeService {

    @Transactional  // read-write for this method
    public void transferEmployee(Long empId, Long newDeptId) {
        Employee emp = repository.findById(empId).orElseThrow();
        Department dept = deptRepository.findById(newDeptId).orElseThrow();
        emp.setDepartment(dept);
        // no explicit save needed — dirty checking handles it
    }

    public Employee getEmployee(Long id) {
        return repository.findById(id).orElseThrow();
    }
}
```

```java
// ❌ Anti-pattern: @Transactional on private methods — Spring proxies won't intercept
@Transactional
private void doWork() { }  // silently ignored!

// ❌ Anti-pattern: catching exceptions inside @Transactional — prevents rollback
@Transactional
public void process() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);  // transaction still commits!
    }
}

// ✅ Best practice: let exceptions propagate for automatic rollback
@Transactional(rollbackFor = Exception.class)
public void process() {
    riskyOperation();  // exception triggers rollback
}
```

---

## Second-Level Cache

```
┌──────────────────────────────────────────────┐
│         Session (1st-level cache)            │  ← per EntityManager
├──────────────────────────────────────────────┤
│      SessionFactory (2nd-level cache)        │  ← shared across sessions
│         (Ehcache, Hazelcast, etc.)           │
└──────────────────────────────────────────────┘
```

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    @Id
    private String code;
    private String name;
}
```

```properties
# application.properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| JPA vs Hibernate | JPA is the spec; Hibernate is an implementation — code to the spec |
| N+1 problem | Identify it, explain JOIN FETCH / EntityGraph as solutions |
| Lazy loading | Always default to LAZY; fetch eagerly only when explicitly needed |
| @Transactional | Class-level `readOnly=true`, method-level override for writes |
| Entity states | Know New → Managed → Detached → Removed lifecycle |

---

[← Previous: Q2 - Dependency Injection](./q02-dependency-injection.md) | [Next: Q4 - JUnit Testing →](./q04-junit-testing.md)
