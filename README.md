<div dir="rtl">

# פרויקט GitOps עם ArgoCD, Kind ו-Podman

פרויקט זה מציג ארכיטקטורת GitOps מלאה לסביבת פיתוח מקומית (Local Development), המבוססת על **ArgoCD** לניהול וסנכרון דקלרטיבי של משאבי קובנרטיס, על גבי קלאסטר **Kind** המורץ במנוע הקונטיינרים **Podman** בסביבת Windows.

## 🚀 ארכיטקטורת המערכת
המערכת מבוססת על עקרון ה-Single Source of Truth, שבו כל קבצי המניפסט (Deployment, Service וכדומה) מנוהלים בריפו זה בגיט. ArgoCD מנטר את הריפו באופן קבוע ומבצע קונסילציה (Reconciliation) אוטומטית מול הקלאסטר המקומי.

---

## 📑 פרק: אתגרים טכנולוגיים והתמודדות (Troubleshooting & Engineering Challenges)

במהלך הקמת פרויקט ה-GitOps ופריסת האפליקציה `whoami-dev` באמצעות **ArgoCD**, נתקלנו במספר אתגרי אינטגרציה ורשת מורכבים בין כלי ה-DevOps השונים (Windows Host, Podman, Kind, ו-containerd). 

להלן פירוט האתגרים המרכזיים והדרך בה הם נפתרו:

### 1. ניתוק רשת וחסימת משיכת אימג'ים (ImagePullBackOff)
* **האתגר:** לאחר הסנכרון הראשוני ב-ArgoCD, הפודים נכנסו למצב תקוע של `ImagePullBackOff` / `ErrImagePull`. בדיקה מעמיקה (בעזרת `kubectl describe`) העלתה כי ה-Container Runtime של קלאסטר ה-Kind המקומי חסום לחלוטין מול שרתים חיצוניים, ולא הצליח לפנות ל-Docker Hub בשל בעיות קישוריות ו-DNS פנימיות של המכונה הווירטואלית.
* **הפתרון:** ניסיון ראשון לעקוף את החסימה על ידי מעבר ל-Registry חלופי ומבוזר (**GitHub Container Registry - ghcr.io**) נכשל אף הוא מאותה סיבת רשת, מה שהוכיח שהקלאסטר מנותק חיצונית לחלוטין. הוחלט לעבור לאסטרטגיית פיתוח אופליין (Offline Workflow) המבוססת על הזרקת אימג'ים מקומיים.

### 2. כשל בכלי הניהול המקומיים (Kind Load vs. Podman Storage)
* **האתגר:** ניסינו להשתמש בכלי המובנה `kind load` כדי להזריק את האימג' שנמשך בהצלחה למחשב המארח (Host) ישירות לתוך הקלאסטר. נתקלנו בשגיאות תאימות חמורות של ה-CLI (`ERROR: unknown command "podman-image"` ו-`unknown flag: --name`), וכן בחוסר יכולת של Kind לזהות את מזהה האימג' (Image Tag) בתוך ה-Storage הפנימי של פודמן, למרות שימוש במשתני סביבה ייעודיים כמו `KIND_EXPERIMENTAL_PROVIDER="podman"`.
* **הפתרון:** במקום להמשיך "להילחם" בשכבת התיווך של ה-CLI, עקפנו את הבעיה על ידי עבודה ישירה מול ה-Container Runtime הפנימי של קובנרטיס.

### 3. קפיאת המנוע הווירטואלי (Podman Machine Refused Connection)
* **האתגר:** במהלך ניסיונות איחול הרשת (Network Reset) של ה-Podman Machine, המכונה הווירטואלית נכנסה למצב קפוא (Deadlock) – מצד אחד דיווחה שהיא כבר רצה (`already running`), ומצד שני חסמה את הגישה ל-Kubernetes API Server והפילה את ה-Port Forward של ArgoCD עם שגיאת `connection refused`.
* **הפתרון:** בוצע אילוץ הנעה מחדש (Forced Start) ברמת הקונטיינר של ה-Control Plane באמצעות פודמן (`podman start gitops-cluster-control-plane`), מה ששיחרר את ה-API Server, ופתח מחדש את הצינור לארגו-CD (`kubectl port-forward`).

### 4. הזרקה ידנית עמוקה (Low-Level Containerd Import) ותיאום שמות (Naming Realignment)
* **האתגר המכריע והפתרון הסופי:** כדי לפתור את בעיית האינטרנט לצמיתות, ביצענו ייצוא פיזי של ה-Image לקובץ ארכיון במחשב המארח:
  ```powershell
  podman save -o whoami-latest.tar traefik/whoami:latest
