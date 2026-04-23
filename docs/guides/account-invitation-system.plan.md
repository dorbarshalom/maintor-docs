# Account Invitation System Implementation Plan

## 1. Data Models

### Account Model Extension (update existing `accounts` collection)
- Add `invitedTo`: Array of objects (accounts that invited THIS account):
  - `accountId`: Account ID that invited this account
  - `invitedBy`: User ID who sent the invitation (from inviting account)
  - `invitedAt`: Timestamp when invitation was accepted
  - `role`: Role assigned (e.g., "CONSULTANT")
  - `inviterFirstName`: First name of user who sent the invitation
  - `inviterLastName`: Last name of user who sent the invitation

- Add `invitedAccounts`: Array of objects (accounts that THIS account invited):
  - `accountId`: Account ID that was invited by this account
  - `invitedBy`: User ID who sent the invitation (from this account)
  - `invitedAt`: Timestamp when invitation was accepted
  - `role`: Role assigned (e.g., "CONSULTANT")
  - `inviterFirstName`: First name of user who sent the invitation
  - `inviterLastName`: Last name of user who sent the invitation

### Account Invitation Model (`account_invitations` collection)
- `id`: Invitation ID
- `invitingAccountId`: Account that sends the invitation
- `invitedAccountId`: Account being invited
- `invitedByUserId`: User ID who sent the invitation (from inviting account)
- `inviterFirstName`: First name of user who sent the invitation
- `inviterLastName`: Last name of user who sent the invitation
- `accountRole`: Role assigned to invited account (e.g., "CONSULTANT")
- `siteIds`: Array of site IDs that the invited account will have access to (sites from inviting account)
- `createdAt`: Creation timestamp
- Note: Invitations are PENDING until accepted, then removed from collection. No status field or history tracking needed.

### Site Roles Model (`site_roles` collection)
- `id`: Site role assignment ID
- `userId`: User ID
- `userAccountId`: The user's account ID (the account the user belongs to)
- `accountId`: Account ID (can be the user's own account OR an account that invited the user's account)
- `siteId`: Site ID
- `role`: Site role (e.g., "ADMIN", "TECHNICIAN", "CONSULTANT", "SITE_MANAGER", "VIEWER")
- `isActive`: Boolean - set to `true` when role is active, `false` when revoked by the inviting account
- `createdAt`: Creation timestamp
- `updatedAt`: Update timestamp
- Note: When account A invites account B, site roles for users of account B are created/updated with `accountId = A` and `isActive = true`. When revoked, `isActive` is set to `false`.

## 2. Database Functions

### Account Invitations Collection
- `src/db/account-invitations.js`:
  - `createAccountInvitationDb(invitationData, env)` - Create invitation
  - `getAccountInvitationDb(invitationId, env)` - Get invitation by ID
  - `getInvitationByAccountsDb(invitingAccountId, invitedAccountId, env)` - Find invitation by account IDs
  - `listInvitationsSentDb(accountId, env)` - List invitations sent by an account
  - `listInvitationsReceivedDb(accountId, env)` - List invitations received by an account
  - `acceptAccountInvitationDb(invitationId, env)` - Accept invitation (adds to invitedTo on invited account, adds to invitedAccounts on inviting account, removes from invitations collection)
  - `deleteAccountInvitationDb(invitationId, env)` - Delete/reject invitation

### Account Model Updates
- `src/db/accounts.js`:
  - `updateAccountInvitedToDb(accountId, invitationData, env)` - Add entry to `invitedTo` array (on invited account)
  - `updateAccountInvitedAccountsDb(accountId, invitationData, env)` - Add entry to `invitedAccounts` array (on inviting account)
  - `getAccountInvitedToDb(accountId, env)` - Get all accounts this account was invited to
  - `getAccountInvitedAccountsDb(accountId, env)` - Get all accounts this account invited

### Site Roles Collection
- `src/db/site-roles.js`:
  - `createSiteRoleDb(siteRoleData, env)` - Create site role assignment
  - `getSiteRoleDb(roleId, env)` - Get site role by ID
  - `getUserSiteRolesDb(userId, accountId, env)` - Get all site roles for a user in an account
  - `getSiteRolesByAccountSiteDb(accountId, siteId, env)` - Get all role assignments for a specific site
  - `updateSiteRoleDb(roleId, updates, env)` - Update site role (e.g., change role, set isActive)
  - `revokeSiteRolesByAccountDb(userAccountId, invitingAccountId, env)` - Set `isActive: false` for all site roles where `userAccountId` matches and `accountId` matches inviting account
  - `activateSiteRolesByAccountDb(userAccountId, invitingAccountId, env)` - Set `isActive: true` for all site roles where `userAccountId` matches and `accountId` matches inviting account

## 3. API Handlers

### Account Invitations
- `src/handlers/createAccountInvitation.js` - Invite an account
  - Validate that inviting account exists
  - Validate that invited account exists
  - Validate that specified siteIds belong to inviting account
  - Create invitation record with specified siteIds
  - Note: Frontend handles displaying pending invitations
  
- `src/handlers/listAccountInvitations.js` - List invitations
  - Query parameter: `direction=sent|received` (default: both)
  - Returns pending invitations
  
- `src/handlers/getAccountInvitation.js` - Get invitation details
  
- `src/handlers/acceptAccountInvitation.js` - Accept invitation
  - Add entry to invited account's `invitedTo` array
  - Add entry to inviting account's `invitedAccounts` array
  - Note: Site roles are NOT automatically created. The invited account owner/admin will assign site-specific access to their users after acceptance.
  - Remove invitation from `account_invitations` collection
  
- `src/handlers/rejectAccountInvitation.js` - Reject invitation
  - Remove invitation from `account_invitations` collection
  
- `src/handlers/cancelAccountInvitation.js` - Cancel invitation (by inviter)
  - Remove invitation from `account_invitations` collection

### Site Role Management (for Invited Account Owners/Admins)
- `src/handlers/assignSiteRole.js` - Assign site role to user (by invited account owner/admin)
  - Validates that user belongs to invited account
  - Validates that account has access to the specified site (via invitation)
  - Creates or updates site role in `site_roles` collection with `isActive: true`
  - Only accessible by invited account owner/admin for their own users

- `src/handlers/updateSiteRole.js` - Update site role for user
  - Validates permissions (invited account owner/admin)
  - Updates role assignment in `site_roles` collection

- `src/handlers/revokeSiteRole.js` - Revoke site role for user
  - Validates permissions (invited account owner/admin)
  - Sets `isActive: false` for site role

- `src/handlers/listUserSiteRoles.js` - List site roles for a user
  - Returns all site roles (active and inactive) for a user
  - Filters by account and site if specified

### Account Invitation Revocation
- `src/handlers/revokeAccountInvitation.js` - Revoke access (async)
  - Validates that inviting account is revoking access
  - Creates async task to revoke site roles for all users of invited account
  - Uses Cloud Tasks to process revocation asynchronously

- `src/handlers/processAccountRevocationTask.js` - Process revocation task
  - Internal endpoint called by Cloud Tasks
  - Receives task payload: `{ invitingAccountId, invitedAccountId }`
  - Finds all users belonging to invited account
  - Sets `isActive: false` for all site roles where `userAccountId = invitedAccountId` and `accountId = invitingAccountId`

## 4. Cloud Tasks Integration

### Cloud Tasks Helper
- `src/utils/cloud-tasks.js`:
  - `pushAccountRevocationTask(invitingAccountId, invitedAccountId, env)` - Queue revocation task

### Task Processor
- `src/handlers/processAccountRevocationTask.js`:
  - Receives task payload: `{ invitingAccountId, invitedAccountId }`
  - Finds all users belonging to invited account
  - Sets `isActive: false` for all site roles where `userAccountId = invitedAccountId` and `accountId = invitingAccountId`

## 5. API Routes

Add to `index.js`:
- `POST /v1/accounts/:accountId/invitations` - Invite account (with siteIds)
- `GET /v1/accounts/:accountId/invitations` - List invitations (sent/received)
- `GET /v1/accounts/:accountId/invitations/:invitationId` - Get invitation
- `POST /v1/accounts/:accountId/invitations/:invitationId/accept` - Accept invitation
- `POST /v1/accounts/:accountId/invitations/:invitationId/reject` - Reject invitation
- `DELETE /v1/accounts/:accountId/invitations/:invitationId` - Cancel invitation (by inviter)
- `POST /v1/accounts/:accountId/invitations/:invitationId/revoke` - Revoke invitation access (async)
- `POST /v1/internal/process-account-revocation-task` - Process revocation task (internal, Cloud Tasks)
- `POST /v1/accounts/:accountId/users/:userId/site-roles` - Assign site role to user (by invited account owner/admin)
- `PATCH /v1/accounts/:accountId/users/:userId/site-roles/:roleId` - Update site role
- `DELETE /v1/accounts/:accountId/users/:userId/site-roles/:roleId` - Revoke site role
- `GET /v1/accounts/:accountId/users/:userId/site-roles` - List site roles for user

## 6. OpenAPI Documentation

Update `openapi.json` with:
- Account invitation endpoints
- Request/response schemas
- `invitedTo` and `invitedAccounts` array structures in Account model
- `site_roles` collection model structure
- Site role management endpoints (if needed for direct role assignment)
- Async revocation endpoint documentation

## 7. Deployment

Update `deploy-complete.sh`:
- Ensure Cloud Tasks queue exists (reuse existing queue or create new one)
- Configure internal endpoint permissions for Cloud Tasks

## Implementation Notes

1. **Invitation Flow**:
   - Account A invites Account B by providing Account B's ID and specifying which sites (siteIds) Account B will have access to
   - Invitation created in `account_invitations` collection with `siteIds` array
   - Account B accepts invitation
   - Entry added to Account B's `invitedTo` array (Account B was invited TO Account A)
   - Entry added to Account A's `invitedAccounts` array (Account A invited Account B)
   - Note: Site roles are NOT automatically created at this point
   - Invitation removed from `account_invitations` collection
   - After acceptance, Account B's owner/admin can assign site-specific roles to their users using the site role management endpoints
   - When assigning site roles, Account B's owner/admin can only assign roles for sites specified in the original invitation

2. **Revocation Flow**:
   - Account A revokes access to Account B
   - Async task queued via Cloud Tasks
   - Task processor finds all users belonging to Account B
   - Sets `isActive: false` for all site roles where `userAccountId = Account B` and `accountId = Account A`
   - Note: The entries in `invitedTo` and `invitedAccounts` arrays remain (no deletion, just revocation of site access)

3. **User Management**:
   - Users are managed separately (Firebase Auth or separate service)
   - This system only handles account-to-account relationships
   - When account B is invited to account A, all users of account B get access (handled by user management system, not this implementation)

4. **Data Model Notes**:
   - `invitedTo` and `invitedAccounts` arrays are stored directly on the account document
   - `account_invitations` collection only stores pending invitations
   - Once accepted, invitation moves from collection to both accounts:
     - Added to invited account's `invitedTo` array
     - Added to inviting account's `invitedAccounts` array
   - No status field needed - pending invitations exist, accepted ones are in both arrays, rejected/cancelled ones are deleted
   - Both arrays track the relationship bidirectionally for efficient querying
   - `site_roles` collection manages actual site-level access:
     - Tracks which users have access to which sites in which accounts
     - `accountId` can be the user's own account OR an account that invited the user's account
     - When an account invitation is revoked, `isActive` is set to `false` for all related site roles
     - Site roles remain in the collection even when revoked (for audit/history purposes)
   - Example: User 123 from Account 456 can have:
     - Admin role in Account 456 (their own account)
     - Technician role in Account 789 > Site 1010 (Account 789 invited Account 456 to site 1010, then Account 456 owner assigned role)
     - Consultant role in Account 876 > Sites 292 and 298 (Account 876 invited Account 456 to sites 292 and 298, then Account 456 owner assigned roles)
   - Site role assignment is managed by the invited account owner/admin after the invitation is accepted

