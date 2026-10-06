# Frontend Guide: Dynamic Work Hours Calculation (Auto-Start to Now)

This document describes the unified work hours calculation logic implemented in ticket forms across **maintor-app** (desktop) and **maintor-engineers** (mobile).

---

## 1. Overview & Problem Solved

In the work hours / labor entries box of ticket forms (under the start/end time inputs), each entry has a summary line displaying the calculated total work hours (`Total tracked` / `tickets.totalTracked` / `סה"כ עבודה`).

Previously, this row only rendered when both `entry.started` and `entry.ended` were present. If a ticket was ongoing without an explicit end time, or if a user had not reported labor entries manually, the field showed no calculation.

With the automated work hours attribution feature, the app now dynamically calculates and displays the elapsed work hours up until `now` whenever an end time is omitted, using deterministic auto-start fallback rules based on ticket type and status.

---

## 2. Core Calculation Rules

For any work hours / labor entry in a ticket form:

### A. End Time Determination
1. **Explicit End Time**: If `entry.ended` is set, that value is used as the end time.
2. **Ongoing / No End Time**: If `entry.ended` is missing or empty, the current timestamp (`now`) is used as the end time.

### B. Start Time Determination
1. **Explicit / Manual Start Time**:
   - If the user clicked "Started" (or "Start Tracking") or manually typed a start date/time into `entry.started`, that value is always used as the start time.
2. **Auto-Start Fallbacks** (when `entry.started` is empty):
   - **Breakdown Maintenance Tickets**:
     - Work started is taken as the **ticket created timestamp** (`ticket.created_at || ticket.created`).
     - If the ticket has not yet been saved/created (e.g. new unsaved draft), no calculation is shown.
   - **Planned Maintenance Tickets**:
     - **Status `OPEN` / `PLANNED`**: Do **not** show total hours yet (unless an explicit manual start time is provided).
     - **Status `IN_PROGRESS` (or beyond)**: Work started is taken as the timestamp when the ticket **moved to `IN_PROGRESS`** (`ticket.timeline.started`).

### C. Multi-Technician & Multiple Entries
- **Technician Multiplier**: If an entry has multiple technicians assigned (`entry.user_ids`), the elapsed time is multiplied by the number of technicians:
  $$\text{Total Minutes} = \text{round}\left(\frac{\text{EndTime} - \text{StartTime}}{60000}\right) \times \max(1, \text{numTechs})$$
- **Secondary Entries**: When an additional work hours entry block is added (e.g. for another technician), the exact same auto-start and ongoing calculation logic applies to it.
- **Negative / Invalid Intervals**: If $\text{EndTime} < \text{StartTime}$, the calculation is treated as invalid and hidden (`''`).

---

## 3. Real-Time UI Updates

To ensure the displayed duration stays accurate without requiring a page refresh while the form is open:
- A reactive `now = ref(new Date())` ticker updates every 60 seconds (`setInterval` cleared on `onUnmounted`).
- The template conditionally renders the summary line using `v-if="getLaborEntryDuration(entry)"`.

---

## 4. Component Implementations

| Repository | Component | Ticket Type | Description |
| :--- | :--- | :--- | :--- |
| `maintor-app` | `src/components/TicketForm.vue` | Breakdown | Handles breakdown ticket creation and editing on desktop. Calculates from `ticket.created_at` or manual start up to `now` / `entry.ended`. |
| `maintor-app` | `src/components/PlannedTicketForm.vue` | Planned | Handles planned maintenance ticket editing on desktop. Hides total hours in `OPEN` status; calculates from `timeline.started` once `IN_PROGRESS`. |
| `maintor-engineers` | `src/pages/Technician/TicketFormPage.vue` | Breakdown & Planned | Handles technician forms on mobile. Distinguishes between `PLANNED` and `BREAKDOWN` types, respecting `OPEN` status hiding and `ticketCreatedTime` / `timeline.started` fallbacks. |

---

## 5. Examples

| Ticket Type | Status | `entry.started` | `entry.ended` | Displayed "Total tracked" |
| :--- | :--- | :--- | :--- | :--- |
| Breakdown | `OPEN` | *(empty)* | *(empty)* | Time elapsed from **ticket creation** until **now** |
| Breakdown | `IN_PROGRESS` | `10:00` | *(empty)* (now is `11:30`) | `1h 30m` (from `10:00` to `now`) |
| Breakdown | `COMPLETED` | `10:00` | `11:15` | `1h 15m` (from `10:00` to `11:15`) |
| Planned | `OPEN` | *(empty)* | *(empty)* | *(Hidden / not shown)* |
| Planned | `OPEN` | `08:00` | *(empty)* (now is `09:00`) | `1h` (from manual start to `now`) |
| Planned | `IN_PROGRESS` | *(empty)* | *(empty)* | Time elapsed from **transition to `IN_PROGRESS`** until **now** |
