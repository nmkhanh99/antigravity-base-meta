---
name: odoo-frontend-patching-error-handling
description: Hướng dẫn patching và error handling frontend Odoo 19 - @web/core/utils/patch, super, patch prototype, unpatch trong tests, JS/Owl error lifecycle, RPC/service error handling. Use when extending existing web client behavior or specifying robust frontend error handling.
---

# Odoo Frontend Patching & Error Handling

## Goal
Giúp agent patch frontend Odoo 19 đúng cách và đặc tả error handling rõ trong FRD/frontend code — không sửa core, không để lỗi lộ ra ngoài lifecycle có kiểm soát.

## When to use this skill
- Cần mở rộng behavior component/service/view có sẵn mà không thay core
- Patch class/object/prototype trong Odoo 19 web client
- Xử lý lỗi RPC/ORM/service/component lifecycle
- Viết test cần cleanup patch sau mỗi test case
- Đặc tả FRD cho tính năng frontend cần patch hoặc error boundary
- Kích hoạt khi user nói: "patch component", "override method frontend", "error handling JS", "unpatch test", "onError Owl", "error_handlers registry"

## Instructions

### Bước 1 — Source analysis trước khi patch
Dùng `odoo-backend-source-analysis` hoặc đọc trực tiếp file JS để xác định:
- Target object/class/prototype cần patch
- Method signature, có gọi `super` không
- Các patch hiện có (tránh conflict)

### Bước 2 — Áp dụng Patching Rules
Xem GUIDE.md §1 để chọn đúng pattern (object, prototype, static, Owl component).

Quy tắc cứng:
- Import `patch` từ `@web/core/utils/patch` (duy nhất, không có alias khác)
- Patch instance methods qua `ClassName.prototype`, KHÔNG patch class trực tiếp
- Dùng ES6 `super` — KHÔNG dùng `_super` (đã bỏ từ Odoo 17+)
- Tham số thứ hai của `patch()` là **object extension**, KHÔNG phải string tên patch
- `patch()` trả về `unpatch` function — lưu lại để cleanup trong tests

### Bước 3 — Error Handling
Xem GUIDE.md §2 để chọn đúng pattern (registry handler, onError, error-free flow).

Quy tắc cứng:
- Đăng ký custom handler qua `registry.category("error_handlers")` — không dùng try/catch toàn cục
- Component-level error boundary dùng `onError` callback của Owl
- Ưu tiên return null/status object thay vì throw khi có thể
- Không throw ở top-level module definition; trì hoãn sang lifecycle có kiểm soát

### Bước 4 — FRD Checklist
Khi viết FRD có liên quan patch/error, bắt buộc điền đủ các hạng mục trong GUIDE.md §3.

### Bước 5 — Tests
- Dùng `patchWithCleanup` từ `@web/../tests/web_test_helpers` (tự cleanup sau mỗi test)
- Nếu dùng `patch()` thủ công trong test, gọi `unpatch()` trong teardown

## Constraints
- BREAKING Odoo 19: `patch(obj, "name", extension)` ném lỗi — tham số thứ hai PHẢI là object
- `_super` không còn tồn tại — phải dùng ES6 `super`
- `patchWithCleanup` nằm tại `@web/../tests/web_test_helpers`, KHÔNG phải legacy helpers path
- Test framework là `@odoo/hoot`, không phải QUnit
- Không patch constructor trực tiếp; patch `setup()` hoặc method được constructor gọi
- Không throw primitive (string, number) — luôn throw `Error` object hoặc subclass
- Catch-all error handler bị discourage — handler phải target specific error type

## References
- Patching Code: https://www.odoo.com/documentation/19.0/developer/reference/frontend/patching_code.html
- Error Handling: https://www.odoo.com/documentation/19.0/developer/reference/frontend/error_handling.html
