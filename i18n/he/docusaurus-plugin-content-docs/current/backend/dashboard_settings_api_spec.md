# Backend API Specification: Dashboard Visibility Settings

**Objective:** Add explicit API support for dashboard UI visibility toggles in Account Settings.

**Endpoints Affected:**
- `GET /accounts/{accountId}/settings`
- `PATCH /accounts/{accountId}/settings`

**Schema Updates (openapi.json):**
We need to add two new boolean fields to the `dashboard` object located within `formSettings` for both the `AccountSettings` and `AccountSettingsUpdate` schemas.

### 1. `AccountSettings` Schema
**Path:** `components.schemas.AccountSettings.properties.formSettings.properties.dashboard.properties`

Add the following properties:
```json
"showDateToolbar": {
  "type": "boolean",
  "description": "Show/hide date and time range selectors on the dashboard"
},
"showCardsStrip": {
  "type": "boolean",
  "description": "Show/hide top row cards with key metrics on the dashboard"
}
```

### 2. `AccountSettingsUpdate` Schema
**Path:** `components.schemas.AccountSettingsUpdate.properties.settings.properties.formSettings.properties.dashboard.properties`

Add the following properties:
```json
"showDateToolbar": {
  "type": "boolean"
},
"showCardsStrip": {
  "type": "boolean"
}
```

**Database/Model Updates:**
- Ensure the backend settings model (e.g., Mongoose schema, ORM model, etc.) can store these two boolean values under the `formSettings.dashboard` nested object.
- **Default Values:** 
  - `showDateToolbar`: `false`
  - `showCardsStrip`: `true`

**Current Frontend Integration Status:**
The frontend is already configured to read from and write to these exact paths (`settings.formSettings.dashboard.showDateToolbar` and `settings.formSettings.dashboard.showCardsStrip`). Once the backend explicitly adds them to its strict schema validations and data fetching logic, the frontend will automatically persist these values across user sessions.
