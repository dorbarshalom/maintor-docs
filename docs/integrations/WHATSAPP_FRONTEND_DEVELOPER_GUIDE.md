# WhatsApp Template Messages - Frontend Developer Guide

## Overview

The WhatsApp notification API allows you to send WhatsApp messages using Meta-approved message templates. The API supports sending to any phone number (provided in the request) or uses a default recipient if not specified.

## API Endpoint

```
POST /v1/whatsapp/notify
Content-Type: application/json
Authorization: Bearer <firebaseIdToken>
```

**Base URL**: `https://api.maintor.systems` (production)

**Authentication**: Required - Firebase ID token in Authorization header

## Request Format

### Required Fields

- `templateName` (string): Name of the approved WhatsApp template
- `languageCode` (string): Language code (e.g., "en_US", "es_ES")

### Optional Fields

- `phoneNumber` (string): Recipient phone number in E.164 format (e.g., "+972546800360")
  - If not provided, uses server default
  - Can include spaces, dashes, parentheses (will be cleaned automatically)
  - Should be 10-15 digits (with or without country code prefix)
- `headerParameters` (object): For templates with header components (image/text)
- `templateParameters` (array): For templates with body variable placeholders

## Request Examples

### Example 1: Basic Template with Image Header (No Phone Number)

```json
{
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": {
      "link": "https://example.com/your-image.jpg"
    }
  }
}
```

**Note**: If `phoneNumber` is not provided, the message will be sent to the server's default recipient.

### Example 2: Template with Custom Phone Number

```json
{
  "phoneNumber": "+972546800360",
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": {
      "link": "https://example.com/your-image.jpg"
    }
  }
}
```

### Example 3: Template with Body Parameters

```json
{
  "phoneNumber": "+972546800360",
  "templateName": "ticket_notification",
  "languageCode": "en_US",
  "templateParameters": [
    {
      "type": "text",
      "text": "John Doe"
    },
    {
      "type": "text",
      "text": "#12345"
    }
  ]
}
```

### Example 4: Template with Both Header and Body

```json
{
  "phoneNumber": "+972546800360",
  "templateName": "full_notification_template",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": {
      "link": "https://example.com/image.jpg"
    }
  },
  "templateParameters": [
    {
      "type": "text",
      "text": "John Doe"
    },
    {
      "type": "text",
      "text": "Ticket #12345"
    }
  ]
}
```

## Phone Number Format

The `phoneNumber` field accepts various formats:

- ✅ `+972546800360` (E.164 format - recommended)
- ✅ `972546800360` (without +, will be added automatically)
- ✅ `+972-54-6800360` (with dashes, will be cleaned)
- ✅ `+972 (54) 6800360` (with spaces and parentheses, will be cleaned)
- ❌ Must be 10-15 digits (excluding country code prefix)

**Important**: The phone number must be valid and the recipient must be in the allowed list (for development mode) or have opted in (for production).

## Response Format

### Success Response (200 OK)

```json
{
  "success": true,
  "message": "WhatsApp template message sent successfully",
  "messageId": "wamid.HBgMOTcyNTQ2ODAwMzYwFQIAERgSOTQzNzU1NzRFRDE3MUQ5QzU4AA==",
  "recipient": "+972546800360",
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US"
}
```

### Error Responses

**400 Bad Request** - Validation Error:
```json
{
  "error": "Validation error",
  "message": "phoneNumber must be a valid phone number (10-15 digits, E.164 format preferred, e.g., +972546800360)"
}
```

**400 Bad Request** - Missing Required Field:
```json
{
  "error": "Validation error",
  "message": "templateName is required and must be a non-empty string"
}
```

**401 Unauthorized** - Authentication Required:
- Missing or invalid Firebase ID token

**500 Internal Server Error** - API Error:
```json
{
  "error": "WhatsApp API error",
  "message": "WhatsApp API error: 400 Bad Request - Template requires an image header but none was provided..."
}
```

## Implementation Examples

### JavaScript/Vue.js Example

```javascript
import { getAuth } from 'firebase/auth';

async function sendWhatsAppNotification(options) {
  try {
    // Get Firebase ID token
    const auth = getAuth();
    const user = auth.currentUser;
    
    if (!user) {
      throw new Error('User must be authenticated');
    }
    
    const idToken = await user.getIdToken();
    
    // Build request body
    const body = {
      templateName: options.templateName,
      languageCode: options.languageCode || 'en_US'
    };
    
    // Add phone number if provided
    if (options.phoneNumber) {
      body.phoneNumber = options.phoneNumber;
    }
    
    // Add header parameters if provided
    if (options.headerImageUrl) {
      body.headerParameters = {
        type: 'image',
        image: {
          link: options.headerImageUrl
        }
      };
    }
    
    // Add body parameters if provided
    if (options.bodyParameters && options.bodyParameters.length > 0) {
      body.templateParameters = options.bodyParameters.map(text => ({
        type: 'text',
        text
      }));
    }
    
    // Send request
    const response = await fetch('https://api.maintor.systems/v1/whatsapp/notify', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${idToken}`
      },
      body: JSON.stringify(body)
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Failed to send WhatsApp message');
    }
    
    const result = await response.json();
    return result;
    
  } catch (error) {
    console.error('Error sending WhatsApp notification:', error);
    throw error;
  }
}

// Usage examples:

// Send to specific phone number
await sendWhatsAppNotification({
  phoneNumber: '+972546800360',
  templateName: 'jaspers_market_image_cta_v1',
  languageCode: 'en_US',
  headerImageUrl: 'https://example.com/image.jpg'
});

// Send to default recipient (no phoneNumber)
await sendWhatsAppNotification({
  templateName: 'jaspers_market_image_cta_v1',
  languageCode: 'en_US',
  headerImageUrl: 'https://example.com/image.jpg'
});

// Send with body parameters
await sendWhatsAppNotification({
  phoneNumber: '+972546800360',
  templateName: 'ticket_notification',
  languageCode: 'en_US',
  bodyParameters: ['John Doe', '#12345']
});
```

### React Hook Example

```javascript
import { useState } from 'react';
import { getAuth } from 'firebase/auth';

export function useWhatsAppNotification() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [result, setResult] = useState(null);
  
  const sendNotification = async (templateName, languageCode, options = {}) => {
    setLoading(true);
    setError(null);
    setResult(null);
    
    try {
      const auth = getAuth();
      const user = auth.currentUser;
      
      if (!user) {
        throw new Error('User must be authenticated');
      }
      
      const idToken = await user.getIdToken();
      
      const body = {
        templateName,
        languageCode
      };
      
      // Add phone number if provided
      if (options.phoneNumber) {
        body.phoneNumber = options.phoneNumber;
      }
      
      if (options.headerImageUrl) {
        body.headerParameters = {
          type: 'image',
          image: {
            link: options.headerImageUrl
          }
        };
      }
      
      if (options.bodyParameters) {
        body.templateParameters = options.bodyParameters.map(text => ({
          type: 'text',
          text
        }));
      }
      
      const response = await fetch('https://api.maintor.systems/v1/whatsapp/notify', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${idToken}`
        },
        body: JSON.stringify(body)
      });
      
      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || 'Failed to send message');
      }
      
      const data = await response.json();
      setResult(data);
      return data;
      
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };
  
  return { sendNotification, loading, error, result };
}

// Usage in component:
function NotificationButton({ phoneNumber, templateName }) {
  const { sendNotification, loading, error } = useWhatsAppNotification();
  
  const handleSend = async () => {
    try {
      await sendNotification(
        templateName || 'jaspers_market_image_cta_v1',
        'en_US',
        {
          phoneNumber: phoneNumber, // Optional - omit to use default
          headerImageUrl: 'https://example.com/image.jpg'
        }
      );
      alert('Message sent successfully!');
    } catch (err) {
      alert(`Failed to send: ${err.message}`);
    }
  };
  
  return (
    <button onClick={handleSend} disabled={loading}>
      {loading ? 'Sending...' : 'Send WhatsApp Notification'}
    </button>
  );
}
```

## Common Templates

### jaspers_market_image_cta_v1

**Required**: Image header  
**Body Parameters**: None  
**Phone Number**: Optional

```json
{
  "phoneNumber": "+972546800360",
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": {
      "link": "https://your-domain.com/image.jpg"
    }
  }
}
```

## Phone Number Handling

### When to Include phoneNumber

- ✅ **Include** when sending to a specific user (e.g., from user profile, ticket assignment)
- ✅ **Include** when sending notifications to different recipients
- ✅ **Omit** when testing or when using a default recipient

### Phone Number Priority

1. **Request body** (`phoneNumber` field) - highest priority
2. **Server environment variable** (`WHATSAPP_DEFAULT_RECIPIENT`) - if set
3. **Hardcoded default** - fallback for testing

### Best Practices

- Always validate phone numbers on the frontend before sending
- Store phone numbers in E.164 format in your database
- Handle cases where phone number might be missing (use default)
- Show the recipient in the UI when sending (from response.recipient)

## Error Handling

### Common Errors

1. **Invalid Phone Number**
   ```
   phoneNumber must be a valid phone number (10-15 digits, E.164 format preferred)
   ```
   → Check phone number format and length

2. **Missing Image Header**
   ```
   Template requires an image header but none was provided
   ```
   → Add `headerParameters` with image

3. **Authentication Error**
   ```
   401 Unauthorized
   ```
   → Check Firebase ID token is valid

4. **Recipient Not Allowed** (Development Mode)
   ```
   Recipient phone number not in allowed list
   ```
   → Add recipient to Meta Business Manager allowed list

## Testing

### Testing with Default Recipient

Omit the `phoneNumber` field to use the server's default:

```javascript
await sendWhatsAppNotification({
  templateName: 'jaspers_market_image_cta_v1',
  languageCode: 'en_US',
  headerImageUrl: 'https://example.com/image.jpg'
  // No phoneNumber - uses default
});
```

### Testing with Specific Recipient

Include `phoneNumber` to test with a specific number:

```javascript
await sendWhatsAppNotification({
  phoneNumber: '+972546800360',
  templateName: 'jaspers_market_image_cta_v1',
  languageCode: 'en_US',
  headerImageUrl: 'https://example.com/image.jpg'
});
```

## Important Notes

- ✅ **Phone number is optional** - if omitted, uses server default
- ✅ **Phone number format is flexible** - accepts various formats (spaces, dashes, etc.)
- ✅ **Authentication required** - Firebase ID token must be valid
- ✅ **Image URLs must be publicly accessible** - HTTPS required
- ✅ **Templates must be pre-approved** - only approved templates can be sent
- ⚠️ **Development mode restrictions** - recipients must be in allowed list

## API Reference

For complete API documentation, see: `openapi.json` (search for `/v1/whatsapp/notify`)

## Support

If you encounter issues:
1. Check the error message in the response
2. Verify phone number format (10-15 digits)
3. Ensure image URLs are publicly accessible
4. Check that authentication token is valid
5. Verify template name and structure match Meta's approved template

