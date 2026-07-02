---
title: "External Secrets Operator: Inject Secret dari Vault ke Kubernetes"
date: "2026-04-10T16:00:00+07:00"
#menu: "main"
#
description: "Cara mengelola dan menyinkronkan secret dari HashiCorp Vault ke Kubernetes secara otomatis menggunakan ESO."
#
# description: "Cara menggunakan External Secrets Operator untuk sinkronisasi secret dari HashiCorp Vault ke Kubernetes secara otomatis"
slug: "external-secret-operator"
tags: ["kubernetes","vault","security","devops","k8s"]
---

Kalau kita manage *secret* di *Kubernetes*, cara paling umum adalah simpan langsung sebagai *Kubernetes Secret*. Tapi ada masalahnya: **nilai di dalam *Kubernetes Secret* hanya di-encode dengan *base64*, bukan dienkripsi**. Siapapun yang punya akses ke *cluster* bisa baca isinya. Belum lagi soal rotasi *secret* yang manual dan menyiksa.

*HashiCorp Vault* hadir sebagai solusi penyimpanan *secret* yang proper, dengan enkripsi, audit log, dan fitur rotasi otomatis. Masalahnya, bagaimana cara aplikasi di *Kubernetes* baca *secret* dari *Vault* tanpa ribet?

Di sinilah ***External Secrets Operator*** atau *ESO* masuk. Tugasnya sederhana: **ambil *secret* dari *Vault*, lalu buat *Kubernetes Secret* secara otomatis**. Aplikasi kita tetap baca *secret* dari *Kubernetes Secret* seperti biasa, tapi sumber kebenarannya ada di *Vault*.

> [External Secrets Operator - Official Docs](https://external-secrets.io/)

### Bagaimana Cara Kerjanya?

Alurnya seperti ini:

```
Vault (sumber secret) → ESO polling/sync → Kubernetes Secret → Pod/Aplikasi
```

*ESO* berjalan sebagai *controller* di dalam *cluster*. Kita definisikan *custom resource* bernama *ExternalSecret* yang berisi informasi: ambil *secret* ini dari *Vault*, simpan sebagai *Kubernetes Secret* dengan nama ini. *ESO* yang akan handle sinkronisasinya, termasuk refresh otomatis kalau *secret* di *Vault* berubah.

### Instalasi ESO

Cara paling mudah adalah lewat *Helm*:

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

Verifikasi *ESO* sudah jalan:

```bash
kubectl get pods -n external-secrets
# NAME                                READY   STATUS    RESTARTS
# external-secrets-xxxxxxxxx-xxxxx   1/1     Running   0
```

### Setup HashiCorp Vault

Asumsikan *Vault* sudah jalan. Kalau belum, bisa deploy di *Kubernetes* juga:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dev.enabled=true  # Untuk development saja
```

Aktifkan *KV secrets engine* dan simpan beberapa *secret*:

```bash
# Masuk ke pod Vault
kubectl exec -it vault-0 -n vault -- /bin/sh

# Aktifkan KV v2
vault secrets enable -path=secret kv-v2

# Simpan secret aplikasi
vault kv put secret/myapp/config \
  database_url="postgres://user:pass@db:5432/mydb" \
  api_key="supersecretapikey123" \
  jwt_secret="myjwtsecret"

# Verifikasi
vault kv get secret/myapp/config
```

### Buat Policy di Vault

*ESO* perlu *token* atau *auth method* untuk baca *secret* dari *Vault*. Buat *policy* yang hanya punya akses ke path yang dibutuhkan:

```hcl
# policy-myapp.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "secret/metadata/myapp/*" {
  capabilities = ["read", "list"]
}
```

Terapkan *policy*:

```bash
vault policy write myapp-policy policy-myapp.hcl
```

### Autentikasi ESO ke Vault

Ada beberapa cara *ESO* autentikasi ke *Vault*. Yang paling direkomendasikan di *Kubernetes* adalah *Vault Kubernetes Auth Method* karena tidak perlu hardcode *token*.

Aktifkan *Kubernetes auth* di *Vault*:

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"
```

Buat *role* yang menghubungkan *service account Kubernetes* ke *policy Vault*:

```bash
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=external-secrets-sa \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h
```

### Buat SecretStore

*SecretStore* adalah konfigurasi koneksi ke *Vault*. Definisikan di *namespace* yang butuh akses:

```yaml
# secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"
          serviceAccountRef:
            name: external-secrets-sa
```

Kalau mau satu *SecretStore* berlaku di seluruh *cluster*, gunakan *ClusterSecretStore*:

```yaml
# clustersecretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend-cluster
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### Buat ExternalSecret

Ini adalah bagian utamanya. *ExternalSecret* mendefinisikan *secret* mana yang mau diambil dari *Vault* dan bagaimana disimpan sebagai *Kubernetes Secret*:

```yaml
# externalsecret-myapp.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: production
spec:
  refreshInterval: 1h  # Sinkronisasi ulang setiap 1 jam
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secret  # Nama Kubernetes Secret yang akan dibuat
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL       # Key di Kubernetes Secret
    remoteRef:
      key: myapp/config           # Path di Vault
      property: database_url      # Field di dalam secret Vault
  - secretKey: API_KEY
    remoteRef:
      key: myapp/config
      property: api_key
  - secretKey: JWT_SECRET
    remoteRef:
      key: myapp/config
      property: jwt_secret
```

Terapkan:

```bash
kubectl apply -f externalsecret-myapp.yaml
```

Cek status sinkronisasi:

```bash
kubectl get externalsecret myapp-secret -n production

# NAME           STORE          REFRESH INTERVAL   STATUS
# myapp-secret   vault-backend  1h                 SecretSynced
```

*Kubernetes Secret* sudah terbuat otomatis:

```bash
kubectl get secret myapp-secret -n production
kubectl describe secret myapp-secret -n production
```

### Ambil Semua Secret Sekaligus

Kalau path di *Vault* punya banyak *field* dan mau ambil semua tanpa sebutkan satu-satu, gunakan `dataFrom`:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret-all
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secret-all
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: myapp/config  # Ambil semua field dari path ini
```

Semua *key-value* di `secret/myapp/config` akan otomatis jadi *key-value* di *Kubernetes Secret*.

### Studi Kasus: ClusterSecretStore untuk Multi-Namespace

Bayangkan kita punya tiga *namespace* berbeda: `production`, `staging`, dan `development`. Ketiganya butuh akses ke *Vault* yang sama, tapi masing-masing punya *secret* yang berbeda. Kalau pakai *SecretStore* biasa, kita harus buat tiga *SecretStore* dengan konfigurasi yang sama persis, hanya berbeda *namespace*-nya. Mubazir.

Solusinya: buat satu *ClusterSecretStore* yang bisa dipakai oleh semua *namespace*.

**Langkah 1: Buat Policy di Vault**

Karena *ClusterSecretStore* akan dipakai oleh banyak *namespace*, buat *policy* yang cukup lebar tapi tetap terbatas di path yang relevan:

```hcl
# policy-cluster.hcl
path "secret/data/production/*" {
  capabilities = ["read"]
}

path "secret/data/staging/*" {
  capabilities = ["read"]
}

path "secret/data/development/*" {
  capabilities = ["read"]
}

path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
```

Terapkan:

```bash
vault policy write cluster-policy policy-cluster.hcl
```

**Langkah 2: Buat Vault Role untuk ClusterSecretStore**

```bash
vault write auth/kubernetes/role/cluster-role \
  bound_service_account_names=external-secrets-sa \
  bound_service_account_namespaces=external-secrets \
  policies=cluster-policy \
  ttl=1h
```

Karena *ClusterSecretStore* dikelola oleh *ESO* yang jalan di *namespace* `external-secrets`, *service account* yang di-bind adalah milik *ESO* itu sendiri, bukan *namespace* aplikasi.

**Langkah 3: Buat ClusterSecretStore**

```yaml
# clustersecretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-cluster
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "cluster-role"
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets  # Namespace ESO itu sendiri
```

```bash
kubectl apply -f clustersecretstore.yaml

# Verifikasi status
kubectl get clustersecretstore vault-cluster
# NAME           AGE   STATUS   CAPABILITIES   READY
# vault-cluster  10s   Valid    ReadWrite      True
```

**Langkah 4: Simpan Secret di Vault per Environment**

```bash
# Secret untuk production
vault kv put secret/production/myapp \
  database_url="postgres://user:prodpass@prod-db:5432/mydb" \
  api_key="prod-api-key-xyz"

# Secret untuk staging
vault kv put secret/staging/myapp \
  database_url="postgres://user:stagingpass@staging-db:5432/mydb" \
  api_key="staging-api-key-abc"

# Secret untuk development
vault kv put secret/development/myapp \
  database_url="postgres://user:devpass@dev-db:5432/mydb" \
  api_key="dev-api-key-123"
```

**Langkah 5: Buat ExternalSecret di Masing-masing Namespace**

Di *namespace* `production`:

```yaml
# externalsecret-production.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-cluster
    kind: ClusterSecretStore  # Referensi ke ClusterSecretStore
  target:
    name: myapp-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: production/myapp  # Path khusus production
```

Di *namespace* `staging`:

```yaml
# externalsecret-staging.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: staging
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-cluster
    kind: ClusterSecretStore  # SecretStore yang sama
  target:
    name: myapp-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: staging/myapp  # Path khusus staging
```

Di *namespace* `development`:

```yaml
# externalsecret-development.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: development
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-cluster
    kind: ClusterSecretStore  # SecretStore yang sama
  target:
    name: myapp-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: development/myapp  # Path khusus development
```

Terapkan semua:

```bash
kubectl apply -f externalsecret-production.yaml
kubectl apply -f externalsecret-staging.yaml
kubectl apply -f externalsecret-development.yaml
```

**Verifikasi semua namespace sudah tersinkronisasi:**

```bash
kubectl get externalsecret -A
# NAMESPACE     NAME           STORE          REFRESH INTERVAL   STATUS
# production    myapp-secret   vault-cluster  1h                 SecretSynced
# staging       myapp-secret   vault-cluster  1h                 SecretSynced
# development   myapp-secret   vault-cluster  1h                 SecretSynced
```

Hasilnya, **satu *ClusterSecretStore* melayani tiga *namespace*** sekaligus. Konfigurasi koneksi ke *Vault* cukup diurus di satu tempat. Kalau ada perubahan *token*, *endpoint Vault*, atau konfigurasi auth, cukup update *ClusterSecretStore*-nya saja.

### Pakai Secret di Deployment

Setelah *Kubernetes Secret* terbuat oleh *ESO*, cara pakainya sama seperti *secret* biasa:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 2
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
        image: myapp:latest
        envFrom:
        - secretRef:
            name: myapp-secret  # Kubernetes Secret yang dibuat ESO
        # Atau pakai env spesifik
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: DATABASE_URL
```

### Rotasi Secret Otomatis

Salah satu kelebihan setup ini adalah **rotasi *secret* jadi otomatis**. Kalau kita update *secret* di *Vault*:

```bash
vault kv put secret/myapp/config \
  database_url="postgres://user:newpass@db:5432/mydb" \
  api_key="newapi key456" \
  jwt_secret="newjwtsecret"
```

*ESO* akan otomatis sinkronisasi ulang sesuai `refreshInterval` yang kita set. *Kubernetes Secret* akan terupdate, dan *pod* bisa restart untuk ambil nilai terbaru.

Kalau mau force refresh sekarang tanpa nunggu interval:

```bash
kubectl annotate externalsecret myapp-secret \
  force-sync=$(date +%s) \
  --overwrite \
  -n production
```

### Cek dan Debug

Kalau *secret* tidak tersinkronisasi, cek event-nya:

```bash
# Cek status ExternalSecret
kubectl describe externalsecret myapp-secret -n production

# Cek log ESO controller
kubectl logs -n external-secrets \
  -l app.kubernetes.io/name=external-secrets \
  --tail=50

# Cek event di namespace
kubectl get events -n production --sort-by=.lastTimestamp
```

Error yang paling umum biasanya:
- *Service account* tidak punya permission yang benar di *Vault*
- Path *secret* di *Vault* salah
- *SecretStore* gagal koneksi ke *Vault*

### Real Talk

Sebelum pakai *ESO*, setup *secret* di *Kubernetes* itu menyiksa. Harus encode manual ke *base64*, paste ke *YAML*, apply ke *cluster*, dan kalau ada perubahan harus ulang dari awal lagi. Belum lagi kalau *secret* bocor di *Git* karena gak sengaja di-commit.

Dengan *ESO* dan *Vault*, **sumber kebenaran *secret* ada di satu tempat yang aman dan teraudit**. Developer tidak perlu tahu isi *secret* production, mereka hanya perlu tahu nama *key*-nya. Dan kalau ada rotasi *secret*, cukup update di *Vault*, sisanya otomatis.

Setup awalnya memang butuh waktu, tapi begitu jalan, **manajemen *secret* jadi jauh lebih bersih dan aman**.
