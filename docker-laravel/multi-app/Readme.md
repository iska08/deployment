# Panduan Deployment Multi-Application Laravel di _VPS_

Panduan ini menjelaskan langkah-langkah untuk melakukan deploy beberapa aplikasi Laravel dalam satu _VPS_ menggunakan _Docker_, _Apache (vhosts)_, _Let's Encrypt (HTTPS)_, otomatisasi _GitLab CI/CD_, serta penyiapan mirroring dari repository GitHub.

---

## 1\. Koneksi _Domain_ dengan _VPS_

- Masuk ke _Domain_ yang kita miliki, lalu klik `Manage DNS` dan pilih bagian `Records`.
- Tambah Records baru dengan isian berikut:
  - _Type_ : A
  - _Name_ : @
  - _Value_ : IP_VPS
  - _TTL_ : 3600
- Pastikan _Domain_ dan _VPS_ sudah terhubung dengan cek di [https://dnschecker.org/](https://dnschecker.org/).

---

## 2\. Atur _VPS_ agar Bisa Diakses Lewat _Command Prompt_

- Masuk ke _VPS_ yang kita miliki.
- Klik _Open Console_ dan login ke _VPS_ (_Ubuntu Server_).
- Masuk ke root dan berjalan di folder `(username@name-server:~$)` dan berganti ke `(root@name-server:/home/username#)`.

```bash
sudo su
```

- Masukkan perintah ini.

```bash
nano /etc/ssh/sshd_config
```

- Cari baris terakhir, ubah isian di bagian _PasswordAuthentication_ dari yang sebelumnya berisi _no_ ke _yes_.
- Lakukan _reload_.

```bash
sudo service ssh reload
```

- Buka _Command Prompt_ di komputer lalu ketikkan perintah di bawah dan masukkan password _VPS_.

```bash
ssh username_kita@domain_kita
```

- Masuk ke _root_.

```bash
sudo su
```

---

## 3\. _Install Docker_ di _VPS_

- Pastikan Anda sudah keluar dari _root_ `(username@name-server:~$)` lalu masukkan perintah update untuk update library yang dimiliki.

```bash
sudo apt update
```

- _Install_ sertifikat ini.

```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

- _Download Docker_.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- Ambil repository _Docker_.

```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```

- Instalasi _Docker_.

```bash
apt-cache policy docker-ce
```

- Masukkan perintah ini.

```bash
sudo apt install docker-ce
```

- Cek status _Docker_.

```bash
sudo systemctl status docker
```

- Mendaftarkan user yang dimiliki bisa berjalan dengan service yang dimiliki oleh _Docker_.

```bash
sudo usermod -aG docker ${USER}
```

- Ketikkan perintah ini dan masukkan password dari root.

```bash
su - ${USER}
```

- Masuk ke root dan ulangi langkah `sudo usermod -aG docker ${USER}` dan `su - ${USER}`.

```bash
sudo su
```

```bash
sudo usermod -aG docker ${USER}
```

```bash
su - ${USER}
```

- Cek grup yang tersedia.

```bash
groups
```

---

## 4\. Proses Membuat _User Deployer_ dan _Private Key_

- Masuk ke root terlebih dahulu `(root@name-server:~#)`.

```bash
sudo su
```

- Lalu masukkan perintah di bawah untuk membuat user _deployer_ dan berikan password yang berbeda dengan root.

```bash
sudo adduser deployer
```

- Lewati semua isian kecuali _Full Name_ dan pilih _Yes_.
- Berikan hak akses untuk _deployer_, dengan masuk ke folder home `cd /home/` lalu klik `ls` untuk memastikan folder _deployer_ dan folder dengan username Anda sudah ada.
- Buat folder baru untuk aplikasi, ulangi sesuai dengan jumlah aplikasi lalu cek isinya.

```bash
mkdir aplikasi1
```

```bash
ls -la
```

- _Install acl_ untuk memberikan hak akses.

```bash
sudo apt install acl
```

- Berikan hak akses ke _deployer_ semua aplikasi.

```bash
sudo setfacl -R -m u:deployer:rwx /home/aplikasi1
```

- Set permission yang perlu di folder tujuan aplikasi.

```bash
chmod 777 -R /home/aplikasi1/storage
```

```bash
chmod 777 -R /home/aplikasi1/public
```

- Ulangi langkah `sudo setfacl -R -m u:deployer:rwx /home/aplikasi1`, `chmod 777 -R /home/aplikasi1/storage`, dan `chmod 777 -R /home/aplikasi1/public` untuk semua aplikasi.
- Masuk ke user deployer `(deployer@name-server:/home$)`.

```bash
su deployer
```

- Jalankan perintah untuk membuat pasangan private key dan public key lalu tekan enter sampai selesai.

```bash
ssh-keygen -t rsa
```

- Copy isi dari public key ke authorized key.

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

---

## 5\. _Install PHP_, _MySQL_, dan _Apache_ dengan _Docker_

- Keluar dari user deployer.

```bash
exit
```

- Buat folder baru dengan nama `docker-stack`, lalu masuk ke folder tersebut `(root@name-server:/home/docker-stack#)`.

```bash
mkdir docker-stack
```

```bash
cd /docker-stack/
```

- Pastikan git sudah terinstall.

```bash
apt install git
```

- Clone file _Dockerfile_, _docker-compose.yml_, dan _vhosts.conf_ dari repository.

`Dockerfile`

```bash
FROM php:8.1-apache

RUN apt-get update && apt-get install -y \
    git libzip-dev zip unzip npm \
    certbot python3-certbot-apache

RUN docker-php-ext-install pdo pdo_mysql zip
RUN a2enmod rewrite

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Copy virtual host config
COPY vhosts.conf /etc/apache2/sites-available/
RUN a2dissite 000-default.conf
RUN a2ensite vhosts.conf

RUN service apache2 restart
```

`docker-compose.yml`

```bash
version: "3.3"
services:
  webserver:
    container_name: webserver
    build: .
    restart: always
    depends_on:
      - database
    volumes:
      - /home/aplikasi1:/var/www/aplikasi1
      - /home/aplikasi2:/var/www/aplikasi2
    ports:
      - "80:80"
      - "443:443"
    networks:
      - webnet

  database:
    container_name: database
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword123 # Ganti jika ingin
    ports:
      - "9906:3306"
    volumes:
      - ./dbdata:/var/lib/mysql
    networks:
      - webnet

  phpmyadmin:
    container_name: phpmyadmin
    image: phpmyadmin/phpmyadmin:latest
    restart: always
    depends_on:
      - database
    environment:
      PMA_HOST: database
      PMA_PORT: 3306
      PMA_ARBITRARY: 1
      UPLOAD_LIMIT: 100M
    ports:
      - "8080:80"
    networks:
      - webnet

networks:
  webnet:
    driver: bridge
```

> Untuk mengakses database melalui [http://IP_VPS:8080](http://IP_VPS:8080).

`vhosts.conf`

```bash
# Aplikasi 1
<VirtualHost *:80>
    ServerName nama-aplikasi1.com
    ServerAlias www.nama-aplikasi1.com
    DocumentRoot /var/www/nama-aplikasi1/public
    <Directory /var/www/nama-aplikasi1/public>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/nama-aplikasi1-error.log
    CustomLog ${APACHE_LOG_DIR}/nama-aplikasi1-access.log combined
</VirtualHost>

# Aplikasi 2
<VirtualHost *:80>
    ServerName nama-aplikasi2.com
    ServerAlias www.nama-aplikasi2.com
    DocumentRoot /var/www/nama-aplikasi2/public
    <Directory /var/www/nama-aplikasi2/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

> Ulangi untuk semua aplikasi.

- Cek isi folder.

```bash
ls -la
```

- Cek isi file _Dockerfile_.

```bash
cat Dockerfile
```

- Cek isi file _docker-compose.yml_.

```bash
cat docker-compose.yml
```

- Cek isi file _vhosts.conf_.

```bash
cat vhosts.conf
```

- Run compose.

```bash
docker compose up -d
```

- Run untuk cek images yang ada.

```bash
docker images
```

- Run untuk cek container yang ada.

```bash
docker container ls
```

---

## 6\. _Install Lets Encrypt HTTPS_ di _VPS_ dengan _Docker_

- Ketikkan perintah ini `(root@...........:var/www/html#)`.

```bash
docker exec -it webserver bash
```

- Masukkan perintah.

```bash
certbot --apache -d domain-anda.com -m email-anda@gmail.com
```

- _“You must agree in order to register with the ACME server. Do you agree?”_ -> Pilih Y.
- _“EFF new, campaigns, and ways to support digital freedom.”_ ->Pilih N.
- Masukkan perintah berikut agar _Domain_ kita terbaca www lalu pilih _Expand_.

```bash
certbot --apache -d domain-anda.com -d www.domain-anda.com -m email-anda@gmail.com
```

- _“Select the appropriate number [1-2] then [enter] (press ‘c’ to cancel):”_ Pilih yang terdapat nama _Domain_ Anda.
- Ulangi untuk aplikasi lainnya.

---

## 7\. Mirroring Repository GitHub ke GitLab Private

Langkah ini digunakan untuk mensinkronisasikan setiap repository aplikasi dari GitHub ke GitLab Private secara otomatis setiap kali ada perbaruan (`git push`). Ulangi konfigurasi ini pada seluruh project aplikasi Anda.

### Langkah 7.1: Buat Project Private Baru di GitLab

1. Login ke [GitLab](https://gitlab.com).
2. Klik **New project** > **Create blank project**.
3. Isi **Project name** (contoh: `aplikasi1`).
4. Set **Visibility Level** ke **Private**.
5. Hilangkan centang pada _Initialize repository with a README_.
6. Klik **Create project**.

### Langkah 7.2: Buat Personal Access Token (PAT) di GitLab

1. Di GitLab, klik foto profil (kiri bawah) > **Preferences** > **Access Tokens**.
2. Klik **Add new token**:

- **Token name:** `github-mirror-token`
- **Select scopes:** Centang **`write_repository`** dan **`read_repository`**.

3. Klik **Create personal access token**, lalu **salin token tersebut**.

### Langkah 7.3: Buat Secret di GitHub

1. Buka repo aplikasi Anda di [GitHub](https://github.com).
2. Masuk ke **Settings** > **Secrets and variables** > **Actions**.
3. Klik **New repository secret**:

- **Name:** `GITLAB_MIRROR_TOKEN`
- **Secret:** _Paste token dari Langkah 7.2_

4. Klik **Add secret**.

### Langkah 7.4: Workflow Mirroring di GitHub

Buat file `.github/workflows/mirror-to-gitlab.yml` pada repository project Anda:

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

## 8\. Menambahkan variabel di _Setting CI/CD GitLab_

- Kembali ke folder `(root@name-server:/home/docker-stack#)`, bisa dengan mengetikkan perintah ini.

```bash
exit
```

- Masuk sebagai _deployer_ `(deployer@name-server:/home/docker-stack$)`.

```bash
su deployer
```

- Ambil _private key_ dengan menjalankan perintah di bawah ini lalu copy _private key_ tersebut.

```bash
cat ~/.ssh/id_rsa
```

- Buka _GitLab_ dan masuk ke repository aplikasi Anda, pilih _Setting_ > _CI/CD_ > _Variable_ dan klik _“Add variable”_ untuk menambahkan SSH*PRIVATE_KEY, FILE_HTACCESS, dan FILE_ENV dengan isian berikut dan selalu matikan *“Protect variable”\_:

- SSH_PRIVATE_KEY
  - `Key` => SSH_PRIVATE_KEY
  - `Value` => Isi Private Key

> Keterangan: sama untuk semua project (cukup buat 1 kali di setiap project)

- FILE_HTACCESS
  - `Key` => FILE_HTACCESS
  - `Value`

```bash
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

> Keterangan: sama untuk semua project (cukup buat 1 kali di setiap project)

- FILE_ENV
  - `Key` => FILE_ENV
  - `Value` => Isi file .env

Contoh:

```bash
DB_CONNECTION=mysql
DB_HOST=database
DB_PORT=3306
DB_DATABASE=db_aplikasi1
DB_USERNAME=user_aplikasi1
DB_PASSWORD=pass_aplikasi1
```

> Keterangan: BERBEDA setiap project (isi .env berbeda, terutama DB_DATABASE, DB_USERNAME, DB_PASSWORD)

---

## 9\. Membuat _Runner_

- Masuk ke folder `docker-stack` dengan posisi root `(root@name-server:/home/docker-stack#)`, lalu jalankan perintah berikut.

```bash
docker run -d --name gitlab-runner --restart always -v /srv/gitlab-runner/config:/etc/gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
```

- Daftarkan _runner_ untuk setiap aplikasi.

```bash
docker exec -it gitlab-runner gitlab-runner register
```

- Masuk ke _GitLab_ tepatnya di repository kita, lalu pilih _Setting_ > _CI/CD_ > _Runners_ lalu klik _Expand_ dan copy _URL_ dan tokennya.

- Kemudian ikuti _prompt_:
  - `Enter GitLab instance URL` => Token dari _project_ aplikasi 1 (ambil dari GitLab)
  - `Enter description` => `runner-aplikasi1`
  - `Enter tags` => `aplikasi1` (atau kosongkan)
  - `Enter optional note` => (kosongkan)
  - `Enter executor` => `shell`
- Kembali ke _GitLab_ dan _uncheck “Enable shared runners for this project”_.
- Ulangi langkah `docker exec -it gitlab-runner gitlab-runner register` sampai `uncheck “Enable shared runners for this project”` untuk masing-masing aplikasi.
- Untuk memberikan akses user _deployer_ agar memiliki _group_ yang sama untuk melakukan akses terhadap _docker_ jalankan perintah berikut:

```bash
sudo usermod -aG docker deployer
```

- Lihat semua _runner_ yang terdaftar.

```bash
docker exec -it gitlab-runner gitlab-runner list
```

---

## 10\. Menambahkan File `.gitlab-ci.yml`

- Buka project _Laravel_ dan buat file `.gitlab-ci.yml`.
- Masukkan kode berikut ke file `.gitlab-ci.yml`:

```bash
stages:
    - deploy

Deploy:
    stage: deploy
    timeout: 30 minutes
    tags:
        - aplikasi1 # Sama dengan isi “Enter tags”

variables:
    VAR_DIREKTORI: "/home/aplikasi1"
    VAR_GIT_URL_TANPA_HTTP: "gitlab.com/username/aplikasi1.git"
    VAR_CLONE_KEY: "xxx"
    VAR_USER: "xxx"
    VAR_IP: "xxx" # IP_VPS
    VAR_FILE_ENV: $FILE_ENV
    VAR_FILE_HTACCESS: $FILE_HTACCESS

before_script:
    - which ssh-agent || ( sudo apt-get update && sudo apt-get install -y openssh-client ) - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - - mkdir -p ~/.ssh - chmod 700 ~/.ssh - ssh-keyscan -H $VAR_IP >> ~/.ssh/known_hosts 2>/dev/null
    - chmod 644 ~/.ssh/known_hosts
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
    - echo "$VAR_FILE_HTACCESS"

script:
    - ssh $VAR_USER@$VAR_IP "git config --global safe.directory '\*'"

    # CLONE ATAU PULL REPOSITORY
    - ssh $VAR_USER@$VAR_IP "if [ -d $VAR_DIREKTORI/.git ]; then cd $VAR_DIREKTORI && git pull https://oauth2:$VAR_CLONE_KEY@$VAR_GIT_URL_TANPA_HTTP; else git clone https://oauth2:$VAR_CLONE_KEY@$VAR_GIT_URL_TANPA_HTTP $VAR_DIREKTORI; fi"

    # Hapus stash lama
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && git stash clear 2>/dev/null || true"

    # Force reset hard
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && git fetch origin && git reset --hard origin/main && git clean -fd"

    # ===============================================================================
    # SETUP ENVIRONMENT FILES
    # ===============================================================================
    - ssh $VAR_USER@$VAR_IP "rm -f $VAR_DIREKTORI/.env"
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && echo '$VAR_FILE_ENV' > .env"
    - ssh $VAR_USER@$VAR_IP "rm -f $VAR_DIREKTORI/.htaccess"
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && echo '$VAR_FILE_HTACCESS' > .htaccess"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/aplikasi1/bootstrap/cache"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/aplikasi1/storage/framework/cache"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/aplikasi1/storage/framework/sessions"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/aplikasi1/storage/framework/views"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver chmod -R 777 /var/www/aplikasi1/bootstrap/cache"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver chmod -R 777 /var/www/aplikasi1/storage"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver chown -R www-data:www-data /var/www/aplikasi1/storage /var/www/aplikasi1/bootstrap/cache"

    # ===============================================================================
    # SETUP DATABASE (BUAT DATABASE DAN USER JIKA BELUM ADA)
    # ===============================================================================
    - ssh $VAR_USER@$VAR_IP "docker exec database mysql -uroot -prootpassword123 -e 'CREATE DATABASE IF NOT EXISTS db_aplikasi1;'"
    - ssh $VAR_USER@$VAR_IP "docker exec database mysql -uroot -prootpassword123 -e 'CREATE USER IF NOT EXISTS \"user_aplikasi1\"@\"%\" IDENTIFIED BY \"pass_aplikasi1\";'"
    - ssh $VAR_USER@$VAR_IP "docker exec database mysql -uroot -prootpassword123 -e 'GRANT ALL PRIVILEGES ON db_aplikasi1.* TO \"user_aplikasi1\"@\"%\";'"
    - ssh $VAR_USER@$VAR_IP "docker exec database mysql -uroot -prootpassword123 -e 'FLUSH PRIVILEGES;'"

    # ===============================================================================
    # INSTALL DEPENDENCIES DAN JALANKAN ARTISAN COMMANDS
    # ===============================================================================
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && composer install --ignore-platform-reqs --no-interaction --no-progress'"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && php artisan key:generate --no-interaction'"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && php artisan migrate:fresh --seed'"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && php artisan storage:link'"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && composer dump-autoload --ignore-platform-reqs --no-interaction'"

    # ===============================================================================
    # CRON JOBS
    # ===============================================================================

    # Aktifkan cron job untuk sitemap
    - echo "Mengaktifkan cron job untuk sitemap..."

    # Generate sitemap pertama kali
    - ssh $VAR_USER@$VAR_IP "docker exec webserver bash -c 'cd /var/www/aplikasi1 && php artisan sitemap:generate'"

    # Hapus cron job lama jika ada
    - ssh $VAR_USER@$VAR_IP "crontab -l 2>/dev/null | grep -v 'sitemap:generate' | crontab - 2>/dev/null || true"

    # ===============================================================================
    # TAMBAHKAN CRON JOB BARU KE CRONTAB
    # ===============================================================================

    # * * * * * = Setiap menit
    # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "* * * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # */5 * * * * = Setiap 5 menit
    # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "*/5 * * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # */30 * * * * = Setiap 30 menit
    # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "*/30 * * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # 0 * * * * = Setiap jam (tepat jam)
    # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 * * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # 0 */6 * * * = Setiap 6 jam
    # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 */6 * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # 0 0 * * * = Setiap hari jam 00:00 (tengah malam)
    - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 0 * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # 0 12 * * * = Setiap hari jam 12:00 siang
    - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 12 * * * docker exec webserver bash -c \"cd /var/www/aplikasi1 && php artisan sitemap:generate\" >> /var/www/aplikasi1/storage/logs/cron.log 2>&1") | crontab -'

    # ===============================================================================
    # CEK APAKAH CRON JOB BERHASIL DITAMBAHKAN
    # ===============================================================================
    - ssh $VAR_USER@$VAR_IP "crontab -l"

    # ===============================================================================
    # SELESAI
    # ===============================================================================
    - echo "Deployment selesai! Cron job untuk sitemap telah diaktifkan!"
```

- Untuk **VAR_CLONE_KEY** kita masuk ke _GitLab_ lalu klik foto profil dan pilih **“Edit profile”**, kemudian cari **“Access Tokens”**.

- Untuk **“Token name”** isi dengan **deployer** dan untuk **“Select scopes”** checklist semuanya lalu klik **“Create personal access token”**.
- Copy token yang sudah dibuat dan letakkan di **VAR_CLONE_KEY** tadi.
- Ganti isi **VAR_USER** dengan **deployer**.
- Ganti isi **VAR_IP** dengan **IP_VPS** Anda.
- Ulangi untuk aplikasi yang lain.

---

## 11\. Tambah permission ke folder `/home/aplikasi1` (Opsional)

Jika muncul pesan error _“/home/aplikasi1/.git: Permission denied”_ saat push ke repository, jalankan perintah di bawah dan pastikan berada di folder dalam mode root `(root@name-server:/home/aplikasi1#)`.

```bash
sudo setfacl -R -m u:deployer:rwx /home/aplikasi1/
```

---

## 12\. Tambah permission ke folder storage dan public (Opsional)

- Coba akses website, jika terdapat pesan error karena tidak ada akses folder storage maka jalankan perintah di bawah.
- Tetap di folder `(root@name-server:/home/aplikasi1#)` dalam mode root lalu jalankan 2 perintah di bawah ini.

```bash
chmod 777 -R /home/aplikasi1/storage
```

```bash
chmod 777 -R /home/aplikasi1/public
```
