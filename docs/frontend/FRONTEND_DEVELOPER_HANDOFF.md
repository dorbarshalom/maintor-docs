# Frontend Developer Handoff - User Invitation Flow

## Quick Start

Your frontend needs to implement the user invitation acceptance flow. Users receive an email with a JWT token in the URL, and your app needs to:

1. Extract token from URL (`?InviteToken=...`)
2. Validate token with API
3. Show invitation details
4. User signs up with Firebase Auth
5. Accept invitation with Firebase ID token

## Essential Information

### API Base URL
- **Production**: `https://api.maintor.systems`
- **Development**: Your Cloud Functions URL (if using GCP)

### Invitation Email Link Format
```
https://yourapp.com/signup?InviteToken=<JWT_TOKEN>
```

The token is passed as a **query parameter** named `InviteToken` (not `token`).

### Two API Endpoints You Need

#### 1. Validate Token (POST)
```
POST /v1/public/user-invitations/validate
Content-Type: application/json

Body: { "token": "<JWT>" }
```

Returns invitation details (user email, account name, roles).

#### 2. Accept Invitation (POST)
```
POST /v1/public/user-invitations/accept
Content-Type: application/json
Authorization: IDTOKEN.<firebaseIdToken>

Body: { "token": "<JWT>" }
```

**Critical**: Requires Firebase ID token in Authorization header with `IDTOKEN.` prefix.

### Key Requirements

1. **Email Matching**: Firebase Auth email **must** match invitation email (API enforces this)
2. **Token Format**: JWT token from URL query parameter `InviteToken`
3. **Firebase ID Token**: Required for accept endpoint (get via `firebaseUser.getIdToken()`)
4. **Token Expiry**: Tokens expire after 7 days (default)

## Complete Documentation

See the detailed guide: **`docs/user-invitation-frontend-guide.md`**

This includes:
- Complete API reference
- Step-by-step implementation guide
- Code examples
- Error handling
- Testing scenarios
- UI/UX recommendations

## Quick Implementation Checklist

- [ ] Extract `InviteToken` from URL query parameters
- [ ] Call validate endpoint to get invitation details
- [ ] Display invitation information (account name, email, roles)
- [ ] Pre-fill email field (read-only recommended)
- [ ] Implement Firebase Auth signup (email must match invitation)
- [ ] Get Firebase ID token after signup
- [ ] Call accept endpoint with token and Firebase ID token
- [ ] Handle errors (invalid token, email mismatch, etc.)
- [ ] Redirect to dashboard after successful acceptance

## Common Issues

1. **Email Mismatch Error**: Ensure Firebase Auth uses the same email as the invitation
2. **Invalid Token**: Token may be expired (7 days) or already used
3. **Missing Authorization Header**: Must include `IDTOKEN.<firebaseIdToken>` for accept endpoint
4. **Wrong Query Parameter**: Use `InviteToken` (capital I, capital T), not `token` or `inviteToken`

## Support

For questions or issues, refer to:
- Detailed guide: `docs/user-invitation-frontend-guide.md`
- API documentation: `/openapi.json`
- Backend team for API-related questions

