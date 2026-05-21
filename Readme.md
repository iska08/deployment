# 🚀 Kumpulan Tutorial Deployment

Repositori ini berisi panduan langkah-demi-langkah (_step-by-step_) untuk melakukan deployment aplikasi web, baik untuk arsitektur aplikasi tunggal (_single application_) maupun banyak aplikasi (_multi-application_) dalam satu server.

Untuk saat ini, panduan berfokus pada ekosistem **PHP** dan **Framework Laravel** menggunakan VPS (Virtual Private Server) dan Web Server Apache (`sites-available`).

---

## 📂 Struktur & Navigasi Panduan

Pilih jenis deployment yang sesuai dengan kebutuhan Anda melalui pintasan direktori di bawah ini:

### 1. Laravel Framework (Menggunakan Docker)

Panduan deployment khusus Laravel yang membutuhkan konfigurasi tambahan seperti optimasi _cache_, permissions folder `storage`, dan environment files.

- [➜ Deployment Single Application (Laravel)](./laravel-docker/single-app)
  _Setup satu project Laravel di VPS._
- [➜ Deployment Multi Application (Laravel)](./laravel-docker/multi-app)
  _Setup beberapa project Laravel (multi-domain/subdomain) dalam satu VPS menggunakan Apache._

---

## 🛠️ Prasyarat Umum (Prerequisites)

Sebelum memulai tutorial di atas, pastikan server Anda sudah terinstall komponen dasar berikut:

- **OS:** Ubuntu Server (atau distro Linux sejenis)
- **Web Server:** Apache2
- **Database:** MySQL / MariaDB
- **PHP:** Sesuai versi yang dibutuhkan aplikasi (lengkap dengan ekstensi seperti `php-xml`, `php-mbstring`, dll.)
- **Version Control:** Git

---

## 📝 Catatan Penting untuk Apache

Bagi Anda yang mengambil jalur **Multi Application**, pastikan untuk selalu memeriksa konfigurasi file Virtual Host Anda di direktori:

```bash
/etc/apache2/sites-available/
```
