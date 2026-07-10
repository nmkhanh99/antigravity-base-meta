# Odoo 19 Frontend Patching & Error Handling — Reference Guide

## §1 Patching Patterns

### 1.1 Import
```js
import { patch } from "@web/core/utils/patch";
```

### 1.2 Patch object đơn giản (POJO)
```js
patch(mySimpleObject, {
    myNewMethod() {
        return "new value";
    },
});
```

### 1.3 Patch instance methods — PHẢI dùng `.prototype`
```js
import { SomeComponent } from "@web/some/path";

patch(SomeComponent.prototype, {
    setup() {
        super.setup(...arguments);   // ES6 super — KHÔNG dùng _super
        this.myField = "value";
    },
    myMethod() {
        return super.myMethod() + " patched";
    },
    get myGetter() {
        return super.myGetter + " extra";
    },
});
```

### 1.4 Patch static methods/fields — patch class trực tiếp (KHÔNG phải .prototype)
```js
patch(MyClass, {
    staticFn() {
        super.staticFn();
    },
    staticField: "new value",
});
```

### 1.5 Lưu unpatch function để cleanup
```js
const unpatch = patch(MyClass.prototype, {
    myMethod() { return "patched"; },
});
// Hoàn tác patch:
unpatch();
```

### 1.6 Multiple patches — stack đúng thứ tự
```js
const unpatch1 = patch(MyClass.prototype, {
    fn() { return super.fn() + " patch1"; },
});
const unpatch2 = patch(MyClass.prototype, {
    fn() { return super.fn() + " patch2"; },
});
// Call order khi gọi instance.fn(): patch2 → patch1 → original
// Gỡ patch1, patch2 vẫn hoạt động:
unpatch1();
```

### 1.7 Patch Owl component
```js
import { patch } from "@web/core/utils/patch";
import { MyComponent } from "@my_module/components/my_component";
import { useState } from "@odoo/owl";

patch(MyComponent.prototype, {
    setup() {
        super.setup(...arguments);
        this.extraState = useState({ value: 0 });
    },
    onWillStart() {
        return super.onWillStart(...arguments);
    },
});
```

### 1.8 Test helper — auto-cleanup (dùng trong @odoo/hoot tests)
```js
import { patchWithCleanup } from "@web/../tests/web_test_helpers";
// Hoặc:
// import { patchWithCleanup } from "@web/static/tests/_framework/patch_test_helpers.js";

patchWithCleanup(MyClass.prototype, {
    myMethod() {
        return "mocked";
    },
});
// Tự động unpatch sau mỗi test — không cần teardown thủ công
```

### 1.9 Patch getter/setter — Odoo 19 tự merge accessor còn thiếu
```js
// Chỉ patch getter: Odoo 19 tự copy setter từ ancestor để tránh descriptor corruption
patch(MyClass.prototype, {
    get myProp() {
        return super.myProp + " extra";
    },
    // setter KHÔNG cần khai báo nếu không muốn override
});
```

---

## §2 Error Handling Patterns

### 2.1 Registry-based handler (cách khuyến nghị Odoo 19)
```js
import { registry } from "@web/core/registry";

registry.category("error_handlers").add("myModuleHandler", (env, error, originalError) => {
    if (error instanceof MySpecificError) {
        // Xử lý lỗi, hiển thị notification...
        env.services.notification.add("Có lỗi xảy ra: " + error.message, {
            type: "danger",
        });
        return true; // đánh dấu đã handled, dừng propagation
    }
    // Không return true → lỗi tiếp tục tới handler khác
});
```

### 2.2 Component-level error boundary (Owl onError)
```js
import { Component, onError } from "@odoo/owl";

class MyComponent extends Component {
    setup() {
        super.setup(...arguments);

        onError((error) => {
            console.error("Component error:", error);
            this.state.hasError = true;
            // Ngăn lỗi leo lên component cha
        });
    }
}
```

### 2.3 Error-free control flow (ưu tiên hơn throw)
```js
// KHÔNG khuyến nghị:
function getRecord(id) {
    if (!id) throw new Error("id is required");
    return fetchRecord(id);
}

// KHUYẾN NGHỊ:
function getRecord(id) {
    if (!id) return null; // caller kiểm tra null
    return fetchRecord(id);
}

// Hoặc dùng status object:
function processData(data) {
    if (!data) return { ok: false, reason: "missing_data" };
    return { ok: true, result: transform(data) };
}
```

### 2.4 RPC/model call với error handling
```js
import { useService } from "@web/core/utils/hooks";
import { useState } from "@odoo/owl";

class MyComponent extends Component {
    setup() {
        this.rpc = useService("rpc");
        this.notification = useService("notification");
        this.state = useState({ isLoading: false, data: null });
    }

    async loadData() {
        this.state.isLoading = true;
        try {
            this.state.data = await this.rpc("/my_module/get_data", { param: 1 });
        } catch (error) {
            // Reset loading state
            this.state.isLoading = false;
            this.notification.add("Không thể tải dữ liệu", { type: "danger" });
            // KHÔNG re-throw trừ khi cần caller xử lý tiếp
            return;
        }
        this.state.isLoading = false;
    }
}
```

### 2.5 Throw Error object đúng cách (khi thực sự cần throw)
```js
// ĐÚNG — throw Error object:
throw new Error("Thiếu thông tin bắt buộc");
throw new TypeError("Giá trị không hợp lệ");

// SAI — throw primitive:
throw "Có lỗi";   // Không làm thế này
throw 404;        // Không làm thế này
```

---

## §3 FRD Checklist — Patch & Error Handling

Khi viết FRD cho tính năng frontend cần patch hoặc error handling, bắt buộc mô tả đủ các hạng mục sau:

| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| **Patch target** | Object/class/prototype/module path đầy đủ (ví dụ: `FormController.prototype`) |
| **Method bị patch** | Tên method, lý do patch, có gọi `super` không, thay thế hay mở rộng |
| **Blast radius** | View/component nào bị ảnh hưởng khi patch được apply |
| **Error cases** | RPC error, validation error, missing asset, permission denied, network timeout |
| **User feedback** | Notification type (danger/warning/info), dialog content, empty state, loading state |
| **Tests** | Dùng `patchWithCleanup` hay `patch + unpatch`, regression scenario cần cover |
| **Cleanup** | Khi nào unpatch được gọi (test teardown, module unload, never) |

### Ví dụ đặc tả FRD:

```
#### F-012: Override FormController để thêm validation trước khi save

**Patch target**: `@web/views/form/form_controller` → `FormController.prototype`
**Method**: `beforeSave()` — mở rộng (gọi `super`), không thay thế
**Blast radius**: Tất cả Form view trong module `my_module`
**Error cases**:
  - API trả về lỗi validation → hiển thị notification danger "Dữ liệu không hợp lệ: {message}"
  - Network timeout → reset loading state, hiển thị notification warning
**User feedback**: Notification danger khi fail, không block UI khi loading
**Tests**: `patchWithCleanup(FormController.prototype, ...)`, test case: save thành công, save fail validation, save fail network
```

---

## §4 Breaking Changes Odoo 19 — Summary

| Vấn đề | Odoo ≤ 18 | Odoo 19 |
|--------|-----------|---------|
| Import patch | Nhiều path khác nhau | `@web/core/utils/patch` (duy nhất) |
| Tham số thứ 2 của patch() | String tên patch | Object extension (string → throw Error) |
| Gọi method gốc | `this._super(...)` | ES6 `super.methodName(...)` |
| Unpatch trong test | `unpatch()` export riêng | Lưu return value: `const unpatch = patch(...)` |
| patchWithCleanup path | Legacy helpers | `@web/../tests/web_test_helpers` |
| Test framework | QUnit | `@odoo/hoot` |
| Accessor (get/set) | Phải khai báo cả đôi | Tự động merge accessor còn thiếu từ ancestor |
