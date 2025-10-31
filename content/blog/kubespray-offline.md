---
title: "Kubespray Offline"
date: "2025-10-31T00:11:28+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."
draft: true
tags: ["container","devops","gitlab",]
---

<img width="750" alt="mosh" src="/img/kuebspray.jpg" />


Kemarin sempat dibuat frustasi waktu mau setup cluster k8s di server kantor, biasanya memang kita pake kubespray buat provisioning cluster k8s karena server kantor on-prem dan biasanya selalu lancar dan aman tanpa kendala karena memang ada koneksi internet di server, tapi kemarin dibuat bingung karena server yang mau kita setup ga ada akses internet, akhirnya setelah beberapa saat mikir dan diskusi, kita uji nyali lah coba buat private mirror server buat keperluan setup cluster, sekalian aja karena kantor belom punya juga private mirror buat k8s. Setelah coba ngulik dikit di internet, ketemu script buat install k8s dengan cara offline (tetep butuh internet buat bikin mirror repo nya sih), setelah kita coba Alhamdulillah bisa dan lancar, bahkan waktu yang di butuhkan buat setup juga jauh lebih singkat daripada harus selalu download dari internet.

### Instalasi
langkah pertama, langsung aja clone repo nya 

```
git clone https://github.com/kubespray-offline/kubespray-offline.git
```


