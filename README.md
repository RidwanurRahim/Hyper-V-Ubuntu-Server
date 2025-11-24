# 🐧 Installing Ubuntu Server on Hyper-V with Network Access

This guide explains how to install **Ubuntu Server** on a **Hyper-V virtual machine**, configure network connectivity using **Virtual Switch Manager**, and ensure the VM has access to the internet and your local network.

---

## 📋 Prerequisites

- **Windows 10 / 11 / Server** with **Hyper-V** enabled.
- **Ubuntu Server ISO** (download from [ubuntu.com/download/server](https://ubuntu.com/download/server)).
- A **network connection** (Ethernet or Wi-Fi).
- (Optional) A **public IP** or domain name if you plan to host web services.

---

## 🧩 Step 1: Create a New Virtual Switch

Open **Hyper-V Manager → Virtual Switch Manager** and choose **New virtual network switch**.

You have two options:

### 🔹 Option A: Default Switch (NAT)
- Usually created automatically by Hyper-V.
- Provides internet via NAT from the host machine.
- IP range: `172.x.x.x`
- Simple and works out-of-the-box (best for testing and local environments).

No setup required — just attach your VM to **Default Switch**.

---

### 🔹 Option B: External Switch (Bridged to LAN)
- Connects your VM directly to the same physical network as your host.
- Your VM gets an IP in the same range as your host (e.g., `192.168.1.x`).
- Required if you want to SSH into the VM from other devices or the internet.

**To create:**
1. In Hyper-V Manager → *Virtual Switch Manager* → *New Virtual Switch → External*.
2. Choose your physical adapter (Ethernet or Wi-Fi).
3. Check ✅ “**Allow management OS to share this network adapter**”.
4. Name it (e.g., `ExternalSwitch`).
5. Click **Apply → OK**.

---

## 🧱 Step 2: Create the Ubuntu Server VM

1. Open **Hyper-V Manager** → *New → Virtual Machine*.
2. Follow the wizard:
   - **Generation**: Generation 2.
   - **Memory**: 2 GB or more.
   - **Network**: Attach to your chosen switch (`Default Switch` or `ExternalSwitch`).
   - **Virtual hard disk**: 20 GB or more.
   - **ISO image**: Browse to your downloaded Ubuntu Server ISO.

3. Finish and **Start** the VM.

---

## ⚙️ Step 3: Ubuntu Server Installation (Network Setup)

During installation, Ubuntu will ask for **network configuration**.

### ✅ If using External Switch (LAN IP)
Use manual configuration:
Subnet: 192.168.1.0/24
Address: 192.168.1.250
Gateway: 192.168.1.1
Name servers: 8.8.8.8, 1.1.1.1

## 🌐 4. Configure Network During Ubuntu Installation

### If using **External Switch**
Use **Manual** network config:

| Field        | Value               |
|--------------|---------------------|
| Subnet       | `192.168.1.0/24`    |
| Address      | `192.168.1.250/24`  |
| Gateway      | `192.168.1.1`       |
| Name servers | `8.8.8.8, 1.1.1.1`  |
| Proxy        | *(leave blank)*     |

### If using **Default Switch (Not)**
Choose **Automatic (DHCP)** — internet works instantly.

Continue installation normally.

---

## 🧾 5. Verify Network After Installation

After first login:

```bash
ip a
ip route
ping 8.8.8.8
ping google.com
```


## 🔧 6. Install Laravel Stack
Update & Install Dependencies
```bash
sudo apt update && sudo apt upgrade -y
```
```bash
sudo apt install -y nginx mysql-server php php-fpm php-mysql php-xml php-mbstring php-bcmath php-zip php-curl git unzip composer
```

## 📦 7. Deploy Laravel App
### 🛠️ 3. Fix permissions

#### Option A — Temporarily use sudo:
```bash
sudo git clone https://eatlbd.net/edutube/edutube-ims-backend.git
```

#### Option B — Give ownership to your user permanently:
```bash
sudo chown -R $USER:$USER /var/www/html
cd /var/www/html
```
#### Choose Option A with Deployment Steps:
```bash
cd /var/www/html/
sudo git clone https://github.com/your-repo/your-laravel-app.git app
cd app
composer install --no-dev --optimize-autoloader
cp .env.example .env
php artisan key:generate
```

Update .env:

```bash
APP_ENV=production
APP_DEBUG=false
APP_URL=http://your-domain-or-ip
DB_HOST=localhost
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=secret
```

## ⚙️ 8. Configure Nginx

Create a site config:

```bash
sudo nano /etc/nginx/sites-available/laravel
sudo nano /etc/nginx/sites-available/{project name}
```

### 🅰️ Without Domain (Access via IP)

```bash
server {
    listen 82;
    server_name _;

    root /var/www/html/app/public;
    index index.php index.html index.htm index.nginx-debian.html;

##    add_header X-Frame-Options "SAMEORIGIN";
##    add_header X-Content-Type-Options "nosniff";

    client_max_body_size 25M;

    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_send_timeout 300;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
##        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
##        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
For Advanced

```bash
server {
    listen 80;
    server_name your-domain.com 192.168.1.200;   # add IPs / hostnames you serve

    root /var/www/html/your-app/public;
    index index.php index.html;

    # Security & performance headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate" always;

    # Global limits (apply to whole server block)
    client_max_body_size 25M;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # gzip (do not change colors unless testing)
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Static assets — let Nginx serve them directly
    location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|svg|woff2?|ttf|eot)$ {
        try_files $uri =404;
        expires 7d;
        access_log off;
        add_header Cache-Control "public";
    }

    # Laravel front controller
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM via FastCGI
    location ~ \.php$ {
        # security: do not allow direct access outside root
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi_params;

        # Use realpath_root (resolves symlinks)
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;

        # Socket - adjust for your php version
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;

        fastcgi_index index.php;

        # FastCGI tuning (prevents 502/504 and header-too-large)
        fastcgi_read_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_buffer_size 32k;
        fastcgi_buffers 8 16k;
        fastcgi_busy_buffers_size 64k;
        fastcgi_temp_file_write_size 64k;

        # Security
        internal;        # optional: make .php handling internal if you want extra protection
    }

    # Deny access to hidden files and .env
    location ~ /\.(?!well-known).* {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Optional: health check endpoint
    location /healthz {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    access_log /var/log/nginx/your-app.access.log;
    error_log /var/log/nginx/your-app.error.log warn;
}

```

Enable & reload:

```bash
sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## 🔐 9. Add SSL
### 🅰️ Option 1: Self-Signed SSL (No Domain)
```bash
sudo mkdir /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/self.key -out /etc/nginx/ssl/self.crt
```


Edit /etc/nginx/sites-available/laravel:

```bash
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;
    root /var/www/html/app/public;
    ssl_certificate /etc/nginx/ssl/self.crt;
    ssl_certificate_key /etc/nginx/ssl/self.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Restart:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 🅱️ Option 2: Let’s Encrypt SSL (With Domain)
Prerequisite

A domain name pointing to your VM’s public IP

Ports 80 and 443 open (via router or firewall)

Install Certbot
```bash
sudo apt install -y certbot python3-certbot-nginx
```

Get Free SSL Certificate
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```


Follow prompts:

Enter email

Agree to terms

Choose “Redirect to HTTPS”

Test Auto-Renewal
```bash
sudo certbot renew --dry-run
```


Now your site is live at https://yourdomain.com 🔒

## 🔒 10. Set Permissions
```bash
sudo chown -R www-data:www-data /var/www/html/app
sudo chmod -R 775 /var/www/app/storage /var/www/html/app/bootstrap/cache
```

## 🧰 11. Enable Firewall
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

## 12. Configure Supervisor
Install supervisor
```bash
sudo apt install supervisor -y
```

### A. Queue Worker with Supervisor
```bash
sudo nano /etc/supervisor/conf.d/app-queue-worker.conf
```

Example:
```bash
[program:app-queue-worker]
process_name=%(program_name)s_%(process_num)02d
directory=/var/www/html/app
command=php artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/log/app-queue-worker.log
stopwaitsecs=3600
```

### B. Schedule Worker with Supervisor

```bash
sudo nano /etc/supervisor/conf.d/app-schedule-worker.conf
```

```bash
[program:app-schedule-worker]
process_name=%(program_name)s_%(process_num)02d
directory=/var/www/html/app
command=php artisan schedule:work
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/app-schedule-worker.log
stopwaitsecs=3600
```

### C. Apply any changes and manage Supervisor Worker:

To apply any Supervisor changes

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

To start a new worker

```bash
sudo supervisorctl start app-queue-worker:*
or
sudo supervisorctl start app-schedule-worker:*
```

To restart a worker

```bash
sudo supervisorctl restart app-queue-worker:*
or
sudo supervisorctl restart app-schedule-worker:*

```

To stop a worker

```bash
sudo supervisorctl stop app-queue-worker:*
or
sudo supervisorctl stop app-schedule-worker:*
```

To view status of Supervisor

```bash
sudo supervisorctl status
```

## 🧱 13. Configure MySQL
### 1️⃣ Installation & Status check

Inatall package
```bash
sudo apt install -y mysql-server php-mysql
```

To view the current status:
```bash
sudo systemctl status mysql
```

To start the service:
```bash
sudo systemctl start mysql
```

And enable on boot:
```bash
sudo systemctl enable mysql
```

### 2️⃣ Access MySQL as root

Ubuntu uses Unix socket authentication for the root MySQL user (no password needed). Run:
```bash
sudo mysql
```

### 3️⃣ (Optional but Recommended) Secure MySQL installation Run:
```bash
sudo mysql_secure_installation
```

### 4️⃣ Create a normal MySQL user for your app

Inside MySQL shell (sudo mysql):
```bash
CREATE DATABASE laravel;
CREATE USER 'laravel'@'localhost' IDENTIFIED BY 'StrongPassword123';
GRANT ALL PRIVILEGES ON laravel.* TO 'laravel'@'localhost';
FLUSH PRIVILEGES;
exit;
```

### 5️⃣ Test login with that user
```bash
mysql -u laravel -p
```

### 6️⃣ (Optional) Change root to use password login (not socket)

If you want to log in as root using mysql -u root -p instead of sudo mysql, run this inside MySQL shell:
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'YourStrongPassword';
FLUSH PRIVILEGES;
```

## 🧱 14. Configure phpMyAdmin 🌐

### 1️⃣ Install phpMyAdmin
```bash
sudo apt install phpmyadmin -y
```

### 2️⃣ Stop & remove Apache (if installed)
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo apt purge apache2 -y
sudo apt autoremove -y
```

### 3️⃣ Remove any conflicting phpMyAdmin Apache configs

Check if a symlink exists:
```bash
ls /etc/apache2/conf-enabled/
```

If you see phpmyadmin.conf, remove it:

```bash
sudo rm /etc/apache2/conf-enabled/phpmyadmin.conf
```

### 4️⃣ Check & Verify Nginx config

Check your Nginx config file:
```bash
sudo nano /etc/nginx/sites-available/phpmyadmin
```

```bash
server {
    listen 85;
    server_name _;

    root /usr/share/phpmyadmin;
    index index.php index.html index.htm;

    access_log /var/log/nginx/phpmyadmin_access.log;
    error_log /var/log/nginx/phpmyadmin_error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### 5️⃣ Enable site, test & reload Nginx
```bash
sudo ln -sf /etc/nginx/sites-available/phpmyadmin /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
or
sudo systemctl restart nginx
sudo systemctl status nginx
```

### 6️⃣ Open firewall
```bash
sudo ufw allow 85/tcp
sudo ufw reload
```
## 🌐 15. Configure Redis
### 1️⃣ Install Redis Server on Ubuntu

Run:
```bash
sudo apt update
sudo apt install redis-server -y
```

### 2️⃣ Enable Redis to run as a system service

Check if Redis is running:
```bash
sudo systemctl status redis
```

If not running, start:
```bash
sudo systemctl start redis
sudo systemctl enable redis
```

### 2️⃣ Configure Redis for production

Edit redis.conf:
```bash
sudo nano /etc/redis/redis.conf
```

Make sure these are set:

1. Turn on supervised mode:
```bash
supervised systemd
```

3. Bind only to localhost (secure):
```bash
bind 127.0.0.1
```

4. Protect with a password (recommended):

Uncomment and set:
```bash
requirepass YOUR_STRONG_PASSWORD
```

Save and exit: CTRL + X → Y → Enter

Restart Redis:

sudo systemctl restart redis

### 4️⃣ Verify Redis is working
```bash
redis-cli
```

Now authenticate (if password enabled):
```bash
AUTH YOUR_STRONG_PASSWORD
```

Test:
```bash
SET test "hello"
GET test
```

If it prints "hello" → Redis is working.

Exit:

exit

### 5️⃣ Setup Laravel to use Predis
1. Install Predis PHP library
In your Laravel project:
```bash
composer require predis/predis
```
2. Update .env

Add:
```bash
REDIS_CLIENT=predis
REDIS_PASSWORD=YOUR_STRONG_PASSWORD
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```
3. Clear Laravel cache
```bash
php artisan config:clear
php artisan cache:clear
php artisan optimize:clear
```
### 6️⃣ Set cache & session drivers to Redis (optional)

In .env:
```bash
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
```
