---
name: odoo-data-setup
description: Hướng dẫn thiết lập dữ liệu cấu hình (Metadata) trong Odoo bao gồm Groups, Rules, Sequences, Menus và Actions. Use when the user asks to setup data, configure groups, sequences, menus, or initialize module data in Odoo 19.
---

# Odoo 19 Data Setup

## Goal
Giúp agent thiết lập dữ liệu cấu hình ban đầu cho module Odoo 19: groups, sequences, menus, actions và metadata — đảm bảo `noupdate`, i18n và thứ tự load đúng chuẩn.

## When to use this skill
- "setup data", "cấu hình dữ liệu", "initial data"
- "tạo menu", "tạo action", "window action"
- "sequence setup", "tạo sequence"
- "cấu hình groups", "phân quyền nhóm"
- "ACL", "ir.model.access"

## Instructions

### Bước 1 — Xác định cấu trúc file
Luôn phân tách dữ liệu theo mục đích (xem GUIDE.md § Cấu trúc file):
- `security/security.xml` — Groups & Record Rules
- `security/ir.model.access.csv` — ACL
- `data/ir_sequence_data.xml` — Sequences
- `data/data.xml` — Default config records

Trong `__manifest__.py`, khai báo `security/security.xml` **trước** views và data khác.

### Bước 2 — Khai báo Groups (Odoo 19)
Odoo 19 dùng `res.groups.privilege` thay `ir.module.category`.
Xem template đầy đủ tại GUIDE.md § Security Groups & Privileges.

### Bước 3 — Record Rules (đa công ty)
Dùng `domain_force` với `company_ids` để bảo vệ dữ liệu đa công ty.
Xem GUIDE.md § Record Rules.

### Bước 4 — Sequences
Bọc trong `<odoo noupdate="1">`. Dùng `company_id eval="False"` cho sequences dùng chung.
Xem GUIDE.md § Sequences.

### Bước 5 — Menus & Window Actions
Khai báo `ir.actions.act_window` trước `<menuitem>`.
Xem GUIDE.md § Menus & Actions.

### Bước 6 — ACL (ir.model.access.csv)
Mỗi model mới bắt buộc có ACL cho tất cả groups liên quan.
Xem GUIDE.md § ACL.

### Bước 7 — i18n
Trường `name` trong XML dùng tiếng Anh, dịch sang tiếng Việt qua `i18n/vi.po`.
Xem GUIDE.md § i18n.

## Constraints
- **Bắt buộc `noupdate="1"`** cho Groups, Sequences, Record Rules, System Parameters.
- **Không hardcode tiếng Việt** vào trường `name` trong XML — dùng i18n.
- **Security files load trước views** trong manifest `data` list.
- **Menu `groups`** dùng XML ID (không phải group name string).
- **Không dùng `ir.module.category`** — Odoo 19 thay bằng `res.groups.privilege`.
- **Mỗi model mới** phải có `ir.model.access.csv` (luật bất biến CLAUDE.md).

## References
- GUIDE.md — template XML và CSV đầy đủ, ví dụ i18n
