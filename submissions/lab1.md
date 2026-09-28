# Lab 1 — SRE Philosophy: Deploy, Break, Understand

## Task 1 — Deploy & Break QuickTicket

### 1.1 Deploy

QuickTicket was successfully deployed using Docker Compose.

Final `docker compose ps` output:

    NAME             IMAGE                COMMAND                  SERVICE    CREATED          STATUS                   PORTS
    app-events-1     app-events           "uvicorn main:app --…"  events     26 minutes ago   Up 10 minutes            0.0.0.0:8081->8081/tcp, [::]:8081->8081/tcp
    app-gateway-1    app-gateway           "uvicorn main:app --…"  gateway    26 minutes ago   Up 20 minutes            0.0.0.0:3080->8080/tcp, [::]:3080->8080/tcp
    app-payments-1   app-payments          "uvicorn main:app --…"  payments   26 minutes ago   Up 27 seconds            0.0.0.0:8082->8082/tcp, [::]:8082->8082/tcp
    app-postgres-1   postgres:17-alpine    "docker-entrypoint.s…"  postgres   24 minutes ago   Up 6 minutes (healthy)   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp
    app-redis-1      redis:7-alpine        "docker-entrypoint.s…"  redis      26 minutes ago   Up 8 minutes (healthy)    0.0.0.0:6379->6379/tcp, [::]:6379->6379/tcp

All five required services were running:

- gateway
- events
- payments
- postgres
- redis

---

## 1.2 Critical Path

### List events

The Events endpoint successfully returned the available events:

    [
        {
            "id": 1,
            "name": "Go Conference 2026",
            "venue": "Main Hall A",
            "date": "2026-09-15T09:00:00+00:00",
            "total_tickets": 100,
            "price_cents": 5000,
            "available": 100
        },
        {
            "id": 4,
            "name": "Python Workshop",
            "venue": "Lab 301",
            "date": "2026-09-22T14:00:00+00:00",
            "total_tickets": 25,
            "price_cents": 2000,
            "available": 25
        },
        {
            "id": 2,
            "name": "SRE Meetup",
            "venue": "Room 204",
            "date": "2026-10-01T18:00:00+00:00",
            "total_tickets": 30,
            "price_cents": 0,
            "available": 30
        },
        {
            "id": 5,
            "name": "Kubernetes Deep Dive",
            "venue": "Auditorium B",
            "date": "2026-10-10T10:00:00+00:00",
            "total_tickets": 80,
            "price_cents": 8000,
            "available": 80
        },
        {
            "id": 3,
            "name": "Cloud Native Summit",
            "venue": "Expo Center",
            "date": "2026-11-20T10:00:00+00:00",
            "total_tickets": 500,
            "price_cents": 15000,
            "available": 500
        }
    ]

### Reserve

One ticket for event 1 was successfully reserved:

    {
        "reservation_id": "6232adc0-0311-4c21-b3e2-387e79d9bac7",
        "event_id": 1,
        "quantity": 1,
        "total_cents": 5000,
        "expires_in_seconds": 300
    }

### Pay

The reservation was successfully paid and confirmed:

    {
        "order_id": "6232adc0-0311-4c21-b3e2-387e79d9bac7",
        "event_id": 1,
        "quantity": 1,
        "total_cents": 5000,
        "status": "confirmed"
    }

### Health

With all services running:

    {
        "status": "healthy",
        "checks": {
            "events": "ok",
            "payments": "ok",
            "circuit_payments": "CLOSED"
        }
    }

---

## 1.3 Dependency Map

```text
                    +-------------+
                    |   Gateway   |
                    |    :3080    |
                    +------+------+
                           |
                 +---------+---------+
                 |                   |
                 v                   v
          +-------------+     +-------------+
          |   Events    |     |  Payments   |
          |    :8081    |     |    :8082    |
          +------+------+     +-------------+
                 |
          +------+------+
          |             |
          v             v
    +-----------+   +-----------+
    | PostgreSQL|   |   Redis   |
    |   :5432   |   |   :6379   |
    +-----------+   +-----------+

The dependency structure is:
Gateway
  ├──> Events
  │      ├──> PostgreSQL
  │      └──> Redis
  │
  └──> Payments

The critical payment path is:
Client
  ↓
Gateway
  ↓
Events → Redis/PostgreSQL
  ↓
Payments
  ↓
Events → Redis/PostgreSQL
  ↓
confirmed

Client
  ↓
Gateway
  ↓
Events → Redis/PostgreSQL
  ↓
Payments
  ↓
Events → Redis/PostgreSQL
  ↓
confirmed

## 1.4 Systematic Failure Exploration

Each component was stopped separately while the other services were kept running.

### Payments stopped

| Operation | Result |
|---|---|
| Events List | Works |
| Reserve | Works |
| Pay | `Payment service unavailable` |
| Health | `degraded`, payments down |

Health response:

    {
        "status": "degraded",
        "checks": {
            "events": "ok",
            "payments": "down",
            "circuit_payments": "CLOSED"
        }
    }

User impact:

Users can still browse events and reserve tickets, but they cannot complete payment.

---

### Events stopped

| Operation | Result |
|---|---|
| Events List | `Events service unavailable` |
| Reserve | `Events service unavailable` |
| Pay | `Payment succeeded but confirmation failed — contact support` |
| Health | `degraded`, events down |

Health response:

    {
        "status": "degraded",
        "checks": {
            "events": "down",
            "payments": "ok",
            "circuit_payments": "CLOSED"
        }
    }

User impact:

Event browsing and reservation are unavailable.

An important observation is that the payment request can reach the Payments service successfully, while the final confirmation through Events fails.

This demonstrates a partial-failure scenario where the payment stage succeeds but order confirmation cannot be completed.

---

### Redis stopped

| Operation | Result |
|---|---|
| Events List | Works |
| Reserve | `Events service timeout` |
| Pay | `Payment succeeded but confirmation failed — contact support` |
| Health | `degraded`, events down |

Health response:

    {
        "status": "degraded",
        "checks": {
            "events": "down",
            "payments": "ok",
            "circuit_payments": "CLOSED"
        }
    }

User impact:

Event listing still works because event data is stored in PostgreSQL.

New reservations fail because reservation data requires Redis.

Payment can reach the Payments service, but order confirmation fails because the reservation cannot be retrieved from Redis.

---

### PostgreSQL stopped

| Operation | Result |
|---|---|
| Events List | `Events service unavailable` |
| Reserve | HTTP 500 `Internal Server Error` |
| Pay | `Payment succeeded but confirmation failed — contact support` |
| Health | `degraded`, events degraded |

Health response:

    {
        "status": "degraded",
        "checks": {
            "events": "degraded",
            "payments": "ok",
            "circuit_payments": "CLOSED"
        }
    }

User impact:

Event listing and reservation are unavailable because the Events service depends on PostgreSQL.

Payment can still reach the Payments service, but final order confirmation cannot be completed because persistent data cannot be written to PostgreSQL.

---

## Failure Summary

| Failed component | Events List | Reserve | Pay | Health | User impact |
|---|---|---|---|---|---|
| Payments | Works | Works | Fails: Payment service unavailable | Degraded | Browsing and reservation work, payment unavailable |
| Events | Fails | Fails | Payment succeeds but confirmation fails | Degraded | Browsing and reservation unavailable |
| Redis | Works | Fails: Events service timeout | Payment succeeds but confirmation fails | Degraded | Browsing works, reservations and confirmation fail |
| PostgreSQL | Fails | Fails: HTTP 500 | Payment succeeds but confirmation fails | Degraded | Event and order operations fail |

The experiments show that different components have different blast radii.

Payments has a relatively limited failure scope: browsing and reservation remain available.

Events is a central dependency and its failure affects browsing, reservation, and order confirmation.

Redis is required for temporary reservations, while PostgreSQL is required for persistent event and order data.

---

## 1.5 Load Generator

### Baseline

The load generator was run with 5 requests per second for 30 seconds while all services were healthy.

    QuickTicket Load Generator
    Target: http://localhost:3080 | RPS: 5 | Duration: 30s
    ---
    [10s] requests=40 success=40 fail=0 error_rate=0%
    [10s] requests=41 success=41 fail=0 error_rate=0%
    [10s] requests=42 success=42 fail=0 error_rate=0%
    [10s] requests=43 success=43 fail=0 error_rate=0%
    [20s] requests=80 success=80 fail=0 error_rate=0%
    [20s] requests=81 success=81 fail=0 error_rate=0%
    [20s] requests=82 success=82 fail=0 error_rate=0%
    ---
    Done. total=118 success=118 fail=0 error_rate=0%

The baseline error rate was `0%`.

### Payments stopped during load

A second load-generator run was started and the Payments service was stopped while traffic was running.

    QuickTicket Load Generator
    Target: http://localhost:3080 | RPS: 5 | Duration: 30s
    ---
    [10s] requests=41 success=37 fail=4 error_rate=9.7%
    [10s] requests=42 success=37 fail=5 error_rate=11.9%
    [10s] requests=43 success=38 fail=5 error_rate=11.6%
    [10s] requests=44 success=38 fail=6 error_rate=13.6%
    [20s] requests=80 success=67 fail=13 error_rate=16.2%
    [20s] requests=81 success=68 fail=13 error_rate=16.0%
    [20s] requests=82 success=69 fail=13 error_rate=15.8%
    [20s] requests=83 success=70 fail=13 error_rate=15.6%
    ---
    Done. total=118 success=98 fail=20 error_rate=16.9%

The baseline run had an error rate of `0%`.

During the Payments failure, the error rate increased and reached `16.9%` by the end of the test, with 20 failed requests out of 118.

This demonstrates that a Payments failure propagates to user-facing requests under load, while other parts of the system remain partially available.

---

## Conclusion

The QuickTicket system was successfully deployed and the complete critical path was verified:

    List events → Reserve ticket → Pay → Confirm order

The failure experiments demonstrated the dependency relationships and different failure scopes of the system.

The main observations were:

1. Payments failure prevents payment but does not prevent event listing or reservation.
2. Events failure affects event listing, reservation, and final order confirmation.
3. Redis failure affects temporary reservations and confirmation, while event listing can still work.
4. PostgreSQL failure affects the Events service and prevents persistent order confirmation.
5. A payment request can succeed while the subsequent confirmation step fails, demonstrating a partial-failure scenario.
6. Under load, stopping Payments increased the observed error rate from `0%` to `16.9%`.

Task 1 completed.