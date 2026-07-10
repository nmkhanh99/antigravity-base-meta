---
name: odoo-frontend-services-registries
description: Đặc tả Odoo 19 frontend registries và services - actions, fields, views, services, main_components, systray, useService, service dependencies và async behavior. Use when writing FRD/code for custom Odoo web client extension points.
---

# Odoo Frontend Services & Registries

## Goal
Giúp agent chọn đúng extension point của Odoo 19 web client, đặc tả registry/service trong FRD, và viết code đúng API Odoo 19 (không dùng API cũ từ Odoo < 17).

## When to use this skill
- Viết FRD cho tính năng mở rộng web client Odoo 19
- Thêm client action (`ir.actions.client` tag)
- Tạo custom field widget trong form/list/kanban
- Tạo custom view type
- Viết long-lived service có shared state hoặc side effect
- Thêm systray item, main component, user menu item, custom effect
- Cần `useService()` trong OWL component
- Cần hooks: `useBus`, `usePosition`, `useAutofocus`, `useSpellCheck`, `usePager`

## Instructions

### Bước 1 — Source-first analysis
Trước khi đặc tả, dùng `odoo-backend-source-analysis` để tìm:
- `registry.category(...)` calls hiện có trong module
- Service `dependencies` và `start()` signatures
- `useService(...)` callers trong component liên quan
- FRD phải ghi đúng registry category/key thực tế nếu modify/extend source hiện có.

### Bước 2 — Chọn đúng extension point
Xem bảng Registry Categories trong `GUIDE.md`. Quy tắc nhanh:
- Client action → `registry.category("actions")`
- Field widget → `registry.category("fields")`
- Custom view type → `registry.category("views")`
- Long-lived shared state/side effect → `registry.category("services")`
- Top-level component → `registry.category("main_components")`
- Navbar icon → `registry.category("systray")`
- User dropdown menu → `registry.category("user_menuitems")`
- Custom visual effect → `registry.category("effects")`

### Bước 3 — Đặc tả trong FRD
Điền đủ FRD Checklist (xem `GUIDE.md`):
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Extension point | Registry category + key |
| Owner file | JS path khai báo registry/service |
| Payload | Props / action params / field props / service API |
| Lifecycle | Khi start/load/unmount, async handling |
| Dependencies | Services khác, module dependency, assets bundle |
| Error handling | Notification / dialog / logging / fallback |
| Tests | Mock service/registry behavior |

### Bước 4 — Viết code theo pattern Odoo 19
Xem code patterns chi tiết trong `GUIDE.md`.

**Lưu ý Odoo 19 bắt buộc:**
- RPC: import trực tiếp `rpc` từ `@web/core/network/rpc`, KHÔNG dùng `useService('rpc')`.
- User info: import `user` từ `@web/core/user` (plain reactive object), KHÔNG dùng service.
- ORM: dùng `useService('orm')` — có `orm.silent.read(...)` để suppress error notification.
- Route URL: path-based `/odoo/action-{id}/{resId}`, KHÔNG dùng hash `#action=...`.
- Systray component: root element phải là `<li>` tag.
- `registry.add(key, value)` throw nếu key đã tồn tại — dùng `{ force: true }` khi override.
- Service `start(env, deps)` là signature bắt buộc — KHÔNG dùng `_start` hay legacy format.
- `useService()` tự động wrap reactive object với `useState()` và bảo vệ async methods khi component bị destroy.

## Constraints
- KHÔNG dùng deprecated API từ Odoo < 17: `odoo.define()`, AMD module path `web.registry`, `name_get()`, `read_group()`, `<tree>`, `attrs`.
- KHÔNG dùng legacy widget system — chỉ dùng OWL components.
- KHÔNG pass HTML Element vào Effect service `message` param — chỉ dùng string.
- KHÔNG dùng cache và xhr options đồng thời trong RPC settings.
- Tất cả hooks (`useService`, `useBus`, v.v.) phải gọi trong `setup()` method.
- Import paths phải dùng `@web/` aliases, KHÔNG dùng đường dẫn tương đối đến core.

## References
- Odoo 19 Registries: https://www.odoo.com/documentation/19.0/developer/reference/frontend/registries.html
- Odoo 19 Services: https://www.odoo.com/documentation/19.0/developer/reference/frontend/services.html
- Odoo 19 Hooks: https://www.odoo.com/documentation/19.0/developer/reference/frontend/hooks.html
