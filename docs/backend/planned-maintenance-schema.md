# Planned Maintenance API Schema for Frontend

## Overview
This document provides the complete schema and API endpoints for implementing the planned maintenance feature in the frontend application.

## API Base URL
```
https://us-central1-maintor.cloudfunctions.net/maintor-api
```

## Authentication
All requests require Firebase Authentication:
```
Authorization: Bearer {firebase-id-token}
```

## API Endpoints

### 1. List Planned Maintenance Templates
```http
GET /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances
```

**Query Parameters:**
- `isActive` (optional): `boolean` - Filter by active status

**Response:**
```json
[
  {
    "id": "template_123",
    "accountId": "account_456",
    "siteId": "site_789",
    "title": "Weekly Compressor Maintenance",
    "description": "Regular maintenance for all compressors",
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
      "description": "Weekly inspection and maintenance",
      "priority": 3,
      "tasks": [
        {
          "description": "Check oil levels"
        },
        {
          "description": "Inspect belts"
        }
      ],
      "estimatedDurationMin": 60,
      "ownerUserId": "user_123",
      "assignees": [
        {
          "user_id": "user_456"
        }
      ]
    },
    "isActive": true,
    "created": "2024-01-15T10:30:00Z",
    "updated": "2024-01-15T10:30:00Z",
    "createdByUserId": "user_123"
  }
]
```

### 2. Create Planned Maintenance Template
```http
POST /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances
```

**Request Body:**
```json
{
  "plannedMaintenance": {
    "title": "Weekly Compressor Maintenance",
    "description": "Regular maintenance for all compressors",
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
      "description": "Weekly inspection and maintenance",
      "priority": 3,
      "tasks": [
        {
          "description": "Check oil levels"
        },
        {
          "description": "Inspect belts"
        }
      ],
      "estimatedDurationMin": 60,
      "ownerUserId": "user_123",
      "assignees": [
        {
          "user_id": "user_456"
        }
      ]
    },
    "isActive": true
  }
}
```

### 3. Get Single Template
```http
GET /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances/{templateId}
```

### 4. Update Template
```http
PATCH /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances/{templateId}
```

**Request Body:** Same as create, but all fields are optional

### 5. Delete Template
```http
DELETE /v1/accounts/{accountId}/sites/{siteId}/planned-maintenances/{templateId}
```

## Schema Definitions

### PlannedMaintenance (Response)
```typescript
interface PlannedMaintenance {
  id: string;
  accountId: string;
  siteId: string;
  title: string;
  description?: string;
  assetIds: string[];
  generateTicketPerAsset: boolean;
  recurrence: RecurrencePattern[];
  ticketTemplate: TicketTemplate;
  isActive: boolean;
  created: string; // ISO datetime
  updated: string; // ISO datetime
  createdByUserId: string;
}
```

### PlannedMaintenanceInput (Create/Update)
```typescript
interface PlannedMaintenanceInput {
  title: string; // Required
  description?: string;
  assetIds: string[]; // Required - array of asset IDs
  generateTicketPerAsset: boolean; // Required
  recurrence: RecurrencePattern[]; // Required
  ticketTemplate: TicketTemplate; // Required
  isActive: boolean; // Required
}
```

### RecurrencePattern
```typescript
interface RecurrencePattern {
  frequency: "WEEKLY"; // Currently only WEEKLY supported
  dayOfWeek: number[]; // 0=Sunday, 1=Monday, ..., 6=Saturday
  interval: number; // Every X weeks (minimum 1)
  time: string; // HH:mm format (e.g., "08:00", "14:30")
}
```

### TicketTemplate
```typescript
interface TicketTemplate {
  title: string; // Required
  description: string; // Required
  priority: number; // Required - 1-5
  tasks: TaskTemplate[]; // Required
  estimatedDurationMin?: number; // Optional - minutes
  ownerUserId: string; // Required
  assignees?: Assignee[]; // Optional
}
```

### TaskTemplate
```typescript
interface TaskTemplate {
  description: string; // Required
}
```

### Assignee
```typescript
interface Assignee {
  user_id: string; // Required
}
```

## Form Implementation Guide

### 1. Basic Form Fields
- **Title**: Text input (required)
- **Description**: Textarea (optional)
- **Asset Selection**: Multi-select from available assets
- **Generate Ticket Per Asset**: Toggle/checkbox
- **Is Active**: Toggle/checkbox

### 2. Recurrence Configuration
- **Frequency**: Currently only "WEEKLY" (dropdown, disabled)
- **Days of Week**: Multi-select checkboxes (Sunday=0, Monday=1, etc.)
- **Interval**: Number input (minimum 1, default 1)
- **Time**: Time picker (HH:mm format)

### 3. Ticket Template Section
- **Title**: Text input (required)
- **Description**: Textarea (required)
- **Priority**: Number input (1-5, required)
- **Estimated Duration**: Number input (minutes, optional)
- **Owner User**: User selector (required)
- **Assignees**: Multi-user selector (optional)
- **Tasks**: Dynamic list of task descriptions

### 4. Task Management
Allow users to add/remove tasks dynamically:
- Add Task button
- Each task has a description field
- Remove task button for each task
- Minimum 1 task required

## Validation Rules

### Frontend Validation
1. **Title**: Required, non-empty string
2. **AssetIds**: Required, at least 1 asset selected
3. **Recurrence**: 
   - At least 1 day of week selected
   - Interval >= 1
   - Time in HH:mm format
4. **Ticket Template**:
   - Title required
   - Description required
   - Priority between 1-5
   - At least 1 task
   - Owner user required

### API Validation
The API will return 400 errors with detailed validation messages if any rules are violated.

## Example Form Data

```json
{
  "title": "Weekly Compressor Maintenance",
  "description": "Regular maintenance for all compressors in Building A",
  "assetIds": ["compressor_001", "compressor_002", "compressor_003"],
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
    "description": "Weekly inspection and maintenance of compressor",
    "priority": 3,
    "tasks": [
      {
        "description": "Check oil levels and top up if needed"
      },
      {
        "description": "Inspect belts for wear and tension"
      },
      {
        "description": "Check for unusual vibrations or noises"
      }
    ],
    "estimatedDurationMin": 45,
    "ownerUserId": "supervisor_123",
    "assignees": [
      {
        "user_id": "technician_456"
      },
      {
        "user_id": "technician_789"
      }
    ]
  },
  "isActive": true
}
```

## Error Handling

### Common Error Responses
```json
// 400 Bad Request
{
  "issues": [
    {
      "code": "invalid_type",
      "expected": "string",
      "received": "undefined",
      "path": ["title"],
      "message": "Required"
    }
  ]
}

// 403 Forbidden
"Forbidden"

// 404 Not Found
"Planned maintenance not found"

// 500 Internal Server Error
"Error creating planned maintenance"
```

## Notes for Implementation

1. **Asset Selection**: You'll need to fetch available assets from the existing assets API
2. **User Selection**: You'll need to fetch available users for owner/assignee selection
3. **Time Format**: Always use 24-hour format (HH:mm)
4. **Day of Week**: Use 0-6 mapping (0=Sunday, 6=Saturday)
5. **Template Creation**: When a template is created with `isActive: true`, it immediately generates 52 weeks of tickets
6. **Template Updates**: When updating a template, all future tickets are deleted and recreated
7. **Asset Name Placeholder**: In ticket titles, `{assetName}` can be used as a placeholder that will be replaced with the actual asset name

## Testing

Use the API endpoints to test your form implementation. The system will automatically generate tickets when templates are created or updated.
