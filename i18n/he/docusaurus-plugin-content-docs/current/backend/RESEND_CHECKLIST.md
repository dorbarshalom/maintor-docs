# רשימת משימות להגדרת Resend

אינטגרציית הקוד עבור Resend מוטמעת לחלוטין ומוכנה לשימוש. עליכם רק להגדיר את חשבון ה-Resend שלכם ולהגדיר את משתני הסביבה.

### 1. הגדרת חשבון Resend (שלבים ידניים)

- [ ] **יצירת חשבון Resend**
  - הרשמו בכתובת https://resend.com

- [ ] **הוספה ואימות דומיין**
  - הוסיפו את הדומיין או הסאב-דומיין שלכם (למשל, `maintor.systems`) תחת לשונית Domains
  - הגדירו את רשומות ה-DNS הנדרשות במנהל הדומיינים שלכם

- [ ] **יצירת מפתח API**
  - צרו מפתח API חדש עם הרשאות שליחה
  - העתיקו אותו מיד

- [ ] **יצירת תבנית אימייל להזמנה**
  - צרו תבנית בשם `invitation-email-1`
  - עצבו את התבנית והוסיפו את המשתנים הנדרשים: `user_first_name`, `user_last_name`, `user_email`, `invitation_link`, `inviter_name`, `inviter_email`, `account_name`

### 2. הגדרת סודות ב-Cloud Function

השתמשו בתסריט האוטומטי המצורף כדי לעדכן את הגדרות ה-GCP Cloud Function שלכם:

```bash
./setup-resend-secrets.sh
```

או הגדירו ידנית דרך מסוף הניהול של Google Cloud או באמצעות gcloud CLI:

משתני סביבה נדרשים:
- `RESEND_API_KEY` - מפתח ה-API של Resend
- `RESEND_INVITATION_TEMPLATE_ID` - מזהה/כינוי התבנית (ברירת מחדל ל-`invitation-email-1`)
- `RESEND_FROM_EMAIL` - אימייל שולח מאומת (למשל, `hello@maintor.systems`)
- `INVITATION_JWT_SECRET` - מפתח סודי לחתימה על טוקני JWT

### 3. אימות משלוח

- [ ] בצעו הזמנה לכתובת אימייל לבדיקה
- [ ] עקבו אחר לוח הבקרה ב-Resend כדי לוודא שסטטוס משלוח האימייל הוא תקין
