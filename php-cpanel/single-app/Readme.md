# 📂 Panduan Deployment Aplikasi PHP Biasa (Native) di cPanel

Panduan ini menjelaskan langkah-langkah praktis untuk melakukan deployment aplikasi PHP Native (tanpa framework) ke dalam **cPanel / Shared Hosting** secara manual melalui File Manager dan MySQL Database Wizard.

---

## 🔗 Prasyarat Sebelum Deployment

Sebelum memulai, pastikan Anda telah menyiapkan:

- Akun cPanel yang aktif beserta informasi loginnya.
- File proyek PHP yang sudah dikompres menjadi format **.zip** (bukan .rar).
- File database (ekstensi `.sql`) jika aplikasi menggunakan basis data.

---

## 🛠️ Langkah-Langkah Deployment

### 1. Mempersiapkan File Proyek (.zip)

1. Buka folder proyek PHP di komputer Anda.
2. Blok seluruh file dan folder di **dalam** proyek tersebut (seperti `index.php`, folder `assets`, folder `config`, dll.).
3. Klik kanan dan kompres menjadi sebuah file `.zip` (misal: `arsip-proyek.zip`).
   > 💡 **Penting:** Pastikan file `index.php` berada di root/luar arsip `.zip`, tidak terbungkus di dalam sub-folder lagi.

---

### 2. Upload File ke File Manager cPanel

1. Login ke akun **cPanel** Anda.
2. Cari dan pilih menu **File Manager** di bagian _Files_.
3. Masuk ke direktori utama website Anda:
   - Jika untuk domain utama, masuk ke folder **`public_html`**.
   - Jika untuk subdomain, masuk ke folder sesuai nama subdomain yang telah dibuat.
4. Klik tombol **Upload** di bagian menu atas.
5. Pilih file `arsip-proyek.zip` yang sudah Anda buat sebelumnya dan tunggu hingga proses unggah selesai (bar indikator berwarna hijau).
6. Kembali ke File Manager, klik kanan pada file `.zip` tersebut, lalu pilih **Extract**.
7. Pastikan seluruh file proyek Anda kini sudah berada langsung di bawah direktori `public_html`.

---

### 3. Membuat Database Melalui MySQL Database Wizard

Jika aplikasi Anda menggunakan database MySQL, lakukan konfigurasi berikut:

1. Kembali ke halaman utama cPanel, cari blok _Databases_ dan pilih **MySQL® Database Wizard**.
2. **Step 1: Create A Database** — Masukkan nama database Anda (misal: `db_toko`), lalu klik _Next Step_.
3. **Step 2: Create Database Users** — Masukkan username database (misal: `user_toko`) dan buat password yang kuat. Catat username dan password ini. Klik _Create User_.
4. **Step 3: Add User to the Database** — Centang pilihan **ALL PRIVILEGES** untuk memberikan hak akses penuh kepada user tersebut ke database. Klik _Make Changes_ / _Next Step_.

---

### 4. Import File SQL ke phpMyAdmin

1. Kembali ke halaman utama cPanel, cari blok _Databases_ dan pilih **phpMyAdmin**.
2. Pada panel sebelah kiri, klik nama database baru yang baru saja Anda buat di Langkah 3.
3. Klik tab **Import** di bagian menu atas.
4. Pada bagian _File to import_, klik **Choose File** dan pilih file database `.sql` dari komputer Anda.
5. Gulir ke bawah halaman dan klik tombol **Import** atau **Go**. Tunggu hingga muncul notifikasi keberhasilan berwarna hijau.

---

### 5. Menyesuaikan Konfigurasi Koneksi Database PHP

Setelah database siap, Anda harus mengubah kredensial koneksi di dalam script PHP agar sesuai dengan database cPanel.

1. Buka kembali **File Manager** dan menuju folder tempat aplikasi di-extract.
2. Cari file konfigurasi database Anda (biasanya bernama `koneksi.php`, `config.php`, atau `database.php`).
3. Klik kanan pada file tersebut dan pilih **Edit**.
4. Sesuaikan variabel koneksi MySQL Anda. Umumnya polanya seperti berikut:

```php
<?php
$host     = "localhost";          // Biasanya tetap localhost di cPanel
$username = "usercpanel_user_toko"; // Sesuaikan dengan Step 3 (ada prefix user cPanel)
$password = "PasswordAnda123!";     // Sesuaikan dengan password di Step 3
$database = "usercpanel_db_toko";   // Sesuaikan dengan Step 3 (ada prefix user cPanel)

$koneksi = mysqli_connect($host, $username, $password, $database);

if (mysqli_connect_errno()) {
    die("Koneksi database gagal: " . mysqli_connect_error());
}
?>
```

5. Klik Save Changes di pojok kanan atas.

6. Menyesuaikan Versi PHP (Opsional)
   Jika aplikasi Anda membutuhkan versi PHP spesifik agar tidak memunculkan error deprecated:

Di halaman utama cPanel, cari fitur Select PHP Version atau MultiPHP Manager.

Pilih domain atau direktori website Anda.

Ubah versi PHP (misalnya ke PHP 8.1 atau PHP 8.2) sesuai dengan spesifikasi kode lokal Anda.

Klik Apply atau Set as Current.

🚀 Pengujian
Buka browser Anda dan akses domain Anda (misal: https://domain-anda.com). Pastikan halaman utama terbuka dengan benar dan fitur yang terhubung ke database berjalan tanpa kendala error Access Denied atau Database Connection Failed.
