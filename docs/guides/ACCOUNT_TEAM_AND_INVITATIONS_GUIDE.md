# Account Team Members and Account Invitations Guide

## Overview

This guide explains the structure of account team members and how accounts can invite other accounts to collaborate. The system supports multi-account collaboration where one account can grant another account access to specific sites.

## Account Team Members Structure

### Users and Accounts

- **Account**: An organization/company that uses the Maintor system
- **User**: An individual person who belongs to an account
- **Role**: Permissions assigned to a user within an account or at specific sites

### User Roles

Users can have different roles with varying levels of access:

#### Account-Level Roles
- **OWNER**: Full account control, exclusive access to billing and account deletion
- **ADMIN**: Account administration, can manage users and settings (but not billing/deletion)

#### Site-Level Roles
- **SITE_MANAGER**: Manages a specific site or all sites
- **TECHNICIAN**: Performs maintenance work
- **CONSULTANT**: Full admin access, can be invited by multiple accounts
- **VIEWER**: Read-only access

### Role Scope

Roles can be assigned at two levels:

1. **Account Scope** (`scope: "account"`):
   - Applies to all sites in the account
   - `siteId` is `null`
   - Only OWNER and ADMIN can have account-level roles

2. **Site Scope** (`scope: "site"`):
   - Applies to specific site(s)
   - `siteId` can be:
     - A specific site ID (e.g., `"site_123"`)
     - `"ALL_SITES"` (access to all sites in the account)

### Team Member Hierarchy

```
Account
├── Users (team members)
│   ├── User 1 (OWNER - account level)
│   ├── User 2 (ADMIN - account level)
│   ├── User 3 (SITE_MANAGER - site level, Site A)
│   ├── User 4 (TECHNICIAN - site level, Site B)
│   └── User 5 (CONSULTANT - site level, ALL_SITES)
└── Sites
    ├── Site A
    └── Site B
```

### Permissions Summary

| Role | Account-Level | Site-Level | Can Invite Users | Can Invite Accounts |
|------|--------------|------------|------------------|---------------------|
| OWNER | ✅ | ✅ | ✅ (all sites) | ✅ |
| ADMIN | ✅ | ✅ (ALL_SITES) | ✅ (all sites) | ✅ |
| SITE_MANAGER | ❌ | ✅ (specific site) | ✅ (their site only) | ❌ |
| TECHNICIAN | ❌ | ✅ (specific site) | ❌ | ❌ |
| CONSULTANT | ❌ | ✅ (specific sites) | ❌ | ❌ |
| VIEWER | ❌ | ✅ (specific sites) | ❌ | ❌ |

## Account-to-Account Invitations

### Overview

Account invitations allow one account (the **inviting account**) to grant another account (the **invited account**) access to specific sites. This enables cross-account collaboration, particularly useful for consultants who work with multiple organizations.

### Use Cases

1. **Consultant Access**: A consulting firm (Account B) is invited by a client (Account A) to access their sites
2. **Multi-Company Collaboration**: Two companies collaborate on shared facilities
3. **Service Provider Access**: A maintenance service provider is granted access to client sites

### How It Works

#### 1. Invitation Flow

```
Account A (Inviting Account)
    ↓ Creates invitation
    ↓ Specifies: Account B ID, role (e.g., "CONSULTANT"), site IDs
    ↓
Account B (Invited Account)
    ↓ Receives invitation
    ↓ Accepts invitation
    ↓
Account B's owner/admin assigns site roles to their users
    ↓
Users from Account B can now access Account A's sites
```

#### 2. Data Structure

**Account Invitation** (temporary, removed after acceptance):
```json
{
  "id": "invitation_123",
  "invitingAccountId": "account_a",
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"],
  "invitedByUserId": "user_456",
  "inviterFirstName": "John",
  "inviterLastName": "Doe",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

**Account `invitedTo` Array** (on invited account):
```json
{
  "invitedTo": [
    {
      "accountId": "account_a",
      "invitedBy": "user_456",
      "invitedAt": "2024-01-15T10:30:00Z",
      "role": "CONSULTANT",
      "inviterFirstName": "John",
      "inviterLastName": "Doe"
    }
  ]
}
```

**Account `invitedAccounts` Array** (on inviting account):
```json
{
  "invitedAccounts": [
    {
      "accountId": "account_b",
      "invitedBy": "user_456",
      "invitedAt": "2024-01-15T10:30:00Z",
      "role": "CONSULTANT",
      "inviterFirstName": "John",
      "inviterLastName": "Doe"
    }
  ]
}
```

#### 3. Site Roles for Invited Accounts

After an invitation is accepted, the invited account's owner/admin must assign site-specific roles to their users:

```json
{
  "userId": "user_from_account_b",
  "userAccountId": "account_b",
  "accountId": "account_a",  // The inviting account
  "siteId": "site_1",         // Site from account A
  "role": "CONSULTANT",
  "isActive": true
}
```

**Key Points:**
- `userAccountId`: The account the user belongs to (Account B)
- `accountId`: The account that invited them (Account A)
- `isActive`: Set to `false` when access is revoked

## API Endpoints

### Account Invitations

#### Create Account Invitation

**Endpoint:** `POST /v1/accounts/{accountId}/invitations`

**Description:** Invite another account to access your sites.

**Permissions:** OWNER or ADMIN (account-level)

**Request Body:**
```json
{
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"]
}
```

**Response (201 Created):**
```json
{
  "id": "invitation_123",
  "invitingAccountId": "account_a",
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"],
  "invitedByUserId": "user_456",
  "inviterFirstName": "John",
  "inviterLastName": "Doe",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

**Validation:**
- `invitedAccountId` must exist
- All `siteIds` must belong to the inviting account
- `accountRole` is required (typically "CONSULTANT")

#### List Account Invitations

**Endpoint:** `GET /v1/accounts/{accountId}/invitations?direction=sent|received|both`

**Description:** List invitations sent or received by an account.

**Query Parameters:**
- `direction`: `sent` (invitations sent), `received` (invitations received), or `both` (default)

**Response (200 OK):**
```json
[
  {
    "id": "invitation_123",
    "invitingAccountId": "account_a",
    "invitedAccountId": "account_b",
    "accountRole": "CONSULTANT",
    "siteIds": ["site_1", "site_2"],
    "createdAt": "2024-01-15T10:00:00Z"
  }
]
```

#### Get Account Invitation

**Endpoint:** `GET /v1/accounts/{accountId}/invitations/{invitationId}`

**Description:** Get details of a specific invitation.

**Response (200 OK):**
```json
{
  "id": "invitation_123",
  "invitingAccountId": "account_a",
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"],
  "invitedByUserId": "user_456",
  "inviterFirstName": "John",
  "inviterLastName": "Doe",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Accept Account Invitation

**Endpoint:** `POST /v1/accounts/{accountId}/invitations/{invitationId}/accept`

**Description:** Accept an invitation to access another account's sites.

**Permissions:** OWNER or ADMIN of the invited account

**Response (200 OK):**
```json
{
  "message": "Invitation accepted successfully",
  "invitation": {
    "id": "invitation_123",
    "status": "accepted"
  }
}
```

**What Happens:**
1. Entry added to invited account's `invitedTo` array
2. Entry added to inviting account's `invitedAccounts` array
3. Invitation removed from `account_invitations` collection
4. **Note:** Site roles are NOT automatically created - the invited account owner/admin must assign them

#### Reject Account Invitation

**Endpoint:** `POST /v1/accounts/{accountId}/invitations/{invitationId}/reject`

**Description:** Reject an invitation.

**Response (200 OK):**
```json
{
  "message": "Invitation rejected"
}
```

#### Cancel Account Invitation

**Endpoint:** `DELETE /v1/accounts/{accountId}/invitations/{invitationId}`

**Description:** Cancel an invitation you sent (before it's accepted).

**Permissions:** OWNER or ADMIN of the inviting account

**Response (200 OK):**
```json
{
  "message": "Invitation cancelled"
}
```

#### Revoke Account Invitation

**Endpoint:** `POST /v1/accounts/{accountId}/invitations/{invitationId}/revoke`

**Description:** Revoke access for an invited account (after it was accepted).

**Permissions:** OWNER or ADMIN of the inviting account

**Response (202 Accepted):**
```json
{
  "message": "Account invitation revocation queued successfully"
}
```

**What Happens:**
1. Async task queued to revoke site roles
2. All site roles for users from the invited account are set to `isActive: false`
3. The `invitedTo` and `invitedAccounts` entries remain (history preserved)

## Complete Workflow Example

### Scenario: Account A invites Account B as a Consultant

#### Step 1: Account A Creates Invitation

**Request:**
```http
POST /v1/accounts/account_a/invitations
Authorization: IDTOKEN.<token>
Content-Type: application/json

{
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"]
}
```

**Response:**
```json
{
  "id": "invitation_123",
  "invitingAccountId": "account_a",
  "invitedAccountId": "account_b",
  "accountRole": "CONSULTANT",
  "siteIds": ["site_1", "site_2"],
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Step 2: Account B Views Received Invitations

**Request:**
```http
GET /v1/accounts/account_b/invitations?direction=received
Authorization: IDTOKEN.<token>
```

**Response:**
```json
[
  {
    "id": "invitation_123",
    "invitingAccountId": "account_a",
    "invitedAccountId": "account_b",
    "accountRole": "CONSULTANT",
    "siteIds": ["site_1", "site_2"],
    "inviterFirstName": "John",
    "inviterLastName": "Doe",
    "createdAt": "2024-01-15T10:00:00Z"
  }
]
```

#### Step 3: Account B Accepts Invitation

**Request:**
```http
POST /v1/accounts/account_b/invitations/invitation_123/accept
Authorization: IDTOKEN.<token>
```

**Response:**
```json
{
  "message": "Invitation accepted successfully"
}
```

**Result:**
- Account B's `invitedTo` array now includes Account A
- Account A's `invitedAccounts` array now includes Account B
- Invitation removed from collection

#### Step 4: Account B Assigns Site Roles to Users

**Request:**
```http
POST /v1/accounts/account_a/users/user_from_b/roles
Authorization: IDTOKEN.<token>
Content-Type: application/json

{
  "scope": "site",
  "siteId": "site_1",
  "role": "CONSULTANT"
}
```

**Note:** The `accountId` in the URL is Account A (the inviting account), but the user belongs to Account B.

**Response:**
```json
{
  "id": "role_789",
  "userId": "user_from_b",
  "userAccountId": "account_b",
  "accountId": "account_a",
  "scope": "site",
  "siteId": "site_1",
  "role": "CONSULTANT",
  "isActive": true
}
```

#### Step 5: User from Account B Can Now Access Account A's Sites

The user can now:
- View tickets for Site 1 and Site 2
- Create tickets (if role permits)
- Access site-specific data

#### Step 6: Account A Revokes Access (Optional)

**Request:**
```http
POST /v1/accounts/account_a/invitations/invitation_123/revoke
Authorization: IDTOKEN.<token>
```

**Result:**
- All site roles for Account B users are set to `isActive: false`
- Users from Account B lose access to Account A's sites
- History preserved in `invitedTo` and `invitedAccounts` arrays

## Important Notes

### Permissions

1. **Creating Invitations:**
   - Only OWNER or ADMIN (account-level) can create account invitations
   - Site managers cannot invite accounts

2. **Accepting Invitations:**
   - Only OWNER or ADMIN of the invited account can accept invitations
   - The invitation must be for the account making the request

3. **Assigning Site Roles:**
   - After acceptance, the invited account's owner/admin assigns site roles
   - Can only assign roles for sites specified in the original invitation
   - Site roles reference the inviting account's ID

### Site Role Assignment

When assigning site roles for invited accounts:

- **URL Account ID**: Use the inviting account's ID (Account A)
- **User ID**: Use a user ID from the invited account (Account B)
- **Site ID**: Must be one of the sites specified in the invitation
- **Account ID in Role**: The role's `accountId` will be the inviting account (Account A)
- **User Account ID in Role**: The role's `userAccountId` will be the invited account (Account B)

### Revocation

- Revoking access sets `isActive: false` on all related site roles
- The invitation history remains in `invitedTo` and `invitedAccounts` arrays
- Access can be restored by setting `isActive: true` on site roles (if invitation still exists)

### Multiple Invitations

- An account can be invited by multiple accounts
- An account can invite multiple accounts
- Each invitation is independent
- Site roles are scoped to the specific inviting account

## Frontend Implementation Tips

1. **Display Invitations:**
   - Show pending invitations in a notifications/invitations section
   - Display inviting account name, role, and sites
   - Show inviter's name for context

2. **Accept/Reject Flow:**
   - Provide clear accept/reject buttons
   - Show what sites will be accessible after acceptance
   - After acceptance, guide user to assign site roles to team members

3. **Site Role Assignment:**
   - After accepting, show a list of sites from the invitation
   - Allow assigning roles to users for each site
   - Show which users already have roles assigned

4. **Access Management:**
   - Show list of accounts you've invited
   - Show list of accounts that invited you
   - Provide revoke/cancel options where appropriate

## API Base URL

- **Production**: `https://api.maintor.systems`
- **Development**: Your Cloud Functions URL

## Authentication

All endpoints require authentication via Firebase ID token:

```
Authorization: IDTOKEN.<firebaseIdToken>
```

## Error Responses

Common error responses:

- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: User doesn't have required permissions
- **404 Not Found**: Account, invitation, or site not found
- **400 Bad Request**: Validation error (e.g., site doesn't belong to account)

## Support

For questions or issues:
- API documentation: `/openapi.json`
- Backend team for API-related questions
- Check existing guides in `/docs/` directory
