---
title: "Conventional Commits: Biar Git Log Kamu Gak Membingungkan"
date: "2026-04-07T18:00:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Panduan menulis commit message yang konsisten dan bermakna menggunakan Conventional Commits"
slug: "conventional-commits"
tags: ["git","devops","productivity"]
---

Jujur saja, siapa yang pernah nulis *commit message* seperti ini:

```
fix
wip
asdfgh
coba lagi
fix beneran
ok done
```

Kita semua pernah. Dan kita semua pernah ngerasain akibatnya: buka `git log` seminggu kemudian, bingung sendiri, gak ada yang bisa ditelusuri, dan susah tau perubahan apa yang terjadi di titik mana.

*Conventional Commits* adalah konvensi penulisan *commit message* yang sederhana tapi **membuat *git log* jadi dokumentasi yang bisa dibaca dan dipahami**. Bukan cuma oleh orang lain, tapi juga oleh diri sendiri tiga bulan ke depan.

> [Conventional Commits - Spesifikasi Lengkap](https://www.conventionalcommits.org/)

### Format Dasarnya

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

Contoh nyata:

```
feat(auth): tambah login dengan Google OAuth

fix(api): perbaiki race condition saat fetch data paralel

chore(deps): update dependency axios ke versi 1.6.0

docs(readme): tambah panduan setup lokal
```

Langsung keliatan bedanya dengan `fix` atau `wip`, kan?

### Tipe-tipe yang Perlu Diketahui

***feat*** → Fitur baru. Ini yang paling sering dipakai.
```
feat(payment): integrasi payment gateway Midtrans
```

***fix*** → Perbaikan bug.
```
fix(upload): perbaiki error saat upload file lebih dari 10MB
```

***chore*** → Pekerjaan maintenance yang tidak affect logika aplikasi. Update *dependency*, konfigurasi *build*, *script* otomasi.
```
chore(ci): tambah step lint di pipeline
chore(deps): bump golang dari 1.21 ke 1.22
```

***docs*** → Perubahan dokumentasi saja, tidak ada perubahan kode.
```
docs(api): tambah contoh request untuk endpoint /users
```

***refactor*** → Perubahan kode yang bukan *fix* dan bukan *feat*. Restrukturisasi, pembersihan kode, tanpa mengubah perilaku.
```
refactor(user): pecah fungsi getUserData jadi lebih kecil
```

***test*** → Tambah atau perbaiki *test*.
```
test(auth): tambah unit test untuk validasi JWT
```

***style*** → Perubahan yang tidak affect logika, hanya soal format: spasi, titik koma, *indentation*.
```
style: jalankan formatter di seluruh codebase
```

***perf*** → Peningkatan performa.
```
perf(query): tambah index di kolom user_id
```

***ci*** → Perubahan di konfigurasi *CI/CD*.
```
ci: tambah job deploy ke staging setelah merge ke main
```

***revert*** → Revert *commit* sebelumnya.
```
revert: revert "feat(auth): tambah login dengan Google OAuth"
```

### Scope: Opsional tapi Sangat Membantu

*Scope* adalah konteks di mana perubahan terjadi. Ditulis dalam tanda kurung setelah *type*.

```
feat(checkout): tambah opsi pembayaran COD
fix(dashboard): perbaiki chart tidak muncul di Safari
refactor(database): pisahkan connection pool per service
```

Tidak ada aturan baku soal *scope*, tergantung struktur proyek. Bisa berupa nama modul, nama halaman, nama *service*, atau nama komponen. Yang penting **konsisten di dalam satu *repo***.

### Breaking Change

Kalau ada perubahan yang tidak *backward compatible*, tandai dengan `!` atau tulis di *footer*:

```
feat(api)!: ubah format response endpoint /users

BREAKING CHANGE: field `name` dipecah jadi `first_name` dan `last_name`
```

Ini penting supaya tim yang *consume* API tau bahwa ada yang berubah secara fundamental dan butuh penyesuaian.

### Kenapa Ini Penting?

**Changelog otomatis**

Dengan *commit message* yang terstruktur, kita bisa generate *changelog* otomatis menggunakan tools seperti `conventional-changelog` atau `release-please`. Tidak perlu tulis *release notes* manual lagi.

```bash
npx conventional-changelog -p conventional -i CHANGELOG.md -s
```

**Semantic versioning otomatis**

*Commit* dengan tipe `fix` → bump *patch version* (1.0.0 → 1.0.1)
*Commit* dengan tipe `feat` → bump *minor version* (1.0.0 → 1.1.0)
*Commit* dengan `BREAKING CHANGE` → bump *major version* (1.0.0 → 2.0.0)

**Git log yang bisa dibaca**

```bash
git log --oneline

# Sebelum conventional commits:
# a1b2c3d fix
# e4f5g6h wip
# i7j8k9l coba lagi
# m1n2o3p ok beres

# Sesudah conventional commits:
# a1b2c3d feat(auth): tambah login dengan Google OAuth
# e4f5g6h fix(upload): perbaiki error file lebih dari 10MB
# i7j8k9l chore(deps): update axios ke 1.6.0
# m1n2o3p docs(readme): tambah panduan setup lokal
```

Langsung keliatan perubahan apa yang terjadi tanpa harus buka satu per satu.

### Enforce dengan Commitlint

Supaya konvensi ini diikuti oleh seluruh tim, pasang *commitlint* yang akan validasi *commit message* sebelum commit berhasil:

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky
```

Buat file konfigurasi `commitlint.config.js`:

```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
}
```

Setup *husky* untuk jalankan *commitlint* di *commit-msg hook*:

```bash
npx husky init
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
```

Sekarang kalau ada yang coba *commit* dengan message yang tidak sesuai konvensi, akan langsung ditolak:

```bash
git commit -m "fix something"
# ⧗ input: fix something
# ✖ subject may not be empty [subject-empty]
# ✖ type may not be empty [type-empty]
# ✖ found 1 problems, 0 warnings
```

### Contoh di Dunia Nyata

Misalnya kita lagi develop fitur checkout di *e-commerce*. Satu sesi kerja bisa menghasilkan *commit* seperti ini:

```
feat(checkout): buat halaman ringkasan order
feat(checkout): integrasi API ongkos kirim
fix(checkout): perbaiki total harga tidak update saat quantity berubah
test(checkout): tambah test untuk kalkulasi diskon
refactor(checkout): pisahkan komponen CartItem ke file terpisah
chore: update snapshot test setelah refactor
```

Lihat `git log` seminggu kemudian, langsung tau apa yang terjadi di fitur *checkout* tanpa perlu buka satu per satu *diff*-nya.

### Real Talk

Awalnya nulis *commit message* yang proper terasa ribet dan memperlambat. Tapi begitu sudah terbiasa, nulis *commit message* yang baik justru jadi **cara kita berpikir tentang perubahan yang kita buat**. Kalau susah nulis *commit message*-nya, biasanya itu tanda *commit*-nya terlalu besar dan perlu dipecah.

Satu aturan yang saya pegang: **satu *commit* harus bisa dijelaskan dalam satu kalimat**. Kalau tidak bisa, berarti ada terlalu banyak perubahan dalam satu *commit*.

Dan percaya atau tidak, *commit message* yang baik itu bentuk *documentation* yang paling sering dibaca tapi paling sering diabaikan.
