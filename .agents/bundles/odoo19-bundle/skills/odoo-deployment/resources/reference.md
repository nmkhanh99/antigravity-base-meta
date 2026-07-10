---
name: Odoo Deployment
description: Hướng dẫn toàn diện về deployment Odoo - Docker, systemd, nginx, PostgreSQL, backup, và production best practices
---

# Odoo Deployment Skill

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Docker Deployment](#docker-deployment)
3. [Systemd Service](#systemd-service)
4. [Nginx Reverse Proxy](#nginx-reverse-proxy)
5. [PostgreSQL Setup](#postgresql-setup)
6. [SSL/HTTPS Configuration](#sslhttps-configuration)
7. [Backup & Restore](#backup--restore)
8. [Monitoring & Logging](#monitoring--logging)
9. [Production Best Practices](#production-best-practices)

---

## Tổng Quan

### Deployment Options

1. **Docker** - Containerized deployment (Recommended)
2. **Systemd** - Native Linux service
3. **Cloud Platforms** - AWS, GCP, Azure
4. **Odoo.sh** - Official Odoo hosting

### Production Architecture

```
Internet
    ↓
[Nginx/Apache] (Reverse Proxy + SSL)
    ↓
[Odoo Server] (Port 8069)
    ↓
[PostgreSQL] (Port 5432)
```

---

## Docker Deployment

### 1. Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: odoo:19.0
    container_name: odoo18
    depends_on:
      - db
    ports:
      - "8069:8069"
      - "8072:8072"  # Longpolling
    volumes:
      - odoo-web-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=odoo_password
    restart: always
    networks:
      - odoo-network

  db:
    image: postgres:15
    container_name: odoo_db
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=odoo
      - POSTGRES_PASSWORD=odoo_password
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - odoo-db-data:/var/lib/postgresql/data/pgdata
    restart: always
    networks:
      - odoo-network
    ports:
      - "5432:5432"

  nginx:
    image: nginx:alpine
    container_name: odoo_nginx
    depends_on:
      - web
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
    restart: always
    networks:
      - odoo-network

volumes:
  odoo-web-data:
  odoo-db-data:

networks:
  odoo-network:
    driver: bridge
```

### 2. Odoo Configuration File

```ini
# config/odoo.conf
[options]
# Server
addons_path = /mnt/extra-addons,/usr/lib/python3/dist-packages/odoo/addons
admin_passwd = strong_admin_password
http_port = 8069
longpolling_port = 8072

# Database
db_host = db
db_port = 5432
db_user = odoo
db_password = odoo_password
db_maxconn = 64
db_template = template0

# Multiprocessing
workers = 4
max_cron_threads = 2
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200

# Logging
log_level = info
log_handler = :INFO
logfile = /var/log/odoo/odoo.log

# Security
list_db = False
proxy_mode = True

# Performance
unaccent = True
```

### 3. Custom Dockerfile

```dockerfile
# Dockerfile
FROM odoo:19.0

USER root

# Install additional dependencies
RUN apt-get update && apt-get install -y \
    python3-pip \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
COPY requirements.txt /tmp/
RUN pip3 install -r /tmp/requirements.txt

# Copy custom addons
COPY ./addons /mnt/extra-addons

# Set permissions
RUN chown -R odoo:odoo /mnt/extra-addons

USER odoo

# Expose ports
EXPOSE 8069 8072

# Start Odoo
CMD ["odoo"]
```

### 4. Docker Commands

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f web

# Stop services
docker-compose down

# Restart Odoo
docker-compose restart web

# Execute commands in container
docker-compose exec web odoo shell -d production

# Backup database
docker-compose exec db pg_dump -U odoo production > backup.sql

# Update Odoo
docker-compose pull
docker-compose up -d
```

---

## Systemd Service

### 1. Install Odoo from Source

```bash
#!/bin/bash
# install_odoo.sh

# Create odoo user
sudo useradd -m -d /opt/odoo -U -r -s /bin/bash odoo

# Install dependencies
sudo apt-get update
sudo apt-get install -y \
    python3-pip \
    python3-dev \
    python3-venv \
    libxml2-dev \
    libxslt1-dev \
    libevent-dev \
    libsasl2-dev \
    libldap2-dev \
    libpq-dev \
    libjpeg-dev \
    xfonts-75dpi \
    xfonts-base \
    wkhtmltopdf \
    postgresql \
    postgresql-client

# Clone Odoo
sudo su - odoo -s /bin/bash
cd /opt/odoo
git clone https://github.com/odoo/odoo.git --depth 1 --branch 19.0 odoo18

# Create virtual environment
python3 -m venv odoo-venv
source odoo-venv/bin/activate

# Install Python dependencies
pip3 install wheel
pip3 install -r odoo18/requirements.txt

# Create directories
mkdir -p /opt/odoo/custom-addons
mkdir -p /var/log/odoo
sudo chown odoo:odoo /var/log/odoo
```

### 2. Systemd Service File

```ini
# /etc/systemd/system/odoo.service
[Unit]
Description=Odoo 19
Documentation=https://www.odoo.com/documentation/19.0
After=network.target postgresql.service

[Service]
Type=simple
User=odoo
Group=odoo
SyslogIdentifier=odoo

Environment="PATH=/opt/odoo/odoo-venv/bin"
ExecStart=/opt/odoo/odoo-venv/bin/python3 /opt/odoo/odoo18/odoo-bin -c /etc/odoo/odoo.conf

# Process management
Restart=always
RestartSec=3

# Security
PrivateTmp=true
NoNewPrivileges=true

# Resource limits
LimitNOFILE=65535
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

### 3. Systemd Commands

```bash
# Enable service
sudo systemctl enable odoo.service

# Start service
sudo systemctl start odoo.service

# Check status
sudo systemctl status odoo.service

# View logs
sudo journalctl -u odoo.service -f

# Restart service
sudo systemctl restart odoo.service

# Stop service
sudo systemctl stop odoo.service
```

---

## Nginx Reverse Proxy

### 1. Nginx Configuration

```nginx
# /etc/nginx/sites-available/odoo
upstream odoo {
    server 127.0.0.1:8069;
}

upstream odoochat {
    server 127.0.0.1:8072;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    
    # Let's Encrypt validation
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    
    # SSL certificates
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Proxy settings
    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    
    # Log files
    access_log /var/log/nginx/odoo-access.log;
    error_log /var/log/nginx/odoo-error.log;
    
    # File upload size
    client_max_body_size 100M;
    
    # Gzip compression
    gzip on;
    gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;
    gzip_min_length 1000;
    
    # Odoo web requests
    location / {
        proxy_pass http://odoo;
        proxy_redirect off;
    }
    
    # Odoo longpolling
    location /longpolling {
        proxy_pass http://odoochat;
    }
    
    # Static files
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo;
    }
    
    # Deny access to sensitive files
    location ~ /\.ht {
        deny all;
    }
}
```

### 2. Enable Nginx Site

```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## PostgreSQL Setup

### 1. PostgreSQL Installation

```bash
# Install PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# Create Odoo user
sudo su - postgres
createuser -s odoo
psql
ALTER USER odoo WITH PASSWORD 'strong_password';
\q
exit
```

### 2. PostgreSQL Configuration

```bash
# /etc/postgresql/15/main/postgresql.conf

# Connection settings
listen_addresses = 'localhost'
port = 5432
max_connections = 100

# Memory settings
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
work_mem = 16MB

# WAL settings
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# Query tuning
random_page_cost = 1.1
effective_io_concurrency = 200

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_min_duration_statement = 1000
```

### 3. PostgreSQL Optimization

```sql
-- Create indexes for better performance
CREATE INDEX IF NOT EXISTS ir_attachment_res_idx 
ON ir_attachment (res_model, res_id);

-- Vacuum and analyze
VACUUM ANALYZE;

-- Check database size
SELECT 
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
```

---

## SSL/HTTPS Configuration

### 1. Let's Encrypt SSL

```bash
# Install certbot
sudo apt-get install certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renewal
sudo certbot renew --dry-run

# Add cron job for auto-renewal
sudo crontab -e
0 0 * * * /usr/bin/certbot renew --quiet
```

### 2. Manual SSL Certificate

```bash
# Generate private key
openssl genrsa -out privkey.pem 2048

# Generate CSR
openssl req -new -key privkey.pem -out cert.csr

# Generate self-signed certificate (for testing)
openssl x509 -req -days 365 -in cert.csr -signkey privkey.pem -out fullchain.pem

# Copy to nginx directory
sudo cp fullchain.pem /etc/nginx/ssl/
sudo cp privkey.pem /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/privkey.pem
```

---

## Backup & Restore

### 1. Database Backup Script

```bash
#!/bin/bash
# backup_odoo.sh

# Configuration
BACKUP_DIR="/opt/odoo/backups"
DB_NAME="production"
DB_USER="odoo"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
pg_dump -U $DB_USER -Fc $DB_NAME > $BACKUP_DIR/db_${DB_NAME}_${TIMESTAMP}.dump

# Backup filestore
tar -czf $BACKUP_DIR/filestore_${DB_NAME}_${TIMESTAMP}.tar.gz \
    /var/lib/odoo/.local/share/Odoo/filestore/$DB_NAME

# Remove old backups
find $BACKUP_DIR -name "db_*.dump" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "filestore_*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Upload to S3 (optional)
# aws s3 cp $BACKUP_DIR/db_${DB_NAME}_${TIMESTAMP}.dump s3://your-bucket/odoo-backups/

echo "Backup completed: $TIMESTAMP"
```

### 2. Restore Script

```bash
#!/bin/bash
# restore_odoo.sh

DB_NAME="production"
DB_USER="odoo"
BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

# Stop Odoo
sudo systemctl stop odoo

# Drop existing database
dropdb -U $DB_USER $DB_NAME

# Create new database
createdb -U $DB_USER -O $DB_USER $DB_NAME

# Restore database
pg_restore -U $DB_USER -d $DB_NAME $BACKUP_FILE

# Start Odoo
sudo systemctl start odoo

echo "Restore completed"
```

### 3. Automated Backup Cron

```bash
# Add to crontab
sudo crontab -e

# Daily backup at 2 AM
0 2 * * * /opt/odoo/scripts/backup_odoo.sh >> /var/log/odoo/backup.log 2>&1

# Weekly full backup on Sunday
0 3 * * 0 /opt/odoo/scripts/full_backup.sh >> /var/log/odoo/backup.log 2>&1
```

---

## Monitoring & Logging

### 1. Log Rotation

```bash
# /etc/logrotate.d/odoo
/var/log/odoo/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 odoo odoo
    sharedscripts
    postrotate
        systemctl reload odoo > /dev/null 2>&1 || true
    endscript
}
```

### 2. Monitoring Script

```python
#!/usr/bin/env python3
# monitor_odoo.py

import psutil
import requests
import smtplib
from email.mime.text import MIMEText

def check_odoo_process():
    """Check if Odoo process is running"""
    for proc in psutil.process_iter(['name']):
        if 'odoo' in proc.info['name'].lower():
            return True
    return False

def check_odoo_http():
    """Check if Odoo responds to HTTP"""
    try:
        response = requests.get('http://localhost:8069/web/health', timeout=5)
        return response.status_code == 200
    except:
        return False

def check_database():
    """Check database connection"""
    import psycopg2
    try:
        conn = psycopg2.connect(
            host='localhost',
            database='production',
            user='odoo',
            password='odoo_password'
        )
        conn.close()
        return True
    except:
        return False

def send_alert(message):
    """Send email alert"""
    msg = MIMEText(message)
    msg['Subject'] = 'Odoo Alert'
    msg['From'] = 'monitor@yourdomain.com'
    msg['To'] = 'admin@yourdomain.com'
    
    s = smtplib.SMTP('localhost')
    s.send_message(msg)
    s.quit()

if __name__ == '__main__':
    if not check_odoo_process():
        send_alert('Odoo process is not running!')
    
    if not check_odoo_http():
        send_alert('Odoo HTTP is not responding!')
    
    if not check_database():
        send_alert('Database connection failed!')
```

### 3. Prometheus Monitoring

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'odoo'
    static_configs:
      - targets: ['localhost:8069']
    metrics_path: '/metrics'
```

---

## Production Best Practices

### 1. Security Checklist

```bash
# Disable database manager
# In odoo.conf:
list_db = False

# Use strong admin password
admin_passwd = $(openssl rand -base64 32)

# Enable proxy mode
proxy_mode = True

# Firewall rules
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Disable unnecessary services
sudo systemctl disable bluetooth
sudo systemctl disable cups
```

### 2. Performance Tuning

```ini
# odoo.conf - Production settings
[options]
# Workers (2 * CPU cores + 1)
workers = 9
max_cron_threads = 2

# Memory limits (in bytes)
limit_memory_hard = 2684354560  # 2.5GB
limit_memory_soft = 2147483648  # 2GB

# Request limits
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200

# Database pool
db_maxconn = 64

# Disable auto-reload in production
dev_mode = False
```

### 3. Health Check Endpoint

```python
# addons/health_check/controllers/main.py
from odoo import http

class HealthCheck(http.Controller):
    
    @http.route('/web/health', type='http', auth='none')
    def health_check(self):
        """Health check endpoint for load balancers"""
        try:
            # Check database
            http.request.env.cr.execute('SELECT 1')
            
            return http.request.make_response(
                'OK',
                headers=[('Content-Type', 'text/plain')]
            )
        except:
            return http.request.make_response(
                'ERROR',
                status=503,
                headers=[('Content-Type', 'text/plain')]
            )
```

---

## 🎯 Deployment Checklist

Pre-deployment:
- [ ] Test all modules in staging environment
- [ ] Backup current database and filestore
- [ ] Review and update odoo.conf
- [ ] Check SSL certificate validity
- [ ] Verify database indexes
- [ ] Test backup and restore procedures

Deployment:
- [ ] Enable maintenance mode
- [ ] Stop Odoo service
- [ ] Update code/modules
- [ ] Run database migrations
- [ ] Clear cache
- [ ] Start Odoo service
- [ ] Verify all services are running
- [ ] Disable maintenance mode

Post-deployment:
- [ ] Monitor logs for errors
- [ ] Check application performance
- [ ] Verify critical workflows
- [ ] Update documentation
- [ ] Notify stakeholders

---

## 📚 Tài Liệu Tham Khảo

- [Odoo Deployment Guide](https://www.odoo.com/documentation/19.0/administration/install.html)
- [Docker Documentation](https://docs.docker.com/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [PostgreSQL Administration](https://www.postgresql.org/docs/current/admin.html)
- [Let's Encrypt](https://letsencrypt.org/getting-started/)
