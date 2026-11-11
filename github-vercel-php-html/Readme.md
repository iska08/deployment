# Panduan Deploy Website ke Vercel

Panduan ini menjelaskan cara memindahkan dan mendeploy website Anda secara otomatis menggunakan **Vercel**, mulai dari membuat akun, menghubungkan GitHub, hingga proses _deployment_ instan tanpa ribet masalah FTP.

```bash
Push ke GitHub (main)
        |
        v
    Vercel (Auto-Build & Deploy)

```

## 1. Buat Akun Vercel

1. Buka situs resmi [Vercel](https://vercel.com/).
2. Klik tombol **Sign Up** (Daftar) di pojok kanan atas.
3. Disarankan untuk memilih **Continue with GitHub** agar akun Vercel Anda terhubung langsung secara otomatis dengan repository GitHub Anda.
4. Ikuti instruksi verifikasi hingga Anda masuk ke Dashboard utama [Vercel](https://vercel.com/iska08s-projects).

## 2. Persiapan Repository GitHub

1. Pastikan kode website Anda (baik HTML, CSS, JavaScript, maupun file PHP jika menggunakan runtime PHP Vercel) sudah berada di dalam repository GitHub pada branch utama **`main`**.
2. Pastikan file halaman utama seperti `index.html` atau `index.php` berada di root folder project.

Contoh struktur project:

```bash
website-saya/
|-- index.html (atau index.php)
|-- tentang.html
`-- style.css
```

## 3. Impor Project ke Vercel

1. Di Dashboard [Vercel](https://vercel.com/iska08s-projects), klik tombol **Add New...** lalu pilih **Project**.
2. Pada bagian _Import Git Repository_, cari nama repository GitHub Anda, lalu klik tombol **Import** di sampingnya.
3. Vercel akan otomatis mendeteksi jenis project Anda.

- Jika menggunakan **HTML/CSS/JS murni (Statik)**, biarkan pengaturan build standar.
- Jika menggunakan **PHP**, pastikan Anda menambahkan konfigurasi `vercel.json` di root folder project Anda (lihat poin 4 di bawah).

4. Klik **Deploy** dan tunggu beberapa detik hingga proses pembangunan (_building_) selesai.

## 4. Konfigurasi Khusus untuk File PHP (Opsional)

Jika website Anda menggunakan file `.php`, buat file bernama **`vercel.json`** di root repository GitHub Anda dengan isi konfigurasi agar Vercel mengenali file PHP:

```bash
{
  "routes": [
    { "handle": "filesystem" },
    { "src": "/(.*)", "dest": "/index.php" }
  ]
}
```

_(Lewati langkah ini jika website Anda murni HTML, CSS, dan JavaScript statis)._

## 5. Uji Deploy Otomatis

1. Setiap kali Anda melakukan perubahan kode di komputer, lakukan `git push` ke branch `main` di GitHub.
2. Vercel secara otomatis akan mendeteksi perubahan tersebut, mengunduh pembaruannya, dan mendeploy ulang website Anda secara _real-time_.
3. Anda akan mendapatkan URL domain gratis berakhiran `.vercel.app` (misalnya `website-anda.vercel.app`) yang bisa langsung dibagikan ke publik. Anda juga bisa mengatur custom domain sendiri melalui menu **Domains** di pengaturan project Vercel.
