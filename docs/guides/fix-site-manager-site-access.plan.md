# Fix SITE_MANAGER and CONSULTANT Site Access and Include Site Names

## Problem

1. `listSites` blocks SITE_MANAGER/CONSULTANT (same issue as `listUsers` had)
2. Roles only include `siteId`, not site names, so frontend can't display site names
3. SITE_MANAGER/CONSULTANT should only see sites they have access to
4. SITE_MANAGER/CONSULTANT should only see users that have access to their accessible sites
5. When SITE_MANAGER views a user's roles, they should:
   - See roles for sites they have access to (editable)
   - See roles for sites they don't have access to (read-only, for visibility)
   - Not be able to modify roles for sites they don't have access to

## Solution

### 1. Update `listSites` Handler
- Replace account ownership check with role-based permissions
- Allow OWNER, ADMIN, SITE_MANAGER, or CONSULTANT to list sites
- Filter sites based on user's roles:
  - OWNER/ADMIN: See all sites
  - SITE_MANAGER/CONSULTANT: Only see sites they have access to (from their roles)

### 2. Update `listUsers` Handler
- Filter users based on requester's site access:
  - OWNER/ADMIN: See all users
  - SITE_MANAGER/CONSULTANT: Only see users that have roles for sites the requester has access to

### 3. Enhance Roles Responses with Site Names
- Update `listUserRoles` and `getMyRoles` to include `siteName` for site-level roles
- Fetch site names from database when returning roles
- Include `siteName: null` for account-level roles or ALL_SITES

## Implementation

### 1. Create Helper Function: Get Accessible Site IDs
- New function in `src/utils/permissions.js`: `getAccessibleSiteIds(authUserId, accountId, env)`
- Returns array of site IDs the user has access to (from their roles)
- Includes sites from:
  - Site-specific roles (siteId = specific site)
  - ALL_SITES roles (returns special marker or all site IDs)
  - Account-level roles (returns all site IDs)
- Returns `null` if user has account-level OWNER/ADMIN (means all sites)

### 2. Update `src/handlers/listSites.js`
- Remove `identity.accountId !== accountId` check
- Add permission check using `canPerformAction` for ['OWNER', 'ADMIN', 'SITE_MANAGER', 'CONSULTANT']
- Get accessible site IDs for requester
- If requester has account-level OWNER/ADMIN, return all sites
- If requester is SITE_MANAGER/CONSULTANT, filter sites to only those in accessible site IDs
- Update `listSitesDb` to accept optional `siteIds` filter parameter

### 3. Update `src/handlers/listUsers.js`
- After getting users, filter based on requester's accessible sites:
  - Get requester's accessible site IDs
  - For each user, get their roles
  - Include user if they have any role for a site in requester's accessible sites
  - OWNER/ADMIN see all users (no filtering)

### 4. Update `src/db/sites.js`
- Add optional `siteIds` parameter to `listSitesDb` for filtering by site IDs
- Filter MongoDB query to include only specified site IDs if provided

### 5. Enhance `src/handlers/listUserRoles.js`
- After getting roles, fetch site names for site-level roles
- For each role with `siteId` (not null, not ALL_SITES), fetch site name using `getSiteDb`
- Add `siteName` field to role object
- For ALL_SITES, set `siteName: "All Sites"` or similar
- For account-level roles, set `siteName: null`
- **Add `canEdit` field to each role**:
  - `canEdit: true` if requester has OWNER/ADMIN role OR requester has access to the role's site
  - `canEdit: false` if requester is SITE_MANAGER/CONSULTANT and doesn't have access to the role's site
  - For account-level roles: `canEdit: true` only if requester is OWNER/ADMIN
- Get requester's accessible site IDs to determine `canEdit` for each role

### 6. Enhance `src/handlers/getMyRoles.js`
- Include site names (same as `listUserRoles`)
- **Do NOT add `canEdit` field** - users can always see their own roles, but editing permissions are handled by assignment endpoints

### 7. Update `src/handlers/updateRole.js`
- Add permission check before updating role:
  - Get requester's accessible site IDs
  - If role is site-level (has siteId), check if requester has access to that site
  - If role is account-level, only OWNER/ADMIN can update
  - SITE_MANAGER/CONSULTANT can only update roles for sites they have access to
- Return 403 if requester doesn't have permission

### 8. Update `src/handlers/revokeRole.js`
- Add same permission check as `updateRole`:
  - Get requester's accessible site IDs
  - If role is site-level, check if requester has access to that site
  - If role is account-level, only OWNER/ADMIN can revoke
  - SITE_MANAGER/CONSULTANT can only revoke roles for sites they have access to
- Return 403 if requester doesn't have permission

### 9. Verify `assignSiteRoleToUser`
- Already has permission checks, but verify they work correctly for the new scenario

### 10. Update `openapi.json`
- Update `listSites` description to mention role requirements and filtering
- Update `listUsers` description to mention filtering for SITE_MANAGER/CONSULTANT
- Update Role schema to include optional `siteName` and `canEdit` fields
- Update `listUserRoles` description to explain `canEdit` field behavior
- Add 403 responses where needed

## Files to Modify

- `src/utils/permissions.js` - Add `getAccessibleSiteIds` helper function
- `src/handlers/listSites.js` - Add role-based permissions and site filtering
- `src/handlers/listUsers.js` - Add user filtering based on accessible sites
- `src/db/sites.js` - Add `siteIds` filter parameter to `listSitesDb`
- `src/handlers/listUserRoles.js` - Include site names and `canEdit` field in response
- `src/handlers/getMyRoles.js` - Include site names in response
- `src/handlers/updateRole.js` - Verify/add permission checks for SITE_MANAGER
- `src/handlers/revokeRole.js` - Verify/add permission checks for SITE_MANAGER
- `openapi.json` - Update documentation

## Testing

- SITE_MANAGER should only see sites they have access to
- SITE_MANAGER should only see users with roles for their accessible sites
- OWNER/ADMIN should see all sites and all users
- Roles responses should include site names for site-level roles
- ALL_SITES roles should show appropriate site name
- Account-level roles should have `siteName: null`
- When SITE_MANAGER views a user's roles:
  - Roles for accessible sites should have `canEdit: true`
  - Roles for inaccessible sites should have `canEdit: false`
  - Account-level roles should have `canEdit: false` (unless requester is OWNER/ADMIN)
- SITE_MANAGER should not be able to update/revoke roles for sites they don't have access to

