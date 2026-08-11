# WhatsApp Integration Guide

## Overview
Maintor is integrated with WhatsApp messaging using **WaSender API** (`wasenderapi`). This integration allows the application to send notifications and messaging directly to recipients via WaSender.

---

## Architectural Flow

1. **Trigger Phase (Frontend/Client)**: The frontend authenticates via Firebase Auth and sends an HTTP `POST` request to the backend endpoint `/v1/whatsapp/notify` containing the message text and recipient details.
2. **Validation Phase (Backend Handler)**: The Firebase Cloud Function (`sendWhatsAppNotify.js`) validates the request payload and formats phone numbers to E.164.
3. **Delivery**: The backend dispatches messages via `src/utils/whatsapp-sender.js` using `wasenderapi` and `WASENDER_API_KEY`.

---

## Technical Details

### Backend Implementation
- `src/handlers/sendWhatsAppNotify.js`: Validates input parameters and triggers notification sends.
- `src/utils/whatsapp-sender.js`: Connects to WaSender API (`wasenderapi`) using `WASENDER_API_KEY`.

---

## Environment Configuration

The integration relies on the following environment variables:
- `WASENDER_API_KEY`: API Key generated from the WaSender portal.
- `WASENDER_SANDBOX`: Set to `true` for sandbox/mock sending or `false` for live sending.


---

## Key Development Rules & Constraints

1. **Sandbox / Dev Rules:** In Meta Development Mode, you can only message phone numbers that have been added and verified inside the Meta Business Manager allowed list. Free text messages are disabled.
2. **Template Accuracy**: `templateName`, `languageCode`, and component arrays must precisely match what Meta Business manager holds.
3. **Phone Formatting:** Maintor backend cleans and enforces `E.164` formatting. Ensure you collect full international prefixes (e.g., `+972` instead of `054-`).
