---
title: "SSH Match User"
date: "2025-11-29T23:20:47+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."

tags: ["container","devops","gitlab","k8s","hardening"]
---

Secara default, SSH dirancang untuk menanggapi semua permintaan koneksi yang masuk. Jika user tidak ditemukan, SSH tetap menjalankan fase autentikasi dengan cara meminta password sebelum akhirnya menolak akses karena user tidak ada. Hal ini bisa menjadi celah kecil yang dapat dieksploitasi dalam skenario brute-force attack, dan dapat menyebabkan ***connection flood*** pada ***network socket***.

Untuk menghindarinya, kita bisa mengaktifkan fitur Match User dari OpenSSH.

### Cari baris config berikut
```sh
PasswordAuthentication yes
```
Ubah menjadi **no** untuk global scope

### Buat blok baru
```sh
Match User dagimal
PasswordAuthentication yes
```
Buat blok baru untuk Match User, hanya izinkan autentikasi password untuk user tertentu.

### Pengujian
<img alt="" src="/img/ssh-match-user-list.png" /> <br>
User tidak ditemukan pada sistem.

**>> Sebelum** <br>
<img alt="" src="/img/ssh-match-user-before.png" /> <br>
**>> Sesudah** <br>
<img alt="" src="/img/ssh-match-user-after.png" /> <br>
**>> Syslog**
<img alt="" src="/img/syslog.png" /> <br>
