# Teams, Assets, Tags, and Team Assignment Guide

This guide details how teams, assets, tags, and responsibility assignments are structured in Maintor, and how the **Responsibility Resolution Engine** determines which team is responsible for maintaining each asset.

---

## Conceptual Overview

In Maintor, maintenance tasks (planned and breakdown) must be assigned to the correct teams. To automate this, the system maps asset ownership dynamically:

```
+------------------+         +--------------------------+         +---------------+
|      Asset       |         | Responsibility Assignment|         |     Team      |
|  - Site          |         |  - Scope Type & ID       |         |  - Site       |
|  - Location      | ------> |  - Target Team           | ------> |  - Leader     |
|  - Tag Values    |         |  - Date Range            |         |  - Members    |
+------------------+         +--------------------------+         +---------------+
```

The system uses three main entities—**Teams**, **Assets**, and **Tags**—joined together by **Responsibility Assignments** and processed by an evaluation algorithm to produce a cached **Resolved Ownership** record for each asset.

---

## 1. Teams

Teams represent the groups of engineers or technicians performing the maintenance work.

### Structure
- **Site Binding**: A team is explicitly associated with a single site (`siteId`).
- **Members**: An array of user objects:
  - `userId`: Reference to the Firebase user.
  - `role`: Either `leader` or `member`.
  - `joinedAt`: Timestamp when the user joined the team.
- **Constraints**: A team can have **at most one leader**.
- **Management Permissions**: Only users with account-level `OWNER`/`ADMIN` roles, or a site-level `SITE_MANAGER` role for that specific site, can create, update, or delete teams.

---

## 2. Assets

Assets are the physical machines and equipment requiring maintenance.

### Structure
- **Site Binding**: Associated with a specific site (`siteId` / `site_id`).
- **Location binding**: Assigned to a specific node in the site's physical location tree (`structureNodeId` / `node_id`).
- **Tags**: A list of associated tag value IDs (`tagValueIds` / `tag_value_ids`).

---

## 3. Tags (Tag Sets & Tag Values)

Tags categorize assets by attribute, criticality, regulation level, or maintenance requirements.

```
Tag Set (e.g., "Criticality")
 ├── Tag Value (e.g., "High Criticality")
 └── Tag Value (e.g., "Low Criticality")
```

### Tag Sets
- A Tag Set represents a category or dimension of tagging (e.g., "Criticality", "Regulatory").
- Configured at the account level.
- Supports multi-language translations (`translations`).

### Tag Values
- Specific options belonging to a Tag Set (e.g., "Critical", "Non-Critical").
- Can be nested under other Tag Values via `parentId`, forming a tag hierarchy.
- Supports multi-language translations (`translations`).

### Permissions
- Only account-level `OWNER` or `ADMIN` roles can manage Tag Sets and Tag Values.

---

## 4. Responsibility Assignments

Responsibility Assignments define the rules that link teams to specific maintenance scopes.

### Scope Types
1. **Asset**: Direct assignment to a specific asset (`scope_type: "Asset"`, `scope_id: assetId`).
2. **Tag**: Assignment to any asset carrying a specific tag value (`scope_type: "Tag"`, `scope_id: tagValueId`).
3. **Location**: Assignment to a physical location/structure node (`scope_type: "Location"`, `scope_id: nodeId`).

### Properties
- **Date Ranges**: Governed by `start_date` and `end_date` to support scheduling and temporary responsibility transfers.
- **Descendant Inheritance**: Specifically for **Location** scope, the flag `applies_to_descendants` controls whether assets nested under that location automatically inherit the assignment.
- **Management Permissions**: Only account-level `OWNER` or `ADMIN` roles can create, update, or end responsibility assignments.

---

## 5. The Responsibility Resolution Engine

To determine which team owns a given asset, the system runs the **Responsibility Resolution Engine** (`responsibility-resolver.js`).

### The Precedence Tiers
When looking for active assignments that match an asset, the engine evaluates matches in four hierarchical tiers:

| Tier | Scope Type | Description | Precedence |
| :--- | :--- | :--- | :--- |
| **Tier 1** | **Asset** | Direct assignment to the specific asset. | **Highest** |
| **Tier 2** | **Tag** | Assignment matches one of the asset's active tag values. | **Medium** |
| **Tier 3** | **Location** | Assignment matches the asset's location node or its ancestors (parent, grandparent, etc.). If matching an ancestor, `applies_to_descendants` must be `true`. | **Lowest** |
| **Tier 4** | **Fallback** | No active assignments match the asset. Ownership is set to `null`. | **None** |

### Tie-Break Rules
If multiple active assignments match within the **same winning tier**, the engine evaluates them in order using the following tie-breaking rules:

1. **Specificity / Distance**: The assignment closest to the asset wins. For example, under **Tag** scope, a tag value closest to the asset's assigned tag value in the hierarchy chain wins (smaller distance value).
2. **List Order (Priority)**: The position of the assignment in the user-defined list. The higher the assignment is in the list (represented by a smaller `order_index` / `orderIndex`), the higher priority it takes. This order is managed directly by drag-and-drop reordering on the Responsibility page.
3. **ID Sort**: A lexicographical sort on the assignment ID (`id`) is used as a final fallback for strict determinism.

---

## 6. Optimization & Downstream Usage

### Cache System
To avoid expensive database queries during operational workflows, resolved ownership is computed and stored in a dedicated cache:
- Collection: `resolved_ownership`
- Recalculated automatically whenever an assignment is created, updated, or ended, or when triggered manually.

### Maintenance Task Assignment
- **Planned Maintenance (Recurring Tasks)**: When a planned maintenance event is generated, the system resolves its owner (assignee user) by identifying the team that owns the first asset in the maintenance task list. The task is then assigned to the **team leader** of that team.
