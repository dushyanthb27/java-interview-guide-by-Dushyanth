# Saga Pattern

[Home](../../README.md) > Bonus > Architecture Patterns > Saga Pattern

---

## Overview
The Saga pattern is a design pattern for managing distributed transactions in microservices architectures. Unlike traditional ACID transactions that span multiple services, sagas break transactions into a sequence of local transactions, each updating data within a single service. If a step fails, compensating transactions undo the changes made by preceding steps, ensuring data consistency across services.

## Pattern Description

A saga is a sequence of transactions that updates each service and publishes an event or message to trigger the next step in the saga. If a step fails, sagas execute compensating transactions that undo the changes made by preceding transactions, returning the system to a consistent state.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Saga Orchestration                     │
│                                                           │
│  Step 1          Step 2          Step 3          Step 4  │
│  ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐  │
│  │ Txn  │ ────▶ │ Txn  │ ────▶ │ Txn  │ ────▶ │ Txn  │  │
│  │      │       │      │       │      │       │      │  │
│  └──┬───┘       └──┬───┘       └──┬───┘       └──┬───┘  │
│     │              │              │              │      │
│     ▼              ▼              ▼              ▼      │
│  ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐  │
│  │Comp  │◀──────│Comp  │◀──────│Comp  │◀──────│Comp  │  │
│  │      │       │      │       │      │       │      │  │
│  └──────┘       └──────┘       └──────┘       └──────┘  │
│                                                           │
└─────────────────────────────────────────────────────────┘

Orchestration Approaches:

1. Choreography (Decentralized)
   ┌─────────┐    Event    ┌─────────┐    Event    ┌─────────┐
   │Service A│ ──────────▶ │Service B│ ──────────▶ │Service C│
   └─────────┘             └─────────┘             └─────────┘
        │                       │                       │
        ▼                       ▼                       ▼
   ┌─────────┐    Event    ┌─────────┐    Event    ┌─────────┐
   │Service A│◀────────── │Service B│◀────────── │Service C│
   └─────────┘  (Compensate)└─────────┘  (Compensate)└─────────┘

2. Orchestration (Centralized)
   ┌─────────────────┐
   │ Saga Orchestrator│
   └────────┬─────────┘
            │
     ┌──────┴──────┬──────────┬──────────┐
     ▼             ▼          ▼          ▼
┌─────────┐  ┌─────────┐ ┌─────────┐ ┌─────────┐
│Service A│  │Service B│ │Service C│ │Service D│
└─────────┘  └─────────┘ └─────────┘ └─────────┘
```

### Key Concepts

**Local Transaction**
- Updates data within a single service
- Uses ACID transactions within the service boundary
- Publishes event or message upon completion
- Has corresponding compensating transaction

**Compensating Transaction**
- Undoes the changes made by a local transaction
- Executed when a saga step fails
- Should be idempotent
- May not perfectly reverse original transaction

**Saga Execution**
- Sequence of local transactions
- Can be synchronous or asynchronous
- Can be orchestrated or choreographed
- Must handle failures gracefully

**Saga Types**
- **Choreography**: Decentralized, event-driven
- **Orchestration**: Centralized coordinator

## When to Use Saga Pattern

### Ideal Scenarios
✅ Distributed transactions across microservices
✅ Long-running business processes
✅ When ACID transactions aren't available
✅ Systems requiring eventual consistency
✅ Complex business workflows with multiple steps
✅ E-commerce order processing
✅ Travel booking systems

### When to Avoid
❌ Single service applications (use ACID)
❌ When strong consistency is required
❌ Simple transactions with few steps
❌ Teams unfamiliar with distributed systems
❌ When compensating logic is complex

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── travelbooking/
│   │               ├── saga/
│   │               │   ├── orchestrator/
│   │               │   │   ├── BookingSagaOrchestrator.java
│   │               │   │   └── SagaState.java
│   │               │   ├── choreography/
│   │               │   │   ├── FlightBookingSaga.java
│   │               │   │   └── HotelBookingSaga.java
│   │               │   └── events/
│   │               │       ├── FlightBookedEvent.java
│   │               │       ├── FlightBookingFailedEvent.java
│   │               │       ├── HotelBookedEvent.java
│   │               │       └── HotelBookingFailedEvent.java
│   │               ├── service/
│   │               │   ├── flight/
│   │               │   │   ├── FlightService.java
│   │               │   │   └── FlightBooking.java
│   │               │   ├── hotel/
│   │               │   │   ├── HotelService.java
│   │               │   │   └── HotelBooking.java
│   │               │   └── payment/
│   │               │       ├── PaymentService.java
│   │               │       └── Payment.java
│   │               └── controller/
│   │                   └── BookingController.java
```

### Orchestration Approach

```java
// Saga State
package com.example.travelbooking.saga.orchestrator;

import java.time.LocalDateTime;
import java.util.UUID;

public class SagaState {
    private UUID sagaId;
    private UUID bookingId;
    private String currentStep;
    private SagaStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String errorMessage;

    public enum SagaStatus {
        STARTED, FLIGHT_BOOKED, HOTEL_BOOKED, PAYMENT_PROCESSED, 
        COMPLETED, FAILED, COMPENSATING, COMPENSATED
    }

    public SagaState(UUID bookingId) {
        this.sagaId = UUID.randomUUID();
        this.bookingId = bookingId;
        this.currentStep = "START";
        this.status = SagaStatus.STARTED;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public void advanceTo(String step, SagaStatus status) {
        this.currentStep = step;
        this.status = status;
        this.updatedAt = LocalDateTime.now();
    }

    public void fail(String errorMessage) {
        this.status = SagaStatus.FAILED;
        this.errorMessage = errorMessage;
        this.updatedAt = LocalDateTime.now();
    }

    public void startCompensation() {
        this.status = SagaStatus.COMPENSATING;
        this.updatedAt = LocalDateTime.now();
    }

    public void completeCompensation() {
        this.status = SagaStatus.COMPENSATED;
        this.updatedAt = LocalDateTime.now();
    }

    // Getters
    public UUID getSagaId() { return sagaId; }
    public UUID getBookingId() { return bookingId; }
    public String getCurrentStep() { return currentStep; }
    public SagaStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public String getErrorMessage() { return errorMessage; }
}

// Saga Orchestrator
package com.example.travelbooking.saga.orchestrator;

import com.example.travelbooking.service.flight.FlightService;
import com.example.travelbooking.service.hotel.HotelService;
import com.example.travelbooking.service.payment.PaymentService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.UUID;

@Service
public class BookingSagaOrchestrator {
    
    private final FlightService flightService;
    private final HotelService hotelService;
    private final PaymentService paymentService;
    private final SagaStateRepository sagaStateRepository;

    public BookingSagaOrchestrator(FlightService flightService,
                                   HotelService hotelService,
                                   PaymentService paymentService,
                                   SagaStateRepository sagaStateRepository) {
        this.flightService = flightService;
        this.hotelService = hotelService;
        this.paymentService = paymentService;
        this.sagaStateRepository = sagaStateRepository;
    }

    @Transactional
    public UUID executeBookingSaga(BookingRequest request) {
        // Initialize saga state
        SagaState sagaState = new SagaState(request.bookingId());
        sagaStateRepository.save(sagaState);

        try {
            // Step 1: Book Flight
            UUID flightBookingId = bookFlight(sagaState, request);
            sagaState.advanceTo("FLIGHT_BOOKED", SagaState.SagaStatus.FLIGHT_BOOKED);
            sagaStateRepository.save(sagaState);

            // Step 2: Book Hotel
            UUID hotelBookingId = bookHotel(sagaState, request);
            sagaState.advanceTo("HOTEL_BOOKED", SagaState.SagaStatus.HOTEL_BOOKED);
            sagaStateRepository.save(sagaState);

            // Step 3: Process Payment
            UUID paymentId = processPayment(sagaState, request, 
                                          request.totalAmount());
            sagaState.advanceTo("PAYMENT_PROCESSED", SagaState.SagaStatus.PAYMENT_PROCESSED);
            sagaStateRepository.save(sagaState);

            // Complete saga
            sagaState.advanceTo("COMPLETED", SagaState.SagaStatus.COMPLETED);
            sagaStateRepository.save(sagaState);

            return sagaState.getSagaId();

        } catch (Exception e) {
            // Start compensation
            compensate(sagaState);
            throw new RuntimeException("Booking failed: " + e.getMessage(), e);
        }
    }

    private UUID bookFlight(SagaState sagaState, BookingRequest request) {
        try {
            return flightService.bookFlight(
                request.flightId(),
                request.passengerName(),
                request.travelDate()
            );
        } catch (Exception e) {
            sagaState.fail("Flight booking failed: " + e.getMessage());
            throw e;
        }
    }

    private UUID bookHotel(SagaState sagaState, BookingRequest request) {
        try {
            return hotelService.bookHotel(
                request.hotelId(),
                request.passengerName(),
                request.checkInDate(),
                request.checkOutDate()
            );
        } catch (Exception e) {
            sagaState.fail("Hotel booking failed: " + e.getMessage());
            throw e;
        }
    }

    private UUID processPayment(SagaState sagaState, BookingRequest request, 
                                BigDecimal amount) {
        try {
            return paymentService.processPayment(
                request.customerId(),
                amount,
                "TRAVEL_BOOKING"
            );
        } catch (Exception e) {
            sagaState.fail("Payment processing failed: " + e.getMessage());
            throw e;
        }
    }

    private void compensate(SagaState sagaState) {
        sagaState.startCompensation();
        sagaStateRepository.save(sagaState);

        try {
            // Compensate in reverse order
            if (sagaState.getStatus() == SagaState.SagaStatus.PAYMENT_PROCESSED ||
                sagaState.getStatus() == SagaState.SagaStatus.FAILED) {
                // Payment compensation would go here
                System.out.println("Compensating payment");
            }

            if (sagaState.getStatus() == SagaState.SagaStatus.HOTEL_BOOKED ||
                sagaState.getStatus() == SagaState.SagaStatus.FAILED) {
                hotelService.cancelBooking(sagaState.getBookingId());
                System.out.println("Compensating hotel booking");
            }

            if (sagaState.getStatus() == SagaState.SagaStatus.FLIGHT_BOOKED ||
                sagaState.getStatus() == SagaState.SagaStatus.FAILED) {
                flightService.cancelBooking(sagaState.getBookingId());
                System.out.println("Compensating flight booking");
            }

            sagaState.completeCompensation();
            sagaStateRepository.save(sagaState);

        } catch (Exception e) {
            System.err.println("Compensation failed: " + e.getMessage());
            // Manual intervention required
        }
    }

    public record BookingRequest(
        UUID bookingId,
        UUID customerId,
        String passengerName,
        UUID flightId,
        UUID hotelId,
        String travelDate,
        String checkInDate,
        String checkOutDate,
        BigDecimal totalAmount
    ) {}
}

// Saga State Repository
package com.example.travelbooking.saga.orchestrator;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;
import java.util.UUID;

@Repository
interface SagaStateRepository extends JpaRepository<SagaState, UUID> {
    Optional<SagaState> findByBookingId(UUID bookingId);
}
```

### Choreography Approach

```java
// Saga Events
package com.example.travelbooking.saga.events;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record FlightBookedEvent(
    UUID sagaId,
    UUID bookingId,
    UUID flightBookingId,
    String passengerName,
    String flightNumber,
    LocalDateTime travelDate,
    BigDecimal amount
) {
    public static FlightBookedEvent create(UUID sagaId, UUID bookingId,
                                           UUID flightBookingId, String passengerName,
                                           String flightNumber, LocalDateTime travelDate,
                                           BigDecimal amount) {
        return new FlightBookedEvent(
            sagaId, bookingId, flightBookingId, passengerName,
            flightNumber, travelDate, amount
        );
    }
}

package com.example.travelbooking.saga.events;

import java.time.LocalDateTime;
import java.util.UUID;

public record FlightBookingFailedEvent(
    UUID sagaId,
    UUID bookingId,
    String reason,
    LocalDateTime failedAt
) {
    public static FlightBookingFailedEvent create(UUID sagaId, UUID bookingId,
                                                   String reason) {
        return new FlightBookingFailedEvent(
            sagaId, bookingId, reason, LocalDateTime.now()
        );
    }
}

package com.example.travelbooking.saga.events;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record HotelBookedEvent(
    UUID sagaId,
    UUID bookingId,
    UUID hotelBookingId,
    String hotelName,
    LocalDateTime checkInDate,
    LocalDateTime checkOutDate,
    BigDecimal amount
) {
    public static HotelBookedEvent create(UUID sagaId, UUID bookingId,
                                          UUID hotelBookingId, String hotelName,
                                          LocalDateTime checkInDate,
                                          LocalDateTime checkOutDate,
                                          BigDecimal amount) {
        return new HotelBookedEvent(
            sagaId, bookingId, hotelBookingId, hotelName,
            checkInDate, checkOutDate, amount
        );
    }
}

package com.example.travelbooking.saga.events;

import java.time.LocalDateTime;
import java.util.UUID;

public record HotelBookingFailedEvent(
    UUID sagaId,
    UUID bookingId,
    String reason,
    LocalDateTime failedAt
) {
    public static HotelBookingFailedEvent create(UUID sagaId, UUID bookingId,
                                                  String reason) {
        return new HotelBookingFailedEvent(
            sagaId, bookingId, reason, LocalDateTime.now()
        );
    }
}

// Flight Service with Saga
package com.example.travelbooking.service.flight;

import com.example.travelbooking.saga.events.FlightBookedEvent;
import com.example.travelbooking.saga.events.FlightBookingFailedEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
@Transactional
public class FlightService {
    
    private final FlightBookingRepository flightBookingRepository;
    private final ApplicationEventPublisher eventPublisher;

    public FlightService(FlightBookingRepository flightBookingRepository,
                        ApplicationEventPublisher eventPublisher) {
        this.flightBookingRepository = flightBookingRepository;
        this.eventPublisher = eventPublisher;
    }

    public UUID bookFlight(UUID sagaId, UUID bookingId, String passengerName,
                          String flightNumber, LocalDateTime travelDate) {
        try {
            // Book flight
            FlightBooking booking = new FlightBooking(
                UUID.randomUUID(),
                sagaId,
                bookingId,
                passengerName,
                flightNumber,
                travelDate,
                new BigDecimal("500.00"), // Would calculate based on flight
                BookingStatus.CONFIRMED
            );

            FlightBooking saved = flightBookingRepository.save(booking);

            // Publish success event
            FlightBookedEvent event = FlightBookedEvent.create(
                sagaId,
                bookingId,
                saved.getId(),
                passengerName,
                flightNumber,
                travelDate,
                saved.getAmount()
            );
            eventPublisher.publishEvent(event);

            return saved.getId();

        } catch (Exception e) {
            // Publish failure event
            FlightBookingFailedEvent event = FlightBookingFailedEvent.create(
                sagaId,
                bookingId,
                e.getMessage()
            );
            eventPublisher.publishEvent(event);
            throw e;
        }
    }

    public void cancelBooking(UUID bookingId) {
        FlightBooking booking = flightBookingRepository.findByBookingId(bookingId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Flight booking not found: " + bookingId
                ));

        booking.setStatus(BookingStatus.CANCELLED);
        flightBookingRepository.save(booking);
    }

    public enum BookingStatus {
        PENDING, CONFIRMED, CANCELLED
    }
}

// Hotel Service with Saga
package com.example.travelbooking.service.hotel;

import com.example.travelbooking.saga.events.FlightBookingFailedEvent;
import com.example.travelbooking.saga.events.HotelBookedEvent;
import com.example.travelbooking.saga.events.HotelBookingFailedEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
@Transactional
public class HotelService {
    
    private final HotelBookingRepository hotelBookingRepository;
    private final ApplicationEventPublisher eventPublisher;

    public HotelService(HotelBookingRepository hotelBookingRepository,
                       ApplicationEventPublisher eventPublisher) {
        this.hotelBookingRepository = hotelBookingRepository;
        this.eventPublisher = eventPublisher;
    }

    public UUID bookHotel(UUID sagaId, UUID bookingId, String passengerName,
                         String hotelName, LocalDateTime checkInDate,
                         LocalDateTime checkOutDate) {
        try {
            // Book hotel
            HotelBooking booking = new HotelBooking(
                UUID.randomUUID(),
                sagaId,
                bookingId,
                passengerName,
                hotelName,
                checkInDate,
                checkOutDate,
                new BigDecimal("300.00"), // Would calculate based on hotel
                BookingStatus.CONFIRMED
            );

            HotelBooking saved = hotelBookingRepository.save(booking);

            // Publish success event
            HotelBookedEvent event = HotelBookedEvent.create(
                sagaId,
                bookingId,
                saved.getId(),
                hotelName,
                checkInDate,
                checkOutDate,
                saved.getAmount()
            );
            eventPublisher.publishEvent(event);

            return saved.getId();

        } catch (Exception e) {
            // Publish failure event
            HotelBookingFailedEvent event = HotelBookingFailedEvent.create(
                sagaId,
                bookingId,
                e.getMessage()
            );
            eventPublisher.publishEvent(event);
            throw e;
        }
    }

    public void cancelBooking(UUID bookingId) {
        HotelBooking booking = hotelBookingRepository.findByBookingId(bookingId)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Hotel booking not found: " + bookingId
                ));

        booking.setStatus(BookingStatus.CANCELLED);
        hotelBookingRepository.save(booking);
    }

    // Listen for flight booking failures and compensate
    @org.springframework.context.event.EventListener
    public void handleFlightBookingFailed(FlightBookingFailedEvent event) {
        // Cancel hotel booking if flight failed
        hotelBookingRepository.findByBookingId(event.bookingId())
                .ifPresent(booking -> {
                    booking.setStatus(BookingStatus.CANCELLED);
                    hotelBookingRepository.save(booking);
                });
    }

    public enum BookingStatus {
        PENDING, CONFIRMED, CANCELLED
    }
}
```

### Service Models

```java
// Flight Booking Model
package com.example.travelbooking.service.flight;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class FlightBooking {
    private UUID id;
    private UUID sagaId;
    private UUID bookingId;
    private String passengerName;
    private String flightNumber;
    private LocalDateTime travelDate;
    private BigDecimal amount;
    private BookingStatus status;

    public FlightBooking(UUID id, UUID sagaId, UUID bookingId, String passengerName,
                        String flightNumber, LocalDateTime travelDate,
                        BigDecimal amount, BookingStatus status) {
        this.id = id;
        this.sagaId = sagaId;
        this.bookingId = bookingId;
        this.passengerName = passengerName;
        this.flightNumber = flightNumber;
        this.travelDate = travelDate;
        this.amount = amount;
        this.status = status;
    }

    // Getters and setters
    public UUID getId() { return id; }
    public UUID getSagaId() { return sagaId; }
    public UUID getBookingId() { return bookingId; }
    public String getPassengerName() { return passengerName; }
    public String getFlightNumber() { return flightNumber; }
    public LocalDateTime getTravelDate() { return travelDate; }
    public BigDecimal getAmount() { return amount; }
    public BookingStatus getStatus() { return status; }
    public void setStatus(BookingStatus status) { this.status = status; }
}

// Hotel Booking Model
package com.example.travelbooking.service.hotel;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class HotelBooking {
    private UUID id;
    private UUID sagaId;
    private UUID bookingId;
    private String passengerName;
    private String hotelName;
    private LocalDateTime checkInDate;
    private LocalDateTime checkOutDate;
    private BigDecimal amount;
    private BookingStatus status;

    public HotelBooking(UUID id, UUID sagaId, UUID bookingId, String passengerName,
                       String hotelName, LocalDateTime checkInDate,
                       LocalDateTime checkOutDate, BigDecimal amount,
                       BookingStatus status) {
        this.id = id;
        this.sagaId = sagaId;
        this.bookingId = bookingId;
        this.passengerName = passengerName;
        this.hotelName = hotelName;
        this.checkInDate = checkInDate;
        this.checkOutDate = checkOutDate;
        this.amount = amount;
        this.status = status;
    }

    // Getters and setters
    public UUID getId() { return id; }
    public UUID getSagaId() { return sagaId; }
    public UUID getBookingId() { return bookingId; }
    public String getPassengerName() { return passengerName; }
    public String getHotelName() { return hotelName; }
    public LocalDateTime getCheckInDate() { return checkInDate; }
    public LocalDateTime getCheckOutDate() { return checkOutDate; }
    public BigDecimal getAmount() { return amount; }
    public BookingStatus getStatus() { return status; }
    public void setStatus(BookingStatus status) { this.status = status; }
}
```

## Pros and Cons

### Advantages
✅ **Distributed Transactions**: Manages transactions across services
✅ **Eventual Consistency**: Achieves consistency without distributed locks
✅ **Scalability**: No single point of coordination
✅ **Resilience**: Handles failures gracefully with compensation
✅ **Flexibility**: Can adapt to changing business processes
✅ **Auditable**: Each step is logged and traceable

### Disadvantages
❌ **Complexity**: More complex than ACID transactions
❌ **Eventual Consistency**: Data is not immediately consistent
❌ **Compensation Logic**: Complex to implement correctly
❌ **Debugging**: Harder to debug distributed failures
❌ **Temporal Coupling**: Steps may have time dependencies
❌ **Idempotency**: All operations must be idempotent

## Real-World Use Cases

### E-commerce Order Processing
- **Steps**: Reserve inventory → Process payment → Ship order → Send confirmation
- **Compensation**: Release inventory → Refund payment → Cancel shipment
- **Benefits**: Reliable order processing across services

### Travel Booking System
- **Steps**: Book flight → Book hotel → Book car → Process payment
- **Compensation**: Cancel car → Cancel hotel → Cancel flight → Refund
- **Benefits**: Coordinated booking across providers

### Financial Trading
- **Steps**: Validate trade → Execute trade → Update positions → Settle
- **Compensation**: Reverse trade → Restore positions → Cancel settlement
- **Benefits**: Reliable trade execution with rollback

## Interview Discussion Points

### Common Questions

**Q: What's the difference between choreography and orchestration?**
A: Choreography: Decentralized, event-driven, services communicate directly. Each service knows when to trigger the next step. Orchestration: Centralized coordinator manages the saga. Orchestrator knows the entire flow and tells services what to do. Choreography is more flexible but harder to debug; orchestration is clearer but creates a central point of coordination.

**Q: How do you handle saga failures?**
A: Failure handling strategies:
- Compensating transactions undo completed steps
- Retry mechanisms for transient failures
- Manual intervention for unrecoverable failures
- Monitoring and alerting for failed sagas
- Dead letter queues for failed events

**Q: How do you ensure idempotency in sagas?**
A: Idempotency strategies:
- Unique request IDs for each operation
- Check if operation already completed before executing
- Use database constraints to prevent duplicates
- Design compensating transactions to be idempotent
- Track saga state to prevent duplicate processing

**Q: What happens if compensation fails?**
A: Compensation failure handling:
- Retry compensation with exponential backoff
- Manual intervention for critical failures
- Alerting and monitoring for failed compensations
- Business processes to handle orphaned transactions
- Periodic cleanup jobs for stuck sagas

**Q: How do you handle long-running sagas?**
A: Long-running saga strategies:
- Use persistent saga state storage
- Implement timeout mechanisms
- Support saga resumption after failures
- Use message queues for reliability
- Monitor saga execution time

### Advanced Topics

**Saga Patterns**
- **Sequential**: Steps execute one after another
- **Parallel**: Multiple steps execute simultaneously
- **Mixed**: Combination of sequential and parallel

**Saga State Management**
- Database storage for saga state
- Event sourcing for saga history
- Distributed coordination services
- Timeout and retry mechanisms

**Saga Monitoring**
- Track saga execution metrics
- Monitor compensation success rates
- Alert on stuck or failed sagas
- Audit trail for compliance

## Best Practices

### Saga Design
1. **Keep sagas short**: Minimize number of steps
2. **Design compensations**: Every step needs compensation
3. **Make idempotent**: All operations should be idempotent
4. **Handle timeouts**: Define timeouts for each step
5. **Log everything**: Comprehensive logging for debugging

### Orchestration
1. **Centralized logic**: Clear saga flow in orchestrator
2. **State persistence**: Persist saga state
3. **Error handling**: Robust error handling and retry
4. **Monitoring**: Track saga execution
5. **Testing**: Test all failure scenarios

### Choreography
1. **Event design**: Clear, self-contained events
2. **Event ordering**: Ensure events are processed in order
3. **Dead letter queues**: Handle failed events
4. **Event versioning**: Support schema evolution
5. **Monitoring**: Track event flow

## Related Patterns

- **Event-Driven Architecture**: Often used with sagas
- **CQRS**: Can be combined with sagas
- **Event Sourcing**: Store saga events for audit trail
- **Circuit Breaker**: Handle service failures

## Technology Choices

### Orchestration Frameworks
- **Camunda**: BPMN-based workflow engine
- **Apache Airflow**: Workflow orchestration
- **Temporal**: Durable execution framework
- **AWS Step Functions**: Serverless workflow

### Message Brokers
- **Apache Kafka**: High-throughput event streaming
- **RabbitMQ**: Reliable message routing
- **Amazon SQS/SNS**: Managed cloud messaging
- **Apache Pulsar**: Multi-tenant messaging

### Java Frameworks
- **Spring Boot**: With custom saga implementation
- **Axon Framework**: Saga support with DDD
- **Seata**: Distributed transaction framework
- **Narayana**: JTA transaction manager

---

**Next**: Learn about [Event Sourcing](./event-sourcing.md) for event-based state management
