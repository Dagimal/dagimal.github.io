---
title: "HPA di Kubernetes: Jangan Asal Scale, Pahami Best Practice-nya"
date: "2026-07-02T10:00:00+07:00"
#menu: "main"
slug: "hpa-best-practices"
tags: ["kubernetes","hpa","autoscaling","devops","performance"]
---

Seringkali kita berpikir bahwa solusi untuk aplikasi yang lambat saat traffic naik adalah dengan mengaktifkan **HPA (Horizontal Pod Autoscaler)**. "Tinggal pasang HPA, nanti kalau CPU naik, pod otomatis nambah sendiri," begitu pikir kita.

Kelihatannya simpel, tapi kalau asal pasang tanpa strategi, HPA justru bisa jadi bumerang. Mulai dari masalah *flapping* (pod nambah-kurang secara cepat), resource cluster habis tiba-tiba, sampai aplikasi yang justru *crash* karena database tidak kuat menampung koneksi dari pod yang terlalu banyak.

### Apa itu HPA?

Singkatnya, HPA adalah fitur Kubernetes yang secara otomatis menambah atau mengurangi jumlah replika Pod berdasarkan metrik yang kita tentukan (biasanya CPU atau Memory). 

HPA bekerja dengan loop kontrol:
1. Query metrik dari *Metrics Server*.
2. Hitung jumlah replika yang dibutuhkan berdasarkan target persentase.
3. Update jumlah replika di *Deployment* atau *ReplicaSet*.

### Kapan Harus Pakai HPA? (Dan Kapan Harus Menghindarinya)

Seperti yang pernah dibahas, tidak semua service cocok pakai HPA.

**Pakai HPA jika:**
- **Aplikasi bersifat Stateless:** Tidak ada data yang tersimpan permanen di dalam pod.
- **Traffic Fluktuatif:** Ada pola lonjakan traffic yang jelas (misal: jam makan siang atau tanggal gajian).
- **Startup Cepat:** Pod bisa *ready* dalam hitungan detik.

**HINDARI HPA jika:**
- **Stateful/Database:** Menambah pod DB tanpa mekanisme replikasi data yang benar hanya akan membuat data korup atau inkonsisten.
- **Singleton/Scheduler:** Aplikasi yang tugasnya mengirim email atau menjalankan cron job. Jika ada 10 pod menjalankan cron yang sama, user akan menerima 10 email.
- **Bottleneck di Layer Lain:** Jika database sudah mencapai limit koneksi, menambah pod aplikasi justru akan memperparah keadaan (efek domino).

### Best Practice Implementasi HPA

Berikut adalah beberapa aturan main agar HPA berjalan stabil dan efektif:

#### 1. Wajib Tentukan Resource Requests
HPA tidak bisa bekerja tanpa `resources.requests`. HPA menghitung persentase penggunaan berdasarkan nilai *request*, bukan *limit*.

**Salah:**
```yaml
resources:
  limits:
    cpu: "500m"
```
*HPA tidak tahu 50% itu dari angka berapa.*

**Benar:**
```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```
*Jika target HPA adalah 50%, maka scaling akan trigger saat penggunaan CPU mencapai 100m.*

#### 2. Atur Stabilization Window (Cegah Flapping)
*Flapping* terjadi ketika pod ditambah karena traffic naik sedikit, lalu langsung dikurangi karena traffic turun sedikit, begitu seterusnya. Ini sangat boros resource dan mengganggu stabilitas.

Gunakan `behavior` untuk memperlambat proses *scale down*.

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # Tunggu 5 menit sebelum benar-benar mengurangi pod
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

#### 3. Pilih Metrik yang Tepat
Jangan terpaku hanya pada CPU.
- **CPU:** Cocok untuk aplikasi yang intensif komputasi.
- **Memory:** Hati-hati! Memory biasanya tidak turun secepat CPU (karena caching/GC). Scaling berdasarkan memory seringkali hanya menambah pod tapi tidak pernah menguranginya.
- **Custom Metrics:** Untuk aplikasi modern, gunakan metrik seperti *Request Per Second (RPS)* dari Prometheus. Ini jauh lebih akurat dalam menggambarkan beban aplikasi.

#### 4. Pastikan Readiness Probe Terkonfigurasi
HPA akan menambah pod, tapi Kubernetes harus tahu kapan pod tersebut benar-benar siap menerima traffic. Tanpa `readinessProbe`, pod yang baru *booting* akan langsung dikirimi traffic dan kemungkinan besar akan *fail*, yang kemudian memicu HPA menambah pod lagi.

#### 5. Integrasikan dengan Cluster Autoscaler (CA)
HPA hanya menambah jumlah pod. Jika node di cluster sudah penuh, pod baru akan berstatus `Pending`. HPA menjadi tidak berguna jika tidak ada tempat untuk meletakkan pod tersebut. 

Pastikan sudah mengaktifkan **Cluster Autoscaler** agar ketika ada pod `Pending`, cluster otomatis menambah node baru.

### Contoh Manifest Lengkap

Berikut adalah contoh implementasi HPA yang sudah mengikuti best practice untuk aplikasi stateless:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60 # Scale up saat CPU > 60% dari request
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # Cegah flapping
```

### Studi Kasus: Scaling On-Premise (Java Application)

Dalam lingkungan *on-premise* dengan resource yang terbatas, implementasi HPA membutuhkan ketelitian lebih dibandingkan di cloud. Ambil contoh kasus aplikasi core berbasis **Java (Spring Boot)** saat terjadi lonjakan traffic yang signifikan.

Aplikasi Java memiliki karakteristik khusus, yaitu penggunaan memory yang cukup besar untuk JVM dan adanya proses *warm-up* (JIT Compilation) sebelum mencapai performa maksimal. Hal ini membuat konfigurasi HPA menjadi lebih kritikal.

**Masalah:**
Traffic melonjak ekstrem sehingga beban CPU meningkat tajam. Jika jumlah pod di-set statis, CPU akan mencapai limit dan menyebabkan *request timeout* atau penurunan performa yang drastis.

**Implementasi Solusi:**
Konfigurasi HPA diterapkan melalui *Helm Values* agar bisa fleksibel antar cluster.

1. **Akurasi Resource Request:** Karena kapasitas node fisik terbatas, `requests` harus presisi. Jika terlalu besar, pod akan `Pending`. Jika terlalu kecil, HPA akan *trigger* terlalu awal.
2. **Headroom untuk Startup:** Mengingat aplikasi *enterprise* (misal: Java/Spring Boot) butuh waktu untuk *warm-up*, target CPU di-set pada 60% agar tersedia ruang saat pod baru sedang booting.
3. **Stabilization Window:** Menggunakan `stabilizationWindowSeconds: 600` untuk mencegah *flapping* dan memberikan stabilitas pada beban kerja yang naik-turun cepat.

**Analisis Risiko On-Premise:**
- **Node Exhaustion:** Tanpa Cluster Autoscaler otomatis, `maxReplicas` harus dibatasi agar tidak menghabiskan seluruh resource node yang bisa berdampak pada service lain (eviction).
- **Database Connection Pool:** Scaling pod dari 3 menjadi 12 berarti jumlah koneksi ke database juga naik 4x lipat. Pastikan `max_connections` pada database sudah di-tuning.
- **Warm-up Period:** Wajib menggunakan `readinessProbe` agar traffic tidak masuk ke pod yang belum siap.

### Real Talk

HPA bukan "tombol ajaib" yang membuat aplikasi otomatis kencang. HPA hanyalah alat untuk mengelola jumlah replika. Performa asli tetap bergantung pada optimasi kode, efisiensi query database, dan konfigurasi resource yang tepat.

Saran saya: **mulailah dengan observasi**. Gunakan Prometheus dan Grafana untuk melihat pola traffic aplikasi. Tentukan nilai `requests` yang realistis, baru kemudian terapkan HPA. Jangan pernah mengaktifkan HPA di production tanpa mencoba *load test* terlebih dahulu untuk melihat bagaimana aplikasi bereaksi saat jumlah pod bertambah secara masif.
