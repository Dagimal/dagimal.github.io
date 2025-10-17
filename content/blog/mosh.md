---
title: "Mosh: Stabil, Responsif, Anti ngelag"
date: "2025-10-17T01:23:20+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."
image: "img/mosh_header.jpg"
tags: ["container","devops","gitlab",]
---

<img width="750" alt="mosh" src="/img/mosh_header.jpg" />

Ketika melakukan _remote access_ ke server melalui SSH _(Secure Shell)_ dengan lokasi server yang dekat, satu kota atau bahkan negara tetangga, kita tidak ada masalah dengan _network latency_ yang kita dapat, semuanya lancar tanpa hambatan dan tanpa delay yang berarti, namun lain cerita jika kita terhubung ke server menggunakan **jaringan yang buruk**, atau lokasi server berada **sangat jauh**, lintas benua, kita akan dibuat kesal dengan latency yang kita dapat, perintah yang kita ketik menjadi terasa sangat delay, bahkan koneksi sering terputus tiba-tiba, hal ini tentu sangat tidak nyaman, dan membuat kita menjadi emosi 🤬

Ternyata ada utilitas bernama Mosh _(Mobile Shell)_ yang dirancang sebagai pengganti atau pelengkap dari SSH dan diperuntukkan untuk koneksi **tidak stabil**.
> [Mosh - Mobile Shell](https://mosh.org/)

### Cara Kerja

SSH berjalan menggunakan protokol TCP, sedangkan Mosh menggunakan UDP. Mosh akan membuat sesi terminal tetap hidup di sisi server, dan klien akan melakukan _resync_ begitu jaringan tersambung lagi. 

### Kelebihan

- Stabil di koneksi buruk ✅
- Tetap hidup setelah ganti jaringan ✅
- Reconnect otomatis ✅
- Respons terminal sangat cepat ✅

### Cara Pakai

Pastikan UDP port pada server dengan range `60000-61000` terbuka.

```
sudo ufw allow 60000:61000/udp`
```

Gunakan seperti SSH

```
mosh user@host
```
<img width="750" alt="mosh" src="/img/mosh_remote.png" />

Saya mencoba melakukan remote ke server saya yang berada di Amerika, hasilnya sangat memuaskan, bahkan terasa seperti menggunakan VPS _(Virtual Private Server)_ Singapore, sama sekali tidak terasa delay yang berarti.

### Skenario Pengujian

Untuk membuktikan keandalan dari Mosh, saya akan membuat dia reconnect ke server secara otomatis dengan cara berganti menggunakan jaringan yang berbeda pada saat sedang melakukan remote.

<img width="750" alt="mosh" src="/img/mosh_remote.gif" />

terlihat network disconnect dan berpindah jaringan, namun session masih bisa dilanjutkan.
