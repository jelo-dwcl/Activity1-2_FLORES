# API Documentation

## PATCH /reservations/{id}/status (Check-out)

**Service:** Booking service (RBS), called through the gateway.

**Auth:** `Authorization: Bearer <JWT>`. The token lasts 15 minutes and the role claim must be `front_desk`.

**What it does:** marks a reservation as checked out. The RBS publishes `guest.checked_out`. The Housekeeping service creates a cleaning task and sets the room to DIRTY, then publishes `room.status.changed`. The RBS sets the room to AVAILABLE once it is ready (within 5 seconds).

### Request

Path parameter: `id` is the reservation ID (for example RES-1001).

Body:

```json
{
  "status": "CHECKED_OUT"
}
```

### Response: 200 OK

```json
{
  "reservationId": "RES-1001",
  "roomId": "R-0305",
  "status": "CHECKED_OUT",
  "event": "guest.checked_out",
  "messageId": "b3f1c2a0-7d4e-4a55-9c1e-2f6a8d0e5a11"
}
```

### Error codes

| Code | Meaning | What to do |
| --- | --- | --- |
| 400 | Invalid body, or a status that is not allowed | Send CHECKED_OUT as the status |
| 401 | Missing, bad, or expired token | Log in again to get a new token |
| 403 | Role is not allowed (for example housekeeping role) | Use a front desk account |
| 404 | Reservation not found | Check the reservation ID |
| 409 | Reservation cannot be checked out (for example already checked out) | Check the current status first |
| 429 | Rate limit reached | Wait and retry |
| 503 | The RBS or something it needs (database, broker) is down | Wait and retry. If it continues, follow the runbook (runbook.md) |

### Sample request

```bash
curl -X PATCH http://localhost:8080/reservations/RES-1001/status \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"CHECKED_OUT"}'
```

**Note:** every message has a unique `messageId`, so repeating the request does not create a second cleaning task. Only the first name and last initial of the guest go to the Housekeeping service.