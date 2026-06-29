# Resend Email Setup Guide

This guide explains how to set up Resend for sending user invitation emails.

## Prerequisites

1. A Resend account (sign up at https://resend.com)
2. A verified domain or subdomain in your Resend account (e.g., `maintor.systems`)
3. A published email template in Resend (e.g. `invitation-email-1`)

## Step 1: Get Your Resend API Key

1. Log in to your Resend account
2. Go to **API Keys** tab in the sidebar
3. Create a new API key and copy it

## Step 2: Create/Publish an Email Template

1. In Resend, go to **Templates** → **Create Template**
2. Name it `invitation-email-1` (or get its template ID)
3. Design your invitation email template using these variable mappings:
   - `{{{user_first_name}}}` - The invited user's first name
   - `{{{user_last_name}}}` - The invited user's last name
   - `{{{user_email}}}` - The invited user's email address
   - `{{{invitation_link}}}` - The full signup/accept invitation URL
   - `{{{inviter_name}}}` - Name of the person who sent the invitation
   - `{{{inviter_email}}}` - Email of the person who sent the invitation
   - `{{{account_name}}}` - Name of the account the user is being invited to

## Step 3: Verify Domain and Sender Email

1. In Resend, go to **Domains**
2. Add your domain/subdomain (e.g., `maintor.systems`) and add the DNS records in your domain provider
3. This domain will be used to send emails from addresses like `hello@maintor.systems`

## Step 4: Set Environment Variables

For Google Cloud Functions, you can set environment variables using the provided script or manually via gcloud CLI.

### Option 1: Use the Setup Script (Recommended)

Run the automated setup script:

```bash
./setup-resend-secrets.sh
```

This script will:
- Set all Resend environment variables
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
  --set-env-vars "RESEND_API_KEY=your-api-key,RESEND_INVITATION_TEMPLATE_ID=invitation-email-1,INVITATION_JWT_SECRET=your-jwt-secret,FRONTEND_URL=https://app.maintor.systems,RESEND_FROM_EMAIL=hello@maintor.systems,INVITATION_JWT_EXPIRY_DAYS=7" \
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

## Environment Variables Summary

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RESEND_API_KEY` | Yes | - | Resend API Key |
| `RESEND_INVITATION_TEMPLATE_ID` | Yes | `invitation-email-1` | Template ID/alias from Resend |
| `RESEND_FROM_EMAIL` | No | `hello@maintor.systems` | Verified sender email |
| `FRONTEND_URL` | No | `https://app.maintor.systems` | Frontend URL for invitation links |
| `INVITATION_JWT_SECRET` | Yes | - | Secret key for signing JWT tokens |
| `INVITATION_JWT_EXPIRY_DAYS` | No | `7` | Days until invitation expires |
