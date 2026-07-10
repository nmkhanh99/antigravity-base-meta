# Odoo Frontend Source Analysis — Reference Guide

## 1. Evidence Matrix

| FRD section | Source cần đọc | Evidence bắt buộc |
|-------------|----------------|-------------------|
| Views | XML view/action + `static/src` | UI type: XML/native, widget name, client action, custom view, based-on module |
| OWL Component | JS class + XML template | component name, template `t-name`, props, state, lifecycle hooks |
| Assets | `__manifest__.py` `assets` | bundle name, file path, operation, load order, lazy assets |
| QWeb Template | `*.xml` với `t-name`/`t-inherit` | `t-name`, `t-inherit`, directives dùng, data context |
| Registry | JS `registry.category(...)` | category, key, registered object/class |
| Services/Hooks | JS `useService`/`useBus`/`usePager` | service name, hook call site, async behavior |
| Patching | JS `patch(` | target class/object, method overridden, `super` call, unpatch risk |
| Tests | `static/tests/**`, `web.assets_unit_tests` | existing/missing JS test coverage per component |

---

## 2. Output Template — Frontend Source Evidence

Ghi vào FRD section hoặc working notes sau khi hoàn thành phân tích:

```markdown
## Frontend Source Evidence

| Area | Based-on module | Module type | Source | Finding | FRD decision |
|------|-----------------|-------------|--------|---------|--------------|
| Assets | `my_module` | Current | `my_module/__manifest__.py` | `web.assets_backend` → load `static/src/js/my_widget.js`, `static/src/xml/my_widget.xml`, `static/src/scss/my_widget.scss` | Khai báo Assets block trong Views section FRD |
| OWL Component | `web` / `my_module` | Odoo base / Current | `static/src/js/my_widget.js` | `class MyWidget extends Component` — props: `{value, readonly}`, state: `{editing}` | Đặc tả OWL Component spec trong FRD |
| Registry | `web` | Odoo base | `static/src/js/my_widget.js` | `registry.category("fields").add("my_widget", MyWidget)` | Field Widget — dùng `widget="my_widget"` trong form view |
| QWeb | `my_module` | Current | `static/src/xml/my_widget.xml` | `t-name="my_module.MyWidget"`, dùng `t-foreach`, `t-out` | Ghi template name trong OWL Component spec |
| Patching | `web` | Odoo base | `static/src/js/form_patch.js` | `patch(FormController.prototype, {...})`, gọi `super` | Nêu blast radius trong FRD; yêu cầu test regression |
| Tests | `my_module` | Current | `static/tests/my_widget_test.js` | Coverage: render + interaction — MISSING: error state | Thêm test case error state vào Test Cases section |
```

---

## 3. Ripgrep Commands Reference

### 3.1 Tìm tất cả custom modules có static/src
```bash
find <custom_parent_path> -maxdepth 3 -type d -name "static" | grep "/src$" | sed 's|/static/src||'
```

### 3.2 Tìm registry categories
```bash
rg "registry\.category\(" <module_path>/static/src --type js -n
```
Output mẫu:
```
static/src/js/dashboard.js:8:registry.category("actions").add("my_module.Dashboard", Dashboard);
static/src/js/my_widget.js:5:registry.category("fields").add("my_widget", MyWidget);
```

### 3.3 Tìm services và hooks
```bash
rg "useService|useAssets|useBus|usePager|useSortBy|useModel" <module_path>/static/src --type js -n
```

### 3.4 Tìm patch calls
```bash
rg "patch\(" <module_path>/static/src --type js -n
```
Kết quả cần kiểm tra:
- Target class/prototype
- Method được override
- Có gọi `super` không

### 3.5 Tìm QWweb template names và inheritance
```bash
rg "t-name=|t-inherit=" <module_path>/static/src --type xml -n
```

### 3.6 Tìm OWL component lifecycle
```bash
rg "setup\(\)|willStart\(\)|mounted\(\)|willUnmount\(\)|onWillStart|onMounted|onWillUnmount" <module_path>/static/src --type js -n
```

### 3.7 Tìm test files
```bash
find <module_path>/static/tests -name "*.js" | sort
```

### 3.8 Kiểm tra lazy loading
```bash
rg "loadAssets|useAssets" <module_path>/static/src --type js -n
```

---

## 4. Registry Categories Phổ Biến (Odoo 19)

| Category | Mục đích | Ví dụ key |
|----------|----------|-----------|
| `actions` | Client actions (client-side routing) | `"my_module.MyDashboard"` |
| `fields` | Custom field widgets | `"my_widget"`, `"signature"` |
| `views` | Custom view types | `"my_view"` |
| `services` | Frontend services | `"my_service"` |
| `main_components` | Components mount vào root app | `"MyRootComponent"` |
| `systray` | Systray icons/menus | `"MyIcon"` |
| `error_handlers` | Global JS error handlers | `"my_error_handler"` |

---

## 5. Asset Bundles Phổ Biến (Odoo 19)

| Bundle | Dùng cho |
|--------|----------|
| `web.assets_backend` | JS/XML/SCSS cho web client backend |
| `web.assets_common` | Assets dùng chung (frontend + backend) |
| `web.assets_frontend` | Assets cho website/portal |
| `web.assets_qweb` | QWeb templates (legacy — dùng `web.assets_backend` cho OWL) |
| `web.assets_unit_tests` | HOOT JS unit tests |
| `web.report_assets_common` | Assets cho QWeb PDF reports |

**Không dùng** key `qweb`, `css`, `js` cũ trong manifest — legacy từ Odoo < 15.

---

## 6. Phân biệt View Types

| Tình huống | Loại view | Có cần `static/src`? |
|------------|-----------|----------------------|
| Form, List, Kanban chuẩn | XML view | Không |
| Field widget custom | OWL Component + Registry `fields` | Có |
| Client action (màn hình riêng) | OWL Component + Registry `actions` | Có |
| Custom view type | OWL Component + Registry `views` | Có |
| Kanban custom template | QWeb XML (t-name) | Có (XML trong assets) |
| Systray icon | OWL Component + Registry `systray` | Có |
| Dashboard | OWL Component + Registry `actions` | Có |

---

## 7. Patching — Blast Radius Checklist

Khi tìm thấy `patch(` trong source, cần ghi vào evidence:

```
Patch target: <ClassName>.prototype.<methodName>
Module bị ảnh hưởng: tất cả component/view dùng <ClassName>
Super call: Có / Không — nếu Không: override hoàn toàn, blast radius cao
Unpatch: patch() trả về unpatch function — tests PHẢI gọi unpatch() sau mỗi test
Conflict risk: kiểm tra có patch khác trên cùng method không (rg "patch.*<ClassName>")
FRD decision: ghi rõ yêu cầu test regression cho tất cả view dùng <ClassName>
```

---

## 8. Checklist Trước Khi Kết Thúc Phân Tích

- [ ] Đã đọc `__manifest__.py` — ghi rõ `depends` và `assets` bundles
- [ ] Đã đọc tất cả JS trong `static/src` của target module
- [ ] Đã đọc tất cả XML templates trong `static/src` của target module
- [ ] Đã map mọi `registry.category` call → FRD category
- [ ] Đã kiểm tra `patch(` calls — ghi blast radius nếu có
- [ ] Đã kiểm tra test coverage trong `static/tests`
- [ ] Đã xác định based-on module cho mỗi evidence
- [ ] Đã kiểm tra dependency: nếu reuse component từ module khác, target `__manifest__.py` đã `depend` module đó chưa
- [ ] Đã điền bảng Evidence vào FRD hoặc working notes
- [ ] Đã kích hoạt skill chuyên sâu phù hợp cho từng area
