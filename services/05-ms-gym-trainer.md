# Trainer Service

> **Tech:** Java 26 (Spring Boot 4) | **DB:** PostgreSQL | **Port:** 50051 (gRPC) / 8080 (REST)

## Responsibilities

- Trainer profile CRUD (admin creates account, trainer updates profile)
- Availability calendar management (trainer sets free slots)
- Booking lifecycle: `REQUESTED → ACCEPTED / REJECTED → COMPLETED / CANCELLED`
- Trainer search with filters (specialty, price range, availability, gym location)
- Coaching history (trainer views past bookings)
- Trainer is **paid by gym** (salary) — no direct customer-to-trainer payment split

---

## Booking Flow

```mermaid
sequenceDiagram
    participant C as Customer App
    participant TS as Trainer Service
    participant PS as Payment Service
    participant KF as Kafka
    participant NS as Notification Service
    participant T as Trainer App

    C->>TS: SearchTrainers(gym_id, date, specialty)
    TS-->>C: [{trainer_id, name, price, available_slots}]

    C->>TS: CreateBooking(trainer_id, slot, notes)
    TS->>TS: Check slot availability (lock with SELECT FOR UPDATE)
    TS->>TS: Create booking (status=PENDING_PAYMENT)
    TS-->>C: {booking_id, amount, status: PENDING_PAYMENT}

    C->>PS: InitiatePayment(type=TRAINER_BOOKING, ref=booking_id)
    PS-->>C: {payment_url}
    C->>C: Pay via Momo/ZaloPay

    PS->>KF: payment.completed.v1 (type=TRAINER_BOOKING)
    KF-->>TS: Consume payment.completed.v1
    TS->>TS: Update booking → REQUESTED (paid, awaiting trainer)
    TS->>KF: booking.requested

    KF-->>NS: Send push notification to trainer
    NS->>T: "New booking request from Nguyen Van A"

    T->>TS: AcceptBooking(booking_id)
    TS->>TS: Update booking → ACCEPTED
    TS->>KF: booking.accepted

    KF-->>NS: Notify customer
    NS->>C: "Your booking with Trainer B is confirmed!"

    Note over TS: After session completes
    T->>TS: CompleteBooking(booking_id, notes)
    TS->>TS: Update booking → COMPLETED
    TS->>KF: booking.completed
```

### Booking State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING_PAYMENT: Customer selects slot
    PENDING_PAYMENT --> REQUESTED: payment.completed.v1
    PENDING_PAYMENT --> EXPIRED: Payment timeout (30 min, scheduled job)

    REQUESTED --> ACCEPTED: Trainer accepts
    REQUESTED --> REJECTED: Trainer rejects
    REQUESTED --> AUTO_REJECTED: Trainer non-response timeout (4 hours, scheduled job)

    ACCEPTED --> COMPLETED: Trainer marks done
    ACCEPTED --> CANCELLED: Customer or Trainer cancels

    REJECTED --> [*]: Trigger refund
    AUTO_REJECTED --> [*]: Trigger refund + notify customer
    CANCELLED --> [*]: Trigger refund (policy-based)
    EXPIRED --> [*]: Release slot, no charge
```

---

## Booking Timeout Enforcement

### Payment Timeout (PENDING_PAYMENT, 30 min)

```
Scheduled Job: BookingPaymentTimeoutJob
  Schedule: every 5 minutes
  Query: SELECT * FROM bookings
         WHERE status = 'PENDING_PAYMENT'
         AND created_at < NOW() - INTERVAL '30 minutes'

  Action per expired booking:
    1. Update status → EXPIRED
    2. Release slot (remove from availability block)
    3. No refund needed (customer never paid)
    4. Publish: booking.expired {booking_id, trainer_id, slot}
```

### Trainer Non-Response Timeout (REQUESTED, 4 hours)

```
Scheduled Job: BookingTrainerTimeoutJob
  Schedule: every 15 minutes
  Query: SELECT * FROM bookings
         WHERE status = 'REQUESTED'
         AND updated_at < NOW() - INTERVAL '4 hours'

  Action per timed-out booking:
    1. Update status → AUTO_REJECTED
    2. Release slot
    3. Publish: booking.auto-rejected {booking_id, customer_id, trainer_id}
       → Notification Service sends: "HLV chua phan hoi. Da hoan tien."
       → Payment Service triggers refund via payment.refunded event
```

---

## Booking Cancellation Policy

```mermaid
sequenceDiagram
    actor U as Customer or Trainer
    participant TS as Trainer Service
    participant KF as Kafka
    participant PS as Payment Service
    participant NS as Notification Service

    U->>TS: gRPC CancelBooking(booking_id, reason)
    TS->>TS: Validate: status must be ACCEPTED

    TS->>TS: Check cancellation policy
    alt More than 24h before scheduled_at
        TS->>TS: Full refund
    else 2-24h before scheduled_at
        TS->>TS: 50% refund
    else Less than 2h before scheduled_at
        TS->>TS: No refund (customer) / Full refund (if trainer cancels)
    end

    TS->>TS: Update booking → CANCELLED
    TS->>TS: Release slot
    TS->>KF: booking.cancelled {booking_id, cancelled_by, reason, refund_percentage}

    KF-->>PS: Trigger refund with specified percentage
    KF-->>NS: Notify other party (if customer cancelled → notify trainer, vice versa)
```

### Cancellation Rules

| Who Cancels | Timing | Refund |
|-------------|--------|--------|
| Customer | > 24h before session | 100% |
| Customer | 2-24h before session | 50% |
| Customer | < 2h before session | 0% |
| Trainer | Any time | 100% (gym absorbs cost) |
| System (auto-reject timeout) | N/A | 100% |

---

## Trainer Suspension Cascade

```mermaid
sequenceDiagram
    actor A as Admin
    participant TS as Trainer Service
    participant KF as Kafka
    participant PS as Payment Service
    participant NS as Notification Service

    A->>TS: gRPC SuspendTrainer(trainer_id)
    TS->>TS: Update trainer status → SUSPENDED

    TS->>TS: Find all bookings WHERE trainer_id = ?<br/>AND status IN (PENDING_PAYMENT, REQUESTED, ACCEPTED)<br/>AND scheduled_at > NOW()

    loop For each affected booking
        TS->>TS: Update booking → CANCELLED (reason: "Trainer suspended")
        TS->>KF: booking.cancelled {booking_id, cancelled_by: SYSTEM, reason, refund: 100%}
    end

    TS->>KF: trainer.suspended {trainer_id, gym_id, affected_booking_count}

    KF-->>PS: Trigger 100% refund for each cancelled booking
    KF-->>NS: Notify each affected customer:<br/>"Lich tap voi HLV X da bi huy. Da hoan tien."
```

---

## Data Model

```mermaid
erDiagram
    TRAINERS {
        uuid id PK
        uuid user_id FK "from Identity Service"
        uuid gym_id FK
        varchar display_name
        varchar avatar_url
        text bio
        text[] specialties "strength, cardio, yoga, crossfit"
        text[] certifications "ACE, NASM, etc"
        bigint hourly_rate_vnd
        varchar status "ACTIVE | SUSPENDED | INACTIVE"
        timestamp created_at
    }

    TRAINER_AVAILABILITY {
        uuid id PK
        uuid trainer_id FK
        int day_of_week "0=Mon, 6=Sun"
        time start_time
        time end_time
        boolean is_recurring "true = every week"
        date specific_date "null if recurring"
        boolean is_blocked "trainer marks off"
    }

    BOOKINGS {
        uuid id PK
        uuid customer_id FK
        uuid trainer_id FK
        uuid gym_id FK
        timestamp scheduled_at
        int duration_minutes "default 60"
        bigint price_vnd
        varchar status "PENDING_PAYMENT | REQUESTED | ACCEPTED | REJECTED | COMPLETED | CANCELLED"
        uuid payment_id
        text customer_notes
        text trainer_notes "post-session"
        varchar rejection_reason
        timestamp created_at
        timestamp updated_at
    }

    TRAINERS ||--o{ TRAINER_AVAILABILITY : "has"
    TRAINERS ||--o{ BOOKINGS : "receives"
```

---

## Availability Logic

```
Slot = 1 hour block

Available slots for Trainer X on Date Y:
  1. Get recurring availability for day_of_week(Y)
  2. Get specific_date overrides for Y
  3. Subtract blocked slots
  4. Subtract existing ACCEPTED/REQUESTED bookings on Y
  5. Return remaining free slots

Example:
  Recurring: Mon 08:00-17:00 (9 slots)
  Blocked: Mon 12:00-13:00 (lunch)
  Booked: Mon 09:00-10:00, Mon 14:00-15:00
  Available: [08:00, 10:00, 11:00, 13:00, 15:00, 16:00] → 6 slots
```

---

## Kafka Events

### Published

| Topic | Key | Payload |
|-------|-----|---------|
| `booking.requested` | `booking_id` | `{booking_id, customer_id, trainer_id, scheduled_at, gym_id}` |
| `booking.accepted` | `booking_id` | `{booking_id, customer_id, trainer_id, gym_id}` |
| `booking.rejected` | `booking_id` | `{booking_id, customer_id, trainer_id, reason, gym_id}` |
| `booking.completed` | `booking_id` | `{booking_id, trainer_id, customer_id, duration_min, gym_id}` |
| `booking.cancelled` | `booking_id` | `{booking_id, customer_id, trainer_id, cancelled_by, reason, refund_percentage, gym_id}` |
| `booking.expired` | `booking_id` | `{booking_id, trainer_id, slot}` |
| `booking.auto-rejected` | `booking_id` | `{booking_id, customer_id, trainer_id, gym_id}` |
| `trainer.created` | `trainer_id` | `{trainer_id, user_id, gym_id, display_name}` |
| `trainer.suspended` | `trainer_id` | `{trainer_id, gym_id, affected_booking_count}` |

### Consumed

| Topic | Action |
|-------|--------|
| `payment.completed.v1` (type=TRAINER_BOOKING) | Move booking from PENDING_PAYMENT → REQUESTED |
| `payment.refunded` (ref=booking_id) | Move booking to CANCELLED |
| `identity.user.suspended.v1` | If user is Trainer: freeze profile, cancel all future bookings, publish cancellations.<br/>If user is Customer: cancel all future bookings, publish cancellations. |

---

## API (gRPC)

```protobuf
service TrainerService {
  // Public (customer)
  rpc SearchTrainers(SearchTrainersRequest) returns (TrainersResponse);
  rpc GetTrainerProfile(GetTrainerProfileRequest) returns (TrainerResponse);
  rpc GetAvailableSlots(GetSlotsRequest) returns (SlotsResponse);
  rpc CreateBooking(CreateBookingRequest) returns (BookingResponse);
  rpc CancelBooking(CancelBookingRequest) returns (BookingResponse);
  rpc GetMyBookings(GetMyBookingsRequest) returns (BookingsResponse);

  // Trainer
  rpc UpdateMyProfile(UpdateTrainerProfileRequest) returns (TrainerResponse);
  rpc SetAvailability(SetAvailabilityRequest) returns (AvailabilityResponse);
  rpc AcceptBooking(AcceptBookingRequest) returns (BookingResponse);
  rpc RejectBooking(RejectBookingRequest) returns (BookingResponse);
  rpc CompleteBooking(CompleteBookingRequest) returns (BookingResponse);
  rpc GetCoachingHistory(GetCoachingHistoryRequest) returns (BookingsResponse);

  // Admin
  rpc CreateTrainer(CreateTrainerRequest) returns (TrainerResponse);
  rpc SuspendTrainer(SuspendTrainerRequest) returns (SuspendTrainerResponse);  // returns affected_booking_count
  rpc ListTrainers(ListTrainersRequest) returns (TrainersResponse);
}
```

---

## Scheduled Jobs

| Job | Schedule | Action |
|-----|----------|--------|
| BookingPaymentTimeoutJob | Every 5 min | PENDING_PAYMENT > 30 min old → EXPIRED, release slot |
| BookingTrainerTimeoutJob | Every 15 min | REQUESTED > 4 hours old → AUTO_REJECTED, refund, notify |

---

## Clean Architecture

```
src/main/java/com/gym/trainer/
├── domain/
│   ├── model/
│   │   ├── Trainer.java
│   │   ├── TrainerAvailability.java
│   │   ├── Booking.java
│   │   ├── BookingStatus.java
│   │   └── TimeSlot.java
│   └── exception/
│       ├── SlotNotAvailableException.java
│       ├── BookingNotFoundException.java
│       └── UnauthorizedBookingActionException.java
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── SearchTrainersUseCase.java
│   │   │   ├── ManageBookingUseCase.java
│   │   │   ├── ManageAvailabilityUseCase.java
│   │   │   └── GetCoachingHistoryUseCase.java
│   │   └── out/
│   │       ├── TrainerRepository.java
│   │       ├── BookingRepository.java
│   │       ├── AvailabilityRepository.java
│   │       └── EventPublisher.java
│   ├── service/
│   │   ├── TrainerSearchService.java
│   │   ├── BookingService.java
│   │   └── AvailabilityService.java
│   └── scheduler/
│       ├── BookingPaymentTimeoutJob.java
│       └── BookingTrainerTimeoutJob.java
├── adapter/
│   ├── in/grpc/
│   │   ├── TrainerGrpcHandler.java
│   │   └── TrainerProtoMapper.java
│   ├── out/persistence/
│   │   ├── TrainerJpaEntity.java
│   │   ├── BookingJpaEntity.java
│   │   └── TrainerPersistenceAdapter.java
│   └── out/kafka/
│       ├── TrainerEventPublisher.java
│       └── TrainerEventConsumer.java
└── config/
    └── GrpcConfig.java
```
