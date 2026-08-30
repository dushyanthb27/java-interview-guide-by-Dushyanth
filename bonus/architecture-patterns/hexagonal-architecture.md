# Hexagonal Architecture

[Home](../../README.md) > Bonus > Architecture Patterns > Hexagonal Architecture

---

## Overview
Hexagonal Architecture, also known as Ports and Adapters pattern, was introduced by Alistair Cockburn. It isolates the core application logic from external concerns like databases, web frameworks, and messaging systems. This pattern enables the application to be driven by external actors (users, automated tests, other systems) through ports and adapters.

## Pattern Description

Hexagonal architecture organizes the application into a hexagonal shape where the core domain logic sits at the center, surrounded by ports (interfaces) that define how the application interacts with the outside world. Adapters implement these ports to connect to specific technologies.

### Architecture Diagram

```
                    ┌─────────────────┐
                    │   Primary Adapters   │
                    │  (Driving Side)      │
                    │                     │
                    │  REST Controller    │
                    │  CLI Interface      │
                    │  Test Harness       │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │      PORTS (Interfaces)    │
                    │  ┌──────────────────┐  │
                    │  │  Input Port      │  │
                    │  │  (Use Case)      │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │  Output Port     │  │
                    │  │  (Repository)    │  │
                    │  └──────────────────┘  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   CORE APPLICATION   │
                    │   (Domain Logic)     │
                    │                     │
                    │  Business Rules     │
                    │  Domain Entities    │
                    │  Use Cases          │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │      PORTS (Interfaces)    │
                    │  ┌──────────────────┐  │
                    │  │  Output Port     │  │
                    │  │  (Repository)    │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │  Event Port      │  │
                    │  │  (Publisher)     │  │
                    │  └──────────────────┘  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Secondary Adapters  │
                    │  (Driven Side)        │
                    │                     │
                    │  Database Adapter   │
                    │  Message Queue      │
                    │  External API      │
                    └─────────────────┘
```

### Key Concepts

**Ports**
- Interfaces that define how the application interacts with the outside world
- Input ports: Define use cases that the application can perform
- Output ports: Define how the application retrieves/stores data
- Belong to the core application (inside the hexagon)

**Adapters**
- Implementations of ports that connect to specific technologies
- Primary adapters (driving): Drive the application (REST, CLI, tests)
- Secondary adapters (driven): Are driven by the application (DB, messaging)
- Live outside the hexagon

**Core Application**
- Contains pure business logic
- No dependencies on external frameworks
- Independent of infrastructure concerns
- Testable without external systems

## When to Use Hexagonal Architecture

### Ideal Scenarios
✅ Applications with multiple interfaces (REST, gRPC, CLI)
✅ When core business logic needs isolation from infrastructure
✅ Testable domain logic without external dependencies
✅ When external dependencies change frequently
✅ Complex business domains requiring clear boundaries
✅ Long-lived projects with evolving requirements

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ When team lacks experience with the pattern
❌ Tight deadlines requiring rapid development
❌ Applications with minimal business logic
❌ When infrastructure and domain are tightly coupled

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── banking/
│   │               ├── application/
│   │               │   ├── port/
│   │               │   │   ├── input/
│   │               │   │   │   ├── CreateAccountUseCase.java
│   │               │   │   │   ├── TransferMoneyUseCase.java
│   │               │   │   │   └── GetAccountBalanceUseCase.java
│   │               │   │   └── output/
│   │               │   │       ├── AccountRepository.java
│   │               │   │       ├── TransactionRepository.java
│   │               │   │       └── EventPublisher.java
│   │               │   ├── service/
│   │               │   │   ├── AccountService.java
│   │               │   │   └── TransferService.java
│   │               │   └── domain/
│   │               │       ├── Account.java
│   │               │       ├── Money.java
│   │               │       └── Transaction.java
│   │               ├── adapter/
│   │               │   ├── input/
│   │               │   │   ├── rest/
│   │               │   │   │   ├── AccountController.java
│   │               │   │   │   └── TransferController.java
│   │               │   │   └── cli/
│   │               │   │       └── BankingCLI.java
│   │               │   └── output/
│   │               │       ├── persistence/
│   │               │       │   ├── JpaAccountRepository.java
│   │               │       │   ├── JpaTransactionRepository.java
│   │               │       │   └── AccountEntity.java
│   │               │       └── messaging/
│   │               │           └── KafkaEventPublisher.java
│   │               └── config/
│   │                   └── DependencyInjectionConfig.java
```

### Core Domain
```java
// Value Object
package com.example.banking.application.domain;

import java.math.BigDecimal;
import java.util.Objects;

public class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        if (currency == null || currency.trim().isEmpty()) {
            throw new IllegalArgumentException("Currency cannot be empty");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot subtract different currencies");
        }
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Insufficient funds");
        }
        return new Money(result, this.currency);
    }

    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) > 0;
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.equals(money.amount) && currency.equals(money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return amount + " " + currency;
    }
}

// Domain Entity
package com.example.banking.application.domain;

import java.time.LocalDateTime;
import java.util.UUID;

public class Account {
    private final UUID id;
    private final String accountNumber;
    private final String ownerName;
    private Money balance;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public Account(UUID id, String accountNumber, String ownerName, Money balance) {
        this.id = id;
        this.accountNumber = accountNumber;
        this.ownerName = ownerName;
        this.balance = balance;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        validate();
    }

    public void credit(Money amount) {
        this.balance = this.balance.add(amount);
        this.updatedAt = LocalDateTime.now();
    }

    public void debit(Money amount) {
        this.balance = this.balance.subtract(amount);
        this.updatedAt = LocalDateTime.now();
    }

    private void validate() {
        if (accountNumber == null || accountNumber.trim().isEmpty()) {
            throw new IllegalArgumentException("Account number is required");
        }
        if (ownerName == null || ownerName.trim().isEmpty()) {
            throw new IllegalArgumentException("Owner name is required");
        }
    }

    // Getters
    public UUID getId() { return id; }
    public String getAccountNumber() { return accountNumber; }
    public String getOwnerName() { return ownerName; }
    public Money getBalance() { return balance; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Domain Entity
package com.example.banking.application.domain;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class Transaction {
    private final UUID id;
    private final UUID fromAccountId;
    private final UUID toAccountId;
    private final Money amount;
    private final String description;
    private final LocalDateTime createdAt;
    private TransactionStatus status;

    public enum TransactionStatus {
        PENDING, COMPLETED, FAILED
    }

    public Transaction(UUID id, UUID fromAccountId, UUID toAccountId,
                      Money amount, String description) {
        this.id = id;
        this.fromAccountId = fromAccountId;
        this.toAccountId = toAccountId;
        this.amount = amount;
        this.description = description;
        this.createdAt = LocalDateTime.now();
        this.status = TransactionStatus.PENDING;
    }

    public void markAsCompleted() {
        this.status = TransactionStatus.COMPLETED;
    }

    public void markAsFailed() {
        this.status = TransactionStatus.FAILED;
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getFromAccountId() { return fromAccountId; }
    public UUID getToAccountId() { return toAccountId; }
    public Money getAmount() { return amount; }
    public String getDescription() { return description; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public TransactionStatus getStatus() { return status; }
}
```

### Ports (Interfaces)
```java
// Input Port - Use Case Interface
package com.example.banking.application.port.input;

import com.example.banking.application.domain.Account;
import com.example.banking.application.domain.Money;
import java.util.UUID;

public interface CreateAccountUseCase {
    CreateAccountCommand createAccount(CreateAccountCommand command);
    Account getAccount(UUID accountId);
}

public record CreateAccountCommand(
    String accountNumber,
    String ownerName,
    Money initialBalance
) {
    public CreateAccountCommand {
        if (accountNumber == null || accountNumber.trim().isEmpty()) {
            throw new IllegalArgumentException("Account number is required");
        }
        if (ownerName == null || ownerName.trim().isEmpty()) {
            throw new IllegalArgumentException("Owner name is required");
        }
        if (initialBalance == null) {
            throw new IllegalArgumentException("Initial balance is required");
        }
    }
}

// Input Port - Use Case Interface
package com.example.banking.application.port.input;

import com.example.banking.application.domain.Transaction;
import java.util.UUID;

public interface TransferMoneyUseCase {
    TransferCommand transfer(TransferCommand command);
    Transaction getTransaction(UUID transactionId);
}

public record TransferCommand(
    UUID fromAccountId,
    UUID toAccountId,
    Money amount,
    String description
) {
    public TransferCommand {
        if (fromAccountId == null) {
            throw new IllegalArgumentException("From account ID is required");
        }
        if (toAccountId == null) {
            throw new IllegalArgumentException("To account ID is required");
        }
        if (amount == null) {
            throw new IllegalArgumentException("Amount is required");
        }
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException("Cannot transfer to same account");
        }
    }
}

// Output Port - Repository Interface
package com.example.banking.application.port.output;

import com.example.banking.application.domain.Account;
import java.util.Optional;
import java.util.UUID;

public interface AccountRepository {
    Account save(Account account);
    Optional<Account> findById(UUID id);
    Optional<Account> findByAccountNumber(String accountNumber);
    boolean existsByAccountNumber(String accountNumber);
}

// Output Port - Repository Interface
package com.example.banking.application.port.output;

import com.example.banking.application.domain.Transaction;
import java.util.Optional;
import java.util.UUID;

public interface TransactionRepository {
    Transaction save(Transaction transaction);
    Optional<Transaction> findById(UUID id);
}

// Output Port - Event Publisher Interface
package com.example.banking.application.port.output;

public interface EventPublisher {
    void publish(String topic, Object event);
}
```

### Application Service (Core Logic)
```java
package com.example.banking.application.service;

import com.example.banking.application.domain.Account;
import com.example.banking.application.domain.Transaction;
import com.example.banking.application.port.input.CreateAccountCommand;
import com.example.banking.application.port.input.CreateAccountUseCase;
import com.example.banking.application.port.output.AccountRepository;
import com.example.banking.application.port.output.EventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@Transactional
public class AccountService implements CreateAccountUseCase {
    
    private final AccountRepository accountRepository;
    private final EventPublisher eventPublisher;

    public AccountService(AccountRepository accountRepository,
                         EventPublisher eventPublisher) {
        this.accountRepository = accountRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public CreateAccountCommand createAccount(CreateAccountCommand command) {
        if (accountRepository.existsByAccountNumber(command.accountNumber())) {
            throw new IllegalArgumentException(
                "Account with number " + command.accountNumber() + " already exists"
            );
        }

        Account account = new Account(
            UUID.randomUUID(),
            command.accountNumber(),
            command.ownerName(),
            command.initialBalance()
        );

        Account saved = accountRepository.save(account);
        
        // Publish domain event
        eventPublisher.publish("account.created", new AccountCreatedEvent(
            saved.getId(),
            saved.getAccountNumber(),
            saved.getOwnerName(),
            saved.getBalance()
        ));

        return new CreateAccountCommand(
            saved.getAccountNumber(),
            saved.getOwnerName(),
            saved.getBalance()
        );
    }

    @Override
    public Account getAccount(UUID accountId) {
        return accountRepository.findById(accountId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Account not found: " + accountId
                ));
    }

    record AccountCreatedEvent(
        UUID accountId,
        String accountNumber,
        String ownerName,
        Money balance
    ) {}
}

package com.example.banking.application.service;

import com.example.banking.application.domain.Account;
import com.example.banking.application.domain.Transaction;
import com.example.banking.application.port.input.TransferCommand;
import com.example.banking.application.port.input.TransferMoneyUseCase;
import com.example.banking.application.port.output.AccountRepository;
import com.example.banking.application.port.output.EventPublisher;
import com.example.banking.application.port.output.TransactionRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@Transactional
public class TransferService implements TransferMoneyUseCase {
    
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final EventPublisher eventPublisher;

    public TransferService(AccountRepository accountRepository,
                          TransactionRepository transactionRepository,
                          EventPublisher eventPublisher) {
        this.accountRepository = accountRepository;
        this.transactionRepository = transactionRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public TransferCommand transfer(TransferCommand command) {
        Account fromAccount = accountRepository.findById(command.fromAccountId())
                .orElseThrow(() -> new IllegalArgumentException(
                    "From account not found: " + command.fromAccountId()
                ));

        Account toAccount = accountRepository.findById(command.toAccountId())
                .orElseThrow(() -> new IllegalArgumentException(
                    "To account not found: " + command.toAccountId()
                ));

        Transaction transaction = new Transaction(
            UUID.randomUUID(),
            fromAccount.getId(),
            toAccount.getId(),
            command.amount(),
            command.description()
        );

        try {
            fromAccount.debit(command.amount());
            toAccount.credit(command.amount());
            
            transaction.markAsCompleted();
            
            accountRepository.save(fromAccount);
            accountRepository.save(toAccount);
            transactionRepository.save(transaction);

            eventPublisher.publish("transfer.completed", new TransferCompletedEvent(
                transaction.getId(),
                fromAccount.getAccountNumber(),
                toAccount.getAccountNumber(),
                command.amount()
            ));

        } catch (Exception e) {
            transaction.markAsFailed();
            transactionRepository.save(transaction);
            throw new RuntimeException("Transfer failed", e);
        }

        return command;
    }

    @Override
    public Transaction getTransaction(UUID transactionId) {
        return transactionRepository.findById(transactionId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Transaction not found: " + transactionId
                ));
    }

    record TransferCompletedEvent(
        UUID transactionId,
        String fromAccountNumber,
        String toAccountNumber,
        Money amount
    ) {}
}
```

### Primary Adapters (Driving Side)
```java
// REST Controller
package com.example.banking.adapter.input.rest;

import com.example.banking.application.domain.Money;
import com.example.banking.application.port.input.CreateAccountCommand;
import com.example.banking.application.port.input.CreateAccountUseCase;
import com.example.banking.application.port.input.TransferCommand;
import com.example.banking.application.port.input.TransferMoneyUseCase;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.UUID;

@RestController
@RequestMapping("/api/accounts")
public class AccountController {
    
    private final CreateAccountUseCase createAccountUseCase;
    private final TransferMoneyUseCase transferMoneyUseCase;

    public AccountController(CreateAccountUseCase createAccountUseCase,
                           TransferMoneyUseCase transferMoneyUseCase) {
        this.createAccountUseCase = createAccountUseCase;
        this.transferMoneyUseCase = transferMoneyUseCase;
    }

    @PostMapping
    public ResponseEntity<AccountResponse> createAccount(
            @RequestBody CreateAccountRequest request) {
        Money initialBalance = new Money(
            new BigDecimal(request.initialBalance()),
            request.currency()
        );

        CreateAccountCommand command = new CreateAccountCommand(
            request.accountNumber(),
            request.ownerName(),
            initialBalance
        );

        CreateAccountCommand result = createAccountUseCase.createAccount(command);
        
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(new AccountResponse(
                    result.accountNumber(),
                    result.ownerName(),
                    result.initialBalance().getAmount().toString(),
                    result.initialBalance().getCurrency()
                ));
    }

    @GetMapping("/{id}")
    public ResponseEntity<AccountResponse> getAccount(@PathVariable UUID id) {
        var account = createAccountUseCase.getAccount(id);
        return ResponseEntity.ok(new AccountResponse(
            account.getAccountNumber(),
            account.getOwnerName(),
            account.getBalance().getAmount().toString(),
            account.getBalance().getCurrency()
        ));
    }

    @PostMapping("/transfer")
    public ResponseEntity<TransferResponse> transfer(
            @RequestBody TransferRequest request) {
        Money amount = new Money(
            new BigDecimal(request.amount()),
            request.currency()
        );

        TransferCommand command = new TransferCommand(
            UUID.fromString(request.fromAccountId()),
            UUID.fromString(request.toAccountId()),
            amount,
            request.description()
        );

        transferMoneyUseCase.transfer(command);
        
        return ResponseEntity.ok(new TransferResponse("Transfer successful"));
    }

    record CreateAccountRequest(
        String accountNumber,
        String ownerName,
        String initialBalance,
        String currency
    ) {}

    record AccountResponse(
        String accountNumber,
        String ownerName,
        String balance,
        String currency
    ) {}

    record TransferRequest(
        String fromAccountId,
        String toAccountId,
        String amount,
        String currency,
        String description
    ) {}

    record TransferResponse(String message) {}
}
```

### Secondary Adapters (Driven Side)
```java
// JPA Repository Implementation
package com.example.banking.adapter.output.persistence;

import com.example.banking.application.domain.Account;
import com.example.banking.application.domain.Money;
import com.example.banking.application.port.output.AccountRepository;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Optional;
import java.util.UUID;

@Repository
public class JpaAccountRepository implements AccountRepository {
    
    private final SpringDataAccountRepository springDataRepo;
    private final AccountMapper mapper;

    public JpaAccountRepository(SpringDataAccountRepository springDataRepo,
                               AccountMapper mapper) {
        this.springDataRepo = springDataRepo;
        this.mapper = mapper;
    }

    @Override
    public Account save(Account account) {
        AccountEntity entity = mapper.toEntity(account);
        AccountEntity saved = springDataRepo.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Account> findById(UUID id) {
        return springDataRepo.findById(id)
                .map(mapper::toDomain);
    }

    @Override
    public Optional<Account> findByAccountNumber(String accountNumber) {
        return springDataRepo.findByAccountNumber(accountNumber)
                .map(mapper::toDomain);
    }

    @Override
    public boolean existsByAccountNumber(String accountNumber) {
        return springDataRepo.existsByAccountNumber(accountNumber);
    }
}

// JPA Entity
package com.example.banking.adapter.output.persistence;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "accounts")
public class AccountEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String accountNumber;
    
    @Column(nullable = false)
    private String ownerName;
    
    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal balance;
    
    @Column(nullable = false)
    private String currency;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    protected AccountEntity() {}

    public AccountEntity(UUID id, String accountNumber, String ownerName,
                        BigDecimal balance, String currency,
                        LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.accountNumber = accountNumber;
        this.ownerName = ownerName;
        this.balance = balance;
        this.currency = currency;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    // Getters and setters
    public UUID getId() { return id; }
    public String getAccountNumber() { return accountNumber; }
    public void setAccountNumber(String accountNumber) { this.accountNumber = accountNumber; }
    public String getOwnerName() { return ownerName; }
    public void setOwnerName(String ownerName) { this.ownerName = ownerName; }
    public BigDecimal getBalance() { return balance; }
    public void setBalance(BigDecimal balance) { this.balance = balance; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}

// Mapper
package com.example.banking.adapter.output.persistence;

import com.example.banking.application.domain.Account;
import com.example.banking.application.domain.Money;
import org.springframework.stereotype.Component;

@Component
public class AccountMapper {
    
    public AccountEntity toEntity(Account account) {
        return new AccountEntity(
            account.getId(),
            account.getAccountNumber(),
            account.getOwnerName(),
            account.getBalance().getAmount(),
            account.getBalance().getCurrency(),
            account.getCreatedAt(),
            account.getUpdatedAt()
        );
    }

    public Account toDomain(AccountEntity entity) {
        return new Account(
            entity.getId(),
            entity.getAccountNumber(),
            entity.getOwnerName(),
            new Money(entity.getBalance(), entity.getCurrency())
        );
    }
}

// Kafka Event Publisher Implementation
package com.example.banking.adapter.output.messaging;

import com.example.banking.application.port.output.EventPublisher;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class KafkaEventPublisher implements EventPublisher {
    
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public KafkaEventPublisher(KafkaTemplate<String, String> kafkaTemplate,
                              ObjectMapper objectMapper) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    @Override
    public void publish(String topic, Object event) {
        try {
            String json = objectMapper.writeValueAsString(event);
            kafkaTemplate.send(topic, json);
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event", e);
        }
    }
}
```

## Pros and Cons

### Advantages
✅ **Testability**: Core logic can be tested without external dependencies
✅ **Flexibility**: Easy to swap adapters (databases, frameworks)
✅ **Isolation**: Domain logic protected from infrastructure changes
✅ **Multiple Interfaces**: Add new interfaces without changing core
✅ **Clear Boundaries**: Explicit contracts between layers
✅ **Technology Agnostic**: Core doesn't depend on specific frameworks
✅ **Parallel Development**: Adapters and core can be developed separately

### Disadvantages
❌ **Complexity**: More layers and interfaces to manage
❌ **Learning Curve**: Requires understanding of ports/adapters concept
❌ **Boilerplate**: Additional interfaces and implementations
❌ **Overhead**: May be overkill for simple applications
❌ **Initial Effort**: Takes longer to set up initially
❌ **Team Buy-in**: Requires team commitment to maintain boundaries

## Real-World Use Cases

### Financial Trading Platform
- **Core**: Trading logic, risk calculations, order matching
- **Primary Adapters**: REST API, WebSocket, FIX protocol
- **Secondary Adapters**: Database, message queue, external exchanges

### Healthcare System
- **Core**: Patient management, treatment protocols, billing
- **Primary Adapters**: Web portal, mobile app, integration APIs
- **Secondary Adapters**: EHR systems, insurance APIs, reporting tools

### E-commerce Platform
- **Core**: Product catalog, order processing, inventory management
- **Primary Adapters**: REST API, GraphQL, admin console
- **Secondary Adapters**: Database, cache, payment gateways, shipping APIs

## Interview Discussion Points

### Common Questions

**Q: What's the difference between hexagonal and layered architecture?**
A: Layered architecture organizes code by technical concerns (presentation, business, data), while hexagonal architecture organizes by business boundaries. Hexagonal explicitly defines ports (interfaces) and adapters (implementations), making the core completely independent of infrastructure. Layered architecture often has implicit dependencies that can leak across layers.

**Q: How do you handle validation in hexagonal architecture?**
A: Validation occurs at multiple levels:
- Input adapters: Format validation (email, phone number)
- Input ports: Business rule validation (account exists, sufficient funds)
- Domain entities: Invariant validation (balance cannot be negative)
Each layer validates what it's responsible for, with domain entities enforcing core invariants.

**Q: What's the role of dependency injection in hexagonal architecture?**
A: DI is essential for wiring adapters to ports:
- Core application depends on port interfaces
- Adapters implement port interfaces
- DI container injects adapter implementations into core
- This enables swapping implementations without changing core

**Q: How do you handle transactions in hexagonal architecture?**
A: Transactions are managed at the application service level:
- Use @Transactional on service methods
- Transactions span multiple repository calls
- Keep transactions short to avoid locking
- For distributed transactions, consider saga pattern

**Q: When would hexagonal architecture be overkill?**
A: Consider simpler approaches when:
- Application is simple CRUD with minimal business logic
- Team is small and unfamiliar with the pattern
- Time constraints require rapid development
- Infrastructure is stable and unlikely to change
- Single interface (e.g., only REST API)

### Advanced Topics

**Testing Strategy**
- Unit test domain entities without any infrastructure
- Test application services with mock repositories
- Integration test adapters with real infrastructure
- End-to-end tests with all components

**Event-Driven Integration**
- Use event ports for publishing domain events
- Implement event adapters for external systems
- Consider event sourcing for complex domains
- Handle eventual consistency appropriately

**Bounded Contexts**
- Each bounded context can use hexagonal architecture
- Define clear contracts between contexts
- Use anti-corruption layers for integration
- Map domain models between contexts

## Best Practices

### Port Design
1. **Keep ports focused**: Each port should have a single responsibility
2. **Use domain language**: Port names should reflect business concepts
3. **Avoid technical leakage**: Ports shouldn't expose framework details
4. **Design for testability**: Ports should be easy to mock
5. **Version ports carefully**: Consider backward compatibility

### Adapter Design
1. **Keep adapters thin**: Adapter logic should be minimal
2. **Handle technical concerns**: Adapters deal with framework specifics
3. **Map appropriately**: Convert between domain and external models
4. **Handle errors gracefully**: Translate external errors to domain errors
5. **Log appropriately**: Add logging at adapter boundaries

### Core Design
1. **Pure business logic**: No framework dependencies in core
2. **Rich domain model**: Entities contain business behavior
3. **Use value objects**: For concepts without identity
4. **Define aggregates**: Group related entities
5. **Enforce invariants**: Validate within domain entities

## Related Patterns

- **Onion Architecture**: Similar concept, emphasizes concentric layers
- **Clean Architecture**: Uncle Bob's version with similar principles
- **Layered Architecture**: Simpler alternative for less complex domains
- **Domain-Driven Design**: Often used with hexagonal architecture

## Migration Strategies

### From Layered to Hexagonal
1. Identify domain entities and business logic
2. Extract ports from existing service interfaces
3. Move core logic to application services
4. Create adapters for existing infrastructure
5. Gradually remove layer dependencies

### Adding New Adapters
1. Define new port interface if needed
2. Implement adapter for new technology
3. Wire adapter through DI
4. Test adapter independently
5. Integrate with existing core

---

**Next**: Explore [Event-Driven Architecture](./event-driven-architecture.md) for asynchronous systems
