<div dir="rtl" align="right">

# פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ותאימות Podman

פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית. הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות קובנרטיס, כאשר כל משאבי המערכת מנוהלים כקוד (IaC) ב-Repository זה.

---

## 1. הסבר על הפתרון
הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami. תהליך העבודה כלל הקמת קלאסטר Kubernetes מקומי באמצעות Kind (על גבי מנוע Podman), הגדרת ArgoCD כקונטרולר לסנכרון, ופריסת האפליקציה תוך שימוש במבנה של Base ו-Overlays (Kustomize). כל שינוי שבוצע ב-Repository תורגם אוטומטית למצב הרצוי בקלאסטר על ידי מנגנון ה-Reconciliation של ArgoCD.

## 2. מבנה ה-Repository
ה-Repo תוכנן לפי עקרונות ה-Kustomize, המאפשרים הפרדה בין בסיס הקוד (Base) לבין התאמות לסביבות (Overlays):

apps/whoami/base: מכיל את המניפסטים הבסיסיים המשותפים לכל הסביבות.
apps/whoami/environments: מכיל הגדרות ספציפיות לכל סביבה (פיתוח/ייצור).
argocd: מכיל את קבצי ה-Application המגדירים ל-ArgoCD מה לסנכרן.

למה בחרנו במבנה זה? כדי לאפשר גמישות (DRY - Don't Repeat Yourself). שינוי בבסיס משפיע על כל הסביבות, בעוד ששינויים ב-Overlays מאפשרים קונפיגורציה ייחודית ללא כפילות קוד.

## 3. תהליך הפריסה
נבחרה שיטת ה-GitOps כיוון שהיא מבטיחה שקיפות מוחלטת (מה שכתוב בגיט הוא מה שרץ בפועל) ומאפשרת שחזור מהיר (Rollback) ע"י חזרה לקומיט קודם. תהליך הפריסה מבוסס על "דחיפת" שינויים ל-Repo, זיהוי אוטומטי על ידי ArgoCD, והחלתם על הקלאסטר ללא התערבות ידנית ב-kubectl.

## 4. החלטות תכנוניות מרכזיות
ההחלטות המרכזיות כללו שימוש ב-Podman במקום Docker כדי להבטיח תאימות סביבתית, וניהול גרסאות עם Image Tags קשיחים (לא Latest) כדי להבטיח יציבות בסביבות שונות.

## 5. תהליך הלמידה
הלמידה הייתה תהליך הדרגתי ומעשי שהתבסס על קריאת תיעוד רשמי של ArgoCD ו-Kind, התמודדות עם שגיאות רשת ו-Runtime, ושימוש ב-AI כשותף להבנת מושגי DevOps מורכבים בזמן אמת.

## 6. כיצד ניגשתי ללמידה?
ניגשתי ללמידה כ-"Problem-Solving Journey". התמקדתי בפתרון שגיאות ספציפיות דרך פירוק פקודות מורכבות (כמו kubectl port-forward) בעזרת AI, במקום ללמוד את כל התיעוד מאפס.

## 7. אתגרים טכנולוגיים והתמודדות
במהלך הקמת סביבת ה-GitOps ופריסת האפליקציה whoami, נתקלתי במספר אתגרים:
1. בעיות רשת: ה-Pods נכנסו למצב ImagePullBackOff כי הקלאסטר היה מנותק מהאינטרנט. הפתרון היה מעבר לעבודה Offline עם טעינת Images ידנית.
2. מגבלות kind load: נתקלתי בבעיות תאימות בין Kind ל-Podman. הפתרון היה עבודה ישירה מול ה-Container Runtime של Kubernetes.
3. קפיאת Podman Machine: קריסת ה-Control Plane דרשה הפעלה מחדש של ה-Container וחיבור מחדש של ה-API Server.
4. התאמת שמות Images: ייבוא ידני יצר שמות כגון localhost/..., מה שחייב עדכון ידני ב-Deployment.

## 8. דוגמאות לפורמטים (Prompts) לשימוש ב-AI
השאלות שסייעו לי לפצח את האתגרים כללו את:
ArgoCD shows 'OutOfSync', how do I debug the generated manifest?
I am getting ImagePullBackOff, how do I check if my container runtime in Kind can see my local images?
How can I import a .tar file directly into Kind's containerd runtime?

## 9. שיפורים עתידיים
Auto-Scaling: הוספת HPA להתאמת מספר הפודים לעומס.
CI/CD Pipeline: שילוב תהליך אוטומטי לבניית אימג'ים (GitHub Actions).

## 10. סביבת Production
בסביבת Prod הייתי משנה: שימוש ב-Private Container Registry מאובטח והוספת ניטור (Prometheus ו-Grafana).

## 11. סיכונים בפתרון הנוכחי
Single Point of Failure: תלות ב-Podman Machine המקומית.
חוסר אבטחה: ניהול אימג'ים ידני ללא סריקת פגיעות.

</div>
