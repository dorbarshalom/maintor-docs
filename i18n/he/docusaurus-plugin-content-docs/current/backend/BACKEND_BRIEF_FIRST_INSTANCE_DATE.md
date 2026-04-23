# Backend Brief: Fix First Instance Date (Retroactive Scheduling)

## The Issue
The **Planned Maintenance** creation endpoint (`POST /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances`) is ignoring the `firstInstanceDate` field when it is set to a date in the past.

Instead of generating backfilled tickets starting from the `firstInstanceDate`, the system seems to:
1.  Ignore the date if it's in the past.
2.  Default to `NOW()` or `TODAY()` as the start date for ticket generation.
3.  Only generate future tickets.

## Expected Behavior
When a user provides a `firstInstanceDate` (e.g., `2025-09-02`):
1.  The schedule calculation should use this date as the **anchor** for the recurrence pattern.
2.  Tickets should be generated starting from this date, **even if they are in the past**.
3.  This allows for "backfilling" of compliance tasks or historical record keeping.

## Technical Details

### Payload Being Sent (Frontend)
The frontend is correctly sending the ISO string.
```json
{
  "plannedMaintenance": {
    "title": "Pump 4 Weekly Check",
    "isActive": true,
    "firstInstanceDate": "2025-09-02T09:00:00.000Z",  <-- Past Date
    "recurrence": [
      {
        "frequency": "WEEKLY",
        "interval": 1,
        "dayOfWeek": [1], // Monday
        "time": "09:00"
      }
    ],
    ...
  }
}
```

### Current Outcome
-   Result: 53 tickets created.
-   First Ticket Date: `2026-02-18` (Next upcoming Wednesday/formatted date).
-   **Missing**: All tickets between `2025-09-02` and `2026-02-18`.

### Likely Fix Location
In the backend logic that calculates the schedule (likely a loop generating `scheduled_date`):
-   Check if the start date of the loop is being clamped to `Math.max(now, firstInstanceDate)`.
-   Please remove this clamp or allow it if an explicit `firstInstanceDate` is provided.
