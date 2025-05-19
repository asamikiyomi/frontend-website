---
sidebar_position: 4
---

# CentOS & AlmaLinux Installation

This guide will walk you through the process of installing PhoenixPanel on CentOS Stream or AlmaLinux.

## Supported Versions

PhoenixPanel officially supports:
- CentOS Stream 8
- CentOS Stream 9
- AlmaLinux 8
- AlmaLinux 9
- Rocky Linux 8
- Rocky Linux 9

## System Requirements

| Type | Minimum | Recommended |
|------|---------|------------|
| CPU  | 1 core  | 2+ cores   |
| RAM  | 2GB     | 4GB+       |
| Disk | 10GB    | 20GB+      |

## Installation Steps

### Step 1: Update System Packages

First, make sure your system is up to date:

```bash
sudo dnf update -y
```

### Step 2: Install Required Repositories

```bash
# Enable EPEL and PowerTools/CRB repositories
sudo dnf install -y epel-release
sudo dnf install -y dnf-utils

# For CentOS/AlmaLinux/Rocky 8
sudo dnf config-manager --set-enabled powertools

# For CentOS/AlmaLinux/Rocky 9
sudo dnf config-manager --set-enabled crb
```

### Step 3: Install PHP 8.2

```bash
# Install Remi repository
sudo dnf install -y http://rpms.remirepo.net/enterprise/remi-release-$(rpm -E %rhel).rpm

# Enable PHP 8.2 repository
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y

# Install PHP and extensions
sudo dnf install -y php php-{common,cli,fpm,mysqlnd,zip,devel,curl,mbstring,bcmath,xml,gd,pdo,tokenizer,intl}
```

### Step 4: Install Additional Dependencies

```bash
# Install MariaDB, Nginx, and other requirements
sudo dnf install -y mariadb-server nginx tar unzip git redis
```

### Step 5: Install Composer

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

### Step 6: Configure MariaDB

Start MariaDB service:

```bash
sudo systemctl enable --now mariadb
```

Secure your MariaDB installation:

```bash
sudo mysql_secure_installation
```

Access MariaDB to create a database and user:

```bash
sudo mysql -u root -p
```

Run the following SQL commands:

```sql
CREATE USER 'phoenixpanel'@'localhost' IDENTIFIED BY 'YourSecurePassword';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'phoenixpanel'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Step 7: Download PhoenixPanel

```bash
sudo mkdir -p /var/www/phoenixpanel
cd /var/www/phoenixpanel
sudo curl -Lo panel.tar.gz https://github.com/phoenixpanel/panel/releases/latest/download/panel.tar.gz
sudo tar -xzvf panel.tar.gz
sudo chmod -R 755 storage/* bootstrap/cache/
```

### Step 8: Configure PhoenixPanel

Copy the example environment file:

```bash
sudo cp .env.example .env
```

Generate an application key:

```bash
sudo php artisan key:generate --force
```

Next we run these commands:

```bash
php artisan p:environment:setup (For panel setup)
php artisan p:environment:database (For database setup)
```

### Step 9: Setup Database

Run the database migrations:

```bash
sudo php artisan migrate --seed --force
```

Create the first admin user:

```bash
sudo php artisan p:user:make
```

### Step 10: Configure PHP-FPM

Edit the PHP-FPM configuration:

```bash
sudo nano /etc/php-fpm.d/www.conf
```

Find and change:

```
user = apache
group = apache
```

To:

```
user = nginx
group = nginx
```

Start and enable PHP-FPM:

```bash
sudo systemctl enable --now php-fpm
```

### Step 11: Configure Nginx

Create an Nginx configuration file:

```bash
sudo nano /etc/nginx/conf.d/phoenixpanel.conf
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

Enable and start Nginx:

```bash
sudo systemctl enable --now nginx
```

### Step 12: Set File Permissions

```bash
# Create the nginx user if it doesn't exist
sudo id -u nginx &>/dev/null || sudo useradd -r nginx

# Set correct permissions
sudo chown -R nginx:nginx /var/www/phoenixpanel/*
```

### Step 13: Configure SELinux

If SELinux is enabled (which is the default on CentOS/AlmaLinux), configure it to allow Nginx to access the required files:

```bash
sudo dnf install -y policycoreutils-python-utils
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/phoenixpanel/storage(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/phoenixpanel/bootstrap/cache(/.*)?"
sudo restorecon -R /var/www/phoenixpanel
```

### Step 14: Setup SSL (Recommended)

Install Certbot:

```bash
sudo dnf install -y certbot python3-certbot-nginx
```

Obtain an SSL certificate:

```bash
sudo certbot --nginx -d your.domain.com
```

### Step 15: Set Up Queue Worker

Create a systemd service:

```bash
sudo nano /etc/systemd/system/phoenixpanel.service
```

Add the following:

```
[Unit]
Description=PhoenixPanel Queue Worker
After=mariadb.service redis.service

[Service]
User=nginx
Group=nginx
Restart=always
ExecStart=/usr/bin/php /var/www/phoenixpanel/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now phoenixpanel.service
```

### Step 16: Configure Firewall

Allow HTTP and HTTPS traffic:

```bash
# For firewalld (default on CentOS/AlmaLinux)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## Troubleshooting

### Common CentOS/AlmaLinux Issues

#### SELinux Context Issues

If you encounter permission denied errors despite setting the correct file permissions:

```bash
# Check if SELinux is enforcing
getenforce

# If "Enforcing", you may need to set additional contexts
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/phoenixpanel(/.*)?"
sudo restorecon -R /var/www/phoenixpanel

# Or temporarily set SELinux to permissive mode for testing
sudo setenforce 0
```

#### PHP Version Issues

If you need to verify the installed PHP version:

```bash
php -v
```

If you need a different version:

```bash
# List available PHP module streams
sudo dnf module list php

# Reset and enable the desired version
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.1 -y  # For PHP 8.1 as an example
sudo dnf install -y php php-{common,cli,fpm,mysqlnd,zip,devel,curl,mbstring,bcmath,xml,gd}
```

#### Redis Issues

If Redis isn't starting:

```bash
# Check the Redis service status
sudo systemctl status redis

# Enable and start if not running
sudo systemctl enable --now redis
```

## Next Steps

Now that you have the panel installed on CentOS/AlmaLinux, you should:

1. [Configure panel settings](/docs/project/panel/configuration)
2. [Install Wings](/docs/project/wings/installing)
3. [Create your first server](/docs/project/servers/creation)