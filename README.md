<div dir="rtl" align="right">

# פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ו-Podman

פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית. הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות קובנרטיס, כאשר כל משאבי המערכת מנוהלים כקוד (IaC) ב-Repository זה.

---

## 1. הסבר על הפתרון
הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami. תהליך העבודה כלל הקמת קלאסטר Kubernetes מקומי באמצעות Kind (על גבי מנוע Podman), הגדרת ArgoCD כקונטרולר לסנכרון, ופריסת האפליקציה תוך שימוש במבנה של Base ו-Overlays (Kustomize). כל שינוי שבוצע ב-Repository תורגם אוטומטית למצב הרצוי בקלאסטר על ידי מנגנון ה-Reconciliation של ArgoCD.

## 2. מבנה ה-Repository
ה-Repo תוכנן לפי עקרונות ה-Kustomize, המאפשרים הפרדה בין בסיס הקוד (Base) לבין התאמות לסביבות (Overlays):
* **apps/whoami/base:** מכיל את המניפסטים הבסיסיים (Deployment, Service) המשותפים לכל הסביבות.
* **apps/whoami/environments:** מכיל הגדרות ספציפיות לכל סביבה (פיתוח/ייצור).
* **argocd:** מכיל את קבצי ה-Application המגדירים ל-ArgoCD מה לסנכרן ומאיפה.
למה בחרנו במבנה זה? כדי לאפשר גמישות (DRY - Don't Repeat Yourself). שינוי בבסיס משפיע על כל הסביבות, בעוד ששינויים ב-Overlays מאפשרים קונפיגורציה ייחודית (כמו כמות Replicas) ללא כפילות קוד.

## 3. תהליך הפריסה
נבחרה שיטת ה-GitOps כיוון שהיא מבטיחה שקיפות מוחלטת (מה שכתוב בגיט הוא מה שרץ בפועל) ומאפשרת שחזור מהיר (Rollback) ע"י חזרה לקומיט קודם. תהליך הפריסה מבוסס על "דחיפת" שינויים ל-Repo, זיהוי אוטומטי על ידי ArgoCD, והחלתם על הקלאסטר ללא התערבות ידנית ב-kubectl.

## 4. החלטות תכנוניות מרכזיות
* **שימוש ב-Podman במקום Docker:** החלטה מבוססת תאימות סביבתית, שדרשה התאמות ב-Runtime של ה-Container.
* **ניהול גרסאות עם Image Tags:** בחירה בשימוש בגרסאות קשיחות במקום latest כדי להבטיח יציבות בסביבות שונות.

## 5. תהליך הלמידה
הלמידה הייתה תהליך הדרגתי ומעשי:
* קריאה: תיעוד רשמי של ArgoCD ו-Kind.
* ניסוי וטעייה: התמודדות עם שגיאות רשת ו-Runtime.
* AI כשותף: השימוש ב-AI (כמו ChatGPT/Gemini) אפשר לי להבין מושגי DevOps מורכבים בזמן אמת, תוך הבנת הקשר שבין שגיאות CLI לפתרונות ארכיטקטוניים.

## 6. כיצד ניגשתי ללמידה?
ניגשתי ללמידה כ-"Problem-Solving Journey". במקום ללמוד את כל התיעוד מאפס, התמקדתי בפתרון שגיאות ספציפיות:
* הבנת ה-Workflow של ArgoCD דרך צפייה בסרטוני הדגמה.
* שימוש ב-AI לפירוק פקודות מורכבות: "מה עושה הפקודה kubectl port-forward?" או "איך אני טוען אימג' ידנית ל-containerd?".

## 7. אתגרים טכנולוגיים והתמודדות
במהלך הקמת סביבת ה-GitOps ופריסת האפליקציה whoami באמצעות ArgoCD, נתקלתי במספר אתגרים משמעותיים הקשורים לרשת, ניהול Images ותאימות בין Podman, Kind ו-Kubernetes.

**1. בעיות רשת ומשיכת Images**
* האתגר: לאחר הסנכרון הראשוני ב-ArgoCD, ה-Pods נכנסו למצבי `ImagePullBackOff` ו-`ErrImagePull`. בדיקה הראתה כי ה-Container Runtime לא הצליח לגשת ל-Docker Hub עקב בעיות קישוריות.
* הפתרון: מכיוון שה-Cluster מנותק לחלוטין, עברתי לתהליך עבודה Offline המבוסס על טעינת Images ידנית.

**2. מגבלות kind load בסביבת Podman**
* האתגר: נתקלתי בבעיות תאימות בין Kind לבין Podman בשימוש ב-`kind load docker-image`.
* הפתרון: עבודה ישירה מול ה-Container Runtime של Kubernetes.

**3. קפיאת Podman Machine וניתוק ה-API Server**
* האתגר: קריסת ה-Control Plane וכישלון של Port Forward ל-ArgoCD.
* הפתרון: הפעלה מחדש של ה-Container המנהל וחיבור מחדש של ה-API Server.

**4. טעינת Image ישירות ל-containerd והתאמת שמות**
* האתגר: אי-התאמה בין השם שה-Image מקבל בייבוא (`localhost/...`) לבין השם ב-Deployment.
* הפתרון: עדכון שדה ה-image ב-Deployment ל-`localhost/traefik/whoami:latest`.

## 8. דוגמאות לפורמטים (Prompts) לשימוש ב-AI
* "ArgoCD shows 'OutOfSync' but the Kustomize base and overlay seem correct. How do I debug the generated manifest?"
* "I am getting ImagePullBackOff, how do I check if my container runtime in Kind can see my local images?"
* "Explain the Kustomize directory structure—why can't I just use a single deployment.yaml?"
* "How can I import a .tar file directly into Kind's containerd runtime when the cluster has no internet access?"

## 9. שיפורים עתידיים
* **Auto-Scaling:** הוספת HPA כדי שהמערכת תתאים את מספר הפודים לעומס בזמן אמת.
* **CI/CD Pipeline:** שילוב תהליך אוטומטי שבונה אימג'ים (GitHub Actions) ומעדכן את ה-Repo אוטומטית.

## 10. סביבת Production
בסביבת Prod הייתי משנה:
* **Registry חיצוני:** שימוש ב-Private Container Registry מאובטח.
* **ניטור (Monitoring):** הוספת Prometheus ו-Grafana לניטור תקינות הפודים.

## 11. סיכונים בפתרון הנוכחי
* **Single Point of Failure:** תלות ב-Podman Machine המקומית; קריסתה משביתה את הקלאסטר.
* **חוסר אבטחה:** ניהול האימג'ים באופן ידני ללא סריקת פגיעות.

</div>
