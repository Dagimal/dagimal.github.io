---
title: "Distroless Images: Obat Obesitas pada Container"
date: "2025-10-06T14:33:06+07:00"
#menu: "main"
#
# description is optional
#
# description: "An optional description for SEO. If not provided, an automatically created summary will be used."

tags: ["container","devops","gitlab",]
---

<img width="1262" height="460" alt="image" src="https://gist.github.com/user-attachments/assets/58cb2a4c-4c20-4a6a-9a3b-0eec535ea150" />


Ketika kita sedang melakukan build image, salah satu hal yang bikin kita kesal adalah _image size_ yang besar, tentu ini sangat mengganggu karena bisa membuat proses _delivery_ menjadi lebih lama dan juga membuat _registry server_ menjadi cepat penuh.

Dalam praktik ***production grade cluster***, hal tersebut juga kurang bagus karena _worker server_ harus menjalankan _image_ dengan ukuran besar, tentu ini sangat tidak efisien, membuang banyak _storage resource_ dan cenderung _bloated_.

### Terus solusinya gimana?

Tenang, gunakan saja _distroless images_ 🤩

### Apa itu distroless images ?

_Distroless / Minimal Images_ adalah base image yang dirancang se-minimal, se-kecil, dan se-ringan mungkin untuk menjalankan _service_ inti dan dependensi penting untuk _runtime_ saja, tidak termasuk komponen yang tidak diperlukan, seperti _Package Manager_, Shell, dan utilitas baris perintah seperti curl, wget, ls, dll. Dengan membuang semua hal yang tidak diperlukan, tidak heran ukuran _image_ menjadi sangat kecil bahkan hanya beberapa ~MiB saja.

Istilah _Distroless_ ini dipopulerkan oleh sebuah proyek yang dikembangkan oleh tim di Google, kemudian proyek ini dikelola secara publik pada git _repository_ berikut :
> https://github.com/GoogleContainerTools/distroless

### Gimana cara pakainya?

Mengutip dari repo mereka, sudah ada beberapa runtime umum yang sudah siap pakai, seperti java dan nodejs,


| Image                                 | Tags                                  | Architecture Suffixes             |
| ------------------------------------- | ------------------------------------- | --------------------------------- |
| gcr.io/distroless/static-debian12     | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/base-debian12       | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/base-nossl-debian12 | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/cc-debian12         | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/python3-debian12    | latest, nonroot, debug, debug-nonroot | amd64, arm64                      |
| gcr.io/distroless/java-base-debian12  | latest, nonroot, debug, debug-nonroot | amd64, arm64, s390x, ppc64le      |
| gcr.io/distroless/java17-debian12     | latest, nonroot, debug, debug-nonroot | amd64, arm64, s390x, ppc64le      |
| gcr.io/distroless/java21-debian12     | latest, nonroot, debug, debug-nonroot | amd64, arm64, ppc64le             |
| gcr.io/distroless/nodejs20-debian12   | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/nodejs22-debian12   | latest, nonroot, debug, debug-nonroot | amd64, arm64, arm, s390x, ppc64le |
| gcr.io/distroless/nodejs24-debian12   | latest, nonroot, debug, debug-nonroot | amd64, arm64, s390x, ppc64le |

cara menggunakannya sama saja, kita bisa mengambilnya dengan cara _pull image_ seperti biasa, atau kita definisikan pada Containerfile, misal saya mau menggunakan java21

```
podman pull gcr.io/distroless/java21-debian12
```

jika menggunakan Containerfile

```
FROM gcr.io/distroless/java21-debian12

WORKDIR /app

COPY ./build/*.jar /app/app.jar

CMD ["java -jar app.jar"]
```

kita juga bisa membuatnya sendiri dengan menggunakan _base image_ tanpa _runtime_ kemudian meraciknya sendiri sesuka hati kita menyesuaikan _runtime_ yang kita gunakan, misal golang.

### Perbandingan

Berikut perbandingan ukuran _image_ jika menggunakan distroless dan tidak

<img width="851" height="117" alt="image_2025-10-06_17-01-47" src="https://gist.github.com/user-attachments/assets/924d3e17-375c-490b-aa41-b69df514358f" />

terlihat, ukurannya sangat berbeda jauh, bahkan untuk _base image_ tanpa _runtime_ hanya ~3MiB saja, sedangkan untuk _base image_ dengan _runtime_ java menggunakan versi yang sama yaitu 21, ukurannya sangat jauh.

### Kesimpulan

Menggunakan distroless _image_ sangat direkomendasikan untuk klaster skala produksi ***(production grade cluster)***, ini merupakan salah satu tahapan _tuning_ yang sangat bagus untuk diterapkan pada _container_.
