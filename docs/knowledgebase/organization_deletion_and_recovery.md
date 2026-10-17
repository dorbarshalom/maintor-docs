---
title: Organization Deletion & Disaster Recovery
sidebar_label: Org Deletion & Recovery
description: How organization deletion works in Maintor Admin, data cascade scope, and disaster recovery procedures via GCP.
---

# Organization Deletion & Disaster Recovery

This guide explains how organization deletion functions within **Maintor Admin**, what data is removed during the process, and the technical disaster recovery procedures available to reverse an accidental deletion.

---

## 1. Deletion Workflow in Maintor Admin

In the Maintor Admin portal (`maintor-admin`), super administrators can delete an organization from the **Organizations** page.

### Confirmation Guardrail
To prevent accidental clicks, the UI displays a modal requiring explicit confirmation:
1. Warning indicator: *"This action cannot be undone. Deleting [Org Name] will remove all associated organization data."*
2. Required input: The administrator must type `delete <orgname>` before the **Delete Organization** button becomes active.

### API Invocation
Once confirmed, the frontend triggers:
* **HTTP Route**: `DELETE /admin/v1/orgs/:id`
* **Handler**: `maintor-api/src/handlers/admin/deleteAdminOrg.js`
* **Database Function**: `deleteAccountDb(orgId, env)` in `maintor-api/src/db/accounts.js`

---

## 2. Cascade Hard-Deletion Scope

Maintor performs a **permanent cascade hard delete**, not a soft delete (e.g. `isDeleted: true`). When the deletion executes, the following data is permanently purged:

### A. Database Collections
Documents matching the deleted `accountId` are deleted across all collections in parallel:
* **Accounts**: The core organization record in `accounts`.
* **Users & Access**: `users`, `roles`, `account_invitations`.
* **Facilities & Hierarchy**: `sites`, `structure_nodes`, `structure_schemas`.
* **Assets & Inventory**: `assets`, `asset_types`, `parts`.
* **Maintenance & Work Orders**: `tickets`, `ticket_status_logs`, `planned_maintenance`.
* **Taxonomy & Settings**: `tag_values`, `root_causes`.

### B. Cloud Storage Buckets
All files stored under the organization ID in Google Cloud Storage (`maintor.firebasestorage.app`) are deleted across prefixes:
* `uploads/<orgId>/`
* `sites/<orgId>/`
* `users/<orgId>/`
* `assets/<orgId>/`
* `tickets/<orgId>/`
* `parts/<orgId>/`
* `orgs/<orgId>/`
* `documents/<orgId>/`
* `invitations/<orgId>/`

---

## 3. Reversing an Accidental Deletion

Because the deletion is an immediate hard cascade delete, **there is no in-app "Undo" or recycle bin in Maintor Admin**. Reversing a deletion requires disaster recovery procedures at the Google Cloud Platform (GCP) infrastructure layer.

### Method 1: Firestore Point-in-Time Recovery (PITR)
*(Fastest and most complete method if PITR is active on the Firestore database)*

PITR allows restoring the database to any specific microsecond in the past 7 days.

1. Open the [Google Cloud Console](https://console.cloud.google.com/) and navigate to **Firestore** → **Databases**.
2. Select your database (typically `(default)`).
3. Click **Restore**.
4. Set the recovery timestamp to a minute or two **before** the accidental deletion occurred.
5. Provide a destination database ID (e.g. `maintor-restore-temp`).
6. Using the Cloud SDK / CLI:
   ```bash
   gcloud firestore databases restore \
     --source-database='(default)' \
     --destination-database='maintor-restore-temp' \
     --recovery-time='YYYY-MM-DDTHH:MM:SSZ'
   ```
7. Once the restore finishes, export the specific organization's records (`accountId == <orgId>`) from the restored database and re-import them into the primary database.

---

### Method 2: Scheduled Backups / Daily Exports
If automated backup schedules or Cloud Storage export jobs are configured:

1. Check **Firestore** → **Backups** in GCP Console.
2. Restore the latest daily backup snapshot to an auxiliary database.
3. Query and restore the records associated with the deleted `accountId`.

---

### Method 3: Restoring Cloud Storage Files (Soft Delete)
By default, modern Google Cloud Storage buckets enable a **7-day soft-delete retention**:

1. In the GCP Console, go to **Cloud Storage** → **Buckets** → `maintor.firebasestorage.app`.
2. Check the **Soft-deleted objects** view for files deleted under `<prefix>/<orgId>/`.
3. Restore the deleted files using the console or the `gcloud storage` CLI:
   ```bash
   gcloud storage restore "gs://maintor.firebasestorage.app/**"
   ```

---

## 4. Recommended Architectural Improvement

To eliminate the risk of irreversible data loss, consider implementing a **Soft Delete** pattern:
* Add `isDeleted: true` and `deletedAt: timestamp` fields to the `accounts` collection.
* Update queries to filter `{ isDeleted: { $ne: true } }`.
* Provide an **Archived / Deleted Organizations** tab in Maintor Admin with an immediate **Restore** button.
* Run a delayed background worker (e.g., 30-day purge) for permanent cascade destruction.
