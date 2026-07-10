---
name: odoo-vietnamese-i18n
description: Hướng dẫn đa ngôn ngữ Odoo 19 - file .po, quản lý labels, Vietnamese translation. Use when the user asks to add Vietnamese, translate module, internationalization, or manage i18n in Odoo 19.
---

# Odoo 19 Vietnamese I18n

## Goal
Giúp agent thêm hỗ trợ Tiếng Việt và quản lý đa ngôn ngữ cho module Odoo 19 đúng chuẩn: không hardcode chuỗi tiếng Việt trong code, toàn bộ bản dịch tập trung vào file `.po`.

## When to use this skill
- "thêm tiếng Việt", "dịch module", "translation"
- "i18n", "internationalization", "đa ngôn ngữ"
- "file .po", "bản dịch"
- "export translation", "reload translation"

## Instructions

### Bước 1 — Định nghĩa strings trong Models (English only)
Đặt `string=` tại field trong Python, KHÔNG đặt trong XML:
```python
class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'  # English only

    name = fields.Char(string='Name')
    partner_id = fields.Many2one('res.partner', string='Customer')
    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
    ], string='Status')
```

### Bước 2 — Dọn dẹp XML Views
Bỏ thuộc tính `string` trong `<field>` nếu đã định nghĩa trong model:
```xml
<!-- Không nên -->
<field name="name" string="Tên kế hoạch"/>

<!-- Đúng chuẩn -->
<field name="name"/>
```

### Bước 3 — Sử dụng `_()` cho strings trong Python code
```python
from odoo import _

raise ValidationError(_("Quantity cannot be negative!"))
self.message_post(body=_("Order %s confirmed", self.name))
```

### Bước 4 — Export .pot template (tùy chọn)
```bash
./odoo-bin -d mydb -l vi_VN \
  --i18n-export=/path/to/my_module/i18n/my_module.pot \
  --modules=my_module
```

### Bước 5 — Tạo file `i18n/vi_VN.po`
Xem cấu trúc chi tiết và các loại msgid trong `references/GUIDE.md`.

### Bước 6 — Reload translation sau khi cập nhật
```bash
# Upgrade module để Odoo load lại file .po
./odoo-bin -d mydb -u my_module
```
Hoặc trong giao diện: Settings → Translations → Load a Translation → chọn "Overwrite Existing Terms".

## Constraints
- Code (Python + XML) giữ **ENGLISH ONLY** — bản dịch qua file `.po`.
- KHÔNG hardcode tiếng Việt trong `string=`, label XML, hay Selection values.
- File đặt tại `i18n/vi_VN.po` (không phải `i18n/vi.po`).
- Dùng `_()` (Python) hoặc `_t()` (JavaScript/OWL) cho runtime strings.
- Không dùng `name_get()` (deprecated từ Odoo 17) — dùng `_compute_display_name()`.

## References
- GUIDE.md: cấu trúc đầy đủ file `.po`, các loại msgid, mẹo tìm reference ID
