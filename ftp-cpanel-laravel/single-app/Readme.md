# Panduan Deployment Single-Application Laravel di _FTP-cPanel_

Berikut adalah panduan lengkap dari awal yang telah disesuaikan menggunakan **PHP 8.3**, dilengkapi dengan build **Node.js/Vite**, konfigurasi **database**, dan **Cron Job** di cPanel.

---

## 1\. Mirroring Repository GitHub ke GitLab Private

### Langkah 1.1: Buat Project Private Baru di GitLab

1. Login ke [GitLab](https://gitlab.com).
2. Klik **New project** > **Create blank project**.
3. Isi **Project name** (contoh: `laravel-app`).
4. Set **Visibility Level** ke **Private**.
5. Hilangkan centang pada _Initialize repository with a README_.
6. Klik **Create project**.

### Langkah 1.2: Buat Personal Access Token (PAT) di GitLab

1. Di GitLab, klik foto profil (kiri bawah) > **Preferences** > **Access Tokens**.
2. Klik **Add new token**:

- **Token name:** `github-mirror-token`
- **Select scopes:** Centang **`write_repository`** dan **`read_repository`**.

3. Klik **Create personal access token**, lalu **salin token tersebut**.

### Langkah 1.3: Buat Secret di GitHub

1. Buka repo Laravel Anda di [GitHub](https://github.com).
2. Masuk ke **Settings** > **Secrets and variables** > **Actions**.
3. Klik **New repository secret**:

- **Name:** `GITLAB_MIRROR_TOKEN`
- **Secret:** _Paste token dari Langkah 1.2_

4. Klik **Add secret**.

### Langkah 1.4: Workflow Mirroring di GitHub

Di project lokal Anda, buat file `.github/workflows/mirror-to-gitlab.yml`:

```bash
name: Mirroring to GitLab

on:
  push:
    branches:
      - main # Sesuaikan jika nama branch utama Anda 'master'

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Push to GitLab
        env:
          GITLAB_TOKEN: ${{ secrets.GITLAB_MIRROR_TOKEN }}
          GITLAB_REPO: gitlab.com/USERNAME_GITLAB/NAMA_REPO_GITLAB.git
        run: |
          git remote add gitlab https://oauth2:${GITLAB_TOKEN}@${GITLAB_REPO}
          git push --prune gitlab +refs/remotes/origin/*:refs/heads/* +refs/tags/*:refs/tags/*
```

---

## 2\. Persiapan cPanel, Routing `.htaccess`, & Database

### Langkah 2.1: Routing Direct `public/` via `.htaccess`

Di root direktori project Laravel Anda (sejajar `composer.json`), buat atau sesuaikan file `.htaccess`:

```bash
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Arahkan semua request ke folder public/ Laravel
    RewriteCond %{REQUEST_URI} !^/public/
    RewriteRule ^$ public/ [L]
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

### Langkah 2.2: Buat Akun FTP di cPanel

1. Login ke **cPanel** > buka menu **FTP Accounts**.
2. Isi detail akun:

- **Log In:** `deployer`
- **Domain:** Pilih domain/subdomain target.
- **Password:** Generate password (simpan password ini).
- **Directory:** Arahkan ke root target domain (contoh: `public_html`).

3. Klik **Create FTP Account**.

### Langkah 2.3: Setup Database MySQL di cPanel

1. Buka menu **MySQL Database Wizard** di cPanel.
2. **Step 1:** Buat nama database (contoh: `user_laraveldb`).
3. **Step 2:** Buat user database dan password (contoh: `user_dbuser`).
4. **Step 3:** Centang **ALL PRIVILEGES**, lalu klik **Make Changes**.

---

## 3\. Pendaftaran Route Webhook di Laravel (`routes/api.php`)

### Langkah 3.1: Aktifkan Ekstensi PHP Zip:

- Buka **cPanel** > **Select PHP Version**.
- Di tab **Extensions**, pastikan modul `zip` sudah dicentang/diaktifkan.

### Langkah 3.2: Buka file `routes/api.php` pada project Laravel Anda dan tambahkan endpoint webhook di bagian bawah:

```bash
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Route;

Route::get('/deploy-webhook-secret-123', function (Request $request) {
    set_time_limit(300);
    ini_set('memory_limit', '512M');

    $zipPath = base_path('vendor.zip');
    $vendorPath = base_path('vendor');
    $messages = [];

    // 1. Ekstrak vendor.zip jika file ada
    if (file_exists($zipPath)) {

        // Paksa hapus folder vendor menggunakan perintah Linux
        if (file_exists($vendorPath)) {
            exec("rm -rf " . escapeshellarg($vendorPath) . " 2>&1", $outputRm, $returnRm);
            if ($returnRm === 0) {
                $messages[] = 'Folder vendor lama berhasil dihapus (via shell rm).';
            } else {
                $messages[] = 'Gagal menghapus folder vendor lama: ' . implode(" ", $outputRm);
            }
        }

        // Ekstrak vendor.zip menggunakan perintah Linux 'unzip'
        exec("unzip -o " . escapeshellarg($zipPath) . " -d " . escapeshellarg(base_path()) . " 2>&1", $outputUnzip, $returnUnzip);

        if ($returnUnzip === 0) {
            @unlink($zipPath); // Hapus zip setelah selesai
            $messages[] = 'vendor.zip berhasil diekstrak (via shell unzip).';
        } else {
            $messages[] = 'Gagal ekstrak vendor.zip: ' . implode(" ", $outputUnzip);
        }

    } else {
        $messages[] = 'File vendor.zip tidak ditemukan (skip vendor update).';
    }

    // 2. Jalankan Migration
    try {
        Artisan::call('migrate', ['--force' => true]);
        $messages[] = 'Database migration executed successfully.';
    } catch (\Exception $e) {
        $messages[] = 'Database migration failed: ' . $e->getMessage();
    }

    // 3. Clear Cache
    Artisan::call('config:clear');
    Artisan::call('route:clear');
    Artisan::call('view:clear');
    $messages[] = 'Laravel caches cleared successfully.';

    return response.json(['status' => 'success', 'details' => $messages]);
});
```

---

## 4\. Konfigurasi CI/CD Pipeline GitLab (`.gitlab-ci.yml`)

### Langkah 4.1: Tambahkan Variables di GitLab

Masuk ke **GitLab** > **Settings** > **CI/CD** > **Variables** > **Add variable**:

| Key            | Value                                  | Flags  |
| -------------- | -------------------------------------- | ------ |
| `FTP_SERVER`   | `ftp.nama-domain.com` (atau IP Server) | Masked |
| `FTP_USERNAME` | `deployer@nama-domain.com`             | Masked |
| `FTP_PASSWORD` | _Password FTP dari Langkah 2.2_        | Masked |

### Langkah 4.2: Buat File `.gitlab-ci.yml`

Buat file `.gitlab-ci.yml` di root repository Laravel Anda:

```bash
image: php:8.3-cli

stages:
  - build
  - deploy

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - node_modules/

build_job:
  stage: build
  script:
    # 1. Update & Install System Dependencies
    - apt-get update -yqq
    - apt-get install -yqq git unzip libpng-dev libonig-dev libxml2-dev libzip-dev zip curl

    # 2. Install Node.js & NPM
    - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
    - apt-get install -yqq nodejs

    # 3. Install Extension PHP
    - docker-php-ext-install pdo_mysql mbstring gd zip

    # 4. Build CSS & JS (Vite/Mix)
    - npm ci || npm install
    - npm run build

    # 5. Cek perubahan composer.json / composer.lock
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - |
      if git diff --name-only HEAD~1 HEAD | grep -E "composer\.(json|lock)"; then
        echo "Composer file changed. Installing vendor and creating vendor.zip..."
        composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts --optimize-autoloader
        zip -r vendor.zip vendor/
      else
        echo "No changes in composer files. Skipping vendor.zip creation."
      fi

  artifacts:
    paths:
      - vendor.zip
      - public/build/
    expire_in: 1 hour
  only:
    - main

deploy_ftp_job:
  stage: deploy
  dependencies:
    - build_job
  before_script:
    - apt-get update -yqq
    - apt-get install -yqq lftp curl
  script:
    # 1. Sync file via FTP (Mencakup file kodingan & file migration baru di database/migrations/)
    - |
      lftp -c "
      set ftp:ssl-allow true;
      set ftp:ssl-force false;
      set ssl:verify-certificate no;
      open -u $FTP_USERNAME,$FTP_PASSWORD $FTP_SERVER;
      mirror --reverse \
             --only-newer \
             --ignore-time \
             --verbose \
             --exclude-glob .git/ \
             --exclude-glob .github/ \
             --exclude-glob .gitlab-ci.yml \
             --exclude-glob .env \
             --exclude-glob node_modules/ \
             --exclude-glob vendor/ \
             --exclude-glob storage/logs/ \
             --exclude-glob storage/framework/cache/ \
             --exclude-glob storage/framework/sessions/ \
             --exclude-glob storage/framework/views/ \
             ./ /;
      quit
      "

    # 2. Trigger Webhook cPanel (Ekstrak Vendor, Jalankan Migration, & Clear Cache)
    - curl -s https://domain-anda.com/api/deploy-webhook-secret-123
  only:
    - main
```

---

## 5\. Konfigurasi Akhir di cPanel & Cron Jobs

Setelah pipeline GitLab CI/CD berhasil berjalan (`passed`):

### Langkah 5.1: Buat File `.env` di cPanel

1. Masuk ke cPanel **File Manager** > buka folder `public_html`.
2. Buat file baru bernama `.env`.
3. Isikan variabel berikut:

```bash
APP_NAME=Laravel
APP_ENV=production
APP_KEY= # Generate di lokal via `php artisan key:generate`
APP_DEBUG=false
APP_URL=https://nama-domain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=user_laraveldb
DB_USERNAME=user_dbuser
DB_PASSWORD=password_db_anda
```

### Langkah 5.2: Storage Link & Database Migration

- **Akses Terminal cPanel:**

```bash
cd public_html
```

```bash
php artisan storage:link
```

Atau jalankan perintah berikut di **Terminal cPanel**:

```bash
ln -s /home/USERNAME_CPANEL/storage/app/public /home/USERNAME_CPANEL/public_html/storage
```

```bash
php artisan migrate --force
```

### Langkah 5.3: Pengaturan Cron Jobs di cPanel

Buka menu Cron Jobs di cPanel, setel interval _Once Per Minute ( \* \* \* *)*_, dan masukkan command:

```bash
cd /home/USERNAME_CPANEL/public_html && /usr/local/bin/php artisan schedule:run >> /dev/null 2>&1
```
