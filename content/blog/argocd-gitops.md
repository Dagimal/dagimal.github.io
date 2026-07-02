---
title: "ArgoCD: CI Cukup Build, Urusan Deploy Serahin ke Git"
date: "2026-04-23T17:00:00+07:00"
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

- Semua konfigurasi *deployment* disimpan di *Git* (*Helm charts* dan *values*)
- *Git* adalah satu-satunya sumber kebenaran
- Perubahan ke *cluster* hanya boleh lewat *Git*, bukan manual `kubectl apply`
- *ArgoCD* continuously watch *repo* dan pastikan kondisi *cluster* selalu sesuai dengan yang ada di *Git*

Dengan pendekatan ini, **history deployment ada di Git history**. Mau tau siapa yang deploy apa dan kapan? Cukup `git log`.

### Arsitektur yang Kita Bangun

Kita memisahkan konfigurasi menjadi beberapa bagian untuk meningkatkan *reusability* dan keamanan:

1. **App Repo** $\rightarrow$ kode aplikasi. *CI pipeline* di sini yang handle build dan push *image*.
2. **Helm Chart Repo** $\rightarrow$ berisi *template* manifest Kubernetes yang generik.
3. **Helm Values Repo** $\rightarrow$ berisi konfigurasi spesifik per aplikasi, per environment, dan per cluster.

Alurnya:

```
Developer push code → CI build & push image → CI update image tag di Helm Values Repo
                                                        ↓
                                          ArgoCD detect perubahan di Helm Values Repo
                                                        ↓
                                          ArgoCD sync ke Kubernetes cluster menggunakan Chart + Values
```

*CI* tidak pernah sentuh *cluster* langsung. Tugasnya selesai setelah update *image tag* di *Helm Values Repo*.

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

### Implementasi Multiple Sources (Helm Chart + Values)

Salah satu fitur powerful ArgoCD adalah **Multiple Sources**. Kita bisa mengambil *Chart* dari satu repo dan *Values*-nya dari repo lain. Ini sangat berguna agar kita tidak perlu menduplikasi chart untuk setiap environment.

#### Struktur Helm Values Repository

*Values Repo* kita strukturkan agar rapi per environment dan per aplikasi:

```
helm-values/
├── staging/
│   └── billing-engine/
│       ├── values.yaml             # Base values untuk staging
│       └── values-gdl-stg.yaml     # Override khusus cluster GDL Staging
└── production/
    └── billing-engine/
        ├── values.yaml
        └── values-gdl-prod.yaml
```

#### Membuat ArgoCD ApplicationSet

Untuk mengelola deployment ke banyak cluster sekaligus, kita menggunakan `ApplicationSet`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: billing-engine
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: gdl-stg
            url: "https://kube.dagimal.com:6443"
  template:
    metadata:
      name: 'billing-engine-{{cluster}}'
    spec:
      project: default
      sources:
        # Source 1: Helm Values Repo (di-refer sebagai 'values')
        - repoURL: https://git.dagimal.com/devops/helm-values.git
          targetRevision: main
          ref: values
        # Source 2: Helm Chart Repo
        - repoURL: https://git.dagimal.com/devops/helm-chart.git
          targetRevision: main
          path: pelayanan-pelanggan
          helm:
            valueFiles:
              - $values/staging/billing-engine/values-{{cluster}}.yaml
              - $values/staging/billing-engine/values.yaml
      destination:
        server: '{{url}}'
        namespace: service01
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**Poin penting:**
- `ref: values` memberikan nama alias pada source pertama.
- `$values/path/to/file` digunakan di source kedua untuk mengambil file dari source pertama.

### CI Pipeline: Tugasnya Hanya Build dan Update Tag

*CI pipeline* kini hanya perlu mengupdate file `.yaml` di *Helm Values Repo*. Contoh menggunakan *GitLab CI*:

```yaml
# .gitlab-ci.yml di App Repo
stages:
  - build
  - update-config

build-image:
  stage: build
  script:
  - docker build -t registry.dagimal.com/myapp:$CI_COMMIT_SHORT_SHA .
  - docker push registry.dagimal.com/myapp:$CI_COMMIT_SHORT_SHA
  only:
  - main

update-helm-values:
  stage: update-config
  image: alpine/git
  script:
  - git clone https://oauth2:$VALUES_REPO_TOKEN@gitlab.example.com/devops/helm-values.git
  - cd helm-values
  # Update image tag di values file menggunakan sed atau yq
  - sed -i "s|tag:.*|tag: \"$CI_COMMIT_SHORT_SHA\"|" staging/myapp/values.yaml
  - git config user.email "ci@example.com"
  - git config user.name "CI Pipeline"
  - git add .
  - git commit -m "chore: update myapp image tag to $CI_COMMIT_SHORT_SHA"
  - git push
  only:
  - main
```

### Sync Manual vs Otomatis

*ArgoCD* punya dua mode sync:

**Sync Otomatis** (menggunakan `automated` di `syncPolicy`)
*ArgoCD* akan langsung sync begitu ada perubahan di *Git*. Cocok untuk *staging* atau *development*.

**Sync Manual**
Kita yang tentukan kapan *deploy* terjadi. Cocok untuk *production* di mana kita mau ada kontrol lebih.

### Rollback Semudah Git Revert

Salah satu kelebihan *GitOps* yang paling terasa: **rollback semudah revert commit di Git**.

Kalau *deploy* terbaru bermasalah:

```bash
# Di Helm Values Repo, revert commit terakhir
git revert HEAD
git push
```

*ArgoCD* akan detect perubahan dan sync ulang ke versi sebelumnya. Atau lewat *ArgoCD CLI*:

```bash
# Lihat history deployment
argocd app history billing-engine-gdl-stg

# Rollback ke revision tertentu
argocd app rollback billing-engine-gdl-stg <revision-id>
```

### Real Talk

Pemisahan antara *Chart* dan *Values* menggunakan fitur *Multiple Sources* ArgoCD membuat manajemen konfigurasi menjadi sangat scalable. Kita tidak perlu mengelola ratusan file manifest Kubernetes yang hampir identik. Cukup satu Chart untuk semua, dan beberapa file *Values* untuk membedakan environment.

Alur kerja menjadi jauh lebih bersih:
- **Developers** fokus pada kode.
- **CI** fokus pada artifact.
- **SRE/DevOps** fokus pada konfigurasi di *Values Repo*.
- **ArgoCD** memastikan semua sinkron.
