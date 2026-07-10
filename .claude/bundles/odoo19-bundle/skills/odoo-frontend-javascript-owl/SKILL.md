---
name: odoo-frontend-javascript-owl
description: Hướng dẫn JavaScript/Owl frontend cho Odoo 19 - native JS modules, Owl components, client actions, field widgets, custom view types, registries, services, hooks và frontend integration. Use when building or specifying custom Odoo web client UI in FRD or code.
---

# Odoo Frontend JavaScript & Owl

## Goal
Giúp agent viết FRD/code frontend Odoo 19 đúng web framework: Owl component, registry, service, hook, QWeb template và assets. Không dùng legacy widget API cho development mới.

## When to use this skill
- Custom OWL component, field widget, client action, dashboard, matrix/grid
- Đăng ký `registry.category("actions"|"fields"|"views"|...)`
- Gọi Python model bằng `orm` service hoặc controller bằng `rpc`
- Cần mô tả frontend trong FRD: props/context, state, interactions, API calls, error handling
- Khi gặp trigger: "viết Owl component", "tạo client action", "custom widget", "dashboard OWL", "đăng ký registry", "useService", "frontend Odoo"

## Instructions

### Bước 1 — Source-first (trước khi viết FRD hoặc code)
1. Dùng `odoo-backend-source-analysis` (hoặc đọc trực tiếp) để đọc `__manifest__.py`, `static/src`, registry keys, services/hooks và templates hiện có.
2. Không thêm block OWL vào FRD nếu source/requirement chỉ là XML view chuẩn.
3. Xác định đúng loại extension cần làm: client action / field widget / custom view / patch.

### Bước 2 — Xác định entry point và registry
- **Client action**: đăng ký `registry.category("actions").add("tag", Component)`
- **Field widget**: đăng ký `registry.category("fields").add("widget_name", Widget)`
- **Custom view**: đăng ký `registry.category("views").add("type", ViewClass)`
- **Systray item**: đăng ký `registry.category("systray").add("key", { Component })`

### Bước 3 — Viết Component theo chuẩn Odoo 19
- Dùng `setup()` — không dùng constructor để khởi tạo services/state
- Template name theo pattern `addon_name.ComponentName`
- Import từ `@odoo/owl` cho Owl primitives, từ `@web/...` cho framework utilities
- Tham khảo patterns chi tiết trong `references/GUIDE.md`

### Bước 4 — Khai báo assets trong `__manifest__.py`
```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/js/my_component.js',
        'my_module/static/src/xml/my_template.xml',
        'my_module/static/src/scss/my_style.scss',
    ],
}
```
- Files trong `static/src` được auto-transpile — không cần `/** @odoo-module **/`
- Files ngoài `static/src` cần thêm `/** @odoo-module **/` ở đầu file

### Bước 5 — FRD Checklist cho Component
Xem bảng đầy đủ trong `references/GUIDE.md#frd-checklist`

## Constraints

**Bất biến Odoo 19 (đã xác minh từ docs chính thức):**
- KHÔNG dùng legacy `Widget` class (AbstractWidget/web.Widget) từ Odoo < 17 — đã bị xóa hoàn toàn
- KHÔNG dùng `odoo.define()` cho code mới — legacy, dùng native ES6 `import/export`
- KHÔNG dùng deprecated API: `name_get()`, `read_group()`, thẻ `<tree>`, `attrs`, `_cr/_uid/_context`
- KHÔNG dùng constructor để inject services — chỉ dùng `setup()` với `useService()`
- KHÔNG raw SQL — dùng ORM hoặc `SQL()` wrapper
- Template name PHẢI theo format `addon_name.ComponentName` để tránh collision
- Testing: dùng **HOOT** framework — KHÔNG dùng QUnit (deprecated từ Odoo 19)
- QWeb JS inheritance system cũ đã deprecated — dùng cơ chế template mới

**Module system (đã xác minh):**
- Files trong `/static/src` hoặc `/static/tests`: auto-transpile, không cần directive
- Files ngoài 2 thư mục trên: thêm `/** @odoo-module **/` để opt-in transpile
- Để tắt transpile (thư viện ngoài): thêm `/** @odoo-module ignore **/`
- Cross-addon import: dùng full path `@other_addon/path/to/file`

**Built-in Owl components (import từ `@web/core`):**
- ActionSwiper, CheckBox (hỗ trợ `indeterminate` prop — mới Odoo 19), ColorList
- Dropdown / DropdownItem / DropdownGroup, Notebook, Pager, SelectMenu, TagsList
- Dropdown mặc định `bottomSheet=true` — render bottom sheet trên mobile touch device

## References
- Framework overview: https://www.odoo.com/documentation/19.0/developer/reference/frontend/framework_overview.html
- JavaScript modules: https://www.odoo.com/documentation/19.0/developer/reference/frontend/javascript_modules.html
- Owl components: https://www.odoo.com/documentation/19.0/developer/reference/frontend/owl_components.html
- JavaScript reference: https://www.odoo.com/documentation/19.0/developer/reference/frontend/javascript_reference.html
