---
name: odoo-backend-views
description: Đặc tả và triển khai Odoo 19 XML/UI views phía backend - view records, view architectures, form/list/kanban/search/calendar/graph/pivot, XPath inheritance, modifiers, actions và menus. Use when writing FRD or code for standard Odoo XML views, view inheritance, action/view_mode, search filters, or server-rendered UI.
---

# Odoo Backend Views

## Goal
Giúp agent viết FRD hoặc code view XML đúng Odoo 19, ưu tiên native view trước khi đề xuất custom OWL. Dùng nguồn chính: Odoo 19 View records và View architectures.

## When to use this skill
- Tạo hoặc sửa form, list, kanban, search, calendar, graph, pivot view.
- Đặc tả `ir.ui.view`, `ir.actions.act_window`, menu, button object/action.
- Inherit view bằng `inherit_id`, XPath, `position`.
- Viết modifier Odoo 19: `readonly`, `required`, `invisible`, `column_invisible`.
- Viết FRD có yêu cầu về view XML, view_mode, search filter, server-rendered UI.
- Trigger phrases: "tạo form view", "viết list view", "inherit view", "thêm field vào view", "search filter", "action window", "menu odoo", "kanban view", "calendar view", "graph view", "pivot view".

## Instructions

### Bước 1 — Source-first
Trước khi viết FRD hoặc code view mới, dùng `scan-odoo-module` hoặc `scan-odoo-base` để đọc view/action/menu hiện có. FRD phải ghi XML ID, file hoặc XPath thực tế khi quyết định inherit/modify.

### Bước 2 — Xác định loại view
Tham chiếu bảng loại view trong `GUIDE.md#view-types`. Chọn đúng root tag:
- List: `<list>` (KHÔNG dùng `<tree>`)
- Form: `<form>`
- Kanban: `<kanban>`
- Search: `<search>`
- Calendar: `<calendar>`
- Graph: `<graph>`
- Pivot: `<pivot>`

### Bước 3 — Viết modifier đúng Odoo 19
- Dùng Python expression trực tiếp: `invisible="state == 'cancel'"`.
- KHÔNG dùng `attrs="{...}"` hoặc `states="..."` — đã bị xóa từ v17, ValidationError trong v19.
- Field kỹ thuật dùng trong expression phải hiện diện trong view, có thể `invisible="True"`.
- Tham chiếu đầy đủ tại `GUIDE.md#modifiers`.

### Bước 4 — Viết inheritance XML
- Dùng `<xpath expr="..." position="...">` với position hợp lệ: `inside`, `replace`, `after`, `before`, `attributes`.
- Đặt `name` ổn định cho `page`, `group`, smart button container để dễ inherit.
- Tham chiếu mẫu tại `GUIDE.md#inheritance`.

### Bước 5 — FRD Checklist
Điền đủ các hạng mục sau cho mỗi view:

| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| View metadata | File, View XML ID, Model, View type, Inherit XML ID |
| Inheritance | XPath/locator, `position`, node được thêm/sửa/xóa |
| Modifiers | Điều kiện `invisible`, `readonly`, `required`, `column_invisible` |
| Data behavior | `domain`, `context`, default/search context, `group_by` |
| Access | `groups`, menu/action group, field security liên quan |
| Actions | `ir.actions.act_window`, `view_mode`, `domain`, `context`, `target` |

## Constraints
- `<list>` thay thế hoàn toàn `<tree>` — dùng `<tree>` sẽ fail validation trong v19.
- `attrs` và `states` bị xóa hoàn toàn — dùng sẽ raise ValidationError.
- VIEW_MODIFIERS hợp lệ: `column_invisible`, `invisible`, `readonly`, `required`.
- `column_invisible` chỉ dùng trong list view để ẩn toàn bộ cột; `invisible` ẩn từng cell/field.
- Extension view (`mode=extension`) KHÔNG được có `group_ids` — dùng `groups` attribute bên trong arch.
- Nếu expression dùng field kỹ thuật không hiển thị, field đó phải có trong view với `invisible="True"`.
- Không dùng `_cr`, `_uid`, `_context` trong code — dùng `self.env.cr`, `self.env.uid`, `self.env.context`.
- QWeb view bắt buộc có `key` field (non-null constraint).

## Khi nào chuyển sang skill khác
- Custom field widget, client action, custom view type, dashboard, matrix/grid → `odoo-owl19`.
- QWeb template custom trong kanban/OWL/website → `odoo-qweb`.
- Frontend assets (JS/CSS) → skill frontend assets.

## References
- Odoo 19 View records: https://www.odoo.com/documentation/19.0/developer/reference/user_interface/view_records.html
- Odoo 19 View architectures: https://www.odoo.com/documentation/19.0/developer/reference/user_interface/view_architectures.html
- GUIDE.md: ./references/GUIDE.md
