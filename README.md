# Deploy-Web-Menggunakan-Docker-Compose
Penugasan Oprec Camin Periode 2021 - Kelompok 1

## Anggota Kelompok : 
- Ishaq
- Iqbal
- Tauchid

## Breakdown Penugasan : 
-  Mount database mysql yang berada di container ke host
-  Web server menggunakan docker Nginx
-  PHP menggunakan docker PHP
-  Database menggunakan docker Mysql
-  Semua service yang ada (Nginx, PHP, dan Mysql) dijalankan bersama dalam docker-compose

## Tools / Requirement Yang Harus Disiapkan : 
- Repository Laravel (https://gitlab.com/kuuhaku86/web-penugasan-individu)
- Vhd Linux Server (https://drive.google.com/drive/folders/1W6en37tTDGLbh6o8pXOdY59EPtNHXG8G)
- Sudah berhasil install Nginx, PHP, MySql, Composer dalam linux server
- Sudah berhasil install docker dan docker-compose pada linux server

## Langkah - Langkah : 

### Clone Repository Laravel kedalam Server Linux
Clone Repository Laravel sesuai kebutuhan dan tujuan masing-masing, dalam kasus ini kami clone dari repo laravel yang sudah tersedia dan menamakan foldernya dengan penugasan-kelompok (dalam kasus kami, repo disimpan didalam folder /var/www/)

```bash
git clone https://gitlab.com/kuuhaku86/web-penugasan-individu penugasan-kelompok
```

### Composer Install Menggunakan Docker
Hal yang harus dilakukan adalah masuk kedalam folder repository laravel berada, kemudian jalankan perintah docker untuk execute composer install, dan set Permission pada directory repo laravel

```bash
cd /var/www/penugasan-kelompok
docker run --rm -v $(pwd):/app composer install
sudo chown -R $USER:$USER /var/www/penugasan-kelompok
sudo chmod -R 777 /var/www/penugasan-kelompok
```

### Membuat docker-compose.yml pada folder laravel
Dalam file docker-compose.yml akan disimpan apasaja service yang akan dipakai didalam container docker nantinya, seperti PHP, MySQL, dan NGINX.

Untuk Service PHP : 
```
#PHP Service
 app:
  build:
   context: .
   dockerfile: Dockerfile
  image: penugasan-kelompok
  container_name: app
  restart: unless-stopped
  tty: true
  environment:
   SERVICE_NAME: app
   SERVICE_TAGS: dev
  working_dir: /var/www/penugasan-kelompok
  volumes:
   - ./:/var/www/penugasan-kelompok
   - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
  networks:
   - app-network
```
Kami menamakan container untuk service php dengan nama "app", nama image penugasan-kelompok (ini biasanya nama repo), dockerfile mengikuti nama dockerfile nantinya yang akan dibuat untuk command instalasi tools yang dibutuhkan, working_dir untuk path ke repo laravel, untuk mount file dari container ke host kami menggunakan volumes.

Untuk Service Nginx :
```
#Nginx Service
 webserver:
  image: nginx:alpine
  container_name: webserver
  restart: unless-stopped
  tty: true
  ports: 
   - "8001:8001"
   - "444:444"
  volumes:
   - ./:/var/www/penugasan-kelompok
   - ./nginx/conf.d/:/etc/nginx/conf.d/
  networks:
   - app-network
```
Kami menggunakan nama container "webserver" untuk service nginx, kami disini menggunakan port 8001 sebagai tempat websitenya, mount file dari container ke host kami menggunakan volumes.

Untuk Service MySQL : 
```
#MySQL Service
 db:
  image: mysql
  container_name: db
  restart: unless-stopped
  tty: true
  ports:
   - "3307:3306"
  environment:
   MYSQL_DATABASE: penugasan_kelompok
   MYSQL_ROOT_PASSWORD: penugasan
   MYSQL_USER: root
   MYSQL_PASSWORD: penugasan
   SERVICE_TAGS: dev
   SERVICE_NAME: mysql
  volumes:
   - dbdata:/var/lib/mysql/
   - ./mysql/my.cnf:/etc/mysql/my.cnf
   - ./mysql-files:/var/lib/mysql-files
  networks:
   - app-network
```
Kami menggunakan nama containernya "db" untuk service MySQL, untuk port kami menggunakan port 3307 untuk local yang nantinya akan di forward ke port 3306 didalam container docker sehingga untuk .env laravel nanti bisa menggunakan 3306 (kasus ini terjadi karena untuk port 3306 di local sudah terinstall MySQL untuk keperluan project lain), untuk environment sesuaikan dengan data masing-masing dan kebutuhan masing-masing (nantinya akan dipakai di .env laravel). MySQL pada container akan di bind-mount ke `etc/mysql/my.cnf/` yang ada pada host.

Untuk Network dan Volumes :
```
#Docker Networks
networks:
 app-network:
  driver: bridge

#Volumes
volumes:
 dbdata:
  driver: local
```
Type network yang kami pakai adalah bridge.

*Untuk File Full dapat dilihat di repo github kami dengan file yang bernama docker-compose.yml

### Membuat DockerFile
Fungsi DockerFile adalah custom image yang dimana bisa digunakan untuk install tools atau aplikasi yang dibutuhkan nantinya pada web yang ingin kita deploy.

```bash
cd /var/www/penugasan-kelompok
sudo nano DockerFile
```

*Note : Jangan lupa sesuaikan versi php dengan kebutuhan

*Untuk File Full dapat dilihat di repo github kami dengan file yang bernama DockerFile

### Konfigurasi PHP
Buat folder PHP pada repo laravel

```bash
cd /var/www/penugasan-kelompok
mkdir php
sudo nano /php/local.ini
```

Isi File local.ini sesuai kebutuhan Aplikasi Web 
```
upload_max_filesize=40M
post_max_size=40M
```

### Konfigurasi NGINX
Buat folder NGINX pada repo laravel

```bash
cd /var/www/penugasan-kelompok
mkdir -p nginx/conf.d
sudo nano /nginx/conf.d/app.conf
```

Sesuaikan Isi file app.conf dengan kebutuhan Aplikasi Web dan Port sesuai yang sudah diatur sebelumnya
```
server {
	listen 8001;
	index index.php index.html;
	error_log /var/log/nginx/error.log;
	access_log /var/log/nginx/access.log;
	root /var/www/penugasan-kelompok/public;
	location ~ \.php$ {
		try_files $uri =404;
		fastcgi_split_path_info ^(.+\.php)(\.+)$;
		fastcgi_pass app:9000;
		fastcgi_index index.php;
		include fastcgi_params;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		fastcgi_param PATH_INFO $fastcgi_path_info;
	}
	location / {
		try_files $uri $uri/ /index.php?$query_string;
		gzip_static on;
	}
}
```

### Konfigurasi MySQL
Buat folder mysql pada repo laravel

```bash
cd /var/www/penugasan-kelompok
mkdir mysql
sudo nano /mysql/my.cnf
```

Isi File my.cnf sesuai kebutuhan WebApp 

```
[mysqld]
general_log = 1
general_log_file = /var/lib/mysql/general.log
```

### Buatlah .env File pada Folder Laravel

```bash
sudo nano .env
```

Isi file .env sesuai kebutuhan dan aturan pada docker-compose.yml yang sudah dibuat pada step sebelumnya

```
PP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:CefiKV2BVIKUzcP+EPYoDnxaoWXifMlTFlySdpSSv6Y=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=penugasan_kelompok
DB_USERNAME=root
DB_PASSWORD=penugasan

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
```
Untuk APP_KEY dibuat dengan : 
```bash
php artisan key:generate
```

Untuk DB, CONNECTION menggunakan "mysql", HOST menggunakan nama container MySQL di docker yaitu dalam kasus kami "db", Untuk PORT, DATABASE, USERNAME, PASSWORD sesuaikan kondisi.

### Build Application Image
Setelah semua step diatas sudah dilakukan dan tidak ada kendala maka, build App Image dengan command : 

```bash
docker-compose build app
```

### Run Docker Environment
Setelah build berhasil dilakukan, sekarang container bisa di run dengan command :

```bash
docker-compose up -d
```

Untuk cek apakah service berhasil jalan semua atau tidak dengan menggunakan command

```bash
docker ps
```

### Cache Laravel
Untuk jaga-jaga jangan lupa melakukan cache ketika mengubah file Docker

```bash
cd /var/www/penugasan-kelompok
docker-compose exec app php artisan config:cache
```

### Setting MySQL
Masuk kedalam MySQL Docker Container dengan cara : 

```bash
docker-compose exec db /bin/bash
```

Kemudian masuk kedalam MySQL dengan perintah : 

```bash
mysql -u root -p
```

Berikan akses Database MySQL kepada user yang dipakai di laravel : 

```sql
GRANT ALL ON * . * TO 'root'@'localhost' IDENTIFIED BY 'penugasan';
```

Flush Privileges untuk memberitahu perubahan pada MySQL

```sql
FLUSH PRIVILEGES;
```

Jika sudah dapat keluar dengan command 

```sql
EXIT
```

### Run Artisan Migrate di Folder Laravel

```bash
docker-compose exec app php artisan migrate
```

Jika migration berhasil maka Deploy selesai dilakukan.

## Referensi

- https://www.digitalocean.com/community/tutorials/how-to-set-up-laravel-nginx-and-mysql-with-docker-compose
- https://www.digitalocean.com/community/tutorials/how-to-install-and-set-up-laravel-with-docker-compose-on-ubuntu-20-04
