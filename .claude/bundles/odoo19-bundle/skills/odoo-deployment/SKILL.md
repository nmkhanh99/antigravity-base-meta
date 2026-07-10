---
name: odoo-deployment
description: Hướng dẫn deployment Odoo 19 - Docker, systemd, nginx, PostgreSQL, backup, và production best practices. Use when the user asks to deploy, setup server, configure nginx, or manage Odoo 19 in production.
---

# Odoo 19 Deployment

## Goal
Giúp agent deploy và quản lý Odoo 19 trên production server: Docker Compose, systemd service, nginx reverse proxy, PostgreSQL, backup/restore và monitoring.

## When to use this skill
- "deploy", "production setup", "cài Odoo lên server"
- "Docker", "docker-compose", "nginx", "systemd"
- "backup", "restore", "khôi phục dữ liệu"
- "server configuration", "odoo.conf", "cấu hình production"
- "SSL", "HTTPS", "Let's Encrypt"
- "monitoring", "log rotation", "health check"

## Instructions

### Bước 1 — Chọn deployment method
Xác định với user:
- **Docker Compose** — Khuyến nghị cho hầu hết trường hợp (cô lập, dễ scale)
- **Systemd native** — Khi cần cài thẳng lên OS (bare-metal, VPS đơn giản)
- Chi tiết cấu hình xem `references/GUIDE.md` → "Docker Deployment" hoặc "Systemd Service"

### Bước 2 — Cấu hình odoo.conf
Các tham số bắt buộc cho production:
```ini
[options]
admin_passwd = <strong_random_password>
db_host = localhost          # hoặc tên service docker: db
db_port = 5432
db_user = odoo
db_password = <db_password>
addons_path = /opt/odoo/addons,/opt/odoo/custom-addons
data_dir = /opt/odoo/data
logfile = /var/log/odoo/odoo.log
log_level = warn
workers = 4                  # 2 * CPU cores + 1
max_cron_threads = 2
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_time_cpu = 600
limit_time_real = 1200
proxy_mode = True            # BẮT BUỘC khi đứng sau reverse proxy
list_db = False              # Tắt database manager trên production
```

### Bước 3 — Nginx reverse proxy
Cấu hình upstream cho cả web (8069) và websocket/longpolling (8072).
Xem template đầy đủ (SSL, security headers, gzip, static cache) tại `references/GUIDE.md` → "Nginx Reverse Proxy".

### Bước 4 — PostgreSQL
- Tạo user `odoo` với quyền superuser
- Tuning `shared_buffers`, `work_mem`, `effective_cache_size` theo RAM server
- Xem `references/GUIDE.md` → "PostgreSQL Setup"

### Bước 5 — SSL/HTTPS
- Dùng Let's Encrypt (certbot) cho domain thực
- Self-signed chỉ dùng cho môi trường test
- Xem `references/GUIDE.md` → "SSL/HTTPS Configuration"

### Bước 6 — Backup & Restore
- Script backup tự động: `pg_dump` (database) + `tar` (filestore)
- Cron job chạy hàng đêm, giữ 7 ngày
- Xem `references/GUIDE.md` → "Backup & Restore"

### Bước 7 — Monitoring & Logging
- Log rotation với logrotate (giữ 30 ngày)
- Health check endpoint `/web/health`
- Script kiểm tra process, HTTP response, database connection
- Xem `references/GUIDE.md` → "Monitoring & Logging"

## Constraints
- Production PHẢI dùng `workers > 0` (multi-process mode — bắt buộc Odoo 19)
- PHẢI enable `proxy_mode = True` khi đứng sau nginx/apache
- KHÔNG expose port 8069/8072 trực tiếp ra internet — phải qua nginx
- PHẢI dùng SSL/TLS (HTTPS) cho mọi production deployment
- `list_db = False` trên production để bảo mật database manager
- `admin_passwd` phải là random string mạnh (min 32 chars)
- PostgreSQL: KHÔNG để POSTGRES_PASSWORD mặc định trên production
- Docker image dùng `odoo:19.0` (không phải `odoo:latest` để tránh breaking change)

## References
- [Odoo 19 Installation Guide](https://www.odoo.com/documentation/19.0/administration/install.html)
- [Odoo 19 System Configuration](https://www.odoo.com/documentation/19.0/administration/odoo_sh/advanced/containers.html)
- [Docker Hub - Odoo](https://hub.docker.com/_/odoo)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Getting Started](https://letsencrypt.org/getting-started/)
- [PostgreSQL 15 Administration](https://www.postgresql.org/docs/15/admin.html)
