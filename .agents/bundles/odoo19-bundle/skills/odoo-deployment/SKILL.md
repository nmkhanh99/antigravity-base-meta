---
name: odoo-deployment
description: Hướng dẫn deployment Odoo 19 - Docker, systemd, nginx, PostgreSQL, backup, và production best practices. Use when the user asks to deploy, setup server, configure nginx, or manage Odoo 19 in production.
---

# Odoo 19 Deployment

## Goal
Giúp agent deploy và quản lý Odoo 19 trên production server: Docker, systemd, nginx reverse proxy, PostgreSQL, backup.

## When to use this skill
- "deploy", "production setup"
- "Docker", "nginx", "systemd"
- "backup", "restore"
- "server configuration", "odoo.conf"

## Instructions

### 1. odoo.conf (Production)
```ini
[options]
admin_passwd = strong_password_here
db_host = localhost
db_port = 5432
db_user = odoo
db_password = db_password
db_name = production_db
addons_path = /opt/odoo/addons,/opt/odoo/custom-addons
data_dir = /opt/odoo/data
logfile = /var/log/odoo/odoo.log
log_level = warn
workers = 4
max_cron_threads = 2
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_time_cpu = 600
limit_time_real = 1200
proxy_mode = True
```

### 2. Nginx Reverse Proxy
```nginx
upstream odoo {
    server 127.0.0.1:8069;
}
upstream odoochat {
    server 127.0.0.1:8072;
}
server {
    listen 443 ssl;
    server_name odoo.example.com;
    ssl_certificate /etc/ssl/certs/odoo.crt;
    ssl_certificate_key /etc/ssl/private/odoo.key;

    location / {
        proxy_pass http://odoo;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /websocket {
        proxy_pass http://odoochat;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 3. Docker Compose
```yaml
version: '3'
services:
  odoo:
    image: odoo:19
    ports: ["8069:8069"]
    volumes:
      - odoo-data:/var/lib/odoo
      - ./custom-addons:/mnt/extra-addons
      - ./odoo.conf:/etc/odoo/odoo.conf
    depends_on: [db]
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    volumes: [db-data:/var/lib/postgresql/data]
volumes:
  odoo-data:
  db-data:
```

### 4. Backup & Restore
```bash
# Backup
pg_dump -U odoo production_db > backup_$(date +%Y%m%d).sql
tar czf filestore_$(date +%Y%m%d).tar.gz /opt/odoo/data/filestore/

# Restore
psql -U odoo production_db < backup.sql
tar xzf filestore.tar.gz -C /opt/odoo/data/
```

## Constraints
- Production PHẢI dùng `workers > 0` (multi-process mode).
- PHẢI enable `proxy_mode = True` khi đứng sau reverse proxy.
- KHÔNG expose port 8069 trực tiếp ra internet.

## Best practices
- Dùng SSL/TLS cho production.
- Set `limit_memory_hard/soft` phù hợp RAM server.
- Đọc `resources/reference.md` cho systemd service, monitoring, scaling.
