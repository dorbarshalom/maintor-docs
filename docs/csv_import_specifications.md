# CSV Import Specifications for Historical Tickets (Current App Schema)

This document outlines the required and optional fields for importing historical tickets into the current application via CSV files.

---

## 1. Breakdown Tickets History CSV
These represent historical unscheduled or emergency maintenance events mapped to the current active database schema.

| CSV Column / Field | Type | Required? | Description / Requirements |
| :--- | :--- | :--- | :--- |
| **`type`** | String | **Yes** | Must be exactly `BREAKDOWN`. |
| **`status`** | String | **Yes** | Must be one of: `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CLOSED`, `SKIPPED`. |
| **`breakdown.root_cause`** | String | **Conditional** | **Required** if `status` is `COMPLETED`. |
| **`site_id`** | String | No | *Highly Recommended.* The target Site ID where the breakdown occurred. |
| **`asset_id`** | String | No | *Highly Recommended.* The target Asset ID associated with the breakdown. |
| **`title`** | String | No | Optional description title (e.g. *"Machine X Breakdown"*). |
| **`breakdown.start_time`** | String | No | Optional. When the breakdown occurred (ISO 8601: `YYYY-MM-DDTHH:mm:ssZ`). Defaults to ticket creation time if missing. |
| **`timeline.started`** | String | No | Optional. When work on the ticket started (ISO 8601). |
| **`timeline.ended`** | String | No | Optional. When work on the ticket ended (ISO 8601). |
| **`breakdown.solution_description`** | String | No | Optional. Description of how the issue was resolved. |

> [!IMPORTANT]
> Breakdown tickets **must not** contain a `scheduled_date` field.

---

## 2. Planned Ticket Instances History CSV
These represent historical completed or scheduled occurrences of planned maintenance tasks (the actual ticket instances, not the recurring schedule templates).

| CSV Column / Field | Type | Required? | Description / Requirements |
| :--- | :--- | :--- | :--- |
| **`type`** | String | **Yes** | Must be exactly `PLANNED`. |
| **`status`** | String | **Yes** | Must be one of: `PLANNED`, `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CLOSED`, `SKIPPED`. |
| **`title`** | String | **Yes** | The name/title of the planned maintenance routine instance (e.g., *"Monthly Filter Replacement"*). |
| **`scheduled_date`** | String | **Yes** | When the planned maintenance was scheduled to happen (ISO 8601: `YYYY-MM-DDTHH:mm:ssZ`). |
| **`site_id`** | String | No | *Highly Recommended.* The target Site ID. |
| **`asset_id`** | String | No | *Highly Recommended.* The target Asset ID. |
| **`timeline.started`** | String | No | Optional. When work started (ISO 8601). |
| **`timeline.ended`** | String | No | Optional. When work ended (ISO 8601). |
| **`timeline.is_downtime`** | Boolean | No | Optional. `true` or `false` indicating if the planned maintenance caused machine downtime. |

---

## 3. General Validation & Formatting Rules

### Date-Time Fields
All timestamps (such as `scheduled_date`, `breakdown.start_time`, `timeline.started`, `timeline.ended`) must follow the ISO 8601 format:
`YYYY-MM-DDTHH:mm:ssZ` (e.g. `2026-06-29T18:00:00Z`).

### Priority
Optional integer value from `1` (lowest priority) to `5` (highest priority).

### Assignees
Optional. If you wish to import assignees, they should map to valid User IDs within the active workspace.
