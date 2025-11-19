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
    listen 80;
    server_name _;
    root /var/www/html/app/public;
    index index.php index.html;

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
[program:app-queue-worker.conf]
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

## 🧱 1. Check MySQL service status
sudo systemctl status mysql

If you see:

Active: active (running)


then MySQL is running fine.

If not, start it:
```bash
sudo systemctl start mysql
```

And enable on boot:
```bash
sudo systemctl enable mysql
```bash

## 🧠 2. Access MySQL as root

Ubuntu uses Unix socket authentication for the root MySQL user (no password needed). Run:
```bash
sudo mysql
```

## 🔐 3. (Optional but Recommended) Secure MySQL installation Run:
```bash
sudo mysql_secure_installation
```

## 👤 4. Create a normal MySQL user for your Laravel app

Inside MySQL shell (sudo mysql):
```bash
CREATE DATABASE laravel;
CREATE USER 'laravel'@'localhost' IDENTIFIED BY 'StrongPassword123';
GRANT ALL PRIVILEGES ON laravel.* TO 'laravel'@'localhost';
FLUSH PRIVILEGES;
exit;
```

## 🧰 6. (Optional) Change root to use password login (not socket)

If you want to log in as root using mysql -u root -p instead of sudo mysql, run this inside MySQL shell:
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'YourStrongPassword';
FLUSH PRIVILEGES;
```

🧱 1. Install phpMyAdmin
sudo apt update
sudo apt install phpmyadmin -y


🌐 2. Verify phpMyAdmin Nginx configuration

Check your Nginx config file:

sudo nano /etc/nginx/sites-available/phpmyadmin


Make sure it looks like this (serving on port 85):

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
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}


Then test and reload Nginx:

sudo nginx -t
sudo systemctl reload nginx


Now visit:

http://<your-server-ip>:85


Example:

http://192.168.1.250:85


✅ You should see the phpMyAdmin login page.


2️⃣ Stop & remove Apache (if installed)
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo apt purge apache2 -y
sudo apt autoremove -y

3️⃣ Remove any conflicting phpMyAdmin Apache configs

Check if a symlink exists:

ls /etc/apache2/conf-enabled/


If you see phpmyadmin.conf, remove it:

sudo rm /etc/apache2/conf-enabled/phpmyadmin.conf

4️⃣ Check Nginx config

Test all configs for syntax errors:

sudo nginx -t


If it says OK, proceed

5️⃣ Enable site & restart Nginx
sudo ln -sf /etc/nginx/sites-available/phpmyadmin /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx


✅ It should now show active (running)

6️⃣ Open firewall
sudo ufw allow 85/tcp
sudo ufw reload
