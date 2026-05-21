# 📂 Panduan Deployment Multi-Application PHP Biasa (Native) di cPanel

Panduan ini menjelaskan langkah-langkah untuk melakukan deploy **beberapa aplikasi PHP Native sekaligus** dalam satu akun cPanel menggunakan fitur **Subdomains / Addon Domains**, pembuatan multi-database, dan isolasi folder file manager.

---

## 🗺️ Gambaran Arsitektur Folder

Berbeda dengan VPS yang mengatur multi-app via Apache Virtual Host (`sites-available`), di cPanel kita memisahkan aplikasi berdasarkan direktori (_Document Root_) yang dihubungkan ke Subdomain/Addon Domain.

Contoh struktur di File Manager:

```bash
public_html/               --> Aplikasi Utama (Aplikasi 1) - domain-utama.com
├── app-kedua/             --> Aplikasi 2 (Subdomain)     - app2.domain-utama.com
└── app-ketiga/            --> Aplikasi 3 (Subdomain)     - app3.domain-utama.com
```

## 🛠️ Langkah-Langkah Deployment

### 1. Membuat Subdomain atau Addon Domain

Langkah pertama adalah menyediakan "wadah" berupa domain/subdomain untuk aplikasi tambahan Anda.

1. Login ke **cPanel** Anda.
2. Cari blok _Domains_ lalu pilih menu **Domains** (atau **Subdomains** tergantung versi tema cPanel Anda).
3. Klik tombol **Create A New Domain**.
4. Konfigurasi domain baru Anda:

- **Domain:** Masukkan nama subdomain (misal: `app2.domainanda.com`) atau addon domain baru.
- **Document Root (File Base):** Pastikan centang _Specify the directory_ **diaktifkan**. Tentukan nama folder isolasi untuk aplikasi ini, misalnya `public_html/app-kedua`.

5. Klik **Submit**. cPanel akan otomatis membuat folder tersebut di File Manager Anda.

---

### 2. Memisahkan dan Upload File Proyek

Setiap aplikasi PHP harus dikompres secara terpisah menjadi file `.zip`.

1. **Aplikasi 1 (Utama):** Kompres semua filenya, upload ke folder `public_html`, lalu extract di sana.
2. **Aplikasi 2 (Kedua):** Kompres semua filenya. Masuk ke folder `public_html/app-kedua/`, upload file `.zip` tersebut di sana, lalu extract.

> 💡 **Penting:** Pastikan file `index.php` dari masing-masing aplikasi berada langsung di dalam folder root-nya masing-masing, bukan terbungkus sub-folder tambahan di dalam ZIP.

---

### 3. Membuat Multi-Database di MySQL Database Wizard

Karena ini adalah multi-aplikasi, Anda memerlukan database yang berbeda untuk setiap aplikasi agar data tidak bercampur.

1. Di cPanel, buka **MySQL® Database Wizard**.
2. **Aplikasi 1:** \* Buat database (misal: `usercpanel_db_app1`), buat user baru, centang **ALL PRIVILEGES**, lalu simpan datanya.
3. **Aplikasi 2:**

- Jalankan kembali Wizard dari awal. Buat database baru (misal: `usercpanel_db_app2`).
- Anda bisa membuat user baru lagi _atau_ menggunakan user database dari Aplikasi 1 tadi untuk menghemat kuota user cPanel. Centang **ALL PRIVILEGES** untuk menghubungkan user ke database kedua ini.

---

### 4. Import Masing-Masing File SQL ke phpMyAdmin

1. Buka menu **phpMyAdmin** di cPanel.
2. Di panel sebelah kiri, Anda akan melihat beberapa database yang sudah dibuat.
3. Klik database `usercpanel_db_app1` ➜ Pilih tab **Import** ➜ Upload file `.sql` milik Aplikasi 1.
4. Klik database `usercpanel_db_app2` ➜ Pilih tab **Import** ➜ Upload file `.sql` milik Aplikasi 2.

---

### 5. Menyesuaikan File Konfigurasi Database PHP

Sekarang, Anda harus memberi tahu masing-masing aplikasi ke database mana mereka harus terhubung.

1. **Konfigurasi Aplikasi 1:**

- Buka File Manager, masuk ke `public_html/`.
- Edit file koneksi database Anda (misal: `config.php`) dan arahkan ke `usercpanel_db_app1`.

2. **Konfigurasi Aplikasi 2:**

- Masuk ke folder `public_html/app-kedua/`.
- Edit file koneksi database miliknya, dan arahkan ke `usercpanel_db_app2`.

Contoh perbedaan isi file koneksi:

```php
// Di dalam public_html/app-kedua/config.php
$host     = "localhost";
$username = "usercpanel_user_shared";
$password = "PasswordKuat123!";
$database = "usercpanel_db_app2"; // <--- Mengarah ke DB Aplikasi 2

$koneksi = mysqli_connect($host, $username, $password, $database);

```

---

### 6. Isolasi Menggunakan `.htaccess` (Opsional namun Direkomendasikan)

Terkadang, saat Anda mengakses domain utama (`domainanda.com/app-kedua`), folder aplikasi kedua masih bisa diakses secara publik lewat url domain utama. Untuk mencegah hal ini dan memaksa user mengaksesnya lewat subdomain (`app2.domainanda.com`), Anda bisa menambahkan script berikut pada file `.htaccess` di dalam folder `public_html/app-kedua/`:

```apache
RewriteEngine On
# Jika diakses menggunakan domain utama, block atau redirect
RewriteCond %{HTTP_HOST} ^(www\.)?domainanda\.com$ [NC]
RewriteRule ^(.*)$ [https://app2.domainanda.com/$1](https://app2.domainanda.com/$1) [L,R=301]

```

---

## 🚀 Pengujian Multi-App

Buka browser Anda dan lakukan pengujian akses secara terpisah:

1. Akses Aplikasi 1: `https://domainanda.com`
2. Akses Aplikasi 2: `https://app2.domainanda.com`

Pastikan kedua aplikasi berjalan normal secara independen, tidak ada error _session_ yang bertabrakan, dan data yang diinput masuk ke database masing-masing dengan benar.
