# Settings Migration Guide - Frontend Developer

## Overview

This guide will help you migrate settings from localStorage to the backend API. All account-level settings are now stored on the server and can be accessed via REST API endpoints.

## API Endpoints

### Base URL
- **Production**: `https://api.maintor.systems`
- **Development**: Your local API URL

### Endpoints

#### 1. Get Account Settings
```
GET /v1/accounts/{accountId}/settings
Authorization: IDTOKEN.<firebaseIdToken>
```

**Response (200 OK)**:
```json
{
  "id": "settings_123",
  "accountId": "account_456",
  "ticketStatuses": {
    "OPEN": true,
    "IN_PROGRESS": true,
    "COMPLETED": true,
    "CLOSED": true,
    "SKIPPED": true
  },
  "formSettings": {
    "breakdownFormDesktop": {
      "title": true,
      "priority": true,
      "status": true,
      "description": true,
      "breakdownEnded": true,
      "laborEntries": true,
      "rootCause": true,
      "problemDescription": true,
      "solutionDescription": true,
      "downtime": true,
      "photos": true,
      "assignees": true,
      "assetOwner": true,
      "reportedBy": true,
      "operatorNotes": true,
      "visualId": true,
      "barcode": true,
      "qrCode": true
    },
    "breakdownFormMobile": {
      "title": true,
      "priority": true,
      "status": true,
      "description": true,
      "breakdownEnded": true,
      "laborEntries": true,
      "rootCause": true,
      "problemDescription": true,
      "solutionDescription": true,
      "downtime": true,
      "photos": true,
      "assignees": true,
      "assetOwner": true,
      "reportedBy": true,
      "operatorNotes": true,
      "visualId": true,
      "barcode": true,
      "qrCode": true
    },
    "plannedMaintenanceForm": {
      "description": true,
      "generateTicketPerAsset": true,
      "isActive": true,
      "interval": true,
      "duration": true,
      "assignees": true
    },
    "dashboard": {
      "autoRefresh": false,
      "autoRefreshInterval": 5
    }
  },
  "created": "2025-01-01T00:00:00.000Z",
  "updated": "2025-01-01T00:00:00.000Z"
}
```

**Features**:
- Auto-creates default settings if they don't exist
- Returns settings immediately (no need to check if they exist first)
- All authenticated users with account access can view settings

#### 2. Update Account Settings
```
PATCH /v1/accounts/{accountId}/settings
Authorization: IDTOKEN.<firebaseIdToken>
Content-Type: application/json
```

**Request Body**:
```json
{
  "settings": {
    "ticketStatuses": {
      "SKIPPED": false
    },
    "formSettings": {
      "dashboard": {
        "autoRefresh": true,
        "autoRefreshInterval": 10
      }
    }
  }
}
```

**Response (200 OK)**: Returns the updated settings object (same format as GET)

**Features**:
- **Partial updates**: Only include fields you want to change
- **Deep merge**: Nested objects are merged, not replaced
- **Role-based access**: Requires OWNER, ADMIN, or SITE_MANAGER role
- Auto-creates default settings if they don't exist before updating

**Important**: The request body must be wrapped in a `settings` object.

## Settings Structure

### Ticket Statuses
Controls which ticket statuses are available/enabled:
- `OPEN`
- `IN_PROGRESS`
- `COMPLETED`
- `CLOSED`
- `SKIPPED`

### Form Settings

#### Breakdown Form (Desktop - maintor-app)
Controls visibility of fields in breakdown maintenance tickets for desktop:
- `title`, `priority`, `status`, `description`, `breakdownEnded`
- `laborEntries`, `rootCause`, `problemDescription`, `solutionDescription`
- `downtime`, `photos`, `assignees`, `assetOwner`, `reportedBy`, `operatorNotes`
- `visualId`, `barcode`, `qrCode`

#### Breakdown Form (Mobile - maintor-eng)
Controls visibility of fields in breakdown maintenance tickets for mobile:
- `title`, `priority`, `status`, `description`, `breakdownEnded`
- `laborEntries`, `rootCause`, `problemDescription`, `solutionDescription`
- `downtime`, `photos`, `assignees`, `assetOwner`, `reportedBy`, `operatorNotes`
- `visualId`, `barcode`, `qrCode`

**Note**: Desktop and mobile forms have separate visibility controls, allowing different field configurations for each platform.

#### Planned Maintenance Form
Controls visibility of fields in planned maintenance tickets:
- `description`, `generateTicketPerAsset`, `isActive`
- `interval`, `duration`, `assignees`

#### Dashboard
Dashboard behavior settings:
- `autoRefresh` (boolean): Enable automatic refresh
- `autoRefreshInterval` (number): Refresh interval in minutes (1-60)

## Implementation Examples

### JavaScript/Vue.js Example

```javascript
import { getAuth } from 'firebase/auth';

// Base API URL
const API_BASE_URL = 'https://api.maintor.systems'; // or your dev URL

/**
 * Get account settings from backend
 */
async function getAccountSettings(accountId) {
  try {
    const auth = getAuth();
    const user = auth.currentUser;
    
    if (!user) {
      throw new Error('User must be authenticated');
    }
    
    const idToken = await user.getIdToken();
    
    const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId}/settings`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `IDTOKEN.${idToken}`
      }
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Failed to get settings');
    }
    
    const settings = await response.json();
    return settings;
    
  } catch (error) {
    console.error('Error fetching settings:', error);
    throw error;
  }
}

/**
 * Update account settings (partial update)
 */
async function updateAccountSettings(accountId, updates) {
  try {
    const auth = getAuth();
    const user = auth.currentUser;
    
    if (!user) {
      throw new Error('User must be authenticated');
    }
    
    const idToken = await user.getIdToken();
    
    // Wrap updates in settings object
    const body = {
      settings: updates
    };
    
    const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId}/settings`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `IDTOKEN.${idToken}`
      },
      body: JSON.stringify(body)
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Failed to update settings');
    }
    
    const updatedSettings = await response.json();
    return updatedSettings;
    
  } catch (error) {
    console.error('Error updating settings:', error);
    throw error;
  }
}

// Usage examples:

// Get all settings
const settings = await getAccountSettings('account_123');

// Update ticket statuses
await updateAccountSettings('account_123', {
  ticketStatuses: {
    SKIPPED: false
  }
});

// Update dashboard settings
await updateAccountSettings('account_123', {
  formSettings: {
    dashboard: {
      autoRefresh: true,
      autoRefreshInterval: 10
    }
  }
});

// Update breakdown form field visibility (desktop)
await updateAccountSettings('account_123', {
  formSettings: {
    breakdownFormDesktop: {
      operatorNotes: false,
      downtime: false
    }
  }
});

// Update breakdown form field visibility (mobile)
await updateAccountSettings('account_123', {
  formSettings: {
    breakdownFormMobile: {
      visualId: true,
      barcode: true,
      qrCode: true
    }
  }
});
```

### React Hook Example

```javascript
import { useState, useEffect, useCallback } from 'react';
import { getAuth } from 'firebase/auth';

const API_BASE_URL = 'https://api.maintor.systems';

export function useAccountSettings(accountId) {
  const [settings, setSettings] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  // Fetch settings
  const fetchSettings = useCallback(async () => {
    if (!accountId) return;
    
    setLoading(true);
    setError(null);
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      const idToken = await user.getIdToken();
      
      const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId}/settings`, {
        headers: {
          'Authorization': `IDTOKEN.${idToken}`
        }
      });
      
      if (!response.ok) {
        throw new Error('Failed to fetch settings');
      }
      
      const data = await response.json();
      setSettings(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [accountId]);
  
  // Update settings
  const updateSettings = useCallback(async (updates) => {
    if (!accountId) return;
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      const idToken = await user.getIdToken();
      
      const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId}/settings`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `IDTOKEN.${idToken}`
        },
        body: JSON.stringify({
          settings: updates
        })
      });
      
      if (!response.ok) {
        throw new Error('Failed to update settings');
      }
      
      const updated = await response.json();
      setSettings(updated);
      return updated;
    } catch (err) {
      setError(err.message);
      throw err;
    }
  }, [accountId]);
  
  useEffect(() => {
    fetchSettings();
  }, [fetchSettings]);
  
  return {
    settings,
    loading,
    error,
    updateSettings,
    refetch: fetchSettings
  };
}

// Usage in component:
function SettingsPage({ accountId }) {
  const { settings, loading, error, updateSettings } = useAccountSettings(accountId);
  
  const handleToggleAutoRefresh = async () => {
    await updateSettings({
      formSettings: {
        dashboard: {
          autoRefresh: !settings.formSettings.dashboard.autoRefresh
        }
      }
    });
  };
  
  if (loading) return <div>Loading settings...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={settings.formSettings.dashboard.autoRefresh}
          onChange={handleToggleAutoRefresh}
        />
        Auto-refresh dashboard
      </label>
    </div>
  );
}
```

### Vue.js Composable Example

```javascript
import { ref, computed } from 'vue';
import { getAuth } from 'firebase/auth';

const API_BASE_URL = 'https://api.maintor.systems';

export function useAccountSettings(accountId) {
  const settings = ref(null);
  const loading = ref(false);
  const error = ref(null);
  
  const fetchSettings = async () => {
    if (!accountId.value) return;
    
    loading.value = true;
    error.value = null;
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      const idToken = await user.getIdToken();
      
      const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId.value}/settings`, {
        headers: {
          'Authorization': `IDTOKEN.${idToken}`
        }
      });
      
      if (!response.ok) {
        throw new Error('Failed to fetch settings');
      }
      
      settings.value = await response.json();
    } catch (err) {
      error.value = err.message;
    } finally {
      loading.value = false;
    }
  };
  
  const updateSettings = async (updates) => {
    if (!accountId.value) return;
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      const idToken = await user.getIdToken();
      
      const response = await fetch(`${API_BASE_URL}/v1/accounts/${accountId.value}/settings`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `IDTOKEN.${idToken}`
        },
        body: JSON.stringify({
          settings: updates
        })
      });
      
      if (!response.ok) {
        throw new Error('Failed to update settings');
      }
      
      settings.value = await response.json();
      return settings.value;
    } catch (err) {
      error.value = err.message;
      throw err;
    }
  };
  
  return {
    settings,
    loading,
    error,
    updateSettings,
    fetchSettings
  };
}
```

## Migration Strategy

### Step 1: Replace localStorage Reads

**Before (localStorage)**:
```javascript
// Old way
const settings = JSON.parse(localStorage.getItem('accountSettings') || '{}');
const autoRefresh = settings.dashboard?.autoRefresh ?? false;
```

**After (Backend API)**:
```javascript
// New way
const settings = await getAccountSettings(accountId);
const autoRefresh = settings.formSettings?.dashboard?.autoRefresh ?? false;
```

### Step 2: Replace localStorage Writes

**Before (localStorage)**:
```javascript
// Old way
const currentSettings = JSON.parse(localStorage.getItem('accountSettings') || '{}');
const updatedSettings = {
  ...currentSettings,
  dashboard: {
    ...currentSettings.dashboard,
    autoRefresh: true
  }
};
localStorage.setItem('accountSettings', JSON.stringify(updatedSettings));
```

**After (Backend API)**:
```javascript
// New way
await updateAccountSettings(accountId, {
  formSettings: {
    dashboard: {
      autoRefresh: true
    }
  }
});
```

### Step 3: Handle Initial Load

**Option A: Load on App Start**
```javascript
// In your app initialization
async function initializeApp(accountId) {
  try {
    const settings = await getAccountSettings(accountId);
    // Store in your state management (Vuex, Pinia, Redux, etc.)
    store.commit('setSettings', settings);
  } catch (error) {
    console.error('Failed to load settings:', error);
    // Fallback to defaults or show error
  }
}
```

**Option B: Load on Demand**
```javascript
// Load settings when needed
const settings = computed(() => {
  if (!store.state.settings) {
    // Trigger fetch
    getAccountSettings(accountId).then(settings => {
      store.commit('setSettings', settings);
    });
    return null; // or return defaults
  }
  return store.state.settings;
});
```

### Step 4: Migration Helper (One-Time)

If you have existing localStorage data, create a one-time migration function:

```javascript
async function migrateSettingsFromLocalStorage(accountId) {
  // Check if migration already done
  const migrationKey = `settings_migrated_${accountId}`;
  if (localStorage.getItem(migrationKey)) {
    return; // Already migrated
  }
  
  // Get old settings from localStorage
  const oldSettings = JSON.parse(localStorage.getItem('accountSettings') || '{}');
  
  if (Object.keys(oldSettings).length === 0) {
    // No old settings to migrate
    localStorage.setItem(migrationKey, 'true');
    return;
  }
  
  try {
    // Map old structure to new structure
    // If old settings had breakdownForm, apply to both desktop and mobile
    const oldBreakdownForm = oldSettings.breakdownForm || {};
    const newSettings = {
      ticketStatuses: oldSettings.ticketStatuses || {},
      formSettings: {
        breakdownFormDesktop: oldBreakdownForm,
        breakdownFormMobile: oldBreakdownForm, // Copy same settings to mobile initially
        plannedMaintenanceForm: oldSettings.plannedMaintenanceForm || {},
        dashboard: oldSettings.dashboard || {
          autoRefresh: false,
          autoRefreshInterval: 5
        }
      }
    };
    
    // Upload to backend
    await updateAccountSettings(accountId, newSettings);
    
    // Mark as migrated
    localStorage.setItem(migrationKey, 'true');
    
    // Optional: Clear old localStorage data after successful migration
    // localStorage.removeItem('accountSettings');
    
    console.log('Settings migrated successfully');
  } catch (error) {
    console.error('Failed to migrate settings:', error);
    // Don't mark as migrated, so it can retry
  }
}
```

## Error Handling

### Common Errors

**401 Unauthorized**:
```javascript
if (response.status === 401) {
  // User not authenticated - redirect to login
  router.push('/login');
}
```

**403 Forbidden**:
```javascript
if (response.status === 403) {
  // User doesn't have permission
  // Show error message or disable settings UI
  showError('You do not have permission to modify settings');
}
```

**404 Not Found**:
```javascript
if (response.status === 404) {
  // Account not found
  // This shouldn't happen if accountId is correct
  showError('Account not found');
}
```

**500 Internal Server Error**:
```javascript
if (response.status === 500) {
  // Server error - retry or show error
  showError('Server error. Please try again later.');
}
```

### Retry Logic

```javascript
async function getAccountSettingsWithRetry(accountId, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await getAccountSettings(accountId);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      // Wait before retry (exponential backoff)
      await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 1000));
    }
  }
}
```

## Best Practices

### 1. Cache Settings Locally (Optional)

You can still cache settings in memory/state for performance, but don't persist to localStorage:

```javascript
// Good: Cache in memory/state
const [settings, setSettings] = useState(null);

// Bad: Don't use localStorage anymore
// localStorage.setItem('settings', JSON.stringify(settings));
```

### 2. Debounce Updates

If users can change settings rapidly, debounce the API calls:

```javascript
import { debounce } from 'lodash';

const debouncedUpdate = debounce(async (accountId, updates) => {
  await updateAccountSettings(accountId, updates);
}, 500);

// Usage
debouncedUpdate(accountId, { formSettings: { dashboard: { autoRefresh: true } } });
```

### 3. Optimistic Updates

Update UI immediately, then sync with backend:

```javascript
async function updateSettingsOptimistic(accountId, updates) {
  // Update local state immediately
  const optimisticSettings = { ...currentSettings, ...updates };
  setSettings(optimisticSettings);
  
  try {
    // Sync with backend
    const actualSettings = await updateAccountSettings(accountId, updates);
    setSettings(actualSettings);
  } catch (error) {
    // Revert on error
    setSettings(currentSettings);
    showError('Failed to save settings');
  }
}
```

### 4. Handle Offline Mode

```javascript
async function getAccountSettingsWithFallback(accountId) {
  try {
    return await getAccountSettings(accountId);
  } catch (error) {
    // If offline, use cached version or defaults
    const cached = sessionStorage.getItem(`settings_cache_${accountId}`);
    if (cached) {
      return JSON.parse(cached);
    }
    // Return defaults
    return getDefaultSettings();
  }
}
```

### 5. Settings Structure Differences

**Important**: Note the structure differences:

- **Old localStorage**: `settings.dashboard.autoRefresh`
- **New API**: `settings.formSettings.dashboard.autoRefresh`

- **Old localStorage**: `settings.breakdownForm.priority`
- **New API**: 
  - Desktop: `settings.formSettings.breakdownFormDesktop.priority`
  - Mobile: `settings.formSettings.breakdownFormMobile.priority`

Make sure to update all references to use the new structure. The breakdown form settings are now split into separate desktop and mobile configurations.

## Testing

### Test Cases

1. **Get settings for new account** (should return defaults)
2. **Update partial settings** (should merge, not replace)
3. **Update nested settings** (should deep merge)
4. **Handle authentication errors**
5. **Handle permission errors** (403)
6. **Handle network errors**

### Example Test

```javascript
describe('Account Settings', () => {
  it('should fetch default settings for new account', async () => {
    const settings = await getAccountSettings('new_account_id');
    expect(settings.ticketStatuses).toBeDefined();
    expect(settings.formSettings).toBeDefined();
  });
  
  it('should update settings partially', async () => {
    await updateAccountSettings('account_id', {
      formSettings: {
        dashboard: { autoRefresh: true }
      }
    });
    
    const updated = await getAccountSettings('account_id');
    expect(updated.formSettings.dashboard.autoRefresh).toBe(true);
    // Other settings should remain unchanged
    expect(updated.ticketStatuses).toBeDefined();
  });
});
```

## Checklist

- [ ] Replace all `localStorage.getItem('accountSettings')` calls with API calls
- [ ] Replace all `localStorage.setItem('accountSettings', ...)` calls with API calls
- [ ] Update all references to use new structure (`formSettings.dashboard` instead of `dashboard`)
- [ ] Update breakdown form references: use `breakdownFormDesktop` for desktop (maintor-app) and `breakdownFormMobile` for mobile (maintor-eng)
- [ ] Add error handling for API calls
- [ ] Test with new accounts (should get defaults with both desktop and mobile breakdown forms)
- [ ] Test partial updates (updating only desktop or only mobile breakdown form)
- [ ] Test permission errors (non-admin users)
- [ ] Remove localStorage migration code after migration is complete
- [ ] Update any documentation that references localStorage settings

## Support

If you encounter issues:
1. Check the API response in browser DevTools Network tab
2. Verify the accountId is correct
3. Verify the user has the required permissions
4. Check server logs for detailed error messages

For API documentation, see: `openapi.json` (search for `/v1/accounts/{accountId}/settings`)

