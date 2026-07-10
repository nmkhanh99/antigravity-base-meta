---
name: odoo-frontend-source-analysis
description: Phân tích source frontend Odoo 19 để viết TSD - đọc manifest assets, static/src JS/XML/SCSS, Owl components, QWeb templates, registries, services, hooks, patches, tests và map vào frontend technical spec. Use before or during odoo-write-tsd when custom frontend evidence is needed.
---

# Odoo Frontend Source Analysis

## Goal
Tạo evidence frontend trước khi viết FRD cho custom UI Odoo 19: OWL components, assets bundles, QWeb templates, registries, services, patching và tests — để tránh đoán và đảm bảo spec phản ánh đúng source.

## When to use this skill
- FRD có custom field widget, client action, custom view type, dashboard, matrix/grid, systray hoặc frontend service
- Custom module có `static/src` hoặc manifest `assets`
- Cần phân biệt XML view chuẩn (server-side) với custom OWL/assets (client-side)
- Khi user nói: "đọc source frontend", "phân tích JS", "check OWL component", "tìm evidence frontend", "scan assets module"
- Trước khi gọi `odoo-frd-owl-component-spec`, `odoo-frontend-assets`, `odoo-frontend-javascript-owl`

## Instructions

### Bước 1 — Thu thập đầu vào bắt buộc
Trước khi bắt đầu, xác nhận:
- **Custom addons parent path(s)**: thư mục cha chứa custom modules
- **Target module**: module sẽ chứa JS/XML/SCSS/assets mới
- **Base/enterprise path** nếu frontend đang reuse/patch asset ngoài workspace
- **Based-on frontend module** nếu user đã biết; nếu chưa, suy từ manifest `depends` và `assets`

Nếu chưa biết target module hoặc custom addons parent path, hỏi user trước. Không tự chọn module.

### Bước 2 — Liệt kê custom modules ứng viên
```bash
find <custom_parent_path> -maxdepth 2 -name "__manifest__.py" | sort
```
Lọc các module có `assets` hoặc thư mục `static/src`.

### Bước 3 — Đọc manifest
Đọc `__manifest__.py` của target module:
- `depends`: xác định based-on frontend modules
- `assets`: bundles, glob paths, operations (append/prepend/before/after/remove/replace)
- `web.assets_unit_tests` nếu có test bundle

### Bước 4 — Đọc static/src
Đọc `static/src/**` của target module và based-on custom modules:
- `*.js` / `*.mjs`: component classes, hooks, services, registries, patches
- `*.xml`: QWeb/OWL templates (`t-name`, `t-inherit`)
- `*.scss` / `*.css`: custom styles

### Bước 5 — Tìm registry
```bash
rg "registry.category" <module_path>/static/src --type js
```
Map category/key → registered object để xác định loại frontend extension.

### Bước 6 — Tìm services, hooks, patches
```bash
rg "useService|useAssets|useBus|usePager|patch\(" <module_path>/static/src --type js
```

### Bước 7 — Tìm QWeb/OWL templates
```bash
rg "t-name|t-inherit|t-foreach|t-out|t-att" <module_path>/static/src --type xml
```

### Bước 8 — Tìm tests
```bash
find <module_path>/static/tests -name "*.js" | sort
```
Xác định test coverage hiện có, test bundle trong manifest.

### Bước 9 — Kích hoạt skill chuyên sâu theo evidence
| Evidence tìm thấy | Skill cần gọi |
|-------------------|---------------|
| OWL component class, `setup()`, props, state | `odoo-frontend-javascript-owl` |
| manifest `assets`, bundles, glob | `odoo-frontend-assets` |
| `*.xml` với `t-name`/`t-inherit` | `odoo-frontend-qweb-templates` |
| `registry.category`, services | `odoo-frontend-services-registries` |
| `static/tests`, `web.assets_unit_tests` | `odoo-frontend-testing-hoot` |
| `patch(`, prototype override | `odoo-frontend-patching-error-handling` |

### Bước 10 — Ghi Evidence vào FRD/notes
Điền bảng Evidence Matrix (xem GUIDE.md) vào FRD section hoặc working notes.

## Constraints
- Không thêm `OWL Component`/`Assets` block trong FRD nếu source không có custom `static/src`
- Không đặc tả OWL nội bộ của Odoo base; chỉ đặc tả custom frontend do dự án build/override
- Mỗi evidence phải ghi **based-on module**; nếu reuse component từ module khác, target phải `depend` module đó hoặc FRD phải mở Open Question
- Nếu source path hoặc target module chưa rõ, hỏi user — không suy đoán
- Nếu dùng `patch(`, nêu blast radius và yêu cầu test regression trong FRD
- Nếu asset dùng glob, vẫn cần chỉ ra file chính chịu trách nhiệm component/action
- Không sửa source khi đang phân tích; chỉ đọc và ghi evidence
- Nếu không tìm thấy module/file, ghi `Not found` và nêu rõ đã kiểm tra path nào

## References
- GUIDE.md: Evidence Matrix chi tiết, template bảng output, ví dụ ripgrep commands
- Odoo 19 Assets: https://www.odoo.com/documentation/19.0/developer/reference/frontend/assets.html
- Odoo 19 OWL: https://www.odoo.com/documentation/19.0/developer/reference/frontend/owl_components.html
- Odoo 19 Registries: https://www.odoo.com/documentation/19.0/developer/reference/frontend/registries.html
- Odoo 19 Services: https://www.odoo.com/documentation/19.0/developer/reference/frontend/services.html
