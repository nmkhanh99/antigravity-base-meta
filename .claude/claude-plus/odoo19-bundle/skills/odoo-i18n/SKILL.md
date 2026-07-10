---
name: odoo-i18n
description: Đa ngôn ngữ và dịch thuật Odoo 19 — hàm _(), translate=True, file PO, with_context(lang=), gửi email theo ngôn ngữ partner, selection động có dịch. Kích hoạt khi user nói "đa ngôn ngữ", "dịch thuật", "translation", "i18n", "PO file", "ngôn ngữ", "locale", "_() function".
---

# Translation & i18n Patterns (Odoo 19)

## Goal
Implement đa ngôn ngữ đúng chuẩn cho module Odoo 19 — chuỗi dịch, field content, PO files, context lang.

**Input**: Mô tả nhu cầu đa ngôn ngữ  
**Output**: Python + XML + PO file patterns đầy đủ

## When to use this skill
- "hỗ trợ đa ngôn ngữ", "dịch module"
- "tạo file PO", "gửi email theo ngôn ngữ partner"
- "field có nội dung dịch theo ngôn ngữ"
- "selection label dịch được"

## Instructions

### Bước 1 — Chuỗi dịch trong Python (hàm `_()`)
Key rules:
- `_("Message")` cho simple strings
- `_("Record %s created") % name` cho placeholders
- `_("%(name)s by %(user)s") % {'name': x, 'user': y}` cho named
- **KHÔNG** dùng f-strings trong `_()` — không được extract
- **KHÔNG** dịch log messages (`_logger.info`)

Xem examples: `references/GUIDE.md#_function`

### Bước 2 — Field content dịch được
```python
name: str = fields.Char(string='Name', translate=True)
description: str = fields.Text(string='Description', translate=True)
```
`translate=True` tốn thêm query — chỉ dùng khi thực sự cần.

### Bước 3 — Đọc/ghi field theo ngôn ngữ
`self.with_context(lang='fr_FR').name` — xem `references/GUIDE.md#with-context-lang`

### Bước 4 — Selection động có dịch
`selection='_get_type_selection'` với method trả về `[('key', _('Label'))]`  
Xem template: `references/GUIDE.md#dynamic-selection`

### Bước 5 — Gửi email theo ngôn ngữ partner
```python
lang = partner.lang or 'en_US'
template.with_context(lang=lang).send_mail(record.id)
```

### Bước 6 — QWeb report theo ngôn ngữ partner
`line.product_id.with_context(lang=doc.partner_id.lang).name` — xem `references/GUIDE.md#report-lang`

### Bước 7 — Export/Import translations
```bash
./odoo-bin -d mydb --i18n-export=/tmp/my_module.pot --modules=my_module
./odoo-bin -d mydb --i18n-import=/tmp/vi.po --language=vi
```

### Bước 8 — Cấu trúc PO file
`my_module/i18n/` → `.pot` (template), `vi.po`, `fr.po`, `de.po`  
Xem format: `references/GUIDE.md#po-format`

### Bước 9 — Test translations
Xem `references/GUIDE.md#test-translation` — ghi fr_FR value, assert với `with_context(lang='fr_FR')`.

## Constraints
- **KHÔNG** dùng f-strings trong `_()` — không được extract bởi i18n tool
- **KHÔNG** concatenate chuỗi dịch
- **KHÔNG** dịch log messages — chỉ `_()` cho UI/user messages

## Best practices
- `%s` hoặc `%(name)s` cho placeholders, không `.format()` hay f-string
- Field `translate=True` cho name/description để admin dịch từ UI
- Gửi email theo `partner.lang` (không phải `user.lang`)
- Bảng tóm tắt: xem `references/GUIDE.md#translation-table`
