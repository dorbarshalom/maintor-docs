# WhatsApp Notification - Quick Start for Frontend

## Endpoint

```
POST /v1/whatsapp/notify
Authorization: Bearer <firebaseIdToken>
Content-Type: application/json

Body (optional):
{
  "message": "Your message text"
}
```

## Quick Implementation

```javascript
async function sendWhatsAppMessage(message) {
  const auth = getAuth();
  const user = auth.currentUser;
  const idToken = await user.getIdToken();
  
  const response = await fetch('https://api.maintor.systems/v1/whatsapp/notify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${idToken}`
    },
    body: JSON.stringify(message ? { message } : {})
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'Failed to send message');
  }
  
  return await response.json();
}

// Usage
await sendWhatsAppMessage('Hello from frontend!');
```

## Response

**Success (200)**:
```json
{
  "success": true,
  "message": "WhatsApp message sent successfully",
  "messageId": "wamid.xxx",
  "recipient": "+972-54-6800360"
}
```

**Errors**:
- `400`: Invalid request
- `401`: Authentication required
- `500`: Server/configuration error

## Important Notes

- ✅ Requires Firebase authentication (ID token)
- ✅ Message is optional (default message sent if omitted)
- ✅ Currently sends to hardcoded recipient for testing
- ✅ Full guide: `docs/whatsapp-notification-frontend-guide.md`

