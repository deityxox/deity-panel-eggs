# 🚀 Pelican Panel + Cloudflare Tunnel Kurulum Rehberi (Docker ile)

---

cr okNick : https://www.youtube.com/watch?v=tdEpa2bRfJk

## 🧰 1. Sunucu Hazırlığı

```bash
apt update -y && apt upgrade -y
add-apt-repository ppa:ondrej/php
apt install php8.4 php8.4-{gd,mysql,mbstring,bcmath,xml,curl,zip,intl,sqlite3,fpm} -y
apt install nginx -y
```

```bash
sudo mkdir -p /var/www/pelican
cd /var/www/pelican
curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | sudo tar -xzv
```

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
sudo COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
```

---

## 🔐 2. SSL Sertifikası Oluşturma

```bash
mkdir /etc/certs
cd /etc/certs
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout privkey.pem -out fullchain.pem
```

---

## 🌐 3. NGINX Yapılandırması

```bash
sudo rm /etc/nginx/sites-enabled/default
nano /etc/nginx/sites-available/pelican.conf
```

### İçeriği yapıştır:

```nginx
server_tokens off;

server {
    listen 80;
    server_name _;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name _;

    root /var/www/pelican/public;
    index index.php;

    access_log /var/log/nginx/pelican.app-access.log;
    error_log  /var/log/nginx/pelican.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    ssl_certificate /etc/certs/fullchain.pem;
    ssl_certificate_key /etc/certs/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
sudo systemctl restart nginx
```

---

## ⚙️ 4. Panel Ortam Ayarları

```bash
cd /var/www/pelican
php artisan p:environment:setup
sudo chmod -R 755 storage/* bootstrap/cache/
sudo chown -R www-data:www-data /var/www/pelican
```

---

## 🐳 5. Docker Kurulumu

```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable sudo sh
```

---

## 🌐 6. Cloudflare Tunnel Kurulumu (Docker)

1. Cloudflare > Zero Trust > Networks > Tunnels > Create Tunnel
2. Tünel ismini belirle
3. Docker seç, komutu kopyala:

```bash
docker run ...
```

4. Bağlantı geldikten sonra işlemi durdur:

```bash
CTRL+C
```

5. Arka planda kalıcı çalıştır:

```bash
docker run -d --restart unless-stopped ...
```

6. Durum "Connected" ise Next > Subdomain ayarlarına geç.

---

## 🌍 7. Subdomain Ayarları

* Subdomain: `panel.domain.com`
* Type: `HTTPS`
* URL: `Server IP`
* No TLS Verify: **Aktif Et**
* Save Tunnel

---

## 🧩 8. Pelican Web Kurulumu

Tarayıcıdan aç:

```bash
https://panel.domain.com/installer
```

Adımlar:

* Next
* Create User
* Next (sqlite)
* Next (filesystem)
* Next (database) → Komutları çalıştır, Check butonuna bas
* Finish

---

## 🧱 9. Node Kurulumu

1. Panel > Nodes > New Node
2. Node adı: `node.domain.com`
3. Create > Edit > Configuration File kısmına geç. Sonrasında oradaki içerik kopyalanacak ve aşağıdaki dosyaya yapıştırılacak:

```bash
mkdir /etc/pelican
nano /etc/pelican/config.yml
```

Yapılandırmaya şu satırları ekle:

```yaml
cert: /etc/certs/fullchain.pem
key: /etc/certs/privkey.pem
```

Kaydet, çık.

---

## 🐦 10. Wings Kurulumu

```bash
sudo mkdir -p /etc/pelican /var/run/wings
sudo curl -L -o /usr/local/bin/wings "https://github.com/pelican-dev/wings/releases/latest/download/wings_linux_$( [[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

```bash
nano /etc/systemd/system/wings.service
```

İçeriği yapıştır:

```ini
[Unit]
Description=Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pelican
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Kaydet, çık ve çalıştır:

```bash
sudo systemctl enable --now wings
```

---

## 🌐 11. Cloudflare Üzerinden Node Yayınlama

Cloudflare Tunnel > Edit > Public Hostname > Add Hostname:

* Subdomain: `node`
* Domain: domain.com
* Type: `HTTPS`
* URL: `Server IP:8080`
* No TLS Verify: Açık
* Save

---

## 🔧 12. Node Ayarları Sonrası

Panel > Edit Node > Basic Settings > Port: `443` olarak ayarla > Save > Health kontrolü yap ✅
