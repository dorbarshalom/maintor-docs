---
title: Technician Site Access & Data Visibility Guide
sidebar_position: 5
---

# Technician Site Access & Data Visibility Guide

This guide documents the role-based access control (RBAC) rules and data visibility boundaries for a user with the **TECHNICIAN** role when accessing sites and site-related data in Maintor.

---

## 1. Site-Level Access Restrictions (RBAC Scope)

At the account level, site access is restricted based on the user's role scope:
* **Restricted View:** Unlike users with account-level roles (`OWNER`, `ADMIN`) or site-level `ADMIN` with `ALL_SITES` access who can view all sites, a user with the `TECHNICIAN` role is restricted to viewing and accessing **only the specific site(s) they are explicitly assigned to** (matching their active roles where `siteId` is specified).
* **API Enforcement (`listSites`):** The backend endpoint `GET /v1/accounts/:accountId/sites` checks user roles and filters out any sites that the user does not have an active site-scoped role for (unless they have account-level full access).
* **Mobile-App Auto-Selection (`maintor-engineers`):** In the technician mobile application, if a technician is assigned to only a single site, that site is automatically pre-selected, and the site selection input is disabled.

---

## 2. Visible Site-Scoped Data

Once a technician is authorized and accesses a specific site, they have read-only or read/write access to the following site-specific resources:

### A. Site Details
Basic information about the physical location (fetched via `GET /v1/accounts/:accountId/sites/:siteId`):
* Site ID, Name, and Status.
* Location coordinates/address details.
* Metadata and configuration specific to the site.

### B. Assets
A technician can view all assets and equipment belonging to the site (fetched via `GET /v1/accounts/:accountId/sites/:siteId/assets`). Visible fields include:
* **Asset Name & Type:** (e.g., Compressor, Conveyor Belt, Pump).
* **Visual ID / Serial Number:** For physically locating and identifying the asset.
* **Status:** Current operational state (e.g., Operational, Down).
* **Node Assignee User:** The default owner/assignee for tasks associated with the asset's organizational chart node.

### C. Tickets (Work Orders)
Technicians can view tickets related to the site (fetched via `GET /v1/accounts/:accountId/tickets`). These are categorized into:

#### 1. Breakdown Tickets (Emergency / Unplanned)
Emergency maintenance tickets created when equipment fails. Visible data includes:
* **Ticket Details:** Title, problem description, priority (1 to 5), status, and assigned asset.
* **Timeline Info:** Work start time, end time, and calculated downtime.
* **Labor Entries:** A record of which technicians worked on the issue, when they started/ended, and the total duration.
* **Root Cause & Solution:** The selected root cause (e.g., normal wear and tear, human error) and the solution description.
* **Photos & Notes:** Photos taken on-site before/after the repair, and user-added work notes.

#### 2. Planned Tickets (Scheduled / Preventive)
Scheduled tasks generated from preventive maintenance templates. Visible data includes:
* **Tasks Checklist:** List of items to verify (e.g., "Check oil level", "Clean filters") with completion statuses (`PENDING`, `DONE`, `SKIPPED`, `FAILED`).
* **Schedules:** Scheduled execution date and estimated duration.
* **Assignees & Owner:** Assigned technician(s) and owner of the ticket.

### D. Root Causes
A list of predefined account-level root causes (fetched via `GET /v1/accounts/:accountId/root-causes`) used by the technician to classify equipment breakdowns (e.g., `normal_wear_and_tear`, `equipment_failure`, `human_error`).

### E. Team Members (Users)
A list of other users in the account (fetched via `GET /v1/accounts/:accountId/users`). This allows technicians to see:
* Assignees for tickets (so they can collaborate or reassign tickets).
* The user who reported a breakdown.
