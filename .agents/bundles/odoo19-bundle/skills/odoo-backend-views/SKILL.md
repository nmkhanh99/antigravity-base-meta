---
name: odoo-backend-views
description: Đặc tả và triển khai Odoo 19 XML/UI views phía backend - view records, view architectures, form/list/kanban/search/calendar/graph/pivot, XPath inheritance, modifiers, actions và menus. Use when writing FRD or code for standard Odoo XML views, view inheritance, action/view_mode, search filters, or server-rendered UI.
---

# Odoo Backend Views

## Goal
Giúp agent viết FRD hoặc code view XML đúng Odoo 19, ưu tiên native view trước khi đề xuất custom OWL. Dùng nguồn chính: Odoo 19 View records và View architectures.

## Source-first khi viết FRD
Trước khi đặc tả view, dùng `odoo-backend-source-analysis` để đọc view/action/menu hiện có trong Odoo base và custom module. FRD phải ghi XML ID, file hoặc XPath thực tế nếu quyết định inherit/modify view hiện có.

## Khi nào dùng
- Tạo/sửa form, list, kanban, search, calendar, graph, pivot.
- Đặc tả `ir.ui.view`, `ir.actions.act_window`, menu, button object/action.
- Inherit view bằng `inherit_id`, XPath, `position`.
- Viết modifier Odoo 19: `readonly`, `required`, `invisible`, `column_invisible`, `domain`, `context`.

## Quy tắc Odoo 19
- List view dùng `<list>`, không dùng `<tree>` cho code/spec mới.
- Không dùng `attrs="{...}"`; dùng Python expression trực tiếp trên `readonly`, `required`, `invisible`, `column_invisible`.
- Nếu expression dùng field kỹ thuật, field đó phải hiện diện trong view, có thể `invisible="True"`.
- Luôn ghi rõ `View XML ID`, `model`, `inherit_id`, XPath/locator, `position`, group, context/domain nếu có.
- Đặt `name` ổn định cho `page`, `group`, smart button container để dễ inherit.

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| View metadata | File, View XML ID, Model, View type, Inherit XML ID |
| Inheritance | XPath/locator, `position`, node được thêm/sửa/xóa |
| Modifiers | Điều kiện `invisible`, `readonly`, `required`, `column_invisible` |
| Data behavior | `domain`, `context`, default/search context, group_by |
| Access | `groups`, menu/action group, field security liên quan |
| Actions | `ir.actions.act_window`, `view_mode`, `domain`, `context`, target |

## Mẫu View Inherit
```xml
<record id="view_my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="base_module.view_my_model_form"/>
    <field name="arch" type="xml">
        <xpath expr="//group[@name='main_group']" position="inside">
            <field name="x_state_reason" invisible="state != 'cancel'"/>
        </xpath>
    </field>
</record>
```

## Mẫu Search
```xml
<search>
    <field name="name" filter_domain="['|', ('name', 'ilike', self), ('code', 'ilike', self)]"/>
    <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
    <separator/>
    <filter name="group_state" string="Status" context="{'group_by': 'state'}"/>
</search>
```

## Khi nào chuyển sang frontend skill
- Nếu chỉ là XML view chuẩn: dùng skill này.
- Nếu cần custom field widget, client action, custom view type, dashboard, matrix/grid: dùng thêm `odoo-frontend-javascript-owl`, `odoo-frontend-assets`, `odoo-frontend-services-registries`.
- Nếu có QWeb template custom trong kanban/OWL/website: dùng thêm `odoo-frontend-qweb-templates`.

## References
- Odoo 19 View records: https://www.odoo.com/documentation/19.0/developer/reference/user_interface/view_records.html
- Odoo 19 View architectures: https://www.odoo.com/documentation/19.0/developer/reference/user_interface/view_architectures.html
