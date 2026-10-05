# Work Hours & Labor Tracking Logic

This guide explains how technician work hours are calculated, attributed, and reported across all ticket types in Maintor.

---

## 1. Overview & Core Principle

Work hours represent the human labor spent servicing machines, equipment, and facilities. 

To maintain clean and reliable maintenance analytics without burdening technicians on the plant floor with unnecessary administrative paperwork, Maintor uses a **unified work hours calculation system** for both **Breakdown Maintenance** and **Planned Maintenance**:

* **Accuracy First**: Whenever technicians track live time or submit manual labor entries, those exact hours are preserved.
* **Smart Fallback**: If a ticket is completed without manual labor records, Maintor automatically calculates work hours based on the ticket's active work span—preventing jobs from registering misleading "0 hours" in reporting.
* **No Artificial Padding**: Estimates and arbitrary duration minimums are never used as actual labor. Work hours always reflect real operational elapsed time or explicitly recorded entries.

---

## 2. Priority Hierarchy: How Work Hours Are Recorded

When a ticket is marked **Completed**, Maintor calculates work hours using the following order of priority:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Explicit Manual Entries & Mobile Timers (Highest Priority)│
│    Saved as recorded; no auto-calculation applied.          │
└──────────────────────────────┬──────────────────────────────┘
                               │ (If no labor entries exist)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Automatic Fallback Calculation                           │
│    Elapsed time from Ticket Work Start → Completion Time    │
└─────────────────────────────────────────────────────────────┘
```

### Priority 1: Explicit Manual Entries & Mobile Timers
If technicians use the mobile app timer (**Start** > **Completed**) or manually enter labor records (with specific start and end times or durations), the platform saves **only** those records. No automatic fallback calculations are triggered.

### Priority 2: Automatic Fallback Calculation
If a ticket is completed with no manual labor entries, Maintor automatically calculates the duration from **Ticket Work Start Time** to **Completion Time**.

---

## 3. How Ticket Start Time Is Determined

Because Breakdown tickets and Planned tickets follow different operational workflows, the start reference time is determined as follows:

### Breakdown Tickets
* **Start Reference**: The breakdown occurrence time (or the ticket creation timestamp).
* **Rationale**: Reflects the real emergency response and recovery window from incident inception to full resolution.

### Planned Maintenance Tickets
Planned tickets have broad scheduled execution windows (e.g., 7 to 30 days) to allow flexible scheduling. To ensure work hours reflect genuine maintenance effort rather than the entire scheduling window, start time is determined by:

1. **Standard Workflow (Open → In Progress → Completed)**:
   * **Start Reference**: The timestamp when the ticket transitioned to **In Progress** (for example, when a technician tapped "Start" on mobile or saved intermediate updates on the ticket).
2. **Direct Completion (Open → Completed)**:
   * If a technician completed the task in a single step directly from **Open** without first setting it to In Progress, Maintor uses the **execution window start time** (or scheduled date) as the start reference.
   * This ensures the completed ticket receives valid work hours and prevents a 0-hour state in workload reports.

---

## 4. Technician Attribution: Who Receives the Work Hours?

When automatic work hours are calculated upon completion, Maintor attributes the hours according to the technician selection:

1. **Technicians Selected in Work Hours Box**:
   * If one or more technicians are chosen in the ticket's Work Hours / Labor section, the calculated duration is credited to **each selected technician**.
   * *Example*: If Technicians A and B are selected and the job took 1 hour, each receives 1 hour of labor (reflecting concurrent teamwork).
2. **Default Fallback (No Technicians Selected)**:
   * If the Work Hours section is left unassigned, the hours are automatically credited to the **user who marked the ticket Completed**.
   * Hours are not assigned to passive ticket assignees unless they performed the completion or were selected in the work hours box.

---

## 5. Estimated Duration vs. Actual Work Hours

Planned maintenance templates and individual planned tickets include an optional **Estimated duration** field (Hours and Minutes):

* **Planning & Scheduling Only**: The estimated duration helps maintenance managers forecast upcoming workloads and schedule technician shifts.
* **Default Value**: This field has no default value (remains empty / null unless configured).
* **Never Substituted for Actuals**: Estimated duration is **never** used as a fallback for actual work hours. If a ticket was estimated at 2 hours but resolved in 45 minutes, reports reflect the 45 minutes of actual elapsed time (or recorded labor), keeping analytics grounded in reality.

---

## 6. Summary Comparison

| Operational Aspect | Breakdown Maintenance | Planned Maintenance |
| :--- | :--- | :--- |
| **Primary Goal** | Incident response & asset uptime recovery | Routine preventive maintenance & inspection |
| **Manual Labor & Mobile Timers** | Fully supported (Highest priority) | Fully supported (Highest priority) |
| **Auto-Calculation Trigger** | When completed without manual labor | When completed without manual labor |
| **Start Time Reference** | Incident breakdown start time | `In Progress` transition timestamp |
| **Direct Completion Fallback** | Ticket creation time | Execution window start / scheduled date |
| **End Time Reference** | Ticket completion time | Ticket completion time |
| **Technician Attribution** | Selected technician(s) &rarr; Fallback: Completing user | Selected technician(s) &rarr; Fallback: Completing user |
| **Estimated Duration Role** | N/A | Informational planning & scheduling only |

---

## 7. Reports & Analytics Impact

By unifying the work hours fallback across all ticket types:

* **No Disappearing Labor**: Planned maintenance tasks completed by technicians are consistently counted in technician workload and site maintenance hours, eliminating 0-hour discrepancies.
* **Clean Team Metrics**: Team workload charts, user activity logs, and monthly management reports reflect both planned and breakdown hours accurately.
* **Fair Attribution**: Credit for completed tasks is reliably assigned to the personnel who performed the work or completed the ticket.
