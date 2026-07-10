---
name: odoo-qweb
description: Viết QWeb template Odoo — directives t-if/t-foreach/t-set/t-field/t-esc, dynamic attributes, template inheritance, report templates, kanban templates, website pages. Kích hoạt khi user nói "qweb", "template", "t-foreach", "t-if", "t-field", "t-call", "kanban template", "report template".
---

# QWeb Template Patterns (Odoo)

## Goal
Viết QWeb templates đúng chuẩn cho reports, kanban, website — directives, formatting, conditions, loops, inheritance.

**Input**: Mô tả cần hiển thị gì, context template nào  
**Output**: QWeb XML đầy đủ

## When to use this skill
- "viết QWeb template", "t-if t-foreach"
- "template cho kanban", "template báo cáo"
- "kế thừa template có sẵn"
- "website template", "portal template"

## Instructions

### Bước 1 — Chọn directive phù hợp

| Directive | Purpose |
|---|---|
| `t-if/t-elif/t-else` | Conditional rendering |
| `t-foreach/t-as` | Loop — biến phụ: `_index`, `_size`, `_first`, `_last`, `_odd` |
| `t-set/t-value` | Variable assignment / expression |
| `t-esc` | Escaped text output (safe, dùng mặc định) |
| `t-field` | Field với formatting (chỉ trong reports/website) |
| `t-raw` | Unescaped HTML (cẩn thận XSS) |
| `t-call` | Include template khác (với `t-set` params) |
| `t-att-{attr}` / `t-attf-{attr}` | Dynamic / format-string attribute |

Xem examples: `references/GUIDE.md#directives`

### Bước 2 — Report Templates
Xem template: `references/GUIDE.md#report`  
Structure: `t-call="web.html_container"` → `t-foreach="docs"` → `t-call="web.external_layout"` → `<div class="page">`.

### Bước 3 — Kanban Templates
Xem template: `references/GUIDE.md#kanban`  
`t-name="kanban-box"` trong `<templates>`, dùng `kanban_color(record.color.raw_value)`.

### Bước 4 — Website / Portal Templates
Xem template: `references/GUIDE.md#website`  
`t-call="website.layout"` cho website, `t-call="portal.portal_layout"` cho portal.

### Bước 5 — Template Inheritance
Xem template: `references/GUIDE.md#inheritance`  
`inherit_id="..."` + `<xpath expr="..." position="...">`.

### Bước 6 — Expressions thường dùng
Xem `references/GUIDE.md#expressions` — string ops, number formatting, date, list ops.

## Constraints
- **KHÔNG** dùng `t-raw` với user data — XSS risk
- `t-field` chỉ dùng trong reports/website (không trong OWL templates)
- Trong OWL dùng `t-esc` hoặc `t-out`
- `t-key` bắt buộc trong OWL foreach

## Best practices
- `t-esc` mặc định cho safety
- `t-field` trong reports để formatting tự động (monetary, date...)
- `t-call` cho reusable blocks (DRY)
- Check null trước khi access field: `t-if="record.partner_id"`
