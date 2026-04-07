---
title: "CI/CD in a Nutshell: Deploy Otomatis Tanpa Repot"
date: "2026-04-07T11:45:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Pengenalan CI/CD dan mengapa development jadi lebih mudah dengan automation"
slug: "cicd-nutshell"
tags: ["devops","cicd","automation","deployment"]
---

Jadi, bayangkan dulu scenario ini: kita habis selesai coding feature baru, test lokal ternyata jalan, tapi pas di-push ke *production* malah error. Kenapa? Karena *environment* production beda, *dependency* yang terinstall beda, atau kita lupa run *migration* database. Panik deh.

Nah, *CI/CD* ini basically adalah cara kita mengotomatisasi supaya scenario kacau gitu gak terjadi. Daripada kita manually test, build, dan deploy (yang biasanya error-prone dan butuh effort besar), **kita biarkan automation tools yang handle semuanya**.

### Apa itu CI/CD?

*CI* singkatan dari *Continuous Integration*, *CD* singkatan dari *Continuous Deployment* atau *Continuous Delivery* (ada bedanya, nanti bahas).

***CI*** adalah proses dimana setiap kali kita *push* code, automation tools langsung:
- Run *tests* otomatis
- Compile/*build* code
- Check *code quality*
- Validate *dependencies*

Kalau ada yang error, kita langsung tau dan bisa fix sebelum code masuk ke *main branch*.

***CD*** adalah proses dimana kalau *CI* berhasil, code kita langsung di-*deploy* ke *production* tanpa perlu manual action. Atau minimal, **deploy jadi super simpel**, tinggal satu klik atau satu command.

### *Continuous Deployment* vs *Continuous Delivery*

Ini sering bikin orang bingung, ada bedanya:

***Continuous Delivery:*** Code yang sudah *pass* *CI* siap untuk di-*deploy*, tapi deploy ke *production* tetap manual. **Ada human decision point sebelum push ke live**.

***Continuous Deployment:*** Code yang *pass* *CI* langsung di-*deploy* otomatis ke *production*. **Gak ada step manual, semuanya automated**.

Kebanyakan orang pakai *Continuous Delivery* karena lebih aman (ada control point), *Continuous Deployment* lebih berani tapi butuh confidence tinggi dengan *test coverage*.

### Workflow Dasar *CI/CD*

Kita *push* code ke *Git* repo:

```
developer push code → Git webhook trigger → CI pipeline run tests → 
build docker image → CD push ke registry → orchestrator pull & deploy
```

Terlihat simple di atas kertas, tapi **di belakang layar ada banyak magic yang terjadi**.

### Tools yang Populer

Ada banyak tools untuk *CI/CD*, tapi ini yang paling umum:

- ***GitLab CI*** - Terintegrasi langsung dengan *GitLab*, *config* simple
- ***GitHub Actions*** - Terintegrasi dengan *GitHub*, *community-driven*
- ***Jenkins*** - Old school tapi powerful, *self-hosted*
- ***CircleCI*** - *Cloud-based*, *free tier* lumayan
- ***DroneCI*** - *Lightweight*, *self-hosted*

Untuk personal projects atau small teams, **GitHub Actions atau GitLab CI** udah lebih dari cukup. **Kedua-duanya bisa pakai untuk free**.

### Contoh Kasus Sederhana

Kita punya *repo* simple, setiap kali *push* code kita mau:

1. Run *unit tests*
2. Build *docker image*
3. *Push* ke *docker registry*
4. *Deploy* ke server

Dengan *GitLab CI*, *config*-nya di file `.gitlab-ci.yml`:

```yaml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  script:
    - npm install
    - npm test
  only:
    - merge_requests

build:
  stage: build
  script:
    - docker build -t myapp:latest .
    - docker push myregistry.com/myapp:latest
  only:
    - main

deploy:
  stage: deploy
  script:
    - ssh user@server 'cd /app && docker pull myregistry.com/myapp:latest && docker compose up -d'
  only:
    - main
```

Sekarang setiap kali kita *push* ke *branch* apapun, *tests* otomatis jalan. Kalau *merge* ke *main* dan *tests* *pass*, build *image* dan *deploy* otomatis jalan. **Gak perlu pindah-pindah window, gak perlu login SSH manual, semua handled** *pipeline*.

### Benefits yang Real

**1. Lebih Cepat Release Features**
Kalau sebelumnya *deploy* bisa ambil 2-3 jam karena manual testing dan step-by-step, **sekarang tinggal *merge* dan selesai**. *Deploy* bisa happen berkali-kali sehari.

**2. Fewer Bugs**
Karena setiap *change* langsung di-*test*, **bugs ketangkap lebih awal** sebelum sampai *production*.

**3. Consistency**
*Build* dan *deploy* *process* selalu sama, gak tergantung siapa yang handle. **Gak ada "works on my machine" excuse**.

**4. Less Stress**
Gak perlu paranoid takut ada yang lupa di-do. ***Pipeline* gak bisa lupa**.

**5. Easy Rollback**
Kalau ada issue, kita bisa *rollback* dengan satu command atau satu klik, karena **semua *deployment* *trackable***.

### Gotcha yang Perlu Diperhatian

***Test Coverage Must Be Good***
Kalau *tests*-nya jelek, *CI/CD* malah bikin masalah. **Automated deployment berarti automated disaster juga** kalau *test* gak cover cases tertentu.

***Database Migrations***
Perlu careful planning kalau *production* *database* *schema* berubah. **Gak bisa langsung run *migration* bersama *deployment***.

***Secrets Management***
*API keys*, *database* passwords, dll harus manage dengan baik. **Jangan hardcode di repo**. Gunakan *environment variables* atau *secret manager*.

***Gradually Rollout ke CD***
Jangan langsung full *Continuous Deployment* hari pertama. **Start dengan *Continuous Integration* dulu** (*auto tests*), lalu *Continuous Delivery* (auto build tapi manual *deploy*), baru ke *CD* kalau confident.

### Start Dari Mana?

Kalau baru mau implement *CI/CD*:

1. Setup *CI* dulu - hanya *tests* dan *build*
2. Add *linting* dan *code quality checks*
3. Setup *artifact management* (*docker registry*, etc)
4. Setup *staging* *environment* *deployment* (masih manual trigger)
5. Graduate ke *production* automation (dengan *safety gates*)

**Gak perlu semuanya sekaligus. Incrementally improve**.

### Real Talk

*CI/CD* itu **investment di awal**, butuh effort untuk setup dengan benar. Tapi **payoff-nya jauh lebih besar**. Kita jadi lebih produktif, less stress, dan *deploy* jadi lebih reliable.

Pernah ada periode saya dimana gak ada *CI/CD*, *deploy* cuma manual *SSH*, dan honestly itu terrible. Sekarang dengan *CI/CD* *pipeline* yang proper, bahkan kalau *deploy* 5 kali sehari gak ada stress, karena **semua automated dan semua reproducible**.

**Start small, build incrementally, dan enjoy the automated life**.
