# User Invitation Flow - Frontend Implementation Guide

## Overview

Users are invited to join an account via email. The invitation email contains a JWT token in the URL. The backend shifts to a **Direct User Invitation model**, which means no database record or user profile is pre-created in the database when an invitation is generated.

The frontend flow is:

1. User clicks invitation link → lands on signup page with JWT token in URL
2. Frontend validates token (`POST /v1/public/user-invitations/validate`) and shows invitation details
3. User signs up via Firebase Auth (email must match invitation email)
4. Frontend accepts invitation (`POST /v1/public/user-invitations/accept`) with Firebase ID token
5. The API handles account association:
   - **If unregistered user**: Creates a brand-new user profile (status `ACTIVE`), lists account membership, and maps site roles.
   - **If pre-registered user**: Appends the new account ID to the user's `accounts` membership list and creates roles under the new account scope.

## API Endpoints

### Public Endpoints (No Authentication Required)

#### 1. Validate Invitation Token
```
POST /v1/public/user-invitations/validate
Content-Type: application/json

Request Body:
{
  "token": "<JWT_TOKEN>"
}
```

**Purpose**: Validate JWT token and get invitation details before signup. Since user documents are not pre-created, it parses details directly from the JWT.

**Response** (200 OK):
```json
{
  "valid": true,
  "user": {
    "id": null, // null if unregistered, otherwise existing user ID
    "email": "user@example.com",
    "firstName": "Jane",
    "lastName": "Smith"
  },
  "account": {
    "id": "account456",
    "name": "Acme Corp"
  },
  "roles": [
    {
      "role": "TECHNICIAN",
      "scope": "site",
      "siteId": "site_123"
    }
  ]
}
```

**Error Responses**:
- `400`: Invalid or expired token, token missing, or user is already a member of the account.
- `500`: Internal server error.

#### 2. Accept Invitation
```
POST /v1/public/user-invitations/accept
Content-Type: application/json
Authorization: IDTOKEN.<firebaseIdToken>

Request Body:
{
  "token": "<JWT_TOKEN>"
}
```

**Purpose**: Accept invitation and activate/append user account membership.

**Important**: 
- Requires Firebase ID token in Authorization header with `IDTOKEN.` prefix
- Firebase Auth email must match the invitation email

**Response** (200 OK):
```json
{
  "message": "Invitation accepted successfully",
  "user": {
    "id": "user123",
    "email": "user@example.com",
    "status": "ACTIVE",
    "authUserId": "firebase_user_id_123",
    "accounts": [
      {
        "accountId": "account456",
        "invitedAt": "2026-06-09T23:00:00Z"
      }
    ],
    "accountIds": ["account456"]
  }
}
```

**Error Responses**:
- `400`: Token missing, invalid, or expired, or user is already a member
- `401`: Firebase ID token missing or invalid
- `400`: Email mismatch (Firebase Auth email doesn't match invitation email)

## Implementation Steps

### Step 1: Extract Token from URL

The invitation email contains a link like:
```
https://yourapp.com/signup?InviteToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Extract the token from the URL:

```javascript
// Extract token from URL query parameter
const urlParams = new URLSearchParams(window.location.search);
const invitationToken = urlParams.get('InviteToken');

if (!invitationToken) {
  // No invitation token - show regular signup form
  // User can sign up normally without invitation
}
```

### Step 2: Validate Token and Get Invitation Details

On signup page load (when `InviteToken` is detected):

```javascript
async function validateInvitationToken(token) {
  try {
    const response = await fetch(
      `${API_BASE_URL}/v1/public/user-invitations/validate`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ token })
      }
    );

    if (response.status === 400 || response.status === 404) {
      const error = await response.json();
      return { 
        valid: false, 
        error: error.error || 'Invalid or expired invitation link' 
      };
    }

    if (!response.ok) {
      throw new Error('Failed to validate invitation');
    }

    const data = await response.json();
    return { valid: true, data };
  } catch (error) {
    console.error('Error validating invitation:', error);
    return { valid: false, error: 'Failed to validate invitation' };
  }
}

// Usage
const { valid, data, error } = await validateInvitationToken(invitationToken);

if (valid) {
  // data.user contains: id, email, firstName, lastName
  // data.account contains: id, name
  // data.roles contains: array of roles assigned to user
} else {
  // Show error message: error
  // Optionally redirect to regular signup
}
```

### Step 3: Display Invitation Information

Show invitation details to the user:

```javascript
if (valid) {
  const { user, account, roles } = data;
  
  // Display invitation message
  const message = `You've been invited to join ${account.name}.`;
  
  // Pre-fill email field (read-only recommended)
  setEmailField(user.email);
  
  // Show account name and any role information
  showInvitationDetails({
    accountName: account.name,
    email: user.email,
    roles: roles
  });
}
```

### Step 4: User Signs Up via Firebase Auth

Proceed with normal Firebase Auth signup flow:

```javascript
import { createUserWithEmailAndPassword, signInWithEmailAndPassword } from 'firebase/auth';

async function signUpWithFirebase(email, password) {
  try {
    const userCredential = await createUserWithEmailAndPassword(auth, email, password);
    return userCredential.user;
  } catch (error) {
    // Handle Firebase Auth errors
    throw error;
  }
}

// Important: Use the email from the invitation
const firebaseUser = await signUpWithFirebase(
  invitationData.user.email, // Must match invitation email
  password
);

// Get Firebase ID token
const idToken = await firebaseUser.getIdToken();
```

**Critical**: The email used for Firebase Auth **must match** the email from the invitation. The API will verify this and return an error if they don't match.

### Step 5: Accept Invitation After Firebase Auth

After successful Firebase Auth signup, call the accept endpoint:

```javascript
async function acceptInvitation(token, firebaseIdToken) {
  try {
    const response = await fetch(
      `${API_BASE_URL}/v1/public/user-invitations/accept`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `IDTOKEN.${firebaseIdToken}`
        },
        body: JSON.stringify({ token })
      }
    );

    if (response.status === 400) {
      const error = await response.json();
      return { 
        success: false, 
        error: error.error || error.message || 'Failed to accept invitation' 
      };
    }

    if (response.status === 401) {
      return { 
        success: false, 
        error: 'Authentication failed. Please try signing up again.' 
      };
    }

    if (!response.ok) {
      throw new Error('Failed to accept invitation');
    }

    const result = await response.json();
    return { success: true, user: result.user };
  } catch (error) {
    console.error('Error accepting invitation:', error);
    return { success: false, error: 'Failed to accept invitation' };
  }
}

// Usage after Firebase Auth
const idToken = await firebaseUser.getIdToken();
const result = await acceptInvitation(invitationToken, idToken);

if (result.success) {
  // User account activated successfully
  // Store user data: result.user
  // Redirect to dashboard or onboarding
  navigate('/dashboard');
} else {
  // Handle error
  showError(result.error);
}
```

## Complete Flow Example

```javascript
// 1. Extract token from URL
const urlParams = new URLSearchParams(window.location.search);
const invitationToken = urlParams.get('InviteToken');

if (invitationToken) {
  // 2. Validate invitation token
  const validation = await validateInvitationToken(invitationToken);
  
  if (validation.valid) {
    const { user, account, roles } = validation.data;
    
    // 3. Show invitation details in UI
    showInvitationMessage({
      accountName: account.name,
      email: user.email,
      roles: roles
    });
    
    // 4. Pre-fill email (read-only)
    setEmailField(user.email);
    
    // 5. Handle signup form submission
    async function handleSignup(formData) {
      try {
        // Sign up with Firebase Auth (email must match invitation)
        const firebaseUser = await signUpWithFirebase(
          user.email, // Use email from invitation
          formData.password
        );
        
        // Get Firebase ID token
        const idToken = await firebaseUser.getIdToken();
        
        // Accept invitation
        const result = await acceptInvitation(invitationToken, idToken);
        
        if (result.success) {
          // Success! User account activated
          // Store user data
          setUserData(result.user);
          
          // Redirect to app
          navigate('/dashboard');
        } else {
          // Handle error
          showError(result.error);
        }
      } catch (error) {
        // Handle Firebase Auth errors
        if (error.code === 'auth/email-already-in-use') {
          // User already exists - try signing in instead
          const firebaseUser = await signInWithEmailAndPassword(
            auth, 
            user.email, 
            formData.password
          );
          const idToken = await firebaseUser.getIdToken();
          const result = await acceptInvitation(invitationToken, idToken);
          // ... handle result
        } else {
          showError(error.message);
        }
      }
    }
  } else {
    // Invalid token - show error
    showError(validation.error || 'Invalid invitation link');
    // Optionally redirect to regular signup after delay
  }
} else {
  // No token - regular signup flow
  // Show normal signup form
}
```

## Error Handling

| Error | Status | Action |
|-------|--------|--------|
| Token missing from URL | - | Show regular signup form |
| Invalid/expired token | 400 | Show "Invalid or expired invitation link" message |
| User already accepted | 400 | Show "Invitation already accepted" message |
| Email mismatch | 400 | Show "Email mismatch" error (Firebase email ≠ invitation email) |
| Firebase Auth error | - | Handle Firebase-specific errors (email already in use, etc.) |
| Network error | - | Show retry option |

## Important Notes

1. **Token Format**: The token in the URL is a JWT, passed as query parameter `InviteToken`
2. **Email Matching**: Firebase Auth email **must** match the invitation email - API enforces this
3. **Token Expiry**: JWT tokens expire (default: 7 days, configurable via `INVITATION_JWT_EXPIRY_DAYS`)
4. **Single Use**: Once accepted, user status changes from PENDING to ACTIVE - cannot accept again
5. **Firebase ID Token**: Required in Authorization header with `IDTOKEN.` prefix for accept endpoint
6. **Public Endpoints**: No authentication required for validate/accept endpoints (except Firebase ID token for accept)
7. **Roles**: Initial roles are assigned during user creation and shown in validation response

## UI/UX Recommendations

1. **Token Validation**: Show loading state while validating token
2. **Invitation Details**: Display account name and role information prominently
3. **Email Pre-fill**: Pre-fill email from invitation (read-only recommended to prevent mismatch)
4. **Error States**: 
   - Clear error messages for invalid/expired tokens
   - Handle email mismatch gracefully
   - Show helpful messages for Firebase Auth errors
5. **Success**: After acceptance, redirect to dashboard or onboarding flow
6. **Edge Cases**: 
   - If user already has Firebase account, show sign-in option
   - If invitation already accepted, show appropriate message

## Testing Scenarios

Test the following scenarios:
1. ✅ Valid token → Should show invitation details
2. ✅ Invalid token → Should show error
3. ✅ Expired token → Should show error
4. ✅ Missing token → Should show regular signup
5. ✅ Accept invitation → Should activate user and return user object
6. ✅ Email mismatch → Should show error
7. ✅ Already accepted → Should show error
8. ✅ Firebase Auth errors → Should handle gracefully
9. ✅ Network errors → Should show retry option

## API Base URL

Use your API base URL:
- Production: `https://api.maintor.systems`
- Or your Cloud Functions URL if using GCP

## Example Invitation Email Link

The invitation email will contain a link like:
```
https://yourapp.com/signup?InviteToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhY2NvdW50SWQiOiJhY2NvdW50MTIzIiwidXNlcklkIjoidXNlcjQ1NiIsImVtYWlsIjoidXNlckBleGFtcGxlLmNvbSIsInR5cGUiOiJ1c2VyX2ludml0YXRpb24iLCJpYXQiOjE3MDAwMDAwMDAsImV4cCI6MTcwMDYwNDgwMH0.signature
```

The frontend should extract `InviteToken` from the query parameters.

---

For detailed API schemas and examples, refer to the OpenAPI specification at `/openapi.json`.
