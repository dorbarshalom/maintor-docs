# WhatsApp Notification - Frontend Implementation Guide

## Overview

The WhatsApp notification endpoint allows you to send WhatsApp messages from the frontend. Currently, messages are sent to a hardcoded recipient phone number for testing purposes. The endpoint requires authentication via Firebase Auth.

## API Endpoint

### Send WhatsApp Notification

```
POST /v1/whatsapp/notify
Content-Type: application/json
Authorization: Bearer <firebaseIdToken>
```

**Purpose**: Send a WhatsApp notification message

**Authentication**: Required - Firebase ID token in Authorization header

**Request Body** (optional):
```json
{
  "message": "Your custom message text"
}
```

**Note**: If `message` is not provided, a default test message will be sent.

## Response Format

### Success Response (200 OK)

```json
{
  "success": true,
  "message": "WhatsApp message sent successfully",
  "messageId": "wamid.xxx",
  "recipient": "+972-54-6800360"
}
```

### Error Responses

**400 Bad Request** - Invalid JSON:
```json
{
  "error": "Invalid JSON",
  "message": "Request body must be valid JSON"
}
```

**401 Unauthorized** - Missing or invalid authentication:
- Missing Authorization header
- Invalid Firebase ID token

**500 Internal Server Error** - Configuration or API error:
```json
{
  "error": "Configuration error",
  "message": "WASENDER_API_KEY environment variable is not set"
}
```

Or:
```json
{
  "error": "WhatsApp API error",
  "message": "WhatsApp API error: 400 Bad Request - {...}"
}
```

## Implementation

### Step 1: Get Firebase ID Token

First, ensure you have a Firebase ID token from your authenticated user:

```javascript
import { getAuth } from 'firebase/auth';

// Get current user
const auth = getAuth();
const user = auth.currentUser;

if (!user) {
  // User not authenticated
  throw new Error('User must be authenticated');
}

// Get ID token
const idToken = await user.getIdToken();
```

### Step 2: Send WhatsApp Message

```javascript
async function sendWhatsAppNotification(message) {
  try {
    // Get Firebase ID token
    const auth = getAuth();
    const user = auth.currentUser;
    
    if (!user) {
      throw new Error('User must be authenticated');
    }
    
    const idToken = await user.getIdToken();
    
    // Prepare request
    const API_BASE_URL = 'https://api.maintor.systems'; // or your API URL
    const response = await fetch(`${API_BASE_URL}/v1/whatsapp/notify`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${idToken}`
      },
      body: JSON.stringify({
        message: message || undefined // Optional - omit if you want default message
      })
    });
    
    // Handle response
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || error.error || 'Failed to send WhatsApp message');
    }
    
    const result = await response.json();
    return {
      success: true,
      messageId: result.messageId,
      recipient: result.recipient
    };
  } catch (error) {
    console.error('Error sending WhatsApp notification:', error);
    throw error;
  }
}
```

### Step 3: Usage Example

```javascript
// Example 1: Send custom message
try {
  const result = await sendWhatsAppNotification('Hello from Maintor!');
  console.log('Message sent successfully:', result.messageId);
  // Show success notification to user
} catch (error) {
  console.error('Failed to send message:', error.message);
  // Show error notification to user
}

// Example 2: Send default message (omit message parameter)
try {
  const result = await sendWhatsAppNotification();
  console.log('Default message sent:', result.messageId);
} catch (error) {
  console.error('Failed to send message:', error.message);
}
```

## Complete Example with Error Handling

```javascript
import { getAuth } from 'firebase/auth';

const API_BASE_URL = 'https://api.maintor.systems'; // or your API URL

async function sendWhatsAppNotification(message) {
  const auth = getAuth();
  const user = auth.currentUser;
  
  // Check authentication
  if (!user) {
    return {
      success: false,
      error: 'User must be authenticated to send WhatsApp messages'
    };
  }
  
  try {
    // Get Firebase ID token
    const idToken = await user.getIdToken();
    
    // Send request
    const response = await fetch(`${API_BASE_URL}/v1/whatsapp/notify`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${idToken}`
      },
      body: JSON.stringify(message ? { message } : {})
    });
    
    // Parse response
    const data = await response.json();
    
    // Handle different status codes
    if (response.status === 200) {
      return {
        success: true,
        messageId: data.messageId,
        recipient: data.recipient,
        message: data.message
      };
    } else if (response.status === 400) {
      return {
        success: false,
        error: data.message || 'Invalid request'
      };
    } else if (response.status === 401) {
      return {
        success: false,
        error: 'Authentication failed. Please log in again.'
      };
    } else if (response.status === 500) {
      return {
        success: false,
        error: data.message || 'Server error. Please try again later.'
      };
    } else {
      return {
        success: false,
        error: 'Unexpected error occurred'
      };
    }
  } catch (error) {
    console.error('Error sending WhatsApp notification:', error);
    return {
      success: false,
      error: error.message || 'Network error. Please check your connection.'
    };
  }
}

// Usage in your component
async function handleSendWhatsApp() {
  const message = 'Test message from frontend';
  const result = await sendWhatsAppNotification(message);
  
  if (result.success) {
    // Show success message
    showNotification('WhatsApp message sent successfully!', 'success');
  } else {
    // Show error message
    showNotification(result.error, 'error');
  }
}
```

## React/Vue Example

### React Hook Example

```javascript
import { useState } from 'react';
import { getAuth } from 'firebase/auth';

function useWhatsAppNotification() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const sendNotification = async (message) => {
    setLoading(true);
    setError(null);
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      
      if (!user) {
        throw new Error('User must be authenticated');
      }
      
      const idToken = await user.getIdToken();
      const API_BASE_URL = 'https://api.maintor.systems';
      
      const response = await fetch(`${API_BASE_URL}/v1/whatsapp/notify`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${idToken}`
        },
        body: JSON.stringify(message ? { message } : {})
      });
      
      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || errorData.error || 'Failed to send message');
      }
      
      const result = await response.json();
      return result;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };
  
  return { sendNotification, loading, error };
}

// Usage in component
function MyComponent() {
  const { sendNotification, loading, error } = useWhatsAppNotification();
  
  const handleClick = async () => {
    try {
      const result = await sendNotification('Hello from React!');
      console.log('Message sent:', result.messageId);
    } catch (err) {
      console.error('Error:', err);
    }
  };
  
  return (
    <button onClick={handleClick} disabled={loading}>
      {loading ? 'Sending...' : 'Send WhatsApp Message'}
    </button>
  );
}
```

## Important Notes

1. **Authentication Required**: The endpoint requires a valid Firebase ID token in the Authorization header
2. **Recipient**: Currently hardcoded to `+972-54-6800360` for testing purposes
3. **Message Optional**: If no message is provided, a default test message will be sent
4. **Rate Limiting**: Be aware of WhatsApp API rate limits - don't send too many messages in quick succession
5. **Error Handling**: Always handle errors gracefully and show appropriate messages to users
6. **Testing**: Use this endpoint for testing WhatsApp integration. Production rules will be added later

## Error Handling Checklist

- [ ] Check if user is authenticated before calling the endpoint
- [ ] Handle 400 errors (invalid request)
- [ ] Handle 401 errors (authentication failed)
- [ ] Handle 500 errors (server/configuration issues)
- [ ] Handle network errors (connection issues)
- [ ] Show user-friendly error messages
- [ ] Log errors for debugging

## Testing

1. **Test with custom message**:
   ```javascript
   await sendWhatsAppNotification('Test message from frontend');
   ```

2. **Test with default message**:
   ```javascript
   await sendWhatsAppNotification();
   ```

3. **Test error handling**:
   - Try without authentication
   - Try with invalid token
   - Test network errors

## API Base URL

Use your API base URL:
- **Production**: `https://api.maintor.systems`
- **Development**: Your Cloud Functions URL (if using GCP)
- **Local**: `http://localhost:8080` (if running locally)

## Future Enhancements

The following features will be added in the future:
- Configurable recipient phone numbers
- Scheduled notifications
- Automatic notifications for overdue tickets
- Notification rules engine
- Message templates
- Delivery status tracking

---

For detailed API schemas and examples, refer to the OpenAPI specification at `/openapi.json`.

