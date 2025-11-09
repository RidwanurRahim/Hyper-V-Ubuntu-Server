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

## 🔁 12. (Optional) Queue Worker with Supervisor
```bash
sudo apt install supervisor -y
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Example:
```bash
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/app/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/html/app/storage/logs/worker.log
```

Apply:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```
