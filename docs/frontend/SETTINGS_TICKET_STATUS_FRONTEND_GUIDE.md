# Frontend Guide: Settings → Ticket Status (Breakdown vs Planned Maintenance)

This guide explains where **Settings → Ticket Status** is persisted, and how the frontend should **read** and **update** it via the API.

Important: “Ticket Status” is **split into two independent sets**:

- **Breakdown maintenance ticket statuses**
- **Planned maintenance ticket statuses**

## What “Ticket Status” means in settings

The backend stores “Ticket Status” settings as **account-level boolean toggle maps**:

- `breakdownMaintenanceTicketStatuses`
- `plannedMaintenanceTicketStatuses`

- **`true`**: that status is enabled (the UI should treat it as available/visible)
- **`false`**: that status is disabled (the UI should treat it as hidden/unavailable)

Supported keys:

- **Breakdown maintenance**: `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CLOSED`
- **Planned maintenance**: `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CLOSED`, `SKIPPED`

## Where it’s stored (database)

Settings are stored in **Firestore** in:

- **Collection**: `account_settings`
- **Document id**: the `accountId` (one settings document per account)

So the document path is:

- **`account_settings/{accountId}`**

Relevant fields inside the document:

- `accountId: string`
- `breakdownMaintenanceTicketStatuses: { OPEN?: boolean, IN_PROGRESS?: boolean, COMPLETED?: boolean, CLOSED?: boolean }`
- `plannedMaintenanceTicketStatuses: { OPEN?: boolean, IN_PROGRESS?: boolean, COMPLETED?: boolean, CLOSED?: boolean, SKIPPED?: boolean }`
- `createdAt`, `updatedAt` (server timestamps)

## Defaults + initialization behavior (important)

Settings are created automatically with defaults from `src/config/default-account-settings.js`:

- `breakdownMaintenanceTicketStatuses` defaults to all `true` (no `SKIPPED`)
- `plannedMaintenanceTicketStatuses` defaults to all `true` (includes `SKIPPED`)

Creation can happen in **two ways**:

- **During self-signup** (account bootstrap) via `src/utils/self-signup.js`
- **Lazy initialization**: if settings doc doesn’t exist, both **GET** and **PATCH** will create it with defaults first (`DEFAULT_ACCOUNT_SETTINGS`)

## API endpoints (what the frontend calls)

### Authentication header

All endpoints require an auth header in this format:

- **`Authorization: IDTOKEN.<Firebase ID token>`**

### Read settings (including ticketStatuses)

- **Method**: `GET`
- **Path**: `/v1/accounts/:accountId/settings`

Notes:

- If the settings doc is missing, the backend will create it with defaults and return it.
- The user must have **at least one active role** (`isActive: true`) for this account or the request is rejected.

Example response shape (trimmed):

```json
{
  "accountId": "abc123",
  "language": "en",
  "breakdownMaintenanceTicketStatuses": {
    "OPEN": true,
    "IN_PROGRESS": true,
    "COMPLETED": true,
    "CLOSED": true
  },
  "plannedMaintenanceTicketStatuses": {
    "OPEN": true,
    "IN_PROGRESS": true,
    "COMPLETED": true,
    "CLOSED": true,
    "SKIPPED": true
  },
  "formSettings": { },
  "id": "abc123",
  "created": "2026-01-01T12:00:00.000Z",
  "updated": "2026-01-16T09:30:00.000Z"
}
```

### Update breakdown maintenance ticket statuses

- **Method**: `PATCH`
- **Path**: `/v1/accounts/:accountId/settings`
- **Body**: `{ settings: { breakdownMaintenanceTicketStatuses: { ... } } }`

Permissions:

- Requires one of: **`OWNER`**, **`ADMIN`**, **`SITE_MANAGER`**

Validation rules / gotchas:

- `breakdownMaintenanceTicketStatuses` must include **at least one key** (you can send just the changed key).
- Values must be booleans.
- Backend does **deep-merge** updates, so unspecified keys are preserved.

Minimal update example (toggle only `CLOSED` off for breakdown tickets):

```json
{
  "settings": {
    "breakdownMaintenanceTicketStatuses": {
      "CLOSED": false
    }
  }
}
```

The response is the **full updated settings object** (same shape as GET).

### Update planned maintenance ticket statuses

- **Method**: `PATCH`
- **Path**: `/v1/accounts/:accountId/settings`
- **Body**: `{ settings: { plannedMaintenanceTicketStatuses: { ... } } }`

Minimal update example (enable `SKIPPED` for planned maintenance tickets):

```json
{
  "settings": {
    "plannedMaintenanceTicketStatuses": {
      "SKIPPED": true
    }
  }
}
```

## Recommended frontend behavior

- **Load**: call `GET /v1/accounts/:accountId/settings` once per app session (or cache it), and drive the Settings UI from:
  - `breakdownMaintenanceTicketStatuses` (breakdown ticket flows)
  - `plannedMaintenanceTicketStatuses` (planned maintenance ticket flows)
- **Save**: when the user toggles a status, send a **minimal PATCH** that includes only the changed key(s) in the appropriate map.
- **Don’t rely on local storage**: the source of truth is the backend `account_settings/{accountId}` doc.
- **Prevent “disable all” in UI**: the backend does not enforce “at least one status must be true”; it only enforces “at least one key must be provided in the PATCH payload”.

## Backward compatibility note (legacy field)

Older clients may still send or receive a legacy field: `ticketStatuses` (single map including `SKIPPED`).

- If the backend receives `ticketStatuses`, it treats it as **planned maintenance statuses**, and derives breakdown statuses by dropping `SKIPPED`.
- New clients should use the split fields: `breakdownMaintenanceTicketStatuses` + `plannedMaintenanceTicketStatuses`.

## Code pointers (backend implementation)

- Routes: `index.js` (Cloud Functions router)
  - `GET /v1/accounts/:accountId/settings`
  - `PATCH /v1/accounts/:accountId/settings`
- Handlers:
  - `src/handlers/getAccountSettings.js`
  - `src/handlers/updateAccountSettings.js`
- Storage / merge logic:
  - `src/db/account-settings.js` (Firestore collection `account_settings`, deep merge update)
- Defaults:
  - `src/config/default-account-settings.js`

