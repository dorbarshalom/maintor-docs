# Planned Maintenance Execution Window - Frontend Developer Guide

## Overview

The planned maintenance system now includes automatic execution windows for tickets. Each planned maintenance ticket has a time window to execute it based on the recurrence frequency of its template. Once the window expires without execution, the ticket status automatically changes to `SKIPPED`.

## Key Changes

### 1. New Ticket Status: `SKIPPED`

The ticket status enum now includes `SKIPPED`:

```typescript
type TicketStatus = 
  | 'OPEN' 
  | 'IN_PROGRESS' 
  | 'COMPLETED' 
  | 'CLOSED' 
  | 'SKIPPED'; // NEW
```

**When does a ticket become SKIPPED?**
- The ticket is of type `PLANNED`
- The ticket status is `OPEN`
- The execution window has expired (calculated based on recurrence frequency)
- The ticket has not been started (no `timeline.started`)

**Important:** Tickets are automatically updated to `SKIPPED` by a scheduled job that runs daily at midnight. The frontend should handle this status appropriately in the UI.

### 2. Scheduled Date Format Change

The `scheduled_date` field now includes both date and time (not just date):

**Old Format (deprecated):**
```json
{
  "scheduled_date": "2025-01-15"
}
```

**New Format (current):**
```json
{
  "scheduled_date": "2025-01-15T08:00:00Z"
}
```

**Format:** ISO 8601 datetime string: `YYYY-MM-DDTHH:mm:ssZ`

The time component comes from the recurrence pattern's `time` field (e.g., "08:00" becomes "08:00:00Z").

### 3. Execution Window Logic

The execution window determines how long a ticket remains `OPEN` before automatically becoming `SKIPPED`:

#### For tasks with frequency < 12 weeks:
- **Window extends from scheduled instance to the next occurrence**
- Example: Daily task scheduled Monday 08:00 AM → window ends Tuesday 08:00 AM
- Example: Task scheduled Monday and Wednesday, ticket for Monday → window ends Wednesday (with time from recurrence)

#### For tasks with frequency ≥ 12 weeks:
- **Fixed 30-day window** from the scheduled instance date

**Note:** The execution window is calculated dynamically from the recurrence patterns in the template. It's not stored on the ticket itself.

## API Changes

### Ticket Status Enum

All ticket status enums now include `SKIPPED`:

- `GET /v1/accounts/{accountId}/tickets?status=SKIPPED` - Filter by SKIPPED status
- Ticket response includes `SKIPPED` in status enum
- Ticket creation/update accepts `SKIPPED` in status field

### Scheduled Date Format

- `scheduled_date` is now always in ISO datetime format (includes time)
- When creating/updating tickets, use ISO datetime format
- When displaying, parse the datetime to show both date and time

## Frontend Implementation Guide

### 1. Update Type Definitions

```typescript
// Update your Ticket type
interface Ticket {
  id: string;
  type: 'PLANNED' | 'BREAKDOWN';
  status: 'OPEN' | 'IN_PROGRESS' | 'COMPLETED' | 'CLOSED' | 'SKIPPED';
  scheduled_date?: string; // ISO datetime: "2025-01-15T08:00:00Z"
  // ... other fields
}
```

### 2. Handle SKIPPED Status in UI

#### Display SKIPPED Tickets

```typescript
// Example: Filter and display SKIPPED tickets
const skippedTickets = tickets.filter(t => t.status === 'SKIPPED');

// In your ticket list component
{ticket.status === 'SKIPPED' && (
  <Badge variant="warning">Skipped</Badge>
)}
```

#### Status Badge Colors

Suggested color scheme:
- `SKIPPED`: Orange/Yellow (warning) - indicates missed execution window
- `OPEN`: Blue (info) - ready to execute
- `IN_PROGRESS`: Purple (active) - currently being worked on
- `COMPLETED`: Green (success) - finished
- `CLOSED`: Gray (neutral) - closed

### 3. Parse Scheduled Date with Time

```typescript
// Parse scheduled_date (now includes time)
const parseScheduledDate = (scheduledDate: string) => {
  // Handle both old format (YYYY-MM-DD) and new format (ISO datetime)
  if (scheduledDate.match(/^\d{4}-\d{2}-\d{2}$/)) {
    // Legacy format - add time component
    return new Date(`${scheduledDate}T00:00:00Z`);
  }
  return new Date(scheduledDate);
};

// Display with time
const formatScheduledDateTime = (scheduledDate: string) => {
  const date = parseScheduledDate(scheduledDate);
  return date.toLocaleString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    timeZoneName: 'short'
  });
  // Example output: "Jan 15, 2025, 08:00 AM UTC"
};
```

### 4. Show Execution Window Information

While the execution window is calculated server-side, you can display helpful information:

```typescript
// For planned maintenance tickets
{ticket.type === 'PLANNED' && ticket.status === 'OPEN' && (
  <div className="execution-window-info">
    <InfoIcon />
    <span>
      This ticket must be executed before the next scheduled occurrence.
      {ticket.scheduled_date && (
        <> Scheduled: {formatScheduledDateTime(ticket.scheduled_date)}</>
      )}
    </span>
  </div>
)}
```

### 5. Filter and Sort Tickets

```typescript
// Filter by SKIPPED status
const filterTickets = (tickets: Ticket[], statusFilter?: string) => {
  if (!statusFilter) return tickets;
  return tickets.filter(t => t.status === statusFilter);
};

// Sort tickets (SKIPPED tickets might go to bottom or separate section)
const sortTickets = (tickets: Ticket[]) => {
  return [...tickets].sort((a, b) => {
    // Priority order: OPEN > IN_PROGRESS > COMPLETED > SKIPPED > others
    const priority = {
      'OPEN': 1,
      'IN_PROGRESS': 2,
      'COMPLETED': 3,
      'SKIPPED': 4,
      'CLOSED': 5,
    };
    return (priority[a.status] || 99) - (priority[b.status] || 99);
  });
};
```

### 6. Handle Status Transitions

```typescript
// When a ticket is updated to SKIPPED (either manually or automatically)
const handleTicketStatusChange = (ticket: Ticket, newStatus: TicketStatus) => {
  if (newStatus === 'SKIPPED') {
    // Show notification or update UI
    showNotification({
      type: 'warning',
      message: `Ticket "${ticket.title}" has been marked as SKIPPED (execution window expired)`,
    });
    
    // Optionally refresh ticket list
    refreshTickets();
  }
};
```

### 7. Prevent Actions on SKIPPED Tickets

```typescript
// Disable certain actions for SKIPPED tickets
const canStartTicket = (ticket: Ticket) => {
  return ticket.status === 'OPEN';
  // SKIPPED tickets cannot be started (they missed their window)
};

// In your component
<Button
  onClick={startTicket}
  disabled={!canStartTicket(ticket)}
>
  Start Ticket
</Button>
```

### 8. Display Scheduled Date with Time

```typescript
// Component for displaying scheduled date
const ScheduledDateDisplay: React.FC<{ scheduledDate: string }> = ({ scheduledDate }) => {
  const date = parseScheduledDate(scheduledDate);
  const dateStr = date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  });
  const timeStr = date.toLocaleTimeString('en-US', {
    hour: '2-digit',
    minute: '2-digit',
  });
  
  return (
    <div>
      <CalendarIcon />
      <span>{dateStr}</span>
      <ClockIcon />
      <span>{timeStr}</span>
    </div>
  );
};
```

## Migration Notes

### Backward Compatibility

The API handles both old and new `scheduled_date` formats:
- **Old format** (`YYYY-MM-DD`): Still accepted, but converted to ISO datetime with `00:00:00Z` time
- **New format** (`YYYY-MM-DDTHH:mm:ssZ`): Preferred format

**Recommendation:** Update your frontend to always send ISO datetime format when creating/updating tickets.

### Existing Tickets

- Existing tickets with old `YYYY-MM-DD` format will continue to work
- The scheduled job will handle legacy format automatically
- Consider migrating display logic to handle both formats gracefully

## Example API Responses

### Ticket with SKIPPED Status

```json
{
  "id": "ticket_123",
  "type": "PLANNED",
  "status": "SKIPPED",
  "title": "Weekly Compressor Maintenance",
  "scheduled_date": "2025-01-15T08:00:00Z",
  "plannedMaintenanceTemplateId": "template_456",
  "created": "2025-01-10T10:00:00Z",
  "updated": "2025-01-16T00:05:00Z"
}
```

### Ticket with OPEN Status (within window)

```json
{
  "id": "ticket_124",
  "type": "PLANNED",
  "status": "OPEN",
  "title": "Daily Pump Inspection",
  "scheduled_date": "2025-01-16T08:00:00Z",
  "plannedMaintenanceTemplateId": "template_789",
  "created": "2025-01-10T10:00:00Z",
  "updated": "2025-01-10T10:00:00Z"
}
```

## Testing Checklist

- [ ] Update TypeScript types to include `SKIPPED` status
- [ ] Update status filter dropdown to include `SKIPPED`
- [ ] Update status badge/icon component to display `SKIPPED`
- [ ] Update scheduled date parser to handle ISO datetime format
- [ ] Update scheduled date display to show time component
- [ ] Test filtering tickets by `SKIPPED` status
- [ ] Test displaying tickets with `SKIPPED` status
- [ ] Test that `SKIPPED` tickets cannot be started
- [ ] Test that scheduled date input accepts ISO datetime format
- [ ] Verify backward compatibility with old date format

## Questions?

If you have questions about:
- Execution window calculation logic
- How to display execution window information
- Handling status transitions
- Migration from old date format

Please refer to the OpenAPI specification at `/openapi.json` or contact the backend team.

## Summary

1. **New Status**: `SKIPPED` - tickets that missed their execution window
2. **Date Format**: `scheduled_date` now includes time (ISO datetime)
3. **Automatic Updates**: Tickets automatically become `SKIPPED` when window expires
4. **UI Updates**: Display `SKIPPED` status appropriately, show scheduled time, handle status transitions


