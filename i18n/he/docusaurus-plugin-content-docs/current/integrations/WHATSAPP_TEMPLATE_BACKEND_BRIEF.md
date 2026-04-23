# WhatsApp Template-Based Messages - Backend Implementation Brief

## Overview

The WhatsApp integration needs to be updated to use **template-based messages** instead of free text messages. This is required because the Meta WhatsApp Business API app is currently in **development mode**, which only allows sending pre-approved message templates.

## Current Implementation

The current API endpoint `/v1/whatsapp/notify` accepts a free text message:

```json
{
  "message": "hello"
}
```

## Required Changes

### 1. Update Request Body Format

The endpoint should now accept template information instead of a free text message:

**Template without parameters** (current use case):
```json
{
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US"
}
```

**Template with parameters** (for future templates):
```json
{
  "templateName": "example_template_with_params",
  "languageCode": "en_US",
  "templateParameters": [
    {
      "type": "text",
      "text": "John Doe"
    },
    {
      "type": "text",
      "text": "12345"
    }
  ]
}
```

### 2. Current Template Details

- **Template Name**: `jaspers_market_image_cta_v1`
- **Language**: `en_US` (English US)
- **Parameters**: None (this template has no variable placeholders)

### 3. Template Parameters Format

When templates require parameters (future use):

- `templateParameters` is an **optional** array of parameter objects
- Each parameter object has:
  - `type`: Parameter type (currently only `"text"` is used)
  - `text`: The actual parameter value (string)
- Parameters are **ordered** and correspond to template placeholders:
  - First parameter (`templateParameters[0]`) maps to `{{1}}` in the template
  - Second parameter (`templateParameters[1]`) maps to `{{2}}` in the template
  - And so on...

**Example**: If a template has the text "Hello `{{1}}`, your ticket `{{2}}` is ready", you would send:
```json
{
  "templateName": "ticket_ready_notification",
  "languageCode": "en_US",
  "templateParameters": [
    { "type": "text", "text": "John" },
    { "type": "text", "text": "#12345" }
  ]
}
```

### 4. Meta WhatsApp Business API Mapping

The backend needs to map the frontend request to Meta's WhatsApp Business API format.

**Meta API Template Message Format**:
```json
{
  "messaging_product": "whatsapp",
  "to": "<recipient_phone_number>",
  "type": "template",
  "template": {
    "name": "<template_name>",
    "language": {
      "code": "<language_code>"
    },
    "components": [
      {
        "type": "body",
        "parameters": [
          {
            "type": "text",
            "text": "<parameter_value>"
          }
        ]
      }
    ]
  }
}
```

**Mapping Guide**:
- `templateName` → `template.name`
- `languageCode` → `template.language.code`
- `templateParameters` → `template.components[0].parameters[]` (only if parameters exist)
- If `templateParameters` is not provided or empty, omit the `components` array entirely

**Example Mapping**:

Frontend request (no parameters):
```json
{
  "templateName": "jaspers_market_image_cta_v1",
  "languageCode": "en_US"
}
```

Meta API request:
```json
{
  "messaging_product": "whatsapp",
  "to": "+972-54-6800360",
  "type": "template",
  "template": {
    "name": "jaspers_market_image_cta_v1",
    "language": {
      "code": "en_US"
    }
  }
}
```

Frontend request (with parameters):
```json
{
  "templateName": "example_template",
  "languageCode": "en_US",
  "templateParameters": [
    { "type": "text", "text": "John" },
    { "type": "text", "text": "12345" }
  ]
}
```

Meta API request:
```json
{
  "messaging_product": "whatsapp",
  "to": "+972-54-6800360",
  "type": "template",
  "template": {
    "name": "example_template",
    "language": {
      "code": "en_US"
    },
    "components": [
      {
        "type": "body",
        "parameters": [
          {
            "type": "text",
            "text": "John"
          },
          {
            "type": "text",
            "text": "12345"
          }
        ]
      }
    ]
  }
}
```

### 5. Backward Compatibility

The old `message` field should be **removed** or **deprecated**. The API should only accept the new template format going forward.

### 6. Error Handling

The backend should validate:
- `templateName` is required and non-empty
- `languageCode` is required and valid (e.g., "en_US", "es_ES", etc.)
- `templateParameters` is optional but, if provided, must be an array
- Each parameter in `templateParameters` must have `type` and `text` fields
- Parameter count should match the template's placeholder count (if known)

Return appropriate error messages if validation fails.

### 7. Meta Development Mode Restrictions

- Only pre-approved templates can be sent
- Templates must be created and approved in Meta Business Manager
- Free text messages are not allowed in development mode
- Once the app is approved for production, free text messages may be allowed (but templates will still be used)

## Reference Documentation

- [Meta WhatsApp Business API - Send Template Messages](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-messages#template-messages)
- [Meta WhatsApp Business API - Message Templates](https://developers.facebook.com/docs/whatsapp/business-management-api/message-templates)
- [Meta WhatsApp Business API - Template Components](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/messages#template-object)

## Testing

After implementation, test with:
1. Template without parameters: `jaspers_market_image_cta_v1`
2. Template with parameters (when available): Any template with placeholders
3. Invalid template name: Should return appropriate error
4. Missing required fields: Should return validation error
5. Invalid parameter format: Should return validation error

## Questions or Issues

If you have questions about:
- Template approval process
- Parameter types beyond "text"
- Header/button components in templates
- Template language codes
- Error handling specifics

Please refer to the Meta documentation links above or reach out for clarification.

