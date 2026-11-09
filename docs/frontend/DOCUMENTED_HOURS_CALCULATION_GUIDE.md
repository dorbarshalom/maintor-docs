# Frontend Guide: Documented Hours (Standard Hours) Chart Calculation

This guide details the mathematical specification, business logic, and code implementation of the **Documented Hours** (or **Standard Hours** / *שעות מתועדות / שעות תקן*) daily bar chart in the Maintor dashboard.

Implementation file: [`maintor-app/src/pages/DashboardPage.vue`](file:///Users/dorbarshalom/Dev/maintor/maintor-app/src/pages/DashboardPage.vue) (specifically `calculateStandardHoursCurrentMonth`).

---

## 1. Overview & Purpose

The Documented Hours chart visualizes the total documented labor hours per calendar day for the **current calendar month**, broken down into two stacked components:
1. **Breakdown Maintenance Hours** (*קריאות שבר*)
2. **Planned Maintenance Hours** (*אחזקה מתוכננת*)

Additionally, the chart plots a benchmark target line (**Expected Daily Hours** / *יעד יומי*) based on the account's configured technician headcount, standard daily shift hours, and target labor efficiency percentage:
$$\text{Expected Daily Hours} = \text{Technicians} \times \text{Work Hours Per Day} \times \frac{\text{Efficiency \%}}{100}$$

---

## 2. Calculation Pipeline

### A. Calendar Day Buckets

1. The calculation inspects the current date `now = new Date()`.
2. It builds day buckets $d \in [1, \dots, N]$, where $N$ is the total days in the active calendar month:
   - Bucket Start: $b_{\text{start}} = \text{Date}(\text{Year}, \text{Month}, d, 0, 0, 0, 0)$
   - Bucket End: $b_{\text{end}} = \text{Date}(\text{Year}, \text{Month}, d, 23, 59, 59, 999)$
3. Future days beyond `now.getDate()` remain unpopulated (`null`) so future days do not display zeroes.

```javascript
const totalDaysInMonth = new Date(currentYear, currentMonth + 1, 0).getDate()
const dayBuckets = []
for (let d = 1; d <= totalDaysInMonth; d++) {
  const dayStart = new Date(currentYear, currentMonth, d, 0, 0, 0, 0)
  const dayEnd = new Date(currentYear, currentMonth, d, 23, 59, 59, 999)
  dayBuckets.push({
    day: d,
    label: String(d),
    startMs: dayStart.getTime(),
    endMs: dayEnd.getTime(),
    breakdownHours: 0,
    plannedHours: 0
  })
}
```

---

### B. Breakdown Tickets Processing

For every breakdown ticket in the current site/account context:
1. **Filter Out Inactive Tickets**: Tickets with status `SKIPPED` or `CANCELLED` are omitted.
2. **Determine Ticket Start Timestamp ($tStart$)**:
   - Resolved from `ticket.breakdown.start_time` $\to$ `ticket.timeline.started` $\to$ `ticket.created` $\to$ `ticket.created_at`.
3. **Determine Ticket End Timestamp ($tEnd$)**:
   - For `COMPLETED` or `CLOSED` tickets: resolved from `ticket.timeline.ended` $\to$ `ticket.timeline.completed_at` $\to$ `ticket.completed_at` $\to$ `ticket.updated`.
   - For ongoing active tickets (`OPEN` or `IN_PROGRESS`): $tEnd = \text{now}$.
4. **Attribution Method 1: Detailed Labor Entries (`labor_entries`)**:
   If `ticket.timeline.labor_entries` (or `ticket.labor_entries`) is present:
   - For each labor entry with recorded start $[eStart]$ and end $[eEnd]$:
     - Effective duration is taken from `entry.duration_min` (or `entry.duration`) if explicit; otherwise $(eEnd - eStart)$ in minutes.
     - The entry window is intersected with the day bucket window $[b_{\text{start}}, b_{\text{end}}]$.
     - The overlap fraction is calculated:
       $$\text{fraction} = \frac{\min(eEnd, b_{\text{end}}) - \max(eStart, b_{\text{start}})}{eEnd - eStart}$$
     - Contributed hours to the day bucket:
       $$\text{hours} = \frac{\text{effectiveMinutes} \times \text{fraction}}{60}$$
5. **Attribution Method 2: Explicit Ticket Duration (No Labor Entries)**:
   If no labor entries exist, but `ticket.timeline.duration_min` or `ticket.duration_min` is recorded:
   - The total ticket span $S = \max(tEnd - tStart, 1\text{ ms})$ is determined.
   - The overlap between $[tStart, tEnd]$ and $[b_{\text{start}}, b_{\text{end}}]$ is calculated:
     $$\text{fraction} = \frac{\min(tEnd, b_{\text{end}}) - \max(tStart, b_{\text{start}})}{S}$$
     $$\text{hours} = \frac{\text{duration\_min} \times \text{fraction}}{60}$$

---

### C. Planned Maintenance Tickets Processing

Planned tickets follow the same day-overlap clipping logic, with specific entry gates:
1. **Started Check**: Only planned tickets that have actually entered execution (`IN_PROGRESS`, `COMPLETED`, `CLOSED`, or having recorded `labor_entries`) contribute documented hours.
2. **Start Timestamp ($tStart$)**: Resolved from `ticket.timeline.started` $\to$ `ticket.in_progress_at` $\to$ `ticket.timeline.in_progress_at` $\to$ `ticket.scheduled_date` $\to$ `ticket.created`.
3. **End Timestamp ($tEnd$)**: Completed/closed timestamp, or `now` if currently active.
4. **Labor Entries vs. Pre-set Estimated Duration**:
   - If `labor_entries` are logged, each entry is proportionally clipped into the matching day bucket.
   - If no labor entries exist, fallback duration is retrieved from `timeline.duration_min` $\to$ `ticket.duration_min` $\to$ `ticket.estimatedDurationMin` $\to$ `ticket.estimated_duration_min`.

---

## 3. Key Behavioral Characteristics

### Proportional Multi-Day Distribution
If a labor entry or ticket spans across calendar midnight (e.g. starting at 20:00 on Day 1 and ending at 04:00 on Day 2):
- Day 1 receives the 4 hours prior to 23:59:59.
- Day 2 receives the 4 hours after 00:00:00.
If a ticket remains open continuously across several days, each intervening full day receives up to 24.0 hours of work contribution.

### Team / Multi-Technician Aggregation
Labor hours represent **person-hours**, not asset downtime:
- If 2 technicians work together on the same ticket for 3 hours and both log their labor entries, the ticket records $3 + 3 = 6$ total documented person-hours.
- If a technician logs into multiple maintenance tickets concurrently, the hours for each ticket are calculated independently according to their respective logged entries.
