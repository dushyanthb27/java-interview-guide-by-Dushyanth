# Clean Architecture

[Home](../../README.md) > Bonus > Architecture Patterns > Clean Architecture

---

## Overview
Clean Architecture, introduced by Robert C. Martin (Uncle Bob), is a software architecture philosophy that emphasizes separation of concerns and dependency inversion. The goal is to create systems that are independent of frameworks, UI, databases, and external agencies, making them testable, maintainable, and adaptable to changing requirements.

## Pattern Description

Clean Architecture organizes code into concentric circles, with the most general business rules at the center and increasingly specific details at the outer layers. The key principle is that dependencies only point inward—outer layers depend on inner layers, but inner layers know nothing about outer layers.

### Architecture Diagram

```
                    ┌─────────────────────────┐
                    │      Frameworks &       │
                    │      Drivers            │  ← UI, Web, DB, Devices
                    │  ┌───────────────────┐  │
                    │  │   Web Adapter     │  │
                    │  │   Presenters      │  │
                    │  │   DB Adapters     │  │
                    │  └───────────────────┘  │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Use Cases /           │
                    │   Application Business   │  ← Application-specific rules
                    │  ┌───────────────────┐  │
                    │  │   Use Cases       │  │
                    │  │   Input Ports     │  │
                    │  │   Output Ports    │  │
                    │  └───────────────────┘  │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Enterprise Business    │  ← General business rules
                    │  ┌───────────────────┐  │
                    │  │   Entities        │  │
                    │  │   Value Objects   │  │
                    │  │   Domain Events   │  │
                    │  └───────────────────┘  │
                    └─────────────────────────┘
```

### Layer Responsibilities

**Entities (Innermost Circle)**
- Core business objects
- Enterprise-wide business rules
- No dependencies on outer layers
- Framework-agnostic
- Most stable, least likely to change

**Use Cases (Application Business Rules)**
- Application-specific business rules
- Orchestrate entity behavior
- Define input and output ports
- Don't depend on UI, database, or frameworks
- Can use entities from inner layer

**Interface Adapters**
- Convert data between layers
- Presenters for formatting data
- Controllers for handling requests
- Repository implementations
- Adapt external systems to internal needs

**Frameworks & Drivers (Outermost Circle)**
- Web frameworks (Spring, Express)
- Database (PostgreSQL, MongoDB)
- External services (Stripe, AWS)
- UI frameworks (React, Angular)
- Most volatile, likely to change

### Key Principles

**Dependency Rule**
- Source code dependencies must only point inward
- Inner layers know nothing about outer layers
- Outer layers depend on inner layers through interfaces
- Use dependency inversion to achieve this

**Independent of Frameworks**
- Architecture doesn't depend on specific frameworks
- Can switch frameworks without changing business logic
- Frameworks are tools, not the architecture

**Testable**
- Business rules can be tested without UI, database, or server
- Fast unit tests for core logic
- Integration tests for adapters

**Independent of UI**
- UI can change without affecting business rules
- Same business logic can support web, mobile, CLI
- UI is just an adapter

**Independent of Database**
- Can swap databases without changing business rules
- Business rules don't know about data persistence
- Database is just a detail

**Independent of External Agencies**
- Business rules don't know about external services
- External services are adapters
- Can mock external services for testing

## When to Use Clean Architecture

### Ideal Scenarios
✅ Long-lived projects requiring maintainability
✅ Complex business domains with rich logic
✅ Systems requiring high testability
✅ Projects with evolving technology requirements
✅ Teams with strong software engineering discipline
✅ When business logic is more valuable than technology

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ Short-term projects with limited lifespan
❌ Teams without architecture experience
❌ When rapid prototyping is the priority
❌ Projects with minimal business logic
❌ When framework-specific features are needed

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── banking/
│   │               ├── BankingApplication.java
│   │               ├── core/
│   │               │   ├── domain/
│   │               │   │   ├── model/
│   │               │   │   │   ├── Account.java
│   │               │   │   │   ├── Transaction.java
│   │               │   │   │   └── Money.java
│   │               │   │   ├── repository/
│   │               │   │   │   └── AccountRepository.java
│   │               │   │   └── service/
│   │               │   │       └── AccountService.java
│   │               │   └── usecase/
│   │               │       ├── CreateAccount.java
│   │               │       ├── DepositMoney.java
│   │               │       ├── WithdrawMoney.java
│   │               │       └── TransferMoney.java
│   │               ├── adapter/
│   │               │   ├── in/
│   │               │   │   ├── web/
│   │               │   │   │   ├── AccountController.java
│   │               │   │   │   └── AccountRequest.java
│   │               │   │   └── console/
│   │               │   │       └── AccountConsole.java
│   │               │   └── out/
│   │               │       ├── persistence/
│   │               │       │   ├── JpaAccountRepository.java
│   │               │       │   └── AccountEntity.java
│   │               │       └── messaging/
│   │               │           └── KafkaEventPublisher.java
│   │               └── config/
│   │                   ├── SpringConfig.java
│   │                   └── DatabaseConfig.java
```

### Core Layer - Domain Entities
```java
// Money Value Object
package com.example.banking.core.domain.model;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (currency == null || currency.trim().isEmpty()) {
            throw new IllegalArgumentException("Currency cannot be null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency.toUpperCase();
    }

    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    public static Money zero(String currency) {
        return new Money(BigDecimal.ZERO, currency);
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
            throw new IllegalArgumentException("Result would be negative");
        }
        return new Money(result, this.currency);
    }

    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) < 0;
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return Objects.equals(amount, money.amount) &&
               Objects.equals(currency, money.currency);
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

// Account Entity
package com.example.banking.core.domain.model;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class Account {
    private final UUID id;
    private final String accountNumber;
    private final String ownerName;
    private Money balance;
    private final List<Transaction> transactions;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public Account(UUID id, String accountNumber, String ownerName, Money initialBalance) {
        if (id == null) {
            throw new IllegalArgumentException("Account ID cannot be null");
        }
        if (accountNumber == null || accountNumber.trim().isEmpty()) {
            throw new IllegalArgumentException("Account number is required");
        }
        if (ownerName == null || ownerName.trim().isEmpty()) {
            throw new IllegalArgumentException("Owner name is required");
        }
        if (initialBalance == null) {
            throw new IllegalArgumentException("Initial balance cannot be null");
        }

        this.id = id;
        this.accountNumber = accountNumber.trim();
        this.ownerName = ownerName.trim();
        this.balance = initialBalance;
        this.transactions = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        
        if (initialBalance.getAmount().compareTo(java.math.BigDecimal.ZERO) > 0) {
            this.transactions.add(new Transaction(
                UUID.randomUUID(),
                id,
                TransactionType.DEPOSIT,
                initialBalance,
                initialBalance,
                "Initial deposit",
                LocalDateTime.now()
            ));
        }
    }

    public void deposit(Money amount, String description) {
        if (amount == null) {
            throw new IllegalArgumentException("Deposit amount cannot be null");
        }
        if (amount.getAmount().compareTo(java.math.BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        if (description == null || description.trim().isEmpty()) {
            throw new IllegalArgumentException("Description is required");
        }

        this.balance = this.balance.add(amount);
        this.transactions.add(new Transaction(
            UUID.randomUUID(),
            id,
            TransactionType.DEPOSIT,
            amount,
            this.balance,
            description.trim(),
            LocalDateTime.now()
        ));
        this.updatedAt = LocalDateTime.now();
    }

    public void withdraw(Money amount, String description) {
        if (amount == null) {
            throw new IllegalArgumentException("Withdrawal amount cannot be null");
        }
        if (amount.getAmount().compareTo(java.math.BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount.isGreaterThan(this.balance)) {
            throw new IllegalStateException("Insufficient funds");
        }
        if (description == null || description.trim().isEmpty()) {
            throw new IllegalArgumentException("Description is required");
        }

        this.balance = this.balance.subtract(amount);
        this.transactions.add(new Transaction(
            UUID.randomUUID(),
            id,
            TransactionType.WITHDRAWAL,
            amount,
            this.balance,
            description.trim(),
            LocalDateTime.now()
        ));
        this.updatedAt = LocalDateTime.now();
    }

    public void transferTo(Account targetAccount, Money amount, String description) {
        if (targetAccount == null) {
            throw new IllegalArgumentException("Target account cannot be null");
        }
        if (targetAccount.getId().equals(this.id)) {
            throw new IllegalArgumentException("Cannot transfer to same account");
        }
        if (amount == null) {
            throw new IllegalArgumentException("Transfer amount cannot be null");
        }
        if (amount.isGreaterThan(this.balance)) {
            throw new IllegalStateException("Insufficient funds for transfer");
        }

        this.withdraw(amount, "Transfer to " + targetAccount.getAccountNumber() + ": " + description);
        targetAccount.deposit(amount, "Transfer from " + this.accountNumber + ": " + description);
    }

    // Getters
    public UUID getId() { return id; }
    public String getAccountNumber() { return accountNumber; }
    public String getOwnerName() { return ownerName; }
    public Money getBalance() { return balance; }
    public List<Transaction> getTransactions() {
        return Collections.unmodifiableList(transactions);
    }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Transaction Entity
package com.example.banking.core.domain.model;

import java.time.LocalDateTime;
import java.util.Objects;
import java.util.UUID;

public class Transaction {
    private final UUID id;
    private final UUID accountId;
    private final TransactionType type;
    private final Money amount;
    private final Money balanceAfter;
    private final String description;
    private final LocalDateTime timestamp;

    public Transaction(UUID id, UUID accountId, TransactionType type,
                      Money amount, Money balanceAfter, String description,
                      LocalDateTime timestamp) {
        this.id = id;
        this.accountId = accountId;
        this.type = type;
        this.amount = amount;
        this.balanceAfter = balanceAfter;
        this.description = description;
        this.timestamp = timestamp;
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getAccountId() { return accountId; }
    public TransactionType getType() { return type; }
    public Money getAmount() { return amount; }
    public Money getBalanceAfter() { return balanceAfter; }
    public String getDescription() { return description; }
    public LocalDateTime getTimestamp() { return timestamp; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Transaction that = (Transaction) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// Transaction Type Enum
package com.example.banking.core.domain.model;

public enum TransactionType {
    DEPOSIT,
    WITHDRAWAL,
    TRANSFER_IN,
    TRANSFER_OUT
}
```

### Core Layer - Repository Interface
```java
package com.example.banking.core.domain.repository;

import com.example.banking.core.domain.model.Account;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface AccountRepository {
    Account save(Account account);
    Optional<Account> findById(UUID id);
    Optional<Account> findByAccountNumber(String accountNumber);
    List<Account> findByOwnerName(String ownerName);
    void delete(UUID id);
    boolean existsByAccountNumber(String accountNumber);
}
```

### Core Layer - Use Cases
```java
// Create Account Use Case
package com.example.banking.core.usecase;

import com.example.banking.core.domain.model.Account;
import com.example.banking.core.domain.model.Money;
import com.example.banking.core.domain.repository.AccountRepository;
import java.util.UUID;

public class CreateAccount {
    private final AccountRepository accountRepository;

    public CreateAccount(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    public Output execute(Input input) {
        if (accountRepository.existsByAccountNumber(input.accountNumber)) {
            throw new IllegalArgumentException(
                "Account number already exists: " + input.accountNumber
            );
        }

        Account account = new Account(
            UUID.randomUUID(),
            input.accountNumber,
            input.ownerName,
            Money.of(input.initialBalance, input.currency)
        );

        Account saved = accountRepository.save(account);
        
        return new Output(
            saved.getId(),
            saved.getAccountNumber(),
            saved.getOwnerName(),
            saved.getBalance().getAmount(),
            saved.getBalance().getCurrency(),
            saved.getCreatedAt()
        );
    }

    public static class Input {
        public final String accountNumber;
        public final String ownerName;
        public final double initialBalance;
        public final String currency;

        public Input(String accountNumber, String ownerName,
                    double initialBalance, String currency) {
            this.accountNumber = accountNumber;
            this.ownerName = ownerName;
            this.initialBalance = initialBalance;
            this.currency = currency;
        }
    }

    public static class Output {
        public final UUID id;
        public final String accountNumber;
        public final String ownerName;
        public final double balance;
        public final String currency;
        public final java.time.LocalDateTime createdAt;

        public Output(UUID id, String accountNumber, String ownerName,
                     double balance, String currency,
                     java.time.LocalDateTime createdAt) {
            this.id = id;
            this.accountNumber = accountNumber;
            this.ownerName = ownerName;
            this.balance = balance;
            this.currency = currency;
            this.createdAt = createdAt;
        }
    }
}

// Deposit Money Use Case
package com.example.banking.core.usecase;

import com.example.banking.core.domain.model.Account;
import com.example.banking.core.domain.model.Money;
import com.example.banking.core.domain.repository.AccountRepository;
import java.util.UUID;

public class DepositMoney {
    private final AccountRepository accountRepository;

    public DepositMoney(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    public Output execute(Input input) {
        Account account = accountRepository.findById(input.accountId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Account not found: " + input.accountId
                ));

        account.deposit(
            Money.of(input.amount, input.currency),
            input.description
        );

        Account saved = accountRepository.save(account);
        
        return new Output(
            saved.getId(),
            saved.getBalance().getAmount(),
            saved.getBalance().getCurrency()
        );
    }

    public static class Input {
        public final UUID accountId;
        public final double amount;
        public final String currency;
        public final String description;

        public Input(UUID accountId, double amount, String currency, String description) {
            this.accountId = accountId;
            this.amount = amount;
            this.currency = currency;
            this.description = description;
        }
    }

    public static class Output {
        public final UUID accountId;
        public final double newBalance;
        public final String currency;

        public Output(UUID accountId, double newBalance, String currency) {
            this.accountId = accountId;
            this.newBalance = newBalance;
            this.currency = currency;
        }
    }
}

// Transfer Money Use Case
package com.example.banking.core.usecase;

import com.example.banking.core.domain.model.Account;
import com.example.banking.core.domain.model.Money;
import com.example.banking.core.domain.repository.AccountRepository;
import java.util.UUID;

public class TransferMoney {
    private final AccountRepository accountRepository;

    public TransferMoney(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    public Output execute(Input input) {
        Account fromAccount = accountRepository.findById(input.fromAccountId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Source account not found: " + input.fromAccountId
                ));

        Account toAccount = accountRepository.findById(input.toAccountId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Target account not found: " + input.toAccountId
                ));

        fromAccount.transferTo(
            toAccount,
            Money.of(input.amount, input.currency),
            input.description
        );

        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        return new Output(
            fromAccount.getId(),
            toAccount.getId(),
            fromAccount.getBalance().getAmount(),
            toAccount.getBalance().getAmount(),
            input.currency
        );
    }

    public static class Input {
        public final UUID fromAccountId;
        public final UUID toAccountId;
        public final double amount;
        public final String currency;
        public final String description;

        public Input(UUID fromAccountId, UUID toAccountId, double amount,
                    String currency, String description) {
            this.fromAccountId = fromAccountId;
            this.toAccountId = toAccountId;
            this.amount = amount;
            this.currency = currency;
            this.description = description;
        }
    }

    public static class Output {
        public final UUID fromAccountId;
        public final UUID toAccountId;
        public final double fromBalance;
        public final double toBalance;
        public final String currency;

        public Output(UUID fromAccountId, UUID toAccountId, double fromBalance,
                     double toBalance, String currency) {
            this.fromAccountId = fromAccountId;
            this.toAccountId = toAccountId;
            this.fromBalance = fromBalance;
            this.toBalance = toBalance;
            this.currency = currency;
        }
    }
}
```

### Adapter Layer - Web Controller
```java
package com.example.banking.adapter.in.web;

import com.example.banking.core.usecase.CreateAccount;
import com.example.banking.core.usecase.DepositMoney;
import com.example.banking.core.usecase.TransferMoney;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/accounts")
public class AccountController {
    
    private final CreateAccount createAccount;
    private final DepositMoney depositMoney;
    private final TransferMoney transferMoney;

    public AccountController(CreateAccount createAccount,
                           DepositMoney depositMoney,
                           TransferMoney transferMoney) {
        this.createAccount = createAccount;
        this.depositMoney = depositMoney;
        this.transferMoney = transferMoney;
    }

    @PostMapping
    public ResponseEntity<CreateAccount.Output> createAccount(
            @RequestBody CreateAccountRequest request) {
        CreateAccount.Input input = new CreateAccount.Input(
            request.getAccountNumber(),
            request.getOwnerName(),
            request.getInitialBalance(),
            request.getCurrency()
        );
        
        CreateAccount.Output output = createAccount.execute(input);
        return ResponseEntity.status(HttpStatus.CREATED).body(output);
    }

    @PostMapping("/{id}/deposit")
    public ResponseEntity<DepositMoney.Output> deposit(
            @PathVariable UUID id,
            @RequestBody DepositRequest request) {
        DepositMoney.Input input = new DepositMoney.Input(
            id,
            request.getAmount(),
            request.getCurrency(),
            request.getDescription()
        );
        
        DepositMoney.Output output = depositMoney.execute(input);
        return ResponseEntity.ok(output);
    }

    @PostMapping("/transfer")
    public ResponseEntity<TransferMoney.Output> transfer(
            @RequestBody TransferRequest request) {
        TransferMoney.Input input = new TransferMoney.Input(
            request.getFromAccountId(),
            request.getToAccountId(),
            request.getAmount(),
            request.getCurrency(),
            request.getDescription()
        );
        
        TransferMoney.Output output = transferMoney.execute(input);
        return ResponseEntity.ok(output);
    }
}

// DTOs
class CreateAccountRequest {
    private String accountNumber;
    private String ownerName;
    private double initialBalance;
    private String currency;

    // Getters and setters
    public String getAccountNumber() { return accountNumber; }
    public void setAccountNumber(String accountNumber) { this.accountNumber = accountNumber; }
    public String getOwnerName() { return ownerName; }
    public void setOwnerName(String ownerName) { this.ownerName = ownerName; }
    public double getInitialBalance() { return initialBalance; }
    public void setInitialBalance(double initialBalance) { this.initialBalance = initialBalance; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
}

class DepositRequest {
    private double amount;
    private String currency;
    private String description;

    // Getters and setters
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}

class TransferRequest {
    private UUID fromAccountId;
    private UUID toAccountId;
    private double amount;
    private String currency;
    private String description;

    // Getters and setters
    public UUID getFromAccountId() { return fromAccountId; }
    public void setFromAccountId(UUID fromAccountId) { this.fromAccountId = fromAccountId; }
    public UUID getToAccountId() { return toAccountId; }
    public void setToAccountId(UUID toAccountId) { this.toAccountId = toAccountId; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

### Adapter Layer - Persistence
```java
package com.example.banking.adapter.out.persistence;

import com.example.banking.core.domain.model.Account;
import com.example.banking.core.domain.model.Money;
import com.example.banking.core.domain.model.Transaction;
import com.example.banking.core.domain.repository.AccountRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.stream.Collectors;

@Repository
@Transactional
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
    public List<Account> findByOwnerName(String ownerName) {
        return springDataRepo.findByOwnerName(ownerName).stream()
                .map(mapper::toDomain)
                .collect(Collectors.toList());
    }

    @Override
    public void delete(UUID id) {
        springDataRepo.deleteById(id);
    }

    @Override
    public boolean existsByAccountNumber(String accountNumber) {
        return springDataRepo.existsByAccountNumber(accountNumber);
    }
}
```

## Pros and Cons

### Advantages
✅ **Framework Independence**: Business logic doesn't depend on frameworks
✅ **Testability**: Core logic can be tested without external dependencies
✅ **Maintainability**: Clear separation makes code easier to maintain
✅ **Flexibility**: Can swap technologies without changing business logic
✅ **Scalability**: Different layers can scale independently
✅ **Reusability**: Core business logic can be reused across different interfaces
✅ **Long-term Viability**: Architecture survives technology changes

### Disadvantages
❌ **Complexity**: More layers and abstractions increase complexity
❌ **Learning Curve**: Requires understanding of architectural principles
❌ **Initial Effort**: More upfront design and setup work
❌ **Overhead**: May be overkill for simple applications
❌ **Verbosity**: More boilerplate code for adapters
❌ **Team Requirements**: Needs disciplined development practices
❌ **Slower Development**: Initial development may be slower

## Real-World Use Cases

### Financial Services
- **Core**: Account management, transaction processing
- **Adapters**: Web API, mobile app, batch processing
- **Benefits**: Regulatory compliance, audit trails, security

### Healthcare Systems
- **Core**: Patient records, treatment protocols
- **Adapters**: Web portal, mobile app, integration with external systems
- **Benefits**: HIPAA compliance, data privacy, interoperability

### E-commerce Platform
- **Core**: Order processing, inventory management
- **Adapters**: Web storefront, mobile app, third-party integrations
- **Benefits**: Business logic independence, multi-channel support

## Interview Discussion Points

### Common Questions

**Q: What's the difference between Clean Architecture and Hexagonal Architecture?**
A: They're very similar in principle—both emphasize dependency inversion and separation of concerns. Clean Architecture uses concentric circles, while Hexagonal uses ports and adapters. Clean Architecture is more prescriptive about layer organization, while Hexagonal focuses more on the port/adapter pattern. Both achieve the same goal of isolating business logic from infrastructure.

**Q: How do you handle database transactions in Clean Architecture?**
A: Transactions are managed in the outer layers (infrastructure/adapters). The use cases in the inner layer don't know about transactions—they just call repository methods. The repository implementation in the adapter layer handles transaction management using framework features like Spring's @Transactional.

**Q: Where do DTOs belong in Clean Architecture?**
A: DTOs belong in the adapter layer. They're used to convert between external representations (API requests/responses) and internal domain models. The use cases work with domain entities, not DTOs. Adapters handle the transformation.

**Q: How do you handle validation in Clean Architecture?**
A: Validation occurs at multiple layers:
- Input validation in adapters (format, required fields)
- Business rule validation in use cases and entities
- Each layer validates what it's responsible for
- Domain entities enforce their own invariants

**Q: Can you use Clean Architecture with microservices?**
A: Yes, Clean Architecture works well with microservices. Each microservice can follow Clean Architecture internally. The service boundaries align with bounded contexts. This gives you the benefits of both microservices (scalability, independent deployment) and Clean Architecture (maintainability, testability).

### Advanced Topics

**Dependency Inversion Principle**
- Depend on abstractions, not concretions
- Use interfaces in inner layers
- Implementations in outer layers
- Dependency injection to wire everything together

**Testing Strategy**
- Unit tests for entities and use cases (no external dependencies)
- Integration tests for adapters
- Contract tests for API boundaries
- End-to-end tests for complete workflows

**Architecture Testing**
- Use ArchUnit to enforce architectural rules
- Ensure dependencies only point inward
- Prevent layer violations
- Maintain architectural integrity over time

## Best Practices

### Layer Design
1. **Keep layers pure**: Each layer has a single responsibility
2. **Follow dependency rule**: Dependencies only point inward
3. **Use interfaces**: Define contracts between layers
4. **Minimize layer coupling**: Use DTOs and mappers
5. **Keep entities rich**: Business logic in domain layer

### Code Organization
1. **Package by layer**: Clear package structure
2. **Separate modules**: Consider multi-module projects
3. **Consistent naming**: Follow conventions
4. **Clear boundaries**: Enforce with architecture tests

### Technology Choices
1. **Framework agnostic core**: No framework dependencies in inner layers
2. **Thin adapters**: Keep adapters focused on adaptation
3. **Dependency injection**: Use DI container to wire dependencies
4. **Configuration externalization**: Keep configuration out of code

## Related Patterns

- **Hexagonal Architecture**: Similar principles, different visualization
- **Onion Architecture**: Another name for similar concepts
- **Domain-Driven Design**: Complements Clean Architecture with domain modeling
- **Layered Architecture**: Simpler alternative for less complex systems
- **Microservices**: Can be combined with Clean Architecture

## Migration Strategies

### From Layered to Clean Architecture
1. Identify business logic in service layer
2. Extract entities with behavior
3. Define use cases as application business rules
4. Create adapters for framework-specific code
5. Move framework dependencies to outer layers

### From Monolith to Clean Microservices
1. Identify bounded contexts
2. Apply Clean Architecture within each context
3. Define clear interfaces between contexts
4. Extract contexts into separate services
5. Implement inter-service communication

---

**Next**: Explore [Domain-Driven Design](./domain-driven-design.md) to learn about business-centric modeling
