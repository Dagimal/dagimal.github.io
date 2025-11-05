---
title: "2fa dengan Gopass"
date: "2025-11-05T15:36:14+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."
slug: "2fa-gopass"
tags: ["container","devops","gitlab","k8s",]
---

Udah lama ga buka github, tiba-tiba kemarin waktu buka di ingetin suruh aktivasi 2fa buat login ke GitHub, yaudah tanpa mikir panjang langsung gas eksekusi pake gopass aja karena ini password manager yang udah integrasi fitur otp. <br>
[Gopass - An awesome password manager for awesome teams]("https://github.com/gopasspw/gopass")

### Buat Credentials
Masukkan secret key 2fa otp.

```bash
gopass insert github/dagimal
```

### Cek Credentials

```bash
gopass show github/dagimal
```

akan tampil seperti ini :

<img alt="gopass insert" src="/img/gopass-insert.png" />

### Generate OTP
```bash
gopass otp github/dagimal
```

gopass akan generate otp :

<img alt="gopass otp" src="/img/gopass-otp.png" />

