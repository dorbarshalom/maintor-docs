# WhatsApp API - Quick Reference Card

## Endpoint
```
POST /v1/whatsapp/notify
Authorization: Bearer <firebaseIdToken>
```

## Required Fields
- `templateName` (string): Template name (e.g., "jaspers_market_image_cta_v1")
- `languageCode` (string): Language code (e.g., "en_US")

## Optional Fields
- `headerParameters` (object): For templates with headers
- `templateParameters` (array): For templates with body variables

## Quick Examples

### Template with Image Header (jaspers_market_image_cta_v1)
```json
{
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": {
      "link": "https://example.com/image.jpg"
    }
  }
}
```

### Template with Body Parameters
```json
{
  "templateName": "example_template",
  "languageCode": "en_US",
  "templateParameters": [
    { "type": "text", "text": "John Doe" },
    { "type": "text", "text": "12345" }
  ]
}
```

### Template with Both Header and Body
```json
{
  "templateName": "full_template",
  "languageCode": "en_US",
  "headerParameters": {
    "type": "image",
    "image": { "link": "https://example.com/image.jpg" }
  },
  "templateParameters": [
    { "type": "text", "text": "John Doe" }
  ]
}
```

## JavaScript Example
```javascript
const auth = getAuth();
const idToken = await auth.currentUser.getIdToken();

const response = await fetch('https://api.maintor.systems/v1/whatsapp/notify', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${idToken}`
  },
  body: JSON.stringify({
    templateName: 'jaspers_market_image_cta_v1',
    languageCode: 'en_US',
    headerParameters: {
      type: 'image',
      image: { link: 'https://example.com/image.jpg' }
    }
  })
});

const result = await response.json();
```

## Common Errors

**Missing Image Header**:
```
Template requires an image header but none was provided
```
→ Add `headerParameters` with image

**Invalid Image URL**:
→ Ensure URL is HTTPS and publicly accessible

**Authentication Error**:
→ Check Firebase ID token is valid

## Full Documentation
See `WHATSAPP_TEMPLATE_FRONTEND_GUIDE.md` for complete guide

