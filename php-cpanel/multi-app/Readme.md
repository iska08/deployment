# Panduan Deployment Multi-Application PHP dengan Repositori Berbeda (GitHub ➔ GitLab ➔ Hosting via FTP)

Panduan ini ditujukan untuk arsitektur multi-aplikasi, di mana **Aplikasi A**, **Aplikasi B**, dst., berada di **repositori GitHub yang terpisah**. Setiap repositori akan melakukan _mirroring_ ke repositori GitLab masing-masing, kemudian dideploy secara otomatis ke direktori target yang berbeda pada satu akun hosting via FTP.

```
[ Repo GitHub App 1 ] --( push )--> [ GitHub Actions ] ➔ [ Repo GitLab App 1 ] --( CI/CD )--┐
                                                                                            ├─➔ [ Satu Akun Hosting ]
[ Repo GitHub App 2 ] --( push )--> [ GitHub Actions ] ➔ [ Repo GitLab App 2 ] --( CI/CD )--┘     (Folder Berbeda)

```

---

## BAGIAN 1: Setup di Sisi GitHub (Lakukan di Setiap Repositori Aplikasi)

Langkah-langkah di bawah ini harus diterapkan pada **setiap repositori GitHub aplikasi Anda** (misalnya pada repositori Aplikasi Utama dan repositori Aplikasi Subdomain).

### 1. Membuat Token Akses di GitLab

Setiap aplikasi membutuhkan token agar GitHub dapat melakukan _push_ ke repositori GitLab yang sesuai.

1. Masuk ke **GitLab**.
2. Anda bisa menggunakan satu _Personal Access Token_ untuk semua repo, atau membuat _Project Access Token_ di masing-masing projek GitLab.
3. Buka **Preferences** > **Access Tokens**.
4. Klik **Add new token**, beri nama (contoh: `github-multi-mirror-token`).
5. Pada bagian **Select Scopes**, centang **`write_repository`** dan **`read_repository`**.
6. Klik **Create personal access token**, lalu salin token tersebut.

### 2. Menyimpan Token di Secrets GitHub

1. Buka repositori **GitHub** aplikasi Anda (misal: `aplikasi-utama`).
2. Pergi ke **Settings** > **Secrets and variables** > **Actions** > **New repository secret**.
3. Isi **Name:** `GITLAB_TOKEN`.
4. Isi **Secret:** _Paste token yang Anda salin dari GitLab_.
5. Klik **Add secret**.
6. _(Ulangi langkah ini di repositori GitHub aplikasi kedua Anda dengan token yang sama/sesuai)._

### 3. Membuat File Alur Kerja Mirroring

Di setiap komputer lokal proyek aplikasi Anda, buat folder `.github/workflows/` dan buat file `mirror-to-gitlab.yml`.

#### 📝 Konfigurasi untuk Aplikasi 1 (Misal: Aplikasi Utama)

Sesuaikan parameter `REMOTE` dengan URL repositori GitLab untuk Aplikasi 1:

```yaml
name: Mirror to GitLab (App 1)

on: [push]

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Push to GitLab
        uses: yesolutions/mirror-action@master
        with:
          REMOTE: "https://gitlab.com/username/app1-utama.git"
          GIT_USERNAME: "oauth2"
          GIT_PASSWORD: ${{ secrets.GITLAB_TOKEN }}
```

#### 📝 Konfigurasi untuk Aplikasi 2 (Misal: Aplikasi Subdomain / Portfolio)

Sesuaikan parameter `REMOTE` dengan URL repositori GitLab untuk Aplikasi 2:

```yaml
name: Mirror to GitLab (App 2)

on: [push]

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Push to GitLab
        uses: yesolutions/mirror-action@master
        with:
          REMOTE: "https://gitlab.com/username/app2-subdomain.git"
          GIT_USERNAME: "oauth2"
          GIT_PASSWORD: ${{ secrets.GITLAB_TOKEN }}
```

---

## BAGIAN 2: Penyesuaian Struktur Folder & Variabel FTP

Sebelum mengatur GitLab CI/CD, Anda harus memetakan folder di server FTP hosting Anda.

### 1. Struktur Direktori Target di Hosting

Umumnya, penyedia hosting gratis atau cPanel mengatur multi-domain/subdomain dengan struktur seperti ini:

- **Aplikasi Utama (Domain Utama):** Mengarah ke folder `/htdocs` atau `/public_html`.
- **Aplikasi Tambahan (Subdomain/Addon):** Mengarah ke folder `/subdomain.domain.com/htdocs` atau `/htdocs/subfolder`.

_Pastikan Anda telah memeriksa File Manager di Client Area untuk memastikan lokasi folder tujuan eksak dari masing-masing aplikasi._

### 2. Mengatur Variabel di GitLab CI/CD (Terpisah per Repositori GitLab)

Buka masing-masing repositori Anda di **GitLab**, lalu akses menu **Settings > CI/CD > Variables**. Tambahkan variabel berikut:

#### Proyek GitLab Aplikasi 1 (Utama):

| Key            | Value / Isian   | Keterangan                 |
| -------------- | --------------- | -------------------------- |
| **FTP_HOST**   | `ftpupload.net` | Hostname FTP               |
| **FTP_USER**   | `if0_3xxxxxx`   | Username FTP               |
| **FTP_PASS**   | `password_anda` | Password FTP               |
| **TARGET_DIR** | `/htdocs`       | Root folder Aplikasi Utama |

#### Proyek GitLab Aplikasi 2 (Subdomain):

| Key            | Value / Isian                  | Keterangan              |
| -------------- | ------------------------------ | ----------------------- |
| **FTP_HOST**   | `ftpupload.net`                | Hostname FTP (Sama)     |
| **FTP_USER**   | `if0_3xxxxxx`                  | Username FTP (Sama)     |
| **FTP_PASS**   | `password_anda`                | Password FTP (Sama)     |
| **TARGET_DIR** | `/subdomain.domain.com/htdocs` | Folder khusus Subdomain |

---

## BAGIAN 3: Menambahkan File `.gitlab-ci.yml` yang Dinamis

Karena kita telah memisahkan direktori tujuan ke dalam variabel `TARGET_DIR`, kita bisa menggunakan satu templat berkas `.gitlab-ci.yml` yang fleksibel untuk kedua repositori tersebut.

Buat file `.gitlab-ci.yml` di _root_ direktori masing-masing aplikasi Anda:

```yaml
stages:
  - deploy

Deploy_Application:
  stage: deploy
  image: ubuntu:latest
  timeout: 15 minutes

  before_script:
    # Mengunduh dan memasang LFTP utilitas transfer file
    - apt-get update -y && apt-get install -y lftp

  script:
    - echo "Memulai transfer file via FTP ke direktori $TARGET_DIR..."
    # Eksekusi satu baris lftp untuk menghindari kesalahan pembacaan karakter backslash (\) oleh GitLab runner
    - lftp -e "set ftp:ssl-allow no; open ftp://$FTP_USER:$FTP_PASS@$FTP_HOST; mirror -R --delete --exclude .git/ --exclude .github/ --exclude .gitlab-ci.yml --exclude README.md ./ $TARGET_DIR; quit"
    - echo "Deployment ke direktori $TARGET_DIR berhasil diselesaikan!"
```

---

## 💡 Cara Kerja & Manajemen Pasca Setup

1. **Isolasi Penuh:** Perubahan kode pada Aplikasi 1 tidak akan mengganggu file Aplikasi 2 di server, karena perintah `mirror -R --delete` dibatasi secara ketat hanya bekerja di dalam folder yang didefinisikan oleh variabel `$TARGET_DIR`.
2. **Pengembangan Mandiri:** Anda dapat melakukan `git push` secara independen di repositori GitHub mana saja. Hanya aplikasi yang menerima commit baru yang akan menjalankan siklus deployment.
3. **Pencegahan File Terhapus:** Flag `--delete` pada perintah `lftp` berfungsi untuk membersihkan file sampah di server _hanya pada folder tujuan aplikasi tersebut_, sehingga file penting milik aplikasi lain yang berada di folder berbeda tetap aman.
