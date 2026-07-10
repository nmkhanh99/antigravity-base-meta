---
name: odoo-vietnamese-i18n
description: Hướng dẫn đa ngôn ngữ Odoo 19 - file .po, quản lý labels, Vietnamese translation. Use when the user asks to add Vietnamese, translate module, internationalization, or manage i18n in Odoo 19.
---

# Odoo 19 Vietnamese I18n

## Goal
Giúp agent thêm hỗ trợ Tiếng Việt và quản lý đa ngôn ngữ cho module Odoo 19.

## When to use this skill
- "thêm tiếng Việt", "dịch module", "translation"
- "i18n", "internationalization", "đa ngôn ngữ"
- "file .po", "bản dịch"

## Instructions

### 1. Export Translation Template
```bash
# Generate .pot file
./odoo-bin -d mydb -l vi_VN --i18n-export=/path/to/my_module/i18n/my_module.pot --modules=my_module
```

### 2. PO File Structure (`i18n/vi_VN.po`)
```po
# Translation of Odoo Server.
msgid ""
msgstr ""
"PO-Revision-Date: 2026-01-01 00:00+0000\n"
"Language: vi_VN\n"
"Content-Type: text/plain; charset=UTF-8\n"

#. module: my_module
#: model:ir.model.fields,field_description:my_module.field_my_model__name
msgid "Name"
msgstr "Tên"

#. module: my_module
#: model:ir.model,name:my_module.model_my_model
msgid "My Model"
msgstr "Mô hình của tôi"

#: model:ir.actions.act_window,name:my_module.action_my_model
msgid "My Models"
msgstr "Danh sách mô hình"
```

### 3. Model Labels (English Only in Code)
```python
class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'  # English only

    name = fields.Char(string='Name')           # English string
    partner_id = fields.Many2one(string='Customer')  # English string
    state = fields.Selection([
        ('draft', 'Draft'),         # English values
        ('confirmed', 'Confirmed'),
    ])
```

### 4. XML Views (English Only)
```xml
<page string="General Information"/>  <!-- English only -->
<button string="Confirm"/>            <!-- English only -->
<filter string="Draft Orders"/>       <!-- English only -->
<!-- Translation handled automatically via .po file -->
```

### 5. Python Code Translations
```python
from odoo import _

# Translated at runtime
raise ValidationError(_("Quantity cannot be negative!"))
self.message_post(body=_("Order %s confirmed", self.name))
```

## Constraints
- Code (Python + XML) giữ ENGLISH ONLY — dịch qua .po file.
- KHÔNG hardcode Vietnamese trong string/label.
- Selection values dùng English, dịch trong .po.

## Best practices
- Đặt file .po trong `i18n/vi_VN.po`.
- Dùng `_()` function cho strings cần dịch trong Python.
- Export .pot template rồi copy thành vi_VN.po để dịch.
- Đọc `resources/reference.md` cho conventions dịch thuật, plural forms.
