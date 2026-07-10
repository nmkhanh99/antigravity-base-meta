# Odoo Frontend JavaScript & Owl — Reference Guide

## FRD Checklist cho Component {#frd-checklist}

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

---

## Module System — Ba dạng hỗ trợ

```javascript
// 1. Native ES6 module (KHUYẾN NGHỊ cho Odoo 19)
// File: my_addon/static/src/my_file.js — auto-transpile, không cần directive
import { Component, useState } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";
export class MyComponent extends Component { ... }

// 2. Opt-in transpile (file ngoài /static/src hoặc /static/tests)
/** @odoo-module **/
import { x } from "./x";
export const y = x + 1;

// 3. Tắt transpile (thư viện ngoài / plain JS)
/** @odoo-module ignore **/
(function () { var myVar = 42; })();

// 4. Aliased module (backward compat với old-style name)
/** @odoo-module alias=web.someName **/
export default function legacyCompat(val) { return val * 2; }

// 5. Cross-addon import — LUÔN dùng full path
import { Dialog } from "@web/core/dialog/dialog";
import { SomeThing } from "@other_addon/path/to/file";
```

---

## Client Action Pattern (chuẩn Odoo 19)

```javascript
// my_module/static/src/js/my_dashboard.js
import { Component, useState, onWillStart } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

export class MyDashboard extends Component {
    static template = "my_module.MyDashboard";

    setup() {
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.action = useService("action");
        this.state = useState({ records: [], loading: true, error: null });

        onWillStart(async () => {
            try {
                this.state.records = await this.orm.searchRead(
                    "my.model",
                    [],
                    ["name", "state", "date"],
                    { limit: 50 }
                );
            } catch (e) {
                this.state.error = e.message;
                this.notification.add("Lỗi tải dữ liệu", { type: "danger" });
            } finally {
                this.state.loading = false;
            }
        });
    }

    openRecord(recordId) {
        this.action.doAction({
            type: "ir.actions.act_window",
            res_model: "my.model",
            res_id: recordId,
            views: [[false, "form"]],
        });
    }
}

registry.category("actions").add("my_module.MyDashboard", MyDashboard);
```

```xml
<!-- my_module/static/src/xml/my_dashboard.xml -->
<templates xml:space="preserve">
    <t t-name="my_module.MyDashboard">
        <div class="o_my_dashboard">
            <t t-if="state.loading">
                <div class="o_loading_spinner">Đang tải...</div>
            </t>
            <t t-elif="state.error">
                <div class="alert alert-danger" t-esc="state.error"/>
            </t>
            <t t-else="">
                <t t-foreach="state.records" t-as="record" t-key="record.id">
                    <div class="o_record_item" t-on-click="() => openRecord(record.id)">
                        <span t-esc="record.name"/>
                    </div>
                </t>
            </t>
        </div>
    </t>
</templates>
```

```xml
<!-- my_module/views/actions.xml -->
<record id="action_my_dashboard" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.MyDashboard</field>
    <field name="target">current</field>
</record>
```

---

## Custom Field Widget Pattern

```javascript
// my_module/static/src/js/my_field_widget.js
import { Component } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { standardFieldProps } from "@web/views/fields/standard_field_props";

export class MyFieldWidget extends Component {
    static template = "my_module.MyFieldWidget";
    static props = { ...standardFieldProps };

    setup() {
        // Không cần useService nếu widget chỉ display
    }

    get value() {
        return this.props.record.data[this.props.name];
    }
}

registry.category("fields").add("my_widget", {
    component: MyFieldWidget,
    displayName: "My Custom Widget",
    supportedTypes: ["char", "text"],
});
```

```xml
<!-- Dùng trong view -->
<field name="my_field" widget="my_widget"/>
```

---

## Services & Hooks Thường Dùng

### Built-in services (qua `useService()`)

| Service | Import path | Mô tả |
|---------|-------------|-------|
| `orm` | built-in | Gọi ORM: searchRead, create, write, unlink |
| `rpc` | built-in | Gọi controller endpoint trực tiếp |
| `notification` | built-in | Hiển thị thông báo cho user |
| `action` | built-in | Điều hướng action, mở form, wizard |
| `dialog` | built-in | Mở dialog/modal |
| `user` | built-in | Thông tin user hiện tại, context |
| `router` | built-in | Đọc/thay đổi URL hash |

### Built-in hooks (import từ `@web/core/utils/hooks`)

| Hook | Mô tả |
|------|-------|
| `useService(name)` | Inject service vào component |
| `useBus(bus, event, cb)` | Subscribe event, auto-unsubscribe |
| `useAutofocus({ refName })` | Auto-focus element khi mount |
| `usePosition(ref, opts)` | Track vị trí element |
| `useAssets({ jsLibs, cssLibs })` | Lazy-load asset bundles |
| `useSpellCheck()` | Spellcheck integration (mới Odoo 19) |

### Ví dụ useService và useBus

```javascript
import { Component, useState } from "@odoo/owl";
import { useService, useBus } from "@web/core/utils/hooks";

export class MyComp extends Component {
    static template = "my_module.MyComp";

    setup() {
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.state = useState({ count: 0 });

        // Subscribe bus event — auto-unsubscribe khi component destroy
        useBus(this.env.bus, "MY_CUSTOM_EVENT", (ev) => {
            this.state.count++;
        });
    }
}
```

### Đăng ký Custom Service

```javascript
import { registry } from "@web/core/registry";

const myService = {
    dependencies: ["rpc", "notification"],
    start(env, { rpc, notification }) {
        return {
            async fetchData(model, domain) {
                try {
                    return await rpc("/web/dataset/call_kw", {
                        model,
                        method: "search_read",
                        args: [domain],
                        kwargs: { fields: ["name"], limit: 50 },
                    });
                } catch (e) {
                    notification.add("Lỗi fetch data", { type: "danger" });
                    return [];
                }
            },
        };
    },
};

registry.category("services").add("myService", myService);
```

---

## Patching (Mở rộng component/class hiện có)

```javascript
import { patch } from "@web/core/utils/patch";
import { FormRenderer } from "@web/views/form/form_renderer";

// Patch Owl component
const unpatch = patch(FormRenderer.prototype, {
    setup() {
        super.setup();
        // thêm logic
    },
    myNewMethod() {
        return 42;
    },
});

// Để revert (quan trọng trong tests):
// unpatch();
```

---

## Dropdown với External State Control

```javascript
import { Component } from "@odoo/owl";
import { Dropdown } from "@web/core/dropdown/dropdown";
import { useDropdownState, useDropdownCloser } from "@web/core/dropdown/dropdown_hooks";

export class MyMenuComp extends Component {
    static template = "my_module.MyMenuComp";
    static components = { Dropdown };

    setup() {
        // External state control
        this.dropdownState = useDropdownState({
            onOpen: () => console.log("opened"),
            onClose: () => console.log("closed"),
        });
    }
}
```

```xml
<t t-name="my_module.MyMenuComp">
    <Dropdown state="dropdownState">
        <button>Menu</button>
        <t t-set-slot="content">
            <DropdownItem onSelected="() => doAction()">Action 1</DropdownItem>
        </t>
    </Dropdown>
</t>
```

**Lưu ý Odoo 19:** `Dropdown` mặc định `bottomSheet=true` — trên mobile touch device sẽ render dạng bottom sheet thay vì popover.

---

## Asset Declaration trong `__manifest__.py`

```python
'assets': {
    # Backend assets (thường dùng nhất)
    'web.assets_backend': [
        'my_module/static/src/js/my_component.js',
        'my_module/static/src/xml/my_template.xml',
        'my_module/static/src/scss/my_style.scss',
    ],
    # Frontend (website/portal)
    'web.assets_frontend': [
        'my_module/static/src/js/portal_component.js',
    ],
    # Tests — dùng HOOT framework (không phải QUnit)
    'web.assets_tests': [
        'my_module/static/tests/my_component.test.js',
    ],
    # Thêm vào đầu bundle (thay vì cuối)
    'web.assets_backend': [
        ('prepend', 'my_module/static/src/js/first_load.js'),
    ],
    # Xóa file khỏi bundle
    'web.assets_backend': [
        ('remove', 'other_module/static/src/js/unwanted.js'),
    ],
}
```

Ngoài `__manifest__.py`, assets cũng có thể quản lý qua model `ir.asset` trong XML data files.

---

## Built-in Owl Components — Quick Reference

### CheckBox (hỗ trợ `indeterminate` — mới Odoo 19)

```xml
<CheckBox value="state.checked" onChange="(val) => state.checked = val"
          indeterminate="state.isIndeterminate" disabled="false"/>
```

### Pager

```xml
<Pager offset="state.offset" limit="state.limit" total="state.total"
       onUpdate="({offset, limit}) => loadPage(offset, limit)"
       isEditable="true" withAccessKey="true"/>
```

### Notebook (với fieldNames cho form validation highlight)

```xml
<Notebook defaultPage="'tab_general'" onPageUpdate="(page) => onPageChange(page)">
    <t t-set-slot="tab_general" title="'Thông tin chung'" isVisible="true"
       fieldNames="['name', 'code']">
        <!-- content tab 1 -->
    </t>
    <t t-set-slot="tab_detail" title="'Chi tiết'" isVisible="true">
        <!-- content tab 2 -->
    </t>
</Notebook>
```

### SelectMenu (multiSelect với TagsList)

```xml
<SelectMenu choices="choices" value="state.selected"
            multiSelect="true" searchable="true"
            onSelect="(vals) => state.selected = vals"/>
```

---

## HOOT Testing (Odoo 19 — thay thế QUnit)

```javascript
// my_module/static/tests/my_component.test.js
import { describe, expect, test } from "@odoo/hoot";
import { mountWithCleanup } from "@web/../tests/web_test_helpers";
import { MyDashboard } from "../src/js/my_dashboard";

describe("MyDashboard", () => {
    test("renders without error", async () => {
        const comp = await mountWithCleanup(MyDashboard, {
            props: {},
        });
        expect(comp).toBeTruthy();
    });
});
```

**Quan trọng:** HOOT thay thế hoàn toàn QUnit từ Odoo 19. Không dùng `QUnit.test()`, `QUnit.module()`.

---

## Các skill liên quan (tách nhỏ theo chủ đề)

| Skill | Khi nào dùng |
|-------|-------------|
| `odoo-frontend-assets` | Lazy loading, bundle operations nâng cao |
| `odoo-frontend-services-registries` | Registry/service patterns chi tiết |
| `odoo-frontend-qweb-templates` | QWeb template nâng cao, directives |
| `odoo-frontend-testing-hoot` | HOOT test suite, mock server |
| `odoo-frontend-patching-error-handling` | Patch patterns, error_handlers registry |
