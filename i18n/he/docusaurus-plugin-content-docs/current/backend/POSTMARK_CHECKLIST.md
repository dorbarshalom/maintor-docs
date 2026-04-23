# Postmark Setup Checklist

## Status: Code Implementation ✅ Complete

The Postmark integration code is fully implemented and ready. You just need to configure your Postmark account and set the environment variables.

## What You Need to Complete

### 1. Postmark Account Setup (Manual Steps)

- [ ] **Create Postmark Account**
  - Sign up at https://postmarkapp.com
  - Complete account setup

- [ ] **Create Postmark Server**
  - Go to **Servers** → **Create Server**
  - Name your server (e.g., "Maintor Production")
  - Note: You'll need this for the API token

- [ ] **Get Server API Token**
  - Go to **Servers** → Select your server
  - Go to **API Tokens** tab
  - Copy the **Server API Token** (starts with something like `abc123...`)
  - **Save this** - you'll need it for step 2

- [ ] **Create Email Template**
  - Go to **Templates** → **Create Template**
  - Design your invitation email
  - Use these template variables:
    - `{{user_email}}` - Invited user's email
    - `{{user_first_name}}` - User's first name
    - `{{user_last_name}}` - User's last name
    - `{{invitation_link}}` - Full invitation URL with JWT token
    - `{{inviter_name}}` - Name of person who sent invitation
    - `{{inviter_email}}` - Email of person who sent invitation
    - `{{account_name}}` - Account name
  - **Note the Template ID** (numeric, shown in URL or template details)
  - **Save this** - you'll need it for step 2

- [ ] **Verify Sender Email**
  - Go to **Signatures** → **Add Signature**
  - Add email address you want to send from (e.g., `noreply@maintor.com`)
  - Verify the email (Postmark will send verification email)
  - **Note the verified email** - you'll need it for step 2

### 2. Set Environment Variables (Using Setup Script)

Run the automated setup script:

```bash
./setup-postmark-secrets.sh
```

This script will:
- Set all Postmark environment variables
- Preserve existing environment variables from your `.env` file
- Redeploy the Cloud Function with the new variables

**Prerequisites:**
- `gcloud` CLI installed and authenticated
- `GOOGLE_CLOUD_PROJECT` environment variable set (or defaults to "maintor")
- `GOOGLE_CLOUD_REGION` environment variable set (or defaults to "us-central1")

**Alternative: Manual Setup**

If you prefer to set them manually, update your deployment script or use:

```bash
gcloud functions deploy maintor-api \
  --set-env-vars "POSTMARK_API_KEY=...,POSTMARK_INVITATION_TEMPLATE_ID=...,..." \
  --region=us-central1
```

### 3. Test the Setup

- [ ] **Create a Test User**
  ```bash
  POST /v1/accounts/{accountId}/users
  {
    "firstName": "Test",
    "lastName": "User",
    "email": "your-test-email@example.com",
    "initialRole": {
      "scope": "account",
      "siteId": null,
      "role": "TECHNICIAN"
    }
  }
  ```

- [ ] **Check Cloud Tasks Queue**
  - Verify the invitation task is processed
  - Check Google Cloud Functions logs for any errors
  - View logs: `gcloud functions logs read maintor-api --region=us-central1 --limit=50`

- [ ] **Verify Email Received**
  - Check the test email inbox
  - Verify email content looks correct
  - Verify invitation link is present

- [ ] **Test Invitation Link**
  - Click the invitation link
  - Verify it opens your frontend signup page
  - Verify the `InviteToken` parameter is in the URL

## Quick Reference

### Required Secrets
- `POSTMARK_API_KEY` - Postmark Server API Token
- `POSTMARK_INVITATION_TEMPLATE_ID` - Numeric template ID
- `INVITATION_JWT_SECRET` - Secure random string for JWT signing

### Optional Secrets (have defaults)
- `POSTMARK_FROM_EMAIL` - Defaults to `noreply@maintor.com`
- `FRONTEND_URL` - Defaults to `https://yourapp.com`
- `INVITATION_JWT_EXPIRY_DAYS` - Defaults to `7`

## Troubleshooting

If emails aren't sending:

1. **Check Secrets Are Set**
   ```bash
   # List secrets (won't show values, but confirms they exist)
   wrangler secret list
   ```

2. **Check Google Cloud Functions Logs**
   - View logs: `gcloud functions logs read maintor-api --region=us-central1 --limit=50`
   - Or check in Google Cloud Console: Cloud Functions → maintor-api → Logs
   - Look for Postmark API errors
   - Check for missing environment variables

3. **Verify Postmark Settings**
   - Confirm API token is correct
   - Verify template ID matches
   - Check sender email is verified

4. **Test Postmark API Directly**
   - Use Postmark's API testing tool
   - Verify template variables are correct

## Next Steps After Setup

Once Postmark is configured:
1. ✅ Test with a real user invitation
2. ✅ Monitor email delivery rates in Postmark dashboard
3. ✅ Set up email bounce/spam handling if needed
4. ✅ Consider improving inviter info (currently uses account name)

## Documentation

- Full setup guide: `POSTMARK_SETUP.md`
- Postmark API docs: https://postmarkapp.com/developer/api/email-api

