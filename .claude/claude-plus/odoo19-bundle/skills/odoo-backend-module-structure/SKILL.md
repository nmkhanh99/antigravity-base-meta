---
name: odoo-backend-module-structure
description: Hướng dẫn cấu trúc module Odoo 19 phía backend - __manifest__.py, depends, data/demo/assets, load order, Python package layout, security/views/report/static/tests. Use when creating, reviewing, or specifying Odoo module structure and manifest.
---

# Odoo Backend Module Structure

## Goal
Giúp agent tạo/spec module Odoo 19 có manifest, dependency, data load order và static assets đúng chuẩn Odoo 19 (không áp dụng cho Odoo < 17).

## When to use this skill
- Tạo/scaffold module mới, review `__manifest__.py` hiện tại.
- Viết FRD-STG kiến trúc module (phần dependency, load order, assets).
- Xác định bundle assets, data/demo/test layout.
- Phát hiện deprecated patterns từ Odoo < 17 (qweb/css/js keys, name_get, attrs...).

## Instructions

### Bước 1 — Source-first trước khi viết FRD
Dùng `odoo-backend-source-analysis` để đọc `__manifest__.py` của custom module và các base modules liên quan. Không thêm dependency vào FRD nếu chưa có lý do từ source/BRD/PRD.

### Bước 2 — Xây dựng cấu trúc thư mục
Tham chiếu layout chuẩn trong `references/GUIDE.md#directory-layout`.

### Bước 3 — Viết `__manifest__.py`
- Các field bắt buộc (không có default gây warning): `name`, `author`, `license`.
- `depends` KHÔNG được để rỗng `[]` — Odoo 19 tự ép thành `['base']` và log warning; khai báo tường minh.
- `version` phải đúng định dạng `19.x.y.z` (2–5 phần ngăn bởi dấu chấm); sai format → `installable=False` tự động.
- `auto_install` có thể là `True`, `False`, hoặc **list** các module trigger (tất cả phải có trong `depends`).
- `pre_init_hook` / `post_init_hook` / `uninstall_hook` nhận tham số `env` (không phải `cr, uid` như Odoo ≤ 16).
- Dùng `Manifest.for_addon('module_name')` để đọc manifest lập trình (thay `load_manifest()` đã deprecated từ 19.0).
- Xem template đầy đủ tại `references/GUIDE.md#manifest-pattern`.

### Bước 4 — Xác định Data Load Order
Tuân thủ thứ tự trong `data` list:
1. `security/security.xml` — Groups/categories.
2. `security/ir.model.access.csv` — ACL.
3. Record rules, metadata dependencies.
4. `data/*.xml` — Sequences, config (dùng `noupdate="1"` cho production data).
5. `views/*.xml` — Views/actions/menus.
6. `report/*.xml` — Reports/templates.

### Bước 5 — Khai báo Assets
- Dùng key `assets` trong manifest (key `qweb`, `css`, `js` từ Odoo < 15 không còn hỗ trợ).
- Bundle phổ biến: `web.assets_backend`, `web.assets_frontend`, `web.assets_unit_tests`, `web.reports_assets_common`.
- Hỗ trợ 7 directive: `append` (default), `prepend`, `before`, `after`, `include`, `remove`, `replace`.
- Glob patterns được hỗ trợ, ví dụ `my_module/static/src/js/**/*.js`.
- Xem ví dụ đầy đủ tại `references/GUIDE.md#asset-bundles`.

### Bước 6 — FRD-STG Checklist
Xác nhận đủ các mục trong `references/GUIDE.md#frd-stg-checklist`.

## Constraints
- KHÔNG dùng `name_get()`, `read_group()`, tag `<tree>`, attrs, `_cr/_uid/_context` (deprecated Odoo < 17).
- KHÔNG dùng `load_manifest()` hoặc `get_modules_with_version()` (deprecated Odoo 19).
- KHÔNG dùng các key `qweb`, `css`, `js` trong manifest (deprecated Odoo < 15).
- `pre_init_hook` / `post_init_hook` / `uninstall_hook` PHẢI nhận `env`, không nhận `(cr, uid)`.
- `external_dependencies.python` phải dùng tên PyPI hợp lệ (PEP 508), không phải tên import module; dùng tên import logs warning trong Odoo 19.
- Mọi model mới phải có ACL (`ir.model.access.csv`).
- Multi-company phải dùng Strategy Pattern (xem `rules/multicompany.skill.md`).
- Không raw SQL trừ khi dùng `SQL()` wrapper.

## References
- Odoo 19 Module Manifests: https://www.odoo.com/documentation/19.0/developer/reference/backend/module.html
- Odoo 19 Assets: https://www.odoo.com/documentation/19.0/developer/reference/frontend/assets.html
