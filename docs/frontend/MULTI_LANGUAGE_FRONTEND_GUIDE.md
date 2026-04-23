# Frontend Developer Guide - Multi-Language Support

## Overview

The application needs to support multiple languages: **English (en)**, **Hebrew (he)**, **Arabic (ar)**, and **Russian (ru)**. All translations are handled on the frontend. The backend API stores language preferences but does not translate messages - it returns error codes/keys that your frontend translates.

## Language Preference Priority

Language preference follows this hierarchy (highest to lowest priority):

1. **User-level language preference** (individual user setting)
2. **Account-level language preference** (organization default)
3. **Default: English (en)** (if neither is set)

**Example:** If an organization has `language: 'en'` set, but a user has `language: 'ru'` set, that user will see the app in Russian, while other users see English.

## Backend API Changes

### User Language Preference

The backend will support storing language preference at the user level:

**Update User Endpoint:**
```
PATCH /v1/accounts/{accountId}/users/{userId}
```

**Request Body:**
```json
{
  "language": "ru"  // Optional: 'en', 'he', 'ar', or 'ru'
}
```

**User Schema (GET response):**
```json
{
  "id": "user123",
  "language": "ru",  // Optional field, null if not set
  // ... other user fields
}
```

### Account Language Preference

The backend will support storing language preference at the account level:

**Update Account Settings Endpoint:**
```
PATCH /v1/accounts/{accountId}/settings
```

**Request Body:**
```json
{
  "settings": {
    "language": "he"  // Optional: 'en', 'he', 'ar', or 'ru'
  }
}
```

**Account Settings Schema (GET response):**
```json
{
  "id": "settings123",
  "accountId": "account456",
  "language": "he",  // Optional field, defaults to 'en'
  // ... other settings
}
```

## Frontend Implementation Requirements

### 1. Translation System Setup

- Use an i18n library (e.g., `react-i18next`, `vue-i18next`, `i18next`)
- Create translation files for each language:
  - `locales/en/translation.json`
  - `locales/he/translation.json`
  - `locales/ar/translation.json`
  - `locales/ru/translation.json`

### 2. Language Detection Logic

**On App Load:**
1. Get current user data (from authenticated session)
2. Check if user has `language` field set
3. If not set, fetch account settings and check `language` field
4. If neither is set, default to `'en'`
5. Initialize i18n with detected language

**Example Flow:**
```javascript
async function detectLanguage(userId, accountId) {
  // 1. Get user data
  const user = await getUser(userId, accountId);
  if (user.language) {
    return user.language; // User preference
  }
  
  // 2. Get account settings
  const settings = await getAccountSettings(accountId);
  if (settings?.language) {
    return settings.language; // Account preference
  }
  
  // 3. Default
  return 'en';
}
```

### 3. Language Preference Management

**User Language Setting:**
- Allow users to change their language preference in user settings/profile
- Update via: `PATCH /v1/accounts/{accountId}/users/{userId}` with `{ "language": "he" }`
- Immediately apply new language to UI after update

**Account Language Setting (Admin/Owner only):**
- Allow account admins/owners to set organization default language
- Update via: `PATCH /v1/accounts/{accountId}/settings` with `{ "settings": { "language": "he" } }`
- This becomes the default for users who don't have a personal language preference

### 4. Error Message Translation

The backend API returns error codes/keys, not translated messages. Your frontend must translate them.

**Backend Error Response Format:**
```json
{
  "error": "FORBIDDEN",
  "code": "PERMISSION_DENIED",
  "message": "You do not have permission to update account details. Requires OWNER or ADMIN role.",
  "details": { ... }
}
```

**Frontend Translation Mapping:**
```json
// locales/en/errors.json
{
  "FORBIDDEN": "You do not have permission",
  "PERMISSION_DENIED": "You do not have permission to perform this action",
  "VALIDATION_FAILED": "Validation failed",
  "RESOURCE_NOT_FOUND": "Resource not found"
}

// locales/he/errors.json
{
  "FORBIDDEN": "אין לך הרשאה",
  "PERMISSION_DENIED": "אין לך הרשאה לבצע פעולה זו",
  "VALIDATION_FAILED": "אימות נכשל",
  "RESOURCE_NOT_FOUND": "המשאב לא נמצא"
}
```

**Error Handling Example:**
```javascript
function handleApiError(errorResponse) {
  const errorCode = errorResponse.code || errorResponse.error;
  const translatedMessage = t(`errors.${errorCode}`, { 
    defaultValue: errorResponse.message // Fallback to API message
  });
  showError(translatedMessage);
}
```

### 5. RTL (Right-to-Left) Support

Hebrew and Arabic are RTL languages. Ensure your UI framework supports RTL:

- **React**: Use `dir="rtl"` attribute and CSS logical properties
- **Vue**: Use `dir` binding and RTL-aware CSS
- **CSS**: Use `direction: rtl` and logical properties (`margin-inline-start` instead of `margin-left`)

**Language-to-Direction Mapping:**
- `he` (Hebrew) → RTL
- `ar` (Arabic) → RTL
- `en` (English) → LTR
- `ru` (Russian) → LTR

### 6. Date/Time Formatting

Consider locale-specific date/time formatting:
- Use libraries like `date-fns` with locale support or `Intl.DateTimeFormat`
- Format dates according to user's language preference

### 7. Number Formatting

Consider locale-specific number formatting:
- Use `Intl.NumberFormat` for currency, percentages, etc.
- Format numbers according to user's language preference

## Translation Keys Structure

Organize your translation files logically:

```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit"
  },
  "errors": {
    "FORBIDDEN": "You do not have permission",
    "VALIDATION_FAILED": "Validation failed",
    "RESOURCE_NOT_FOUND": "Resource not found"
  },
  "validation": {
    "required": "This field is required",
    "invalid_email": "Invalid email address",
    "min_length": "Must be at least {{min}} characters"
  },
  "tickets": {
    "status": {
      "OPEN": "Open",
      "IN_PROGRESS": "In Progress",
      "COMPLETED": "Completed"
    }
  }
}
```

## Implementation Checklist

- [ ] Set up i18n library (react-i18next, vue-i18next, etc.)
- [ ] Create translation files for en, he, ar, ru
- [ ] Implement language detection logic (user → account → default)
- [ ] Add language selector to user settings/profile
- [ ] Add language selector to account settings (admin/owner only)
- [ ] Implement API calls to update user/account language preference
- [ ] Map backend error codes to translated messages
- [ ] Implement RTL support for Hebrew and Arabic
- [ ] Test language switching and persistence
- [ ] Test error message translations
- [ ] Test RTL layout for Hebrew and Arabic
- [ ] Consider date/time formatting per locale
- [ ] Consider number formatting per locale

## API Endpoints Reference

### Get User (includes language field)
```
GET /v1/accounts/{accountId}/users/{userId}
```

### Update User Language
```
PATCH /v1/accounts/{accountId}/users/{userId}
Body: { "language": "he" }
```

### Get Account Settings (includes language field)
```
GET /v1/accounts/{accountId}/settings
```

### Update Account Language
```
PATCH /v1/accounts/{accountId}/settings
Body: { "settings": { "language": "he" } }
```

## Language Codes

- `en` - English
- `he` - Hebrew
- `ar` - Arabic
- `ru` - Russian

## Notes

- Language preference is stored in the backend but translations are handled entirely by the frontend
- Backend API responses will continue to include English messages as fallback, but frontend should use translations
- When a user changes their language preference, immediately update the UI without requiring a page reload
- Account-level language changes affect all users who don't have a personal language preference set
- Consider caching language preference in localStorage/sessionStorage for performance

## Support

For questions or issues:
- API documentation: `/openapi.json`
- Backend team for API-related questions
- Check existing frontend guides in `/docs/` directory
