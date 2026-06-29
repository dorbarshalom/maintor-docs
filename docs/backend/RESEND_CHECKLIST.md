# Resend Setup Checklist

The Resend integration code is fully implemented and ready. You just need to configure your Resend account and set the environment variables.

### 1. Resend Account Setup (Manual Steps)

- [ ] **Create Resend Account**
  - Sign up at https://resend.com

- [ ] **Add and Verify Domain**
  - Add your domain or subdomain (e.g., `maintor.systems`) under the Domains tab
  - Configure the required DNS records in your domain manager

- [ ] **Create an API Key**
  - Create a new API key with sending permissions
  - Copy it immediately

- [ ] **Create Invitation Email Template**
  - Create a template named `invitation-email-1`
  - Design the template and add the required template variables: `user_first_name`, `user_last_name`, `user_email`, `invitation_link`, `inviter_name`, `inviter_email`, `account_name`

### 2. Configure Cloud Function Secrets

Use the provided automated script to update your GCP Cloud Function settings:

```bash
./setup-resend-secrets.sh
```

Or configure manually via the Google Cloud Console / gcloud CLI:

Required environment variables:
- `RESEND_API_KEY` - Resend API Key
- `RESEND_INVITATION_TEMPLATE_ID` - Template ID/alias (defaults to `invitation-email-1`)
- `RESEND_FROM_EMAIL` - Verified sender email (e.g. `hello@maintor.systems`)
- `INVITATION_JWT_SECRET` - JWT signing key

### 3. Verify Delivery

- [ ] Trigger an invitation to a test email address
- [ ] Monitor the Resend dashboard to confirm email delivery status
