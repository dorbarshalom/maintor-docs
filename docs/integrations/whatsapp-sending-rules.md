# WhatsApp Sending Trigger Conditions & Rules

This document outlines the specific conditions under which the Maintor system triggers and sends WhatsApp messages, how recipients are determined, and how Sandbox vs. Live mode affects message delivery.

---

## 1. Global Activation Setting (Live vs. Sandbox Mode)

Even when a trigger event is met, the system determines whether to call the live messaging API or simulate the delivery based on the account configuration:

* **Live Mode:** The account must have `whatsappLive: true` enabled in the `account_settings` collection for the corresponding `accountId`, and the `WASENDER_API_KEY` environment variable must be set. Messages are dispatched directly to the recipient using the Wasender API.
* **Sandbox Mode:** If `whatsappLive` is `false` (or unset), the integration runs in **Sandbox Mode**. The system intercepts the call, bypasses the actual Meta/Wasender API, and logs the payload to the database in the `whatsapp_logs` collection for verification.

---

## 2. Trigger Events & Message Content

WhatsApp messages are automatically triggered by the backend under four main circumstances:

### A. Hourly 24-Hour Reminders (`sendPlanned24hReminders`)
An hourly cron job scans upcoming planned tasks. It triggers a reminder if a ticket meets **all** of the following conditions:
* The ticket `type` is `PLANNED`.
* The ticket `status` is `PLANNED`.
* The ticket's `scheduled_date` is scheduled to start within the next 24 hours.
* The field `planned_24h_whatsapp_sent_at` is empty, null, or missing (ensuring it is only sent once).

**Message Content:**
> "היי, משימה מתוכננת עומדת להתחיל" *(Hey, a planned task is about to start)*

---

### B. Breakdown Ticket Creation (`notifyBreakdownCreated`)
Triggered immediately when a ticket is created with a `type` of `BREAKDOWN`.

**Message Content:**
> "היי, נפתח דיווח תקלה חדש עבור מכונה" *(Hey, a new breakdown report has been opened for a machine)*

---

### C. Status Transition from Planned to Open (`notifyPlannedToOpen`)
Triggered when a planned ticket is updated and its status transitions specifically from `PLANNED` to `OPEN`.

**Message Content:**
> "היי, יש לבצע אחזקה מתוכננת עבור מכונה." *(Hey, planned maintenance needs to be performed for a machine.)*

---

### D. Ad-hoc Admin Notifications (`sendWhatsAppNotify`)
Triggered manually when an administrator issues a POST request to the `/v1/whatsapp/notify` endpoint (typically through the WhatsApp Console panel on the dashboard).

---

## 3. Recipient Resolution Rules

For automated ticket-based triggers, the system compiles a list of recipient phone numbers using the following priority and hierarchy:

1. **Ticket Assignees:** The phone numbers from the user profiles of all users assigned to the ticket (`assignees`, `assignee_user_id`).
2. **Asset Owners & Co-Owners:** The phone numbers from the user profiles of the owner (`ownerUserId`, `owners`) and co-owners (`coOwners`, `co_owners`) of the asset assigned to the ticket.
3. **Test Recipients:** Phone numbers defined in the static config `test-whatsapp-recipients.json` (if any are present) are appended to all ticket notifications.
4. **Ad-hoc Validation:** For ad-hoc admin messages, if no specific recipient number is supplied, the system returns a validation error `400` unless test recipients are configured.

All phone numbers are parsed and cleaned dynamically to match the standard international E.164 formatting before transmission.
