# פרויקט GitOps עם ArgoCD, Kind ו-Podman

## 🎯 מטרת הפרויקט

בפרויקט זה הקמתי סביבת GitOps מלאה המבוססת על ArgoCD, Kind ו-Podman.

מטרת הפרויקט הייתה להעמיק את הידע שלי ב-Kubernetes ובמתודולוגיית GitOps, להבין לעומק את מנגנון ה-Reconciliation של ArgoCD ולהתמודד עם אתגרי תשתית ורשת בסביבת פיתוח מקומית.

---

## 🚀 ארכיטקטורת המערכת

המערכת מבוססת על עקרון ה-Single Source of Truth.

כל קובצי ה-Manifest מנוהלים ב-Git Repository, ו-ArgoCD אחראי לנטר את ה-Repository ולבצע Reconciliation אוטומטי מול ה-Cluster המקומי.

כל שינוי שנדחף ל-Git מסונכרן אוטומטית אל Kubernetes.

---

## 📑 אתגרים טכנולוגיים והתמודדות

במהלך הקמת סביבת ה-GitOps ופריסת האפליקציה `whoami-dev` באמצעות ArgoCD, נתקלתי במספר אתגרים משמעותיים הקשורים לרשת, ניהול Images ותאימות בין Podman, Kind ו-Kubernetes.

### 1. בעיות רשת ומשיכת Images

#### האתגר

לאחר הסנכרון הראשוני ב-ArgoCD, ה-Pods נכנסו למצבי:

```text
ImagePullBackOff
ErrImagePull
```

בדיקה באמצעות:

```bash
kubectl describe pod <pod-name>
```

הראתה כי ה-Container Runtime של Cluster ה-Kind לא הצליח לגשת ל-Docker Hub עקב בעיות DNS וקישוריות מתוך המכונה הווירטואלית.

#### הפתרון

ניסיתי לעבור ל-GitHub Container Registry (`ghcr.io`), אך גם פתרון זה נכשל.

מכאן הבנתי שה-Cluster מנותק לחלוטין מגישה חיצונית, ולכן החלטתי לעבור לתהליך עבודה Offline המבוסס על טעינת Images באופן ידני.

---

### 2. מגבלות kind load בסביבת Podman

#### האתגר

ניסיתי להשתמש בפקודת:

```bash
kind load docker-image
```

אך נתקלתי בבעיות תאימות בין Kind לבין Podman.

התקבלו שגיאות כגון:

```text
ERROR: unknown command "podman-image"
unknown flag: --name
```

בנוסף, Kind לא הצליח לזהות Images שהיו קיימים ב-Podman Storage גם לאחר הגדרת:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman
```

#### הפתרון

במקום להמשיך להשתמש במנגנון הטעינה של Kind, החלטתי לעבוד ישירות מול ה-Container Runtime של Kubernetes.

---

### 3. קפיאת Podman Machine וניתוק ה-API Server

#### האתגר

במהלך ניסיונות איפוס רשת, Podman Machine נכנסה למצב לא תקין.

מצד אחד התקבלה ההודעה:

```text
already running
```

ומצד שני Kubernetes API Server הפסיק להגיב.

בנוסף, פעולות Port Forward ל-ArgoCD נכשלו עם השגיאה:

```text
connection refused
```

#### הפתרון

הפעלתי מחדש את Container ה-Control Plane:

```powershell
podman start gitops-cluster-control-plane
```

לאחר מכן ה-API Server חזר לפעול באופן תקין ויכולתי להפעיל מחדש Port Forward אל ArgoCD.

---

### 4. טעינת Image ישירות ל-containerd והתאמת שמות

#### האתגר

כדי לעקוף לחלוטין את התלות בגישה לאינטרנט, ייצאתי את ה-Image מהמחשב המקומי:

```powershell
podman save -o whoami-latest.tar traefik/whoami:latest
```

לאחר מכן העתקתי את הקובץ אל Container ה-Control Plane:

```powershell
podman cp whoami-latest.tar gitops-cluster-control-plane:/whoami-latest.tar
```

ולאחר מכן ייבאתי אותו ישירות אל containerd:

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /whoami-latest.tar
```

לאחר הייבוא גיליתי שה-Image נרשם בשם:

```text
localhost/traefik/whoami:latest
```

בעוד שה-Deployment עדיין הפנה אל:

```text
traefik/whoami:latest
```

כתוצאה מכך ה-Pods המשיכו להיכשל.

#### הפתרון

עדכנתי את שדה ה-image ב-Deployment:

```yaml
image: localhost/traefik/whoami:latest
```

לאחר ביצוע Commit ו-Push ל-Repository, ArgoCD זיהה את השינוי, עדכן את ה-ReplicaSet ופרס מחדש את האפליקציה.

בתוך זמן קצר האפליקציה עברה למצב:

```text
Healthy
Running
```

---

## 💡 תובנות מרכזיות

### עבודה עם Podman בסביבת Windows

השילוב של Podman, Kind ו-Windows חשף אותי לאתגרים שאינם קיימים תמיד בסביבות Docker סטנדרטיות.

במקרים מסוימים נדרשתי לעבוד ישירות מול כלים ברמה נמוכה יותר, כגון `ctr`, במקום להסתמך על מנגנוני האוטומציה המובנים.

### היתרון של GitOps

אחד הדברים המרשימים ביותר שחוויתי בפרויקט היה מנגנון ה-Self-Healing של ArgoCD.

למרות תקלות תשתית, ניתוקי רשת ואתחולים של הסביבה המקומית, ברגע שהמצב הרצוי הוגדר ב-Git, ArgoCD דאג להחזיר את המערכת למצב תקין באופן אוטומטי.

---

## 🛠️ הוראות הפעלה ותחזוקה

### פתיחת Port Forward ל-ArgoCD

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### טעינת Image חדש ללא גישה לאינטרנט

#### 1. משיכת Image למחשב המקומי

```powershell
podman pull <image>
```

#### 2. שמירת Image לקובץ TAR

```powershell
podman save -o image.tar <image>
```

#### 3. העתקת קובץ ה-TAR אל ה-Control Plane

```powershell
podman cp image.tar gitops-cluster-control-plane:/image.tar
```

#### 4. ייבוא ה-Image אל containerd

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /image.tar
```

#### 5. עדכון Deployment

```yaml
image: localhost/<image>
```

---

## ✅ סטטוס הפרויקט

| רכיב | סטטוס |
|-------|--------|
| ArgoCD | פעיל |
| Kind Cluster | פעיל |
| Podman | פעיל |
| GitOps Synchronization | פעיל |
| whoami-dev | נפרס בהצלחה |
| Image Loading | מקומי ללא Docker Hub |
| Application Health | Healthy |
| Synchronization Status | Synced |

---

פרויקט זה סיפק לי ניסיון מעשי בעבודה עם Kubernetes, GitOps, ArgoCD, Podman, פתרון תקלות תשתית וניתוח בעיות רשת בסביבת פיתוח מקומית.
