# 🚀 Kumpulan Tutorial Deployment

Repositori ini berisi panduan langkah-demi-langkah (*step-by-step*) untuk melakukan deployment aplikasi web, baik untuk arsitektur aplikasi tunggal (*single application*) maupun banyak aplikasi (*multi-application*) dalam satu server.

Untuk saat ini, panduan berfokus pada ekosistem **PHP** dan **Framework Laravel** menggunakan VPS (Virtual Private Server) dan Web Server Apache (`sites-available`).

---

## 📂 Struktur & Navigasi Panduan

Pilih jenis deployment yang sesuai dengan kebutuhan Anda melalui pintasan direktori di bawah ini:

### 1. PHP Biasa (Native)
Panduan deployment untuk aplikasi PHP tanpa framework (konfigurasi dasar web server dan database).
*   [➜ Deployment Single Application (PHP)](./php/single-app)
    *Panduan setup satu aplikasi PHP pada satu domain.*
*   [➜ Deployment Multi Application (PHP)](./php/multi-app)
    *Panduan setup beberapa aplikasi PHP menggunakan Apache Virtual Host.*

### 2. Laravel Framework (Menggunakan Docker)
Panduan deployment khusus Laravel yang membutuhkan konfigurasi tambahan seperti optimasi *cache*, permissions folder `storage`, dan environment files.
*   [➜ Deployment Single Application (Laravel)](./laravel/single-app)
    *Setup satu project Laravel di VPS.*
*   [➜ Deployment Multi Application (Laravel)](./laravel/multi-app)
    *Setup beberapa project Laravel (multi-domain/subdomain) dalam satu VPS menggunakan Apache.*

---

## 🛠️ Prasyarat Umum (Prerequisites)

Sebelum memulai tutorial di atas, pastikan server Anda sudah terinstall komponen dasar berikut:
*   **OS:** Ubuntu Server (atau distro Linux sejenis)
*   **Web Server:** Apache2
*   **Database:** MySQL / MariaDB
*   **PHP:** Sesuai versi yang dibutuhkan aplikasi (lengkap dengan ekstensi seperti `php-xml`, `php-mbstring`, dll.)
*   **Version Control:** Git

---

## 📝 Catatan Penting untuk Apache
Bagi Anda yang mengambil jalur **Multi Application**, pastikan untuk selalu memeriksa konfigurasi file Virtual Host Anda di direktori:
```bash
/etc/apache2/sites-available/