---
title: "ArgoCD: CI Cukup Build, Urusan Deploy Serahin ke Git"
date: "2026-04-07T17:00:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Cara memisahkan proses CI dan CD menggunakan ArgoCD sebagai GitOps deployment tool di Kubernetes"
slug: "argocd-gitops"
tags: ["kubernetes","argocd","gitops","cicd","devops"]
---

Kalau kita bicara *CI/CD*, kebanyakan orang masih sering campur aduk antara proses *CI* dan *CD* dalam satu *pipeline*. Build *image*, push ke *registry*, terus langsung *deploy* ke *Kubernetes* dalam satu alur. Kelihatannya simpel, tapi ada beberapa masalah yang muncul seiring waktu:

- *Pipeline* jadi satu titik kegagalan. Kalau ada yang salah di proses *deploy*, susah dipisahkan dari proses *build*
- Tidak ada *audit trail* yang jelas siapa yang trigger *deploy* dan kapan
- *Rollback* butuh re-run *pipeline*, padahal seharusnya cukup balik ke versi sebelumnya
- *CI pipeline* butuh akses langsung ke *Kubernetes cluster*, yang berarti ada *credential* cluster yang disimpan di *CI system*

*ArgoCD* hadir dengan pendekatan *GitOps*: **sumber kebenaran untuk kondisi *cluster* adalah *Git repository***. *ArgoCD* yang akan watch *repo*, dan kalau ada perbedaan antara yang ada di *Git* dengan yang berjalan di *cluster*, dia akan sync otomatis atau minta kita untuk sync manual.

Intinya, **CI cukup urusin build dan push image. CD sepenuhnya urusan ArgoCD**.

### Konsep GitOps

Sebelum masuk ke praktek, penting untuk paham filosofi *GitOps*:

- Semua konfigurasi *deployment* disimpan di *Git* (*Kubernetes manifest*, *Helm chart*, atau *Kustomize*)
- *Git* adalah satu-satunya sumber kebenaran
- Perubahan ke *cluster* hanya boleh lewat *Git*, bukan manual `kubectl apply`
- *ArgoCD* continuously watch *repo* dan pastikan kondisi *cluster* selalu sesuai dengan yang ada di *Git*

Dengan pendekatan ini, **history deployment ada di Git history**. Mau tau siapa yang deploy apa dan kapan? Cukup `git log`.

### Arsitektur yang Kita Bangun

Kita akan pisahkan dua *repository*:

***App Repo*** → kode aplikasi. *CI pipeline* di sini yang handle build dan push *image*.

***Config Repo*** → *Kubernetes manifest* atau *Helm values*. *ArgoCD* watch *repo* ini. Kalau ada perubahan di sini, *ArgoCD* akan sync ke *cluster*.

Alurnya:

```
Developer push code → CI build & push image → CI update image tag di Config Repo
                                                        ↓
                                          ArgoCD detect perubahan di Config Repo
                                                        ↓
                                          ArgoCD sync ke Kubernetes cluster
```

*CI* tidak pernah sentuh *cluster* langsung. Tugasnya selesai setelah update *image tag* di *Config Repo*.

### Instalasi ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Tunggu semua *pod* ready:

```bash
kubectl get pods -n argocd -w
```

Akses *ArgoCD UI* lewat *port-forward*:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Ambil *password* awal:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Buka `https://localhost:8080`, login dengan user `admin` dan *password* yang barusan didapat.

### Struktur Config Repository

*Config Repo* kita strukturkan seperti ini:

```
config-repo/
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── production/
│   │       │   ├── kustomization.yaml
│   │       │   └── values.yaml
│   │       └── staging/
│   │           ├── kustomization.yaml
│   │           └── values.yaml
```

Contoh `base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: registry.example.com/myapp:latest  # Tag ini yang akan diupdate CI
        ports:
        - containerPort: 8080
```

Contoh `overlays/production/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namespace: production
replicas:
- name: myapp
  count: 3
images:
- name: registry.example.com/myapp
  newTag: "1.0.0"  # CI yang update nilai ini
```

### Buat ArgoCD Application

*ArgoCD Application* adalah *custom resource* yang mendefinisikan: watch *repo* ini, deploy ke *cluster* ini, di *namespace* ini.

```yaml
# argocd-app-production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/config-repo.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # Hapus resource yang sudah tidak ada di Git
      selfHeal: true   # Auto sync kalau ada yang berubah manual di cluster
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f argocd-app-production.yaml
```

*ArgoCD* akan langsung mulai sync *manifest* dari *repo* ke *cluster*.

### CI Pipeline: Tugasnya Hanya Build dan Update Tag

Sekarang kita lihat bagaimana *CI pipeline* berinteraksi dengan *Config Repo*. Contoh menggunakan *GitLab CI*:

```yaml
# .gitlab-ci.yml di App Repo
stages:
  - build
  - update-config

build-image:
  stage: build
  script:
  - docker build -t registry.example.com/myapp:$CI_COMMIT_SHORT_SHA .
  - docker push registry.example.com/myapp:$CI_COMMIT_SHORT_SHA
  only:
  - main

update-image-tag:
  stage: update-config
  image: alpine/git
  script:
  # Clone config repo
  - git clone https://oauth2:$CONFIG_REPO_TOKEN@github.com/org/config-repo.git
  - cd config-repo

  # Update image tag di kustomization production
  - |
    sed -i "s|newTag:.*|newTag: \"$CI_COMMIT_SHORT_SHA\"|" \
    apps/myapp/overlays/production/kustomization.yaml

  # Commit dan push perubahan
  - git config user.email "ci@example.com"
  - git config user.name "CI Pipeline"
  - git add .
  - git commit -m "chore: update myapp image tag to $CI_COMMIT_SHORT_SHA"
  - git push
  only:
  - main
```

Selesai. *CI* tidak tau apa-apa soal *Kubernetes*. Tugasnya hanya:
1. Build dan push *image* dengan tag yang unik (biasanya *commit hash*)
2. Update *image tag* di *Config Repo*
3. *ArgoCD* yang akan lanjutkan dari sini

### Sync Manual vs Otomatis

*ArgoCD* punya dua mode sync:

**Sync Otomatis** (seperti contoh di atas dengan `automated`)
*ArgoCD* akan langsung sync begitu ada perubahan di *Git*. Cocok untuk *staging* atau *development*.

**Sync Manual**
Kita yang tentukan kapan *deploy* terjadi. Cocok untuk *production* di mana kita mau ada kontrol lebih.

Untuk sync manual, hapus bagian `automated` di *Application*:

```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true
  # Tidak ada automated, sync dilakukan manual
```

Lalu sync lewat *UI* atau *CLI*:

```bash
# Install ArgoCD CLI
brew install argocd  # macOS
# atau
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Login
argocd login localhost:8080 --username admin --password <password>

# Lihat status aplikasi
argocd app list

# Sync aplikasi
argocd app sync myapp-production

# Lihat detail dan history
argocd app get myapp-production
argocd app history myapp-production
```

### Rollback Semudah Git Revert

Salah satu kelebihan *GitOps* yang paling terasa: **rollback semudah revert commit di Git**.

Kalau *deploy* terbaru bermasalah:

```bash
# Di Config Repo, revert commit terakhir
git revert HEAD
git push
```

*ArgoCD* akan detect perubahan dan sync ulang ke versi sebelumnya. Atau lewat *ArgoCD CLI*:

```bash
# Lihat history deployment
argocd app history myapp-production

# Rollback ke revision tertentu
argocd app rollback myapp-production <revision-id>
```

Tidak perlu re-run *pipeline*, tidak perlu ingat versi *image* apa yang sebelumnya jalan. Semua sudah tercatat di *Git history*.

### Notifikasi Sync

Supaya tim tau kalau ada *deploy* yang berhasil atau gagal, *ArgoCD* punya fitur *notifications*. Contoh konfigurasi notifikasi ke *Slack*:

```yaml
# argocd-notifications-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: "xoxb-your-slack-token"
```

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      Aplikasi *{{.app.metadata.name}}* berhasil di-deploy.
      Revision: {{.app.status.sync.revision}}
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

### Real Talk

Pemisahan *CI* dan *CD* seperti ini terasa ribet di awal, tapi begitu sudah jalan, **alur kerja jadi jauh lebih bersih**. *CI pipeline* jadi lebih ringan karena tidak perlu akses *cluster*, cukup akses *container registry* dan *Config Repo*. *Cluster credential* tidak perlu disimpan di *CI system* sama sekali.

Salah satu hal yang paling terasa manfaatnya adalah saat rollback. Sebelum pakai *ArgoCD*, rollback berarti cari tahu *image tag* yang sebelumnya jalan, ubah di *pipeline*, trigger manual, tunggu. Sekarang cukup `argocd app rollback` atau revert satu *commit* di *Git*.

Dan karena semua perubahan *deployment* lewat *Git*, **audit trail jadi sangat jelas**. Bisa lihat kapan *deploy* terjadi, siapa yang trigger, dan apa yang berubah, hanya dari `git log`.
