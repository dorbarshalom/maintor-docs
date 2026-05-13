# Planned Maintenance Ticket Lifecycle

This document explains the lifecycle of a planned maintenance ticket, including how tasks move between statuses (`PLANNED`, `OPEN`, `SKIPPED`, `IN_PROGRESS`, `COMPLETED`, `CLOSED`) and the Google Cloud services orchestrating these transitions.

## Google Cloud Services Involved

The automated state transitions for planned maintenance tickets are powered by two primary Google Cloud services:

1. **Google Cloud Tasks:** Used for queuing heavy, asynchronous workloads like bulk ticket generation (e.g., generating the next 52 weeks of planned maintenance instances). Cloud Tasks ensures that heavy operations don't block the API and are retried if they fail.
2. **Google Cloud Scheduler:** Acts as the cron job engine, triggering specific backend endpoints periodically (hourly/daily) to evaluate time-sensitive status changes, such as opening planned tasks when their scheduled date arrives, or marking expired open tasks as skipped.

---

## The Ticket Lifecycle & Logic

### 1. Generating Tickets (Status: `PLANNED`)

When a user creates or updates a planned maintenance template, the system enqueues a background job to **Google Cloud Tasks** (specifically to the `planned-maintenance-ticket-generation` queue).

- The `processTicketGenerationTask.js` handler processes the task and generates the future occurrences for up to a year based on the template's recurrence rules.
- These tickets are inserted into the database with an initial status of **`PLANNED`**.
- *Note: A backup cron job (`scheduledMaintenanceTrigger`) is also run periodically via Cloud Scheduler to ensure active templates continually generate their next batches of tickets over time without manual intervention.*

### 2. Maturing Tickets (Status: `PLANNED` ➔ `OPEN`)

Tickets don't show up as actionable on the dashboard immediately. A single Cloud Scheduler job runs hourly and triggers the `processHourlyTicketUpdates` endpoint, which internally calls the `openPlannedTickets` handler.

- This handler queries the database for all `PLANNED` tickets where the `scheduled_date` is less than or equal to the current time (i.e., the scheduled date has arrived or passed).
- The system automatically updates the status of these mature tickets to **`OPEN`**, making them visible and actionable for technicians on the platform.

### 3. Missed Deadlines (Status: `OPEN` ➔ `SKIPPED`)

If an `OPEN` ticket is ignored and its execution window expires, it is automatically marked as skipped to prevent a massive backlog of old, undone tasks. The same hourly Cloud Scheduler job orchestrates this by invoking the `updateSkippedTickets` handler:

- The handler checks all `OPEN` planned tickets that have not yet been started.
- It compares the current time against the ticket's `executionWindowDays` (defined in the template).
- If the current time is past the allowed execution window end date, the system automatically transitions the ticket's status to **`SKIPPED`**.

### 4. Manual Operations (Statuses: `IN_PROGRESS`, `COMPLETED`, `CLOSED`)

Once a ticket is `OPEN`, further status transitions are handled manually by users interacting with the frontend web application.

- By calling the API (`PUT /v1/accounts/{accountId}/tickets/{ticketId}/status`), users can transition the task to **`IN_PROGRESS`**, **`COMPLETED`**, and eventually **`CLOSED`**.
- The API validates these transitions based on account-level settings, which dictate whether specific statuses (like `IN_PROGRESS`) are enabled or disabled for the particular workspace.
