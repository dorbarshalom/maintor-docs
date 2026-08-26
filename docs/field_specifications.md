# Asset and Ticket Field Specifications

This document outlines the schema field requirements for Assets, Breakdown Tickets, Planned Tickets, and Planned Maintenance Templates.

---

## 1. Asset Fields

Assets represent machines, equipment, or components in the system.

### Required Fields (Request Input)
*   **`name`** (String): The name/label of the asset (minimum 1 character).

### Non-Required Fields
#### Optional Input Fields (User Editable)
*   **`type`** (String | Object): The asset type category. Can be a string slug or an object containing a `slug` property.
*   **`description`** (String): A detailed description of the asset.
*   **`image`** (String/URI): A URL pointing to an image of the asset.
*   **`visualId`** (or **`visual_id`**) (String): An optional visual identifier for search/display (e.g. `'LN3-PMP-004'`).
*   **`barcode`** (String): Barcode identifier for scanning.
*   **`qrCode`** (String): QR code identifier for scanning.
*   **`media`** (or **`mediaFiles`**) (Array of objects): List of media items/photos associated with the asset.
*   **`attachments`** (or **`attachmentFiles`**) (Array of objects): List of documentation attachments associated with the asset.
*   **`customFields`** (Object): Key-value pairs for tenant-defined custom fields.

#### System & Context Fields (Automatic / Resolved)
*   **`id`** (String): Unique document ID.
*   **`accountId`** (or **`account_id`**) (String): Account/Company reference context.
*   **`siteId`** (or **`site_id`**) (String): Site/Plant reference context.
*   **`nodeId`** (or **`node_id`** / **`structureNodeId`**) (String): Organizational chart node reference context.
*   **`created`** (or **`createdAt`**) (String / ISO Timestamp): Creation timestamp.
*   **`updated`** (or **`updatedAt`**) (String / ISO Timestamp): Last update timestamp.

---

## 2. Breakdown Ticket Fields

Breakdown tickets track unscheduled/emergency maintenance events on assets.

### Required Fields (Request Input)
*   **`type`** (String): Must be exactly `'BREAKDOWN'`.
*   **`status`** (String): The current status of the ticket (`'OPEN'`, `'IN_PROGRESS'`, `'COMPLETED'`, `'CLOSED'`, `'SKIPPED'`).
    *   *Note: If `status` is set to `'COMPLETED'`, then `breakdown.root_cause` becomes a **required** field.*

### Non-Required Fields
#### Optional Input Fields (User Editable)
*   **`title`** (String): The title of the ticket (optional for breakdown tickets).
*   **`description`** (String): Description of the breakdown/problem.
*   **`priority`** (Integer): Numerical priority (1 to 5).
*   **`asset_id`** (String): ID of the asset associated with the breakdown (strongly recommended, though structurally marked optional in base schema).
*   **`site_id`** (String): ID of the site where the breakdown occurred.
*   **`operator_notes`** (String): Notes provided by operators at the time of the event.
*   **`timeline`** (Object):
    *   `started` (String / ISO DateTime)
    *   `ended` (String / ISO DateTime)
    *   `duration_min` (Number)
    *   `manual_downtime_min` (Number)
    *   `is_downtime` (Boolean)
    *   `downtime_start` (String / ISO DateTime)
    *   **`labor_entries`** (Array of objects containing `user_id`, `started`, `ended`, `duration_min`)
        *   **Work Hours Auto-Creation & Calculation System Logic on Completion (`COMPLETED` status):**
            1. **Explicit Manual Labor Override (Highest Priority):** If explicit manual labor entries (with custom start and end times) are defined on the ticket, the system saves **only** those manual entries. No default work hours are calculated or assigned to any other user.
            2. **Selected Technician Override (No explicit times):** If a specific technician (or technicians) is selected in the labor section without start/end times, the system automatically calculates default work hours (`Breakdown Start Time` → `Completion Time`) and assigns them **only to the selected technician(s)**. No work hours are assigned to the completer or ticket assignee unless explicitly selected.
            3. **Default Assignment (No labor entries touched):** If no labor entries were added or selected, the system automatically calculates default work hours (`Breakdown Start Time` → `Completion Time`) and assigns them **only to the logged-in user completing the ticket**. No work hours are assigned to the ticket assignee (unless the completer is the assignee).
*   **`breakdown`** (Object):
    *   `root_cause` (String): (Required on `COMPLETED` status)
    *   `solution_description` (String)
    *   `start_time` (String / ISO DateTime)
*   **`tasks`** (Array of objects): Specific tasks required to resolve the ticket. Each task includes:
    *   `description` (String)
    *   `status` (String: `'PENDING'`, `'DONE'`, `'SKIPPED'`, `'FAILED'`)
    *   `photos` (Array of objects containing `url`, `name`, `status`)
*   **`notes`** (Array of objects): Internal audit trail comments/notes. Contains `at` (DateTime), `by` (User ID), and `note` (String).
*   **`owner_user_id`** (String): Owner/supervisor user ID.
*   **`reported_by_user_id`** (String): Reporting operator/user ID.
*   **`assignees`** (Array of objects): Assigned technician list. Contains objects with `user_id`.
*   **`photos`** (Array of objects): Verification photos. Contains `{ url, caption }`.
*   **`attachments`** (Array of objects): Diagnostic documents. Contains `{ doc_id, title, kind }`.
*   **`files`** (Array of objects): Files associated with the ticket. Contains `{ name, url }`.
*   **`estimatedDurationMin`** (Number): Estimated time to repair in minutes.
*   **`customFields`** (Object): Key-value pairs for custom fields.

#### System & Context Fields (Automatic / Resolved)
*   **`id`** (String): Unique document ID.
*   **`accountId`** (or **`account_id`**) (String): Account/Company reference context.
*   **`created_at`** (String / ISO Timestamp): Creation timestamp.
*   **`updated_at`** (String / ISO Timestamp): Last update timestamp.

---

## 3. Planned Ticket Fields

Planned tickets represent active instances of scheduled maintenance tasks.

### Required Fields (Request Input)
*   **`type`** (String): Must be exactly `'PLANNED'`.
*   **`title`** (String): The title of the planned maintenance ticket.
*   **`scheduled_date`** (String): The scheduled date for the maintenance event.
*   **`status`** (String): The current status of the ticket (`'PLANNED'`, `'OPEN'`, `'IN_PROGRESS'`, `'COMPLETED'`, `'CLOSED'`, `'SKIPPED'`).

### Non-Required Fields
#### Optional Input Fields (User Editable)
*   **`description`** (String): Specific procedures or details.
*   **`priority`** (Integer): Numerical priority (1 to 5).
*   **`asset_id`** (String): ID of the asset to undergo maintenance.
*   **`site_id`** (String): ID of the site where the asset is located.
*   **`operator_notes`** (String): Notes provided by operators/technicians.
*   **`timeline`** (Object):
    *   `started` (String / ISO DateTime)
    *   `ended` (String / ISO DateTime)
    *   `duration_min` (Number)
    *   `manual_downtime_min` (Number)
    *   `is_downtime` (Boolean)
    *   `downtime_start` (String / ISO DateTime)
    *   `labor_entries` (Array of objects containing `user_id`, `started`, `ended`, `duration_min`)
*   **`tasks`** (Array of objects): Task checklists. Each task includes:
    *   `description` (String)
    *   `status` (String: `'PENDING'`, `'DONE'`, `'SKIPPED'`, `'FAILED'`)
    *   `photos` (Array of objects containing `url`, `name`, `status`)
*   **`notes`** (Array of objects): Audit comments. Contains `at` (DateTime), `by` (User ID), and `note` (String).
*   **`owner_user_id`** (String): Owner/supervisor user ID.
*   **`reported_by_user_id`** (String): Supervisor/scheduler user ID.
*   **`assignees`** (Array of objects): Assigned technician list. Contains objects with `user_id`.
*   **`photos`** (Array of objects): Completed work validation photos. Contains `{ url, caption }`.
*   **`attachments`** (Array of objects): Completed checklist/report links. Contains `{ doc_id, title, kind }`.
*   **`files`** (Array of objects): General files. Contains `{ name, url }`.
*   **`plannedMaintenanceTemplateId`** (String): ID referencing the originating Planned Maintenance template.
*   **`estimatedDurationMin`** (Number): Target execution time in minutes.
*   **`customFields`** (Object): Key-value pairs for custom fields.

#### System & Context Fields (Automatic / Resolved)
*   **`id`** (String): Unique document ID.
*   **`accountId`** (or **`account_id`**) (String): Account/Company reference context.
*   **`created_at`** (String / ISO Timestamp): Creation timestamp.
*   **`updated_at`** (String / ISO Timestamp): Last update timestamp.

---

## 4. Planned Maintenance Template Fields

Planned Maintenance templates define recurring schedules and standard checklist templates for generating Planned Tickets.

### Required Fields (Request Input)
*   **`title`** (String): The title/name of the PM routine.
*   **`assetIds`** (Array of Strings): The list of asset IDs covered by this PM routine.
*   **`generateTicketPerAsset`** (Boolean): Whether to generate separate tickets for each asset (true) or a single ticket containing all assets (false).
*   **`recurrence`** (Array of Recurrence Objects): Define how often the task repeats. Each recurrence object requires:
    *   `frequency` (String): Must be `'WEEKLY'`.
    *   `dayOfWeek` (Array of Integers): Weekday index array (`0` = Sunday, `1` = Monday, ..., `6` = Saturday).
    *   `interval` (Integer): Interval frequency in weeks (e.g. `1` for every week, `2` for every 2 weeks).
    *   `time` (String): Trigger time in 24-hour format (`HH:mm`, e.g., `'08:00'`).
*   **`ticketTemplate`** (Object): Template for generated ticket instances. Requires:
    *   `title` (String): Title of generated tickets.
    *   `tasks` (Array of objects): Checklist items. Each task template requires:
        *   `description` (String): Task checklist description.
    *   `ownerUserId` (String): Default owner user ID.
    *   *Optional template fields:* `priority` (Integer, 1-5), `estimatedDurationMin` (Integer), `assignees` (Array of assignee objects containing `user_id`).
*   **`isActive`** (Boolean): Whether the recurrence pattern is active and scheduled for ticket generation.

### Non-Required Fields
#### Optional Input Fields (User Editable)
*   **`executionWindowDays`** (Integer): Custom duration in days to complete the task before it is overdue.
*   **`firstInstanceDate`** (String / ISO DateTime): Initial start date to schedule future occurrences or backfill.
*   **`isDemo`** (Boolean): Mark template as demo/tutorial content.

#### System & Context Fields (Automatic / Resolved)
*   **`id`** (String): Unique template ID.
*   **`accountId`** (String): Associated account ID (inferred from path parameter).
*   **`siteId`** (String): Associated site ID (inferred from path parameter).
*   **`createdAt`** (String / ISO Timestamp): Creation timestamp.
*   **`createdByUserId`** (String): Creator user ID.
