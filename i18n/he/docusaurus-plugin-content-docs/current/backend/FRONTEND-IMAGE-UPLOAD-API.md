# Image Upload API - Frontend Integration Guide

## Overview

The backend now provides a dedicated image upload endpoint that automatically processes images (crops to 16:9 aspect ratio, optimizes, and uploads to Firebase Storage). This replaces direct Firebase Storage uploads and resolves CORS issues.

## Endpoint

**URL:** `POST /v1/upload/image`

**Authentication:** Required (include Firebase ID token in Authorization header)

## Request Format

### Content-Type
`multipart/form-data`

### Form Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | Yes | Image file to upload (JPEG, PNG, WebP, GIF). Maximum size: 10MB |
| `type` | String | No | Optional type to organize uploads (e.g., `"sites"`, `"users"`, `"assets"`). Defaults to `"uploads"` |

### File Requirements

- **Allowed types:** JPEG, PNG, WebP, GIF
- **Maximum size:** 10MB
- **Processing:** Images are automatically:
  - Cropped to 16:9 aspect ratio (centered crop)
  - Resized to maximum width of 1920px (maintains 16:9)
  - Converted to JPEG format with 85% quality
  - Optimized for web delivery

## Response Format

### Success Response (200 OK)

```json
{
  "url": "https://storage.googleapis.com/maintor.firebasestorage.app/sites/accountId/1234567890_abc123_filename.jpg",
  "fileName": "sites/accountId/1234567890_abc123_filename.jpg",
  "size": 245678,
  "type": "image/jpeg"
}
```

**Fields:**
- `url` (string): Public URL of the uploaded image - **use this when creating/updating entities**
- `fileName` (string): Storage path of the uploaded file
- `size` (integer): Size of the processed image in bytes
- `type` (string): MIME type of the processed image (always `image/jpeg`)

### Error Responses

#### 400 Bad Request
```json
{
  "error": "Invalid file type",
  "message": "File type image/bmp is not allowed. Allowed types: image/jpeg, image/jpg, image/png, image/webp, image/gif"
}
```

**Common 400 errors:**
- Invalid file type
- File too large (>10MB)
- Missing file field
- Failed to parse multipart form data

#### 401 Unauthorized
```json
{
  "error": "Unauthorized",
  "message": "Authentication required"
}
```

#### 500 Internal Server Error
```json
{
  "error": "Failed to upload image",
  "message": "Error details..."
}
```

## Example Implementation

### JavaScript/TypeScript Example

```typescript
async function uploadImage(imageFile: File, type?: string): Promise<string> {
  const formData = new FormData();
  formData.append('file', imageFile);
  
  if (type) {
    formData.append('type', type);
  }

  const idToken = await getFirebaseIdToken(); // Your existing auth method
  
  const response = await fetch('/v1/upload/image', {
    method: 'POST',
    headers: {
      'Authorization': `IDTOKEN.${idToken}` // Your existing auth header format
    },
    body: formData
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'Failed to upload image');
  }

  const data = await response.json();
  return data.url; // Use this URL when creating/updating entities
}

// Usage example for Site image upload
async function updateSiteImage(siteId: string, imageFile: File) {
  // Upload image first
  const imageUrl = await uploadImage(imageFile, 'sites');
  
  // Then update the site with the image URL
  await fetch(`/v1/accounts/${accountId}/sites/${siteId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `IDTOKEN.${idToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      image: imageUrl
    })
  });
}
```

### Vue.js Example (Quasar)

```javascript
// In your component
async function handleImageUpload(file) {
  try {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('type', 'sites'); // or 'users', 'assets', etc.

    const response = await api.post('/v1/upload/image', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
        'Authorization': `IDTOKEN.${idToken}`
      }
    });

    // Use response.data.url when creating/updating entities
    return response.data.url;
  } catch (error) {
    console.error('Image upload failed:', error);
    throw error;
  }
}
```

## Migration from Direct Firebase Storage Uploads

### Before (Direct Firebase Storage - Remove This)
```javascript
// ❌ OLD: Direct Firebase Storage upload
const storageRef = ref(storage, `sites/${accountId}/${fileName}`);
await uploadBytes(storageRef, file);
const url = await getDownloadURL(storageRef);
```

### After (Backend API - Use This)
```javascript
// ✅ NEW: Backend upload endpoint
const formData = new FormData();
formData.append('file', file);
formData.append('type', 'sites');

const response = await fetch('/v1/upload/image', {
  method: 'POST',
  headers: {
    'Authorization': `IDTOKEN.${idToken}`
  },
  body: formData
});

const { url } = await response.json();
// Use url when creating/updating entities
```

## Benefits

1. **No CORS Issues:** Backend handles Firebase Storage uploads server-side
2. **Automatic Processing:** Images are automatically cropped to 16:9 and optimized
3. **Consistent Format:** All images are converted to JPEG with consistent quality
4. **Better Error Handling:** Clear error messages for validation failures
5. **Organized Storage:** Files are organized by type and account ID

## Notes

- **No frontend cropping needed:** The backend automatically crops images to 16:9 aspect ratio
- **Image format:** All processed images are converted to JPEG format
- **File organization:** Use the `type` field to organize uploads (e.g., `"sites"`, `"users"`, `"assets"`)
- **URL usage:** Always use the `url` field from the response when setting the `image` field on entities (Site, User, Asset, Company Structure Template, Organization Chart Node)

## Testing

You can test the endpoint using curl:

```bash
curl -X POST http://localhost:8080/v1/upload/image \
  -H "Authorization: IDTOKEN.YOUR_TOKEN_HERE" \
  -F "file=@/path/to/image.jpg" \
  -F "type=sites"
```

## Support

If you encounter any issues, check:
1. File type is one of the allowed types (JPEG, PNG, WebP, GIF)
2. File size is under 10MB
3. Authentication token is valid and included in the request
4. Content-Type header is `multipart/form-data` (usually set automatically by FormData)

