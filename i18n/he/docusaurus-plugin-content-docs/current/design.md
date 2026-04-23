# מסמך תכנון Maintor API

## סקירה כללית

Maintor היא מערכת לניהול תחזוקה עבור חברות תעשייתיות. ה-API מספק נקודות קצה (endpoints) לניהול כרטיסי תחזוקה, שיכולים להיות תחזוקה מונעת (מתוכננת) או תחזוקת שבר.

## URL בסיס
```
https://your-api-domain.com/v1
```

## אימות (Authentication)

כל נקודות הקצה דורשות אימות באמצעות ה-middleware `identityValidator`. ה-API משתמש באימות Firebase עם Google OAuth או אימייל/סיסמה.

### כותרות (Headers)
```
Authorization: Bearer <firebase-token>
Content-Type: application/json
```

## מודלי נתונים (Data Models)

### מודל כרטיס (Ticket Model)

מודל הכרטיס המאוחד תומך הן בתחזוקה מונעת והן בתחזוקת שבר:

```typescript
interface Ticket {
  id: string;                     // מזהה ייחודי
  type: 'PLANNED' | 'BREAKDOWN'; // סוג הכרטיס
  assetId?: string;               // מזהה נכס - חובה לכרטיסי שבר

  // מידע בסיסי
  title: string;                  // כותרת הכרטיס
  description: string;            // תיאור
  priority: number;               // 1-5 (1=חירום, 5=נמוך)
  status: 'OPEN' | 'IN_PROGRESS' | 'COMPLETED' | 'CLOSED' | 'SKIPPED';

  // תזמון
  scheduled_date?: string;        // מתי התחזוקה המתוכננת צריכה לקרות (רק לכרטיסים מתוכננים)
  
  // ציר זמן (משותף לכרטיסים מתוכננים וכרטיסי שבר)
  timeline?: {
    started?: string;             // מתי העבודה/השבר התחילו (ISO datetime)
    ended?: string;               // מתי העבודה/השבר הסתיימו (ISO datetime)
    duration_min?: number;        // משך זמן כולל בדקות
    downtime_min?: number;        // זמן השבתת מכונה בדקות
    labor_entries?: Array<{      // מעקב עבודה מפורט
      user_id: string;            // מזהה משתמש
      started: string;            // זמן התחלה (ISO datetime)
      ended: string;              // זמן סיום (ISO datetime)
      duration_min: number;       // משך העבודה עבור ערך זה (דקות)
    }>;
  };

  // שדות ספציפיים לשבר (רק עבור סוג: "BREAKDOWN")
  breakdown?: {
    problem_description: string;  // מה השתבש - חובה
    solution_description?: string; // איך זה תוקן
    root_cause?: string;          // מקטלוג סיבות שורש
  };

  // הערות מפעיל (רלוונטי לשני הסוגים)
  operator_notes?: string;       // הקשר נוסף

  // משימות (מתוכנן) או הערות (שבר)
  tasks?: Array<{
    description: string;
    status: 'PENDING' | 'DONE' | 'SKIPPED' | 'FAILED';
  }>;

  notes?: Array<{
    at: string;                  // ISO datetime
    by: string;                  // user_id
    note: string;
  }>;

        // אנשים
        owner_user_id: string;         // בעלים (מתכנן/מפקח)
        reported_by_user_id: string;  // מי דיווח על השבר
        assignees: Array<{
          user_id: string;
        }>;

  // ראיות ותיעוד
  photos?: Array<{
    url: string;
    caption?: string;
  }>;

  attachments?: Array<{
    doc_id: string;
    title: string;
    kind: string;
  }>;

  // ביקורת (Audit)
  created_at: string;           // ISO datetime
  updated_at: string;           // ISO datetime
  archived_at?: string;         // ISO datetime
}
```

## נקודות קצה של ה-API (API Endpoints)

### ניהול חשבון

#### קבלת החשבון שלי
```http
GET /accounts/myAccount
```

**תגובה (Response):**
```json
{
  "id": "account_123",
  "name": "Acme Corp",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

### ניהול כרטיסים

#### רשימת כרטיסים
```http
GET /accounts/{accountId}/tickets
```

**פרמטרי שאילתה:**
- `type` (אופציונלי): סינון לפי סוג כרטיס (`planned` | `breakdown`)
- `status` (אופציונלי): סינון לפי סטטוס

**דוגמה:**
```http
GET /accounts/acme/tickets?type=breakdown&status=open
```

**תגובה:**
```json
[
  {
    "id": "tkt_2025_001245",
    "type": "breakdown",
    "title": "Pump failure on Line 3",
    "priority": 1,
    "status": "open",
    "scheduled_date": "2025-09-14",
    "assetId": "asset_pmp_004",
    "created_at": "2025-09-14T13:40:00Z"
  }
]
```

#### יצירת כרטיס
```http
POST /accounts/{accountId}/tickets
```

**גוף הבקשה (Request Body):**
```json
{
  "ticket": {
    "type": "planned",
    "title": "500-hour pump service",
    "description": "Routine maintenance per schedule",
    "priority": 3,
    "status": "open",
    "scheduled_date": "2025-09-26",
    "assetId": "asset_pmp_004",
    "owner_user_id": "tu_acme_super1",
    "reported_by_user_id": "tu_acme_super1",
    "assignees": [
      { "user_id": "tu_acme_alex" }
    ],
    "tasks": [
      {
        "description": "Inspect coupling & alignment",
        "status": "pending"
      }
    ]
  }
}
```

**תגובה:** `201 Created`
```json
{
  "id": "tkt_2025_001245",
  "type": "planned",
  "title": "500-hour pump service",
  "status": "open",
  "created_at": "2025-09-26T06:55:00Z"
}
```

#### קבלת כרטיס
```http
GET /accounts/{accountId}/tickets/{ticketId}
```

**תגובה:** `200 OK`
```json
{
  "id": "tkt_2025_001245",
  "type": "planned",
  "title": "500-hour pump service",
  "description": "Routine maintenance per schedule",
  "priority": 3,
  "status": "open",
  "scheduled_date": "2025-09-26",
        "timeline": {
          "started": "2025-09-26T07:15:00Z",
          "ended": "2025-09-26T08:55:00Z",
          "duration_min": 100,
          "downtime_min": 30,
          "labor_entries": [
            {
              "user_id": "tu_acme_alex",
              "started": "2025-09-26T07:15:00Z",
              "ended": "2025-09-26T08:55:00Z",
              "duration_min": 100
            }
          ]
        },
  "tasks": [
    {
      "description": "Inspect coupling & alignment",
      "status": "DONE"
    }
  ],
  "assignees": [
    { "user_id": "tu_acme_alex" }
  ],
  "created_at": "2025-09-26T06:55:00Z",
  "updated_at": "2025-09-26T09:00:00Z"
}
```

#### עדכון כרטיס
```http
PATCH /accounts/{accountId}/tickets/{ticketId}
```

**גוף הבקשה:**
```json
{
  "status": "in_progress",
  "actual_work": {
    "started": "2025-09-26T07:15:00Z"
  },
  "tasks": [
    {
      "description": "Inspect coupling & alignment",
      "status": "done"
    }
  ]
}
```

**תגובה:** `200 OK`
```json
{
  "id": "tkt_2025_001245",
  "status": "in_progress",
  "actual_work": {
    "started": "2025-09-26T07:15:00Z"
  },
  "updated_at": "2025-09-26T09:30:00Z"
}
```

#### מחיקת כרטיס
```http
DELETE /accounts/{accountId}/tickets/{ticketId}
```

**תגובה:** `200 OK`
```json
{
  "success": true
}
```

## תגובות שגיאה (Error Responses)

### 400 Bad Request
```json
{
  "issues": [
    {
      "path": ["priority"],
      "message": "Number must be greater than or equal to 1"
    }
  ]
}
```

### 401 Unauthorized (לא מורשה)
```json
"Unauthorized"
```
*מוחזר כאשר האימות נכשל (טוקן לא חוקי או חסר)*

### 403 Forbidden (אסור)
```json
"Forbidden"
```
*מוחזר כאשר למשתמש אין גישה לחשבון*

### 404 Not Found (לא נמצא)
```json
"Ticket not found"
```

### 500 Internal Server Error (שגיאת שרת פנימית)
```json
"Error creating ticket"
```

## סולם עדיפויות (Priority Scale)

- **1** = חירום (עדיפות עליונה)
- **2** = גבוהה
- **3** = בינונית
- **4** = נמוכה
- **5** = נמוכה מאוד (העדיפות הנמוכה ביותר)

## זרימת סטטוסים (Status Flow)

### תחזוקה מתוכננת
```
draft → open → in_progress → completed → closed
```

### תחזוקת שבר
```
draft → open → in_progress → awaiting_approval → closed
```

## דוגמאות לשימוש

### יצירת כרטיס שבר
```javascript
const response = await fetch('/v1/accounts/acme/tickets', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer <firebase-token>',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    ticket: {
      type: 'breakdown',
      title: 'Pump failure on Line 3',
      description: 'Seal leak causing motor overload trip',
      priority: 1,
      status: 'open',
      assetId: 'asset_pmp_004',
      breakdown: {
        started: '2025-09-14T13:22:00Z',
        problem_description: 'Seal leak; pump tripped on motor overload.',
        root_cause: 'improper_alignment'
      },
      owner_user_id: 'tu_acme_super1',
      reported_by_user_id: 'usr_op_42',
      assignees: [
        { user_id: 'tu_acme_alex' }
      ]
    }
  })
});
```

### עדכון סטטוס כרטיס
```javascript
const response = await fetch('/v1/accounts/acme/tickets/tkt_123', {
  method: 'PATCH',
  headers: {
    'Authorization': 'Bearer <firebase-token>',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    status: 'in_progress',
    actual_work: {
      started: '2025-09-14T13:45:00Z'
    }
  })
});
```

### סינון כרטיסים
```javascript
// קבלת כל כרטיסי השבר
const response = await fetch('/v1/accounts/acme/tickets?type=breakdown');

// קבלת כרטיסים פתוחים בלבד
const response = await fetch('/v1/accounts/acme/tickets?status=open');

// קבלת כרטיסי שבר פתוחים
const response = await fetch('/v1/accounts/acme/tickets?type=breakdown&status=open');
```

## הערות לפיתוח פרונטאנד

1. **אימות (Authentication)**: כל הבקשות חייבות לכלול את טוקן ה-Firebase בכותרת ה-Authorization
2. **כתובות מבוססות חשבון**: כל פעולות הכרטיסים דורשות `accountId` בנתיב ה-URL
3. **שדות ספציפיים לסוג**: שדות מסוימים כמו `breakdown` רלוונטיים רק לכרטיסי שבר
4. **ניהול סטטוסים**: לסוגי כרטיסים שונים יש זרימות סטטוס שונות
5. **סינון**: השתמש בפרמטרי שאילתה כדי לסנן כרטיסים לפי סוג וסטטוס
6. **טיפול בשגיאות**: בדוק שגיאות 401/403 כדי לטפל בבעיות אימות/הרשאה
7. **פורמט תאריכים**: כל התאריכים הם בפורמט ISO 8601 (UTC)
8. **עדיפות**: מספרים נמוכים יותר מצביעים על עדיפות גבוהה יותר (1 = חירום)
9. **שדות תזמון**: 
   - **כרטיסים מתוכננים**: חייבים להכיל את `scheduled_date` (מתי התחזוקה מתוכננת)
   - **כרטיסי שבר**: חייבים להכיל את `breakdown.started` (מתי השבר התרחש), לא אמורים להכיל את `scheduled_date`
10. **שיוך נכסים (Assets)**: 
    - **כרטיסי שבר**: חייבים להכיל את השדה `assetId` (מזהה נכס)
    - **כרטיסים מתוכננים**: עשויים להכיל את השדה `assetId` (אופציונלי)
