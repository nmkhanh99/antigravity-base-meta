---
name: odoo-backend-reports
description: Hướng dẫn Reports backend Odoo 19 - QWeb PDF, ir.actions.report, report templates, paper format, custom report models, report assets/fonts và Excel exports. Use when writing FRD or code for Odoo reports/exports.
---

# Odoo Backend Reports

## Goal
Giúp agent đặc tả/tạo report Odoo 19: QWeb PDF, paper format, report action, custom report data và export Excel.

## Source-first khi viết FRD
Trước khi đặc tả report, dùng `odoo-backend-source-analysis` để đọc report action/template/paperformat hiện có trong base/custom. Nếu report dùng frontend template/assets hoặc custom fonts, dùng thêm `odoo-frontend-source-analysis`.

## Khi nào dùng
- PDF report, print action, `ir.actions.report`.
- QWeb report template, barcode, translatable report.
- Custom paper format, custom font/CSS cho PDF.
- Excel export qua controller/wizard.

## Quy tắc QWeb Report
- Report PDF viết bằng HTML/QWeb và render qua `wkhtmltopdf`.
- Report cần `ir.actions.report` và template tương ứng.
- Template PDF thường wrap bằng `web.html_container` và `web.external_layout`.
- Custom font/CSS cho report phải khai báo vào `web.reports_assets_common`, không phải `web.assets_backend`.

## Report Action Pattern
```xml
<record id="action_report_my_model" model="ir.actions.report">
    <field name="name">My Report</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.report_my_model</field>
    <field name="report_file">my_module.report_my_model</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="paperformat_id" ref="my_module.paperformat_my_report"/>
</record>
```

## Template Pattern
```xml
<template id="report_my_model">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2 t-out="doc.name"/>
                    <table class="table table-sm">
                        <t t-foreach="doc.line_ids" t-as="line" t-key="line.id">
                            <tr>
                                <td t-out="line.display_name"/>
                                <td t-out="line.amount"
                                    t-options="{'widget': 'monetary', 'display_currency': doc.currency_id}"/>
                            </tr>
                        </t>
                    </table>
                </div>
            </t>
        </t>
    </t>
</template>
```

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Report metadata | Name, XML ID, model, binding, menu/button |
| Data source | `docs`, custom report model, wizard params |
| Template | Template XML ID, layout, sections, fields |
| Format | PDF/HTML/XLSX, paper format, orientation, margins |
| Security | Groups, record access, export permission |
| Assets | CSS/font bundle, especially `web.reports_assets_common` |
| Test | Render, permissions, multi-company/language/currency |

## References
- Odoo 19 QWeb Reports: https://www.odoo.com/documentation/19.0/developer/reference/backend/reports.html
- Odoo 19 QWeb frontend syntax: https://www.odoo.com/documentation/19.0/developer/reference/frontend/qweb.html
