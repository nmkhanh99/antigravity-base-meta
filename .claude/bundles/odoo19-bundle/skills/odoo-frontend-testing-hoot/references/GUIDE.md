# GUIDE — Odoo Frontend Testing HOOT (Odoo 19)

> Tài liệu tham khảo chi tiết cho skill `odoo-frontend-testing-hoot`.
> Source: https://www.odoo.com/documentation/19.0/developer/reference/frontend/unit_testing.html

---

## 1. Import Reference

### Framework core
```js
import { describe, expect, test, before, after, beforeEach, afterEach, onError } from "@odoo/hoot";
```

### DOM interaction helpers (từ @odoo/hoot)
```js
import {
  click, dblclick, hover, press, fill, edit, clear,
  check, uncheck, drag, scroll, rightClick, middleClick,
  keyDown, keyUp, pointerDown, pointerUp, select,
  setInputFiles, setInputRange, unload, leave,
} from "@odoo/hoot";
```

### DOM query helpers (từ @odoo/hoot)
```js
import {
  queryOne, queryFirst, queryAll, queryAny,
  queryAllTexts, queryAllValues, queryAllAttributes,
  queryAllRects, queryAllProperties,
  queryAttribute, queryRect, queryText, queryValue,
} from "@odoo/hoot";
```

### DOM state helpers (từ @odoo/hoot)
```js
import {
  isVisible, isDisplayed, isFocusable, isEditable,
  isInDOM, isInViewPort, isScrollable, matches,
  getActiveElement, getFocusableElements,
} from "@odoo/hoot";
```

### Time & network mocks (từ @odoo/hoot — KHÔNG dùng @odoo/hoot-mock)
```js
import {
  advanceFrame, advanceTime, animationFrame, runAllTimers,
  cancelAllTimers, freezeTime, unfreezeTime, tick, microTick,
  delay, setFrameRate, Deferred, waitUntil,
  mockFetch, withFetch, mockWebSocket, mockWorker,
  mockLocation, mockDate, mockLocale, mockTimeZone,
  mockPermission, mockUserAgent, mockSendBeacon,
  mockVibrate, mockMatchMedia, mockTouch,
} from "@odoo/hoot";
```

### Web test helpers (entry point duy nhất)
```js
import {
  // Mock server setup
  defineModels, defineActions, defineMenus, defineParams,
  onRpc, makeMockServer, authenticate, logout, withUser,
  stepAllNetworkCalls,
  // Models & fields
  fields, models, serverState,
  // Environment
  makeMockEnv, getMockEnv, getService,
  mockService, makeDialogMockEnv,
  clearRegistry, restoreRegistry, patchWithCleanup,
  // Component mount
  mountWithCleanup, waitUntilIdle, findComponent, getDropdownMenu,
  // View mount
  mountView, mountViewInDialog, parseViewProps,
  clickSave, clickCancel, clickButton, clickModalButton, clickViewButton,
  fieldInput, expectMarkup,
  // Search helpers
  mountWithSearch, toggleSearchBarMenu, toggleFilterMenu,
  toggleGroupByMenu, toggleFavoriteMenu, editSearch, saveFavorite,
  selectGroup, switchView, pagerNext, pagerPrevious, editPager, getButtons,
  // Kanban helpers
  getKanbanRecord, getKanbanColumn, clickKanbanRecord,
  editKanbanRecord, createKanbanRecord, quickCreateKanbanRecord,
  toggleKanbanRecordDropdown, getKanbanRecordTexts,
} from "@web/../tests/web_test_helpers";
```

---

## 2. Định nghĩa Model Test

```js
class Partner extends models.Model {
  _name = "res.partner";
  _rec_name = "name";

  name = fields.Char({ required: true });
  active = fields.Boolean({ default: true });
  age = fields.Integer();
  salary = fields.Float();
  company_id = fields.Many2one({ relation: "res.company" });
  tag_ids = fields.Many2many({ relation: "res.partner.tag" });
  child_ids = fields.One2many({ relation: "res.partner", relation_field: "parent_id" });
  parent_id = fields.Many2one({ relation: "res.partner" });
  state = fields.Selection({
    selection: [["draft", "Draft"], ["done", "Done"]],
    default: "draft",
  });

  _records = [
    { id: 1, name: "Alice", active: true, age: 30 },
    { id: 2, name: "Bob", active: false, age: 25 },
  ];

  _views = {
    form: `<form><field name="name"/><field name="age"/></form>`,
    list: `<list><field name="name"/></list>`,
    search: `<search/>`,
  };
}

defineModels([Partner]);
```

### Fields có sẵn
`Binary`, `Boolean`, `Char`, `Date`, `Datetime`, `Float`, `Html`, `Image`, `Integer`, `Json`, `Many2many`, `Many2one`, `Many2oneReference`, `Monetary`, `One2many`, `Properties`, `PropertiesDefinition`, `Reference`, `Selection`, `Text`.

---

## 3. Cấu trúc Test cơ bản

```js
import { describe, expect, test } from "@odoo/hoot";
import { click, queryOne, animationFrame } from "@odoo/hoot";
import { defineModels, fields, models, mountView, onRpc } from "@web/../tests/web_test_helpers";

class Partner extends models.Model {
  _name = "res.partner";
  name = fields.Char({ required: true });
  _records = [{ id: 1, name: "Alice" }];
  _views = { form: `<form><field name="name"/></form>` };
}

defineModels([Partner]);

describe("Partner form", () => {
  test("hiển thị tên record", async () => {
    await mountView({ type: "form", resModel: "res.partner", resId: 1 });
    expect(queryOne(".o_field_char input").value).toBe("Alice");
  });

  test("lưu record gọi write", async () => {
    onRpc("res.partner", "write", ({ args }) => {
      expect.step("write:" + args[0][0]);
      return true;
    });
    await mountView({ type: "form", resModel: "res.partner", resId: 1 });
    await click(".o_form_button_save");
    expect.verifySteps(["write:1"]);
  });
});
```

---

## 4. onRpc — 4 overload signatures

```js
// 1. Catch-all callback
onRpc((route, args) => { /* ... */ });

// 2. Theo route
onRpc("/web/dataset/call_kw", (params) => { /* ... */ });

// 3. Theo method (bất kỳ model)
onRpc("write", ({ model, args }) => true);

// 4. Theo model + method (khuyến nghị)
onRpc("res.partner", "write", ({ args, kwargs }) => {
  expect.step("write");
  return true;
});
```

---

## 5. Mock Service

```js
import { mockService } from "@web/../tests/web_test_helpers";

mockService("notification", {
  add: (message, options) => {
    expect.step(`notify:${message}`);
  },
});

mockService("action", {
  doAction: (action) => {
    expect.step(`action:${action}`);
  },
});
```

---

## 6. Mount Component tự do

```js
import { mountWithCleanup } from "@web/../tests/web_test_helpers";
import { MyComponent } from "@my_module/components/my_component";

test("render component", async () => {
  const comp = await mountWithCleanup(MyComponent, {
    props: { title: "Test", items: [] },
  });
  expect(queryOne(".my-component").textContent).toContain("Test");
});
```

`mountWithCleanup` tự destroy sau mỗi test — không cần cleanup thủ công.

---

## 7. Mount View

```js
import { mountView, clickSave, fieldInput } from "@web/../tests/web_test_helpers";

test("form view", async () => {
  await mountView({
    type: "form",
    resModel: "res.partner",
    resId: 1,
    arch: `<form><field name="name"/></form>`,
  });
  await fieldInput("name").edit("Bob");
  await clickSave();
  expect.verifySteps(["write"]);
});

test("list view", async () => {
  await mountView({ type: "list", resModel: "res.partner" });
  expect(queryAll(".o_data_row")).toHaveLength(2);
});
```

---

## 8. stepAllNetworkCalls — Assert toàn bộ RPC calls

```js
test("xác nhận thứ tự RPC", async () => {
  stepAllNetworkCalls();
  await mountView({ type: "form", resModel: "res.partner", resId: 1 });
  expect.verifySteps([
    "/web/action/load",
    "get_views",
    "web_read",
  ]);
});
```

---

## 9. Time Mocking

```js
import { mockDate, runAllTimers, animationFrame, advanceTime } from "@odoo/hoot";

test("timeout trigger", async () => {
  mockDate("2024-06-15 09:00:00");
  await mountWithCleanup(MyTimerComponent);
  await runAllTimers();
  await animationFrame();
  expect(queryText(".timer")).toBe("00:00");
});

test("advance time 5 minutes", async () => {
  await advanceTime(5 * 60 * 1000);
  // ...
});
```

---

## 10. withUser — Chạy test với user cụ thể

```js
import { withUser, authenticate } from "@web/../tests/web_test_helpers";

test("manager thấy nút Delete", async () => {
  await withUser(7, async () => {  // userId = 7
    await mountView({ type: "form", resModel: "res.partner", resId: 1 });
    expect(queryOne(".o_btn_delete")).not.toBe(null);
  });
});
```

---

## 11. Lifecycle hooks

```js
import { before, after, beforeEach, afterEach } from "@odoo/hoot";

describe("Suite với setup", () => {
  before(() => { /* chạy 1 lần trước toàn suite */ });
  after(() => { /* chạy 1 lần sau toàn suite */ });
  beforeEach(() => { /* chạy trước mỗi test */ });
  afterEach(() => { /* chạy sau mỗi test */ });

  test("test case", async () => { /* ... */ });
});
```

---

## 12. patchWithCleanup — Patch tạm trong test

```js
import { patchWithCleanup } from "@web/../tests/web_test_helpers";
import { MyService } from "@my_module/services/my_service";

test("patch method", async () => {
  patchWithCleanup(MyService.prototype, {
    computeTotal() { return 999; },
  });
  // patch tự revert sau khi test kết thúc
});
```

---

## 13. Cấu trúc file manifest

```python
# my_module/__manifest__.py
{
    "name": "My Module",
    "version": "19.0.1.0.0",
    "assets": {
        "web.assets_backend": [
            "my_module/static/src/**/*",
        ],
        "web.assets_unit_tests": [
            "my_module/static/tests/**/*",
        ],
    },
}
```

---

## 14. FRD Checklist cho JS Tests

| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Unit under test | Component/service/registry/action/widget (tên class + path) |
| Test file path | `static/tests/my_component.test.js` |
| Asset bundle | `web.assets_unit_tests` trong manifest |
| Mock models | Class name, fields, `_records`, `_views` cần thiết |
| Mock RPC | `onRpc(model, method, callback)` — route hoặc method |
| Mock services | `mockService(name, factory)` — service nào cần stub |
| User flow | Render → interaction (click/fill) → DOM/state assertion |
| Error flow | RPC error, empty state, permission denied |
| Time-dependent | `mockDate`, `runAllTimers`, `advanceTime` nếu cần |

---

## 15. Test Planning Pattern

```text
static/tests/sale_order_form.test.js
describe("Sale Order Form")
  - renders form với dữ liệu từ mock server
  - thay đổi quantity cập nhật subtotal
  - click Confirm gọi action_confirm RPC
  - RPC error hiển thị notification
  - nút Cancel ẩn khi state = 'sale'
```

---

## 16. Lưu ý Odoo 19 — Thay đổi so với phiên bản cũ

| Cũ (< 17/18) | Mới (Odoo 19) |
|---|---|
| QUnit (`define`, `module`, `test`) | HOOT (`describe`, `test`) |
| `@odoo/hoot-mock` | Deprecated — import từ `@odoo/hoot` trực tiếp |
| `mockRPC` function param | `onRpc(model, method, cb)` |
| Manual mock server object | `class X extends models.Model` + `defineModels` |
| `getFixture()` thủ công | Auto-managed bởi HOOT |
| `mount(Component, { target })` | `mountWithCleanup(Component, options)` |
| `QUnit.test("...", assert =>` | `test("...", async () =>` |
