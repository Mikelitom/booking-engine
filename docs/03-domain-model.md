# Booking Engine — Domain Model

**Project:** Booking Engine  
**Document Version:** 0.1.0  
**Status:** Draft  
**Last Updated:** November 2026

---

## 1. Introduction

### 1.1 Purpose

This document describes the domain model of Booking Engine, including its entities, value objects, aggregate boundaries, relationships, and business invariants.

The model is designed to represent business rules independently of application frameworks, database technologies, and HTTP interfaces.

### 1.2 Design Principles

The domain model follows these principles:

- Business rules must remain independent of infrastructure.
- Entities must encapsulate their own valid state transitions.
- Value objects should be immutable.
- Aggregate boundaries should remain small and cohesive.
- Cross-aggregate consistency must be explicitly coordinated.
- Database constraints must protect critical invariants under concurrency.
- Unnecessary abstractions should be avoided.

---

## 2. Domain Concepts

The system contains four primary domain concepts:

| Concept | Classification | Responsibility |
|---|---|---|
| User | Entity / Aggregate Root | Represent a registered user and their role |
| Room | Aggregate Root | Manage room properties and operating schedules |
| Booking | Aggregate Root | Manage reservation state and lifecycle |
| OperatingSchedule | Value Object collection | Represent recurring room operating hours |

Additional value objects encapsulate reusable domain rules.

---

## 3. Entities and Aggregates

### 3.1 User

A User represents an individual authorized to interact with the booking system.

**Attributes:**

| Attribute | Description |
|---|---|
| id | Unique user identifier |
| email | Normalized email address |
| password_hash | Securely hashed password |
| role | User authorization role |
| created_at | Account creation timestamp |

**Roles:**

- `USER`
- `ADMIN`

**Invariants:**

- A user must have a valid email address.
- A user must have a valid role.
- A password must never be stored in plaintext.
- User identifiers must remain stable.

Email uniqueness is enforced through persistence constraints.

Authentication and authorization workflows belong to the application and security boundaries rather than the booking domain itself.

### 3.2 Room

Room represents a physical meeting room that can be reserved.

Room is an aggregate root responsible for maintaining its configuration.

**Attributes:**

| Attribute | Description |
|---|---|
| id | Unique room identifier |
| name | Room name |
| capacity | Maximum occupancy |
| is_active | Whether new reservations are allowed |
| time_zone | IANA time zone identifier |
| operating_schedule | Weekly operating intervals |
| created_at | Creation timestamp |

**Responsibilities:**

- Validate room properties.
- Activate and deactivate the room.
- Manage recurring operating schedules.
- Prevent overlapping operating intervals.
- Determine whether a requested interval fits its operating schedule.

**Invariants:**

- Room names cannot be empty.
- Capacity must be greater than zero.
- The configured time zone must be valid.
- Operating intervals must have positive duration.
- Operating intervals for the same day must not overlap.
- A reservation must fit entirely within one permitted operating interval.
- An inactive room cannot accept new reservations.

Room does not contain a collection of Booking entities.

Existing reservations are managed independently.

### 3.3 Booking

Booking represents a reservation of a specific room by a specific user.

Booking is an independent aggregate root.

**Attributes:**

| Attribute | Description |
|---|---|
| id | Unique booking identifier |
| room_id | Reserved room identifier |
| user_id | Reservation owner identifier |
| interval | Reservation time interval |
| status | Current reservation status |
| created_at | Creation timestamp |

**Responsibilities:**

- Validate reservation-specific rules.
- Maintain a valid reservation state.
- Enforce permitted lifecycle transitions.
- Preserve reservation ownership and scheduling information.

**Invariants:**

- A booking must reference a room.
- A booking must reference a user.
- The reservation interval must have positive duration.
- The reservation duration must be between 15 minutes and 4 hours.
- A cancelled booking cannot become confirmed again.
- Booking identifiers must remain stable.

**Cross-Aggregate Invariants:**

- A booking cannot be created for an inactive room.
- A booking must comply with the room's operating schedule.
- Confirmed bookings for the same room must not overlap.
- A booking must begin in the future.
- A booking cannot be created more than 30 days in advance.

These rules require coordination beyond the Booking aggregate.

---

## 4. Value Objects

### 4.1 TimeInterval

TimeInterval represents an immutable interval between two instants.

**Attributes:**

- `start`
- `end`

**Invariants:**

- Both values represent unambiguous instants.
- Start must precede end.

**Semantics:**

Time intervals are half-open:

`[start, end)`

Two intervals overlap when:

`A.start < B.end AND B.start < A.end`

Adjacent intervals do not overlap.

**Operations:**

- `overlaps(other)`
- `contains(instant)`
- `duration`

TimeInterval does not enforce booking-specific duration limits.

Those rules belong to the Booking domain.

### 4.2 OperatingInterval

OperatingInterval represents a recurring local-time interval during which a room can be reserved.

**Attributes:**

- `start_time`
- `end_time`

**Invariants:**

- Start time must precede end time.
- Intervals must not cross midnight in the MVP.

OperatingInterval uses local wall-clock times rather than absolute timestamps.

### 4.3 OperatingSchedule

OperatingSchedule represents the recurring weekly operating hours of a room.

It is modeled as an immutable collection of operating intervals grouped by weekday.

**Attributes:**

- `weekday`
- `intervals`

**Invariants:**

- Weekdays must be valid.
- Intervals for the same weekday must not overlap.
- Empty intervals represent a closed day.

**Responsibilities:**

- Validate weekly operating hours.
- Determine whether a reservation interval fits an operating period.
- Interpret local operating times using the room's configured time zone.

OperatingSchedule has no independent identity or lifecycle.

It belongs to the Room aggregate.

---

## 5. Aggregate Boundaries

The domain defines three aggregate roots:

- User
- Room
- Booking

### 5.1 Room Aggregate

The Room aggregate contains:

- Room identity and properties.
- Room operating schedule.
- Room activation state.

All modifications to operating schedules must preserve Room invariants.

### 5.2 Booking Aggregate

The Booking aggregate contains:

- Booking identity.
- Room and user references.
- Reservation interval.
- Reservation status.

Booking does not directly modify Room or User.

### 5.3 User Aggregate

The User aggregate contains:

- User identity.
- Account information.
- Role.

Booking references User through its identifier.

### 5.4 Cross-Aggregate Coordination

The application layer coordinates operations involving multiple aggregates.

For example, creating a booking requires:

1. Loading the requested room.
2. Verifying that the room is active.
3. Validating the reservation interval.
4. Verifying operating schedule compatibility.
5. Creating the Booking aggregate.
6. Persisting the reservation transactionally.

The database must independently enforce reservation non-overlap.

An application-level availability check is not sufficient to guarantee correctness under concurrency.

Room configuration changes and booking creation must also be coordinated to prevent inconsistent decisions based on concurrent modifications.

---

## 6. Booking Lifecycle

The MVP supports two booking states:

- `CONFIRMED`
- `CANCELLED`

### 6.1 State Transitions

| Current State | Operation | Result |
|---|---|---|
| Not created | Create valid booking | CONFIRMED |
| CONFIRMED | Cancel | CANCELLED |
| CANCELLED | Cancel | CANCELLED |
| CANCELLED | Confirm | Not permitted |

Repeated cancellation is treated as an idempotent operation.

### 6.2 Cancellation Rules

Cancellation permissions depend on the reservation's temporal state and the requesting user's role.

| Reservation Temporal State | Owner | Administrator |
|---|---|---|
| Future | Allowed | Allowed |
| Ongoing | Not allowed | Allowed |
| Completed | Not allowed | Not allowed |

The temporal state is derived from the reservation interval and the current instant.

It is not stored as an independent booking status.

**Additional Rules:**

- Only confirmed reservations may transition to `CANCELLED`.
- Already cancelled reservations remain cancelled.
- Cancellation must preserve reservation history.
- Cancelled reservations do not block room availability.
- Authorization must be evaluated before modifying reservation state.

The application layer is responsible for evaluating the requesting user's permissions.

The Booking aggregate is responsible for enforcing valid state transitions.

## 7. Domain Relationships

The primary relationships are:

| Relationship | Cardinality |
|---|---|
| User → Booking | One-to-many |
| Room → Booking | One-to-many |
| Room → OperatingSchedule | One-to-one ownership |
| OperatingSchedule → OperatingInterval | One-to-many composition |

The relationships describe domain associations, not necessarily ORM relationships.

Aggregate references should use identifiers rather than requiring direct object references.

---

## 8. Business Invariants

The following invariants must always hold.

### INV-001 — Valid Reservation Interval

Every reservation must have a start instant earlier than its end instant.

### INV-002 — Valid Reservation Duration

Reservation duration must be between 15 minutes and 4 hours.

### INV-003 — Active Room Requirement

New reservations can only be created for active rooms.

### INV-004 — Operating Schedule Compliance

Reservations must fit entirely within an allowed operating interval.

### INV-005 — No Overlapping Confirmed Reservations

Confirmed reservations for the same room must not overlap.

### INV-006 — Valid Reservation State

Bookings must follow the permitted lifecycle transitions.

### INV-007 — Future Reservation Requirement

New reservations must start after the current instant.

### INV-008 — Advance Booking Limit

Reservations cannot start more than 30 days after the current instant.

### INV-009 — Valid Operating Schedule

Operating intervals for the same room and weekday must not overlap.

### INV-010 — Historical Preservation

Cancellation must preserve the booking record.

### INV-011 — Schedule Change Consistency

Changes to room operating schedules must not invalidate existing future confirmed bookings.

### INV-012 — Room Deactivation Consistency

Room deactivation must prevent new reservations without automatically cancelling existing bookings.

---

## 9. Concurrency and Consistency

### 9.1 Reservation Conflicts

Two requests may attempt to reserve overlapping intervals simultaneously.

The application must not rely exclusively on querying existing reservations before insertion.

PostgreSQL will enforce non-overlapping confirmed reservations using an exclusion constraint.

### 9.2 Transaction Boundaries

Reservation creation must execute within a database transaction.

If persistence fails due to a reservation conflict, the transaction must roll back.

The application must translate the database conflict into an appropriate domain or application error.

### 9.3 Schedule Modification Consistency

Schedule modifications must be coordinated with reservation creation.

The implementation must prevent a schedule update from invalidating a reservation created concurrently.

The exact locking strategy will be documented in a separate Architecture Decision Record.

### 9.4 Consistency Responsibilities

| Responsibility | Primary Enforcement |
|---|---|
| Valid interval | Value Object |
| Valid booking duration | Domain |
| Valid booking state transition | Booking aggregate |
| Room operating schedule validity | Room aggregate |
| Booking ownership authorization | Application layer |
| No overlapping confirmed bookings | PostgreSQL constraint |
| Referential integrity | PostgreSQL foreign keys |
| Atomic persistence | Database transaction |
| Concurrent schedule consistency | Transactional coordination |

---

## 10. Domain Design Constraints

The domain model must not depend directly on:

- FastAPI.
- SQLAlchemy.
- Pydantic HTTP schemas.
- PostgreSQL-specific types.
- Redis.
- Celery.

Infrastructure-specific concerns must remain outside the domain model.

The domain may define abstractions when required by concrete use cases, but unnecessary interfaces should be avoided.

---

## 11. Open Design Decisions

The following decisions will be addressed during architecture and implementation design:

1. Transaction and locking strategy for concurrent room configuration changes.
2. Representation of operating schedules in PostgreSQL.
3. Handling of daylight saving time transitions.
4. Domain exception hierarchy.
5. Repository interfaces and transaction management.
6. Mapping between domain entities and persistence models.

These decisions must be documented before implementing the affected functionality.