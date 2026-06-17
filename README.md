<div dir="rtl">

# פרויקט GitOps עם ArgoCD, Kind ו-Podman

פרויקט זה מציג ארכיטקטורת GitOps מלאה עבור סביבת פיתוח מקומית (Local Development), המבוססת על **ArgoCD** לניהול וסנכרון דקלרטיבי של משאבי Kubernetes, על גבי **Kind Cluster** המורץ באמצעות **Podman** בסביבת Windows.

## 🚀 ארכיטקטורת המערכת

המערכת מבוססת על עקרון ה־Single Source of Truth, שבו כל קובצי ה־Manifest (כגון Deployment ו־Service) מנוהלים ב־Git Repository זה.

ArgoCD מנטר את ה־Repository באופן רציף ומבצע Reconciliation אוטומטי מול ה־Cluster המקומי.

---

## 📑 אתגרים טכנולוגיים והתמודדות (Troubleshooting & Engineering Challenges)

במהלך הקמת סביבת ה־GitOps ופריסת האפליקציה `whoami-dev` באמצעות ArgoCD, נתקלנו במספר אתגרי אינטגרציה ורשת בין Windows Host, Podman, Kind ו־containerd.

להלן האתגרים המרכזיים והפתרונות שיושמו.

### 1. בעיות רשת ומשיכת Images (`ImagePullBackOff`)

#### האתגר

לאחר הסנכרון הראשוני ב־ArgoCD, ה־Pods נכנסו למצב `ImagePullBackOff` ו־`ErrImagePull`.

בדיקה באמצעות:

```bash
kubectl describe pod <pod-name>
```

העלתה כי ה־Container Runtime של ה־Kind Cluster אינו מצליח לגשת ל־Docker Hub עקב בעיות DNS וקישוריות מתוך המכונה הווירטואלית.

#### הפתרון

בוצע ניסיון מעבר ל־GitHub Container Registry (`ghcr.io`), אך גם הוא נכשל מאותה סיבה.

המסקנה הייתה שה־Cluster מנותק לחלוטין מגישה חיצונית, ולכן הוחלט לעבור לתהליך עבודה Offline המבוסס על טעינת Images באופן ידני.

---

### 2. מגבלות `kind load` בסביבת Podman

#### האתגר

ניסיון להשתמש בפקודת:

```bash
kind load docker-image
```

נכשל עקב בעיות תאימות בין Kind לבין Podman.

התקבלו שגיאות כגון:

```text
ERROR: unknown command "podman-image"
unknown flag: --name
```

בנוסף, Kind לא הצליח לזהות Images שהיו קיימים ב־Podman Storage גם לאחר הגדרת:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman
```

#### הפתרון

במקום להמשיך להשתמש במנגנון הטעינה של Kind, הוחלט לעבוד ישירות מול ה־Container Runtime של Kubernetes.

---

### 3. קפיאת Podman Machine וניתוק ה־API Server

#### האתגר

במהלך ניסיונות איפוס רשת, ה־Podman Machine נכנסה למצב לא תקין.

מצד אחד התקבלה הודעה:

```text
already running
```

ומצד שני ה־Kubernetes API Server הפסיק להגיב.

בנוסף, פעולות Port Forward ל־ArgoCD נכשלו עם:

```text
connection refused
```

#### הפתרון

בוצעה הפעלה מחדש של ה־Control Plane Container:

```powershell
podman start gitops-cluster-control-plane
```

לאחר מכן ה־API Server חזר לפעול וניתן היה להפעיל מחדש Port Forward.

---

### 4. טעינת Image ישירות ל־containerd והתאמת שמות

#### האתגר

כדי לעקוף לחלוטין את התלות בגישה לאינטרנט, בוצע ייצוא של ה־Image מהמחשב המקומי:

```powershell
podman save -o whoami-latest.tar traefik/whoami:latest
```

לאחר מכן הועתק הקובץ אל ה־Control Plane Container ויובא ישירות ל־containerd:

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /whoami-latest.tar
```

לאחר הייבוא התברר כי ה־Image נרשם תחת השם:

```text
localhost/traefik/whoami:latest
```

בעוד שה־Deployment עדיין הפנה אל:

```text
traefik/whoami:latest
```

כתוצאה מכך ה־Pods המשיכו להיכשל.

#### הפתרון

בוצע עדכון של שדה ה־`image` ב־Deployment:

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

שילוב של Podman, Kind ו־Windows עשוי לדרוש פתרונות ברמה נמוכה יותר (Low-Level Operations), מכיוון שחלק מהכלים תוכננו במקור לעבודה מול Docker.

### היתרון של GitOps

למרות תקלות תשתית, ניתוקי רשת ואתחולים של הסביבה המקומית, ברגע שהמצב הרצוי הוגדר ב־Git Repository, ArgoCD החזיר את המערכת למצב תקין באופן אוטומטי באמצעות מנגנון Self-Healing.

---

## 🛠️ הוראות הפעלה ותחזוקה

### פתיחת Port Forward ל־ArgoCD

במידה והגישה לממשק ArgoCD נותקה:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### טעינת Image חדש ללא גישה לאינטרנט

#### 1. משיכת ה־Image למחשב המקומי

```powershell
podman pull <image>
```

#### 2. שמירת ה־Image לקובץ TAR

```powershell
podman save -o image.tar <image>
```

#### 3. העתקת הקובץ אל ה־Control Plane Node

```powershell
podman cp image.tar gitops-cluster-control-plane:/image.tar
```

#### 4. ייבוא ה־Image אל containerd

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
* Kind Cluster פועל על גבי Podman.
* GitOps Synchronization פעיל.
* פריסת `whoami-dev` מבוצעת דרך ArgoCD.
* Images נטענים מקומית ללא תלות ב־Docker Hub.
* המערכת נמצאת במצב Healthy ו־Synced.

</div>
