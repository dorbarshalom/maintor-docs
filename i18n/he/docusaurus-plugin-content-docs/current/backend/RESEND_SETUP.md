# מדריך הגדרת Resend למשלוח אימייל

מדריך זה מסביר כיצד להגדיר את Resend למשלוח אימיילים של הזמנת משתמשים.

## דרישות קדם

1. חשבון Resend (הרשמה בכתובת https://resend.com)
2. דומיין או סאב-דומיין מאומת בחשבון Resend שלכם (למשל, `maintor.systems`)
3. תבנית אימייל שפורסמה ב-Resend (למשל, `invitation-email-1`)

## שלב 1: קבלת מפתח ה-API של Resend

1. התחברו לחשבון ה-Resend שלכם
2. עברו ללשונית **API Keys** בתפריט הצדדי
3. צרו מפתח API חדש והעתיקו אותו

## שלב 2: יצירה ופרסום של תבנית אימייל

1. ב-Resend, עברו אל **Templates** ← **Create Template**
2. קראו לה בשם `invitation-email-1` (או העתיקו את ה-ID של התבנית)
3. עצבו את תבנית הזמנת המשתמש והוסיפו את משתני התבנית הנדרשים:
   - `{{{user_first_name}}}` - השם הפרטי של המשתמש המוזמן
   - `{{{user_last_name}}}` - שם המשפחה של המשתמש המוזמן
   - `{{{user_email}}}` - כתובת האימייל של המשתמש המוזמן
   - `{{{invitation_link}}}` - הקישור המלא להרשמה / קבלת ההזמנה
   - `{{{inviter_name}}}` - שם השולח שהזמין את המשתמש
   - `{{{inviter_email}}}` - אימייל השולח שהזמין את המשתמש
   - `{{{account_name}}}` - שם החשבון אליו המשתמש מוזמן

## שלב 3: אימות דומיין וכתובת שולח

1. ב-Resend, עברו אל **Domains**
2. הוסיפו את הדומיין/סאב-דומיין שלכם (למשל, `maintor.systems`) והגדירו את רשומות ה-DNS הנדרשות אצל רשם הדומיינים שלכם
3. דומיין זה ישמש למשלוח אימיילים מכתובות כגון `hello@maintor.systems`

## שלב 4: הגדרת משתני סביבה

עבור Google Cloud Functions, תוכלו להגדיר משתני סביבה באמצעות התסריט המצורף או ידנית דרך ה-gcloud CLI.

### אפשרות 1: שימוש בתסריט ההגדרה (מומלץ)

הריצו את תסריט ההגדרה האוטומטי:

```bash
./setup-resend-secrets.sh
```

תסריט זה יבצע:
- הגדרת כל משתני הסביבה של Resend
- שמירה על משתני סביבה קיימים מקובץ ה-`.env` שלכם
- פריסה מחדש של ה-Cloud Function עם המשתנים החדשים

### אפשרות 2: הגדרה ידנית דרך gcloud CLI

עדכנו את ה-Cloud Function עם משתני הסביבה:

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

**הערה**: ודאו שאתם כוללים את כל משתני הסביבה הקיימים שלכם (FIREBASE_PRIVATE_KEY, ACCESS_TOKEN_SECRET וכדומה) בפרמטר `--set-env-vars`, אחרת הם יוסרו.

## שלב 5: בדיקת ההגדרה

1. צרו משתמש בדיקה דרך ה-API:
   ```bash
   POST /v1/accounts/{accountId}/users
   {
     "firstName": "Test",
     "lastName": "User",
     "email": "test@example.com"
   }
   ```

2. בדקו את תור ה-Cloud Tasks כדי לוודא שמשימת ההזמנה מעובדת
3. בדקו את תיבת הדואר הנכנס של המשתמש כדי לוודא שאימייל ההזמנה התקבל
4. ודאו שקישור ההזמנה עובד על ידי לחיצה עליו

## סיכום משתני סביבה

| משתנה | נדרש | ברירת מחדל | תיאור |
|-------|------|------------|-------|
| `RESEND_API_KEY` | כן | - | מפתח ה-API של Resend |
| `RESEND_INVITATION_TEMPLATE_ID` | כן | `invitation-email-1` | מזהה/כינוי התבנית ב-Resend |
| `RESEND_FROM_EMAIL` | לא | `hello@maintor.systems` | כתובת אימייל מאומתת של השולח |
| `FRONTEND_URL` | לא | `https://app.maintor.systems` | כתובת ה-Frontend עבור קישורי הזמנה |
| `INVITATION_JWT_SECRET` | כן | - | מפתח סודי לחתימת טוקני JWT |
| `INVITATION_JWT_EXPIRY_DAYS` | לא | `7` | ימים עד לפקיעת תוקף ההזמנה |
