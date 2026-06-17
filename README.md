<div dir="rtl" align="right">

# פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ו-Podman

פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית.  
הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות Kubernetes, כאשר כל משאבי המערכת מנוהלים כקוד (IaC) בריפוזיטורי זה.

---

## 1. הסבר על הפתרון

הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami.  
תהליך העבודה כלל הקמת קלאסטר Kubernetes מקומי באמצעות Kind (על גבי Podman), הגדרת ArgoCD כסנכרון אוטומטי, ופריסת האפליקציה באמצעות Kustomize (Base + Overlays).

כל שינוי ב-Repository מתורגם אוטומטית למצב הרצוי בקלאסטר באמצעות מנגנון Reconciliation של ArgoCD.

---

## 2. מבנה ה-Repository

הריפוזיטורי בנוי לפי עקרונות Kustomize:

<p align="right">

**apps/whoami/base**  
מניפסטים בסיסיים המשותפים לכל הסביבות.

**apps/whoami/environments**  
קונפיגורציות לפי סביבות (פיתוח / ייצור).

**argocd**  
הגדרות Application עבור ArgoCD.

</p>

**למה זה בנוי כך?**  
גישה זו מונעת כפילות קוד (DRY) ומאפשרת הפרדה ברורה בין בסיס לשינויים סביבתיים.

---

## 3. תהליך הפריסה

GitOps מבטיח התאמה מלאה בין Git לבין הקלאסטר.

<p align="right">

דחיפה ל-Git → זיהוי ע"י ArgoCD → סנכרון אוטומטי → עדכון קלאסטר

</p>

אין צורך בהרצת kubectl ידנית.

---

## 4. החלטות תכנוניות

- שימוש ב-Podman במקום Docker לשמירה על סביבת עבודה אחידה  
- שימוש ב-Image Tags קבועים (לא latest) ליציבות בין סביבות  

---

## 5. תהליך הלמידה

הלמידה התבצעה באופן הדרגתי ומעשי:
- קריאת תיעוד רשמי
- ניסוי וטעייה בזמן אמת
- שימוש ב-AI לפירוק בעיות DevOps מורכבות

---

## 6. גישת העבודה

גישה של Problem-Solving Journey:

<p align="right">
פירוק בעיות קטנות → הבנה דרך שגיאות → שימוש ב-AI ככלי ניתוח
</p>

---

## 7. אתגרים טכנולוגיים

<p align="right">

**ImagePullBackOff**  
בעיה של חוסר גישה לרשת → מעבר ל-Offline images

**Kind + Podman תאימות**  
פתרון באמצעות עבודה ישירה מול container runtime

**קריסות Podman Machine**  
Restart ל-control plane וחיבור מחדש ל-API server

**שמות Images לא עקביים**  
עדכון ידני ל-Deployment

</p>

---

## 8. פרומפטים לפתרון בעיות

- ArgoCD shows 'OutOfSync', how do I debug the manifest?
- ImagePullBackOff in Kind – how to verify local images?
- How to import .tar image into containerd in Kind?

---

## 9. שיפורים עתידיים

- HPA (Auto Scaling)
- CI/CD עם GitHub Actions
- בניית pipeline אוטומטי ל-images

---

## 10. סביבת Production

- שימוש ב-Private Registry
- ניטור עם Prometheus + Grafana

---

## 11. סיכונים

- תלות ב-Podman Machine (Single Point of Failure)
- חוסר סריקת אבטחה ל-images

---

</div>
