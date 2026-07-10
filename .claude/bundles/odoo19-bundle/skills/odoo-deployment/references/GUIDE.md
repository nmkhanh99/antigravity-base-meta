---
name: Odoo 19 Deployment Reference Guide
description: Tài liệu kỹ thuật đầy đủ về deployment Odoo 19 - Docker, systemd, nginx, PostgreSQL, backup, và production best practices
---

# Odoo 19 Deployment — Reference Guide

## Mục Lục

1. [Docker Deployment](#docker-deployment)
2. [Systemd Service](#systemd-service)
3. [Nginx Reverse Proxy](#nginx-reverse-proxy)
4. [PostgreSQL Setup](#postgresql-setup)
5. [SSL/HTTPS Configuration](#sslhttps-configuration)
6. [Backup & Restore](#backup--restore)
7. [Monitoring & Logging](#monitoring--logging)
8. [Production Best Practices](#production-best-practices)
9. [Deployment Checklist](#deployment-checklist)

---

## Docker Deployment

### docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: odoo:19.0
    container_name: odoo19
    depends_on:
      - db
    ports:
      - "127.0.0.1:8069:8069"   # Bind localhost only — nginx làm reverse proxy
      - "127.0.0.1:8072:8072"   # Longpolling/Websocket
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
    image: postgres:16
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
    # KHÔNG expose port 5432 ra ngoài trên production

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

### config/odoo.conf (Production)

```ini
[options]
# Server
addons_path = /mnt/extra-addons,/usr/lib/python3/dist-packages/odoo/addons
admin_passwd = CHANGE_ME_STRONG_RANDOM_PASSWORD
http_port = 8069
longpolling_port = 8072

# Database
db_host = db
db_port = 5432
db_user = odoo
db_password = odoo_password
db_maxconn = 64
db_template = template0

# Multiprocessing (2 * CPU cores + 1)
workers = 4
max_cron_threads = 2
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200

# Logging
log_level = warn
logfile = /var/log/odoo/odoo.log

# Security
list_db = False
proxy_mode = True
```

### Custom Dockerfile

```dockerfile
FROM odoo:19.0

USER root

RUN apt-get update && apt-get install -y \
    python3-pip \
    git \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /tmp/
RUN pip3 install -r /tmp/requirements.txt

COPY ./addons /mnt/extra-addons
RUN chown -R odoo:odoo /mnt/extra-addons

USER odoo

EXPOSE 8069 8072

CMD ["odoo"]
```

### Docker Commands

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f web

# Restart Odoo only
docker-compose restart web

# Execute odoo shell
docker-compose exec web odoo shell -d production

# Backup database từ container
docker-compose exec db pg_dump -U odoo production > backup_$(date +%Y%m%d).sql

# Update image
docker-compose pull && docker-compose up -d

# Stop all
docker-compose down
```

---

## Systemd Service

### Cài đặt từ source

```bash
#!/bin/bash
# install_odoo19.sh

# Tạo user odoo
sudo useradd -m -d /opt/odoo -U -r -s /bin/bash odoo

# Cài dependencies
sudo apt-get update
sudo apt-get install -y \
    python3-pip python3-dev python3-venv \
    libxml2-dev libxslt1-dev libevent-dev \
    libsasl2-dev libldap2-dev libpq-dev libjpeg-dev \
    xfonts-75dpi xfonts-base wkhtmltopdf \
    postgresql postgresql-client

# Clone Odoo 19
sudo su - odoo -s /bin/bash -c "
    git clone https://github.com/odoo/odoo.git --depth 1 --branch 19.0 /opt/odoo/odoo19
    python3 -m venv /opt/odoo/odoo-venv
    source /opt/odoo/odoo-venv/bin/activate
    pip3 install wheel
    pip3 install -r /opt/odoo/odoo19/requirements.txt
    mkdir -p /opt/odoo/custom-addons
"

# Log directory
sudo mkdir -p /var/log/odoo
sudo chown odoo:odoo /var/log/odoo
```

### /etc/systemd/system/odoo.service

```ini
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
ExecStart=/opt/odoo/odoo-venv/bin/python3 /opt/odoo/odoo19/odoo-bin -c /etc/odoo/odoo.conf

Restart=always
RestartSec=3

# Security hardening
PrivateTmp=true
NoNewPrivileges=true

# Resource limits
LimitNOFILE=65535
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

### Systemd Commands

```bash
sudo systemctl daemon-reload
sudo systemctl enable odoo.service
sudo systemctl start odoo.service
sudo systemctl status odoo.service

# View logs
sudo journalctl -u odoo.service -f

# Restart
sudo systemctl restart odoo.service
```

---

## Nginx Reverse Proxy

### /etc/nginx/sites-available/odoo

```nginx
upstream odoo {
    server 127.0.0.1:8069;
}

upstream odoochat {
    server 127.0.0.1:8072;
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

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

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

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

    access_log /var/log/nginx/odoo-access.log;
    error_log /var/log/nginx/odoo-error.log;

    client_max_body_size 100M;

    # Gzip
    gzip on;
    gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;
    gzip_min_length 1000;

    # Odoo web
    location / {
        proxy_pass http://odoo;
        proxy_redirect off;
    }

    # Websocket / Longpolling (Odoo 19 dùng /websocket thay vì /longpolling)
    location /websocket {
        proxy_pass http://odoochat;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Legacy longpolling endpoint
    location /longpolling {
        proxy_pass http://odoochat;
    }

    # Static files cache
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### Enable site

```bash
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## PostgreSQL Setup

### Tạo user và database

```bash
sudo su - postgres
createuser -s odoo
psql -c "ALTER USER odoo WITH PASSWORD 'strong_password';"
exit
```

### /etc/postgresql/16/main/postgresql.conf (tuning)

```ini
# Connections
max_connections = 100

# Memory (điều chỉnh theo RAM server)
shared_buffers = 256MB          # 25% RAM
effective_cache_size = 1GB      # 75% RAM
maintenance_work_mem = 64MB
work_mem = 16MB

# WAL
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# Query tuning
random_page_cost = 1.1
effective_io_concurrency = 200

# Logging
log_min_duration_statement = 1000   # Log queries > 1s
```

### Tối ưu indexes

```sql
-- Index cho ir.attachment (tìm kiếm attachment theo record)
CREATE INDEX IF NOT EXISTS ir_attachment_res_idx
ON ir_attachment (res_model, res_id);

-- Vacuum và analyze
VACUUM ANALYZE;

-- Kiểm tra kích thước database
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

---

## SSL/HTTPS Configuration

### Let's Encrypt (Khuyến nghị)

```bash
sudo apt-get install certbot python3-certbot-nginx

# Cấp certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Test auto-renewal
sudo certbot renew --dry-run

# Cron tự động gia hạn
echo "0 0 * * * /usr/bin/certbot renew --quiet" | sudo crontab -
```

### Self-signed (Chỉ dùng cho testing)

```bash
openssl genrsa -out privkey.pem 2048
openssl req -new -key privkey.pem -out cert.csr
openssl x509 -req -days 365 -in cert.csr -signkey privkey.pem -out fullchain.pem

sudo mkdir -p /etc/nginx/ssl
sudo cp fullchain.pem /etc/nginx/ssl/
sudo cp privkey.pem /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/privkey.pem
```

---

## Backup & Restore

### backup_odoo.sh

```bash
#!/bin/bash
BACKUP_DIR="/opt/odoo/backups"
DB_NAME="production"
DB_USER="odoo"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

# Backup database (custom format — nhỏ hơn, faster restore)
pg_dump -U $DB_USER -Fc $DB_NAME > $BACKUP_DIR/db_${DB_NAME}_${TIMESTAMP}.dump

# Backup filestore
tar -czf $BACKUP_DIR/filestore_${DB_NAME}_${TIMESTAMP}.tar.gz \
    /var/lib/odoo/.local/share/Odoo/filestore/$DB_NAME

# Xóa backup cũ
find $BACKUP_DIR -name "db_*.dump" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "filestore_*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Upload lên S3 (tuỳ chọn)
# aws s3 cp $BACKUP_DIR/db_${DB_NAME}_${TIMESTAMP}.dump s3://your-bucket/odoo-backups/

echo "Backup completed: $TIMESTAMP"
```

### restore_odoo.sh

```bash
#!/bin/bash
DB_NAME="production"
DB_USER="odoo"
BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file.dump>"
    exit 1
fi

# Stop Odoo
sudo systemctl stop odoo

# Recreate database
dropdb -U $DB_USER $DB_NAME
createdb -U $DB_USER -O $DB_USER $DB_NAME

# Restore
pg_restore -U $DB_USER -d $DB_NAME $BACKUP_FILE

# Start Odoo
sudo systemctl start odoo
echo "Restore completed"
```

### Cron backup

```bash
# Daily backup 2 AM
0 2 * * * /opt/odoo/scripts/backup_odoo.sh >> /var/log/odoo/backup.log 2>&1
```

---

## Monitoring & Logging

### /etc/logrotate.d/odoo

```
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

### Health Check Controller (Odoo 19)

```python
# addons/health_check/controllers/main.py
from odoo import http
from odoo.http import request


class HealthCheck(http.Controller):

    @http.route('/web/health', type='http', auth='none')
    def health_check(self):
        """Health check endpoint cho load balancer / monitoring."""
        try:
            request.env.cr.execute('SELECT 1')
            return request.make_response(
                'OK',
                headers=[('Content-Type', 'text/plain')]
            )
        except Exception:
            return request.make_response(
                'ERROR',
                status=503,
                headers=[('Content-Type', 'text/plain')]
            )
```

### Monitor script

```python
#!/usr/bin/env python3
# monitor_odoo.py
import psutil
import requests


def check_odoo_process():
    return any('odoo' in p.name().lower() for p in psutil.process_iter(['name']))


def check_odoo_http():
    try:
        r = requests.get('http://localhost:8069/web/health', timeout=5)
        return r.status_code == 200
    except Exception:
        return False


def check_database():
    import psycopg2
    try:
        conn = psycopg2.connect(host='localhost', database='production', user='odoo', password='odoo_password')
        conn.close()
        return True
    except Exception:
        return False


if __name__ == '__main__':
    alerts = []
    if not check_odoo_process():
        alerts.append('Odoo process is not running!')
    if not check_odoo_http():
        alerts.append('Odoo HTTP is not responding!')
    if not check_database():
        alerts.append('Database connection failed!')
    for a in alerts:
        print(f'ALERT: {a}')
```

---

## Production Best Practices

### Security Checklist

```bash
# Tắt database manager
# odoo.conf: list_db = False

# Firewall — chỉ mở port cần thiết
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 8069/tcp   # KHÔNG expose trực tiếp
sudo ufw deny 5432/tcp   # KHÔNG expose PostgreSQL
sudo ufw enable

# Admin password random
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

### Performance Tuning (odoo.conf)

```ini
[options]
# Workers = 2 * CPU cores + 1
workers = 9
max_cron_threads = 2

# Memory (bytes)
limit_memory_hard = 2684354560   # 2.5 GB
limit_memory_soft = 2147483648   # 2.0 GB

# Request limits
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200

# DB pool
db_maxconn = 64

# Production — tắt dev mode
dev_mode =
```

---

## Deployment Checklist

### Pre-deployment
- [ ] Test toàn bộ module trên staging
- [ ] Backup database và filestore hiện tại
- [ ] Review và cập nhật odoo.conf
- [ ] Kiểm tra SSL certificate còn hiệu lực
- [ ] Verify database indexes
- [ ] Test quy trình backup/restore

### Deployment
- [ ] Enable maintenance mode
- [ ] Stop Odoo service
- [ ] Update code/modules
- [ ] Chạy database migrations: `odoo-bin -u all -d production --stop-after-init`
- [ ] Clear cache
- [ ] Start Odoo service
- [ ] Verify tất cả service đang chạy
- [ ] Disable maintenance mode

### Post-deployment
- [ ] Monitor logs 15-30 phút sau deploy
- [ ] Kiểm tra performance (response time)
- [ ] Verify các workflow quan trọng
- [ ] Cập nhật documentation
- [ ] Thông báo stakeholders

---

## Tài Liệu Tham Khảo

- [Odoo 19 Installation Guide](https://www.odoo.com/documentation/19.0/administration/install.html)
- [Docker Hub - Odoo](https://hub.docker.com/_/odoo)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Getting Started](https://letsencrypt.org/getting-started/)
- [PostgreSQL 16 Administration](https://www.postgresql.org/docs/16/admin.html)
- [Certbot Instructions](https://certbot.eff.org/instructions)
