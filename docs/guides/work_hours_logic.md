# Work Hours & Labor Tracking Logic (Planned vs. Breakdown)

This document explains the business and operational logic for how technician work hours are calculated and reported across Maintor.

---

## 1. Core Principle

Work hours represent the human labor spent servicing assets. Because **Planned Maintenance** and **Breakdown Maintenance** serve fundamentally different operational purposes, their work hours are handled with distinct business logic:

* **Planned Maintenance**: Strictly measures **actual recorded technician labor**. If no hours are entered, work hours remain **zero**. There is no automatic estimation or time-window fallback.
* **Breakdown Maintenance**: Represents **reactive incident resolution**. If technicians do not log individual work entries, the system aligns labor with the actual incident duration (from when the breakdown occurred until it was resolved).

---

## 2. Planned Maintenance Logic

Planned maintenance tasks are generated on schedules (weekly, monthly, quarterly) and have broad execution windows (often 7 to 30 days) during which the work can be completed.

### Strict Actuals Only
1. **No Window Fallback**: The duration between when a planned task opens and when it is marked completed is **not** used as work duration. A task open for 3 weeks does not mean 3 weeks of continuous work.
2. **Zero Default**: If a technician completes a planned ticket without logging time, the ticket records **0 work hours**.
3. **No Estimated Duration Fallbacks**: Template estimates (such as an estimated 45-minute inspection) are planning guidelines only and are **never** substituted as actual logged labor.
4. **No Arbitrary Padding**: The system does not apply default padding (such as automatic 30-minute minimums).

### How Planned Work Hours Are Recorded
Planned work hours are logged through only two valid operational workflows:
* **Live Timer ("Start > Completed")**: The technician taps **Start** in the mobile app when beginning work and taps **Completed** (or **Stop**) when finished. The system logs the active elapsed time.
* **Manual Entry**: The technician or manager manually enters labor records with specific dates, times, or durations.

---

## 3. Breakdown Maintenance Logic

Breakdown maintenance tasks represent urgent, unscheduled machine failures and stoppages.

### Incident Window Alignment
* When an asset breaks down, the primary concern is the total downtime and repair span.
* If technicians do not itemize separate labor entries, the platform defaults the work duration to the elapsed time between the breakdown event and the final resolution.
* This ensures that mean time to repair (MTTR) and operational labor reflect the actual stoppage span without requiring extra administrative overhead during an emergency.
* If technicians do enter explicit labor entries, those explicit entries take priority.

---

## 4. Operational Comparison

| Feature | Planned Maintenance | Breakdown Maintenance |
| :--- | :--- | :--- |
| **Primary Goal** | Routine asset upkeep | Rapid failure recovery & uptime restoration |
| **Execution Window** | Broad scheduled window (days to weeks) | Immediate reactive event |
| **Unlogged Work Hours** | Strictly **0 hours** | Defaults to breakdown event duration |
| **Live Mobile Timer** | Supported (**Start > Completed**) | Supported (**Start > Completed**) |
| **Manual Labor Entries** | Supported | Supported |
| **Template Estimate Fallback** | **None** (remains 0) | Not applicable |
| **Arbitrary Minimum Padding** | **None** (remains 0) | Not applicable |

---

## 5. Reporting & Analytics Consistency

Across all dashboards, monthly performance reports, and exports:
* **Planned Work Hours KPI**: Aggregates only genuine labor logged by technicians. Empty entries do not inflate maintenance costs or workload statistics.
* **Breakdown Work Hours KPI**: Reflects repair labor, ensuring downtime recovery is fully visible even if technicians did not log separate itemized entries.
* **Workload Comparisons**: The comparison of Planned vs. Breakdown maintenance accurately reflects real maintenance hours rather than calendar window differences.
