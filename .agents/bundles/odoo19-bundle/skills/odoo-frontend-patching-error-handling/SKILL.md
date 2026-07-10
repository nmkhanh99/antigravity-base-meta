---
name: odoo-frontend-patching-error-handling
description: Hướng dẫn patching và error handling frontend Odoo 19 - @web/core/utils/patch, super, patch prototype, unpatch trong tests, JS/Owl error lifecycle, RPC/service error handling. Use when extending existing web client behavior or specifying robust frontend error handling.
---

# Odoo Frontend Patching & Error Handling

## Goal
Giúp agent patch frontend Odoo 19 đúng cách và đặc tả error handling rõ trong FRD/frontend code.

## Source-first khi viết FRD
Trước khi đề xuất patch, dùng `odoo-frontend-source-analysis` để đọc target class/object/prototype và các patch hiện có. FRD phải nêu source target, method bị patch, blast radius và regression test.

## Khi nào dùng
- Cần mở rộng behavior component/service/view có sẵn mà không thay core.
- Patching class/object/prototype.
- Xử lý lỗi RPC/ORM/service/component lifecycle.
- Viết test cần unpatch sau test.

## Patching Rules
- Import `patch` từ `@web/core/utils/patch`.
- Patch class instance methods qua `ClassName.prototype`.
- Dùng method syntax để gọi `super`; không dùng arrow/function property nếu cần `super`.
- Không patch trực tiếp constructor; patch `setup()` hoặc method được constructor gọi.
- `patch()` trả về `unpatch`, dùng trong tests để cleanup.

```javascript
import { patch } from "@web/core/utils/patch";
import { SomeComponent } from "@web/some/path";

patch(SomeComponent.prototype, {
    setup() {
        super.setup(...arguments);
        // custom hook/service setup
    },
});
```

## Error Handling Rules
- Throw `Error` object, không throw primitive.
- Với user-facing errors, hiển thị notification/dialog có message rõ.
- Với RPC/model call, mô tả catch/fallback/loading reset.
- Không để module throw ngay lúc definition nếu có thể trì hoãn sang lifecycle có kiểm soát.

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Patch target | Object/class/prototype/module path |
| Method | Method bị patch, lý do, call `super` hay thay thế |
| Blast radius | View/component nào bị ảnh hưởng |
| Error cases | RPC error, validation error, missing asset, permission |
| User feedback | Notification/dialog/empty/error state |
| Tests | Patch cleanup/unpatch, regression scenario |

## References
- Odoo 19 Patching code: https://www.odoo.com/documentation/19.0/developer/reference/frontend/patching_code.html
- Odoo 19 Error handling: https://www.odoo.com/documentation/19.0/developer/reference/frontend/error_handling.html
