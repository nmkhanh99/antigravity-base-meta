---
name: odoo-xml-views
description: Viết XML views Odoo 17+/19 đúng chuẩn — form, list, kanban, search, kế thừa view (inheritance), visibility syntax, actions, menus. Kích hoạt khi user nói "viết view", "tạo form view", "list view", "kanban view", "kế thừa view", "view inheritance", "xml view", "tạo menu".
---

# Odoo XML View Patterns

## Goal
Tạo XML views Odoo 19 đúng chuẩn — form, list, kanban, search, view inheritance.

**Input**: Tên model, loại view cần tạo, fields hiển thị  
**Output**: XML views đầy đủ, sẵn sàng dùng

## When to use this skill
- "tạo form view", "tạo list view", "tạo kanban"
- "kế thừa view Odoo", "thêm field vào view"
- "tạo search view", "tạo action, menu"
- "chuyển attrs sang invisible="

## Instructions

### Bước 1 — Quy tắc v17+ bắt buộc

| Tính năng | ❌ REMOVED | ✅ REQUIRED |
|---|---|---|
| Visibility | `attrs="{'invisible': [...]}"` | `invisible="expr"` |
| List view | `<tree ...>` | `<list ...>` |
| view_mode | `'tree,form'` | `'list,form'` |

Bảng chuyển đổi attrs→expressions: xem `references/GUIDE.md#attrs-migration`

### Bước 2 — Form View
Xem template: `references/GUIDE.md#form`  
Structure: `<header>` buttons+statusbar → `<sheet>` button_box+ribbon+title+groups+notebook → `oe_chatter`.

### Bước 3 — List View
Xem template: `references/GUIDE.md#list`  
Key: `decoration-danger/warning/success`, `column_invisible="True"`, `optional="hide"`.

**Delete control theo state**: `delete` chỉ nhận `"0"`/`"1"` cứng — dùng 2 view riêng + chọn view trong action. Xem `references/GUIDE.md#delete-control`.

### Bước 4 — Search View
Xem template: `references/GUIDE.md#search`  
Key: `<field>`, `<filter>`, `<group>` group-by, `<searchpanel>` sidebar.

### Bước 5 — Kanban View
Xem template: `references/GUIDE.md#kanban`  
Key: `default_group_by`, `kanban_color()`, `o_kanban_record_top/body/bottom`.

### Bước 6 — View Inheritance
Xem template: `references/GUIDE.md#inheritance`  
**QUAN TRỌNG**: Luôn đọc parent view trước khi viết xpath.  
XPath positions: `after/before/replace/inside/attributes`.

### Bước 7 — Actions và Menus
Xem template: `references/GUIDE.md#actions`  
`ir.actions.act_window` → root menuitem → submenu → action menuitem.

### Bước 8 — Common Widgets
Xem `references/GUIDE.md#widgets` — statusbar, badge, priority, many2one_avatar_user, handle, progressbar...

## Constraints
- **KHÔNG** dùng `<tree>` — phải dùng `<list>`
- **KHÔNG** dùng `attrs=` — phải dùng `invisible=`, `readonly=`, `required=`
- **PHẢI** đọc parent view trước khi viết view inheritance

## Best practices
- Form header: buttons + statusbar
- Groups theo 2 cột: general info bên trái, details bên phải
- Notebook cho One2many lines và notes
- Chatter (`oe_chatter`) chỉ khi model kế thừa `mail.thread`
- Dùng `column_invisible="True"` thay `invisible="1"` trong list
