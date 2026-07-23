---
title: Has any role
sidebar_label: Has any role
---

# Has any role

I have successfully resolved the permission issue that prevented technicians from listing users and caused the assignee and work hours technician fields to show IDs or remain empty.

## Changes Made

### backend (maintor-api)

#### [permissions.js](file:///Users/dorbarshalom/Dev/maintor-api/src/utils/permissions.js)
- Added the `hasAnyRole` helper function, which verifies whether a user has at least one of the specified roles active on the account, bypassing specific site-scope requirements for global read-only actions like listing user names.

#### [listUsers.js](file:///Users/dorbarshalom/Dev/maintor-api/src/handlers/listUsers.js)
- Imported and utilized the `hasAnyRole` function in the permission check:
  ```javascript
  const canList = await hasAnyRole(identity.authUserId, accountId, ['OWNER', 'ADMIN', 'SITE_MANAGER', 'TECHNICIAN'], env);
  ```
- This ensures any user with the `TECHNICIAN` role is permitted to retrieve user details to resolve names/assignees.

#### [permissions.test.js](file:///Users/dorbarshalom/Dev/maintor-api/src/__tests__/permissions.test.js)
- Added unit tests for `hasAnyRole` verifying:
  - Grants access if the user has the required role on any site.
  - Denies access if the role is marked as inactive.
  - Denies access if the user does not possess any matching roles.

---

## Verification Results

### Automated Tests
Ran the vitest test suite via `npm test`. All 94 tests passed successfully, including the new tests for `hasAnyRole`:

```bash
Test Files  18 passed (18)
     Tests  94 passed (94)
  Start at  11:08:42
  Duration  593ms (transform 892ms, setup 0ms, import 1.75s, tests 482ms, environment 1ms)
```
