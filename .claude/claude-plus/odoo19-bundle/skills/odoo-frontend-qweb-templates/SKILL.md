---
name: odoo-frontend-qweb-templates
description: Hướng dẫn QWeb frontend trong Odoo 19 - OWL templates, kanban templates, website/portal snippets, template inheritance, t-name/t-inherit/t-out/t-foreach/t-att và rendering an toàn. Use when specifying or writing frontend QWeb templates, not QWeb PDF reports.
---

# Odoo Frontend QWeb Templates

## Goal
Giúp agent đặc tả hoặc viết QWeb template frontend Odoo 19 cho OWL, kanban, website/portal và template inheritance. QWeb report PDF dùng skill `odoo-backend-reports`.

## When to use this skill
- Viết OWL component XML template trong `static/src/.../*.xml`
- Viết kanban card/template custom
- Viết website/portal frontend templates
- Kế thừa template bằng `t-inherit` và XPath
- Câu hỏi về `t-out`, `t-foreach`, `t-att`, `t-call`, `t-set`, `t-if`
- Đặc tả FRD phần frontend QWeb

## Instructions

### Bước 1 — Source-first
Trước khi viết mới, dùng `odoo-backend-source-analysis` (hoặc đọc file XML hiện có) để xác định:
- `t-name` của template gốc cần kế thừa
- XPath target hợp lệ
- Data context/props đã có sẵn

### Bước 2 — Chọn loại template
| Loại | File location | Wrapper |
|------|--------------|---------|
| OWL Component | `static/src/components/*.xml` | `<templates xml:space="preserve">` |
| Kanban | `views/*_kanban_view.xml` | `<kanban>` > `<templates>` |
| Website/Portal | `views/website_*.xml` | `<odoo><data>` |
| Template kế thừa | Cùng thư mục với loại tương ứng | `t-inherit` + XPath |

### Bước 3 — Áp dụng directives đúng cách
- Dùng `t-out` cho mọi output — auto-escape HTML, hỗ trợ cả `Markup` safe content
- Dùng `t-key` bắt buộc trong mọi `t-foreach` của OWL template
- Dùng `t-attf-{name}` cho dynamic class/attr với format string (`{{ }}`)
- Dùng `t-att` cho dict nhiều attr hoặc pair `[name, value]` một attr
- Dùng `t-set` với body (không có `t-value`) để capture safe HTML content
- Dùng `t-call` kèm `t-out="0"` bên trong callee để inject body từ caller

### Bước 4 — Template inheritance
- Dùng `t-inherit` + `t-inherit-mode="primary"` (tạo template mới) hoặc `"extension"` (sửa in-place)
- XPath phải trỏ đúng node; `position="inside|before|after|replace|attributes"`
- Không dùng `t-extend` + `t-jquery` (deprecated)

### Bước 5 — FRD Checklist
Xem bảng đầy đủ trong `references/GUIDE.md` > FRD Checklist section.

### Bước 6 — Safety review
- Không dùng `t-raw` (deprecated từ Odoo 15+)
- Chỉ output HTML không escaped khi dùng `markupsafe.Markup` từ Python hoặc `t-set` body
- Không render raw user input trực tiếp

## Constraints
- `t-raw` bị deprecated — KHÔNG dùng; thay bằng `t-out` + `Markup`
- `t-extend` + `t-jquery` bị deprecated — KHÔNG dùng cho code mới
- Loop variable `{as}_all`, `{as}_parity`, `{as}_even`, `{as}_odd` deprecated — dùng `{as}_index % 2`
- Không dùng `t-esc` (tuy không deprecated nhưng `t-out` là chuẩn Odoo 19)
- `t-field` chỉ dùng phía Python/server-side; không dùng trong OWL JS template
- Mọi OWL template phải nằm trong `<templates xml:space="preserve">`
- Tên template theo convention: `addon_name.ComponentName` hoặc `addon_name.TemplateName`
- `t-set` bên trong `t-call` body là local scope — không visible sau call

## References
- Odoo 19 QWeb frontend: https://www.odoo.com/documentation/19.0/developer/reference/frontend/qweb.html
- Odoo 19 OWL components: https://www.odoo.com/documentation/19.0/developer/reference/frontend/owl_components.html
- Code examples chi tiết: `references/GUIDE.md`
