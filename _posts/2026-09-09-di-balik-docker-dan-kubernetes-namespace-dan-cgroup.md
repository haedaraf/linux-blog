---
layout: post
title: "Di Balik Docker dan Kubernetes: Sebenarnya Cuma Namespace dan Cgroup"
date: 2026-09-09
excerpt: "Docker dan Kubernetes terasa seperti sihir, padahal tidak ada kode khusus container di kernel Linux. Yang bekerja sebenarnya cuma dua mekanisme lawas — namespace dan cgroup."
image: /assets/images/covers/namespace-cgroup-og.png
cover: /assets/images/covers/namespace-cgroup.svg
---

Dulu saya kira Docker punya semacam "mode container" rahasia yang tertanam di kernel. Ternyata
tidak. Buka source code kernel Linux, cari kata `container` — tidak akan ketemu subsistem apa pun,
tidak ada system call khusus. Docker dan Kubernetes bukan penemu container, dan mereka juga tidak
dapat bantuan istimewa dari kernel yang tidak bisa kamu pakai sendiri. Yang sebenarnya mereka
lakukan adalah mengendalikan dua fitur lawas yang jauh lebih membosankan, yang sudah ada di Linux
lebih dari satu dekade: **namespace** dan **cgroup**. Sisanya — image, registry, manifest YAML,
scheduler — cuma tooling yang dibungkus di atas dua hal itu.

## Sebenarnya apa yang diisolasi?

Gimana caranya satu proses Linux bisa seolah-olah punya mesinnya sendiri — RAM sendiri, CPU
sendiri, bahkan serasa punya OS sendiri — padahal jelas-jelas jalan berdampingan dengan ratusan
proses lain di kernel yang sama?

Itu masalah yang harus dipecahkan sebuah container: harus terasa seperti mesin kecilnya sendiri —
punya filesystem sendiri, daftar proses sendiri, hostname sendiri, dan jatah CPU/memori sendiri
supaya tidak menghabiskan sumber daya proses lain di mesin yang sama. Tapi kalau dikupas dari
segala pemasaran, container itu cuma proses biasa, jalan di kernel yang sama persis dengan semua
proses lain di host itu — jadi ia harus *mengira* dirinya punya mesin sendirian, sekaligus tetap
*berperilaku* hanya mengambil jatah yang adil.

Ternyata kernel sudah membagi tugas itu jadi dua bagian rapi. Namespace menentukan apa yang boleh
**dilihat** oleh sebuah proses. Cgroup menentukan apa yang boleh **dipakainya**. Itu saja intinya.

## Namespace: jawaban agar aplikasi seolah-olah jalan di server terpisah

Namespace mengambil satu sumber daya global yang dikelola kernel — process ID, antarmuka jaringan,
mount point — lalu membungkusnya sehingga proses di dalam namespace itu hanya melihat salinan
privatnya sendiri, bukan yang asli milik host. Efeknya persis seperti aplikasi itu dipindah ke
server sendiri, padahal fisiknya tidak ke mana-mana. Linux punya beberapa jenis namespace: `pid`
menyembunyikan proses-proses lain di sistem sehingga container mengira aplikasinya adalah PID 1;
`net` memberinya stack jaringan sendiri; `mnt` memberinya pandangan filesystem sendiri; `uts`
membiarkannya mengatur hostname sendiri tanpa menyentuh punya host; `ipc` menjaga shared memory
dan message queue tetap privat; dan `user` membuat "root" di dalam container jadi cuma pengguna
biasa yang tidak punya hak istimewa apa pun di host.

Saat Docker menjalankan container, di baliknya ia cuma memanggil `clone()` dengan sederet flag
`CLONE_NEW*`, dan proses hasilnya mendapat namespace baru yang kosong, bukan ikut namespace milik
host. Itulah sebabnya `ps aux` di dalam container cuma menampilkan segelintir proses, walaupun
host-nya mungkin sedang menjalankan ribuan proses lain — container-nya memang secara harfiah tidak
bisa melihat yang lain.

Kamu tidak perlu Docker untuk mencobanya sendiri. Ini jalan di mesin Linux biasa:

```bash
# Jalankan shell di namespace UTS, PID, dan mount yang baru
sudo unshare --uts --pid --mount --fork /bin/bash

# Di dalamnya, shell ini benar-benar mengira dirinya PID 1
ps aux
```

Itu saja. Itulah "isolasi container." Tidak ada hypervisor, tidak ada perangkat keras virtual,
tidak ada sihir — cuma kernel yang memberi proses pandangan privat atas state yang biasanya
dibagikan ke semua orang.

## Cgroup: memastikan tidak ada yang rakus sendiri

Namespace saja tidak cukup untuk mencegah satu container menghabiskan seluruh CPU atau memori di
host — namespace cuma mengatur apa yang terlihat, bukan seberapa banyak yang dipakai. Di sinilah
**cgroup** (control group) berperan. Cgroup mengelompokkan sekumpulan proses lalu membiarkan
kernel membatasi, mengukur, dan memprioritaskan apa yang boleh dikonsumsi kelompok itu secara
keseluruhan.

Contoh konkretnya, kalau kamu mau batasi jatah container sampai ketat:

```bash
docker run --cpus="1.5" --memory="512m" my-image
```

Docker tidak mengawasi batas itu sendiri di level userspace. Ia menuliskannya ke filesystem
cgroup, dan setelah itu scheduler serta memory manager kernel-lah yang menegakkannya, sama seperti
untuk kelompok proses lain mana pun. Di sistem cgroup v2, kamu bisa langsung melihat file-nya
sendiri:

```bash
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/cpu.max
```

Angka-angka itu *adalah* batasannya. Tidak ada "polisi runtime container" terpisah yang mengawasi
proses tersebut — itu murni akuntansi cgroup, melakukan apa yang selalu dilakukannya.

## Kalau kernel sudah melakukan semua itu, Docker buat apa?

Pertanyaan bagus. Kalau namespace menangani isolasi dan cgroup menangani batasan, tugas Docker
sebenarnya adalah semua hal *di sekitar* dua primitif itu, bukan penggantinya. Docker memberi
format image yang portabel — layer di atas `overlayfs` — supaya ada sesuatu yang konsisten untuk
di-mount ke namespace `mnt` yang baru. Ia menjalankan daemon yang berbicara ke runtime tingkat
rendah bernama `runc` (yang mengimplementasikan
[spesifikasi OCI runtime](https://github.com/opencontainers/runtime-spec)) supaya kamu tidak
perlu memanggil `clone()` dan mengatur cgroup secara manual. Ia juga merangkai antarmuka jaringan
virtual dan aturan NAT supaya container-container yang berada di namespace `net` terpisah tetap
bisa saling terhubung dan menjangkau dunia luar. Dan ia menyediakan registry — tempat untuk push
dan pull image-image itu.

Kubernetes berada satu lapis lebih tinggi lagi. Ia sama sekali tidak menyentuh namespace atau
cgroup secara langsung — ia berbicara ke container runtime lewat CRI (`containerd`, `CRI-O`, dan
sejenisnya), yang lalu memanggil `runc`, yang baru berbicara ke kernel. Yang sebenarnya diputuskan
Kubernetes adalah *node mana* tempat sebuah pod ditempatkan, dan *berapa banyak* jatah cgroup yang
didapatnya (itulah `resources.requests` dan `resources.limits` di spesifikasi pod). Ini juga
alasan kenapa container-container dalam satu pod berbagi `localhost` — mereka sengaja ditaruh di
namespace `net` yang sama. Kubernetes adalah orkestrasi yang duduk di atas primitif-primitif yang
sudah dimiliki kernel jauh sebelum istilah "cloud native" dipakai orang.

## Kenapa semua ini penting diketahui

Begitu paham bahwa "container itu cuma proses dengan namespace berbeda dan sebuah cgroup," banyak
hal yang tadinya terasa seperti trivia acak soal Docker jadi masuk akal dengan sendirinya. `docker
run --pid=host` membuat container bisa melihat semua proses di host karena ia memang cuma
melewatkan pembuatan namespace PID baru — tidak pernah ada "mode visibilitas host" yang istimewa,
cuma satu tembok lebih sedikit yang dibangun. Container dulu bisa membaca `/proc/meminfo` dan
melihat total memori *host*, bahkan ketika batas `--memory` sudah diset, karena cgroup membatasi
pemakaian, bukan mengubah apa yang dilaporkan `/proc` — ketidaksesuaian ini sempat bikin banyak JVM
bermasalah sebelum tooling-nya menyusul. Kerentanan "container escape" pada dasarnya cuma
kesalahan konfigurasi namespace atau capability, bukan model keamanan eksotis khusus container.
Dan container bisa mulai dalam hitungan milidetik serta jalan tanpa ekstensi virtualisasi karena
memang tidak ada hypervisor sama sekali dalam gambarannya.

Semua ini tidak membuat Docker atau Kubernetes jadi kurang berguna — packaging, scheduling,
self-healing, dan jaringan dalam skala besar tetaplah masalah sulit yang layak diselesaikan
untukmu. Tapi bagian isolasi, bagian yang terasa seperti sihir itu, sebenarnya bukan milik mereka.
Itu Linux, melakukan apa yang sudah diam-diam dilakukannya sejak kernel 2.6.24 merampungkan
namespace inti terakhir. Container bukan jenis benda baru yang berjalan di mesinmu. Mereka cuma
cara baru yang lebih ramah untuk meminta kernel lama yang sama menyembunyikan dan membagi sumber
daya atas namamu.
