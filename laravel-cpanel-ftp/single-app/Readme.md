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

    # 1. Redirect domain nama-domain.id ke nama-domain.com (Permanen 301)
    RewriteCond %{HTTP_HOST} ^(www\.)?nama-domain\.id$ [NC]
    RewriteRule ^(.*)$ https://nama-domain.com/$1 [R=301,L]

    # 2. Arahkan semua request ke folder public/ Laravel
    RewriteRule ^$ public/ [L]
    RewriteRule (.*) public/$1 [L]
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

## 3\. Konfigurasi GitLab CI/CD Pipeline (PHP 8.3 + Node.js Build)

### Langkah 3.1: Tambahkan Variables di GitLab

Masuk ke **GitLab** > **Settings** > **CI/CD** > **Variables** > **Add variable**:

| Key            | Value                                  | Flags  |
| -------------- | -------------------------------------- | ------ |
| `FTP_SERVER`   | `ftp.nama-domain.com` (atau IP Server) | Masked |
| `FTP_USERNAME` | `deployer@nama-domain.com`             | Masked |
| `FTP_PASSWORD` | _Password FTP dari Langkah 2.2_        | Masked |

### Langkah 3.2: Buat File `.gitlab-ci.yml`

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

    # 2. Install Node.js & NPM (Metode NodeSource)
    - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
    - apt-get install -yqq nodejs

    # 3. Install Extension PHP
    - docker-php-ext-install pdo_mysql mbstring gd zip

    # 4. Install Composer Dependencies
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts --optimize-autoloader

    # 5. Install NPM Dependencies & Build Assets
    - npm ci || npm install
    - npm run build

  artifacts:
    paths:
      - vendor/
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
    - apt-get install -yqq lftp
  script:
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
             --exclude-glob storage/logs/ \
             --exclude-glob storage/framework/cache/ \
             --exclude-glob storage/framework/sessions/ \
             --exclude-glob storage/framework/views/ \
             ./ /;
      quit
      "
  only:
    - main
```

---

## 4\. Konfigurasi Akhir di cPanel & Cron Jobs

Setelah pipeline GitLab CI/CD berhasil berjalan (`passed`):

### Langkah 4.1: Buat File `.env` di cPanel

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

### Langkah 4.2: Storage Link & Database Migration

- **Akses Terminal cPanel:**

```bash
cd public_html
```

```bash
php artisan storage:link
```

Atau jalankan perintah berikut di _Terminal_ cPanel:

```bash
ln -s /home/USERNAME_CPANEL/storage/app/public /home/USERNAME_CPANEL/public_html/storage
```

```bash
php artisan migrate --force
```

- **Jika Tidak Ada Terminal cPanel:**
  Jalankan migration melalui **phpMyAdmin** (Import file SQL) dan jalankan route symlink sementara di `routes/web.php`:

```bash
Route::get('/init-app', function () {
    Artisan::call('storage:link');
    Artisan::call('migrate', ['--force' => true]);
    return 'Storage link & Database migration successfully executed.';
});
```

### Langkah 4.3: Pengaturan Cron Jobs di cPanel

Untuk menjalankan `php artisan schedule:run` secara otomatis setiap menit:

1. Di cPanel, buka menu **Cron Jobs**.
2. Pada bagian **Common Settings**, pilih **Once Per Minute (\* \* \* \* \*)**.
3. Di kolom **Command**, masukkan perintah berikut (sesuaikan path PHP cPanel & path folder domain Anda):

```bash
/usr/local/bin/php /home/USERNAME_CPANEL/public_html/artisan schedule:run >> /dev/null 2>&1
```

> **Tip:** Cek lokasi pasti PHP binary di cPanel jika beda versi (misal: `/usr/bin/php` atau `/opt/alt/php83/usr/bin/php`).
