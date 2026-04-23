# Backend Task: Account Settings API Implementation

## Overview

Implement API endpoints to manage all account-level settings that need to be shared between the maintor-app (web application) and maintor-eng (technician mobile application). This includes:

1. **Root Causes** - Customizable root cause options for breakdown maintenance tickets (already implemented)
2. **Ticket Statuses** - Enable/disable ticket status options (OPEN, IN_PROGRESS, COMPLETED, CLOSED, SKIPPED)
3. **Form Settings** - Field visibility settings for breakdown and planned maintenance forms (optional, for future use)

Currently, root causes are already implemented via API, but ticket statuses and form settings are stored in localStorage in the frontend, which prevents synchronization between the two applications.

This unified API will centralize all account settings in the database, allowing both applications to access and modify the same settings.

## Part 1: Root Causes API (Already Implemented)

Root causes are already implemented and working. This section documents the existing API for reference.

### 1.1 Database Schema

Create a `root_causes` collection/table with the following structure:

```typescript
interface RootCause {
  id: string                    // Unique identifier
  accountId: string            // Account this root cause belongs to
  label: string                // Display name (e.g., "Mechanical")
  value: string                // Slug/value (e.g., "mechanical") - must be unique per account
  order?: number               // Optional: for sorting/ordering
  created: string              // ISO datetime
  updated: string              // ISO datetime
}
```

**Constraints:**
- `value` must be unique per account (case-insensitive)
- `label` should be unique per account (case-insensitive, but allow flexibility)
- `accountId` is required and must reference a valid account

### 1.2 API Endpoints

#### 1.2.1 List Root Causes

```
GET /v1/accounts/{accountId}/root-causes
Authorization: IDTOKEN.<firebaseIdToken>
```

**Response** (200 OK):
```json
[
  {
    "id": "rc123",
    "accountId": "account456",
    "label": "Mechanical",
    "value": "mechanical",
    "order": 1,
    "created": "2025-01-15T10:30:00Z",
    "updated": "2025-01-15T10:30:00Z"
  },
  {
    "id": "rc124",
    "accountId": "account456",
    "label": "Electrical",
    "value": "electrical",
    "order": 2,
    "created": "2025-01-15T10:30:00Z",
    "updated": "2025-01-15T10:30:00Z"
  }
]
```

**Error Responses:**
- `401`: Unauthorized (invalid or missing token)
- `403`: Forbidden (user doesn't have access to this account)
- `404`: Account not found

**Notes:**
- Return empty array `[]` if account has no root causes
- Results should be sorted by `order` (if present), then by `label` alphabetically

---

#### 1.2.2 Create Root Cause

```
POST /v1/accounts/{accountId}/root-causes
Authorization: IDTOKEN.<firebaseIdToken>
Content-Type: application/json

Body:
{
  "rootCause": {
    "label": "Mechanical",
    "value": "mechanical"  // Optional - if not provided, generate from label
  }
}
```

**Request Body:**
- `label` (required): Display name for the root cause
- `value` (optional): Slug/value. If not provided, generate from label:
  - Convert to lowercase
  - Replace spaces and special characters with underscores
  - Remove leading/trailing underscores
  - Example: "Raw Material" → "raw_material"

**Response** (201 Created):
```json
{
  "id": "rc123",
  "accountId": "account456",
  "label": "Mechanical",
  "value": "mechanical",
  "order": 1,
  "created": "2025-01-15T10:30:00Z",
  "updated": "2025-01-15T10:30:00Z"
}
```

**Error Responses:**
- `400`: Bad Request
  - Missing `label`
  - `value` already exists for this account (case-insensitive)
  - `label` already exists for this account (case-insensitive, optional validation)
- `401`: Unauthorized
- `403`: Forbidden (user must have OWNER, ADMIN, or SITE_MANAGER role for the account)
- `404`: Account not found

**Validation:**
- `label`: Required, non-empty string, max 100 characters
- `value`: Must be unique per account (case-insensitive), alphanumeric + underscores only
- Auto-generate `value` from `label` if not provided

---

#### 1.2.3 Update Root Cause

```
PATCH /v1/accounts/{accountId}/root-causes/{rootCauseId}
Authorization: IDTOKEN.<firebaseIdToken>
Content-Type: application/json

Body:
{
  "label": "Updated Label",
  "order": 5
}
```

**Request Body:**
- `label` (optional): Updated display name
- `order` (optional): Updated sort order
- `value` (optional): Updated slug (must still be unique)

**Response** (200 OK):
```json
{
  "id": "rc123",
  "accountId": "account456",
  "label": "Updated Label",
  "value": "mechanical",
  "order": 5,
  "created": "2025-01-15T10:30:00Z",
  "updated": "2025-01-15T11:00:00Z"
}
```

**Error Responses:**
- `400`: Bad Request (validation errors, duplicate value/label)
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Root cause not found

---

#### 1.2.4 Delete Root Cause

```
DELETE /v1/accounts/{accountId}/root-causes/{rootCauseId}
Authorization: IDTOKEN.<firebaseIdToken>
```

**Response** (200 OK):
```json
{
  "message": "Root cause deleted successfully"
}
```

**Error Responses:**
- `401`: Unauthorized
- `403`: Forbidden (user must have OWNER, ADMIN, or SITE_MANAGER role)
- `404`: Root cause not found or doesn't belong to this account

**Notes:**
- Check that root cause belongs to the specified account
- Consider if root causes are referenced in existing tickets (see "Data Integrity" section)

---

### 1.3 Default Values for Root Causes

When a new account is created, automatically initialize it with the following default root causes:

1. **Mechanical** (value: `mechanical`)
2. **Electrical** (value: `electrical`)
3. **Control** (value: `control`)
4. **Raw Material** (value: `raw_material`)
5. **Operational** (value: `operational`)

**Implementation:**
- Trigger this initialization when:
  - A new account is created via `POST /v1/accounts` (if this endpoint exists)
  - A user signs up and account is auto-created via `GET /v1/accounts/myAccount` (first-time access)
- Set `order` values: 1, 2, 3, 4, 5 respectively
- Use current timestamp for `created` and `updated`

**Migration for Existing Accounts:**
- Consider adding a migration script to initialize default root causes for existing accounts that don't have any
- Or let them be initialized on first access (lazy initialization)

---

## Part 2: Account Settings API (Ticket Statuses & Form Settings)

### 2.1 Database Schema

Create an `account_settings` collection/table with the following structure:

```typescript
interface AccountSettings {
  id: string                    // Unique identifier
  accountId: string            // Account these settings belong to (unique constraint)
  ticketStatuses: {
    OPEN: boolean
    IN_PROGRESS: boolean
    COMPLETED: boolean
    CLOSED: boolean
    SKIPPED: boolean
  }
  formSettings?: {              // Optional - for future use
    breakdownForm: {
      priority: boolean
      status: boolean
      description: boolean
      breakdownEnded: boolean
      laborEntries: boolean
      rootCause: boolean
      problemDescription: boolean
      solutionDescription: boolean
      downtime: boolean
      photos: boolean
      assignees: boolean
      assetOwner: boolean
      reportedBy: boolean
      operatorNotes: boolean
    }
    plannedMaintenanceForm: {
      description: boolean
      generateTicketPerAsset: boolean
      isActive: boolean
      interval: boolean
      duration: boolean
      assignees: boolean
    }
    dashboard: {
      autoRefresh: boolean
      autoRefreshInterval: number  // minutes (1-60)
    }
  }
  created: string              // ISO datetime
  updated: string              // ISO datetime
}
```

**Constraints:**
- `accountId` must be unique (one settings document per account)
- `accountId` must reference a valid account
- All ticket status boolean values are required
- `formSettings` is optional but if present, all nested fields should be validated

**Indexes:**
- Create unique index on `accountId` for fast lookups

---

### 2.2 API Endpoints

#### 2.2.1 Get Account Settings

```
GET /v1/accounts/{accountId}/settings
Authorization: IDTOKEN.<firebaseIdToken>
```

**Response** (200 OK):
```json
{
  "id": "settings123",
  "accountId": "account456",
  "ticketStatuses": {
    "OPEN": true,
    "IN_PROGRESS": true,
    "COMPLETED": true,
    "CLOSED": true,
    "SKIPPED": false
  },
  "formSettings": {
    "breakdownForm": {
      "priority": true,
      "status": true,
      "description": true,
      "breakdownEnded": true,
      "laborEntries": true,
      "rootCause": true,
      "problemDescription": true,
      "solutionDescription": true,
      "downtime": true,
      "photos": true,
      "assignees": true,
      "assetOwner": true,
      "reportedBy": true,
      "operatorNotes": true
    },
    "plannedMaintenanceForm": {
      "description": true,
      "generateTicketPerAsset": true,
      "isActive": true,
      "interval": true,
      "duration": true,
      "assignees": true
    },
    "dashboard": {
      "autoRefresh": false,
      "autoRefreshInterval": 5
    }
  },
  "created": "2025-01-15T10:30:00Z",
  "updated": "2025-01-15T11:00:00Z"
}
```

**Error Responses:**
- `401`: Unauthorized (invalid or missing token)
- `403`: Forbidden (user doesn't have access to this account)
- `404`: Account not found

**Special Case - 404 Handling:**
- If settings don't exist for the account, return `404 Not Found`
- Frontend will handle this by creating default settings or using localStorage fallback
- **Alternative**: Implement auto-creation on first GET (see "Default Values" section) - return defaults immediately if missing

**Notes:**
- If `formSettings` is not present in the database, it can be omitted from the response (frontend will use defaults)
- All ticket statuses must be present in the response

---

#### 2.2.2 Update Account Settings

```
PATCH /v1/accounts/{accountId}/settings
Authorization: IDTOKEN.<firebaseIdToken>
Content-Type: application/json
```

**Request Body Examples:**

Update ticket statuses only:
```json
{
  "settings": {
    "ticketStatuses": {
      "SKIPPED": false
    }
  }
}
```

Update form settings only:
```json
{
  "settings": {
    "formSettings": {
      "dashboard": {
        "autoRefresh": true,
        "autoRefreshInterval": 10
      }
    }
  }
}
```

Update both:
```json
{
  "settings": {
    "ticketStatuses": {
      "SKIPPED": false
    },
    "formSettings": {
      "breakdownForm": {
        "priority": false
      }
    }
  }
}
```

**Request Body Rules:**
- `settings` object is required
- `ticketStatuses` (optional): Partial updates allowed - only include statuses that are changing
- `formSettings` (optional): Partial updates allowed - only include sections/fields that are changing
- Use merge strategy: Update only the fields provided, keep existing values for fields not included

**Response** (200 OK):
```json
{
  "id": "settings123",
  "accountId": "account456",
  "ticketStatuses": {
    "OPEN": true,
    "IN_PROGRESS": true,
    "COMPLETED": true,
    "CLOSED": true,
    "SKIPPED": false
  },
  "formSettings": {
    // ... full formSettings object with updated values merged
  },
  "created": "2025-01-15T10:30:00Z",
  "updated": "2025-01-15T11:30:00Z"
}
```

**Error Responses:**
- `400`: Bad Request
  - Invalid ticket status key (must be one of: OPEN, IN_PROGRESS, COMPLETED, CLOSED, SKIPPED)
  - Invalid boolean value
  - Invalid formSettings structure
  - Invalid autoRefreshInterval (must be 1-60 if autoRefresh is true)
- `401`: Unauthorized
- `403`: Forbidden (user must have OWNER, ADMIN, or SITE_MANAGER role for the account)
- `404`: Account not found or settings not found (if auto-creation is not implemented)

**Validation:**
- `ticketStatuses`: All keys must be valid status values (case-sensitive)
- `ticketStatuses`: All values must be booleans
- `formSettings.breakdownForm`: All values must be booleans
- `formSettings.plannedMaintenanceForm`: All values must be booleans
- `formSettings.dashboard.autoRefresh`: Must be boolean
- `formSettings.dashboard.autoRefreshInterval`: Must be number between 1-60 (minutes)

**Update Strategy:**
- **Merge, don't replace**: If only `ticketStatuses.SKIPPED` is sent, only update that field, keep all other statuses unchanged
- **Nested objects**: For `formSettings`, merge at the section level (breakdownForm, plannedMaintenanceForm, dashboard)
- **Deep merge**: Within each section, merge individual fields

**Auto-Creation on PATCH:**
- If settings don't exist when PATCH is called, create them with defaults first, then apply the update
- This ensures settings exist after first modification

---

### 2.3 Default Values for Account Settings

When a new account is created, automatically initialize it with default settings:

**Default Ticket Statuses:**
```json
{
  "OPEN": true,
  "IN_PROGRESS": true,
  "COMPLETED": true,
  "CLOSED": true,
  "SKIPPED": true
}
```

**Default Form Settings:**
```json
{
  "breakdownForm": {
    "priority": true,
    "status": true,
    "description": true,
    "breakdownEnded": true,
    "laborEntries": true,
    "rootCause": true,
    "problemDescription": true,
    "solutionDescription": true,
    "downtime": true,
    "photos": true,
    "assignees": true,
    "assetOwner": true,
    "reportedBy": true,
    "operatorNotes": true
  },
  "plannedMaintenanceForm": {
    "description": true,
    "generateTicketPerAsset": true,
    "isActive": true,
    "interval": true,
    "duration": true,
    "assignees": true
  },
  "dashboard": {
    "autoRefresh": false,
    "autoRefreshInterval": 5
  }
}
```

**Implementation Options:**

**Option A: Auto-Create on Account Creation (Recommended)**
- When a new account is created via `POST /v1/accounts` or auto-created via `GET /v1/accounts/myAccount`
- Automatically create default settings document
- Use current timestamp for `created` and `updated`

**Option B: Lazy Initialization**
- Create settings document on first GET request if it doesn't exist
- Return default values immediately
- This allows frontend to always get a response without handling 404

**Option C: Create on First PATCH**
- If settings don't exist when PATCH is called, create them with defaults first, then apply the update
- This ensures settings exist after first modification

**Recommendation:** Use Option A (auto-create on account creation) + Option B (lazy initialization as fallback) for robustness.

---

### 3.1 Root Causes Permissions

**Who can manage root causes:**
- `OWNER` (account-level role)
- `ADMIN` (account-level role or site-level with `ALL_SITES`)
- `SITE_MANAGER` (site-level role) - optional, can be restricted to account-level only

**Who can view root causes:**
- All authenticated users with access to the account

### 3.2 Account Settings Permissions

**Who can view settings:**
- All authenticated users with access to the account
- This includes: OWNER, ADMIN, SITE_MANAGER, CONSULTANT, and regular users

**Who can modify settings:**
- `OWNER` (account-level role)
- `ADMIN` (account-level role or site-level with `ALL_SITES`)
- `SITE_MANAGER` (site-level role) - **Optional**: Can be restricted to account-level roles only

**Rationale:** Settings affect the entire account, so they should typically be managed by account administrators. However, if you want to allow site managers to customize settings for their workflow, that's acceptable.

---

### 4. Data Integrity

#### 4.1 Root Causes Considerations

1. **Existing Tickets:**
   - Root causes are stored as strings in ticket `breakdown.root_cause` field
   - If a root cause is deleted, existing tickets that reference it should still be valid
   - The root cause value in tickets is just a string, not a foreign key
   - **Recommendation**: Allow deletion even if referenced in tickets (tickets keep the string value)

2. **Uniqueness:**
   - `value` must be unique per account (case-insensitive)
   - Example: Cannot have both "Mechanical" and "mechanical" in the same account
   - Check uniqueness before insert/update

3. **Slug Generation:**
   - If `value` is not provided, generate from `label`:
     ```javascript
     function generateSlug(label) {
       return label
         .toLowerCase()
         .trim()
         .replace(/[^\w\s-]/g, '')      // Remove special characters
         .replace(/[\s_-]+/g, '_')      // Replace spaces/hyphens with underscores
         .replace(/^-+|-+$/g, '')       // Remove leading/trailing dashes
     }
     ```

#### 4.2 Account Settings Considerations

1. **Ticket Status References:**
   - Tickets store status as a string (e.g., "OPEN", "IN_PROGRESS")
   - If a status is disabled (set to `false`), existing tickets with that status should still be valid
   - The status value in tickets is just a string, not a foreign key
   - **Recommendation**: Allow disabling statuses even if tickets use them (tickets keep their status string)

2. **Uniqueness:**
   - Only one settings document per account (enforced by unique index on `accountId`)
   - If settings don't exist, create them; if they exist, update them

3. **Partial Updates:**
   - PATCH should merge updates, not replace entire document
   - If only `ticketStatuses.SKIPPED` is updated, keep all other fields unchanged
   - This allows frontend to send minimal updates

4. **Form Settings Optional:**
   - `formSettings` is optional - accounts may not have it set
   - Frontend will use defaults if `formSettings` is missing
   - This allows gradual rollout of form settings feature

---

### 5. Migration for Existing Accounts

#### 5.1 Root Causes Migration

**Strategy:**
- Consider adding a migration script to initialize default root causes for existing accounts that don't have any
- Or let them be initialized on first access (lazy initialization)

#### 5.2 Account Settings Migration

**Strategy:**

1. **Lazy Migration (Recommended):**
   - When an existing account first accesses settings (GET request)
   - If settings don't exist, create them with defaults
   - This happens automatically without a separate migration script

2. **Bulk Migration (Optional):**
   - Create a one-time migration script to initialize settings for all existing accounts
   - Use default values for all accounts
   - Run this after deploying the API

3. **User-Initiated Migration:**
   - Frontend can detect if settings are missing
   - Frontend can call PATCH with default values to initialize
   - Backend creates settings document on first PATCH

**Recommendation:** Use lazy migration (Option 1) - it's simplest and doesn't require a separate migration step.

---

### 6. Example Implementation Flow

**Account Creation:**
```
1. User signs up → Account created
2. Trigger: Initialize default root causes
3. Create 5 root causes with default values
4. Set order: 1-5
5. Trigger: Initialize default account settings
6. Create settings document with all defaults
```

**User Updates Ticket Status:**
```
1. Frontend sends: PATCH /accounts/{id}/settings
   Body: { settings: { ticketStatuses: { SKIPPED: false } } }
2. Backend:
   - Verify account exists
   - Verify user has permission
   - Load existing settings (or create with defaults if missing)
   - Merge update (only change SKIPPED, keep others)
   - Update timestamp
   - Return full settings object
3. Frontend receives updated settings
```

**User Adds Root Cause:**
```
1. Frontend sends: { rootCause: { label: "Vibration" } }
2. Backend generates: value = "vibration"
3. Check uniqueness (case-insensitive)
4. Create root cause
5. Return created object
```

**User Deletes Root Cause:**
```
1. Frontend sends: DELETE /accounts/{id}/root-causes/{rootCauseId}
2. Backend verifies:
   - Root cause exists
   - Belongs to account
   - User has permission
3. Delete root cause
4. Return success message
```

**User Views Settings:**
```
1. Frontend sends: GET /accounts/{id}/settings
2. Backend:
   - Verify account exists
   - Verify user has access
   - Load settings (or create with defaults if missing - if lazy init implemented)
   - Return settings object
3. Frontend displays settings
```

---

### 7. Testing Checklist

#### 7.1 Root Causes

- [ ] List root causes returns empty array for new account
- [ ] List root causes returns default values after account creation
- [ ] Create root cause with label only (value auto-generated)
- [ ] Create root cause with both label and value
- [ ] Create root cause fails if value already exists (case-insensitive)
- [ ] Create root cause fails if label already exists (case-insensitive)
- [ ] Delete root cause succeeds
- [ ] Delete root cause fails if doesn't exist
- [ ] Delete root cause fails if doesn't belong to account
- [ ] Update root cause succeeds
- [ ] Update root cause fails if new value conflicts
- [ ] Permissions: OWNER can manage
- [ ] Permissions: ADMIN can manage
- [ ] Permissions: Regular user cannot manage
- [ ] Default values created on account creation
- [ ] Slug generation works correctly for various labels

#### 7.2 Account Settings

- [ ] GET settings returns defaults for new account
- [ ] GET settings returns 404 if account doesn't exist
- [ ] GET settings returns existing settings if they exist
- [ ] GET settings auto-creates defaults if missing (if lazy init implemented)
- [ ] PATCH settings updates ticketStatuses correctly
- [ ] PATCH settings merges updates (doesn't replace entire object)
- [ ] PATCH settings updates formSettings correctly
- [ ] PATCH settings validates ticket status keys
- [ ] PATCH settings validates boolean values
- [ ] PATCH settings validates autoRefreshInterval range (1-60)
- [ ] PATCH settings returns 404 if account doesn't exist
- [ ] PATCH settings returns 404 if settings don't exist (unless auto-create on PATCH)
- [ ] PATCH settings creates settings if missing (if auto-create on PATCH implemented)
- [ ] Permissions: OWNER can update
- [ ] Permissions: ADMIN can update
- [ ] Permissions: Regular user cannot update (403)
- [ ] Default values created on account creation
- [ ] Partial updates work correctly (only update provided fields)
- [ ] Nested object merging works correctly
- [ ] Concurrent updates handled correctly (last write wins, or implement optimistic locking)

---

### 8. API Contract Details

**Base URL:**
- Development: `http://localhost:8080/v1`
- Production: `https://api.maintor.systems/v1` (or your production URL)

**Authentication:**
- All endpoints require Firebase ID token
- Header format: `Authorization: IDTOKEN.<firebaseIdToken>`
- Verify token and extract user/account information

**Error Response Format:**
```json
{
  "error": "Error code",
  "message": "Human-readable error message",
  "details": {
    // Additional error details (optional)
  }
}
```

**Validation Error Format (400 Bad Request):**
```json
{
  "error": "Validation failed",
  "message": "Invalid request data",
  "issues": [
    {
      "path": ["settings", "ticketStatuses", "INVALID_STATUS"],
      "message": "Invalid ticket status key. Must be one of: OPEN, IN_PROGRESS, COMPLETED, CLOSED, SKIPPED"
    },
    {
      "path": ["settings", "formSettings", "dashboard", "autoRefreshInterval"],
      "message": "autoRefreshInterval must be between 1 and 60"
    }
  ]
}
```

---

### 9. Priority

**High Priority** - This feature is needed to:
1. Synchronize settings between maintor-app and maintor-eng
2. Complete the Settings page functionality (ticket statuses are currently in localStorage)
3. Enable consistent configuration across both applications

The Settings page UI is complete and currently uses localStorage for ticket statuses, which prevents settings from being shared between the two applications.

---

### 10. Questions/Clarifications Needed

1. **Database Choice**: Which database are you using? (Firestore, PostgreSQL, etc.)
2. **Auto-Creation Strategy**: Do you prefer auto-creation on account creation, lazy initialization on GET, or explicit POST for settings?
3. **Form Settings**: Should `formSettings` be required or optional? (Recommendation: optional for now)
4. **Optimistic Locking**: Should we implement version/ETag checking to prevent concurrent update conflicts?
5. **Settings History**: Do we need to track history of settings changes (audit log)?
6. **Site-Level Settings**: Should settings be account-level only, or do we need site-level overrides in the future?
7. **Root Causes Ordering**: Should `order` field be required or optional? How should ordering work if not provided?
8. **Root Causes Value Immutability**: Should `value` be immutable after creation, or can it be updated?
9. **Soft Delete**: Should root causes be soft-deleted (marked as deleted) or hard-deleted?
10. **Migration**: Should we migrate existing accounts to have default root causes and settings?

---

## Reference

- Root causes are already implemented and working
- Frontend currently uses localStorage with key: `maintor_form_settings` for ticket statuses and form settings
- Settings are managed in:
  - `src/utils/settings.js` - Settings utility functions
  - `src/pages/SettingsPage.vue` - Settings page UI
  - `src/api/rootCauses.js` - Root causes API client (already implemented)
- See `ROOT-CAUSES-IMPLEMENTATION-NOTES.md` for frontend implementation details and API contract examples

---

## Acceptance Criteria

### Root Causes (Already Implemented)
✅ All 4 endpoints implemented and working
✅ Default root causes created for new accounts
✅ Proper error handling and validation
✅ Permissions enforced correctly
✅ Unique constraints working (case-insensitive)
✅ Slug generation working correctly
✅ API tested and documented

### Account Settings (New)
✅ GET endpoint implemented and returns settings (or creates defaults)
✅ PATCH endpoint implemented with merge strategy
✅ Default values created for new accounts
✅ Proper error handling and validation
✅ Permissions enforced correctly
✅ Partial updates work correctly (merge, don't replace)
✅ All ticket status keys validated
✅ Form settings structure validated (if implemented)
✅ API tested and documented
✅ Settings synchronized between maintor-app and maintor-eng

Once implemented, the frontend will be updated to use the API endpoints instead of localStorage, enabling settings synchronization between both applications.

