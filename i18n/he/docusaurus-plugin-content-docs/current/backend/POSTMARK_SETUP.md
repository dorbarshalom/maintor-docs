# Postmark Email Setup Guide

This guide explains how to set up Postmark for sending user invitation emails.

## Prerequisites

1. A Postmark account (sign up at https://postmarkapp.com)
2. A Postmark server (create one in your Postmark dashboard)
3. An email template in Postmark

## Step 1: Get Your Postmark API Key

1. Log in to your Postmark account
2. Go to **Servers** → Select your server (or create a new one)
3. Go to **API Tokens** tab
4. Copy your **Server API Token**

## Step 2: Create an Email Template

1. In Postmark, go to **Templates** → **Create Template**
2. Design your invitation email template
3. Use these template variables (they will be automatically populated):
   - `user_email` - The invited user's email address
   - `user_first_name` - The invited user's first name
   - `user_last_name` - The invited user's last name
   - `invitation_link` - The full invitation URL with JWT token
   - `inviter_name` - Name of the person who sent the invitation
   - `inviter_email` - Email of the person who sent the invitation
   - `account_name` - Name of the account the user is being invited to

4. Note the **Template ID** (numeric ID shown in the template URL or details)

## Step 3: Verify Sender Email

1. In Postmark, go to **Signatures** → **Add Signature**
2. Add and verify the email address you want to send from (e.g., `noreply@maintor.com`)
3. This email will be used as the `From` address in invitation emails

## Step 4: Set Environment Variables

For Google Cloud Functions, you can set environment variables using the provided script or manually via gcloud CLI.

### Option 1: Use the Setup Script (Recommended)

Run the automated setup script:

```bash
./setup-postmark-secrets.sh
```

This script will:
- Set all Postmark environment variables
- Preserve existing environment variables from your `.env` file
- Redeploy the Cloud Function with the new variables

### Option 2: Manual Setup via gcloud CLI

Update the Cloud Function with environment variables:

```bash
gcloud functions deploy maintor-api \
  --runtime nodejs20 \
  --trigger-http \
  --allow-unauthenticated \
  --source=. \
  --entry-point=maintorApi \
  --memory=512MB \
  --timeout=300s \
  --set-env-vars "POSTMARK_API_KEY=your-api-key,POSTMARK_INVITATION_TEMPLATE_ID=your-template-id,INVITATION_JWT_SECRET=your-jwt-secret,FRONTEND_URL=https://app.maintor.com,POSTMARK_FROM_EMAIL=hello@maintor.systems,INVITATION_JWT_EXPIRY_DAYS=7" \
  --region=us-central1 \
  --project=maintor
```

**Note**: Make sure to include all your existing environment variables (FIREBASE_PRIVATE_KEY, ACCESS_TOKEN_SECRET, etc.) in the `--set-env-vars` parameter, or they will be removed.

## Step 5: Test the Setup

1. Create a test user via the API:
   ```bash
   POST /v1/accounts/{accountId}/users
   {
     "firstName": "Test",
     "lastName": "User",
     "email": "test@example.com"
   }
   ```

2. Check the Cloud Tasks queue to ensure the invitation task is processed
3. Check the user's email inbox for the invitation email
4. Verify the invitation link works by clicking it

## Template Example

Here's an example Postmark template structure:

```
Subject: You've been invited to join {{account_name}}

Hi {{user_first_name}},

{{inviter_name}} ({{inviter_email}}) has invited you to join {{account_name}}.

Click the link below to accept your invitation:
{{invitation_link}}

This invitation will expire in 7 days.

If you didn't expect this invitation, you can safely ignore this email.
```

## Troubleshooting

### Emails not sending
- Check that `POSTMARK_API_KEY` is set correctly
- Verify the template ID matches your Postmark template
- Check Google Cloud Functions logs for error messages
- Verify the sender email is verified in Postmark
- Verify environment variables are set: `gcloud functions describe maintor-api --region=us-central1 --format="value(environmentVariables)"`

### Invalid template variables
- Ensure all template variables are spelled correctly
- Check that the template uses the exact variable names listed above

### JWT token errors
- Verify `INVITATION_JWT_SECRET` is set
- Check that the secret is the same across all environments
- Ensure `INVITATION_JWT_EXPIRY_DAYS` is a valid number

## Environment Variables Summary

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `POSTMARK_API_KEY` | Yes | - | Postmark Server API Token |
| `POSTMARK_INVITATION_TEMPLATE_ID` | Yes | - | Numeric template ID from Postmark |
| `POSTMARK_FROM_EMAIL` | No | `noreply@maintor.com` | Verified sender email |
| `FRONTEND_URL` | No | `https://yourapp.com` | Frontend URL for invitation links |
| `INVITATION_JWT_SECRET` | Yes | - | Secret key for signing JWT tokens |
| `INVITATION_JWT_EXPIRY_DAYS` | No | `7` | Days until invitation expires |

