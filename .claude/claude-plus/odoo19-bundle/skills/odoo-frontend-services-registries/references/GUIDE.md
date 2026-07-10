# GUIDE — Odoo 19 Frontend Services & Registries

## Registry Categories Thường Gặp

| Category | Dùng cho | Root element |
|----------|----------|--------------|
| `actions` | `ir.actions.client` tag | OWL Component |
| `fields` | Field widget trong form/list/kanban | OWL Component |
| `views` | Custom view type/controller/renderer | OWL Component |
| `services` | Long-lived service, DI dependencies | Object `{ dependencies, start }` |
| `main_components` | Top-level webclient component | `{ Component, props? }` |
| `systray` | Item trên navbar systray (root: `<li>`) | `{ Component, props?, isDisplayed? }` |
| `effects` | Custom visual effects | Handler function |
| `user_menuitems` | Item trong dropdown menu user | Function `(env) => { description, ... }` |
| `formatters` | Format giá trị field ra string | Object với `format(value, options?) => string` |
| `parsers` | Parse string thành giá trị field | Object với `parse(value, options?) => T` |

---

## Registry API

```javascript
import { registry } from "@web/core/registry";
import { Registry } from "@web/core/registry"; // nếu cần tạo registry mới

// Truy cập sub-registry (tự tạo nếu chưa có)
const fieldRegistry = registry.category("fields");
const serviceRegistry = registry.category("services");

// Thêm entry
registry.category("actions").add("my_module.MyAction", MyAction);

// Override entry đã tồn tại
registry.category("fields").add("my_widget", MyWidget, { force: true });

// Với sequence (systray: số nhỏ = phải nhất, số lớn = trái nhất)
registry.category("systray").add("my_module.my_item", {
    Component: MySystrayItem,
    isDisplayed: (env) => true,
}, { sequence: 43 }); // default sequence = 50

// Lấy giá trị
const MyAction = registry.category("actions").get("my_module.MyAction");

// Kiểm tra tồn tại
registry.category("fields").contains("my_widget"); // boolean

// Lấy tất cả (sắp xếp theo sequence)
registry.category("main_components").getAll();

// Xóa entry
registry.category("services").remove("my_module.my_service");

// Listen UPDATE event
registry.category("services").addEventListener("UPDATE", (event) => {
    console.log("registry changed", event.detail);
});
```

---

## Service Pattern

```javascript
import { registry } from "@web/core/registry";

export const myService = {
    dependencies: ["notification", "orm"],
    start(env, { notification, orm }) {
        // Trả về object là public API của service
        return {
            async doSomething(recordId) {
                try {
                    const result = await orm.read("res.partner", [recordId], ["name"]);
                    notification.add(`Loaded: ${result[0].name}`, { type: "success" });
                    return result;
                } catch (e) {
                    notification.add("Error loading record", { type: "danger" });
                    throw e;
                }
            },
        };
    },
};

registry.category("services").add("my_module.my_service", myService);
```

### Dùng service trong OWL component
```javascript
import { Component } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

class MyComponent extends Component {
    setup() {
        // useService tự wrap reactive object + bảo vệ async khi component bị destroy
        this.myService = useService("my_module.my_service");
        this.notification = useService("notification");
        this.orm = useService("orm");
        this.action = useService("action");
        this.dialog = useService("dialog");
        this.title = useService("title");
    }

    async onSave() {
        await this.myService.doSomething(this.props.recordId);
    }
}
```

---

## RPC (Odoo 19 — import trực tiếp, KHÔNG dùng useService)

```javascript
import { rpc } from "@web/core/network/rpc";
import { user } from "@web/core/user"; // reactive object, không phải service

// Call RPC
const result = await rpc("/web/dataset/call_kw/res.partner/search_read", {
    model: "res.partner",
    method: "search_read",
    args: [],
    kwargs: {
        domain: [["active", "=", true]],
        fields: ["name", "email"],
        context: user.context,
    },
});

// Abort in-flight request
const promise = rpc("/my/route", {});
promise.abort(); // triggers ConnectionAbortedError

// RPC errors
import { RPCError, ConnectionLostError, ConnectionAbortedError } from "@web/core/network/rpc";
try {
    await rpc("/my/route", {});
} catch (e) {
    if (e instanceof ConnectionLostError) {
        // network fail or HTTP 502
    } else if (e instanceof RPCError) {
        // server-side error
    }
}
```

---

## ORM Service

```javascript
import { useService } from "@web/core/utils/hooks";

class MyComponent extends Component {
    setup() {
        this.orm = useService("orm");
    }

    async loadData() {
        // Basic CRUD
        const records = await this.orm.searchRead(
            "res.partner",
            [["active", "=", true]],
            ["name", "email"],
            { limit: 80, offset: 0 }
        );

        const ids = await this.orm.search("res.partner", [["name", "like", "Odoo"]]);
        const count = await this.orm.searchCount("res.partner", []);
        const newId = await this.orm.create("res.partner", [{ name: "New Partner" }]);
        await this.orm.write("res.partner", [1, 2], { active: false });
        await this.orm.unlink("res.partner", [1]);

        // Silent mode (suppress error notification)
        const silentResult = await this.orm.silent.read("res.partner", [1], ["name"]);

        // Cache mode (RAM)
        const cached = await this.orm.cache({ type: "ram", update: "once" })
            .read("res.partner", [1], ["name"]);

        // formattedReadGroup (Odoo 19 — thay thế read_group deprecated)
        const grouped = await this.orm.formattedReadGroup(
            "sale.order",
            [],
            ["partner_id"],
            { aggregates: ["amount_total:sum"] }
        );

        // Custom method call
        const res = await this.orm.call("res.partner", "my_custom_method", [[1, 2]], {
            context: { lang: "vi_VN" },
        });

        // webRead, webSave (view-level operations)
        const webData = await this.orm.webRead("res.partner", [1], { specification: {} });
        await this.orm.webSave("res.partner", { id: 1, name: "Updated" });
    }
}
```

---

## Notification Service

```javascript
class MyComponent extends Component {
    setup() {
        this.notification = useService("notification");
    }

    showMessages() {
        // Types: 'info' | 'success' | 'warning' | 'danger'
        this.notification.add("Saved successfully!", {
            type: "success",
            sticky: false,
            autocloseDelay: 3000,
            title: "My Module",
        });

        // Sticky với nút bấm
        const close = this.notification.add("Action required", {
            type: "warning",
            sticky: true,
            buttons: [{
                name: "Dismiss",
                onClick: () => close(),
            }],
        });
    }
}
```

---

## Effect Service (RainbowMan)

```javascript
class MyComponent extends Component {
    setup() {
        this.effect = useService("effect");
    }

    celebrate() {
        this.effect.add({
            type: "rainbow_man",
            message: "Well Done!", // string only — KHÔNG truyền HTML Element
            fadeout: "fast",       // 'slow' | 'medium' | 'fast' | 'no'
            img_url: "/web/static/img/smile.svg",
        });
    }
}

// Custom effect (đăng ký vào registry)
import { Component, xml } from "@odoo/owl";
class MyEffect extends Component {
    static template = xml`<div class="my-effect">✨</div>`;
}

registry.category("effects").add("my_effect", (env, params) => {
    return { Component: MyEffect, props: { ...params } };
});
```

---

## User Info (Odoo 19 — plain reactive object)

```javascript
import { user } from "@web/core/user"; // KHÔNG dùng useService("user")

// Properties
console.log(user.name);         // string
console.log(user.login);        // string
console.log(user.userId);       // number
console.log(user.isAdmin);      // boolean
console.log(user.isSystem);     // boolean
console.log(user.isInternalUser); // boolean
console.log(user.partnerId);    // number
console.log(user.context);      // object (lang, tz, uid, ...)
console.log(user.lang);         // string
console.log(user.tz);           // string

// Company
console.log(user.activeCompany);    // { id, name }
console.log(user.activeCompanies);  // array
console.log(user.allowedCompanies); // object

// Async methods
const inGroup = await user.hasGroup("base.group_system"); // Promise<bool>
const canRead = await user.checkAccessRight("res.partner", "read", [1]); // Promise<bool>

await user.setUserSettings("my_setting", "value");
user.updateContext({ default_company_id: 1 });

await user.activateCompanies([1, 2], {
    includeChildCompanies: true,
    reload: true,
});
```

---

## Router (Odoo 19 — path-based URLs)

```javascript
import { router } from "@web/core/browser/router";
// URL format: /odoo/action-{id}/{resId}
// KHÔNG dùng hash /web#action=...

// Read current state
const state = router.current; // { action, resId, model, ... }

// Navigate
router.pushState({ action: "my_module.action_partners", resId: 42 });
router.replaceState({ action: "my_module.action_partners" }, { replace: true });

// Sync (skip debounce)
router.pushState({ action: "my_module.action_partners" }, { sync: true });

// Convert state ↔ URL
const url = router.stateToUrl({ action: "my_module.action_partners", resId: 1 });
const state2 = router.urlToState("/odoo/action-my_module.action_partners/1");

// Control key visibility
router.addLockedKey("my_key");     // persist across navigation
router.hideKeyFromUrl("temp_key"); // hide from URL
```

---

## Cookie Service

```javascript
import { cookie } from "@web/core/browser/cookie";

cookie.set("my_key", "value");           // default TTL = 1 year (31536000s)
cookie.set("session_key", "v", 3600);   // TTL 1 hour
const val = cookie.get("my_key");       // string | undefined
cookie.delete("my_key");
```

---

## HTTP Service

```javascript
class MyComponent extends Component {
    setup() {
        this.http = useService("http");
    }

    async fetchData() {
        // GET — default readMethod = 'json'
        const data = await this.http.get("/my/api/route");

        // POST — params auto-converted to FormData
        const res = await this.http.post("/my/api/route", { key: "value" }, "json");

        // Throws ConnectionLost on HTTP 502, Error on HTTP 413
    }
}
```

---

## Title Service

```javascript
class MyComponent extends Component {
    setup() {
        this.titleService = useService("title");
    }

    onOpen() {
        // Format: "(counter) Part1 - Part2"
        this.titleService.setParts({ zopenerp: "My App", action: "Contacts" });
        this.titleService.setCounters({ messages: 5 });
        console.log(this.titleService.current); // "(5) My App - Contacts"
    }
}
```

---

## Systray Item

```javascript
import { Component, xml } from "@odoo/owl";
import { registry } from "@web/core/registry";

class MySystrayItem extends Component {
    // Root element PHẢI là <li> tag
    static template = xml`
        <li class="o_menu_systray_item my_item">
            <a href="#" t-on-click="onClick">
                <i class="fa fa-bell"/>
            </a>
        </li>
    `;

    onClick(ev) {
        ev.preventDefault();
        // handle click
    }
}

registry.category("systray").add("my_module.my_systray", {
    Component: MySystrayItem,
    isDisplayed: (env) => env.services.user?.isInternalUser ?? false,
}, { sequence: 43 }); // thấp hơn = phải hơn, cao hơn = trái hơn; default 50
```

---

## Main Component

```javascript
import { Component, xml } from "@odoo/owl";
import { registry } from "@web/core/registry";

class MyMainComponent extends Component {
    static template = xml`
        <div class="my_global_component">...</div>
    `;
}

registry.category("main_components").add("my_module.MyMainComponent", {
    Component: MyMainComponent,
    props: { someProp: "value" },
});
```

---

## User Menu Item

```javascript
import { registry } from "@web/core/registry";

registry.category("user_menuitems").add("my_module.my_item", (env) => {
    return {
        description: env._t("My Custom Action"),
        callback: () => {
            env.services.action.doAction("my_module.action_custom");
        },
        hide: !env.services.user?.isAdmin,
        sequence: 50, // default 100; thấp hơn = trên hơn
    };
});
```

---

## Hooks Odoo 19

```javascript
import { useBus, useAutofocus, useSpellCheck } from "@web/core/utils/hooks";
import { usePosition } from "@web/core/position_hook";
import { usePager } from "@web/search/pager_hook";
import { useState, useRef } from "@odoo/owl";

class MyComponent extends Component {
    setup() {
        // useBus — auto cleanup khi unmount
        useBus(this.env.bus, "my-event", (event) => {
            console.log(event.detail);
        });

        // useAutofocus — focus element t-ref="autofocus" khi xuất hiện
        this.inputRef = useAutofocus();

        // useSpellCheck — kích hoạt spellcheck khi focus
        this.spellRef = useSpellCheck();
        this.customRef = useSpellCheck({ refName: "my_textarea" });

        // usePosition — đặt vị trí popper relative to reference
        const togglerRef = useRef("toggler");
        usePosition(() => togglerRef.el, {
            popper: "dropdown",
            position: "bottom-start",
            onPositioned: (el, { direction, variant }) => {
                el.classList.add(`dropdown-${direction}`);
            },
        });

        // usePager — hiển thị pager trong control panel
        const state = useState({ offset: 0, limit: 80, total: 0 });
        usePager(() => ({
            offset: state.offset,
            limit: state.limit,
            total: state.total,
            onUpdate: (newState) => Object.assign(state, newState),
        }));
    }
}
```

---

## FRD Checklist Template

```markdown
### [Tên tính năng] — Registry/Service Spec

| Hạng mục | Mô tả |
|----------|-------|
| Extension point | `registry.category("services").add("my_module.my_service", ...)` |
| Owner file | `my_module/static/src/services/my_service.js` |
| Payload / API | `{ doSomething(recordId): Promise<{...}> }` |
| Dependencies | `["notification", "orm"]` |
| Lifecycle | Start khi app load; async promise bị cancel nếu component destroy |
| Error handling | `notification.add(..., { type: "danger" })` + re-throw |
| Assets bundle | `web.assets_backend` trong `__manifest__.py` |
| Tests | Mock `orm` và `notification` service; kiểm tra return value và error path |
```

---

## x2Many Commands (ORM relational field)

```javascript
// Import helper (Odoo 19)
import { x2ManyCommands } from "@web/core/orm_service";

// Command constants: CREATE=0, UPDATE=1, DELETE=2, UNLINK=3, LINK=4, CLEAR=5, SET=6
const commands = [
    x2ManyCommands.create({ name: "New Line" }),        // [0, false, {...}]
    x2ManyCommands.update(42, { name: "Updated" }),     // [1, 42, {...}]
    x2ManyCommands.delete(42),                          // [2, 42, false]
    x2ManyCommands.unlink(42),                          // [3, 42, false]
    x2ManyCommands.link(42),                            // [4, 42, false]
    x2ManyCommands.clear(),                             // [5, false, false]
    x2ManyCommands.set([1, 2, 3]),                      // [6, false, [1,2,3]]
];

await this.orm.write("sale.order", [orderId], { order_line: commands });
```

---

## Async Service — Behavior khi component bị destroy

```javascript
// useService() tự động bảo vệ: async method của service KHÔNG resolve nếu component đã bị destroy
// (trả về unresolved promise thay vì gây lỗi)

// Pattern an toàn cho long-running operation trong service:
export const myAsyncService = {
    dependencies: ["orm"],
    start(env, { orm }) {
        let isDestroyed = false;

        // Service tự quản lý lifecycle nếu cần cancel logic
        return {
            async loadHeavyData(ids) {
                const result = await orm.read("res.partner", ids, ["name"]);
                // useService() đã handle component destroy case tự động
                return result;
            },
        };
    },
};
```
