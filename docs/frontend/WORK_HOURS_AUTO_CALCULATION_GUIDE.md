# Frontend Guide: Dynamic Work Hours Calculation (Auto-Start to Now)

This document describes the unified work hours calculation logic implemented in ticket forms across **maintor-app** (desktop) and **maintor-engineers** (mobile).

---

## 1. Overview & Problem Solved

In the work hours / labor entries box of ticket forms (under the start/end time inputs), each entry has a summary line displaying the calculated total work hours (`Total tracked` / `tickets.totalTracked` / `סה"כ עבודה`).

Previously, this row only rendered when both `entry.started` and `entry.ended` were present. If a ticket was ongoing without an explicit end time, or if a user had not reported labor entries manually, the field showed no calculation.

With the automated work hours attribution feature:
1. The app dynamically calculates and displays the elapsed work hours up until `now` whenever an end time is omitted.
2. When no user reporting is provided, the app also automatically populates the start date and time in the labor entry (`entry.started`) based on the ticket type and status.

---

## 2. Core Calculation & Pre-Fill Rules

For any work hours / labor entry in a ticket form:

### A. End Time Determination
1. **Explicit End Time**: If `entry.ended` is set, that value is used as the end time.
2. **Ongoing / No End Time**: If `entry.ended` is missing or empty, the current timestamp (`now`) is used as the end time.

### B. Start Time Determination & Pre-Fill
1. **Explicit / Manual Start Time**:
   - If the user clicked "Started" (or "Start Tracking") or manually typed a start date/time into `entry.started`, that value is preserved.
2. **Auto-Start Pre-Fill** (when `entry.started` is not reported):
   - **Breakdown Maintenance Tickets**:
     - `entry.started` is populated with the **ticket created timestamp** (`ticket.created_at || ticket.created`).
     - If the ticket has not yet been saved/created (e.g. new unsaved draft), `entry.started` remains empty and no calculation is shown.
   - **Planned Maintenance Tickets**:
     - **Status `OPEN` / `PLANNED`**: `entry.started` remains empty and total work hours are **not** shown yet.
     - **Status `IN_PROGRESS` (or beyond)**: `entry.started` is populated with the timestamp when the ticket **moved from OPEN to `IN_PROGRESS`** (`ticket.timeline.started`).

### C. Multi-Technician & Multiple Entries
- **Technician Multiplier**: If an entry has multiple technicians assigned (`entry.user_ids`), the elapsed time is multiplied by the number of technicians:
  ```js
  Total Minutes = Math.round((EndTime - StartTime) / 60000) * Math.max(1, numTechs)
  ```
- **Secondary Entries**: When an additional work hours entry block is added (e.g. for another technician), the exact same auto-start pre-fill and ongoing calculation logic applies to it.
- **Negative / Invalid Intervals**: If `EndTime < StartTime`, the calculation is treated as invalid and hidden (`''`).

---

## 3. Real-Time UI Updates

To ensure the displayed duration stays accurate without requiring a page refresh while the form is open:
- A reactive `now = ref(new Date())` ticker updates every 60 seconds (`setInterval` cleared on `onUnmounted`).
- The template conditionally renders the summary line using `v-if="getLaborEntryDuration(entry)"`.

---

## 4. Component Implementations

| Repository | Component | Ticket Type | Description |
| :--- | :--- | :--- | :--- |
| `maintor-app` | `src/components/TicketForm.vue` | Breakdown | Handles breakdown ticket creation and editing on desktop. Pre-fills `entry.started` with `ticket.created_at` when empty; calculates elapsed time up to `now` / `entry.ended`. |
| `maintor-app` | `src/components/PlannedTicketForm.vue` | Planned | Handles planned maintenance ticket editing on desktop. Keeps `entry.started` empty in `OPEN`; pre-fills with `timeline.started` once moved to `IN_PROGRESS` and updates live up to `now`. |
| `maintor-engineers` | `src/pages/Technician/TicketFormPage.vue` | Breakdown & Planned | Handles technician forms on mobile. Distinguishes between `PLANNED` and `BREAKDOWN` types, pre-filling `entry.started` with `ticketCreatedTime` (breakdown) or `timeline.started` (planned in `IN_PROGRESS`). |

---

## 5. Examples

| Ticket Type | Status | User Reporting | `entry.started` (Pre-filled) | `entry.ended` | Displayed "Total tracked" |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Breakdown | `OPEN` | *(none)* | `Ticket Created Time` (e.g. `10:00`) | *(empty)* (now is `12:00`) | `2h` (from creation to `now`) |
| Breakdown | `IN_PROGRESS` | Manual start `11:00` | `11:00` | *(empty)* (now is `12:30`) | `1h 30m` (from `11:00` to `now`) |
| Breakdown | `COMPLETED` | Manual start & end | `10:00` | `11:15` | `1h 15m` (from `10:00` to `11:15`) |
| Planned | `OPEN` | *(none)* | *(empty)* | *(empty)* | *(Hidden / not shown)* |
| Planned | `OPEN` | Manual start `08:00` | `08:00` | *(empty)* (now is `09:00`) | `1h` (from manual start to `now`) |
| Planned | `IN_PROGRESS` | *(none)* | `Time moved to IN_PROGRESS` (e.g. `14:00`) | *(empty)* (now is `15:30`) | `1h 30m` (from transition to `now`) |
