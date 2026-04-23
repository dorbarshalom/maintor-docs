# Calendar Task Creation for Planned Maintenance - Frontend Developer Guide

## Overview

When a planned maintenance template is created or updated, the system generates the work-schedule tickets (calendar tasks) for the next 52 weeks **asynchronously** using Google Cloud Tasks. This ensures fast API responses while ticket generation happens in the background.

## What Changed

### Before
- Calendar tasks were only created by scheduled jobs
- Creating/updating templates didn't trigger immediate calendar task creation
- Frontend had to wait for scheduled jobs to run

### After
- Calendar tasks are automatically created when templates are created/updated
- Process happens asynchronously (non-blocking)
- Tickets appear in the calendar within seconds (local dev) or minutes (production)

## How It Works

### Flow Diagram

```
User Creates/Updates Template
         ↓
Template Saved to Database
         ↓
Tickets Generated (if active)
         ↓
Calendar Update Task Queued (async)
         ↓
Cloud Task Processes Template
         ↓
Tickets Generated for Next 52 Weeks
         ↓
Tickets Appear in Calendar
```

### Technical Details

1. **Template Creation/Update**: When you create or update a planned maintenance template via the API, the backend:
   - Saves the template to the database
   - Queues a **ticket generation** task asynchronously (if template is active)
   - (Optional/legacy) Queues a calendar update task asynchronously

2. **Ticket Generation Task**: The task:
   - Fetches the template from the database
   - Generates tickets for the next ~52 weeks
   - Only processes if template is active

3. **Local Development**: Tasks are processed immediately (direct handler call)
4. **Production**: Tasks are queued via Google Cloud Tasks and processed asynchronously

## API Behavior

### Create Planned Maintenance Template

**Endpoint:** `POST /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances`

**Response:** Returns immediately with the created template

**What Happens:**
- Template is saved
- If `isActive: true`, ticket generation is queued (asynchronous)
- Response returns immediately (doesn't wait for tickets to be created)

**Example Request:**
```json
{
  "plannedMaintenance": {
    "title": "Weekly Compressor Maintenance",
    "assetIds": ["asset_001", "asset_002"],
    "generateTicketPerAsset": true,
    "recurrence": [
      {
        "frequency": "WEEKLY",
        "dayOfWeek": [1, 3, 5],
        "interval": 1,
        "time": "08:00"
      }
    ],
    "ticketTemplate": {
      "title": "Compressor Maintenance - {assetName}",
      "priority": 3,
      "tasks": [
        { "description": "Check oil levels" },
        { "description": "Inspect belts" }
      ],
      "ownerUserId": "user_123",
      "assignees": [
        { "user_id": "user_456" }
      ]
    },
    "isActive": true
  }
}
```

**Example Response:**
```json
{
  "id": "template_123",
  "accountId": "account_456",
  "siteId": "site_789",
  "title": "Weekly Compressor Maintenance",
  "isActive": true,
  "created": "2025-01-15T10:30:00Z",
  "updated": "2025-01-15T10:30:00Z"
}
```

### Update Planned Maintenance Template

**Endpoint:** `PATCH /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances/{templateId}`

**Response:** Returns immediately with the updated template

**What Happens:**
- Template is updated in database
- Future tickets are deleted and regenerated (if template is active)
- Calendar update task is queued asynchronously
- Response returns immediately

**Example Request:**
```json
{
  "plannedMaintenance": {
    "title": "Updated Weekly Compressor Maintenance",
    "recurrence": [
      {
        "frequency": "WEEKLY",
        "dayOfWeek": [1, 3, 5],
        "interval": 1,
        "time": "09:00"
      }
    ],
    "isActive": true
  }
}
```

## Frontend Implementation Guide

### 1. Handle Template Creation/Update

The API returns immediately, but calendar tasks are created asynchronously. You have two options:

#### Option A: Poll for Tickets (Recommended)

After creating/updating a template, poll the tickets endpoint to check when tickets appear:

```typescript
// After creating/updating template
const createTemplate = async (templateData) => {
  const response = await fetch(`/v1/accounts/${accountId}/sites/${siteId}/planned-maintenances`, {
    method: 'POST',
    headers: {
      'Authorization': `IDTOKEN.${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ plannedMaintenance: templateData })
  });
  
  const template = await response.json();
  
  // Poll for tickets to appear
  if (template.isActive) {
    await pollForTickets(template.id, accountId, siteId);
  }
  
  return template;
};

const pollForTickets = async (templateId, accountId, siteId, maxAttempts = 20) => {
  for (let i = 0; i < maxAttempts; i++) {
    await new Promise(resolve => setTimeout(resolve, 1000)); // Wait 1 second
    
    const tickets = await fetchTickets(accountId, {
      type: 'PLANNED',
      siteId,
      plannedMaintenanceTemplateId: templateId
    });
    
    if (tickets.length > 0) {
      console.log(`Tickets created: ${tickets.length}`);
      return tickets;
    }
  }
  
  console.warn('Tickets may still be generating in the background');
};
```

#### Option B: Show Optimistic UI

Show a loading state and refresh the calendar after a delay:

```typescript
const createTemplate = async (templateData) => {
  setLoading(true);
  
  try {
    const template = await createTemplateAPI(templateData);
    
    // Show success message
    showNotification('Template created successfully. Calendar tasks are being generated...');
    
    // Refresh calendar after delay
    setTimeout(() => {
      refreshCalendar();
    }, 5000); // 5 seconds delay
    
  } finally {
    setLoading(false);
  }
};
```

### 2. Display Calendar Tasks

Calendar tasks are regular PLANNED tickets. Query them like any other tickets:

```typescript
// Fetch tickets for calendar
const fetchCalendarTickets = async (accountId, siteId, startDate, endDate) => {
  const response = await fetch(
    `/v1/accounts/${accountId}/tickets?type=PLANNED&siteId=${siteId}`,
    {
      headers: {
        'Authorization': `IDTOKEN.${token}`
      }
    }
  );
  
  const tickets = await response.json();
  
  // Filter by date range for FullCalendar
  return tickets.filter(ticket => {
    const scheduledDate = new Date(ticket.scheduled_date);
    return scheduledDate >= startDate && scheduledDate <= endDate;
  });
};
```

### 3. FullCalendar Integration

Map tickets to FullCalendar events:

```typescript
const mapTicketsToCalendarEvents = (tickets) => {
  return tickets.map(ticket => ({
    id: ticket.id,
    title: ticket.title,
    start: ticket.scheduled_date, // ISO datetime format
    end: calculateEndTime(ticket), // Add estimated duration
    extendedProps: {
      ticketId: ticket.id,
      priority: ticket.priority,
      status: ticket.status,
      assetIds: ticket.assetIds,
      plannedMaintenanceTemplateId: ticket.plannedMaintenanceTemplateId
    },
    backgroundColor: getStatusColor(ticket.status),
    borderColor: getPriorityColor(ticket.priority)
  }));
};

const calculateEndTime = (ticket) => {
  const start = new Date(ticket.scheduled_date);
  const duration = ticket.estimatedDurationMin || 60; // Default 1 hour
  const end = new Date(start.getTime() + duration * 60000);
  return end.toISOString();
};

const getStatusColor = (status) => {
  const colors = {
    'OPEN': '#3b82f6',        // Blue
    'IN_PROGRESS': '#8b5cf6', // Purple
    'COMPLETED': '#10b981',   // Green
    'SKIPPED': '#f59e0b',     // Orange
    'CLOSED': '#6b7280'       // Gray
  };
  return colors[status] || '#6b7280';
};
```

### 4. Handle Template Updates

When a template is updated, existing future tickets are deleted and new ones are generated:

```typescript
const updateTemplate = async (templateId, updates) => {
  setLoading(true);
  
  try {
    const updatedTemplate = await updateTemplateAPI(templateId, updates);
    
    if (updatedTemplate.isActive) {
      // Show notification
      showNotification('Template updated. Regenerating calendar tasks...');
      
      // Poll for new tickets
      await pollForTickets(templateId, accountId, siteId);
      
      // Refresh calendar
      refreshCalendar();
    }
    
    return updatedTemplate;
  } finally {
    setLoading(false);
  }
};
```

### 5. Error Handling

Calendar task creation is non-blocking. If it fails, the template is still created/updated:

```typescript
const createTemplate = async (templateData) => {
  try {
    const template = await createTemplateAPI(templateData);
    
    // Template is created even if calendar task fails
    // Check for tickets after a delay
    if (template.isActive) {
      setTimeout(async () => {
        const tickets = await fetchTicketsForTemplate(template.id);
        if (tickets.length === 0) {
          console.warn('No tickets found. Calendar task may have failed.');
          // Optionally show a warning to the user
        }
      }, 10000); // Check after 10 seconds
    }
    
    return template;
  } catch (error) {
    // Handle API errors
    console.error('Failed to create template:', error);
    throw error;
  }
};
```

## Testing

### Local Development

In local development, calendar tasks are created immediately:

1. Create a template with `isActive: true`
2. Tickets should appear within 1-2 seconds
3. Check the browser console for logs:
   ```
   [cloud-tasks.js] pushCalendarUpdateTask: local development detected, calling handler directly
   [processCalendarUpdateTask] handler: generating tickets for template: ...
   ```

### Production

In production, tasks are queued and processed asynchronously:

1. Create a template with `isActive: true`
2. Tickets should appear within 1-5 minutes
3. Check Cloud Tasks queue in Google Cloud Console

### Verification

To verify calendar tasks were created:

```typescript
// Check tickets for a specific template
const verifyCalendarTasks = async (templateId, accountId, siteId) => {
  const tickets = await fetch(
    `/v1/accounts/${accountId}/tickets?type=PLANNED&siteId=${siteId}`
  ).then(r => r.json());
  
  const templateTickets = tickets.filter(
    t => t.plannedMaintenanceTemplateId === templateId
  );
  
  console.log(`Found ${templateTickets.length} tickets for template ${templateId}`);
  
  // Check date range (should cover next 52 weeks)
  const now = new Date();
  const futureDate = new Date();
  futureDate.setDate(futureDate.getDate() + (52 * 7));
  
  const futureTickets = templateTickets.filter(t => {
    const scheduled = new Date(t.scheduled_date);
    return scheduled >= now && scheduled <= futureDate;
  });
  
  console.log(`Found ${futureTickets.length} future tickets (next 52 weeks)`);
  
  return {
    total: templateTickets.length,
    future: futureTickets.length,
    tickets: futureTickets
  };
};
```

## Important Notes

### 1. Asynchronous Processing

- Calendar task creation is **asynchronous** and **non-blocking**
- API responses return immediately
- Tickets may take a few seconds (local) or minutes (production) to appear

### 2. Active Templates Only

- Calendar tasks are only created for templates with `isActive: true`
- Inactive templates won't generate calendar tasks

### 3. Template Updates

- When a template is updated, **all future tickets are deleted** and regenerated
- This ensures calendar always reflects the latest template configuration
- Past tickets are not affected

### 4. Date Format

- `scheduled_date` is in ISO datetime format: `YYYY-MM-DDTHH:mm:ssZ`
- Includes time component from recurrence pattern (e.g., "08:00" becomes "08:00:00Z")
- Use this directly with FullCalendar

### 5. Error Resilience

- Calendar task failures don't block template creation/update
- Template is saved even if calendar task fails
- Check for tickets after a delay to detect failures

## Troubleshooting

### Tickets Not Appearing

1. **Check template is active:**
   ```typescript
   const template = await getTemplate(templateId);
   if (!template.isActive) {
     console.log('Template is inactive - no tickets will be generated');
   }
   ```

2. **Check for errors in console:**
   - Look for error messages from `processCalendarUpdateTask`
   - Check Cloud Tasks queue in production

3. **Poll with longer timeout:**
   ```typescript
   await pollForTickets(templateId, accountId, siteId, 60); // 60 attempts
   ```

4. **Manually trigger calendar update:**
   - In production, check Cloud Tasks queue
   - Manually trigger the task if needed

### Performance Considerations

- Polling should be done with reasonable intervals (1-2 seconds)
- Limit polling attempts (20-30 attempts max)
- Show loading states to users
- Consider using WebSockets or Server-Sent Events for real-time updates (future enhancement)

## Summary

1. **Automatic**: Calendar tasks are created automatically when templates are created/updated
2. **Asynchronous**: Process happens in background, API returns immediately
3. **Non-blocking**: Template creation/update succeeds even if calendar task fails
4. **Polling**: Frontend should poll for tickets or show optimistic UI
5. **FullCalendar**: Tickets are regular PLANNED tickets, map them to calendar events

## Questions?

If you have questions about:
- Calendar task creation timing
- Polling strategies
- Error handling
- FullCalendar integration

Please refer to the OpenAPI specification at `/openapi.json` or contact the backend team.

