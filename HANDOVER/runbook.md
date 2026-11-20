# Operations Runbook

## Restart the Primary API (Booking service)

**When to use:** the Booking API does not respond, is hung, or returns 5xx errors. Symptoms: GET /health does not return 200 or times out, check-out requests fail with 502/503, or the logs show "connection pool exhausted".

**Impact:** the API is unavailable for about 10 to 30 seconds. Messages stay in RabbitMQ and are processed when it is back. Nothing is lost.

### Steps

1. **Check that the API is really the problem.** If health returns 200, stop and check the gateway and the queue instead.

   ```bash
   docker compose ps rbs
   curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health
   ```

2. **Save the logs first** so the cause is not lost.

   ```bash
   docker compose logs --tail=100 rbs
   ```

3. **Restart the Booking API only.** Do not restart PostgreSQL or RabbitMQ, and never use `docker compose down -v` (it deletes the data).

   ```bash
   docker compose restart rbs
   ```

4. **Wait about 15 seconds and check health.** Expected: the container shows healthy and the command returns 200.

   ```bash
   docker compose ps rbs
   curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health
   ```

5. **Check that it is connected to the queue again.** Expected: each RBS queue has at least 1 consumer and the message count goes down to 0.

   ```bash
   docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers
   ```

6. **Run the smoke test.** Check out a test reservation with PATCH /reservations/{id}/status. Expected: a housekeeping task appears within 3 seconds and the room is AVAILABLE within 5 seconds.

7. **If it still fails after 2 restarts,** ask the team lead for help and check the host disk space, memory, and the database connection. Roll back (see the Abort Protocol in DEPLOYMENT.md) only if the problem started right after a new release.

Record the time, the cause, and what you did in the incident log.