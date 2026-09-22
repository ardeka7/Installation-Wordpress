# Install WordPress di Ubuntu (AWS EC2) — LEMP Stack

Panduan lengkap instalasi WordPress dari nol di instance Ubuntu EC2, menggunakan **Nginx + MySQL + PHP** (LEMP stack).

## Prasyarat

- Instance EC2 Ubuntu sudah berjalan (`Running`)
- Bisa SSH ke instance (via PuTTY/terminal)
- Security Group instance sudah membuka port:
  - **22** (SSH)
  - **80** (HTTP)
  - **443** (HTTPS)

---

## 1. Update Sistem

```bash
sudo apt update && sudo apt upgrade -y
```

> Jika muncul pesan kernel baru tersedia (misal `7.0.0-1012-aws`), lakukan reboot agar kernel aktif:
> ```bash
> sudo reboot
> ```
> Tunggu 30–60 detik, lalu SSH ulang. Sesi lama akan otomatis terputus ("Remote side unexpectedly closed network connection") — ini normal.

Cek kernel aktif:

```bash
uname -r
```

---

## 2. Install Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Cek status:

```bash
sudo systemctl status nginx
```

Harus menampilkan `active (running)`.

### Cek IP publik instance

```bash
curl -s http://checkip.amazonaws.com
```

Atau lihat di **AWS Console → EC2 → Instances → kolom Public IPv4 address**.

> ⚠️ Gunakan **IP publik**, bukan IP privat (`172.31.x.x`) — IP privat tidak bisa diakses dari luar VPC.

Uji akses: buka `http://<IP-publik>` di browser, harus muncul halaman default **"Welcome to nginx!"**.

### Jika muncul "refused to connect" / "site can't be reached"

Cek berurutan:

1. **Security Group** (AWS Console → EC2 → Security Groups → pilih SG instance → Inbound rules) — pastikan ada rule `HTTP`, port `80`, source `0.0.0.0/0`.
2. **Firewall lokal (ufw)**:
   ```bash
   sudo ufw status
   ```
   Jika `active` dan port 80/443 belum diizinkan:
   ```bash
   sudo ufw allow 'Nginx Full'
   sudo ufw allow OpenSSH
   sudo ufw reload
   ```
3. **Nginx benar-benar listen di port 80**:
   ```bash
   sudo ss -tulpn | grep :80
   ```
   Harus muncul proses `nginx` di baris `0.0.0.0:80`.
4. **Network ACL** (VPC → Network ACLs) — pastikan tidak ada rule DENY untuk port 80 di subnet instance.

---

## 3. Install MySQL (MariaDB/MySQL Server)

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

Ikuti wizard: set password root, jawab `Y` untuk semua pertanyaan keamanan (remove anonymous users, disallow remote root login, remove test database, reload privilege tables).

### Buat database & user WordPress

```bash
sudo mysql -u root -p
```

Di prompt MySQL:

```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'PasswordKuat123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> Ganti `PasswordKuat123!` dengan password kuat versi sendiri, dan simpan baik-baik (dipakai lagi di step 6).

---

## 4. Install PHP dan Modul yang Dibutuhkan

```bash
sudo apt install php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y
```

Cek versi PHP terpasang:

```bash
php -v
```

Catat versi PHP-nya (contoh di deployment ini: **PHP 8.5**), dan pastikan nama file socket PHP-FPM:

```bash
ls /run/php/
```

Contoh hasil: `php8.5-fpm.sock` — nama ini dipakai di konfigurasi Nginx pada step 6.

---

## 5. Download & Pasang WordPress

```bash
cd /tmp
curl -O https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mkdir -p /var/www/wordpress
sudo cp -a wordpress/. /var/www/wordpress/
```

Set permission:

```bash
sudo chown -R www-data:www-data /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
```

---

## 6. Konfigurasi Nginx untuk WordPress

Buat file config baru:

```bash
sudo nano /etc/nginx/sites-available/wordpress
```

Isi (sesuaikan `server_name` dengan IP publik atau domain, dan `fastcgi_pass` dengan versi PHP kamu):

```nginx
server {
    listen 80;
    server_name 34.230.73.113;  # ganti dengan IP publik / domain
    root /var/www/wordpress;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;  # sesuaikan versi PHP
    }

    location ~ /\.ht {
        deny all;
    }

    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt { log_not_found off; access_log off; allow all; }

    client_max_body_size 64M;
}
```

Simpan (`Ctrl+O`, `Enter`, `Ctrl+X`).

Aktifkan config dan nonaktifkan default:

```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

`nginx -t` harus menampilkan:
```
syntax is ok
test is successful
```

---

## 7. Instalasi WordPress via Browser

Buka `http://<IP-publik-atau-domain>` di browser. Wizard instalasi WordPress akan muncul:

1. Pilih bahasa
2. Isi detail koneksi database:
   - **Database Name**: `wordpress_db`
   - **Username**: `wp_user`
   - **Password**: password yang dibuat di step 3
   - **Database Host**: `localhost`
   - **Table Prefix**: biarkan default `wp_`
3. Klik **Run the installation**
4. Isi judul situs, username admin, password admin, email
5. Klik **Install WordPress**

Selesai — WordPress sudah bisa diakses dan dikelola lewat `/wp-admin`.

---

## 8. (Disarankan) Pasang SSL Gratis dengan Let's Encrypt

Hanya bisa dilakukan jika sudah punya **domain** yang diarahkan (DNS A record) ke IP publik instance ini.

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d namadomainmu.com -d www.namadomainmu.com
```

Ikuti instruksi di terminal. Certbot akan otomatis mengonfigurasi ulang Nginx untuk HTTPS dan mengatur perpanjangan sertifikat otomatis.

---

## Troubleshooting Cepat

| Masalah | Kemungkinan Penyebab | Solusi |
|---|---|---|
| SSH putus setelah `reboot` | Normal, instance sedang restart | Tunggu ~30–60 detik, connect ulang |
| Browser "refused to connect" | Port 80 belum dibuka di Security Group | Tambah inbound rule HTTP port 80, source `0.0.0.0/0` |
| Browser "site can't be reached" pakai IP privat | Salah pakai IP privat (`172.31.x.x`) | Gunakan Public IPv4 address dari AWS Console |
| Nginx error 502 Bad Gateway | Socket PHP-FPM salah / PHP-FPM tidak jalan | Cek `ls /run/php/` dan cocokkan dengan `fastcgi_pass` di config Nginx |
| WordPress tidak bisa connect ke database | Salah nama DB/user/password | Cek ulang kredensial di step 3 |

---

## Ringkasan Perintah (Quick Reference)

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install nginx -y
sudo systemctl enable nginx --now

# Install MySQL
sudo apt install mysql-server -y
sudo mysql_secure_installation

# Install PHP
sudo apt install php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y

# Download WordPress
cd /tmp && curl -O https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mkdir -p /var/www/wordpress
sudo cp -a wordpress/. /var/www/wordpress/
sudo chown -R www-data:www-data /var/www/wordpress

# Test & restart Nginx setelah config dibuat
sudo nginx -t
sudo systemctl restart nginx
```
