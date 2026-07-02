---
title: "RBAC di Kubernetes: Atur Siapa Boleh Ngapain"
date: "2026-04-15T15:30:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Pengenalan RBAC di Kubernetes dan cara implementasinya agar akses ke cluster terkontrol dengan benar"
slug: "rbac-kubernetes"
tags: ["kubernetes","devops","security","k8s"]
---

Kalau kita sudah punya *Kubernetes cluster*, salah satu hal yang paling sering diabaikan adalah siapa yang boleh akses apa. Default-nya, kalau kita kasih seseorang *kubeconfig*, mereka bisa ngapa-ngapain di dalam *cluster*. Bisa delete *pod*, bisa baca *secret*, bisa bikin kekacauan.

*RBAC* atau *Role-Based Access Control* adalah mekanisme bawaan *Kubernetes* untuk mengontrol akses ke resource di dalam *cluster*. Dengan *RBAC*, kita bisa tentuin dengan tepat: **user A hanya boleh baca *pod* di *namespace* tertentu**, atau **service account B hanya boleh create *deployment***, tidak lebih dari itu.

### Konsep Dasar RBAC di Kubernetes

Sebelum mulai nulis *YAML*, penting untuk paham dulu empat komponen utama *RBAC*:

***Role*** dan ***ClusterRole*** adalah kumpulan permission. *Role* berlaku di dalam satu *namespace* saja, sedangkan *ClusterRole* berlaku di seluruh *cluster*.

***RoleBinding*** dan ***ClusterRoleBinding*** adalah yang menghubungkan *Role* atau *ClusterRole* ke *user*, *group*, atau *service account*. Tanpa *binding*, *role* tidak berarti apa-apa.

Singkatnya:
- *Role* → definisi "boleh ngapain"
- *RoleBinding* → definisi "siapa yang dikasih role itu"

### Contoh Sederhana: Developer Hanya Bisa Baca Pod

Misalnya kita punya developer yang perlu debug aplikasi, tapi gak perlu akses lebih dari itu. Kita buatkan *Role* yang hanya boleh baca *pod* dan lihat *log*:

```yaml
# role-developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: developer-readonly
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

Lalu hubungkan ke user dengan *RoleBinding*:

```yaml
# rolebinding-developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: production
subjects:
- kind: User
  name: budi
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

Terapkan ke *cluster*:

```bash
kubectl apply -f role-developer.yaml
kubectl apply -f rolebinding-developer.yaml
```

Sekarang user `budi` **hanya bisa lihat *pod* dan baca *log*** di *namespace* `production`. Gak lebih, gak kurang.

### Verbs yang Perlu Diketahui

Di *RBAC Kubernetes*, kita kontrol akses lewat *verbs*. Ini padanan CRUD-nya:

| Verb | Artinya |
|------|---------|
| `get` | Lihat satu resource |
| `list` | Lihat semua resource |
| `watch` | Stream perubahan resource |
| `create` | Buat resource baru |
| `update` | Update resource yang ada |
| `patch` | Update sebagian resource |
| `delete` | Hapus resource |
| `deletecollection` | Hapus banyak resource sekaligus |

### ClusterRole: Akses Seluruh Cluster

Kalau *role* kita perlu berlaku di semua *namespace*, gunakan *ClusterRole*:

```yaml
# clusterrole-monitoring.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
```

Lalu bind ke *service account* yang dipakai *Prometheus* atau *Grafana*:

```yaml
# clusterrolebinding-monitoring.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-reader-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

### Contoh Role per Tim

Ini contoh setup yang lebih nyata untuk tim yang terdiri dari beberapa role berbeda:

**Role untuk DevOps** (full access di *namespace* tertentu):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: devops-full
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
```

**Role untuk Developer** (deploy dan debug saja):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: developer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

**Role untuk QA** (read-only semua resource):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: qa-readonly
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Service Account untuk Aplikasi

*RBAC* bukan hanya untuk manusia. Aplikasi yang berjalan di dalam *cluster* juga butuh *service account* dengan permission yang tepat. Jangan biarkan aplikasi jalan dengan *default service account* yang tidak jelas permission-nya.

Buat *service account* khusus:

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
```

Buat *role* sesuai kebutuhan aplikasi:

```yaml
# role-myapp.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["myapp-secret"]  # Hanya secret tertentu saja
  verbs: ["get"]
```

Bind ke *service account*:

```yaml
# rolebinding-myapp.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

Lalu di *deployment*, tentukan *service account* yang dipakai:

```yaml
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: myapp
    image: myapp:latest
```

### Cek dan Debug Permission

Kalau ada yang komplen "gak bisa akses resource", kita bisa cek permission-nya:

```bash
# Cek apa yang boleh dilakukan user tertentu
kubectl auth can-i create deployments --namespace production --as budi

# Cek semua permission yang dimiliki user
kubectl auth can-i --list --namespace production --as budi

# Cek apakah service account bisa akses sesuatu
kubectl auth can-i get secrets --namespace production --as system:serviceaccount:production:myapp-sa
```

Lihat *role* dan *binding* yang ada:

```bash
# List semua role di namespace
kubectl get roles -n production
kubectl get rolebindings -n production

# Lihat detail role tertentu
kubectl describe role developer -n production

# Lihat siapa yang punya role tertentu
kubectl describe rolebinding developer-readonly-binding -n production
```

### Hal yang Perlu Dihindari

**Jangan pakai wildcard kalau tidak perlu**
```yaml
# Hindari ini kalau tidak benar-benar butuh
resources: ["*"]
verbs: ["*"]
```

**Jangan biarkan akses ke *secret* tanpa batas**
```yaml
# Ini berbahaya - bisa baca semua secret di namespace
resources: ["secrets"]
verbs: ["get", "list"]

# Lebih baik batasi ke secret tertentu saja
resources: ["secrets"]
resourceNames: ["myapp-config"]
verbs: ["get"]
```

**Jangan pakai *ClusterRoleBinding* kalau cukup *RoleBinding* saja**
Kalau user cuma butuh akses di satu *namespace*, jangan kasih *ClusterRoleBinding*. Scope sekecil mungkin.

### Real Talk

*RBAC* di *Kubernetes* memang butuh effort awal untuk setup. Tapi begitu sistemnya jalan, kita jadi tenang karena **setiap orang dan setiap aplikasi hanya punya akses yang memang mereka butuhkan**. Gak lebih.

Pernah ada kasus di mana *service account* punya akses terlalu lebar, dan ada bug di aplikasi yang tidak sengaja baca *secret* yang bukan miliknya. Dengan *RBAC* yang proper, hal seperti itu gak akan bisa terjadi karena aksesnya memang sudah di-lock dari awal.

Satu tips praktis: mulai dari permission paling kecil, lalu tambah kalau ada yang kurang. Jauh lebih aman dibanding mulai dari akses lebar lalu potong belakangan. **Principle of least privilege berlaku di sini, sama seperti di sistem operasi**.
