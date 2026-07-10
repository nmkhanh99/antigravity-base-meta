# Odoo Vietnamese I18n - Reference Guide

## Cấu trúc file `i18n/vi_VN.po`

### Header bắt buộc
```po
# Translation of Odoo Server.
# This file contains the translation of the following modules:
# * my_module
msgid ""
msgstr ""
"PO-Revision-Date: 2026-01-01 00:00+0000\n"
"Last-Translator: <>\n"
"Language-Team: \n"
"Language: vi_VN\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: \n"
"Plural-Forms: nplurals=1; plural=0;\n"
"X-Generator: Odoo\n"
```

---

## Các loại msgid và cú pháp reference

### 1. Dịch tên Model
```po
#. module: my_module
#: model:ir.model,name:my_module.model_my_model
msgid "My Model"
msgstr "Mô hình của tôi"
```

### 2. Dịch nhãn Field (field_description)
Quy tắc: `field_<module>.<field_<model_name_underscores>__<field_name>>`
```po
#. module: my_module
#: model:ir.model.fields,field_description:my_module.field_my_model__name
msgid "Name"
msgstr "Tên"

#: model:ir.model.fields,field_description:my_module.field_my_model__start_date
msgid "Start Date"
msgstr "Ngày bắt đầu"
```

### 3. Dịch Selection values
Quy tắc: `selection__<model_name_underscores>__<field_name>__<key>`
```po
#: model:ir.model.fields.selection,name:my_module.selection__my_model__state__draft
msgid "Draft"
msgstr "Nháp"

#: model:ir.model.fields.selection,name:my_module.selection__my_model__state__confirmed
msgid "Confirmed"
msgstr "Đã xác nhận"
```

### 4. Dịch Menu
```po
#: model:ir.ui.menu,name:my_module.menu_root
msgid "Manufacturing Planning"
msgstr "Kế hoạch sản xuất"

#: model:ir.ui.menu,name:my_module.menu_sub_item
msgid "Plans"
msgstr "Kế hoạch"
```

### 5. Dịch Action (act_window)
```po
#: model:ir.actions.act_window,name:my_module.action_my_model
msgid "My Models"
msgstr "Danh sách mô hình"
```

### 6. Dịch văn bản tĩnh trong XML view (arch_db)
Dùng cho `string=` trong button, page, filter, label không gắn với field:
```po
#: model_terms:ir.ui.view,arch_db:my_module.my_model_view_form
msgid "General Information"
msgstr "Thông tin chung"

#: model_terms:ir.ui.view,arch_db:my_module.my_model_view_form
msgid "<span class=\"o_stat_text\">Planned Order</span>"
msgstr "<span class=\"o_stat_text\">Kế hoạch đặt hàng</span>"
```

### 7. Dịch chuỗi Python/JS runtime
```po
#. odoo-python
#: code:addons/my_module/models/my_model.py:0
#, python-format
msgid "Order %s confirmed"
msgstr "Đơn hàng %s đã được xác nhận"

#. odoo-python
#: code:addons/my_module/models/my_model.py:0
msgid "Quantity cannot be negative!"
msgstr "Số lượng không được âm!"
```

### 8. Dịch thông báo Constraint
Quy tắc: `constraint_<model_name_underscores>_<constraint_name>`
```po
#: model:ir.model.constraint,message:my_module.constraint_my_model_name_uniq
msgid "Name must be unique!"
msgstr "Tên phải là duy nhất!"
```

---

## Quy trình làm việc đầy đủ

```
1. Viết model + field string= (English) trong Python
2. Export .pot: ./odoo-bin --i18n-export=... --modules=my_module
3. Copy .pot → i18n/vi_VN.po
4. Điền msgstr tiếng Việt cho từng entry
5. Upgrade module: ./odoo-bin -u my_module
6. Kiểm tra trong UI: Settings → Language → Vietnamese
```

---

## Mẹo tìm reference ID nhanh

### Cách 1: Qua giao diện Odoo
1. Settings → Translations → Translated Terms
2. Tìm chuỗi tiếng Anh (Source)
3. Xem cột "Technical Name" = reference trong `.po`

### Cách 2: Grep trong .pot file đã export
```bash
grep -A2 "msgid \"Name\"" i18n/my_module.pot
```

### Cách 3: Đọc file .po của module tương tự trong Odoo core
```bash
find /path/to/odoo/addons -name "vi_VN.po" | head -5
```

---

## Quy ước đặt tên file

| Ngôn ngữ | File |
|----------|------|
| Tiếng Việt (Việt Nam) | `i18n/vi_VN.po` |
| Tiếng Anh (UK) | `i18n/en_GB.po` |
| Template export | `i18n/my_module.pot` |

---

## JavaScript / OWL translation (Odoo 19)

```javascript
import { _t } from "@web/core/l10n/translation";

// Trong OWL component
setup() {
    this.label = _t("My Label");
}

// Trong service/utility
const msg = _t("Error: %s", errorText);
```

File `.po` sẽ có entry:
```po
#. odoo-javascript
#: code:addons/my_module/static/src/components/my_component.js:0
msgid "My Label"
msgstr "Nhãn của tôi"
```

---

## Checklist trước khi release

- [ ] Tất cả `string=` trong Python là tiếng Anh
- [ ] Không có tiếng Việt hardcode trong XML (trừ nội dung data)
- [ ] File `i18n/vi_VN.po` tồn tại và có header đầy đủ
- [ ] Đã dịch: menu, action, field labels, selection values
- [ ] Đã dịch: thông báo lỗi ValidationError / Constraint
- [ ] Upgrade module thành công sau khi thêm/sửa `.po`
- [ ] Kiểm tra UI với ngôn ngữ Vietnamese
