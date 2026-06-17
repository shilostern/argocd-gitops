פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ותאימות Podman
פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית. הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות קובנרטיס, כאשר כל משאבי המערכת מנוהלים כקוד (IaC) ב-Repository זה.

1. הסבר על הפתרון
הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami. תהליך העבודה כלל הקמת קלאסטר Kubernetes מקומי באמצעות Kind, הגדרת ArgoCD כקונטרולר לסנכרון, ופריסת האפליקציה תוך שימוש במבנה של Base ו-Overlays.

2. מבנה ה-Repository
apps/whoami/base: מכיל את המניפסטים הבסיסיים המשותפים לכל הסביבות.

apps/whoami/environments: מכיל הגדרות ספציפיות לכל סביבה (פיתוח/ייצור).

argocd: מכיל את קבצי ה-Application המגדירים ל-ArgoCD מה לסנכרן.

3. תהליך הפריסה
נבחרה שיטת ה-GitOps כיוון שהיא מבטיחה שקיפות מוחלטת (מה שכתוב בגיט הוא מה שרץ בפועל) ומאפשרת שחזור מהיר (Rollback) ע"י חזרה לקומיט קודם.

4. החלטות תכנוניות מרכזיות
ההחלטות המרכזיות כללו שימוש ב-Podman במקום Docker כדי להבטיח תאימות סביבתית, וניהול גרסאות עם Image Tags קשיחים כדי להבטיח יציבות בסביבות שונות.

5. תהליך הלמידה
הלמידה הייתה תהליך הדרגתי שהתבסס על קריאת תיעוד, התמודדות עם שגיאות רשת ו-Runtime, ושימוש ב-AI כשותף להבנת מושגי DevOps מורכבים.

6. כיצד ניגשתי ללמידה?
ניגשתי ללמידה כ-"Problem-Solving Journey". התמקדתי בפתרון שגיאות ספציפיות דרך פירוק פקודות מורכבות בעזרת AI, במקום ללמוד את כל התיעוד מאפס.

7. אתגרים טכנולוגיים והתמודדות
בעיות רשת: ה-Pods נכנסו למצב ImagePullBackOff כי הקלאסטר היה מנותק מהאינטרנט. הפתרון היה מעבר לעבודה Offline.

מגבלות kind load: נתקלתי בבעיות תאימות בין Kind ל-Podman. הפתרון היה עבודה ישירה מול ה-Container Runtime.

קפיאת Podman Machine: קריסת ה-Control Plane דרשה הפעלה מחדש של ה-Container.

התאמת שמות Images: ייבוא ידני יצר שמות כגון localhost/..., מה שחייב עדכון ידני ב-Deployment.

8. דוגמאות לפורמטים (Prompts) לשימוש ב-AI
ArgoCD shows 'OutOfSync', how do I debug the generated manifest?

I am getting ImagePullBackOff, how do I check if my container runtime in Kind can see my local images?

How can I import a .tar file directly into Kind's containerd runtime?

9. שיפורים עתידיים
Auto-Scaling: הוספת HPA להתאמת מספר הפודים לעומס.

CI/CD Pipeline: שילוב תהליך אוטומטי לבניית אימג'ים.

10. סביבת Production
Registry חיצוני: שימוש ב-Private Container Registry מאובטח.

ניטור: הוספת Prometheus ו-Grafana לניטור תקינות הפודים.

11. סיכונים בפתרון הנוכחי
Single Point of Failure: תלות ב-Podman Machine המקומית.

חוסר אבטחה: ניהול אימג'ים ידני ללא סריקת פגיעות.
