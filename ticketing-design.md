
# TicketHub - Event Ticketing System Design

## 1. Context

TicketHub is an online event ticketing platform where users can discover events, view available seats, purchase tickets and access their purchased tickets.

The system must support 2 million registered users and handle high traffic when popular concerts go on sale.

The main challenge is preventing two users from purchasing the same seat while keeping the system fast and available.

### Goals

- Allow users to browse concerts and events.
- Display available seats.
- Allow users to temporarily hold seats.
- Process ticket payments.
- Generate tickets after successful payment.
- Prevent double-booking.
- Support high traffic during popular ticket sales.

### Non-Goals

- Organising or managing the actual concerts.
- Providing transport or accommodation booking.
- Building a payment processing service from scratch.

## 2. Requirements and Estimates

### Functional Requirements

The system must allow users to:

1. Register and log in.
2. Browse available events.
3. View event details and available seats.
4. Temporarily hold available seats.
5. Pay for reserved seats.
6. Receive tickets after successful payment.
7. View their purchased tickets.

### Non-Functional Requirements

- **Performance:** Event pages and seat availability should load quickly, ideally within one second under normal conditions.
- **Availability:** The system should remain accessible during high-demand ticket sales.
- **Correctness:** A seat must never be sold to two different customers.
- **Fairness:** Seat reservations should be processed consistently, without allowing users to bypass the booking process.
- **Scalability:** The system should handle sudden increases in traffic.
- **Security:** User accounts and payment information must be protected.

### Traffic Estimates

Given facts:

- Registered users: 2,000,000
- Normal daily visitors: 50,000
- Each visitor views 10 pages.
- Normal tickets sold per day: 5,000
- Popular concert seats: 20,000
- People trying to buy tickets: 200,000
- Big sale duration: 10 minutes

### Normal Traffic

Daily page views:

50,000 × 10 = 500,000 page views per day.

Average requests per second:

500,000 ÷ 86,400 = approximately 5.8 requests per second.

Normal ticket sales:

5,000 tickets per day.

### Big Sale Traffic

200,000 people attempt to buy tickets within 10 minutes.

10 minutes = 600 seconds.

Average incoming users per second:

200,000 ÷ 600 = approximately 333 users per second.

Maximum successful ticket sales per second:

20,000 ÷ 600 = approximately 33 tickets per second.

There are 10 people competing for each available seat on average:

200,000 ÷ 20,000 = 10.

### Traffic Comparison

| Measurement | Normal Traffic | Big Sale |
|---|---:|---:|
| Visitors / attempted buyers | 50,000 per day | 200,000 in 10 minutes |
| Page views | 500,000 per day | Depends on user actions |
| Average incoming rate | 5.8 requests/second | 333 users/second |
| Ticket sales | 5,000 per day | Maximum 20,000 |
| Main challenge | Regular browsing and purchases | Sudden traffic and seat contention |

The big sale creates a major increase in traffic and competition for seats. The system must therefore scale its application servers and protect the database from too many simultaneous requests.

## 3. API Design

The TicketHub API uses REST principles and JSON for communication.

Base URL:

`https://api.tickethub.com/v1`

| Method | Endpoint | Description | Success Status |
|---|---|---|---|
| GET | /events | Retrieve available events | 200 OK |
| GET | /events/{id} | Retrieve details of one event | 200 OK |
| GET | /events/{id}/seats | View available seats for an event | 200 OK |
| POST | /seat-holds | Temporarily reserve an available seat | 201 Created |
| POST | /payments | Process payment for a held seat | 201 Created |
| GET | /users/me/tickets | Retrieve the logged-in user's tickets | 200 OK |

### Example: View Available Seats

Request:

`GET /events/25/seats`

Response:

```json
{
  "eventId": 25,
  "availableSeats": [
    {
      "id": 101,
      "seatNumber": "A1",
      "status": "AVAILABLE"
    },
    {
      "id": 102,
      "seatNumber": "A2",
      "status": "AVAILABLE"
    }
  ]
}
```

### Example: Hold a Seat

Request:

`POST /seat-holds`

Request body:

```json
{
  "eventId": 25,
  "seatId": 101
}
```

Response:

Status: `201 Created`

```json
{
  "holdId": 501,
  "seatId": 101,
  "status": "HELD",
  "expiresAt": "2026-10-31T15:10:00Z"
}
```

### Example: Make a Payment

Request:

`POST /payments`

Request body:

```json
{
  "holdId": 501,
  "paymentMethod": "card",
  "idempotencyKey": "payment-501-unique-key"
}
```

Response:

Status: `201 Created`

```json
{
  "paymentId": 801,
  "holdId": 501,
  "status": "SUCCESS",
  "message": "Payment completed successfully."
}
```

### Example: View Purchased Tickets

Request:

`GET /users/me/tickets`

Response:

Status: `200 OK`

```json
{
  "tickets": [
    {
      "ticketId": 901,
      "eventId": 25,
      "seatNumber": "A1",
      "status": "CONFIRMED"
    }
  ]
}
```

## 4. Data Model

TicketHub uses a relational SQL database such as PostgreSQL because ticket purchases require reliable transactions, constraints and relationships.

### Tables

#### Users

Stores registered customer information.

| Column | Type | Key / Constraint |
|---|---|---|
| id | INTEGER | Primary Key |
| name | VARCHAR(100) | NOT NULL |
| email | VARCHAR(255) | NOT NULL, UNIQUE |
| password_hash | VARCHAR(255) | NOT NULL |

#### Events

Stores concert and event information.

| Column | Type | Key / Constraint |
|---|---|---|
| id | INTEGER | Primary Key |
| event_name | VARCHAR(150) | NOT NULL |
| venue | VARCHAR(150) | NOT NULL |
| event_date | TIMESTAMP | NOT NULL |

#### Seats

Stores seats available for each event.

| Column | Type | Key / Constraint |
|---|---|---|
| id | INTEGER | Primary Key |
| event_id | INTEGER | Foreign Key |
| seat_number | VARCHAR(20) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| held_by_user_id | INTEGER | Foreign Key, optional |
| hold_expires_at | TIMESTAMP | Optional |

The combination of event_id and seat_number must be unique.

#### Orders

Stores ticket purchase records.

| Column | Type | Key / Constraint |
|---|---|---|
| id | INTEGER | Primary Key |
| user_id | INTEGER | Foreign Key |
| seat_id | INTEGER | Foreign Key, UNIQUE |
| payment_status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

The unique seat_id constraint prevents a seat from appearing in multiple orders.

### Relationships

- **Users and Orders (One-to-Many):** One user can have many orders, but each order belongs to one user.
- **Events and Seats (One-to-Many):** One event has many seats, but each seat belongs to one event.
- **Seats and Orders (One-to-One):** Each seat can appear in at most one successful order, and each order purchases one seat.

### Preventing Double-Booking

The system uses database transactions and constraints to prevent two customers from purchasing the same seat.

The process is:

1. A user requests to hold a seat.
2. The server starts a database transaction.
3. It checks whether the seat is available.
4. The server changes the seat status from AVAILABLE to HELD only if it is still available.
5. The database confirms that only one request successfully changed the seat.
6. The successful user receives a temporary hold with an expiry time.
7. After successful payment, another transaction verifies the hold and changes the seat status to SOLD.
8. The order is saved with the seat ID.
9. The UNIQUE constraint on orders.seat_id provides an additional database-level protection against duplicate sales.

If another user tries to hold the same seat, the database update will not succeed because the seat is no longer available.

If the hold expires without payment, the seat can be released and made available again.

### Example SQL for Preventing Double-Booking

```sql
BEGIN;

UPDATE seats
SET
    status = 'HELD',
    held_by_user_id = 10,
    hold_expires_at = CURRENT_TIMESTAMP + INTERVAL '10 minutes'
WHERE id = 101
  AND status = 'AVAILABLE';

-- Application checks that exactly one row was updated.
-- If zero rows were updated, the seat was already unavailable.

COMMIT;
```

The application must check the number of affected rows. If no row was updated, the request must be rejected and the transaction handled accordingly.

When payment is confirmed, the application verifies the active hold, marks the seat as SOLD and creates the order in a database transaction.

The payment process should use an idempotency key so that repeated payment requests do not create duplicate charges or orders.

## 5. High-Level Architecture

### Architecture Diagram

```text
                      USERS
                        |
                        v
                      CLIENT
                        |
                        v
                       DNS
                        |
                        v
                       CDN
                        |
                        v
                 LOAD BALANCER
                        |
                +-------+-------+
                |               |
                v               v
           APP SERVER 1    APP SERVER 2
                |               |
                +-------+-------+
                        |
              +---------+---------+
              |                   |
              v                   v
            CACHE           PRIMARY DATABASE
                                  |
                                  v
                             READ REPLICA

                APP SERVERS
                     |
                     v
                   QUEUE
                     |
                     v
                   WORKER
                     |
                     v
             TICKET CONFIRMATIONS
```

### Component Explanations

- **Client:** Allows users to browse events, select seats and purchase tickets.
- **DNS:** Converts the TicketHub domain name into an IP address.
- **CDN:** Delivers static files from locations closer to users, reducing loading time.
- **Load Balancer:** Distributes incoming requests across application servers.
- **App Servers:** Handle API requests, user authentication, seat reservations and purchases.
- **Cache:** Stores frequently accessed event information and reduces repeated database reads.
- **Primary Database:** Stores permanent information and handles seat reservations and ticket purchases.
- **Read Replica:** Handles read requests to reduce the workload on the primary database.
- **Queue:** Holds background tasks such as ticket confirmation notifications.
- **Worker:** Processes queued tasks without delaying the main purchase request.

### How the Architecture Handles a Popular Sale

1. The CDN serves static content to users.
2. The load balancer distributes incoming requests across multiple application servers.
3. The cache handles frequently requested event information.
4. Application servers process seat availability and reservation requests.
5. The primary database handles seat holds and purchases using transactions.
6. The read replica handles browsing and other read operations.
7. The queue stores background tasks such as sending ticket confirmation messages.
8. Workers process those tasks independently.

The primary database remains responsible for deciding whether a seat can be reserved or sold. Cached seat availability must never be treated as proof that a seat is still available.

## 6. Trade-offs and Alternatives

### Trade-off 1: Caching vs Accurate Seat Availability

Caching makes event information faster to retrieve and reduces database workload.

However, cached seat information can become outdated when another customer reserves a seat.

**Decision:** Cache event details and other frequently accessed information, but confirm seat availability against the primary database before accepting a reservation.

### Trade-off 2: Temporary Holds vs More Available Seats

Temporary holds give customers time to complete payment and reduce competition during checkout.

However, seats can remain unavailable while users delay payment.

**Decision:** Give each hold a limited expiry time. Release expired holds so other customers can purchase those seats.

### Trade-off 3: Synchronous vs Background Processing

Processing every task during the purchase request can make checkout slower.

Background processing makes the main purchase flow faster, but notifications may be delayed.

**Decision:** Save and confirm the purchase first, then use a queue and worker to send ticket confirmation notifications.

### Alternatives Considered

A single application server and database would be simpler and cheaper to operate, but they could become bottlenecks during a popular sale and create single points of failure.

A distributed architecture with multiple application servers, caching and database replication is more suitable for TicketHub's traffic requirements.

## 7. Risks and Open Questions

- Payment gateway failures could interrupt purchases.
- High demand could create heavy competition for the same seats.
- Database failures could interrupt reservations and purchases.
- Expired holds must be released reliably.
- The system needs a recovery process for failed payments and interrupted bookings.
- The exact hold duration and payment timeout need to be determined.
- The system needs monitoring to detect slow requests, failed payments and database problems.

## Conclusion

TicketHub uses a scalable architecture with multiple application servers, caching, database replication and background workers.

The most important design decision is preventing double-booking through atomic database transactions, seat status checks and unique constraints.

This approach allows TicketHub to handle normal traffic and sudden increases in demand while protecting the correctness of ticket purchases.
