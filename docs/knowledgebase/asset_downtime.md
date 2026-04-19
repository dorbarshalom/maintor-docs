# Asset Downtime Calculation

Asset downtime is a critical metric in Maintor that measures the total time an asset is unavailable for production due to maintenance issues. This document explains how the system calculates, stores, and prioritizes downtime data.

---

## Overview

Downtime is automatically tracked via **Breakdown Tickets** but can also be explicitly tracked on **Planned Maintenance Tickets**. By default, breakdown tickets assume the machine is down, while planned maintenance tickets do not accumulate downtime unless explicitly marked with the `is_downtime` toggle.

The system uses three main fields to manage downtime:
1. **Default Downtime**: Automatically calculated based on timestamps.
2. **Manual Downtime**: User-provided overrides.
3. **Effective Downtime**: The final value used for dashboards and reports.

---

## Detailed Logic

### 1. Default Downtime (`default_downtime_min`)
The default downtime is the raw duration of the ticket in minutes.

*   **In-Progress Tickets**: If a ticket is open, the downtime is calculated dynamically from the assumed start time until **now**. 
*   **Completed Tickets**: Once a ticket has an `ended` timestamp, the downtime is "frozen" as the difference between the `ended` time and the start time.

**Determining the Start Time:**
*   **Breakdown Tickets**: Uses `breakdown.start_time` if available; otherwise falls back to the ticket's creation time.
*   **Planned Maintainance**: Only calculates if `timeline.is_downtime = true`. Uses `timeline.downtime_start` if available, falls back to `timeline.started`, and finally the ticket creation time.

**Formula:**
```
Default Downtime = (End Time OR Now) - Assumed Start Time
```

### 2. Manual Downtime (`manual_downtime_min`)
Users have the option to manually override the calculated downtime. This is useful when:
*   An asset was down before the ticket was officially opened.
*   The asset returned to service before/after the ticket was technically closed in the app.
*   Internal administrative delays shouldn't count towards machine downtime.

### 3. Effective Downtime (`effective_downtime_min`)
This is the "source of truth" for all reporting. The system determines this value using a specific priority order:

1.  **Manual Overrides**: If `manual_downtime_min` is provided, it is always used as the effective downtime.
2.  **Calculated Default**: If no manual value exists, `default_downtime_min` is used.
3.  **Legacy Data**: For older tickets, the system may fall back to a legacy `downtime_min` field.

---

## Technical Implementation

### Data Persistence
To ensure performance and data integrity, the system handles downtime differently depending on the ticket status:

*   **Open Tickets**: The backend calculates `effective_downtime_min` on-the-fly when reading the ticket. It is **not** stored in the database to avoid stale data (unless a manual value is set).
*   **Closed Tickets**: When a ticket is updated to include an `ended` timestamp, the system calculates the final `default_downtime_min` and `effective_downtime_min` and **persists** them to the database. This "freezes" the data for historical reporting.

### Relevant Code Files
*   `src/db/ticket-mapper.js`: Contains the mapping logic and dynamic calculation for API responses.
*   `src/db/tickets.js`: Implements the persistence logic during ticket creation and updates.

---

## Examples

### Example 1: Standard Breakdown
1.  **Ticket Created**: Jan 1st, 10:00 AM.
2.  **Current Time**: Jan 1st, 10:30 AM.
    *   *Default Downtime*: 30 minutes.
    *   *Effective Downtime*: 30 minutes.
3.  **Ticket Ended**: Jan 1st, 11:00 AM.
    *   *Stored Default Downtime*: 60 minutes.
    *   *Stored Effective Downtime*: 60 minutes.

### Example 2: Manual Override
1.  **Ticket Created**: Jan 1st, 10:00 AM.
2.  **Ticket Ended**: Jan 1st, 11:00 AM (Calculated duration: 60 mins).
3.  **User Input**: User sets Manual Downtime to **45 minutes** because the machine was partially running.
    *   *Default Downtime*: 60 minutes (kept for reference).
    *   *Manual Downtime*: 45 minutes.
    *   *Effective Downtime*: **45 minutes**.

### Example 3: Pre-existing Downtime (Using Breakdown Start Time)
1.  **Machine Fails**: 08:00 AM.
2.  **Ticket Created**: 09:00 AM (Breakdown Start Time is manually updated to 08:00 AM).
3.  **Ticket Ended**: 10:00 AM.
    *   *Default Downtime*: 120 minutes (10:00 AM - 08:00 AM).
    *   *Effective Downtime*: **120 minutes**.

*(Note: Users can still use `manual_downtime_min` to override things if they prefer that method, but setting `breakdown.start_time` natively adjusts the default calculation.)*
