# Panduan Deploy Website HTML Statis ke Vercel

Panduan ini menyiapkan alur deployment otomatis: setiap perubahan kode di branch `main` pada GitHub akan langsung di-build dan di-deploy secara instan ke [Vercel](https://vercel.com/).

```bash
Push ke GitHub (main)
        |
        v
 Vercel (Auto-Build & Deploy)

```

## 1. Siapkan Repository GitHub

1. Pastikan seluruh file website Anda (seperti `index.html`, halaman-halaman HTML lainnya, file CSS, dan JavaScript) berada di root folder repository GitHub.
2. Gunakan branch utama bernama `main`.
3. Pastikan halaman utama website Anda wajib bernama **`index.html`** di root project.

Contoh struktur project:

```bash
website-saya/
|-- index.html
|-- tentang.html
`-- style.css

```

---

## 2. Buat Akun Vercel

1. Buka situs resmi [Vercel](https://vercel.com/).
2. Klik tombol **Sign Up** di pojok kanan atas.
3. Pilih **Continue with GitHub** agar akun Vercel Anda terhubung langsung secara otomatis dengan repository GitHub Anda.
4. Ikuti instruksi verifikasi hingga Anda masuk ke Dashboard utama [Vercel](https://vercel.com/).

---

## 3. Impor Project ke Vercel

1. Di Dashboard [Vercel](https://vercel.com/), klik tombol **Add New...** lalu pilih **Project**.
2. Pada bagian *Import Git Repository*, cari repository GitHub Anda, lalu klik tombol **Import** di sampingnya.
3. Pada pengaturan *Configure Project*:
* **Project Name:** Biarkan default atau sesuaikan.
* **Framework Preset:** Vercel akan otomatis mendeteksi project Anda sebagai **Other** atau **Static Site**. Biarkan pengaturan tersebut apa adanya.
* **Build and Output Settings:** Kosongkan karena ini adalah website HTML statis murni yang tidak memerlukan proses *build*.


4. Klik tombol **Deploy** dan tunggu beberapa detik hingga proses selesai.

---

## 4. Uji Deploy Otomatis

1. Setiap kali Anda melakukan perubahan kode (misalnya mengedit file HTML atau CSS) di komputer, lakukan `git push` ke branch `main` di GitHub.
2. Vercel secara otomatis akan mendeteksi perubahan tersebut dan memperbarui website Anda secara *real-time*.
3. Anda akan mendapatkan URL domain publik berakhiran `.vercel.app` yang bisa langsung dikunjungi atau dibagikan ke publik.

---

## Pemecahan Masalah

* **File terunduh alih-alih tampil sebagai web:** Pastikan halaman utama Anda benar-benar berekstensi `.html` (bukan `.php` atau ekstensi file mentah lainnya) dan sudah di-push dengan benar ke branch `main`.
* **Halaman 404 (Not Found):** Pastikan file utama Anda bernama `index.html` dan diletakkan persis di root direktori project, bukan di dalam folder tersembunyi.