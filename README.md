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

העתקנו את הקובץ לתוך קונטיינר ה-Control Plane, והזרקנו אותו ישירות לתוך ה-Namespace הפנימי של קובנרטיס ב-containerd באמצעות:

PowerShell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /whoami-latest.tar
האתגר האחרון: ה-Container Runtime רשם את האימג' תחת ה-Namespace המקומי שלו כ-localhost/traefik/whoami:latest. פודים שביקשו את השם המקורי המשיכו להיכשל.

התיקון בקוד: ביצענו התאמה (Realignment) מלאה בקוד המקור בגיט (בתוך ה-Manifest של ה-Deployment ב-Base):

YAML
image: localhost/traefik/whoami:latest
ברגע שהשינוי נדחף, ArgoCD זיהה את הקומיט, עדכן את ה-ReplicaSet, וקובנרטיס משך את האימג' לוקאלית בשבריר שנייה והביא את האפליקציה למצב Healthy & Running.

💡 תובנות מפתח (Key Takeaways)
עבודה בסביבת פודמן ווינדוס: דורשת לעיתים קרובות פתרונות מותאמים (כמו שימוש ב-ctr) מכיוון שכלי ה-Automated Loading של קלאסטרים מקומיים מותאמים בבסיסם ל-Docker Daemon הסטנדרטי.

עוצמת ה-GitOps (ארגו-CD): למרות כל הצרות והריסטארטים ברמת התשתית והמחשב, ברגע שהתשתית חזרה לעצמה והצהרנו על ה-Image הנכון ב-Git, המערכת תיקנה את עצמה אוטומטית (Self-Healing) והגיעה למצב הרצוי בלי אף התערבות ידנית בתוך קובנרטיס.

🛠️ הוראות הפעלה ותחזוקה מקומית
פתיחת פורט לארגו-CD
במידה והקשר לדפדפן ניתק עקב ריסטארט של פודמן, יש להרים מחדש את הצינור:

PowerShell
kubectl port-forward svc/argocd-server -n argocd 8080:443
עדכון גרסאות ואימג'ים ללא רשת חיצונית
אם ברצונכם להוסיף אימג' חדש לקלאסטר ללא תלות ברשת:

משכו אותו ללוקאל (podman pull <image>)

שמרו ל-Tar: podman save -o image.tar <image>

העתיקו לנוד: podman cp image.tar gitops-cluster-control-plane:/image.tar

הכניסו ל-Runtime: podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /image.tar

עדכנו את קובץ ה-deployment.yaml לשם המתאים עם הקידומת localhost/.
