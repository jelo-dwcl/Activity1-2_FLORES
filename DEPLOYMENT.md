# Deployment Plan: Hotel Booking and Housekeeping (Integrated System)

Stack: PostgreSQL database, RabbitMQ message queue, Booking service (RBS) on port 8080, Housekeeping service (HMS), Integration Service, and an API gateway, all run with Docker Compose.

## Pre-Flight Checklist

### 1. Sequence Services

| Order | Service | Depends on | Ready when |
| --- | --- | --- | --- |
| 1 | Database (PostgreSQL) | Nothing | Accepts connections (pg_isready) and migrations are applied |
| 2 | Message queue (RabbitMQ) | Nothing | Broker is up, the hotel.events exchange, the booking and housekeeping queues, and the dead-letter queue exist |
| 3 | Booking service (RBS) | Database, message queue | GET /health returns 200 and its consumer is connected |
| 4 | Housekeeping service (HMS) | Database, message queue | GET /health returns 200 and it is connected to its queue as a consumer |
| 5 | Integration Service | Message queue, RBS, HMS | Connected to the broker as a consumer |
| 6 | Gateway / web front end | Booking service, Housekeeping service | GET /health returns 200 and it routes to the API; public traffic opens last |

**Reason:** each service starts only after what it depends on is ready, so nothing starts and crashes while looking for a missing database or queue. Shutdown runs in the reverse order, and the broker stays up until the services have stopped so no message is lost.

### 2. Define Go/No-Go

Three health checks before public traffic:

- **API health:** GET /health on the gateway, RBS and HMS returns 200 within 500 ms and reports the database and the queue as "up". All containers show healthy in `docker compose ps`.
- **Database check:** a test read and a test write on the bookings table both succeed, and the schema version matches the release.
- **Queue check, end to end:** a test check-out creates a housekeeping task within 3 seconds and the room shows AVAILABLE in the RBS within 5 seconds. This proves the message passes through the queue and the Housekeeping service consumes it. A call with no or a bad token must return 401. Use test data only.

**Go:** all three checks pass.
**No-Go:** any one check fails. Fix the problem, re-run all three checks, and do not open public traffic until they pass.

### 3. Abort Trigger

Abort and roll back if any of these happens within the first 30 minutes of public traffic:

- The error rate (HTTP 5xx) is above 2% of requests for 5 minutes in a row.
- Fewer than 95% of check-outs create a housekeeping task within 3 seconds, over 5 minutes in a row.
- Fewer than 99% of room status updates reach the RBS within 5 seconds, over 5 minutes.
- Any message is lost (1 or more; the target is 0).
- The dead-letter queue has more than 5 messages, or it keeps growing.
- Any request without a valid token is accepted, or a secret or full guest data shows up in the Housekeeping service.
- Any of the 3 Go/No-Go checks fails after release.

## Rollback Plan: Abort Protocol

1. Stop the new containers and revert the container tag to the last stable image (for example v1.4.2). Keep PostgreSQL and RabbitMQ running so no messages are lost.

   ```bash
   docker compose up -d rbs hms integration-service gateway   # with the stable tag set
   ```

2. Execute the pre-tested down-migration script for any database change.
3. Let the queued messages replay. Duplicates are ignored because each message has a unique ID.
4. Re-run the 3 Go/No-Go checks.
5. Report the rollback with the time and the cause.