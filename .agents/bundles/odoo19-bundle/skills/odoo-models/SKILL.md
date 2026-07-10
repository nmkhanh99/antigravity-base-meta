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

### 1. Model Definition
```python
from odoo import models, fields, api

class MyModel(models.Model):
    _name = 'my.module.mymodel'
    _description = 'My Model Description'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'sequence, name'
    _rec_name = 'name'
```

### 2. Field Types (Quick Reference)
- **Char/Text/Html**: `fields.Char(string='Name', size=100, required=True)`
- **Integer/Float/Monetary**: `fields.Float(string='Price', digits=(16, 2))`
- **Boolean**: `fields.Boolean(default=True)`
- **Date/Datetime**: `fields.Date(default=fields.Date.today)`
- **Selection**: `fields.Selection([('draft','Draft'),('done','Done')], default='draft')`
- **Many2one**: `fields.Many2one('res.partner', ondelete='cascade', index=True)`
- **One2many**: `fields.One2many('model.line', 'parent_id')`
- **Many2many**: `fields.Many2many('my.tag', 'rel_table', 'model_id', 'tag_id')`

### 3. Odoo 19 Key Changes
- **Declarative constraints (thay thế _sql_constraints)**:
  - Dùng `_name_uniq = models.UniqueIndex('(name)', 'Message')` hoặc `_code_uniq = models.Constraint('UNIQUE(code)', 'Message')`.
  - Lưu ý: Biến phải bắt đầu bằng `_` (VD: `_check_amount`), Odoo sẽ tự động định nghĩa DB constraint là `<table_name>_<tên biến bỏ gạch dưới>`.
- **Domain class**: `from odoo.osv.expression import Domain`
- **search_fetch()**: Replaces search_read for optimized queries
- **@api.private**: Prevents RPC access to internal methods
- **_compute_display_name**: Replaces deprecated `name_get()`
- **check_access()**: Combined rights + rules check (replaces separate calls)
- **`<list>`**: Replaces `<tree>` tag in views
- **Direct modifiers**: `invisible="state == 'draft'"` (NOT attrs dict)

### 4. Computed Field Pattern
```python
total = fields.Float(compute='_compute_total', store=True)

@api.depends('line_ids.amount')
def _compute_total(self):
    for rec in self:
        rec.total = sum(rec.line_ids.mapped('amount'))
```

### 5. ORM Best Practices
```python
# ✅ DO: Batch operations
partners.write({'active': False})
emails = partners.mapped('email')
active = partners.filtered('active')

# ❌ DON'T: Loop updates, raw SQL, missing @api.depends
```

## Constraints
- **SQL Constraints (MỚI trong Odoo 19)**: Khai báo bằng declarative objects ở class level, thay vì list `_sql_constraints`. Tên biến phải bắt đầu bằng `_`.
  ```python
  class MyModel(models.Model):
      _name = 'my.model'

      _name_uniq = models.UniqueIndex('(name)', 'Name must be unique!')
      _check_amount = models.Constraint('CHECK(amount >= 0)', 'Amount must be positive!')
  ```
- Luôn dùng `@api.model_create_multi` thay vì `@api.model` cho create().
- KHÔNG dùng `name_get()`, `_cr`, `_uid`, `_context` — đã deprecated trong Odoo 19.
- KHÔNG dùng raw SQL trừ khi thực sự cần thiết.
- Luôn khai báo `_description` cho mỗi model.

## Best practices
- Dùng `search_fetch()` thay `search_read()` cho performance.
- Dùng declarative constraints (`models.UniqueIndex`, `models.Constraint`) thay `_sql_constraints`.
- Dùng `@api.private` cho internal methods.
- Đọc `resources/reference.md` để xem full field types, ORM patterns, context/environment usage.
