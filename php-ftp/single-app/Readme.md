# Panduan Deployment Single-Application PHP di _VPS CPanel_

Dengan menggunakan konfigurasi ini, repositori utama Anda berada di GitHub, namun Anda memanfaatkan infrastruktur GitLab CI/CD untuk proses _deploy_ otomatis ke cPanel.

```
[ Lokal / PC ] --( git push )--> [ GitHub ] --( GitHub Actions )--> [ GitLab ] --( GitLab CI/CD )--> [ VPS cPanel ]

```

Berikut adalah langkah penyesuaian yang perlu ditambahkan pada file project Anda sebelum masuk ke pengaturan GitLab CI/CD.

## Setup GitHub Actions (.github/workflows/mirror-to-gitlab.yml)

### 1. Membuat Token di GitLab (Personal Access Token)

Agar GitHub diizinkan untuk melakukan _push_ kode ke repositori GitLab Anda, GitHub memerlukan token akses.

1. Masuk ke **GitLab** Anda.
2. Klik foto profil di pojok kiri bawah/atas, lalu pilih **Preferences** > **Access Tokens** (atau **Project Access Tokens** di dalam repositori terkait).
3. Klik **Add new token**.
4. Beri nama token (contoh: `github-mirror-token`).
5. Pada bagian **Select Scopes**, centang **`write_repository`** (dan **`read_repository`** jika diperlukan).
6. Klik **Create personal access token**, lalu **copy token** yang muncul. _Jangan sampai hilang karena token ini hanya muncul sekali._

### 2. Memasukkan Token ke Repository Secrets GitHub

1. Buka repositori **GitHub** Anda.
2. Pergi ke tab **Settings** > **Secrets and variables** > **Actions**.
3. Klik tombol **New repository secret**.
4. Isi kotak dengan ketentuan berikut:

- **Name:** `GITLAB_TOKEN`
- **Secret:** _Paste token yang tadi Anda copy dari GitLab._

5. Klik **Add secret**.

### 3. Membuat File Alur Kerja di GitHub

Di dalam _root project_ PHP Anda di komputer, buat struktur folder `.github/workflows/` dan buat file bernama `mirror-to-gitlab.yml`. Masukkan kode yang Anda gunakan:

```bash
name: Mirror to GitLab

on: [push] # Pemicu: Setiap kali ada git push ke GitHub

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Wajib diambil semua history commit-nya agar tidak error saat mirror

      - name: Push to GitLab
        uses: yesolutions/mirror-action@master
        with:
          REMOTE: "https://gitlab.com/username/nama_aplikasi.git"
          GIT_USERNAME: "oauth2" # Username khusus untuk autentikasi token GitLab
          GIT_PASSWORD: ${{ secrets.GITLAB_TOKEN }}
```

## Penyesuaian Langkah Setelah Mirroring

### 1. Ambil Detail Akun FTP

1. Masuk ke **Client Area**.
2. Pilih akun hosting Anda, lalu masuk ke tab **FTP Details** (atau cari di bagian samping dashboard).
3. Catat data berikut:

- **FTP Hostname** (biasanya formatnya seperti `ftpupload.net`)
- **FTP Username** (contoh: `if0_3xxxxxx`)
- **FTP Password** (klik _Show Password_)

> ⚠️ **Catatan Penting:** Semua file website yang ingin bisa diakses oleh publik **wajib** dimasukkan ke dalam folder `htdocs`. Jadi, path tujuan kita nanti adalah folder `htdocs` tersebut.

---

### 2. Mengubah Variabel di GitLab CI/CD

Hapus variabel SSH yang lama di GitLab (**Settings > CI/CD > Variables**), lalu ganti dengan variabel FTP baru berikut (matikan _"Protect variable"_):

| Key          | Value / Isian                                 |
| ------------ | --------------------------------------------- |
| **FTP_HOST** | _Hostname dari FTP Anda (cth: ftpupload.net)_ |
| **FTP_USER** | _Username FTP Anda (cth: if0_3xxxxxx)_        |
| **FTP_PASS** | _Password FTP Anda_                           |

---

### 3. Menambahkan File .gitlab-ci.yml

Buat file bernama .gitlab-ci.yml di root project PHP Anda, lalu masukkan kode berikut:

```bash
stages:
  - deploy

Deploy:
  stage: deploy
  image: ubuntu:latest
  timeout: 15 minutes

  before_script:
    - apt-get update -y && apt-get install -y lftp

  script:
    - echo "Memulai transfer file via FTP..."
    - lftp -e "set ftp:ssl-allow no; open ftp://$FTP_USER:$FTP_PASS@$FTP_HOST; mirror -R --delete --exclude .git/ --exclude .github/ --exclude .gitlab-ci.yml --exclude README.md ./ /htdocs; quit"
```

> Skrip ini akan otomatis mendeteksi file yang berubah dan mengunggahnya ke folder `htdocs` via FTP setiap kali GitHub selesai melakukan _mirroring_ ke GitLab.

---
