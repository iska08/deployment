# Panduan Deployment Single-Application Laravel di _VPS_

Panduan ini menjelaskan langkah-langkah untuk melakukan deploy beberapa aplikasi Laravel dalam satu _VPS_ menggunakan _Docker_, _Apache (vhosts)_, _Let's Encrypt (HTTPS)_, dan otomatisasi _GitLab CI/CD_.

## 1\. Koneksi _Domain_ dengan _VPS_

- Masuk ke domain yang kita miliki, lalu klik `Manage DNS` dan pilih bagian `Records`.
- Tambah Records baru dengan isian berikut:
  - Type : A
  - Name : @
  - Value : IP_VPS
  - TTL : 3600
- Pastikan _Domain_ dan _VPS_ sudah terhubung dengan cek di [https://dnschecker.org/](https://dnschecker.org/).

## 2\. _Setting VPS_ Agar Bisa Diakses Lewat _Command Prompt_

- Masuk ke _VPS_ yang kita miliki.
- Klik Open Console dan login ke _VPS_ (_Ubuntu Server_).
- Masuk ke root dan pastikan berjalan di folder `(username@name-server:~$)` lalu berganti ke `(root@name-server:/home/username#)`.

```bash
sudo su
```

- Masukkan perintah di bawah. Cari baris terakhir, ubah isian di bagian _PasswordAuthentication_ dari yang sebelumnya berisi _no_ ke _yes_.

```bash
nano /etc/ssh/sshd_config
```

- Lakukan reload.

```bash
sudo service ssh reload
```

- Buka _Command Prompt_ di komputer lalu ketikkan perintah berikut dan masukkan password _VPS_.

```bash
ssh username_kita@domain_kita
```

- Masuk ke root.

```bash
sudo su
```

## 3\. _Install Docker_ di _VPS_

- Pastikan Anda sudah keluar dari root `(username@name-server:~$)` lalu masukkan perintah berikut untuk update library yang dimiliki.

```bash
sudo apt update
```

- Install sertifikat ini.

```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

- Download _Docker_.

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

- Masukkan perintah berikut:

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

- Ketikkan perintah berikut dan masukkan password dari root.

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

- Cek grup yang tersedia

```bash
groups
```

## 4\. Proses Membuat _User Deployer_ dan _Private Key_

- Masuk ke root terlebih dahulu `(root@name-server:~#)`.

```bash
sudo su
```

- Lalu buat user deployer dan berikan password yang berbeda dengan root.

```bash
sudo adduser deployer
```

- Lewati semua isian kecuali _Full Name_ dan pilih _Yes_.
- Berikan hak akses untuk _deployer_, dengan masuk ke folder home `cd /home/` lalu klik `ls` untuk memastikan folder _deployer_ dan folder dengan username Anda sudah ada.
- Buat folder baru lalu cek isinya.

```bash
mkdir nama_aplikasi
```

```bash
ls -la
```

- Install acl untuk memberikan hak akses.

```bash
sudo apt install acl
```

- Berikan hak akses ke deployer.

```bash
sudo setfacl -R -m u:deployer:rwx /home/nama_aplikasi
```

- Set permission yang perlu di folder tujuan.

```bash
chmod 777 -R /home/nama_aplikasi/storage
```

```bash
chmod 777 -R /home/nama_aplikasi/public
```

- Masuk ke user deployer `(deployer@name-server:/home$)`.

```bash
su deployer
```

- Jalankan perintah untuk membuat pasangan private key dan public key, lalu tekan enter sampai selesai.

```bash
ssh-keygen -t rsa
```

- Copy isi dari public key ke authorized key.

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

## 5\. _Install PHP_, _MySQL_, dan _Apache_ dengan _Docker_

- Keluar dari user deployer.

```bash
exit
```

- Masuk ke folder aplikasi kita `(root@name-server:/home/nama_aplikasi#)`.

```bash
cd /nama_aplikasi/
```

- Pastikan git sudah terinstall.

```bash
apt install git
```

- Clone file `Dockerfile` dan `docker-compose.yml` dari repository.

```bash
git clone https://github.com/iska08/deploy-laravel-single-application.git .
```

`Dockerfile`

```bash
FROM php:8.1-apache

RUN apt-get update
RUN apt-get install -y git libzip-dev zip unzip npm
RUN docker-php-ext-install pdo pdo_mysql zip
RUN a2enmod rewrite
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# install lets encrypt
RUN apt-get -y install certbot python3-certbot-apache
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
            - ./www:/var/www/html/
        ports:
            - 80:80 - 443:443

    database:
        container_name: database
        image: mysql:latest
        restart: always
        environment:
            MYSQL_ROOT_PASSWORD: MYSQL_PASSWORD
            MYSQL_DATABASE: MYSQL_DATABASE
            MYSQL_USER: MYSQL_USER
            MYSQL_PASSWORD: MYSQL_PASSWORD
        ports:
            - 9906:3306
        volumes:
            - ./dbdata:/var/lib/mysql

    phpmyadmin:
        container_name: phpmyadmin
        image: phpmyadmin/phpmyadmin:latest
        restart: always
        depends_on:
            - database
        environment:
            PMA_HOST: database # Nama container MySQL Anda (database)
            PMA_PORT: 3306
            PMA_ARBITRARY: 1 # Memungkinkan koneksi ke server lain
            UPLOAD_LIMIT: 100M
        ports:
            - 8080:80 # Akses melalui port 8080
        networks:
            - default
```

- Cek isi folder.

```bash
ls -la
```

- Cek isi file `Dockerfile`.

```bash
cat Dockerfile
```

- Cek isi file `docker-compose.yml`.

```bash
cat docker-compose.yml
```

- Run compose.

```bash
docker compose up -d
```

- Run images untuk cek images yang ada.

```bash
docker images
```

- Run container untuk cek container yang ada.

```bash
docker container ls
```

## 6\. _Install Lets Encrypt HTTPS_ di _VPS_ dengan _Docker_

- Masuk ke folder www, `cd /www/` lalu ketikkan perintah di bawah `(root@********:var/www/html#)`.

```bash
docker exec -it webserver bash
```

- Masukkan perintah berikut:

```bash
certbot --apache -d domain-anda.id -m email-anda@gmail.com
```

- _“You must agree in order to register with the ACME server. Do you agree?”_ -> Pilih Y.
- _“EFF new, campaigns, and ways to support digital freedom.”_ ->Pilih N.
- Masukkan perintah berikut agar domain kita terbaca `www`, lalu pilih Expand.

```bash
certbot --apache -d domain-anda.id -d www.domain-anda.id -m email-anda@gmail.com
```

- _“Select the appropriate number [1-2] then [enter] (press ‘c’ to cancel):”_ Pilih yang terdapat nama domain Anda.

## 7\. Menambahkan variabel di _Setting CI/CD GitLab_

- Kembali ke folder `(root@name-server:/home/nama_aplikasi/www#)`, bisa dengan mengetikkan perintah `exit`.

```bash
exit
```

- Masuk sebagai `deployer` `(deployer@name-server:/home/nama_aplikasi/www$)`.

```bash
su deployer
```

- Ambil private key lalu copy private key tersebut.

```bash
cat ~/.ssh/id_rsa
```

- Buka _GitLab_ dan masuk ke repository aplikasi Anda, pilih _Setting_ > _CI/CD_ > _Variable_ dan klik _“Add variable”_ untuk menambahkan SSH*PRIVATE_KEY, FILE_HTACCESS, dan FILE_ENV dengan isian berikut dan selalu matikan *“Protect variable”\_:

- SSH_PRIVATE_KEY
  - `Key` => SSH_PRIVATE_KEY
  - `Value` => Isi Private Key

- FILE_HTACCESS
  - `Key` => FILE_HTACCESS
  - `Value`

```bash
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.\*)$ public/$1 [L]
</IfModule>
```

- FILE_ENV
  - `Key` => FILE_ENV
  - `Value` => Isi file .env

Ganti isi “DB_HOST” dengan nama container mysql _“database”_ atau bisa dicek dengan exit dari user deployer `(root@name-server:/home/nama_aplikasi/www#)` lalu jalankan `docker container ls` lalu cari _“NAMES”_ dari mysql. Samakan isi DB_DATABASE, DB_USERNAME, dan DB_PASSWORD dengan yang ada di file `docker-compose.yml`.

## 8\. Membuat _Runner_

- Pastikan Anda di folder ini (root@name-server:/home/nama_aplikasi#) dengan posisi root, lalu jalankan perintah berikut:

```bash
docker run -d --name gitlab-runner --restart always -v /srv/gitlab-runner/config:/etc/gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
```

- Daftarkan _Runner_.

```bash
docker exec -it gitlab-runner gitlab-runner register
```

- Masuk ke _GitLab_ tepatnya di repository kita. Pilih _Setting_ > _CI/CD_ > _Runners_ lalu klik _Expand_ dan copy _URL_ dan tokennya.
- Masukkan _URL_ dan tokennya lalu lewati semuanya dengan cara enter.
- _“Enter an executor:”_ pilih shell.
- Kembali ke _GitLab_ dan uncheck _“Enable shared runners for this project”_.
- Berikan akses user _deployer_ agar memiliki group yang sama untuk melakukan akses terhadap _docker_.

```bash
sudo usermod -aG docker deployer
```

## 9\. Menambahkan file `.gitlab-ci.yml`

- Buka project Laravel dan buat file `.gitlab-ci.yml`.
- Masukkan kode berikut ke file `.gitlab-ci.yml`:

```bash
stages:
    - deploy

Deploy:
    stage: deploy
    timeout: 30 minutes

    variables:
        VAR_DIREKTORI: "/home/nama_aplikasi/www"
        VAR_GIT_URL_TANPA_HTTP: "gitlab.com/iska08/nama_aplikasi.git"
        VAR_CLONE_KEY: "xxx"
        VAR_USER: "xxx"
        VAR_IP: "xxx"
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
        - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/html/bootstrap/cache"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/html/storage/framework/cache"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/html/storage/framework/sessions"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver mkdir -p /var/www/html/storage/framework/views"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver chmod -R 777 /var/www/html/bootstrap/cache"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver chmod -R 777 /var/www/html/storage"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache"

        # ===============================================================================
        # INSTALL DEPENDENCIES DAN JALANKAN ARTISAN COMMANDS
        # ===============================================================================
        - ssh $VAR_USER@$VAR_IP "docker exec webserver composer install --ignore-platform-reqs --no-interaction --no-progress"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan key:generate --no-interaction"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan migrate --seed --force"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan storage:link"
        - ssh $VAR_USER@$VAR_IP "docker exec webserver composer dump-autoload --ignore-platform-reqs --no-interaction"

        # ===============================================================================
        # CRON JOBS
        # ===============================================================================

        # Aktifkan cron job untuk sitemap
        - echo "Mengaktifkan cron job untuk sitemap..."

        # Generate sitemap pertama kali
        - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan sitemap:generate"

        # Hapus cron job lama jika ada
        - ssh $VAR_USER@$VAR_IP "crontab -l 2>/dev/null | grep -v 'sitemap:generate' | crontab - 2>/dev/null || true"

        # ===============================================================================
        # TAMBAHKAN CRON JOB BARU KE CRONTAB
        # ===============================================================================

        # * * * * * = Setiap menit
        # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "* * * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # */5 * * * * = Setiap 5 menit
        # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "*/5 * * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # */30 * * * * = Setiap 30 menit
        # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "*/30 * * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # 0 * * * * = Setiap jam (tepat jam)
        # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 * * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # 0 */6 * * * = Setiap 6 jam
        # - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 */6 * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # 0 0 * * * = Setiap hari jam 00:00 (tengah malam)
        - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 0 * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # 0 12 * * * = Setiap hari jam 12:00 siang
        - ssh $VAR_USER@$VAR_IP '(crontab -l 2>/dev/null; echo "0 12 * * * docker exec webserver php /var/www/html/artisan sitemap:generate >> /var/www/html/storage/logs/cron.log 2>&1") | crontab -'

        # ===============================================================================
        # CEK APAKAH CRON JOB BERHASIL DITAMBAHKAN
        # ===============================================================================
        - ssh $VAR_USER@$VAR_IP "crontab -l"

        # ===============================================================================
        # SELESAI
        # ===============================================================================
        - echo "Deployment selesai! Cron job untuk sitemap telah diaktifkan!"
```

- Untuk VAR_CLONE_KEY kita masuk ke Gitlab lalu klik foto profil dan pilih _“Edit profile”_, kemudian cari _“Access Tokens”_.
- Untuk _“Token name”_ isi dengan _deployer_ dan untuk _“Select scopes”_ checklist semuanya lalu klik _“Create personal access token”_.
- Copy token yang sudah dibuat dan letakkan di VAR_CLONE_KEY tadi.
- Ganti isi VAR_USER dengan _deployer_.
- Ganti isi VAR_IP dengan IP_VPS Anda.

## 10\. Tambah permission ke folder `/home/nama_aplikasi/www` (Opsional)

Jika muncul pesan error “/home/nama_aplikasi/www/.git: Permission denied” saat push ke repository.

- Pastikan berada di folder `(root@name-server:/home/nama_aplikasi/www#)`, lalu jalankan perintah ini.

```bash
sudo setfacl -R -m u:deployer:rwx /home/nama_aplikasi/www
```

- Pastikan saat menjalankan perintah di atas berada di folder `/home/nama_aplikasi/www/` dalam mode `root`.

## 11\. Tambah permission ke folder _storage_ dan _public_ (Opsional)

- Coba akses website, jika terdapat pesan error karena tidak ada akses folder storage maka jalankan perintah di bawah.
- Tetap di folder `(root@name-server:/home/nama_aplikasi/www#)` dalam mode `root` lalu jalankan ini.

```bash
chmod 777 -R /home/nama_aplikasi/www/storage
```

```bash
chmod 777 -R /home/nama_aplikasi/www/public
```
