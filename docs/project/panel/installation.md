---
sidebar_position: 1
---

# Panel Installation

This guide provides an overview of installing PhoenixPanel. For OS-specific installation instructions, please select your operating system from the sidebar.

## Supported Operating Systems

PhoenixPanel officially supports the following operating systems:

| Operating System | Version | Status |
|-----------------|---------|--------|
| Ubuntu | 20.04 LTS | ✅ Fully Supported |
| Ubuntu | 22.04 LTS | ✅ Fully Supported |
| Debian | 11 (Bullseye) | ✅ Fully Supported |
| Debian | 12 (Bookworm) | ✅ Fully Supported |
| CentOS Stream | 8 | ✅ Fully Supported |
| CentOS Stream | 9 | ✅ Fully Supported |
| AlmaLinux | 8 | ✅ Fully Supported |
| AlmaLinux | 9 | ✅ Fully Supported |
| Rocky Linux | 8 | ✅ Fully Supported |
| Rocky Linux | 9 | ✅ Fully Supported |

## System Requirements

Ensure your system meets these requirements before installing PhoenixPanel:

| Component | Minimum | Recommended |
|-----------|---------|------------|
| CPU | 1 core | 2+ cores |
| RAM | 2GB | 4GB+ |
| Disk | 10GB | 20GB+ |
| Network | 100Mbps | 1Gbps+ |
| PHP | 8.1 | 8.2+ |
| Database | MySQL 8.0 / MariaDB 10.5 | MySQL 8.0 |
| Redis | 6.0 | 7.0+ |

## Quick Installation Reference

These are general steps for installing PhoenixPanel. For detailed, OS-specific instructions, please select your operating system from the sidebar.

## Dependency Installation

### Install PHP 8.2

First, we need to ensure we have the correct PHP version installed:

```bash
# Ubuntu 22.04
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install -y php8.2 php8.2-common php8.2-cli php8.2-gd php8.2-mysql php8.2-mbstring php8.2-bcmath php8.2-xml php8.2-curl php8.2-zip php8.2-fpm
```

### Install Composer

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

### Install MySQL

```bash
sudo apt install -y mysql-server
```

Configure MySQL:

```bash
sudo mysql_secure_installation
```

Create a database and user:

```bash
sudo mysql -u root -p
```

```sql
CREATE USER 'phoenixpanel'@'localhost' IDENTIFIED BY 'yourpassword';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'phoenixpanel'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Install Nginx

```bash
sudo apt install -y nginx
```

### Install Redis

```bash
sudo apt install -y redis-server
```

## Download Files

Download the latest panel files:

```bash
mkdir -p /var/www/phoenixpanel
cd /var/www/phoenixpanel
curl -Lo panel.tar.gz https://github.com/phoenixpanel/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
chmod -R 755 storage/* bootstrap/cache/
```

## Panel Setup

### Configure Environment

Copy the default environment file:

```bash
cp .env.example .env
```

Generate an application key:

```bash
php artisan key:generate --force
```

Next run these commands:

```bash
php artisan p:environment:setup (For panel setup)
php artisan p:environment:database (For database setup)
```

### Setup Database

Run the database migrations and create the first user:

```bash
php artisan migrate --seed --force
php artisan p:user:make
```

### Set Permissions

Make sure the web server has the correct permissions:

```bash
sudo chown -R www-data:www-data /var/www/phoenixpanel/*
```

### Configure Nginx

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/phoenixpanel.conf
```

Add the following configuration:

```nginx
server {
    # Replace the example <domain> with your domain name or IP address
    listen 80;
    server_name <domain>;

    root /var/www/phoenixpanel/public;
    index index.html index.htm index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/phoenixpanel.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
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
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/phoenixpanel.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### SSL Configuration (Optional but Recommended)

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Obtain an SSL certificate:

```bash
sudo certbot --nginx -d your.domain.com
```

### Configure Queue Worker

PhoenixPanel uses queues for handling tasks. Create a systemd service:

```bash
sudo nano /etc/systemd/system/phoenixpanel.service
```

Add the following configuration:

```
[Unit]
Description=PhoenixPanel Queue Worker
After=mysql.service redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/phoenixpanel/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Start and enable the service:

```bash
sudo systemctl enable --now phoenixpanel.service
```

## Updating PhoenixPanel

To update PhoenixPanel to the latest version, follow these steps:

```bash
cd /var/www/phoenixpanel
php artisan down
curl -L https://github.com/phoenixpanel/panel/releases/latest/download/panel.tar.gz | tar -xzv
chmod -R 755 storage/* bootstrap/cache/
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan view:clear
php artisan config:clear
sudo chown -R www-data:www-data /var/www/phoenixpanel/*
php artisan queue:restart
php artisan up
```

## Next Steps

Now that you have installed the panel, you need to:

1. [Configure settings](/docs/project/panel/configuration)
2. [Install Wings](/docs/project/wings/installing) on your game servers
3. [Create your first server](/docs/project/servers/creation)

## Troubleshooting

If you encounter any issues during installation, check the following:

1. Make sure all the required PHP extensions are installed
2. Check the logs at `storage/logs/laravel.log`
3. Ensure the permissions are set correctly
4. Verify that all the services (MySQL, Redis, etc.) are running
5. Make sure your `.env` file is properly configured

For more help, join our [Discord server](https://discord.gg/4EWAVyJY9z) or check the [Community forums](https://community.phoenixpanel.io).