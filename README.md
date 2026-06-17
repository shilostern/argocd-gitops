# פרויקט GitOps עם ArgoCD, Kind ו-Podman

## 🎯 מטרת הפרויקט

בפרויקט זה הקמתי סביבת GitOps מלאה המבוססת על ArgoCD, Kind ו-Podman.

מטרת הפרויקט הייתה להעמיק את ההיכרות שלי עם Kubernetes ועם עקרונות GitOps, להבין כיצד ArgoCD מנהל את מצב המערכת באופן דקלרטיבי, ולהתמודד עם אתגרי תשתית ורשת בסביבת פיתוח מקומית המבוססת על Windows.

---

## 🚀 ארכיטקטורת המערכת

המערכת מבוססת על עקרון ה־Single Source of Truth, שבו כל קובצי ה־Manifest (כגון Deployment ו-Service) מנוהלים ב־Git Repository.

ArgoCD מנטר באופן רציף את ה־Repository ומבצע Reconciliation אוטומטי מול ה־Cluster המקומי. כל שינוי שנדחף ל־Git מסונכרן באופן אוטומטי אל סביבת Kubernetes.

---

## 📑 אתגרים טכנולוגיים והתמודדות

במהלך הקמת סביבת ה־GitOps ופריסת האפליקציה `whoami-dev` באמצעות ArgoCD, נתקלתי במספר אתגרי אינטגרציה ורשת בין Windows, Podman, Kind ו־containerd.

להלן האתגרים המרכזיים והדרך שבה פתרתי אותם.

---

### 1. בעיות רשת ומשיכת Images (`ImagePullBackOff`)

#### האתגר

לאחר הסנכרון הראשוני ב־ArgoCD, ה־Pods נכנסו למצבי `ImagePullBackOff` ו־`ErrImagePull`.

בדיקה באמצעות:

```bash
kubectl describe pod <pod-name>
```

העלתה כי ה־Container Runtime של Cluster ה־Kind לא הצליח לגשת ל־Docker Hub עקב בעיות DNS וקישוריות מתוך המכונה הווירטואלית.

#### הפתרון

תחילה ניסיתי לעבור ל־GitHub Container Registry (`ghcr.io`), אך גם פתרון זה נכשל מאותה סיבה.

בשלב זה הגעתי למסקנה שה־Cluster מנותק לחלוטין מגישה חיצונית, ולכן החלטתי לעבור לתהליך עבודה Offline המבוסס על טעינת Images באופן ידני.

---

### 2. מגבלות `kind load` בסביבת Podman

#### האתגר

ניסיתי להשתמש בפקודת:

```bash
kind load docker-image
```

כדי לטעון Images ישירות אל ה־Cluster.

הפקודה נכשלה עקב בעיות תאימות בין Kind לבין Podman, והתקבלו שגיאות כגון:

```text
ERROR: unknown command "podman-image"
unknown flag: --name
```

בנוסף, Kind לא הצליח לזהות Images שהיו קיימים ב־Podman Storage, גם לאחר הגדרת:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman
```

#### הפתרון

במקום להמשיך להשתמש במנגנון הטעינה של Kind, החלטתי לעבוד ישירות מול ה־Container Runtime של Kubernetes.

---

### 3. קפיאת Podman Machine וניתוק ה־API Server

#### האתגר

במהלך ניסיונות איפוס רשת, Podman Machine נכנסה למצב לא תקין.

מצד אחד התקבלה ההודעה:

```text
already running
```

ומצד שני Kubernetes API Server הפסיק להגיב.

בנוסף, פעולות Port Forward ל־ArgoCD נכשלו עם השגיאה:

```text
connection refused
```

#### הפתרון

הפעלתי מחדש את Container ה־Control Plane:

```powershell
podman start gitops-cluster-control-plane
```

לאחר מכן ה־API Server חזר לפעול באופן תקין ויכולתי להפעיל מחדש Port Forward אל ArgoCD.

---

### 4. טעינת Image ישירות ל־containerd והתאמת שמות

#### האתגר

כדי לעקוף לחלוטין את התלות בגישה לאינטרנט, ייצאתי את ה־Image מהמחשב המקומי:

```powershell
podman save -o whoami-latest.tar traefik/whoami:latest
```

לאחר מכן העתקתי את הקובץ אל Container ה־Control Plane:

```powershell
podman cp whoami-latest.tar gitops-cluster-control-plane:/whoami-latest.tar
```

ולאחר מכן ייבאתי אותו ישירות אל containerd:

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /whoami-latest.tar
```

לאחר הייבוא גיליתי שה־Image נרשם בשם:

```text
localhost/traefik/whoami:latest
```

בעוד שה־Deployment עדיין הפנה אל:

```text
traefik/whoami:latest
```

כתוצאה מכך ה־Pods המשיכו להיכשל.

#### הפתרון

עדכנתי את שדה ה־`image` ב־Deployment:

```yaml
image: localhost/traefik/whoami:latest
```

לאחר ביצוע Commit ו־Push ל־Repository, ArgoCD זיהה את השינוי, עדכן את ה־ReplicaSet ופרס מחדש את האפליקציה.

בתוך זמן קצר האפליקציה עברה למצב:

```text
Healthy
Running
```

---

## 💡 תובנות מרכזיות

### עבודה עם Podman בסביבת Windows

העבודה עם Podman, Kind ו־Windows חשפה אותי לאתגרים שלא תמיד קיימים בסביבות Docker סטנדרטיות.

במקרים מסוימים נדרשתי לעבוד ישירות מול כלי מערכת ברמה נמוכה יותר, כגון `ctr`, במקום להסתמך על מנגנוני האוטומציה המובנים.

### היתרון של GitOps

אחד הדברים המרשימים ביותר שחוויתי בפרויקט היה מנגנון ה־Self-Healing של ArgoCD.

למרות תקלות תשתית, ניתוקי רשת ואתחולים של הסביבה המקומית, ברגע שהמצב הרצוי הוגדר ב־Git, ArgoCD דאג להחזיר את המערכת למצב התקין באופן אוטומטי.

---

## 🛠️ הוראות הפעלה ותחזוקה

### פתיחת Port Forward ל־ArgoCD

במידה והגישה לממשק ArgoCD נותקה:

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

#### 3. העתקת הקובץ אל Control Plane Node

```powershell
podman cp image.tar gitops-cluster-control-plane:/image.tar
```

#### 4. ייבוא Image אל containerd

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /image.tar
```

#### 5. עדכון Deployment

יש לעדכן את שדה ה־`image` בקובץ `deployment.yaml` כך שיכלול את הקידומת `localhost/`:

```yaml
image: localhost/<image>
```

---

## ✅ סטטוס הפרויקט

* ArgoCD מותקן ופועל.
* Cluster מסוג Kind פועל על גבי Podman.
* סנכרון GitOps פעיל.
* פריסת `whoami-dev` מנוהלת באמצעות ArgoCD.
* Images נטענים באופן מקומי ללא תלות ב־Docker Hub.
* המערכת נמצאת במצב Healthy ו־Synced.

פרויקט זה סיפק לי התנסות מעשית בעבודה עם Kubernetes, GitOps, ArgoCD, Podman ופתרון תקלות תשתית בסביבת פיתוח מקומית.
