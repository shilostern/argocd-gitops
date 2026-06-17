<div dir="rtl">

# פרויקט GitOps עם ArgoCD, Kind ו-Podman

פרויקט זה מציג ארכיטקטורת GitOps מלאה לסביבת פיתוח מקומית (Local Development), המבוססת על **ArgoCD** לניהול וסנכרון דקלרטיבי של משאבי Kubernetes, על גבי קלאסטר **Kind** המורץ במנוע הקונטיינרים **Podman** בסביבת Windows.

## 🚀 ארכיטקטורת המערכת

המערכת מבוססת על עקרון ה-Single Source of Truth, שבו כל קבצי המניפסט (Deployment, Service וכדומה) מנוהלים בריפו זה ב-Git. ArgoCD מנטר את הריפו באופן קבוע ומבצע Reconciliation אוטומטי מול הקלאסטר המקומי.

---

## 📑 אתגרים טכנולוגיים והתמודדות (Troubleshooting & Engineering Challenges)

במהלך הקמת פרויקט ה-GitOps ופריסת האפליקציה `whoami-dev` באמצעות **ArgoCD**, נתקלנו במספר אתגרי אינטגרציה ורשת מורכבים בין כלי ה-DevOps השונים (Windows Host, Podman, Kind ו-containerd).

להלן פירוט האתגרים המרכזיים והדרך שבה הם נפתרו:

### 1. ניתוק רשת וחסימת משיכת Images (ImagePullBackOff)

**האתגר:**  
לאחר הסנכרון הראשוני ב-ArgoCD, הפודים נכנסו למצב `ImagePullBackOff` / `ErrImagePull`. בדיקה מעמיקה באמצעות `kubectl describe` העלתה כי ה-Container Runtime של קלאסטר ה-Kind המקומי חסום לחלוטין מול שרתים חיצוניים, ולא הצליח לפנות ל-Docker Hub עקב בעיות DNS וקישוריות פנימיות.

**הפתרון:**  
ניסיון לעבור ל-GitHub Container Registry (`ghcr.io`) נכשל מאותה סיבה, מה שהוכיח שהקלאסטר מנותק חיצונית לחלוטין. לכן הוחלט לעבור לאסטרטגיית פיתוח Offline המבוססת על הזרקת Images מקומיים.

### 2. כשל בכלי הניהול המקומיים (Kind Load vs Podman Storage)

**האתגר:**  
ניסינו להשתמש בפקודה `kind load` לצורך טעינת Image ישירות לקלאסטר. נתקלנו בשגיאות תאימות כגון:

```text
ERROR: unknown command "podman-image"
unknown flag: --name
```

בנוסף, Kind לא הצליח לזהות את ה-Image המאוחסן ב-Podman למרות שימוש ב:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman
```

**הפתרון:**  
במקום להמשיך לעבוד דרך שכבת ה-CLI של Kind, בוצעה עבודה ישירה מול ה-Container Runtime של Kubernetes.

### 3. קפיאת Podman Machine

**האתגר:**  
במהלך ניסיונות איפוס רשת, Podman Machine נכנסה למצב שבו דיווחה שהיא כבר רצה (`already running`) אך במקביל חסמה את ה-Kubernetes API Server וגרמה לשגיאות `connection refused`.

**הפתרון:**  
בוצע Forced Start לקונטיינר ה-Control Plane:

```powershell
podman start gitops-cluster-control-plane
```

פעולה זו החזירה את ה-API Server לפעולה תקינה ואפשרה מחדש שימוש ב-`kubectl port-forward`.

### 4. הזרקה ידנית ל-containerd ותיאום שמות (Naming Realignment)

#### ייצוא Image מהמחשב המארח

```powershell
podman save -o whoami-latest.tar traefik/whoami:latest
```

#### ייבוא Image ישירות ל-containerd

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /whoami-latest.tar
```

**האתגר האחרון:**  
לאחר הייבוא, ה-Image נרשם בשם:

```text
localhost/traefik/whoami:latest
```

בעוד שה-Deployment עדיין ביקש:

```text
traefik/whoami:latest
```

כתוצאה מכך הפודים המשיכו להיכשל.

**הפתרון בקוד:**

```yaml
image: localhost/traefik/whoami:latest
```

לאחר Commit ו-Push, ArgoCD זיהה את השינוי, עדכן את ה-ReplicaSet, והאפליקציה עלתה בהצלחה למצב Healthy.

---

## 💡 תובנות מפתח

### עבודה עם Podman על Windows

עבודה עם Podman בסביבת Windows דורשת לעיתים פתרונות מותאמים אישית, מכיוון שכלי הטעינה האוטומטיים של קלאסטרים מקומיים מותאמים בעיקר ל-Docker.

### עוצמת GitOps ו-ArgoCD

למרות תקלות ברמת התשתית, ברגע שהוגדר המצב הרצוי ב-Git, ArgoCD החזיר את המערכת למצב תקין באופן אוטומטי באמצעות מנגנון Self-Healing.

---

## 🛠️ הוראות הפעלה ותחזוקה

### פתיחת פורט ל-ArgoCD

במידה וה-Port Forward נסגר:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### טעינת Image חדש ללא גישה לאינטרנט

#### 1. משיכת Image

```powershell
podman pull <image>
```

#### 2. שמירה לקובץ TAR

```powershell
podman save -o image.tar <image>
```

#### 3. העתקה לנוד

```powershell
podman cp image.tar gitops-cluster-control-plane:/image.tar
```

#### 4. ייבוא ל-containerd

```powershell
podman exec gitops-cluster-control-plane ctr --namespace k8s.io images import /image.tar
```

#### 5. עדכון Deployment

יש לעדכן את שדה ה-image כך שיכלול את הקידומת:

```yaml
image: localhost/<image>
```

</div>
