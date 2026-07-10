---
name: odoo-data-files
description: Hướng dẫn tạo Data Files trong Odoo 19 - XML/CSV data, noupdate, sequences, server actions, scheduled actions. Use when the user asks to create data file, add default data, define sequence, configure server action, or setup scheduled action.
---

# Odoo 19 Data Files

## Goal
Giúp agent tạo và quản lý data files (XML/CSV) cho default data, sequences, server actions, và scheduled actions theo chuẩn Odoo 19.

## When to use this skill
- "tạo data file", "default data", "demo data"
- "sequence", "auto number", "tạo sequence"
- "server action", "automated action"
- "scheduled action", "cron job", "tác vụ định kỳ"
- "noupdate", "ref()", "xml_id"
- "email template", "mail template"
- "system parameter", "config parameter"
- "csv data", "master data"

## Instructions

### Bước 1 — Xác định loại data file cần tạo

| Loại | File | noupdate |
|------|------|----------|
| Sequence | `data/sequence.xml` | `1` (bắt buộc) |
| Master data | `data/data.xml` | `1` |
| Cron jobs | `data/cron.xml` | `1` (bắt buộc) |
| Server actions | `data/server_action.xml` | `1` |
| Email templates | `data/mail_template.xml` | `1` |
| Config params | `data/config.xml` | `1` |
| Demo data | `demo/demo.xml` | `1` |
| Views/ACL | `views/`, `security/` | `0` |

### Bước 2 — Tạo file XML theo template phù hợp

Tham khảo `references/GUIDE.md` cho code examples chi tiết:
- Section 2: Basic structure + noupdate flag
- Section 3: Field types (simple, relational, eval expressions)
- Section 4: Sequences
- Section 6: Configuration data + Cron jobs
- Section 7: Email templates
- Section 8: Server actions
- Section 9: Demo data
- Section 10: CSV data files

### Bước 3 — Đăng ký vào `__manifest__.py`

```python
{
    'data': [
        'security/ir.model.access.csv',
        'data/sequence.xml',
        'data/data.xml',
        'views/my_views.xml',
    ],
    'demo': [
        'demo/demo.xml',
    ],
}
```

### Bước 4 — Sử dụng sequence trong model (nếu cần)

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', '/') == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals_list)
```

### Bước 5 — Kiểm tra Many2many operations

```xml
<!-- Replace all -->
<field name="tag_ids" eval="[(6, 0, [ref('tag_1'), ref('tag_2')])]"/>
<!-- Add -->
<field name="group_ids" eval="[(4, ref('base.group_user'))]"/>
<!-- Remove -->
<field name="tag_ids" eval="[(3, ref('tag_old'))]"/>
<!-- Clear all -->
<field name="tag_ids" eval="[(5,)]"/>
```

## Constraints
- Data cho security/sequences/cron PHẢI dùng `noupdate="1"` — không được bỏ qua.
- KHÔNG hardcode database IDs — luôn dùng `ref('module.xml_id')`.
- CSV files chỉ dùng cho `ir.model.access` (ACL), không dùng cho master data phức tạp.
- Demo data chỉ load khi install với demo flag — KHÔNG dùng cho production config.
- `model_id` trong cron/server action phải dùng `ref="model_my_model"` (underscore, không dấu chấm).
- Thứ tự trong `__manifest__.py` phải đúng dependency: security trước views, data trước demo.
- Odoo 19: dùng `@api.model_create_multi` thay vì `@api.model` + `create` đơn.

## References
- GUIDE.md: `/Users/khanhnm/Desktop/odoo-19.0/.claude/skills/odoo-data-files/references/GUIDE.md`
- Odoo 19 ORM docs: https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html
- Odoo 19 Data files: https://www.odoo.com/documentation/19.0/developer/reference/backend/data.html
