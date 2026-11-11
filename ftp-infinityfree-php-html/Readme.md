# Deploy Otomatis Website PHP & HTML ke InfinityFree

Panduan ini menyiapkan alur berikut: perubahan di branch `main` pada GitHub otomatis disalin ke GitLab, lalu pipeline GitLab CI/CD mengemas seluruh file menjadi arsip ZIP, mengunggahnya melalui FTP ke InfinityFree, dan mengekstraknya secara otomatis menggunakan skrip PHP di server.

```bash
Push ke GitHub (main)
        |
        v
GitHub Actions -- mirror --> GitLab
                              |
                              v
                     GitLab CI/CD -- ZIP + FTP --> InfinityFree (Auto-Extract)
```

Panduan ini untuk website PHP/HTML biasa, bukan Laravel. Metode pengarsipan ZIP dipilih untuk menghindari pembatasan koneksi aktif (_data connection_) dan error pembatasan _bandwidth_ dari server FTP gratisan InfinityFree.

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

| Nama secret    | Nilai                        |
| -------------- | ---------------------------- |
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

Ganti `USERNAME/website-saya` dengan path project GitLab tujuan. Simpan file sebagai `.github/workflows/mirror-to-gitlab.yml` di repository website.

> Gunakan token hanya sebagai GitHub Actions secret. Jangan menambahkan URL yang berisi token ke repository atau mencetaknya di log.

## 3. Siapkan FTP InfinityFree & Skrip Ekstraktor

1. Buat atau pilih akun hosting dan domain dari InfinityFree Client Area, lalu tunggu hingga akun aktif.
2. Buka detail FTP akun di Client Area dan catat host FTP, username, password, serta port yang diberikan.
3. Sambungkan menggunakan aplikasi FTP seperti FileZilla atau File Manager, lalu pastikan Anda masuk ke direktori web publik (`htdocs`).
4. Buat file helper bernama **`unzip.php`** di dalam folder `htdocs/` Anda untuk menangani ekstraksi otomatis dengan kode berikut:

```bash
<?>
$secret = 'RAHASIA_ZIP_DEPLOY';

if (!isset($_GET['token']) || $_GET['token'] !== $secret) {
    http_response_code(403);
    exit("GAGAL: Akses ditolak, token salah.");
}

// 4. Menghapus semua file dan folder lama di htdocs KECUALI file unzip.php itu sendiri
$files = scandir('./');
foreach ($files as $file) {
    if ($file !== '.' && $file !== '..' && $file !== 'unzip.php') {
        if (is_dir($file)) {
            rmdir_recursive($file);
        } else {
            unlink($file);
        }
    }
}

echo "SUKSES: Seluruh file dan folder lama berhasil dibersihkan dari server!";

// Fungsi bantu untuk menghapus folder secara rekursif
function rmdir_recursive($dir) {
    $it = new RecursiveDirectoryIterator($dir, RecursiveDirectoryIterator::SKIP_DOTS);
    $files = new RecursiveIteratorIterator($it, RecursiveIteratorIterator::CHILD_FIRST);
    foreach($files as $file) {
        if ($file->isDir()){
            rmdir($file->getRealPath());
        } else {
            unlink($file->getRealPath());
        }
    }
    rmdir($dir);
}
?>
```

> Token RAHASIA_ZIP_DEPLOY bukanlah kode otomatis atau bawaan dari sistem GitLab maupun InfinityFree, melainkan bebas Anda buat sendiri (bebas dikarang).

## 4. Simpan Kredensial FTP di GitLab

Di GitLab project, buka **Settings > CI/CD > Variables** dan tambahkan variables berikut:

| Key            | Value                         | Pengaturan |
| -------------- | ----------------------------- | ---------- |
| `FTP_SERVER`   | Host FTP dari Client Area     | Masked     |
| `FTP_USERNAME` | Username FTP dari Client Area | Masked     |
| `FTP_PASSWORD` | Password FTP dari Client Area | Masked     |

Aktifkan **Masked** untuk tiga variable kredensial tersebut.

## 5. Tambahkan Pipeline Deploy GitLab

Buat file `.gitlab-ci.yml` di root repository:

```bash
image: alpine:3.22

stages:
  - deploy

deploy_zip:
  stage: deploy
  only:
    - main
  before_script:
    - apk add --no-cache lftp curl zip
  script:
    - test -n "$FTP_SERVER" || (echo "FTP_SERVER belum diatur"; exit 1)
    - test -n "$FTP_USERNAME" || (echo "FTP_USERNAME belum diatur"; exit 1)
    - test -n "$FTP_PASSWORD" || (echo "FTP_PASSWORD belum diatur"; exit 1)

    - echo "1. Commit perubahan dan mirroring dari GitHub ke GitLab berhasil."
    - echo "2. Memulai proses pengemasan (zipping) seluruh file dan folder project di GitLab..."
    - zip -r deploy.zip . -x "*.git*" "*.github*" "*.gitlab-ci.yml" "deploy.zip"
    - echo "3. Arsip ZIP berhasil dibuat (deploy.zip)."

    - echo "4. Memicu skrip server untuk menghapus/membersihkan file dan folder lama di InfinityFree..."
    - curl -X GET "https://NAMA_DOMAIN_ANDA/unzip.php?token=RAHASIA_ZIP_DEPLOY&action=clean"

    - echo "5. Memulai proses pengunggahan file deploy.zip yang baru ke server InfinityFree via FTP..."
    - |
      lftp -c "
        set net:max-retries 10;
        set net:timeout 60;
        set ftp:passive-mode true;
        set xfer:clobber true;
        debug 3;
        open -u \"$FTP_USERNAME\",\"$FTP_PASSWORD\" \"$FTP_SERVER\";
        put deploy.zip -o htdocs/deploy.zip;
        bye
      "
    - echo "6. File deploy.zip berhasil diunggah ke direktori htdocs server."

    - echo "7. Proses ekstraksi file ZIP baru dan pembersihan arsip zip di server selesai."
    - echo "Deployment selesai sepenuhnya dengan sukses!"
```

_(Catatan: Ganti `https://NAMA_DOMAIN_ANDA/unzip.php` dengan URL domain website Anda yang sebenarnya)._

## 6. Uji Deploy Otomatis

1. Commit file `.gitlab-ci.yml` dan perubahan website, lalu push ke branch `main` di GitHub.
2. Buka tab **Actions** di GitHub dan pastikan job mirror berhasil.
3. Buka **Build > Pipelines** di GitLab dan pastikan job berjalan sampai sukses.
4. Buka domain Anda dan periksa apakah perubahan website sudah tampil dengan baik.

## Pemecahan Masalah

- **GitHub Actions gagal:** Pastikan secret `GITLAB_TOKEN` valid dan memiliki izin _write_repository_.
- **Pipeline error saat upload ZIP:** Pastikan koneksi server stabil dan ukuran file ZIP tidak melebihi batas wajar hosting gratis.
- **Ekstraksi gagal (Curl error/403):** Pastikan file `unzip.php` sudah diunggah ke folder `htdocs/` dan token rahasia di URL curl sama persis dengan yang ada di dalam skrip PHP.
- **PHP source code tampil di browser:** Pastikan file berekstensi `.php` dan server hosting memproses PHP dengan benar.

## Catatan

- Jangan commit informasi rahasia, token, atau kredensial database ke dalam repository publik.
- Pastikan skrip `unzip.php` sudah terpasang di server sebelum menjalankan pipeline otomatis untuk pertama kalinya.
