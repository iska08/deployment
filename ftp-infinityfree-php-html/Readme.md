# Deploy Otomatis Website PHP & HTML ke InfinityFree

Panduan ini menyiapkan alur berikut: perubahan di branch `main` pada GitHub otomatis disalin ke GitLab, lalu pipeline GitLab CI/CD mengunggah file website melalui FTP ke InfinityFree.

```bash
Push ke GitHub (main)
        |
        v
GitHub Actions -- mirror --> GitLab
                              |
                              v
                     GitLab CI/CD -- FTP --> InfinityFree
```

Panduan ini untuk website PHP/HTML biasa, bukan Laravel. GitLab menjadi sumber pipeline dan kredensial FTP; file website tidak perlu diedit langsung dari File Manager setiap kali ada perubahan.

## 1. Siapkan Repository

1. Pastikan kode website PHP, HTML, CSS, dan JavaScript berada di root repository GitHub.
2. Gunakan branch utama `main`. Jika repository memakai nama branch lain, sesuaikan nama branch di kedua file workflow/pipeline pada panduan ini.
3. Pastikan halaman awal, misalnya `index.php` atau `index.html`, berada di root project.
4. Jangan masukkan password FTP, token, atau file konfigurasi rahasia ke repository.

Contoh struktur:

```bash
website-saya/
|-- .github/
|   `-- workflows/
|       `-- mirror-to-gitlab.yml
|-- .gitlab-ci.yml
|-- .gitignore
|-- index.php
|-- tentang.html
`-- style.css
```

## 2. Hubungkan GitHub ke GitLab

### 2.1 Buat repository GitLab

1. Buat project kosong di GitLab, misalnya `website-saya`.
2. Jangan inisialisasi project dengan README jika GitHub sudah memiliki commit.
3. Catat alamat HTTPS repository GitLab, misalnya `gitlab.com/USERNAME/website-saya.git`.

### 2.2 Buat token GitLab untuk mirror

1. Buat Personal Access Token di GitLab dengan izin yang diperlukan untuk push ke repository, umumnya scope `write_repository`.
2. Simpan token saat dibuat; jangan tempelkan token langsung ke file workflow.
3. Di GitHub repository, buka **Settings > Secrets and variables > Actions** lalu tambahkan:

| Nama secret | Nilai |
| --- | --- |
| `GITLAB_TOKEN` | Personal Access Token GitLab |

### 2.3 Tambahkan workflow mirror di GitHub

Buat file `.github/workflows/mirror-to-gitlab.yml`:

```bash
name: Mirror to GitLab

on: [push]

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Push to GitLab
        uses: yesolutions/mirror-action@master
        with:
          REMOTE: "https://gitlab.com/USERNAME/website-saya.git"
          GIT_USERNAME: "oauth2"
          GIT_PASSWORD: ${{ secrets.GITLAB_TOKEN }}
```

Ganti `USERNAME/website-saya` dengan path project GitLab tujuan. Simpan file sebagai `.github/workflows/mirror-to-gitlab.yml` di repository website. Workflow ini berjalan setiap kali ada push ke GitHub dan menggunakan action yang mencerminkan repository ke URL GitLab yang ditentukan. Periksa tab **Actions** di GitHub untuk memastikan proses mirror berhasil.

> Gunakan token hanya sebagai GitHub Actions secret. Jangan menambahkan URL yang berisi token ke repository atau mencetaknya di log.

## 3. Siapkan FTP InfinityFree

1. Buat atau pilih akun hosting dan domain dari InfinityFree Client Area, lalu tunggu hingga akun aktif.
2. Buka detail FTP akun di Client Area dan catat host FTP, username, password, serta port yang diberikan. Nilainya khusus untuk akun Anda.
3. Periksa direktori web publik setelah login FTP. Umumnya bernama `htdocs`. Jika login FTP langsung membuka direktori web publik, catat bahwa tujuan upload-nya adalah direktori saat ini (`./`).
4. Sebelum menyiapkan CI/CD, sambungkan sekali dengan aplikasi FTP seperti FileZilla dan pastikan Anda dapat melihat direktori web publik.
5. Pastikan `index.php` atau `index.html` ada di direktori tersebut agar domain menampilkan halaman utama.

## 4. Simpan Kredensial FTP di GitLab

Di GitLab project, buka **Settings > CI/CD > Variables** dan tambahkan variables berikut:

| Key | Value | Pengaturan |
| --- | --- | --- |
| `FTP_SERVER` | Host FTP dari Client Area | Masked |
| `FTP_USERNAME` | Username FTP dari Client Area | Masked |
| `FTP_PASSWORD` | Password FTP dari Client Area | Masked |

Aktifkan **Masked** untuk tiga variable kredensial. Aktifkan **Protected** hanya jika branch `main` juga sudah diatur sebagai protected branch; jika tidak, job tidak akan menerima variable tersebut. Jangan menaruh kredensial di `.gitlab-ci.yml`.

## 5. Tambahkan Pipeline Deploy GitLab

Buat file `.gitlab-ci.yml` di root repository:

```bash
image: alpine:3.22

stages:
  - deploy

deploy_ftp:
  stage: deploy
  only:
    - main
  before_script:
    - apk add --no-cache lftp
  script:
    - test -n "$FTP_SERVER" || (echo "FTP_SERVER belum diatur"; exit 1)
    - test -n "$FTP_USERNAME" || (echo "FTP_USERNAME belum diatur"; exit 1)
    - test -n "$FTP_PASSWORD" || (echo "FTP_PASSWORD belum diatur"; exit 1)
    - |
      lftp -c "
      set net:max-retries 5;
      set net:timeout 30;
      set net:reconnect-interval-base 5;
      set net:reconnect-interval-max 30;
      open -u \"$FTP_USERNAME\",\"$FTP_PASSWORD\" \"$FTP_SERVER\";
      mirror --reverse \
             --verbose \
             --ignore-time \
             --no-perms \
             --exclude-glob .git/ \
             --exclude-glob .github/ \
             --exclude-glob .gitlab-ci.yml \
             --exclude-glob .gitignore \
             --exclude-glob README.md \
             ./ htdocs/;
      bye
      "
```

Pada contoh ini, pipeline mengunggah isi repository ke `htdocs/`. Jika login FTP Anda langsung membuka direktori web publik, ganti `htdocs/` pada perintah `mirror` menjadi `./`. Web server akan menampilkan `index.php` atau `index.html` secara otomatis selama file tersebut berada langsung di direktori web publik; nama tujuan upload tetap perlu sesuai dengan struktur FTP akun. File konfigurasi pipeline dan file development yang dikecualikan tidak diunggah. `--ignore-time` membuat sinkronisasi tidak hanya bergantung pada timestamp file. `--no-perms` mencegah `lftp` menjalankan `chmod`, yang dapat ditolak oleh FTP hosting bersama.

Perintah di atas tidak menghapus file yang hanya ada di hosting. Ini disengaja agar file hosting lain tidak terhapus otomatis; jika menghapus file dari repository, hapus file terkait di hosting secara manual setelah memastikan file tersebut memang tidak diperlukan.

## 6. Uji Deploy Otomatis

1. Commit file `.gitlab-ci.yml` dan perubahan website, lalu push ke branch `main` di GitHub.
2. Buka **Actions** di GitHub dan pastikan job mirror berhasil.
3. Setelah commit masuk ke GitLab, buka **Build > Pipelines** dan pastikan job `deploy_ftp` berjalan sampai sukses.
4. Buka domain InfinityFree dan periksa perubahan. Jika konten lama masih terlihat, coba refresh tanpa cache atau periksa apakah pipeline mengunggah ke direktori yang benar.
5. Untuk perubahan berikutnya, cukup push commit ke `main` di GitHub. GitHub Actions akan mirror ke GitLab dan GitLab CI/CD menjalankan deploy FTP.

## Pemecahan Masalah

- **GitHub Actions gagal:** Pastikan secret `GITLAB_TOKEN` valid, memiliki izin push, dan URL project GitLab pada workflow benar.
- **Pipeline tidak berjalan di GitLab:** Pastikan mirror berhasil masuk ke branch `main`, CI/CD tersedia untuk project, dan file `.gitlab-ci.yml` ada di root repository.
- **Variable FTP kosong di job:** Periksa nama variables. Jika ditandai Protected, pastikan branch `main` juga protected.
- **FTP login gagal:** Salin ulang host, username, dan password dari Client Area. Gunakan detail dan mode koneksi yang didukung akun.
- **Log menampilkan `550 Could not change perms` atau `chmod: Access failed`:** Pastikan opsi `--no-perms` ada pada perintah `mirror`; InfinityFree/shared hosting dapat menolak perubahan permission lewat FTP.
- **Log menampilkan `Fatal error: max-retries exceeded` saat transfer file:** Pipeline akan mencoba ulang koneksi beberapa kali. Jika masih terjadi, periksa log FTP untuk respons server, pastikan host dan direktori tujuan benar, lalu jalankan ulang pipeline; kegagalan berulang dapat mengindikasikan koneksi FTP terputus atau server menolak transfer file tersebut.
- **Pipeline sukses tetapi website tidak berubah:** Periksa apakah tujuan upload `htdocs/` atau `./` sesuai dengan struktur FTP akun, lokasi file `index.php`/`index.html`, dan log job untuk melihat hasil `lftp`.
- **PHP source code tampil di browser:** Jangan biarkan halaman tersebut tersedia untuk publik. Pastikan file berekstensi `.php` dan hosting memproses PHP sebelum mengunggah kembali.

## Catatan

- `.html` adalah halaman statis dan tidak menjalankan kode PHP. Simpan halaman yang berisi kode PHP dengan ekstensi `.php`.
- Jangan commit rahasia, backup database, atau file lokal yang tidak seharusnya dipublikasikan.
- Contoh pipeline ini mengunggah source website langsung; tidak menjalankan build, instalasi Composer, atau migrasi database.
- Periksa dokumentasi dan batasan terbaru InfinityFree untuk detail FTP, hosting, dan penggunaan layanan.
