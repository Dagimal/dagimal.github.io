---
title: "Kubespray Offline: Setup k8s Cluster Tanpa Koneksi Internet"
date: "2025-10-31T00:11:28+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."
#draft: true
slug: kubespray-offline
tags: ["container","devops","gitlab","k8s"]
---

<img alt="kubespray-failed" src="/img/kubespray-failed.png" />


Kemarin sempat dibuat frustasi ketika setup cluster k8s baru di server kantor, biasanya memang kita pakai kubespray buat provisioning cluster k8s karena server kantor on-prem dan biasanya selalu lancar dan aman tanpa kendala karena memang ada koneksi internet di server, tapi kemarin dibuat bingung karena server yang mau kita setup ga ada akses internet, akhirnya setelah beberapa saat mikir dan diskusi, kita uji nyali lah coba buat private repo mirror buat keperluan setup cluster, sekalian aja karena kantor belom punya juga private mirror buat keperluan setup k8s. Setelah coba ngulik dikit di internet, ketemu script buat install k8s dengan cara offline (tetep butuh internet buat bikin mirror repo nya sih), setelah kita coba Alhamdulillah bisa dan lancar, bahkan waktu yang di butuhkan buat setup juga jauh lebih singkat daripada harus selalu download dari internet.

***Disclaimer : Saya asumsikan kalian sudah pernah menggunakan kubespray sebelumnya***

> [https://github.com/kubespray-offline/kubespray-offline](https://github.com/kubespray-offline/kubespray-offline)

### Persiapan
Sebelum kita mulai instalasi, pastikan hal - hal berikut:
* Punya mesin yang ***internet accessible***, maksudnya adalah mesin yang ada akses internet buat download semua utilitas yang diperlukan untuk setup k8s. Terserah mau pakai server jumphost atau pakai host laptop kita sendiri juga bisa.
* Mesin target ***(cluster on-prem)*** yang tidak punya akses ke internet (jelas dong, kalo ga ada mau install dimana)
* Sistem operasi mesin target, sementara sampai panduan ini ditulis, kubespray-offline baru support untuk Keluarga OS GNU/Linux berikut :
	- RHEL dan turunannya.
	- Debian dan turunannya.
* Pastikan juga mesin untuk menjalankan kubespray bisa terhubung ke target node.
* Mesin untuk mirror repo / registry **(opsional, bisa menggunakan mesin yang sama)**

### Instalasi
1. langkah pertama, langsung aja clone repo nya

	```bash
	git clone https://github.com/kubespray-offline/kubespray-offline.git
	```
2. Install container tools <br>
	Fungsi dari container tools adalah untuk download / pull image yang dibutuhkan k8s. <br>
	karena saya mau pakai docker, download dulu dockernya dengan menjalankan script :
	```bash
	./install-docker.sh
	```
	alasan menggunakan docker karena praktis, sudah satu paket dengan containerd, jika kalian ga mau pakai docker bisa pilih menggunakan nerdctl, tapi harus download containerd nya secara terpisah, scriptnya sudah tersedia juga.

2. Edit file `config.sh`
	```bash
	#!/bin/bash

	source ./target-scripts/config.sh

	# container runtime for preparation node
	docker=${docker:-podman}
	#docker=${docker:-docker}
	#docker=${docker:-/usr/local/bin/nerdctl}

	# Run ansible in container?
	ansible_in_container=${ansible_in_container:-false}
	```

	Silahkan sesuaikan saja mau pakai yang mana, fungsinya hanya untuk download / pulling image yang dibutuhkan pada saat proses setup cluster, contoh:

	```bash
	docker=${docker:-podman}
	```
	artinya, jika docker belum terinstall / tidak tersedia di environment, maka dia akan menggunakan alternatifnya yaitu podman. Karena saya sudah menginstall docker pada langkah sebelumnya, maka biarkan dia memilih docker sebagai default container tools.

3. Jalankan script untuk download semua artefak yang dibutuhkan
	```bash
	./download-all.sh
	```

	semua artefak yang ter-download akan disimpan pada direktori `outputs`. Di dalam `outputs` akan ada beberapa script dan folder :

	```bash
	root@crm-gitlab-runner-stg:~/kubespray-offline/outputs# ll
	total 108
	drwxr-xr-x 10 root root 4096 Nov  2 01:43 ./
	drwxr-xr-x 16 root root 4096 Oct 28 03:52 ../
	-rw-r--r--  1 root root  777 Oct 28 03:59 config.sh
	-rw-r--r--  1 root root  778 Oct 28 03:49 config.sht
	-rw-r--r--  1 root root  647 Oct 28 03:08 config.toml
	-rw-r--r--  1 root root 1270 Oct 28 03:08 containerd.service
	drwxr-xr-x  3 root root 4096 Oct 28 03:08 debs/
	-rwxr-xr-x  1 root root  719 Oct 28 03:08 extract-kubespray.sh*
	drwxr-xr-x  8 root root 4096 Oct 28 02:56 files/
	drwxr-xr-x  2 root root 4096 Oct 28 03:05 images/
	-rwxr-xr-x  1 root root 2480 Oct 28 03:08 install-containerd.sh*
	drwxrwxr-x 19 root root 4096 Oct 28 09:16 kubespray-2.29.0/
	-rwxr-xr-x  1 root root 1034 Oct 28 03:08 load-push-all-images.sh*
	-rw-r--r--  1 root root  349 Oct 28 04:28 nginx-default.conf
	drwxr-xr-x  3 root root 4096 Oct 28 03:08 patches/
	drwxr-xr-x  3 root root 4096 Oct 28 03:08 playbook/
	drwxr-xr-x 24 root root 4096 Oct 28 02:56 pypi/
	-rw-r--r--  1 root root  607 Oct 28 03:08 pyver.sh
	-rwxr-xr-x  1 root root  394 Oct 28 03:08 setup-all.sh*
	-rwxr-xr-x  1 root root  408 Oct 28 03:08 setup-container.sh*
	-rwxr-xr-x  1 root root 2106 Oct 28 03:08 setup-offline.sh*
	-rwxr-xr-x  1 root root 1213 Oct 28 03:08 setup-py.sh*
	-rwxr-xr-x  1 root root  654 Oct 28 03:08 start-nginx.sh*
	-rwxr-xr-x  1 root root  890 Oct 28 08:21 start-registry-docker.sh*
	-rwxr-xr-x  1 root root  445 Oct 28 03:08 start-registry.sh*
	drwxr-xr-x  3 root root 4096 Oct 28 03:08 tmp/
	-rw-r--r--  1 root root  322 Oct 28 03:08 venv.sh
	```

### Install Kubespray
- Jalankan script di dalam direktori `outputs` <br>
	```bash
	./extract-kubespray.sh
	```
- Aktifkan virtual environment :
	```bash
	source kubespray-<version>/venv/bin/activate
	```
	Jika virtual environment sudah aktif, akan muncul tanda (venv) pada shell seperti berikut :
	```bash
	(venv) root@crm-gitlab-runner-stg:~/kubespray-offline/outputs#
	```
- Install ansible
	```bash
	pip install -U pip # update versi pip
	pip install -r requirements.txt # install ansible
	```

### Buat Mirror Server / Registry
Setelah semua artefak ter-download, langkah selanjutnya adalah membuat mirror server untuk package kubernetes, seperti mirror deb dan container registry

1. Copy semua isi direktori `outputs` ke server yang akan kita buat server mirror ***(skip saja jika menggunakan server yang sama)***
2. Edit `config.sh`
	```bash
	#!/bin/bash
	# Kubespray version to download. Use "master" for latest master branch.
	KUBESPRAY_VERSION=${KUBESPRAY_VERSION:-2.29.0}
	#KUBESPRAY_VERSION=${KUBESPRAY_VERSION:-master}
	NGINX_PORT=80

	# Versions of containerd related binaries used in `install-containerd.sh`
	# These version must be same as kubespray.
	# Refer `roles/kubespray_defaults/vars/main/checksums.yml` of kubespray.
	RUNC_VERSION=1.3.2
	CONTAINERD_VERSION=2.1.4
	NERDCTL_VERSION=2.1.6
	CNI_VERSION=1.8.0

	# Some container versions, must be same as ../imagelists/images.txt
	NGINX_VERSION=1.29.2
	REGISTRY_VERSION=2.8.3

	# container registry port
	REGISTRY_PORT=${REGISTRY_PORT:-35000}

	# Additional container registry hosts
	ADDITIONAL_CONTAINER_REGISTRY_LIST=${ADDITIONAL_CONTAINER_REGISTRY_LIST:-"myregistry.io"}
	```
	Sesuaikan versi kubespray yang kita gunakan pada langkah sebelumnya.

2. Masih pada folder `outputs`, jalankan script berikut :
	```bash
	./start-nginx.sh
	./start-registry.sh
	```

	script tersebut akan menjalankan :
	- menjalankan nginx container untuk membuat repo lokal (DEB/RPM)
	- menjalankan private registry container

### Offline Ansible Config
1. Copy file `offline.yml` dari direktori `kubespray-offline` ke direktori inventory 
	```bash
	cp offline.yml kubespray-<version>/inventory/mycluster/group_vars/all/
	```

2. Ganti `YOUR_HOST` menjadi host server mirror kita.

### Deploy Offline Repo
Setelah mirror server ready, langkah selanjutnya adalah config semua server target untuk menggunakan mirror server yang sudah kita buat.

1. Copy script ke direktori kubespray
	```bash
	cp -r outputs/playbook kubespray-<version>
	```

2. Deploy repo
	```bash
	ansible-playbook -i inventory/mycluster/hosts.yaml playbook/offline-repo.yml
	```

### Spray!
- Ini langkah terakhir, jalankan seperti biasa kita deploy cluster k8s
	```bash
	ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
	``` 

Alhamdulillah, cluster sudah ready dan siap untuk menjalankan workload aplikasi.
> <img alt="kubespray-failed" src="/img/ready-cluster.png" />
