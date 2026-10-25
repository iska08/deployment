# 🚀 Kumpulan Tutorial Deployment

Repositori ini berisi panduan langkah-demi-langkah (_step-by-step_) untuk melakukan deployment website dan aplikasi web ke berbagai jenis hosting.

Panduan mencakup deployment aplikasi **PHP** dan **Laravel** melalui FTP, cPanel, Docker, VPS, serta hosting InfinityFree.

---

## 📂 Struktur & Navigasi Panduan

Pilih jenis deployment yang sesuai dengan kebutuhan Anda melalui pintasan direktori di bawah ini:

### 1. Menggunakan Docker (Laravel Framework)

Panduan deployment khusus Laravel yang membutuhkan konfigurasi tambahan seperti optimasi _cache_, permissions folder `storage`, dan environment files.

- [➜ Deployment Single Application (Docker)](./docker-laravel/single-app)
  _Setup satu project Laravel di VPS._
- [➜ Deployment Multi Application (Docker)](./docker-laravel/multi-app)
  _Setup beberapa project Laravel (multi-domain/subdomain) dalam satu VPS menggunakan Apache._

### 2. Menggunakan FTP-CPanel (Laravel Framework)

Panduan deployment khusus Laravel yang membutuhkan konfigurasi tambahan seperti optimasi _cache_, permissions folder `storage`, dan environment files.

- [➜ Deployment Single Application (FTP-CPanel)](./ftp-cpanel-laravel/single-app)
  _Setup satu project Laravel di FTP-CPanel._
- [➜ Deployment Multi Application (FTP-CPanel)](./ftp-cpanel-laravel/multi-app)
  _Setup beberapa project Laravel (multi-domain/subdomain) dalam satu FTP-CPanel menggunakan Apache._

### 3. Menggunakan FTP di InfinityFree (PHP & HTML)

Panduan untuk mengunggah website PHP/HTML sederhana ke hosting InfinityFree menggunakan FTP, tanpa Laravel.

- [➜ Deployment PHP & HTML ke InfinityFree](./ftp-infinityfree-php-html/Readme.md)
  _Deploy otomatis website PHP/HTML dari GitHub melalui GitLab CI/CD dan FTP ke InfinityFree._

---

## 🛠️ Prasyarat

Prasyarat berbeda untuk setiap panduan. Ikuti kebutuhan server dan aplikasi yang tercantum di panduan masing-masing. Panduan VPS/Docker umumnya memerlukan:

- **OS:** Ubuntu Server (atau distro Linux sejenis)
- **Web Server:** Apache2
- **Database:** MySQL / MariaDB
- **PHP:** Sesuai versi yang dibutuhkan aplikasi (lengkap dengan ekstensi seperti `php-xml`, `php-mbstring`, dll.)
- **Version Control:** Git

Panduan InfinityFree menggunakan hosting bersama, jadi tidak memerlukan VPS atau instalasi Apache. Alur deploy otomatisnya memerlukan repository GitHub, GitLab, dan GitLab CI/CD.

---

## 📝 Catatan Penting untuk Apache

Bagi Anda yang mengambil jalur **Multi Application** di VPS, pastikan untuk selalu memeriksa konfigurasi file Virtual Host Anda di direktori:

```bash
/etc/apache2/sites-available/
```
