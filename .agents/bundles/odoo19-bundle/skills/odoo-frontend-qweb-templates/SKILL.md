---
name: odoo-frontend-qweb-templates
description: Hướng dẫn QWeb frontend trong Odoo 19 - OWL templates, kanban templates, website/portal snippets, template inheritance, t-name/t-inherit/t-out/t-foreach/t-att và rendering an toàn. Use when specifying or writing frontend QWeb templates, not QWeb PDF reports.
---

# Odoo Frontend QWeb Templates

## Goal
Giúp agent đặc tả hoặc viết QWeb template frontend Odoo 19 cho OWL, kanban, website/portal và template inheritance. QWeb report PDF dùng `odoo-backend-reports`.

## Source-first khi viết FRD
Trước khi đặc tả QWeb frontend, dùng `odoo-frontend-source-analysis` để đọc template XML hiện có, `t-name`, `t-inherit`, XPath và data context. Không dùng template name giả nếu có XML ID/name thực tế.

## Khi nào dùng
- OWL component XML template trong `static/src/.../*.xml`.
- Kanban card/template custom.
- Website/portal frontend templates.
- Kế thừa template bằng `t-inherit` và XPath.

## Quy tắc Odoo 19
- Template OWL đặt trong `<templates xml:space="preserve">`.
- Tên template theo `addon_name.ComponentName` hoặc `addon_name.TemplateName`.
- Dùng `t-out` để output dữ liệu; tránh `t-raw`. Chỉ render HTML safe khi dữ liệu đã sanitize/Markup có kiểm soát.
- Với loop trong OWL template, luôn có `t-key`.
- Dùng `t-inherit` + `t-inherit-mode="primary|extension"`; không dùng cơ chế cũ `t-extend`/`t-jquery` cho code mới.

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Template | `t-name`, file path, owner component/view |
| Type | New / Inherit / OWL / Kanban / Website / Portal |
| Parent | `t-inherit` target, mode, XPath |
| Data | Context/props/state variables dùng trong template |
| Directives | `t-if`, `t-foreach`, `t-out`, `t-att`, `t-call`, `t-set` |
| Safety | Nguồn HTML, escaping/sanitizing, không dùng raw user input |

## OWL Template Pattern
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="my_module.MyDashboard">
        <div class="o_my_dashboard">
            <t t-if="state.loading">
                <span>Loading</span>
            </t>
            <t t-else="">
                <t t-foreach="state.records" t-as="record" t-key="record.id">
                    <button type="button" t-on-click="() => this.openRecord(record.id)">
                        <span t-out="record.name"/>
                    </button>
                </t>
            </t>
        </div>
    </t>
</templates>
```

## Template Inheritance Pattern
```xml
<templates xml:space="preserve">
    <t t-inherit="base.TemplateName" t-inherit-mode="extension">
        <xpath expr="//div[contains(@class, 'target')]" position="inside">
            <span t-out="props.extraLabel"/>
        </xpath>
    </t>
</templates>
```

## References
- Odoo 19 QWeb frontend: https://www.odoo.com/documentation/19.0/developer/reference/frontend/qweb.html
- Odoo 19 Owl components: https://www.odoo.com/documentation/19.0/developer/reference/frontend/owl_components.html
