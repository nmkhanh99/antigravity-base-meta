---
name: odoo-models
description: Hướng dẫn định nghĩa Models, Fields, ORM, Constraints, và Computed Fields trong Odoo 19. Use when the user asks to create model, define fields, write ORM methods, add constraints, or work with Odoo 19 Python models.
---

# Odoo 19 Models

## Goal
Giúp agent tạo và quản lý Odoo models đúng chuẩn Odoo 19, bao gồm model definition, field types, ORM methods, decorators, constraints, và common patterns.

## When to use this skill
- "tạo model", "define model", "create model"
- "thêm field", "add field", "define field"
- "computed field", "related field"
- "ORM", "search", "create", "write", "unlink"
- "constraint", "validation", "unique"
- "@api.depends", "@api.onchange", "@api.constrains"

## Instructions

### Bước 1 — Xác định model structure
Tham khảo `references/GUIDE.md` Section 1 để khai báo đúng class, `_name`, `_description`, `_inherit`, `_order`.

### Bước 2 — Chọn đúng field types
Xem `references/GUIDE.md` Section 2 để chọn field phù hợp: Char, Integer, Float, Monetary, Date, Datetime, Selection, Many2one, One2many, Many2many, Computed, Related.

### Bước 3 — Viết ORM methods
Dùng `references/GUIDE.md` Section 3 cho search, create, write, unlink. Ưu tiên:
- `search_fetch()` thay vì `search_read()` (Odoo 19)
- `@api.model_create_multi` cho `create()`
- Batch operations thay vì vòng lặp

### Bước 4 — Khai báo constraints
- **SQL/Unique constraint (Odoo 19)**: Dùng declarative API tại class level:
  ```python
  _name_uniq = models.UniqueIndex('(name)', 'Name must be unique!')
  _check_amount = models.Constraint('CHECK(amount >= 0)', 'Amount must be positive!')
  ```
  Tên biến phải bắt đầu bằng `_`.
- **Python constraint**: Dùng `@api.constrains` + `ValidationError`.

### Bước 5 — Override CRUD nếu cần
Xem `references/GUIDE.md` Section 7 để override `create`, `write`, `unlink` đúng chuẩn.

### Bước 6 — Kiểm tra deprecated API
Xem danh sách deprecated trong GUIDE.md Section 8 trước khi commit code.

## Constraints
- KHÔNG dùng `name_get()` — deprecated trong Odoo 19. Thay bằng `_compute_display_name`.
- KHÔNG dùng `_sql_constraints` list — thay bằng declarative `models.UniqueIndex` / `models.Constraint`.
- KHÔNG dùng `read_group()` — deprecated. Thay bằng `_read_group()` (private) hoặc `formatted_read_group()` (public).
- KHÔNG dùng `self._cr`, `self._uid`, `self._context` — dùng `self.env.cr`, `self.env.uid`, `self.env.context`.
- KHÔNG dùng `@api.model` cho `create()` — dùng `@api.model_create_multi`.
- Luôn khai báo `_description` cho mỗi model.
- KHÔNG raw SQL trừ khi thực sự cần thiết (dùng `SQL()` wrapper nếu bắt buộc).
- Tên biến declarative constraint phải bắt đầu bằng `_`.

## References
- Odoo 19 ORM API: https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html
- Odoo 19 Fields: https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#fields
- Odoo 19 Changelog: https://www.odoo.com/documentation/19.0/developer/reference/backend/changelog.html
- Chi tiết code examples: `references/GUIDE.md`
