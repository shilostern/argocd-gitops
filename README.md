| פרויקט GitOps: תשתית דקלרטיבית עם ArgoCD, Kind ותאימות Podman |
| :--- |
| פרויקט זה מדגים יישום של מתודולוגיית GitOps בסביבת עבודה מקומית. הפתרון משתמש ב-ArgoCD לניהול קונפיגורציות קובנרטיס. |
| --- |
| **1. הסבר על הפתרון** |
| הפתרון מבוסס על ניהול דקלרטיבי של אפליקציית whoami באמצעות ArgoCD ו-Kind על גבי Podman. |
| **2. מבנה ה-Repository** |
| 1. apps/whoami/base: מניפסטים בסיסיים המשותפים לכל הסביבות. |
| 2. apps/whoami/environments: הגדרות ספציפיות לסביבות פיתוח וייצור. |
| 3. argocd: הגדרות הסנכרון של ArgoCD. |
| **3. תהליך הפריסה** |
| שימוש בשיטת GitOps המבטיחה שקיפות מלאה ושחזור מהיר (Rollback) ע"י ניהול מצב רצוי בגיט. |
| **4. החלטות תכנוניות** |
| שימוש ב-Podman לתאימות וניהול גרסאות קשיח (ללא latest) ליציבות. |
| **5. תהליך הלמידה** |
| הלמידה התבססה על ניסוי וטעייה, קריאת תיעוד ושימוש ב-AI להבנת מושגי DevOps. |
| **6. גישת הלמידה** |
| התמקדות בפתרון בעיות ספציפיות (Problem-Solving) במקום קריאת תיעוד תיאורטי בלבד. |
| **7. אתגרים טכנולוגיים** |
| 1. בעיות רשת ב-Kind: הפתרון היה מעבר לעבודה Offline. |
| 2. מגבלות kind load: עבודה ישירה מול ה-Container Runtime. |
| 3. קפיאת Podman Machine: הפעלה מחדש של ה-Control Plane. |
| 4. התאמת שמות Images: עדכון ידני ב-Deployment. |
| **8. דוגמאות לפורמטים (Prompts)** |
| 1. ArgoCD shows 'OutOfSync', how do I debug? |
| 2. How to check if Kind can see local images? |
| 3. How to import .tar files to containerd? |
| **9. שיפורים עתידיים** |
| 1. Auto-Scaling עם HPA. |
| 2. CI/CD Pipeline אוטומטי. |
| **10. סביבת Production** |
| 1. שימוש ב-Private Container Registry מאובטח. |
| 2. ניטור עם Prometheus ו-Grafana. |
| **11. סיכונים** |
| 1. Single Point of Failure ב-Podman. |
| 2. חוסר אבטחה בניהול אימג'ים ידני. |
