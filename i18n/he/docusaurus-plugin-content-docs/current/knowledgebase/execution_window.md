---
sidebar_label: חלון זמן לביצוע (חלון ביצוע)
title: חלון זמן לביצוע (חלון ביצוע)
---
# חלון זמן לביצוע (חלון ביצוע)

The **Time-to-Execute Window** (or Execution Window) defines the period during which a planned maintenance ticket is considered valid and should be completed. This window ensures that maintenance tasks are performed within a reasonable timeframe relative to their next scheduled occurrence.

---

## Overview

Every **Planned Maintenance Ticket** has a formal execution window. If a ticket is not completed within this window, it may be marked as expired or skipped (depending on system settings), as it is no longer relevant given the next scheduled maintenance cycle.

The window starts at the ticket's `scheduled_date` and ends at a calculated `execution_window_end_date`.

---

## Detailed Logic

The system calculates the window end date using the following priority and rules:

### 1. Manual Override (`execution_window_days`)
If a specific number of days is defined in the maintenance template or ticket, the system uses it as a direct override.
*   **Formula**: `Scheduled Date + execution_window_days`

### 2. Long-Interval Tasks (12+ Weeks)
For maintenance tasks that occur infrequently (defined as having a recurrence interval of **12 weeks or more** for any pattern), the system applies a fixed safety window.
*   **Window Duration**: **30 Days** fixed.
*   **Formula**: `Scheduled Date + 30 Days`

### 3. Cycle-Based Window (Standard)
For standard frequent tasks, the window is designed to close exactly when the **next occurrence** of the maintenance is scheduled to begin. This prevents overlapping tickets for the same task.
*   **Window Duration**: Variable, based on the next scheduled date.
*   **Formula**: `Scheduled Date + (Time until Next Occurrence)`

#### How "Next Occurrence" is Found:
The system looks at all recurrence patterns associated with the template and calculates the earliest date that appears after the current ticket's scheduled date.

---

## Technical Implementation

### Core Components
*   `src/utils/execution-window.js`: Contains the primary logic for calculating the window end date.
*   `calculateExecutionWindowEnd()`: The main exported function that determines which logic to apply based on the template configuration.
*   `calculateNextOccurrenceDate()`: A helper function that searches forward (up to 1 year) to find the next maintenance event.

### Key Logic Fragment
```javascript
if (hasLongInterval) {
  // Intervals >= 12 weeks get a fixed 30-day window
  return windowEnd + 30 days;
} else {
  // Standard tasks end when the next one is scheduled to start
  return calculateNextOccurrenceDate(scheduledDate, recurrencePatterns);
}
```

---

## Examples

### Example 1: Weekly Maintenance
*   **Schedule**: Every Monday at 08:00 AM.
*   **Current Ticket**: Monday, Jan 1st.
*   **Next Occurrence**: Monday, Jan 8th.
    *   *Execution Window*: Jan 1st to Jan 8th (7 days).

### Example 2: infrequent Maintenance (Annual/Quarterly)
*   **Schedule**: Every 26 weeks (6 months).
*   **Current Ticket**: Jan 1st.
*   **Rule Applied**: Long-Interval (26 >= 12).
    *   *Execution Window*: Jan 1st to Jan 31st (30 days fixed).

### Example 3: Multiple Patterns
*   **Schedule A**: Every 4 weeks.
*   **Schedule B**: Every 12 weeks.
*   **Current Ticket**: (From Schedule A) Jan 1st.
*   **Next Occurrence**: Schedule B might occur on Jan 15th.
    *   *Execution Window*: Jan 1st to Jan 15th (ends at the EARLIEST next occurrence).
