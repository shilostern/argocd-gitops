<div dir="rtl" style="text-align: right;">

# פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ו-Podman

פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית. הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות Kubernetes, כאשר כל משאבי המערכת מנוהלים כקוד (IaC) ב-Repository זה.

---

## 1. הסבר על הפתרון

הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami.

תהליך העבודה כלל:
- הקמת קלאסטר Kubernetes מקומי באמצעות Kind (על גבי Podman)
- הגדרת ArgoCD כקונטרולר סנכרון
- פריסת האפליקציה באמצעות מבנה Base ו-Overlays (Kustomize)

כל שינוי ב-Repository מתורגם אוטומטית למצב הרצוי בקלאסטר באמצעות מנגנון ה-Reconciliation של ArgoCD.

---

## 2. מבנה ה-Repository

ה-Repo בנוי לפי עקרונות Kustomize:

- `apps/whoami/base`  
  מניפסטים בסיסיים (Deployment, Service) לכל הסביבות

- `apps/whoami/environments`  
  קונפיגורציות ייעודיות לסביבות (פיתוח / ייצור)

- `argocd`  
  קבצי Application של ArgoCD לסנכרון המשאבים

**למה מבנה זה?**  
כדי לאפשר עקרון DRY: שינוי ב-base משפיע על כל הסביבות, בעוד Overlays מאפשרים התאמות בלי שכפול קוד.

---

## 3. תהליך הפריסה

נבחר GitOps כי הוא מאפשר:
- שקיפות מלאה (מה שבגיט = מה שרץ)
- יכולת Rollback מהירה

תהליך העבודה:
- Push ל-Repo
- ArgoCD מזהה שינוי
- סנכרון אוטומטי לקלאסטר
- ללא שימוש ישיר ב-kubectl

---

## 4. החלטות תכנוניות מרכזיות

- שימוש ב-Podman במקום Docker  
  התאמות נדרשו ברמת runtime

- שימוש ב-Image Tags גרסאיים  
  הימנעות מ-latest לצורך יציבות

---

## 5. תהליך הלמידה

הלמידה בוצעה בצורה הדרגתית:
- קריאה בתיעוד ArgoCD ו-Kind
- ניסוי וטעייה בסביבת עבודה אמיתית
- שימוש ב-AI להבנת מושגים ופתרון בעיות CLI

---

## 6. גישת הלמידה

גישה של Problem Solving:
- הבנת workflow דרך דוגמאות
- שימוש ב-AI לפירוק בעיות מורכבות
- חקירה ממוקדת של שגיאות בפועל

---

## 7. אתגרים טכנולוגיים והתמודדות

### 7.1 בעיות רשת ומשיכת Images

**בעיה:**  
ImagePullBackOff עקב חוסר גישה ל-Docker Hub מתוך הקלאסטר

**פתרון:**  
מעבר לעבודה Offline וטעינת Images ידנית

---

### 7.2 מגבלות kind load עם Podman

**בעיה:**  
חוסר תאימות בין Kind ל-Podman בטעינת Images

**פתרון:**  
עבודה ישירה מול container runtime

---

### 7.3 קפיאת Podman Machine

**בעיה:**  
ניתוק של API Server וקריסת הקלאסטר

**פתרון:**  
הפעלת control plane מחדש באמצעות Podman

---

### 7.4 טעינת Image ל-containerd והתאמת שמות

**בעיה:**  
חוסר התאמה בשם ה-Image בין deployment ל-import

**פתרון:**  
עדכון ה-Deployment כך שיתאים ל-image המקומי ולאחר מכן redeploy דרך ArgoCD

---

## 8. פרומפטים לשימוש ב-AI

דוגמאות לשאלות עבודה:

- ArgoCD shows OutOfSync but manifests look correct. How to debug?
- How to check ImagePullBackOff in Kind?
- Why use Kustomize instead of single YAML?
- How to import tar image into containerd offline?

---

## 9. שיפורים עתידיים

- Auto Scaling עם HPA
- CI/CD עם GitHub Actions
- בניית אימג'ים אוטומטית ועדכון repo

---

## 10. סביבת Production

- שימוש ב-Private Registry
- הוספת Monitoring עם Prometheus + Grafana

---

## 11. סיכונים בפתרון הנוכחי

- Single Point of Failure (תלות ב-Podman Machine)
- חוסר סריקת אבטחה לאימג'ים

---

</div>
