---
name: odoo-frontend-javascript-owl
description: Hướng dẫn JavaScript/Owl frontend cho Odoo 19 - native JS modules, Owl components, client actions, field widgets, custom view types, registries, services, hooks và frontend integration. Use when building or specifying custom Odoo web client UI in FRD or code.
---

# Odoo Frontend JavaScript & Owl

## Goal
Giúp agent viết FRD/code frontend Odoo 19 đúng web framework: Owl component, registry, service, hook, QWeb template và assets. Không dùng legacy widget API cho development mới.

## Source-first khi viết FRD
Trước khi đặc tả OWL/client action/widget, dùng `odoo-frontend-source-analysis` để đọc `__manifest__.py`, `static/src`, registry keys, services/hooks và templates hiện có. Không thêm block OWL vào FRD nếu source/requirement chỉ là XML view chuẩn.

## Khi nào dùng
- Custom OWL component, field widget, client action, dashboard, matrix/grid.
- Đăng ký `registry.category("actions"|"fields"|"views"|...)`.
- Gọi Python model bằng `orm` service hoặc controller bằng `rpc`.
- Cần mô tả frontend trong FRD: props/context, state, interactions, API calls, error handling.

## Quy tắc Odoo 19
- Component dùng `setup()`, không dùng constructor.
- Template name theo `addon_name.ComponentName` để tránh collision.
- Dùng `useService()` trong component để gọi `orm`, `notification`, `action`, `dialog`, `rpc`, `user`.
- JS trong `static/src` được Odoo xử lý như native JS module; nếu file nằm ngoài vùng auto-transpile hoặc project convention yêu cầu, thêm `/** @odoo-module **/`.
- Asset JS/XML/SCSS phải được khai báo bằng `assets` trong `__manifest__.py`, thường ở `web.assets_backend`.

## FRD Checklist Cho Component
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Entry point | Menu/action/button/field widget/custom view |
| Component | Class name, template name, JS/XML/SCSS path |
| Registry | Category, key, sequence nếu có |
| Props/context | Input từ action params, active_id, record, field props |
| State | State fields, default, lifecycle load |
| Services/hooks | `useService`, `useState`, `onWillStart`, `useBus`, `useAssets`, ... |
| API calls | Model/method hoặc route, payload, return, error handling |
| UX states | loading, empty, readonly/edit, error, mobile behavior |

## Client Action Pattern
```javascript
/** @odoo-module **/
import { Component, useState, onWillStart } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

export class MyDashboard extends Component {
    static template = "my_module.MyDashboard";

    setup() {
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.state = useState({ records: [], loading: true });
        onWillStart(async () => {
            this.state.records = await this.orm.searchRead("my.model", [], ["name", "state"]);
            this.state.loading = false;
        });
    }
}

registry.category("actions").add("my_module.MyDashboard", MyDashboard);
```

```xml
<record id="action_my_dashboard" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.MyDashboard</field>
    <field name="target">current</field>
</record>
```

## Tách skill liên quan
- Assets/lazy loading: `odoo-frontend-assets`.
- Registries/services chi tiết: `odoo-frontend-services-registries`.
- QWeb template: `odoo-frontend-qweb-templates`.
- JS testing: `odoo-frontend-testing-hoot`.
- Patching/error handling: `odoo-frontend-patching-error-handling`.

## References
- Odoo 19 Framework overview: https://www.odoo.com/documentation/19.0/developer/reference/frontend/framework_overview.html
- Odoo 19 JavaScript modules: https://www.odoo.com/documentation/19.0/developer/reference/frontend/javascript_modules.html
- Odoo 19 Owl components: https://www.odoo.com/documentation/19.0/developer/reference/frontend/owl_components.html
- Odoo 19 JavaScript reference: https://www.odoo.com/documentation/19.0/developer/reference/frontend/javascript_reference.html
