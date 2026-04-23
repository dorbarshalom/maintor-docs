# Maintor API Design Document

## Overview

Maintor is a maintenance management system for industrial companies. The API provides endpoints for managing maintenance tickets, which can be either planned maintenance or breakdown maintenance.

## Base URL
```
https://your-api-domain.com/v1
```

## Authentication

All endpoints require authentication via the `identityValidator` middleware. The API uses Firebase authentication with Google OAuth or email/password.

### Headers
```
Authorization: Bearer <firebase-token>
Content-Type: application/json
```

## Data Models

### Ticket Model

The unified ticket model supports both planned and breakdown maintenance:

```typescript
interface Ticket {
  id: string;                     // unique identifier
  type: 'PLANNED' | 'BREAKDOWN'; // ticket type
  assetId?: string;               // Asset ID - required for breakdown tickets

  // Basic Info
  title: string;                  // Ticket title
  description: string;            // Description
  priority: number;               // 1-5 (1=emergency, 5=low)
  status: 'OPEN' | 'IN_PROGRESS' | 'COMPLETED' | 'CLOSED' | 'SKIPPED';

  // Timing
  scheduled_date?: string;        // When planned maintenance should happen (only for planned tickets)
  
  // Timeline (shared by both planned and breakdown tickets)
  timeline?: {
    started?: string;             // When work/breakdown started (ISO datetime)
    ended?: string;               // When work/breakdown ended (ISO datetime)
    duration_min?: number;        // Total duration in minutes
    downtime_min?: number;        // Machine downtime in minutes
    labor_entries?: Array<{      // Detailed labor tracking
      user_id: string;            // User ID
      started: string;            // Start time (ISO datetime)
      ended: string;              // End time (ISO datetime)
      duration_min: number;       // Duration for this labor entry (minutes)
    }>;
  };

  // Breakdown-specific fields (only for type: "BREAKDOWN")
  breakdown?: {
    problem_description: string;  // What went wrong - REQUIRED
    solution_description?: string; // How it was fixed
    root_cause?: string;          // From root cause catalog
  };

  // Operator notes (relevant for both types)
  operator_notes?: string;       // Additional context

  // Tasks (planned) or Notes (breakdown)
  tasks?: Array<{
    description: string;
    status: 'PENDING' | 'DONE' | 'SKIPPED' | 'FAILED';
  }>;

  notes?: Array<{
    at: string;                  // ISO datetime
    by: string;                  // user_id
    note: string;
  }>;

        // People
        owner_user_id: string;         // Owner (planner/supervisor)
        reported_by_user_id: string;  // Who reported the breakdown
        assignees: Array<{
          user_id: string;
        }>;

  // Evidence & Documentation
  photos?: Array<{
    url: string;
    caption?: string;
  }>;

  attachments?: Array<{
    doc_id: string;
    title: string;
    kind: string;
  }>;

  // Audit
  created_at: string;           // ISO datetime
  updated_at: string;           // ISO datetime
  archived_at?: string;         // ISO datetime
}
```

## API Endpoints

### Account Management

#### Get My Account
```http
GET /accounts/myAccount
```

**Response:**
```json
{
  "id": "account_123",
  "name": "Acme Corp",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

### Ticket Management

#### List Tickets
```http
GET /accounts/{accountId}/tickets
```

**Query Parameters:**
- `type` (optional): Filter by ticket type (`planned` | `breakdown`)
- `status` (optional): Filter by status

**Example:**
```http
GET /accounts/acme/tickets?type=breakdown&status=open
```

**Response:**
```json
[
  {
    "id": "tkt_2025_001245",
    "type": "breakdown",
    "title": "Pump failure on Line 3",
    "priority": 1,
    "status": "open",
    "scheduled_date": "2025-09-14",
    "assetId": "asset_pmp_004",
    "created_at": "2025-09-14T13:40:00Z"
  }
]
```

#### Create Ticket
```http
POST /accounts/{accountId}/tickets
```

**Request Body:**
```json
{
  "ticket": {
    "type": "planned",
    "title": "500-hour pump service",
    "description": "Routine maintenance per schedule",
    "priority": 3,
    "status": "open",
    "scheduled_date": "2025-09-26",
    "assetId": "asset_pmp_004",
    "owner_user_id": "tu_acme_super1",
    "reported_by_user_id": "tu_acme_super1",
    "assignees": [
      { "user_id": "tu_acme_alex" }
    ],
    "tasks": [
      {
        "description": "Inspect coupling & alignment",
        "status": "pending"
      }
    ]
  }
}
```

**Response:** `201 Created`
```json
{
  "id": "tkt_2025_001245",
  "type": "planned",
  "title": "500-hour pump service",
  "status": "open",
  "created_at": "2025-09-26T06:55:00Z"
}
```

#### Get Ticket
```http
GET /accounts/{accountId}/tickets/{ticketId}
```

**Response:** `200 OK`
```json
{
  "id": "tkt_2025_001245",
  "type": "planned",
  "title": "500-hour pump service",
  "description": "Routine maintenance per schedule",
  "priority": 3,
  "status": "open",
  "scheduled_date": "2025-09-26",
        "timeline": {
          "started": "2025-09-26T07:15:00Z",
          "ended": "2025-09-26T08:55:00Z",
          "duration_min": 100,
          "downtime_min": 30,
          "labor_entries": [
            {
              "user_id": "tu_acme_alex",
              "started": "2025-09-26T07:15:00Z",
              "ended": "2025-09-26T08:55:00Z",
              "duration_min": 100
            }
          ]
        },
  "tasks": [
    {
      "description": "Inspect coupling & alignment",
      "status": "DONE"
    }
  ],
  "assignees": [
    { "user_id": "tu_acme_alex" }
  ],
  "created_at": "2025-09-26T06:55:00Z",
  "updated_at": "2025-09-26T09:00:00Z"
}
```

#### Update Ticket
```http
PATCH /accounts/{accountId}/tickets/{ticketId}
```

**Request Body:**
```json
{
  "status": "in_progress",
  "actual_work": {
    "started": "2025-09-26T07:15:00Z"
  },
  "tasks": [
    {
      "description": "Inspect coupling & alignment",
      "status": "done"
    }
  ]
}
```

**Response:** `200 OK`
```json
{
  "id": "tkt_2025_001245",
  "status": "in_progress",
  "actual_work": {
    "started": "2025-09-26T07:15:00Z"
  },
  "updated_at": "2025-09-26T09:30:00Z"
}
```

#### Delete Ticket
```http
DELETE /accounts/{accountId}/tickets/{ticketId}
```

**Response:** `200 OK`
```json
{
  "success": true
}
```

## Error Responses

### 400 Bad Request
```json
{
  "issues": [
    {
      "path": ["priority"],
      "message": "Number must be greater than or equal to 1"
    }
  ]
}
```

### 401 Unauthorized
```json
"Unauthorized"
```
*Returned when authentication fails (invalid/missing token)*

### 403 Forbidden
```json
"Forbidden"
```
*Returned when user doesn't have access to the account*

### 404 Not Found
```json
"Ticket not found"
```

### 500 Internal Server Error
```json
"Error creating ticket"
```

## Priority Scale

- **1** = Emergency (highest priority)
- **2** = High
- **3** = Medium  
- **4** = Low
- **5** = Very Low (lowest priority)

## Status Flow

### Planned Maintenance
```
draft → open → in_progress → completed → closed
```

### Breakdown Maintenance
```
draft → open → in_progress → awaiting_approval → closed
```

## Example Usage

### Creating a Breakdown Ticket
```javascript
const response = await fetch('/v1/accounts/acme/tickets', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer <firebase-token>',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    ticket: {
      type: 'breakdown',
      title: 'Pump failure on Line 3',
      description: 'Seal leak causing motor overload trip',
      priority: 1,
      status: 'open',
      assetId: 'asset_pmp_004',
      breakdown: {
        started: '2025-09-14T13:22:00Z',
        problem_description: 'Seal leak; pump tripped on motor overload.',
        root_cause: 'improper_alignment'
      },
      owner_user_id: 'tu_acme_super1',
      reported_by_user_id: 'usr_op_42',
      assignees: [
        { user_id: 'tu_acme_alex' }
      ]
    }
  })
});
```

### Updating Ticket Status
```javascript
const response = await fetch('/v1/accounts/acme/tickets/tkt_123', {
  method: 'PATCH',
  headers: {
    'Authorization': 'Bearer <firebase-token>',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'in_progress',
    actual_work: {
      started: '2025-09-14T13:45:00Z'
    }
  })
});
```

### Filtering Tickets
```javascript
// Get all breakdown tickets
const response = await fetch('/v1/accounts/acme/tickets?type=breakdown');

// Get open tickets only
const response = await fetch('/v1/accounts/acme/tickets?status=open');

// Get breakdown tickets that are open
const response = await fetch('/v1/accounts/acme/tickets?type=breakdown&status=open');
```

## Notes for Frontend Development

1. **Authentication**: All requests must include the Firebase token in the Authorization header
2. **Account-based URLs**: All ticket operations require `accountId` in the URL path
3. **Type-specific fields**: Some fields like `breakdown` are only relevant for breakdown tickets
4. **Status management**: Different ticket types have different status flows
5. **Filtering**: Use query parameters to filter tickets by type and status
6. **Error handling**: Check for 401/403 errors to handle authentication/authorization issues
7. **Date formats**: All dates are in ISO 8601 format (UTC)
8. **Priority**: Lower numbers indicate higher priority (1 = emergency)
9. **Timing fields**: 
   - **Planned tickets**: Must have `scheduled_date` (when maintenance is planned)
   - **Breakdown tickets**: Must have `breakdown.started` (when breakdown occurred), should NOT have `scheduled_date`
10. **Asset association**: 
    - **Breakdown tickets**: Must have `assetId` field (Asset ID)
    - **Planned tickets**: May have `assetId` field (optional)
