# Panduan Deployment PHP Native: GitHub → GitLab Mirroring → VPS cPanel

Panduan ini menjelaskan langkah-langkah untuk melakukan deploy aplikasi PHP Native (Tanpa Framework/Docker) yang dikembangkan di **GitHub**, di-_mirror_ secara otomatis ke **GitLab**, dan dideploy ke **VPS cPanel** menggunakan **GitLab CI/CD**.

---

## 1. Alur Kerja (Workflow)

```
[ Local Laptop ] --- (git push) ---> [ GitHub Repo ] ---> (GitHub Actions Mirror) ---> [ GitLab Repo ] ---> (rsync/SSH) ---> [ VPS cPanel ]

```

1. Anda melakukan `git push` ke **GitHub**.
2. **GitHub Actions** mendeteksi _push_ tersebut dan menduplikasi (_mirroring_) seluruh _code_ & _commit history_ ke **GitLab**.
3. **GitLab CI/CD** langsung terpicu (_triggered_) untuk mengirimkan file ke **cPanel** menggunakan `rsync` via **SSH**.

---

## 2. Langkah 1: Setup Mirroring di GitHub Actions

Agar GitHub bisa melakukan _push_ otomatis ke GitLab tanpa meminta password manual, kita perlu membuat token akses di GitLab terlebih dahulu.

### A. Membuat Personal Access Token (PAT) di GitLab

1. Masuk ke akun **GitLab** Anda.
2. Klik foto profil Anda di pojok kiri bawah/atas, pilih **Edit Profile** > **Access Tokens**.
3. Klik **Add new token**.
4. Isi nama token (misal: `github_mirror_token`).
5. Pada bagian **Select scopes**, centang/checklist **`write_repository`** (dan `read_repository` jika diperlukan).
6. Klik **Create personal access token**, lalu **Copy** token yang muncul (Token ini hanya muncul sekali).

### B. Memasukkan Token ke Secrets GitHub

1. Buka repositori **GitHub** Anda.
2. Pergi ke menu **Settings** > **Secrets and variables** > **Actions**.
3. Klik **New repository secret**.
4. Isikan data berikut:

- **Name:** `GITLAB_TOKEN`
- **Secret:** _Paste token yang tadi di-copy dari GitLab_

5. Klik **Add secret**.

### C. Membuat File Alur Kerja GitHub Actions

Di dalam _project_ lokal Anda, buat struktur folder `.github/workflows/` dan buat file baru bernama `mirror-to-gitlab.yml`. Masukkan kode berikut:

```yaml
name: Mirror to GitLab

on: [push] # Memicu mirror setiap kali ada git push ke GitHub

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Wajib, untuk mengambil semua history commit

      - name: Push to GitLab
        uses: yesolutions/mirror-action@master
        with:
          REMOTE: "https://gitlab.com/iska08/lkbb.git"
          GIT_USERNAME: "oauth2" # Username khusus untuk autentikasi token
          GIT_PASSWORD: ${{ secrets.GITLAB_TOKEN }}
```

---

## 3. Langkah 2: Persiapan di cPanel

1. **Akses SSH:** Masuk ke cPanel > **SSH Access**. Pastikan akses SSH aktif dan catat nomor **Port SSH** Anda (biasanya `22`, namun beberapa provider hosting mengubahnya demi keamanan).
2. **Path Direktori:** Cari tahu _Document Root_ website Anda (Umumnya di `/home/username_cpanel/public_html/`).
3. **Database:** Buat database baru di **MySQL® Database Wizard** dan catat nama DB, user, serta password-nya.

---

## 4. Langkah 3: Setup SSH Key untuk GitLab ke cPanel

Agar GitLab CI/CD bisa mengirim file hasil _mirroring_ ke cPanel tanpa mengetik password:

1. Buka Git Bash / Terminal di komputer Anda, buat SSH Key baru:

```bash
ssh-keygen -t rsa -b 4096 -f cpanel_deploy_key

```

_(Tekan Enter terus sampai selesai, kosongkan passphrase)_.

2. **Di cPanel:** Masuk ke **SSH Access** > **Manage SSH Keys** > **Import Key**. Paste isi file `cpanel_deploy_key.pub` (Public Key) ke kotak yang disediakan. Setelah sukses di-import, klik **Manage** > **Authorize** pada key tersebut. 3. **Di GitLab:** Masuk ke repositori GitLab Anda (`iska08/lkbb`), pilih **Settings** > **CI/CD** > **Variables** > **Add variable** (Matikan opsi _Protect variable_). Tambahkan variabel berikut:

| Key                 | Value / Isian                                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------------------------ |
| **SSH_PRIVATE_KEY** | Buka file `cpanel_deploy_key` (Tanpa `.pub`) di laptop, copy seluruh isinya dari baris `BEGIN` sampai `END`. |
| **SSH_USER**        | Username login cPanel Anda.                                                                                  |
| **SSH_HOST**        | IP Address VPS cPanel Anda atau nama domain utama.                                                           |
| **SSH_PORT**        | Port SSH cPanel Anda (contoh: `22` atau `2222`).                                                             |
| **TARGET_DIR**      | Path tujuan di cPanel, contoh: `/home/username_cpanel/public_html/`                                          |

---

## 5. Langkah 4: Membuat File `.gitlab-ci.yml`

Buat file bernama `.gitlab-ci.yml` di root project Anda. File ini yang nantinya akan otomatis dibaca dan dieksekusi oleh GitLab Runner begitu proses _mirroring_ dari GitHub selesai.

```yaml
stages:
  - deploy

Deploy_to_cPanel:
  stage: deploy
  image: ubuntu:latest
  timeout: 15 minutes

  before_script:
    # Instalasi SSH Client dan Rsync di Runner GitLab
    - apt-get update -y && apt-get install -y openssh-client rsync
    - eval $(ssh-agent -s)

    # Memasukkan SSH Private Key ke dalam Agent
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -

    # Mendaftarkan host agar menghindari prompt key verification
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan -p $SSH_PORT -H $SSH_HOST >> ~/.ssh/known_hosts 2>/dev/null
    - chmod 644 ~/.ssh/known_hosts

  script:
    - echo "Memulai transfer file hasil mirror ke cPanel..."

    # Proses deploy menggunakan rsync
    # --exclude digunakan agar file/folder internal tidak ikut terupload ke hosting cPanel
    rsync -avz --delete \
      --exclude '.git*' \
      --exclude '.github/' \
      --exclude '.gitlab-ci.yml' \
      --exclude 'README.md' \
      --exclude 'cpanel_deploy_key*' \
      -e "ssh -p $SSH_PORT" \
      ./ $SSH_USER@$SSH_HOST:$TARGET_DIR

    - echo "Deployment PHP Native ke cPanel selesai dengan sukses!"

```

---

## 6. Tips Tambahan untuk PHP Native

### Koneksi Database Dinamis (`config.php`)

Karena PHP Native tidak menggunakan arsitektur `.env` bawaan seperti framework, Anda bisa mengatur file konfigurasi database agar mendeteksi otomatis apakah aplikasi sedang berjalan di komputer lokal atau sudah di server cPanel:

```php
<?php
if ($_SERVER['REMOTE_ADDR'] == '127.0.0.1' || $_SERVER['SERVER_NAME'] == 'localhost') {
    // Konfigurasi Server Lokal (XAMPP / Laragon)
    define('DB_HOST', 'localhost');
    define('DB_USER', 'root');
    define('DB_PASS', '');
    define('DB_NAME', 'lkbb_lokal');
} else {
    // Konfigurasi Server Produksi (cPanel)
    define('DB_HOST', 'localhost'); // Di cPanel biasanya tetap localhost
    define('DB_USER', 'user_cpanel_anda');
    define('DB_PASS', 'PasswordDatabaseCpanelAnda');
    define('DB_NAME', 'nama_database_cpanel');
}

$conn = new mysqli(DB_HOST, DB_USER, DB_PASS, DB_NAME);
if ($conn->connect_error) {
    die("Koneksi Database Gagal: " . $conn->connect_error);
}

```

---
