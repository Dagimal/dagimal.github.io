---
title: "Buildah: Build Container Image Tanpa Docker"
date: "2026-04-07T13:20:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Mengenal Buildah sebagai alternatif untuk build container image tanpa daemon Docker"
slug: "buildah"
tags: ["container","docker","devops","linux"]
---

Jadi, saat ngerjain container image, biasanya kita langsung pakai *Docker*. Push *Dockerfile*, jalankan `docker build`, selesai. Tapi ada masalah kalau kita pakai *Docker* di environment tertentu: daemon yang berjalan dengan privilege tinggi, butuh setup kompleks, atau environment yang restricted seperti *CI/CD* *pipeline* atau *rootless* *container*.

Nah, ada tool bernama *Buildah* yang hadir sebagai solusi alternatif untuk **build *container image* tanpa perlu *Docker daemon***. *Buildah* dirancang untuk flexible, secure, dan bisa dijalankan di mana saja.

> [Buildah - Official Website](https://buildah.io/)

### Apa itu Buildah?

*Buildah* adalah tool yang berfungsi untuk membangun *container image* secara declarative atau imperative. Tool ini bagian dari *Podman* ecosystem dan dikembangkan oleh Red Hat.

Perbedaan utama dengan *Docker*:
- **Gak butuh daemon** - *Buildah* bekerja tanpa perlu *Docker daemon* berjalan di background
- **Rootless-friendly** - Bisa jalan di *rootless container* tanpa perlu privilege escalation
- **Imperative dan Declarative** - Bisa pakai *Dockerfile* atau scripting langsung
- **Standard OCI-compliant** - Image yang dihasilkan compatible dengan standard *container image* format

### Kelebihan Buildah

- **Security** - **Gak perlu daemon privilege**, lebih aman untuk multi-user system
- **Flexibility** - Bisa script complex logic, gak terbatas syntax *Dockerfile*
- **Lightweight** - Overhead minimal dibanding *Docker*
- **Standalone** - Berjalan independently, gak tergantung service lain
- **OCI-compliant** - Image yang dihasilkan standard, bisa dijalankan di *Docker*, *Podman*, atau orchestrator apapun
- **Layering Control** - Fine-grained control terhadap layer yang dihasilkan

### Installasi

**Ubuntu/Debian:**
```bash
sudo apt install buildah
```

**Fedora/RHEL:**
```bash
sudo dnf install buildah
```

**macOS:**
```bash
brew install buildah
```

Verify installation:
```bash
buildah version
```

### Konsep Dasar

Di *Buildah*, ada beberapa istilah penting:

- ***Working Container:*** Container temporary yang dipakai untuk build. Bisa dimanipulasi dan di-commit
- ***Image:*** Hasil akhir dari build process, format *OCI*
- ***Commit:*** Proses untuk save state dari *working container* menjadi *layer* baru

### Workflow Sederhana

Mari kita buat contoh sederhana, build image Go application:

**Cara 1: Pakai Dockerfile (Declarative)**

Bikin `Dockerfile`:
```dockerfile
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN go build -o app .
EXPOSE 8080
CMD ["./app"]
```

Build pakai *Buildah*:
```bash
buildah build-using-dockerfile -f Dockerfile -t myapp:latest .
```

**Cara 2: Script imperative (procedural)**

Bikin `build.sh`:
```bash
#!/bin/bash

# Create working container
ctr=$(buildah from golang:1.21-alpine)

# Run commands di container
buildah run $ctr apk add --no-cache git
buildah run $ctr go mod download

# Copy files
buildah copy $ctr . /app

# Set working directory
buildah config --workingdir /app $ctr

# Build app
buildah run $ctr go build -o app .

# Set metadata
buildah config --exposed-ports 8080/tcp $ctr
buildah config --cmd ./app $ctr

# Commit ke image
buildah commit $ctr myapp:latest

# Cleanup
buildah delete $ctr
```

Jalankan:
```bash
bash build.sh
```

### Real Example: Multi-stage Build

Bikin aplikasi yang lebih optimized dengan *multi-stage*:

```bash
#!/bin/bash

# Stage 1: Builder
builder=$(buildah from golang:1.21-alpine)
buildah run $builder apk add --no-cache git
buildah copy $builder . /src
buildah run $builder -c "cd /src && go build -o app ."
buildah commit $builder builder-stage

# Stage 2: Runtime
runtime=$(buildah from alpine:latest)
buildah copy --from builder-stage $runtime /src/app /usr/local/bin/app
buildah config --cmd /usr/local/bin/app $runtime
buildah config --port 8080 $runtime
buildah commit $runtime myapp:latest

# Cleanup
buildah delete $builder $runtime
```

Hasilnya adalah image yang kecil karena hanya executable yang tercopy, bukan seluruh *Go* toolchain.

### Integrase dengan Podman

*Buildah* dan *Podman* adalah best friends. Setelah build image dengan *Buildah*, langsung bisa jalankan dengan *Podman*:

```bash
# Build dengan Buildah
buildah build-using-dockerfile -t myapp:latest .

# Run dengan Podman
podman run -p 8080:8080 myapp:latest
```

Atau push ke registry:
```bash
buildah push myapp:latest docker://registry.example.com/myapp:latest
```

### Tips & Tricks

**Lebih Verbose Output**
```bash
buildah --log-level=debug build-using-dockerfile -f Dockerfile -t myapp:latest .
```

**List Working Containers**
```bash
buildah containers
```

**Inspect Container State**
```bash
buildah inspect mycontainer
```

**Copy Files dari Host**
```bash
buildah copy mycontainer /local/path /container/path
```

**Run dengan Environment Variables**
```bash
buildah run -e FOO=bar mycontainer printenv FOO
```

### Performance Consideration

*Buildah* biasanya lebih cepat dari *Docker* karena:
- Gak ada daemon overhead
- Bisa run commands parallel
- Direct OS-level operations

Tapi, untuk build yang complex, performa tergantung disk I/O dan network (saat pull base image).

### Kapan Pakai Buildah?

- **Dalam CI/CD Pipeline** - Gak butuh spin up *Docker daemon*
- **Rootless Environment** - Environment yang restricted privilege
- **Complex Build Logic** - Butuh scripting yang flexible
- **Multi-tenant System** - Security isolation better
- **Kubernetes Pod** - Build image tanpa DinD (*Docker in Docker*)

### Saat Gak Cocok Pakai Buildah

- **Simple project** - *Docker* sudah cukup dan familiar
- **Team sudah invest di Docker** - Learning curve mungkin gak worth
- **Butuh GUI** - *Docker Desktop* ada GUI, *Buildah* CLI only

### Real Talk

Saya dulu skeptis dengan *Buildah*, pikir cuma tool alternative yang gak begitu berguna. Tapi setelah pakai di *CI/CD pipeline* tanpa *Docker daemon*, **impression berubah drastis**. Build jadi lebih cepat, gak perlu mounting *Docker socket* atau complex DinD setup, dan hasilnya same quality.

Khususnya di Kubernetes *pod*, pakai *Buildah* jauh lebih clean dibanding *Docker in Docker*. Gak ada privilege escalation, gak ada extra layer complexity, **semuanya just works**.

Kalau project kamu pakai *Dockerfile* dan butuh flexibility atau security improvement, definitely worth trying *Buildah*. Worst case, *Buildah* syntax bisa drop-in replacement untuk `docker build` command.

```bash
# Old
docker build -t myapp:latest .

# New
buildah build-using-dockerfile -t myapp:latest .
```

Start kecil, coba di development environment dulu, dan judge sendiri apakah worth migrate atau tidak.
