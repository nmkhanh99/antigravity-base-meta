---
name: odoo-frontend-services-registries
description: Đặc tả Odoo 19 frontend registries và services - actions, fields, views, services, main_components, systray, useService, service dependencies và async behavior. Use when writing FRD/code for custom Odoo web client extension points.
---

# Odoo Frontend Services & Registries

## Goal
Giúp agent chọn đúng extension point của Odoo 19 web client và mô tả registry/service trong FRD.

## Source-first khi viết FRD
Trước khi đặc tả registry/service, dùng `odoo-frontend-source-analysis` để tìm `registry.category`, service `dependencies/start`, `useService` và callers hiện có. FRD phải ghi registry category/key thực tế nếu modify/extend source hiện có.

## Khi nào dùng
- Client action: `registry.category("actions")`.
- Field widget: `registry.category("fields")`.
- Custom view type: `registry.category("views")`.
- Long-lived side effect/shared state: service registry.
- Systray/main component/user menu/effects.

## Registry Categories Thường Gặp
| Category | Dùng cho |
|----------|----------|
| `actions` | `ir.actions.client` tag |
| `fields` | Field widget trong form/list/kanban |
| `views` | Custom view type/controller/renderer |
| `services` | Long-lived service, DI dependencies |
| `main_components` | Top-level webclient component |
| `systray` | Item trên navbar systray |
| `effects` | Custom visual effects |

## Service Rules
- Component dùng service qua `useService()`.
- Service khai báo `dependencies` và `start(env, deps)`.
- Code có side effect hoặc shared state dài hạn nên đóng gói service để dễ test.
- Với async service, đặc tả behavior khi component bị destroy trước khi promise kết thúc.

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Extension point | Registry category/key |
| Owner file | JS path khai báo registry/service |
| Payload | Props/action params/field props/service API |
| Lifecycle | Khi start/load/unmount, async handling |
| Dependencies | Services khác, module dependency, assets |
| Error handling | Notification/dialog/logging/fallback |
| Tests | Mock service/registry behavior |

## Patterns
```javascript
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

registry.category("actions").add("my_module.MyAction", MyAction);

export const myService = {
    dependencies: ["notification"],
    start(env, { notification }) {
        return {
            warn(message) {
                notification.add(message, { type: "warning" });
            },
        };
    },
};

registry.category("services").add("my_module.my_service", myService);
```

## References
- Odoo 19 Registries: https://www.odoo.com/documentation/19.0/developer/reference/frontend/registries.html
- Odoo 19 Services: https://www.odoo.com/documentation/19.0/developer/reference/frontend/services.html
- Odoo 19 Hooks: https://www.odoo.com/documentation/19.0/developer/reference/frontend/hooks.html
