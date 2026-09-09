---
layout: default
title: Kontribusi
permalink: /kontribusi/
---

# Kontribusi

**linuxmd** terbuka untuk tulisan dari siapa saja — selama kontennya soal Linux dan sekitarnya:
kernel, sistem, container, jaringan, shell, tooling, sampai cerita di balik layar sebuah masalah
yang berhasil kamu pecahkan. Situs ini jalan di atas Jekyll dan kontennya cuma file Markdown di
repo GitHub, jadi cara publish artikel baru ya lewat **pull request (PR)**, seperti kontribusi ke
proyek open source lain.

## Alur singkatnya

1. **Fork** repo [`linuxmd-id/blog`](https://github.com/linuxmd-id/blog) ke akun GitHub-mu.
2. Buat branch baru dari `main`, misalnya `tulisan/nama-topikmu`.
3. Tambahkan satu file Markdown baru di folder `_posts/`.
4. Commit, push ke fork-mu, lalu buka PR ke branch `main` di repo asli.
5. Tunggu review — kalau ada revisi kecil, biasanya cukup ditambahkan lewat commit baru di PR
   yang sama.

```bash
git clone git@github.com:<username-kamu>/blog.git
cd blog
git checkout -b tulisan/nama-topikmu
# ...tulis artikelnya...
git add _posts/2026-09-20-judul-artikelmu.md
git commit -m "Tambah artikel: judul artikelmu"
git push origin tulisan/nama-topikmu
```

Lalu buka PR dari fork-mu ke `linuxmd-id/blog` branch `main` di GitHub.

## Format file

Nama file harus mengikuti format Jekyll: `TAHUN-BULAN-TANGGAL-judul-singkat.md`, ditaruh di
`_posts/`. Judul di nama file dipakai sebagai bagian URL artikel, jadi pakai huruf kecil dan
tanda hubung (`-`), bukan spasi atau garis bawah.

```
_posts/2026-09-20-cara-kerja-systemd-unit.md
```

Setiap file wajib punya front matter di baris paling atas seperti ini:

```yaml
---
layout: post
title: "Judul Artikelmu di Sini"
date: 2026-09-20
excerpt: "Satu-dua kalimat yang muncul di daftar artikel dan halaman beranda."
---
```

Isi artikel ditulis pakai Markdown biasa di bawah front matter itu — heading dengan `##`, kode
dengan ``` ```, tautan dengan `[teks](url)`, dan seterusnya.

## Gaya tulisan

- **Bahasa Indonesia**, gaya santai tapi jelas — anggap kamu sedang menjelaskan ke teman satu tim,
  bukan menulis dokumentasi resmi. Boleh pakai "kamu", boleh pakai contoh personal.
- Utamakan penjelasan **cara kerja**, bukan sekadar tutorial "copy-paste perintah ini". Kalau ada
  contoh command atau kode, pastikan sudah kamu coba sendiri dan benar-benar jalan.
- Kalau mengklaim sesuatu yang teknis (versi kernel, perilaku spesifik, angka), sertakan sumber
  atau cara pembaca bisa verifikasi sendiri.
- Artikel tidak harus panjang. Yang penting jelas dan jujur soal apa yang kamu tahu dan tidak tahu.
- Hindari plagiarisme — parafrase dengan pemahamanmu sendiri, dan kalau mengutip, sebutkan
  sumbernya.

## Yang dicek saat review

- Apakah kontennya relevan dengan tema Linux/sistem/infrastruktur.
- Apakah contoh perintah/kode akurat dan aman untuk dicoba pembaca.
- Front matter lengkap dan formatnya benar (`layout`, `title`, `date`, `excerpt`).
- Tidak ada informasi yang menyesatkan atau mengarah ke tindakan berbahaya (misalnya perintah
  destruktif tanpa peringatan).

## Pertanyaan?

Kalau ragu sebelum mulai nulis — mau ambil topik apa, atau bingung soal formatnya — buka saja
**issue** di repo [`linuxmd-id/blog`](https://github.com/linuxmd-id/blog/issues) dulu sebelum
capek-capek nulis draft penuh.
