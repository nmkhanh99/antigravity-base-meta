---
name: odoo-owl19
description: Viết OWL 3.x component Odoo 19 — component structure, hooks, field widget, template, registry. Kích hoạt khi user nói "owl component", "viết component", "tạo widget", "OWL 3", "frontend odoo", "javascript odoo", "custom widget".
---

# Odoo OWL 3.x Component (v19)

## Goal
Tạo OWL 3.x components đúng chuẩn Odoo 19 — basic component, field widget, client action — với lifecycle hooks, JSDoc annotations và cleanup pattern.

**Input**: Mô tả component cần tạo  
**Output**: File JS + XML template + đăng ký manifest

## When to use this skill
- "tạo OWL component", "viết custom widget"
- "tạo client action với OWL"
- "tạo field widget tùy chỉnh"
- "OWL 3.x", "Owl frontend Odoo 19"

## Instructions

### Bước 1 — Quy tắc OWL 3.x bắt buộc
- Tất cả hooks (`useState`, `useRef`, ...) phải trong `setup()`
- Dùng `/** @odoo-module **/` directive ở đầu file
- JSDoc type annotations cho props và state
- Cleanup trong `onWillUnmount`
- Import từ `"@odoo/owl"` (không phải owljs)

### Bước 2 — Basic Component
Component đầy đủ với services, state, refs, lifecycle hooks, async data loading với AbortController, và event cleanup.
Xem template đầy đủ: `references/GUIDE.md#basic-component`

Key points:
- Services inject qua `useService()` trong `setup()`
- `useState({})` tạo reactive state — mutate trực tiếp được
- `onWillUnmount` cleanup event listeners và abort async operations
- Đăng ký client action: `registry.category("actions").add(...)`

### Bước 3 — XML Template
Template với loading state, error state, empty state, và data rendering.
Xem template đầy đủ: `references/GUIDE.md#xml-template`

Key points:
- `t-if/t-elif/t-else` cho conditional rendering
- `t-attf-class` cho dynamic CSS classes
- `t-on-click.stop` để ngăn event propagation
- `t-att-disabled` cho conditional disable

### Bước 4 — Custom Field Widget
Widget kế thừa `standardFieldProps`, đăng ký vào `registry.category("fields")`.
Xem template: `references/GUIDE.md#field-widget`

Key points:
- `...standardFieldProps` để có đủ props chuẩn
- `this.props.record.update({[this.props.name]: value})` để update field value
- `supportedTypes` xác định field types widget hỗ trợ
- `extractProps` map attrs XML → component props

### Bước 5 — Manifest assets
```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/js/my_component.js',
        'my_module/static/src/xml/my_component.xml',
        'my_module/static/src/scss/my_component.scss',
    ],
},
```

### Bước 6 — OWL 3.x Checklist
```
□ /** @odoo-module **/ ở đầu file
□ Import từ "@odoo/owl"
□ JSDoc type annotations cho props và state
□ static props với type validation
□ static defaultProps cho optional props
□ Tất cả hooks trong setup()
□ Cleanup trong onWillUnmount
□ AbortController cho async operations
□ File được khai báo trong manifest assets
```

## Constraints
- **KHÔNG** đặt hooks ngoài `setup()` — sẽ crash trong OWL 3.x
- **KHÔNG** dùng OWL 2.x patterns
- **KHÔNG** import component mà không đăng ký trong `registry`

## Best practices
- Luôn cleanup event listeners trong `onWillUnmount`
- Dùng `AbortController` cho fetch/async để tránh memory leak
- `static props` validation giúp debug rõ ràng hơn
- `t-on-click.stop` để ngăn event propagation khi cần
- State Set/Map trong OWL 3.x có thể mutate trực tiếp (reactive)
