---
sidebar_label: מדריך אינטגרציית WhatsApp
title: מדריך אינטגרציית WhatsApp
---
# WhatsApp Integration Guide

## Overview
Maintor is integrated with WhatsApp messaging using the **Meta WhatsApp Business Cloud API**. This integration allows the application (from both frontend interactions and background tasks) to send targeted notifications to appropriate parties based on predefined WhatsApp message templates.

Because our WhatsApp Business app is currently in **development mode**, the integration exclusively uses **template-based messages**. Free-form text messages are not supported by Meta in development mode or outside a user-initiated 24-hour service window.

---

## Architectural Flow

1. **Trigger Phase (Frontend/Client)**: The frontend (React/Vue app) authenticates via Firebase Auth and sends an HTTP `POST` request to the backend endpoint `/v1/whatsapp/notify` containing the template name, language code, template parameters (body variables, header images), and the recipient's phone number.
2. **Validation Phase (Backend Handler)**: The Firebase Cloud Function (`sendWhatsAppNotify.js`) validates the request payload, defaults to a test phone number if none is supplied, and enforces E.164 phone number formatting requirements.
3. **Mapping Phase (Backend Utils)**: A utility script (`src/utils/whatsapp.js`) translates our internal API schema into Meta's required schema, adding headers, components, and parameter dictionaries required by the Graph API syntax.
4. **Delivery**: The backend sends the outgoing request securely to Meta's API (`graph.facebook.com/v21.0/.../messages`) using a Server Access Token stored in `WHATSAPP_ACCESS_TOKEN` environment variable.

---

## API Summary

### Send WhatsApp Notification Endpoint
**POST** `/v1/whatsapp/notify`
**Headers**: `Authorization: Bearer <firebaseIdToken>`

**Payload Format:**
```json
{
  "phoneNumber": "+972546800360", // Optional. Defaults to WHATSAPP_DEFAULT_RECIPIENT
  "templateName": "jaspers_market_image_cta_v1", // Required. Ensure it matches an approved Meta template
  "languageCode": "en_US", // Required. Add matching translation in Meta.
  "headerParameters": { // Optional depending on template
    "type": "image",
    "image": { "link": "https://example.com/image.jpg" }
  },
  "templateParameters": [ // Optional depending on template 
    { "type": "text", "text": "Variable #1" }
  ]
}
```

---

## Technical Details

### Backend Implementation
The backend implementation lies primarily in 2 files:
1. `src/handlers/sendWhatsAppNotify.js`: Responsible for Firebase auth verification validation, parsing input parameters, filling in defaults (like `WHATSAPP_DEFAULT_RECIPIENT`), and returning normalized standard HTTP responses on success and error.
2. `src/utils/whatsapp.js`: Manages the translation from Maintor JSON to WhatsApp Graph JSON schema, appending the `WHATSAPP_PHONE_NUMBER_ID` and `WHATSAPP_ACCESS_TOKEN` credentials and executing the fetch. It also includes comprehensive mapping of specific Meta API errors (e.g., token invalidation, missing headers) to user-friendly messages.

### Frontend Implementation
The frontend uses the `POST /v1/whatsapp/notify` endpoint by attaching the JWT from Firebase's `auth.currentUser.getIdToken()`. 

To effectively use the integration, always include proper error handling parsing. Frontend clients receive clean HTTP statuses (`200`, `400`, `401`, `500`) with actionable `{ message, error }` strings describing if parameters were missing (e.g., forgotten Image Header for media templates).

*Detailed frontend implementations, React Hooks, and error catch examples are documented in:*
- `WHATSAPP_FRONTEND_DEVELOPER_GUIDE.md`
- `docs/whatsapp-notification-frontend-guide.md`

---

## Token Management & Configuration

To interact with Meta's cloud infrastructure, the integration relies heavily on environment variables:
- `WHATSAPP_PHONE_NUMBER_ID`: The specific sender ID.
- `WHATSAPP_ACCESS_TOKEN`: The bearer token providing access rights.
- `WHATSAPP_API_VERSION`: Graph API version (typically v21.0).

### Token Invalidation Troubleshooting 
Often during development, Meta **Temporary Access Tokens** expire (after a few hours) or invalidate when the issuing user logs out of Facebook.
When encountering `Session has expired` or `The session is invalid because the user logged out` (401 errors):

1. Switch to a **System User Token** in Meta Business Settings -> Users -> System Users.
2. Ensure you assign permissions for `whatsapp_business_messaging` and `whatsapp_business_management`.
3. Save the new token in the environment file: `WHATSAPP_ACCESS_TOKEN=your_new_token_here`.
4. Deploy using the helper script: `./update-whatsapp-token.sh`.

*Token details are found in: `docs/WHATSAPP_TOKEN_REFRESH.md`*

---

## Key Development Rules & Constraints

1. **Sandbox / Dev Rules:** In Meta Development Mode, you can only message phone numbers that have been added and verified inside the Meta Business Manager allowed list. Free text messages are disabled.
2. **Template Accuracy**: `templateName`, `languageCode`, and component arrays must precisely match what Meta Business manager holds.
3. **Phone Formatting:** Maintor backend cleans and enforces `E.164` formatting. Ensure you collect full international prefixes (e.g., `+972` instead of `054-`).
