# Maintor Environment Key Rotation Guide

> [!CAUTION]
> If secrets from [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env) or service account key files were committed to GitHub, updating local `.env` files alone is **insufficient**. You must revoke/rotate credentials directly on the respective service provider platforms so exposed keys become immediately invalid.

---

## 1. Firebase & GCP Service Account (`FIREBASE_PRIVATE_KEY` / `GOOGLE_APPLICATION_CREDENTIALS`)

- **Project**: `maintor`
- **Service Account**: `firebase-adminsdk-fbsvc@maintor.iam.gserviceaccount.com`

### Steps to Rotate:
1. Open [Google Cloud Console - Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts) for project **`maintor`**.
2. Click on `firebase-adminsdk-fbsvc@maintor.iam.gserviceaccount.com`.
3. Click **Add Key** $\rightarrow$ **Create new key** $\rightarrow$ **JSON** to download the key file.
4. Open the downloaded `.json` file in a text editor, copy the `private_key` value (e.g. `"-----BEGIN PRIVATE KEY-----\n..."`), and paste it directly into `FIREBASE_PRIVATE_KEY` in [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env).
5. Delete both old keys in Google Cloud Console.
6. Discard/delete the downloaded `.json` file from your Downloads folder.

---

## 2. Resend Email API (`RESEND_API_KEY`)

1. Go to the [Resend API Keys Dashboard](https://resend.com/api-keys).
2. Delete/Revoke the exposed key (`re_TD3PFVPE...`).
3. Click **Create API Key** and grant sending permissions for `mail.maintor.systems`.
4. Update `RESEND_API_KEY` in [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env).

---

## 3. WaSender WhatsApp Gateway (`WASENDER_API_KEY`)

1. Log into your WaSender dashboard and rotate your API Key.
2. Update `WASENDER_API_KEY` in [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env).

---

## 4. Internal Secrets (`INVITATION_JWT_SECRET`, `DEV_TOKEN_SECRET`, `INTERNAL_API_SECRET`)

Generate fresh 256-bit cryptographically secure secrets in terminal:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```
Update the following variables in [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env):
- `INVITATION_JWT_SECRET`
- `DEV_TOKEN_SECRET`
- `INTERNAL_API_SECRET`

---

## 5. MongoDB / Database Connection String (`MONGODB_CONNECTION_STRING`)

1. Access your MongoDB / Cloud Database cluster control panel.
2. Change the password for the database user (`api-user`).
3. Update `MONGODB_CONNECTION_STRING` in [`maintor-api/.env`](file:///Users/dorbarshalom/Dev/maintor-api/.env) with the new password.

---

## 6. Verify Git Safety

Ensure `.env` and `service-account-key.json` are listed in [`maintor-api/.gitignore`](file:///Users/dorbarshalom/Dev/maintor-api/.gitignore).
Check local git status:
```bash
git status
```
