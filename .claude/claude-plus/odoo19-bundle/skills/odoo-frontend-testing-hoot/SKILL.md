---
name: odoo-frontend-testing-hoot
description: Hướng dẫn JavaScript unit testing frontend Odoo 19 - HOOT, web test helpers, mock server, web.assets_unit_tests, static/tests và /web/tests. Use when writing or specifying tests for Owl components, services, registries, field widgets, or client actions.
---

# Odoo Frontend Testing HOOT

## Goal
Giúp agent đặc tả và viết test frontend Odoo 19 cho OWL/component/service/registry bằng HOOT — test stack chính thức của Odoo 19 thay thế QUnit.

## When to use this skill
- Test OWL component, field widget, client action.
- Test frontend service/registry behavior.
- Cần mock server/RPC/ORM call.
- Cần mount view và assert DOM state.
- Cần thêm test assets vào manifest (`web.assets_unit_tests`).
- Viết FRD section cho JS unit tests.
- Người dùng hỏi về "HOOT", "web test helpers", "mock server", "onRpc", "mountWithCleanup", "mountView", "defineModels".

## Instructions

### Bước 1 — Đọc source trước khi viết test
Dùng `odoo-frontend-source-analysis` (hoặc đọc trực tiếp) để hiểu component/service cần test và xem test hiện có trong `static/tests/`. Không đề xuất test mà không biết code đang làm gì.

### Bước 2 — Xác định unit under test
Xác định rõ:
- Component / service / registry / field widget / client action nào cần test.
- Mock data: records, RPC responses, action params.
- User flow: render → interaction → DOM/state assertion.
- Error flow: RPC error, empty state, permission failure.

### Bước 3 — Cấu trúc test file
Đặt test file tại `static/tests/`, dùng extension `.test.js` hoặc `.hoot.js`.

Import từ đúng nguồn (xem GUIDE.md):
- Framework: `@odoo/hoot`
- DOM helpers: `@odoo/hoot` (KHÔNG dùng `@odoo/hoot-dom` riêng, KHÔNG dùng `@odoo/hoot-mock` — đã deprecated)
- Web test helpers: `@web/../tests/web_test_helpers` (một entry point duy nhất)

### Bước 4 — Khai báo models và data
Dùng `defineModels([...])` với class extend `models.Model`. Khai báo fields dạng class property, `_records`, `_views`. Xem GUIDE.md - section "Định nghĩa model test".

### Bước 5 — Mock RPC / service
- `onRpc(model, method, callback)` — intercept ORM call.
- `onRpc(route, callback)` — intercept route.
- `mockService(name, factory)` — thay thế service trong test.

### Bước 6 — Mount và assert
- Component tự do: `mountWithCleanup(Component, options)` — auto-destroy sau mỗi test.
- View Odoo: `mountView({ type, resModel, resId, ... })`.
- Assert DOM: `queryOne()`, `queryAll()`, `queryText()`, `expect(...).toBe(...)`.
- Await async: `await animationFrame()`, `await runAllTimers()`.

### Bước 7 — Thêm vào manifest
```python
"assets": {
    "web.assets_unit_tests": [
        "my_module/static/tests/**/*",
    ],
}
```

### Bước 8 — Chạy test
Truy cập `/web/tests` trên browser, hoặc Debug menu → Run Unit Tests.

## Constraints
- **HOOT là framework chính** của Odoo 19. QUnit chỉ còn trong `addons/web/static/tests/legacy/` — không dùng QUnit cho code mới.
- **`@odoo/hoot-mock` đã deprecated hoàn toàn** — import `advanceFrame`, `animationFrame`, `mockDate`, v.v. trực tiếp từ `@odoo/hoot`.
- **Entry point duy nhất** cho web test helpers: `@web/../tests/web_test_helpers`.
- **`mountWithCleanup`** tự destroy Owl App sau mỗi test — không cần cleanup thủ công.
- **`defineModels()`** tự đăng ký vào `before()` hook — không setup thủ công.
- Không dùng `name_get()`, `read_group()` deprecated API trong mock model.
- Test file phải được include trong `web.assets_unit_tests` để runner nhận.

## References
- Odoo 19 JavaScript Unit Testing: https://www.odoo.com/documentation/19.0/developer/reference/frontend/unit_testing.html
- Chi tiết code patterns và API reference: xem `references/GUIDE.md`
