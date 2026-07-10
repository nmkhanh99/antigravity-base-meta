---
name: odoo-report
description: Tạo báo cáo PDF Odoo — QWeb report template, report action, custom report model, paper format, report inheritance. Kích hoạt khi user nói "tạo báo cáo", "PDF report", "in phiếu", "QWeb report", "report template", "kế thừa báo cáo", "báo cáo PDF", "in hóa đơn".
---

# Odoo Report Patterns (QWeb PDF)

## Goal
Tạo báo cáo PDF Odoo đúng chuẩn — report action, QWeb template, custom report model, paper format, inheritance.

**Input**: Tên model cần in, nội dung cần hiển thị  
**Output**: XML report action + QWeb template + Python report model (nếu cần)

## When to use this skill
- "tạo báo cáo PDF", "in phiếu"
- "QWeb template cho report"
- "custom paper format"
- "kế thừa báo cáo invoice"

## Instructions

### Bước 1 — Cấu trúc file
```
my_module/report/
├── __init__.py
├── my_report.py           # Logic (optional)
└── my_report_templates.xml
```
Manifest: thêm `'report/my_report_templates.xml'` vào `data`.

### Bước 2 — Report Action + QWeb Template
Xem template: `references/GUIDE.md#action-template`  
Key points:
- `report_name` phải trùng với `template id`
- `binding_model_id` để hiện trong Print menu
- Structure: `web.html_container` → `t-foreach="docs"` → `web.external_layout` → `<div class="page">`

### Bước 3 — QWeb directives trong Reports
Xem `references/GUIDE.md#directives`  
- `t-field` + `t-options` cho monetary, date, partner address
- `t-foreach` với `_index`, `_last` cho numbered rows
- `t-att-class` cho conditional row styling

### Bước 4 — Custom Report Model (AbstractModel)
Xem template: `references/GUIDE.md#report-model`  
`_name = 'report.{module}.{report_name}'`, method `_get_report_values()` trả về dict context.

### Bước 5 — Paper Format
Xem template: `references/GUIDE.md#paper-format`  
Standards: `base.paperformat_euro` (A4), `base.paperformat_us` (Letter).

### Bước 6 — Print từ Python
Xem `references/GUIDE.md#print-python` — `report_action(self)`, `report_action(ids)`, `report_action(self, data=data)`.

### Bước 7 — Report Inheritance
Xem template: `references/GUIDE.md#inheritance`  
`inherit_id` + `<xpath>` để thêm field vào báo cáo có sẵn.

### Bước 8 — CSS cho Report
Xem `references/GUIDE.md#css` — inline styles, page break, external SCSS qua `web.report_assets_common`.

## Constraints
- `report_name` **phải** trùng với `template id`
- Report model `_name`: `report.{module}.{report_name}`
- `t-field` chỉ dùng trong reports/website

## Best practices
- `web.external_layout` cho tài liệu chuyên nghiệp (có company header)
- Monetary widget + currency cho tất cả số tiền
- Check `t-if` trước khi display field có thể null
