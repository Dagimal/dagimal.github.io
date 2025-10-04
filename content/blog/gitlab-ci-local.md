---
title: "Capek Ngepush? Testing Gitlab CI/CD dengan gitlab-ci-local"
date: "2024-11-24T17:38:07+07:00"
slug: "testing-ci-cd-dengan-gitlab-ci-local"
#menu: "main"
#
# description is optional
#
description: "Testing CI/CD Tanpa Capek Ngepush"

tags: ["Container","GitLab","DevOps"]
---

<img width="1262" height="460" alt="image" src="https://github.com/user-attachments/assets/31b5f4c0-ffea-4ff2-abfc-a9bba299dfd2" />


Pasti kalian kesel banget dong liat gambar diatas, gimana nggak kesel, udah capek2 bikin pipeline, ehh ketika di push malah error, belom lagi nunggunya lama cuma buat tau pipeline yang kita bikin sukses apa nggak.. Pengen nangis aja rasanya, apalagi errornya ga cuma sekali (baca: ***skill issue***) 😭

Untungnya, ada om Nielsen yang bikin tools menarik banget nih, yaitu

> **[GitHub - firecow/gitlab-ci-local](https://github.com/firecow/gitlab-ci-local)**

<img width="810" height="337" alt="image" src="https://github.com/user-attachments/assets/c81d6be8-971e-46bb-aec6-8ae213ef5409" />

### Sebenernya benda apa sih itu?

Simpelnya gini, *agent* yang ada pada ***[gitlab runner](https://docs.gitlab.com/runner/)*** di *cloning* dan bisa kita jalanin di mesin lokal kita 😱

### Emangnya kenapa tuh kalo bisa jalan di mesin lokal kita ?

<img width="400" alt="image" src="https://github.com/user-attachments/assets/3617e030-f6e0-4dab-9f28-289a11688bc8" />

hadeuhhh, jadi gini.. yang dimaksud bisa jalan di mesin lokal kita itu, dia bisa di **trigger langsung di mesin kita** tanpa lewat GitLab Web UI nya. Jadi kita bisa langsung testing dengan leluasa tanpa harus repot2 trigger dari GitLab, tanpa harus nunggu lama buat ngecek job yang error, karena disini kita bisa milih mau run job yang mana aja tanpa harus ngulang dari awal 😜

### Lahh... Terus apa bedanya dong sama gitlab runner biasa ? Kan bisa jalan di lokal juga tuh

Bisa... tapi fiturnya nggak selengkap dan se-leluasa ini, lagian fitur ini udah ***deprecated*** dari gitlab nya alias udah ga bakal kepake lagi dan udah nggak di maintain lagi sama tim gitlab.

> source : https://gitlab.com/gitlab-org/gitlab/-/issues/385235

Oke, daripada kelamaan basa - basi, langsung kita praktek aja 🔬

## Instalasi

Karena aku disini pake Distro linux berbasis debian, instalasinya gini aja..

pertama, tambahin reponya dulu

```
sudo wget -O /etc/apt/sources.list.d/gitlab-ci-local.sources https://gitlab-ci-local-ppa.firecow.dk/gitlab-ci-local.sources
```

jangan lupa di update reponya

```
sudo apt-get update
```

terus install deh packagenya

```
sudo apt-get install gitlab-ci-local
```

untuk OS lain atau distribusi lain, bisa langsung cek aja di dokumentasi resminya

> https://github.com/firecow/gitlab-ci-local?tab=readme-ov-file#installation

## Pengetesan

setelah proses instalasi berhasil, pastikan dulu tools nya udah kepasang atau belum

```
gitlab-ci-local --version
```

ntar bakal muncul kaya gini

<img width="391" height="61" alt="image" src="https://github.com/user-attachments/assets/7189e130-8f03-484b-8a23-440a70029c4d" />

kalau udah kaya gitu, berarti tools udah kepasang, selanjutnya kita langsung masuk aja ke direktori dimana file .gitlab-ci.yml nya berada.

<img width="1092" height="698" alt="image" src="https://github.com/user-attachments/assets/e5e54079-c34c-4b3e-82b8-b92811c0ca8c" />


### List Pipeline Jobs

Kita bisa melakukan list job pipeline yang ada, dia akan auto detect dari file .gitlab-ci.yml nya itu

```
gitlab-ci-local --list
```

outputnya akan menjadi seperti ini

<img width="1160" height="347" alt="image" src="https://github.com/user-attachments/assets/fed098e0-22e3-412b-84ae-4873a9e3b133" />


bisa dilihat, semua stage yang ada pada ci file nya akan terbaca. Kita juga bisa menambahkan option `--list-all` untuk menampilkan stage / job yang di set ke never

<img width="1165" height="346" alt="image" src="https://github.com/user-attachments/assets/84e1095e-602b-42f3-8e4f-207e0687c968" />


### Run Stage Jobs

Nahh, untuk melakukan pengujian pertama, kita bisa menjalankan stage tertentu pada file ci kita. Contohnya, aku pengen ngerun job / stage bernama `deploy_development`

```
gitlab-ci-local deploy_development
```

maka dia akan melakukan running job tersebut secara langsung

<img width="1173" height="465" alt="image" src="https://github.com/user-attachments/assets/b1efdccb-2a20-4e90-bbf8-ee98c81aaf58" />

bisa dilihat seperti pada gambar, job nya **PASS** berarti sukses dijalankan.

kalian tinggal sesuaikan saja mau testing stage yang mana.

### Penutup

Dengan tools ini, kita bisa dengan leluasa testing dan run job tanpa harus mengotori histori pipeline kita. Sebenarnya masih banyak fitur yang bisa di jelajahi, kalian bisa ngulik sendiri di dokumentasi resminya, yang di praktekkan disini cuma yang sering aku pake aja.

Terimakasih sudah membaca 😄

**_Last Update : 2025/10/04_**
