# Panduan Deployment Multi-Application Laravel di _FTP-cPanel_

## 1\. Mirroring Repository GitHub ke GitLab Private

### Langkah 1.1: Buat Project Private Baru di GitLab

1. Login ke [GitLab](https://gitlab.com).
2. Klik **New project** > **Create blank project**.
3. Isi **Project name** (contoh: `multi-laravel-apps`).
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

1. Buka repo Monorepo Anda di [GitHub](https://github.com).
2. Masuk ke **Settings** > **Secrets and variables** > **Actions**.
3. Klik **New repository secret**:

- **Name:** `GITLAB_MIRROR_TOKEN`
- **Secret:** _Paste token dari Langkah 1.2_

4. Klik **Add secret**.

### Langkah 1.4: Workflow Mirroring di GitHub

Buat file `.github/workflows/mirror-to-gitlab.yml` di root repository:

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

## 2\. Persiapan Struktur Folder cPanel, Routing `.htaccess`, & Database

Struktur folder monorepo:

- `apps/web/` (Main App -> `public_html`)

- `apps/admin/` (Sub-app Admin -> `public_html/admin`)

### Langkah 2.1: Routing Direct `public/` via `.htaccess` (Tanpa Redirect Domain)

#### A. File `.htaccess` di Root Main App (`apps/web/.htaccess`)

```bash
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Arahkan semua request ke folder public/ milik Main App
    RewriteCond %{REQUEST_URI} !^/public/
    RewriteRule ^$ public/ [L]
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

#### B. File `.htaccess` di Root Sub-App (`apps/admin/.htaccess`)

```bash
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Arahkan semua request ke folder public/ milik Sub-App Admin
    RewriteCond %{REQUEST_URI} !^/public/
    RewriteRule ^$ public/ [L]
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

### Langkah 2.2: Buat Akun FTP di cPanel

1. Login ke **cPanel** > **FTP Accounts**.
2. Buat akun FTP untuk masing-masing aplikasi:

- **Main App (`web`):**
- **Log In:** `deployer-web`
- **Directory:** `public_html`

- **Sub-App (`admin`):**
- **Log In:** `deployer-admin`
- **Directory:** `public_html/admin`

### Langkah 2.3: Setup Database MySQL di cPanel

Buat database via **MySQL Database Wizard** di cPanel:

- Database Main App: `user_webdb`
- Database Admin App: `user_admindb`

---

## 3\. Pendaftaran Route Webhook di Masing-Masing Aplikasi

Tambahkan route ini pada file `routes/api.php` di **KEDUA** aplikasi (`apps/web/routes/api.php` dan `apps/admin/routes/api.php`). Route ini bertugas mengekstrak `vendor.zip`, menjalankan `php artisan migrate --force`, serta membersihkan cache secara otomatis.

```bash
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\File;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
*/

Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});

// Endpoint Webhook Deployment Otomatis
Route::get('/deploy-webhook-secret-123', function (Request $request) {
    $zipPath = base_path('vendor.zip');
    $messages = [];

    // 1. Ekstrak vendor.zip jika ada update composer
    if (file_exists($zipPath)) {
        File::deleteDirectory(base_path('vendor'));

        $zip = new ZipArchive;
        if ($zip->open($zipPath) === TRUE) {
            $zip->extractTo(base_path());
            $zip->close();
            @unlink($zipPath);
            $messages[] = 'Vendor updated and extracted successfully.';
        } else {
            $messages[] = 'Failed to extract vendor.zip.';
        }
    } else {
        $messages[] = 'No composer changes detected (vendor.zip skipped).';
    }

    // 2. Jalankan Database Migration Otomatis
    try {
        Artisan::call('migrate', ['--force' => true]);
        $messages[] = 'Database migration executed successfully.';
    } catch (\Exception $e) {
        $messages[] = 'Database migration failed: ' . $e->getMessage();
    }

    // 3. Clear & Recache Laravel
    Artisan::call('config:clear');
    Artisan::call('route:clear');
    Artisan::call('view:clear');

    $messages[] = 'Laravel caches cleared successfully.';

    return response()->json(['status' => 'success', 'details' => $messages]);
});
```

---

## 4\. Konfigurasi GitLab CI/CD Pipeline Multi-App

### Langkah 4.1: Tambahkan Variables di GitLab

Di **GitLab** > **Settings** > **CI/CD** > **Variables**, tambahkan:

| Key              | Value                            | Flags  |
| ---------------- | -------------------------------- | ------ |
| `FTP_SERVER`     | `ftp.nama-domain.com`            | Masked |
| `FTP_USER_WEB`   | `deployer-web@nama-domain.com`   | Masked |
| `FTP_USER_ADMIN` | `deployer-admin@nama-domain.com` | Masked |
| `FTP_PASSWORD`   | _Password FTP dari Langkah 2.2_  | Masked |

### Langkah 4.2: Buat File `.gitlab-ci.yml`

Buat file `.gitlab-ci.yml` di root monorepo:

```bash
image: php:8.3-cli

stages:
  - build
  - deploy

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - apps/web/vendor/
    - apps/web/node_modules/
    - apps/admin/vendor/
    - apps/admin/node_modules/

.build_template: &build_definition
  stage: build
  before_script:
    - apt-get update -yqq
    - apt-get install -yqq git unzip libpng-dev libonig-dev libxml2-dev libzip-dev zip curl
    - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
    - apt-get install -yqq nodejs
    - docker-php-ext-install pdo_mysql mbstring gd zip
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# --- BUILD JOBS ---
build_web:
  <<: *build_definition
  script:
    - cd apps/web
    - npm ci || npm install
    - npm run build
    - |
      if git diff --name-only HEAD~1 HEAD | grep -E "apps/web/composer\.(json|lock)"; then
        echo "Composer file changed in web. Installing vendor and zipping..."
        composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts --optimize-autoloader
        zip -r vendor.zip vendor/
      else
        echo "No changes in apps/web composer files. Skipping vendor.zip creation."
      fi
  artifacts:
    paths:
      - apps/web/vendor.zip
      - apps/web/public/build/
    expire_in: 1 hour
  only:
    - main

build_admin:
  <<: *build_definition
  script:
    - cd apps/admin
    - npm ci || npm install
    - npm run build
    - |
      if git diff --name-only HEAD~1 HEAD | grep -E "apps/admin/composer\.(json|lock)"; then
        echo "Composer file changed in admin. Installing vendor and zipping..."
        composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts --optimize-autoloader
        zip -r vendor.zip vendor/
      else
        echo "No changes in apps/admin composer files. Skipping vendor.zip creation."
      fi
  artifacts:
    paths:
      - apps/admin/vendor.zip
      - apps/admin/public/build/
    expire_in: 1 hour
  only:
    - main

# --- DEPLOY JOBS ---
deploy_web:
  stage: deploy
  dependencies:
    - build_web
  before_script:
    - apt-get update -yqq && apt-get install -yqq lftp curl
  script:
    - |
      lftp -c "
      set ftp:ssl-allow true;
      set ftp:ssl-force false;
      set ssl:verify-certificate no;
      open -u $FTP_USER_WEB,$FTP_PASSWORD $FTP_SERVER;
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
             ./apps/web/ /;
      quit
      "
    # Trigger Webhook API Main App
    - curl -s https://nama-domain.com/api/deploy-webhook-secret-123
  only:
    - main

deploy_admin:
  stage: deploy
  dependencies:
    - build_admin
  before_script:
    - apt-get update -yqq && apt-get install -yqq lftp curl
  script:
    - |
      lftp -c "
      set ftp:ssl-allow true;
      set ftp:ssl-force false;
      set ssl:verify-certificate no;
      open -u $FTP_USER_ADMIN,$FTP_PASSWORD $FTP_SERVER;
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
             ./apps/admin/ /;
      quit
      "
    # Trigger Webhook API Sub-App Admin
    - curl -s https://nama-domain.com/admin/api/deploy-webhook-secret-123
  only:
    - main
```

---

## 5\. Konfigurasi Akhir di cPanel & Multi Cron Jobs

### Langkah 5.1: Buat File `.env` Masing-Masing Aplikasi

1. File `.env` di `/public_html/.env` (Main App):

```bash
APP_NAME="Web Main"
APP_URL=https://nama-domain.com
DB_DATABASE=user_webdb
```

2. File `.env` di `/public_html/admin/.env` (Sub-App Admin):

```bash
APP_NAME="Web Admin"
APP_URL=https://nama-domain.com/admin
DB_DATABASE=user_admindb
```

### Langkah 5.2: Storage Link

Jalankan symlink di Terminal cPanel (cukup sekali di awal):

<!-- Main App -->

```bash
cd /home/USERNAME_CPANEL/public_html && php artisan storage:link
```

<!-- Sub-App Admin -->

```bash
cd /home/USERNAME_CPANEL/public_html/admin && php artisan storage:link
```

_(Catatan: Migrasi database tidak perlu dijalankan manual lagi dari terminal karena sudah ditangani secara otomatis oleh Webhook API tiap kali push)._

### Langkah 5.3: Pengaturan Multi-Cron Jobs di cPanel

Daftarkan dua baris perintah di **cPanel** > **Cron Jobs**:

1. **Cron Job Main App:**

```bash
cd /home/USERNAME_CPANEL/public_html && /usr/local/bin/php artisan schedule:run >> /dev/null 2>&1
```

2. **Cron Job Sub-App Admin:**

```bash
cd /home/USERNAME_CPANEL/public_html/admin && /usr/local/bin/php artisan schedule:run >> /dev/null 2>&1
```
