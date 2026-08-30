# Event Sourcing

[Home](../../README.md) > Bonus > Architecture Patterns > Event Sourcing

---

## Overview
Event Sourcing is a pattern where state changes are stored as a sequence of events rather than storing the current state. Instead of updating a record in a database, we append events to an event log. The current state is derived by replaying all events. This pattern provides a complete audit trail, enables temporal queries, and supports complex business logic.

## Pattern Description

In traditional CRUD systems, we store the current state of an entity. When state changes, we update the record, losing the history of how we got there. Event sourcing stores all state changes as immutable events. To reconstruct the current state, we replay all events from the beginning.

### Architecture Diagram

```
Traditional CRUD:
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Service    │
└──────┬──────┘
       │
       ├─────────────┬─────────────┐
       │             │             │
       ▼             ▼             ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Update     │ │  Read       │ │  Delete     │
│  State      │ │  State      │ │  State      │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │                │                │
       └────────────────┴────────────────┘
                    │
                    ▼
           ┌─────────────┐
           │  Database   │
           │  (Current   │
           │   State)    │
           └─────────────┘

Event Sourcing:
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Service    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Command    │
│  Handler    │
└──────┬──────┘
       │
       ├─────────────┐
       │             │
       ▼             ▼
┌─────────────┐ ┌─────────────┐
│  Validate   │ │  Generate   │
│  Command    │ │  Event      │
└──────┬──────┘ └──────┬──────┘
       │                │
       └────────────────┘
                    │
                    ▼
           ┌─────────────┐
           │  Event      │
           │  Store      │
           └──────┬──────┘
                  │
                  ├─────────────┬─────────────┐
                  │             │             │
                  ▼             ▼             ▼
           ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
           │  Event      │ │  Snapshot   │ │  Project    │
           │  Log        │ │  Store      │ │  to Read    │
           │             │ │             │ │  Model      │
           └─────────────┘ └─────────────┘ └─────────────┘

Event Flow:
1. Command received (CreateOrder, UpdateProduct)
2. Command validated
3. Event generated (OrderCreated, ProductUpdated)
4. Event appended to event store
5. Event triggers projections (read models)
6. Current state derived by replaying events
```

### Key Concepts

**Event**
- Represents something that happened
- Immutable once stored
- Contains all relevant data
- Named in past tense (OrderCreated, PaymentProcessed)

**Event Store**
- Append-only log of events
- Optimized for writing events
- Supports event replay
- Provides event ordering guarantees

**Aggregate**
- Group of related entities
- Consistency boundary
- Processes commands
- Generates events

**Command**
- Intent to change state
- Validated before processing
- Triggers event generation
- Named imperatively (CreateOrder, UpdateProduct)

**Projection**
- Read model built from events
- Optimized for queries
- Can be rebuilt from events
- Supports multiple projections

**Snapshot**
- Periodic state capture
- Optimizes event replay
- Contains state at specific point
- Used with event replay

## When to Use Event Sourcing

### Ideal Scenarios
✅ Audit trail requirements
✅ Temporal queries (what was the state at time T?)
✅ Complex business logic with state transitions
✅ When replaying events is valuable
✅ Systems requiring complete history
✅ Regulatory compliance needs
✅ Complex domain modeling

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ When query performance is critical
❌ Teams unfamiliar with the pattern
❌ When event schema changes frequently
❌ When storage costs are a concern
❌ Simple domains with minimal state

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── bankaccount/
│   │               ├── domain/
│   │               │   ├── event/
│   │               │   │   ├── AccountCreatedEvent.java
│   │               │   │   ├── MoneyDepositedEvent.java
│   │               │   │   ├── MoneyWithdrawnEvent.java
│   │               │   │   └── AccountClosedEvent.java
│   │               │   ├── command/
│   │               │   │   ├── CreateAccountCommand.java
│   │               │   │   ├── DepositMoneyCommand.java
│   │               │   │   ├── WithdrawMoneyCommand.java
│   │               │   │   └── CloseAccountCommand.java
│   │               │   ├── aggregate/
│   │               │   │   └── BankAccount.java
│   │               │   └── valueobject/
│   │               │       └── Money.java
│   │               ├── eventstore/
│   │               │   ├── EventStore.java
│   │               │   ├── InMemoryEventStore.java
│   │               │   └── EventStream.java
│   │               ├── snapshot/
│   │               │   ├── SnapshotStore.java
│   │               │   └── AccountSnapshot.java
│   │               ├── projection/
│   │               │   ├── AccountProjection.java
│   │               │   └── AccountReadModel.java
│   │               ├── service/
│   │               │   └── AccountService.java
│   │               └── controller/
│   │                   └── AccountController.java
```

### Domain Events
```java
package com.example.bankaccount.domain.event;

import java.time.LocalDateTime;
import java.util.UUID;

public interface DomainEvent {
    UUID getEventId();
    String getAggregateId();
    String getEventType();
    LocalDateTime getTimestamp();
    long getVersion();
}

package com.example.bankaccount.domain.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record AccountCreatedEvent(
    UUID eventId,
    String aggregateId,
    String eventType,
    LocalDateTime timestamp,
    long version,
    UUID accountId,
    String accountNumber,
    String ownerName,
    BigDecimal initialBalance,
    String currency
) implements DomainEvent {

    public AccountCreatedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "AccountCreated";
        if (timestamp == null) timestamp = LocalDateTime.now();
        if (aggregateId == null) aggregateId = accountId.toString();
    }

    public static AccountCreatedEvent create(
            UUID accountId,
            String accountNumber,
            String ownerName,
            BigDecimal initialBalance,
            String currency) {
        return new AccountCreatedEvent(
            UUID.randomUUID(),
            accountId.toString(),
            "AccountCreated",
            LocalDateTime.now(),
            0L,
            accountId,
            accountNumber,
            ownerName,
            initialBalance,
            currency
        );
    }
}

package com.example.bankaccount.domain.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record MoneyDepositedEvent(
    UUID eventId,
    String aggregateId,
    String eventType,
    LocalDateTime timestamp,
    long version,
    UUID accountId,
    BigDecimal amount,
    String description,
    BigDecimal newBalance
) implements DomainEvent {

    public MoneyDepositedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "MoneyDeposited";
        if (timestamp == null) timestamp = LocalDateTime.now();
        if (aggregateId == null) aggregateId = accountId.toString();
    }

    public static MoneyDepositedEvent create(
            UUID accountId,
            BigDecimal amount,
            String description,
            BigDecimal newBalance,
            long version) {
        return new MoneyDepositedEvent(
            UUID.randomUUID(),
            accountId.toString(),
            "MoneyDeposited",
            LocalDateTime.now(),
            version,
            accountId,
            amount,
            description,
            newBalance
        );
    }
}

package com.example.bankaccount.domain.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record MoneyWithdrawnEvent(
    UUID eventId,
    String aggregateId,
    String eventType,
    LocalDateTime timestamp,
    long version,
    UUID accountId,
    BigDecimal amount,
    String description,
    BigDecimal newBalance
) implements DomainEvent {

    public MoneyWithdrawnEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "MoneyWithdrawn";
        if (timestamp == null) timestamp = LocalDateTime.now();
        if (aggregateId == null) aggregateId = accountId.toString();
    }

    public static MoneyWithdrawnEvent create(
            UUID accountId,
            BigDecimal amount,
            String description,
            BigDecimal newBalance,
            long version) {
        return new MoneyWithdrawnEvent(
            UUID.randomUUID(),
            accountId.toString(),
            "MoneyWithdrawn",
            LocalDateTime.now(),
            version,
            accountId,
            amount,
            description,
            newBalance
        );
    }
}

package com.example.bankaccount.domain.event;

import java.time.LocalDateTime;
import java.util.UUID;

public record AccountClosedEvent(
    UUID eventId,
    String aggregateId,
    String eventType,
    LocalDateTime timestamp,
    long version,
    UUID accountId,
    String reason
) implements DomainEvent {

    public AccountClosedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "AccountClosed";
        if (timestamp == null) timestamp = LocalDateTime.now();
        if (aggregateId == null) aggregateId = accountId.toString();
    }

    public static AccountClosedEvent create(
            UUID accountId,
            String reason,
            long version) {
        return new AccountClosedEvent(
            UUID.randomUUID(),
            accountId.toString(),
            "AccountClosed",
            LocalDateTime.now(),
            version,
            accountId,
            reason
        );
    }
}
```

### Value Object
```java
package com.example.bankaccount.domain.valueobject;

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

    public boolean isGreaterThanOrEqual(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) >= 0;
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
```

### Aggregate
```java
package com.example.bankaccount.domain.aggregate;

import com.example.bankaccount.domain.event.*;
import com.example.bankaccount.domain.valueobject.Money;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class BankAccount {
    private UUID id;
    private String accountNumber;
    private String ownerName;
    private Money balance;
    private AccountStatus status;
    private long version;
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();

    public enum AccountStatus {
        ACTIVE, CLOSED, FROZEN
    }

    // Constructor for creating new account
    public BankAccount(UUID id, String accountNumber, String ownerName, 
                      Money initialBalance) {
        apply(AccountCreatedEvent.create(
            id, accountNumber, ownerName, 
            initialBalance.getAmount(), initialBalance.getCurrency()
        ));
    }

    // Constructor for replaying events
    public BankAccount(List<DomainEvent> events) {
        events.forEach(this::apply);
    }

    public void deposit(Money amount, String description) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("Account is not active");
        }
        if (amount.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }

        Money newBalance = balance.add(amount);
        apply(MoneyDepositedEvent.create(
            id, amount.getAmount(), description, newBalance.getAmount(), version + 1
        ));
    }

    public void withdraw(Money amount, String description) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("Account is not active");
        }
        if (amount.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (!balance.isGreaterThanOrEqual(amount)) {
            throw new IllegalArgumentException("Insufficient funds");
        }

        Money newBalance = balance.subtract(amount);
        apply(MoneyWithdrawnEvent.create(
            id, amount.getAmount(), description, newBalance.getAmount(), version + 1
        ));
    }

    public void close(String reason) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("Account is not active");
        }
        if (!balance.getAmount().equals(BigDecimal.ZERO)) {
            throw new IllegalStateException("Account must have zero balance to close");
        }

        apply(AccountClosedEvent.create(id, reason, version + 1));
    }

    private void apply(DomainEvent event) {
        if (event instanceof AccountCreatedEvent e) {
            applyAccountCreated(e);
        } else if (event instanceof MoneyDepositedEvent e) {
            applyMoneyDeposited(e);
        } else if (event instanceof MoneyWithdrawnEvent e) {
            applyMoneyWithdrawn(e);
        } else if (event instanceof AccountClosedEvent e) {
            applyAccountClosed(e);
        }

        this.version = event.getVersion();
        this.uncommittedEvents.add(event);
    }

    // Apply events without adding to uncommitted (for replay)
    public void applyEvent(DomainEvent event) {
        if (event instanceof AccountCreatedEvent e) {
            applyAccountCreated(e);
        } else if (event instanceof MoneyDepositedEvent e) {
            applyMoneyDeposited(e);
        } else if (event instanceof MoneyWithdrawnEvent e) {
            applyMoneyWithdrawn(e);
        } else if (event instanceof AccountClosedEvent e) {
            applyAccountClosed(e);
        }

        this.version = event.getVersion();
    }

    private void applyAccountCreated(AccountCreatedEvent event) {
        this.id = event.accountId();
        this.accountNumber = event.accountNumber();
        this.ownerName = event.ownerName();
        this.balance = new Money(event.initialBalance(), event.currency());
        this.status = AccountStatus.ACTIVE;
    }

    private void applyMoneyDeposited(MoneyDepositedEvent event) {
        this.balance = new Money(event.newBalance(), balance.getCurrency());
    }

    private void applyMoneyWithdrawn(MoneyWithdrawnEvent event) {
        this.balance = new Money(event.newBalance(), balance.getCurrency());
    }

    private void applyAccountClosed(AccountClosedEvent event) {
        this.status = AccountStatus.CLOSED;
    }

    public List<DomainEvent> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }

    // Getters
    public UUID getId() { return id; }
    public String getAccountNumber() { return accountNumber; }
    public String getOwnerName() { return ownerName; }
    public Money getBalance() { return balance; }
    public AccountStatus getStatus() { return status; }
    public long getVersion() { return version; }
}
```

### Event Store
```java
package com.example.bankaccount.eventstore;

import com.example.bankaccount.domain.event.DomainEvent;

import java.util.List;
import java.util.UUID;

public interface EventStore {
    void saveEvents(String aggregateId, List<DomainEvent> events, long expectedVersion);
    List<DomainEvent> getEvents(String aggregateId);
    List<DomainEvent> getEvents(String aggregateId, long fromVersion);
    EventStream getEventStream(String aggregateId);
}

package com.example.bankaccount.eventstore;

import com.example.bankaccount.domain.event.DomainEvent;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryEventStore implements EventStore {
    
    private final Map<String, List<DomainEvent>> eventStore = new ConcurrentHashMap<>();

    @Override
    public void saveEvents(String aggregateId, List<DomainEvent> events, 
                          long expectedVersion) {
        List<DomainEvent> currentEvents = eventStore.computeIfAbsent(
            aggregateId, k -> new ArrayList<>()
        );

        // Optimistic concurrency check
        if (!currentEvents.isEmpty() && 
            currentEvents.get(currentEvents.size() - 1).getVersion() != expectedVersion) {
            throw new ConcurrentModificationException(
                "Concurrent modification detected for aggregate: " + aggregateId
            );
        }

        currentEvents.addAll(events);
        System.out.println("Saved " + events.size() + " events for aggregate: " + aggregateId);
    }

    @Override
    public List<DomainEvent> getEvents(String aggregateId) {
        return new ArrayList<>(eventStore.getOrDefault(aggregateId, new ArrayList<>()));
    }

    @Override
    public List<DomainEvent> getEvents(String aggregateId, long fromVersion) {
        List<DomainEvent> allEvents = eventStore.getOrDefault(aggregateId, new ArrayList<>());
        return allEvents.stream()
                .filter(event -> event.getVersion() >= fromVersion)
                .toList();
    }

    @Override
    public EventStream getEventStream(String aggregateId) {
        List<DomainEvent> events = getEvents(aggregateId);
        long version = events.isEmpty() ? 0 : events.get(events.size() - 1).getVersion();
        return new EventStream(aggregateId, events, version);
    }
}

package com.example.bankaccount.eventstore;

import com.example.bankaccount.domain.event.DomainEvent;

import java.util.List;

public record EventStream(
    String aggregateId,
    List<DomainEvent> events,
    long currentVersion
) {
    public boolean isEmpty() {
        return events.isEmpty();
    }
}
```

### Snapshot Store
```java
package com.example.bankaccount.snapshot;

import com.example.bankaccount.domain.aggregate.BankAccount;
import java.util.Optional;
import java.util.UUID;

public interface SnapshotStore {
    void saveSnapshot(UUID accountId, AccountSnapshot snapshot);
    Optional<AccountSnapshot> getSnapshot(UUID accountId);
    void deleteSnapshot(UUID accountId);
}

package com.example.bankaccount.snapshot;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record AccountSnapshot(
    UUID accountId,
    String accountNumber,
    String ownerName,
    BigDecimal balance,
    String currency,
    BankAccount.AccountStatus status,
    long version,
    LocalDateTime timestamp
) {
    public static AccountSnapshot from(BankAccount account) {
        return new AccountSnapshot(
            account.getId(),
            account.getAccountNumber(),
            account.getOwnerName(),
            account.getBalance().getAmount(),
            account.getBalance().getCurrency(),
            account.getStatus(),
            account.getVersion(),
            LocalDateTime.now()
        );
    }
}

package com.example.bankaccount.snapshot;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

public class InMemorySnapshotStore implements SnapshotStore {
    
    private final Map<UUID, AccountSnapshot> snapshots = new HashMap<>();

    @Override
    public void saveSnapshot(UUID accountId, AccountSnapshot snapshot) {
        snapshots.put(accountId, snapshot);
        System.out.println("Saved snapshot for account: " + accountId);
    }

    @Override
    public Optional<AccountSnapshot> getSnapshot(UUID accountId) {
        return Optional.ofNullable(snapshots.get(accountId));
    }

    @Override
    public void deleteSnapshot(UUID accountId) {
        snapshots.remove(accountId);
    }
}
```

### Service Layer
```java
package com.example.bankaccount.service;

import com.example.bankaccount.domain.aggregate.BankAccount;
import com.example.bankaccount.domain.command.*;
import com.example.bankaccount.domain.valueobject.Money;
import com.example.bankaccount.eventstore.EventStore;
import com.example.bankaccount.eventstore.EventStream;
import com.example.bankaccount.snapshot.AccountSnapshot;
import com.example.bankaccount.snapshot.SnapshotStore;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Service
@Transactional
public class AccountService {
    
    private final EventStore eventStore;
    private final SnapshotStore snapshotStore;
    private static final long SNAPSHOT_INTERVAL = 10;

    public AccountService(EventStore eventStore, SnapshotStore snapshotStore) {
        this.eventStore = eventStore;
        this.snapshotStore = snapshotStore;
    }

    public UUID createAccount(CreateAccountCommand command) {
        Money initialBalance = new Money(
            command.initialBalance(),
            command.currency()
        );

        BankAccount account = new BankAccount(
            command.accountId(),
            command.accountNumber(),
            command.ownerName(),
            initialBalance
        );

        saveAccount(account);
        return account.getId();
    }

    public void depositMoney(DepositMoneyCommand command) {
        BankAccount account = loadAccount(command.accountId());
        Money amount = new Money(command.amount(), command.currency());
        
        account.deposit(amount, command.description());
        saveAccount(account);
    }

    public void withdrawMoney(WithdrawMoneyCommand command) {
        BankAccount account = loadAccount(command.accountId());
        Money amount = new Money(command.amount(), command.currency());
        
        account.withdraw(amount, command.description());
        saveAccount(account);
    }

    public void closeAccount(CloseAccountCommand command) {
        BankAccount account = loadAccount(command.accountId());
        account.close(command.reason());
        saveAccount(account);
    }

    public BankAccount getAccount(UUID accountId) {
        return loadAccount(accountId);
    }

    private BankAccount loadAccount(UUID accountId) {
        // Try to load from snapshot first
        Optional<AccountSnapshot> snapshot = snapshotStore.getSnapshot(accountId);
        
        if (snapshot.isPresent()) {
            AccountSnapshot snap = snapshot.get();
            BankAccount account = reconstructFromSnapshot(snap);
            
            // Replay events after snapshot
            List<DomainEvent> events = eventStore.getEvents(
                accountId.toString(), 
                snap.version() + 1
            );
            events.forEach(account::applyEvent);
            
            return account;
        } else {
            // Load all events
            EventStream eventStream = eventStore.getEventStream(accountId.toString());
            if (eventStream.isEmpty()) {
                throw new IllegalArgumentException("Account not found: " + accountId);
            }
            return new BankAccount(eventStream.events());
        }
    }

    private BankAccount reconstructFromSnapshot(AccountSnapshot snapshot) {
        Money balance = new Money(snapshot.balance(), snapshot.currency());
        BankAccount account = new BankAccount(
            snapshot.accountId(),
            snapshot.accountNumber(),
            snapshot.ownerName(),
            balance
        );
        // Set status and version directly
        account = new BankAccount(List.of()); // This is a simplification
        // In real implementation, we'd have a constructor that accepts snapshot data
        return account;
    }

    private void saveAccount(BankAccount account) {
        List<DomainEvent> uncommittedEvents = account.getUncommittedEvents();
        if (!uncommittedEvents.isEmpty()) {
            long expectedVersion = account.getVersion() - uncommittedEvents.size();
            eventStore.saveEvents(
                account.getId().toString(),
                uncommittedEvents,
                expectedVersion
            );
            account.markEventsAsCommitted();

            // Create snapshot periodically
            if (account.getVersion() % SNAPSHOT_INTERVAL == 0) {
                snapshotStore.saveSnapshot(
                    account.getId(),
                    AccountSnapshot.from(account)
                );
            }
        }
    }
}

// Command DTOs
package com.example.bankaccount.domain.command;

import java.math.BigDecimal;
import java.util.UUID;

public record CreateAccountCommand(
    UUID accountId,
    String accountNumber,
    String ownerName,
    BigDecimal initialBalance,
    String currency
) {}

public record DepositMoneyCommand(
    UUID accountId,
    BigDecimal amount,
    String currency,
    String description
) {}

public record WithdrawMoneyCommand(
    UUID accountId,
    BigDecimal amount,
    String currency,
    String description
) {}

public record CloseAccountCommand(
    UUID accountId,
    String reason
) {}
```

## Pros and Cons

### Advantages
✅ **Complete Audit Trail**: Every state change is recorded
✅ **Temporal Queries**: Query state at any point in time
✅ **Debugging**: Replay events to understand issues
✅ **Event Replay**: Rebuild state from events
✅ **Scalability**: Event store optimized for appends
✅ **Business Logic**: Complex logic in event handlers
✅ **Integration**: Events can be published to other systems

### Disadvantages
❌ **Complexity**: More complex than traditional CRUD
❌ **Event Schema**: Schema evolution is challenging
❌ **Query Performance**: Need projections for queries
❌ **Storage**: More storage than current state
❌ **Learning Curve**: Requires paradigm shift
❌ **Debugging**: Harder to debug event replay
❌ **Tooling**: Limited tooling support

## Real-World Use Cases

### Financial Trading System
- **Events**: TradeExecuted, PositionUpdated, RiskCalculated
- **Benefits**: Complete audit trail, regulatory compliance
- **Temporal**: What was the position at time T?

### Healthcare System
- **Events**: PatientAdmitted, TreatmentAdministered, DiagnosisRecorded
- **Benefits**: Complete patient history, regulatory requirements
- **Audit**: Who changed what and when

### E-commerce Platform
- **Events**: OrderCreated, PaymentProcessed, OrderShipped
- **Benefits**: Order lifecycle tracking, analytics
- **Replay**: Rebuild order state from events

## Interview Discussion Points

### Common Questions

**Q: How do you handle event schema evolution?**
A: Schema evolution strategies:
- Version events: Include version field
- Backward compatibility: New fields optional
- Event upcasting: Transform old events to new format
- Multiple event handlers: Support old and new versions
- Migration: Gradual rollout of new schema

**Q: How do you optimize event replay performance?**
A: Performance optimization strategies:
- Snapshots: Periodic state capture
- Event partitioning: By aggregate or time
- Caching: Cache reconstructed state
- Parallel replay: For independent aggregates
- Event compression: Reduce storage size

**Q: How do you handle event ordering?**
A: Event ordering strategies:
- Single event stream per aggregate
- Version numbers for ordering
- Timestamps for cross-aggregate ordering
- Causality tracking for distributed events
- Event sequencing within aggregate

**Q: What's the relationship between event sourcing and CQRS?**
A: They're complementary:
- Event sourcing: Store state as events
- CQRS: Separate read and write models
- Often used together: Event sourcing on write side
- Can use event sourcing without CQRS
- CQRS benefits from event sourcing's events

**Q: How do you handle duplicate events?**
A: Duplicate handling strategies:
- Idempotent event handlers
- Event deduplication by event ID
- Unique constraints in event store
- Exactly-once semantics
- Idempotent aggregates

### Advanced Topics

**Event Versioning**
- Semantic versioning for events
- Backward and forward compatibility
- Event upcasters
- Schema registry

**Snapshot Strategies**
- Time-based snapshots
- Version-based snapshots
- Hybrid approaches
- Snapshot compression

**Event Store Implementation**
- Append-only log
- Event partitioning
- Event replication
- Event retention policies

## Best Practices

### Event Design
1. **Use past tense**: OrderCreated, not CreateOrder
2. **Include all data**: Events should be self-contained
3. **Immutable**: Events should never change
4. **Version events**: Support schema evolution
5. **Unique IDs**: Every event needs unique identifier

### Aggregate Design
1. **Consistency boundary**: Aggregate enforces invariants
2. **Event generation**: Commands generate events
3. **Event application**: Events update state
4. **Version tracking**: Track aggregate version
5. **Optimistic concurrency**: Handle concurrent modifications

### Snapshot Design
1. **Periodic snapshots**: Create at regular intervals
2. **Version-based**: Create after N events
3. **Compression**: Compress snapshot data
4. **Validation**: Validate snapshot integrity
5. **Cleanup**: Remove old snapshots

## Related Patterns

- **CQRS**: Often used with event sourcing
- **Event-Driven Architecture**: Natural fit
- **Saga Pattern**: Can use event sourcing
- **Materialized View**: Projections as materialized views

## Technology Choices

### Event Stores
- **EventStoreDB**: Dedicated event store
- **Apache Kafka**: Event streaming platform
- **PostgreSQL**: With event store pattern
- **MongoDB**: Document store for events

### Java Frameworks
- **Axon Framework**: Event sourcing and CQRS
- **Spring Boot**: With custom implementation
- **Eventuate**: Event sourcing platform
- **Lagom**: Reactive microservices framework

---

**Next**: Learn about [Microservices Patterns](./microservices-patterns.md) for distributed system design
