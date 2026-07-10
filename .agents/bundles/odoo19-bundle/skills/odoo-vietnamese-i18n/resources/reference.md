---
name: Odoo Vietnamese I18n
description: Skill này được sử dụng khi người dùng yêu cầu "thêm tiếng Việt", "dịch module", "internationalization (i18n)", hoặc "bản dịch Odoo". Nó tập trung vào việc sử dụng file .po, quản lý labels trong model và giữ XML views sạch sẽ (English-only).
metadata:
  author: Antigravity
  version: 1.1.0
---

# Quy trình Thêm Ngôn ngữ Tiếng Việt (Odoo I18n)

Tài liệu này hướng dẫn cách thêm hỗ trợ tiếng Việt cho một module Odoo một cách chuyên nghiệp, tránh Hardcode chuỗi tiếng Việt vào XML views.

## 🎯 Mục tiêu
- Giữ XML Views nguyên bản (Tiếng Anh) để hỗ trợ đa ngôn ngữ.
- Tập trung toàn bộ bản dịch vào file `i18n/vi.po`.
- Định nghĩa các nhãn (labels/strings) trực tiếp trong Python Models để đồng nhất dữ liệu.

## 📋 Các bước thực hiện

### 1. Định nghĩa Strings trong Models
Thay vì để `string="..."` trong XML, hãy định nghĩa chúng ngay tại trường (Field) trong Python model.

```python
# Ví dụ trong models/your_model.py
name = fields.Char(string='Plan Name', required=True)
warehouse_id = fields.Many2one('stock.warehouse', string='Demand Warehouse')
```

### 2. Dọn dẹp XML Views
Gỡ bỏ các thuộc tính `string` trong các thẻ `<field>` nếu chúng đã được định nghĩa trong Model.

```xml
<!-- VIEW GỐC (Nên tránh) -->
<field name="name" string="Tên kế hoạch"/>

<!-- VIEW CHUẨN (Khuyên dùng) -->
<field name="name"/>
```

### 3. Cấu hình file dịch `i18n/vi.po`
Tạo thư mục `i18n/` trong module và thêm file `vi.po`.

**Cấu trúc file `vi.po`:**
```po
msgid "Plan Name"
msgstr "Mã kế hoạch"

msgid "Demand Warehouse"
msgstr "Kho nhu cầu"
```

### 4. Dịch Menu và Action
Để dịch tên Menu hoặc Action, hãy tìm mã định danh (id) của chúng.

```po
#. module: your_module
#: model:ir.ui.menu,name:your_module.menu_root
msgid "Manufacturing Planning"
msgstr "Kế hoạch sản xuất"
```

## 💡 Lưu ý quan trọng
- **Thống nhất thuật ngữ**: Đảm bảo các nhãn giống nhau có cùng một bản dịch.
- **Không Hardcode**: Tuyệt đối không viết trực tiếp Tiếng Việt vào file `.xml` hoặc `.py` (trừ Docstrings).
- **Reload Translation**: Sau khi cập nhật file `.po`, cần nâng cấp module (`-u module_name`) và chọn "Overwrite Existing Terms" nếu cần thiết trong giao diện Odoo.

---

## 🏗️ Cấu trúc Chi tiết File `.po`

Hiểu cách Odoo phân loại các chuỗi dịch giúp bạn tìm kiếm và sửa đổi nhanh hơn:

### 1. Dịch Menu (Menus)
Sử dụng ID của menu và thuộc tính `name`.
```po
#: model:ir.ui.menu,name:scx_mrp_mps.menu_scx_mrp_mps_root
msgid "Manufacturing Planning"
msgstr "Kế hoạch sản xuất"
```

### 2. Dịch Nhãn Trường (Model Fields)
Odoo sử dụng định dạng `field_model_name__field_name`. (Lưu ý: model name dùng dấu gạch dưới thay vì dấu chấm).
```po
#: model:ir.model.fields,field_description:scx_mrp_mps.field_scx_production_plan__start_date
msgid "Start Date"
msgstr "Ngày bắt đầu"
```

### 3. Dịch Các Lựa chọn (Selection Fields)
Hỗ trợ dịch các giá trị trong dropdown (`fields.Selection`).
```po
#: model:ir.model.fields.selection,name:scx_mrp_mps.selection__scx_forecast_plan__frequency__month
msgid "Monthly"
msgstr "Hàng tháng"
```
**Quy tắc:** `selection__<model_name>__<field_name>__<key_name>`

### 4. Dịch Văn bản Tĩnh trong View (XML Terms)
Các chuỗi nằm trong thẻ `<span>`, `<label>`, hoặc thuộc tính `string` của view (nếu không phải field).
```po
#: model_terms:ir.ui.view,arch_db:scx_mrp_mps.scx_production_plan_view_form
msgid "<span class=\"o_stat_text\">Planned Order</span>"
msgstr "<span class=\"o_stat_text\">Kế hoạch sản xuất</span>"
```

### 5. Dịch Chuỗi trong Code (Python/JS)
Các chuỗi được bao bọc bởi `_()` (Python) hoặc `_t()` (JavaScript).
```po
#. odoo-python
#: code:addons/scx_mrp_mps/models/scx_do_plan.py:0
msgid "Internal transfer completed."
msgstr "Chuyển kho nội bộ hoàn thành."
```

### 6. Dịch Ràng buộc Dữ liệu (Constraints)
Dùng cho các thông báo lỗi khi vi phạm `_sql_constraints` hoặc `models.Constraint`.
```po
#: model:ir.model.constraint,message:scx_mrp_mps.constraint_scx_do_plan_line_plan_product_partner_uniq
msgid "Each product/customer pair must be unique per DO Plan!"
msgstr "Mỗi cặp sản phẩm/khách hàng phải là duy nhất trong cùng một kế hoạch DO!"
```
**Quy tắc:** `model:ir.model.constraint,message:<module_name>.constraint_<model_name_with_underscores>_<constraint_name>`

## 🪜 Mẹo tìm nhanh Model/Field ID
Nếu bạn không chắc chắn `msgid` hoặc `reference` là gì:
1. Vào Odoo -> **Settings** -> **Translations** -> **Translated Terms**.
2. Tìm kiếm chuỗi tiếng Anh (Source).
3. Odoo sẽ hiển thị "Technical Name" chính là chuỗi `reference` trong file `.po`.
